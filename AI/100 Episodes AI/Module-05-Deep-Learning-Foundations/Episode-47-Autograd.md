# Day 47: Autograd

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** intermediate. You wrote backpropagation by hand on Day 43 and you know tensors from Day 46. Today you let PyTorch do the backward pass, and you learn exactly where its rules can surprise you.

**By the end of this day, you will have:**

- A clear model of how PyTorch records a computational graph while the forward pass runs
- Hands-on evidence of gradient accumulation, detaching and the difference between training and no-grad evaluation
- A test that matches PyTorch's gradients to the hand-written gradients from Day 43, to float64 precision
- A small toolkit for gradient control: zeroing, clipping, freezing, hooks and gradient checks

**What you need:**

- A computer with Python 3.11 or 3.12
- PyTorch 2.5.1 installed in the environment from Day 46 (CPU or CUDA build)
- NumPy 2.1.3 and pytest 8.3.3, as in earlier days
- The `TwoLayerNet` forward and backward from your Day 43 repository, which you will compare against
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 43: the computational graph, the chain rule and the backward pass
- Day 46: tensors, shapes, dtypes and devices
- Day 45: train and eval modes, since they affect what gets recorded

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Computational graphs](#computational-graphs)
- [requires_grad](#requires_grad)
- [Gradient accumulation](#gradient-accumulation)
- [backward](#backward)
- [Detaching](#detaching)
- [Gradient control](#gradient-control)
- [Hands-on exercises](#hands-on-exercises)
- [Project: autograd checked against hand-written gradients](#project-autograd-checked-against-hand-written-gradients)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Writing the backward pass by hand, as you did on Day 43, is the best way to understand gradients. It is also too slow and too error-prone for a real model. A transformer has hundreds of layers and dozens of operation types, and nobody writes its gradients manually. Autograd is the PyTorch system that records every tensor operation during the forward pass, builds a graph from it, and runs the backward pass for you.

The system is simple once you see its rules. Operations on tensors that require gradients are recorded. Calling `backward` on a scalar loss walks the recorded graph in reverse and accumulates gradients into the `.grad` field of each leaf tensor. Most of the surprises come from three rules: gradients accumulate rather than replace, the graph is freed after one backward pass unless you ask otherwise, and in-place changes to values needed for the backward pass raise errors.

On the job, autograd bugs show up as models that do not train, gradients that double between steps, memory that grows every iteration, or errors that mention an inplace operation. Engineers who understand the rules fix them in minutes. Engineers who do not often spend a day rewriting code that was almost correct.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1970 | Seppo Linnainmaa describes reverse-mode automatic differentiation, which computes many partial derivatives in one backward pass. | The algorithm that autograd implements. |
| 1980s | Neural network researchers adopt backpropagation as a special case, and frameworks such as Theano begin to build graphs symbolically. | Automatic gradients become a feature of research tools. |
| 2015 | Theano and early TensorFlow compile static graphs ahead of execution, which makes debugging hard for many researchers. | The motivation for dynamic graphs. |
| 2015 | Chainer introduces define-by-run, where the graph is recorded as the Python code runs. | The idea behind PyTorch autograd used in this lesson. |
| 2017 | PyTorch is released, with eager define-by-run autograd, and later versions add higher-order gradients and more operations. | Autograd now handles most operations used in research and production. |

**Takeaway:** PyTorch's autograd is reverse-mode automatic differentiation applied to a graph recorded as the code runs. The graph is the same one you drew on Day 43, built for you.

---

## Glossary

- **Autograd**: PyTorch's automatic differentiation system, which records operations and computes gradients.
- **Leaf tensor**: a tensor created directly by the user, such as a parameter, rather than the result of an operation. Only leaves store `.grad` by default.
- **requires_grad**: a flag on a tensor that tells autograd to record operations involving it.
- **grad_fn**: an attribute of a non-leaf tensor that points to the operation that created it, used to walk the graph backward.
- **Gradient accumulation**: the behavior in which each call to `backward` adds to `.grad` instead of replacing it.
- **Detach**: returning a tensor that shares data with the original but is cut off from the graph.
- **no_grad**: a context that disables recording, used during evaluation and inference.
- **inference_mode**: a stricter context than no_grad that also disables some internal bookkeeping, used for serving.
- **retain_graph**: an option to `backward` that keeps the graph after the pass, so you can run backward again.
- **Gradient clipping**: rescaling the gradients when their total norm exceeds a threshold.
- **Hook**: a function registered on a tensor that runs when its gradient is computed, and can inspect or modify it.
- **Gradient check**: a comparison of an autograd gradient with a finite-difference estimate.

---

## Computational graphs

Imagine a stenographer who records every step of a meeting in order, so that the minutes can be replayed backward when someone asks how a decision was reached. Autograd is that stenographer. It records each operation as the forward pass runs, and it keeps enough information to replay the chain in reverse.

From first principles, when you perform an operation on tensors that require gradients, PyTorch creates a new output tensor and attaches a `grad_fn` that points to the operation and to its inputs. The result is a graph whose nodes are operations and whose edges are the input-output relationships. The graph is built during the forward pass, which is why PyTorch is called define-by-run. A Python `if` statement inside the model decides which operations are recorded on that pass.

```python
import torch

x = torch.tensor(2.0, requires_grad=True)
y = x * x + 3 * x
print(y.grad_fn)             # shows the operation that produced y, such as AddBackward0
print(y.grad_fn.next_functions)
```

The limitation of the graph is memory. Every intermediate value needed for the backward pass is kept until the backward pass runs. A large model with a large batch can exhaust GPU memory during the forward pass, even though the forward pass alone would fit. Techniques such as gradient checkpointing trade computation for memory by recomputing some intermediate values.

Do not assume that the graph is built only when you call `backward`. It is built as operations run. If you run a long forward pass that you do not need to differentiate, you waste memory, and the fix is to use `torch.no_grad`, which you will meet below. In the real market, memory problems in training are often caused by graphs that are kept alive by a stored loss or by a Python list that holds tensors across iterations.

---

## requires_grad

A tensor with `requires_grad=True` is like a microphone switched on in a room. Everything said near it is recorded. Tensors without the flag are silent, and operations on them alone are not recorded. Setting the flag on the parameters of a model is the first step of training.

From first principles, when any input to an operation requires gradients, the output requires gradients too. This propagation is automatic, so you usually set the flag only on the leaves, such as the weights. In a model, the weights are created with `requires_grad=True` by the layer definitions, and the input data normally does not require gradients, because you do not need the gradient of the loss with respect to the data.

```python
import torch

w = torch.randn(3, requires_grad=True)
x = torch.randn(3)                   # data, no gradient needed
y = (w * x).sum()
print(w.requires_grad, x.requires_grad, y.requires_grad)   # True False True
```

The limitation is that `requires_grad` cannot be set on a non-leaf tensor after it was created by an operation that did not track it. If you create a tensor from a NumPy array, it does not require gradients by default, and you must set the flag yourself.

Do not set `requires_grad=True` on data tensors out of caution. It wastes memory and time, and it can mask a bug where the code should have detached a value. In the real market, a frozen pretrained layer is one with its `requires_grad` set to `False`, which is a deliberate choice that saves memory and keeps the pretrained weights fixed.

---

## Gradient accumulation

Picture a tally sheet on which a judge adds points after each performance. The sheet is not cleared between performances, so the total grows. Autograd works the same way: each `backward` call adds its gradients to `.grad` and never clears them. You must reset the sheet yourself.

From first principles, `.grad` is a tensor that is created the first time gradients flow into a leaf, and every later backward pass adds to it with `+=`. This design allows gradient accumulation across several mini-batches, which is how you simulate a larger batch than fits in memory: run several small batches, accumulate, then update once. It also causes a common bug, where forgetting to clear the gradients makes each update larger than intended.

```python
import torch

x = torch.tensor(2.0, requires_grad=True)
for _ in range(2):
    y = x * x + 3 * x
    y.backward()
print(x.grad)                # tensor(14.): 7 from each backward pass
x.grad.zero_()
```

The limitation of accumulation is that it is only correct if the loss is scaled properly. When you accumulate over k micro-batches, divide each loss by k, or the effective gradient is k times larger than a single batch of the same total size.

Do not forget `optimizer.zero_grad()` at the start of each training step. The usual pattern is zero, forward, loss, backward, step, in that order. The order matters, because zeroing after the backward pass throws away the gradients before the optimizer uses them. In industry, a missing zero call usually makes the loss curve look plausible but slightly unstable, which makes it hard to diagnose without comparing against a clean run.

---

## backward

Calling `backward` is like pressing the rewind button on the recorded graph. It starts from the tensor you call it on and runs the backward pass through every recorded operation, computing the gradient of that tensor with respect to every leaf that requires gradients.

From first principles, `backward` expects a scalar by default, because the gradient of a scalar with respect to itself is one. If the tensor is not a scalar, you must pass a gradient argument of the same shape, which is the upstream gradient, or reduce the tensor to a scalar first, usually by summing or averaging the loss. After `backward` runs, the graph is freed by default, so calling it again raises an error unless you pass `retain_graph=True`. Keeping the graph costs memory, so you should do it only when you need a second pass, such as for some higher-order gradients.

```python
import torch

x = torch.tensor([1.0, 2.0, 3.0], requires_grad=True)
y = (x ** 2).sum()           # scalar loss
y.backward()
print(x.grad)                # tensor([2., 4., 6.]), which is 2x

z = x * 2                    # non-scalar output
z.sum().backward()           # reduce to a scalar first
```

The limitation of `backward` is that it gives the gradient only of the loss you name. If you want the gradient of an intermediate value, you need a hook or `torch.autograd.grad`. Also, the backward pass must match the forward pass in device and dtype, and a mismatch raises a clear error.

Do not call `backward` on a loss that has already been detached or converted to a Python float with `.item()`. The `.item()` call returns a plain number that is not part of the graph, so `backward` fails or, worse, does nothing useful. Keep the loss as a tensor until after the backward pass, and call `.item()` only for logging.

---

## Detaching

Think of a photocopy of a document. The copy has the same text, but edits to the copy do not change the original, and the copy has no link to the author's notes. Detaching a tensor makes that kind of copy in the graph: the values are shared, but the history is cut.

From first principles, `tensor.detach()` returns a tensor that shares storage with the original and has `requires_grad=False`, with no `grad_fn`. Gradients will not flow through it. You use detaching when you need a value for logging, for a target that should not be trained, or for a stop-gradient in a loss, such as a target network in reinforcement learning or a teacher model in distillation. The `torch.no_grad` context is a broader tool that disables recording for every operation inside it.

```python
import torch

x = torch.tensor(2.0, requires_grad=True)
y = x * 3
z = (y.detach() * x)         # gradient flows only through the second x
z.backward()
print(x.grad)                # tensor(6.): y is treated as a constant of 6
```

The limitation of detach is that it shares memory. Changing the detached tensor in place changes the original, which can break autograd if the original is needed in the backward pass. Use `clone` if you need an independent copy with the same values.

Do not detach by accident, for example by converting to NumPy with `.numpy()` in the middle of a loss computation, which raises an error when the tensor requires gradients, or by moving it to a Python list. The fix is to convert only at the final logging step. In the real market, a silent detach is a common reason a model has parameters that never change, and a quick check of `.grad` after one step catches it.

---

## Gradient control

Gradient control is the set of tools that decide which gradients are computed, how they are used and when they are cleared. The main tools are `torch.no_grad` and `torch.inference_mode` for evaluation, `requires_grad_` for freezing, `clip_grad_norm_` for stability, and hooks for inspection.

From first principles, `torch.no_grad()` disables recording inside its block, which saves memory and time. Use it for validation and for any forward pass that does not need gradients. `torch.inference_mode()` is a stricter version for deployment, and its outputs cannot be used in autograd later. Freezing a layer sets `requires_grad` to `False` on its parameters, so the optimizer skips them after their gradients are cleared. Clipping rescales the gradient norm in place with `torch.nn.utils.clip_grad_norm_`, which returns the norm before clipping, and that value is worth logging. A hook, registered with `register_hook`, receives the gradient and can return a modified one.

```python
import torch
import torch.nn as nn

model = nn.Linear(4, 1)
x = torch.randn(8, 4)
y = torch.randn(8, 1)

loss = ((model(x) - y) ** 2).mean()
loss.backward()
total_norm = nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
print(float(total_norm))     # the norm before clipping

with torch.no_grad():
    preds = model(x)         # no graph is built
print(preds.requires_grad)   # False
```

The limitation of clipping is that it changes the direction of the update when it triggers, and a model that always clips is not really training at the rate you set. Log how often it triggers, as Day 44 recommended. Freezing also has a subtle risk: a frozen layer still runs its forward pass, so it still costs time and memory, and freezing many layers can make a model unable to adapt to a new domain.

Do not evaluate a model without `no_grad` and then wonder why memory grows during validation. Each validation batch builds a graph that is never used, and the memory is kept until the references are dropped. In industry, the standard evaluation loop puts the model in eval mode and wraps the forward pass in `torch.no_grad`, and the two calls do different jobs, so both are needed.

---

## Hands-on exercises

1. **Gradient by hand.** For y = x^3 + 2x at x = 1.5, compute dy/dx by hand. Then compute it with autograd and compare.
2. **Accumulation count.** Run the `backward` call three times on the same scalar without zeroing, and print `.grad` after each run. Explain the pattern in one sentence.
3. **Retain graph.** Build a graph, call `backward` once, then call it again without `retain_graph`. Read the error and explain in your own words what the framework is protecting you from.
4. **Detach experiment.** Build the expression z = x * x with a detached copy of x in one factor. Compare the gradient with and without the detach, and explain the difference.
5. **Clipping trace.** Create a linear model with large random inputs and targets, run one backward pass, and print the norm returned by `clip_grad_norm_` with max_norm 1.0. Then print the new norm of the gradients, and confirm that it equals 1.0 or less.

---

## Project: autograd checked against hand-written gradients

**Goal:** rebuild the Day 43 two-layer network in PyTorch, check that autograd produces the same gradients as your hand-written backward pass, and build a small set of gradient control tools with tests. Write up the cases where autograd surprised you.

**Repo name:** `100-days-ai-day-47-autograd`

**Repo layout:**

```
100-days-ai-day-47-autograd/
├── README.md
├── requirements.txt
├── src/
│   └── autograd_lab/
│       ├── __init__.py
│       ├── torch_net.py
│       ├── control.py
│       └── numpy_reference.py
├── tests/
│   ├── test_matches_handwritten.py
│   ├── test_accumulation.py
│   └── test_control.py
└── experiments/
    └── compare_gradients.py
```

**Starter code.** Create `requirements.txt` with PyTorch 2.5.1, NumPy 2.1.3 and pytest 8.3.3 pinned. Copy `network.py` from Day 43 into `src/autograd_lab/numpy_reference.py`, keeping its `forward` and `backward` methods. Then create `src/autograd_lab/torch_net.py`:

```python
import torch


def forward(params, X):
    H = torch.tanh(X @ params["W1"] + params["b1"])
    return torch.sigmoid(H @ params["W2"] + params["b2"])


def bce_loss(p, y):
    return -(y * torch.log(p) + (1 - y) * torch.log(1 - p)).mean()


def params_from_numpy(params_np, dtype=torch.float64):
    return {
        k: torch.tensor(v, dtype=dtype, requires_grad=True)
        for k, v in params_np.items()
    }
```

Create `src/autograd_lab/control.py` with two helpers: `freeze(module)`, which sets `requires_grad` to `False` on all parameters, and `grad_norm(parameters)`, which returns the total L2 norm of the gradients as a float without modifying them.

**Step-by-step tasks:**

1. Create the repo, initialize Git, make a virtual environment with Python 3.11 or 3.12 and install the pinned packages.
2. Write `tests/test_matches_handwritten.py`. Create a `TwoLayerNet` from the reference file with input size 2, hidden size 4 and a seed, and a batch of the four XOR points with labels. Compute the NumPy forward and backward, then convert the parameters with `params_from_numpy` and compute the autograd loss. Assert that every gradient matches the NumPy gradient with `np.allclose` at an absolute tolerance of 1e-10.
3. Write `tests/test_accumulation.py` with two tests. The first checks that two backward passes on the same loss double the gradient. The second checks that zeroing the gradient with `grad.zero_()` restores the single-pass value.
4. Write `tests/test_control.py`. Test that `freeze` makes a parameter's `requires_grad` False and that a frozen parameter's `.grad` stays `None` after backward. Test that `grad_norm` matches a hand-computed norm for a tensor of known values.
5. Write `experiments/compare_gradients.py`. Print a table with each parameter name, the maximum absolute difference between the NumPy and autograd gradients, and the shape. The table should show differences near 1e-15.
6. Add one experiment that trains the autograd network for 500 steps with a `torch.no_grad` validation pass every 50 steps, and prints the validation loss. Record whether memory use changes with the number of validation passes, if you can measure it on your machine, or say that you could not.
7. Write the README section that lists three surprises from the project, such as gradient accumulation, the freezing behavior or the float64 conversion.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the comparison
- [ ] The comparison table, pasted from the experiment output
- [ ] The three surprises, each with the test or line of code that shows it
- [ ] A note on where float64 was used and why
- [ ] A short comparison of the hand-written and autograd code length, in lines

**Done when:**

- The autograd gradients match the hand-written ones within 1e-10 for every parameter
- The accumulation test and the control tests pass
- The comparison table shows that the largest difference is below 1e-10
- The README explains each of the three surprises in plain words
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the README with the comparison table and the surprises, and the output of the experiment in `results/compare_gradients.txt`. The comparison output is the strongest evidence that you can verify a framework's gradients against a reference you wrote yourself.

**Why this project matters:** a framework that produces gradients you cannot check is a black box. Being able to show that a gradient matches a reference is the habit that lets you trust custom layers and debug the ones you did not write.

---

## Check your understanding

1. Why does PyTorch accumulate gradients by default instead of replacing them, and what design does that make possible?
2. After `loss.backward()`, a colleague calls `loss.backward()` again inside the same training step and sees that the gradients doubled. Explain why, and say what the correct pattern is.
3. You compute `z = y.detach() * x` with y = 3x. What is the gradient of z with respect to x, and why is it different from the gradient of `y * x`, where y is still tracked?
4. A validation loop runs inside the training loop without `torch.no_grad`, and the GPU runs out of memory after several epochs, though training alone fit. Explain the cause.
5. You freeze the first layers of a pretrained model and the training loss does not change at all. List two causes, one of them in how the optimizer was built.
6. Why does `clip_grad_norm_` return the norm before clipping, and how would you use that value in a training run?

**Answers:**

1. Accumulation lets you run several micro-batches before one update, which simulates a larger batch on limited memory. It also lets you add gradients from several losses before one step. The cost is that you must clear the gradients yourself at the right time.
2. The first call builds and frees the graph, and the second call adds the same gradients again to `.grad`, so the values double. The correct pattern is one `zero_grad` at the start of each step, one forward pass, one loss, one backward call and one optimizer step.
3. The gradient is 3x, which equals the detached value of y. Detaching treats y as a constant, so only the direct factor x contributes its derivative. Without the detach, `y * x` equals 3x squared, whose derivative is 6x, because the path through y adds the second term 3x.
4. Each validation forward pass without `no_grad` records a graph that is kept alive until the references are dropped, and the graphs accumulate in memory. The training pass alone is freed after each backward, but the validation graphs are not needed and stay in memory. The fix is to wrap validation in `torch.no_grad()` and to switch the model to eval mode.
5. The optimizer may have been built from a list that contains only frozen parameters, or from no parameters at all, so no update reaches the layers that should learn. The second cause is that the trainable layers are cut off from the loss, for example by a detach or a no-grad block, so their gradients stay `None`. A check is to print the number of trainable parameters and the gradient norm of each one after a single backward pass.
6. The norm before clipping shows how large the raw gradient was. Logging it over time reveals instability, such as spikes, and it tells you how often clipping changed the update. A frequent clip with a high norm is a sign that the learning rate is too large.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Automatic differentiation", for the forward and reverse modes
- Wikipedia, "Backpropagation", for the algorithm you wrote by hand on Day 43
- Wikipedia, "Gradient clipping", for the rescaling method and its purpose
- PyTorch documentation, "Autograd mechanics", at pytorch.org/docs/stable/notes/autograd.html, which describes the graph, the leaf tensors and the rules you met today
- PyTorch documentation, "torch.autograd", at pytorch.org/docs/stable, for `backward`, `grad` and the context managers

**Tutorials**

- PyTorch Tutorials, "Automatic Differentiation with torch.autograd", at pytorch.org/tutorials/beginner/basics/autogradqs_tutorial.html
- PyTorch Tutorials, "Learn the Basics", at pytorch.org/tutorials/beginner/basics/intro.html, which includes a short autograd section
- PyTorch documentation, "torch.nn.utils.clip_grad_norm_", at pytorch.org, for the exact return value

**Watch**

- Andrej Karpathy, "The spelled-out intro to neural networks and backpropagation: building micrograd", on YouTube, which builds the same kind of engine you wrote on Day 43
- The PyTorch official channel on YouTube, for talks on autograd and the graph

**Practice**

- Andrej Karpathy's micrograd repository at github.com/karpathy/micrograd, which you can use to compare its engine with PyTorch's on the same expressions

---

## Tomorrow

Day 48 uses autograd to build real models. You will write classes that subclass `nn.Module`, define layers in the constructor and the computation in `forward`, and connect a loss and an optimizer. The parameters you tracked today are the ones `nn.Module` registers automatically, and the gradient checks you wrote will apply to any model you define. The two-layer network from Day 43 becomes about twenty lines of framework code.

# Day 43: Backpropagation

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 4 hours
**Level:** intermediate to advanced. This is the hardest day of the module so far. You need the chain rule from Day 11 and the forward pass and losses from Days 39 to 42. Take the scalar example slowly before you move to matrices.

**By the end of this day, you will have:**

- A scalar automatic differentiation engine of about 60 lines, built on a computational graph, that computes gradients by reverse accumulation
- A NumPy implementation of forward and backward passes for a two-layer network with tanh hidden units and a sigmoid output, written by hand
- A gradient check that compares every analytic gradient with a finite-difference estimate
- A trained network that solves XOR, along with a record of which random seeds converged and which did not

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the loss curve
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 11: the derivative and the chain rule for a composition of two functions
- Day 10: matrix multiplication and transposes
- Day 39: the forward pass of a dense layer
- Day 41: the derivatives of sigmoid and tanh
- Day 42: binary cross-entropy and its gradient with respect to the logit

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Computational graphs](#computational-graphs)
- [Chain rule](#chain-rule)
- [Forward pass](#forward-pass)
- [Loss](#loss)
- [Gradients](#gradients)
- [Backward pass](#backward-pass)
- [Parameter updates](#parameter-updates)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a tiny autograd engine and a hand-written network](#project-a-tiny-autograd-engine-and-a-hand-written-network)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Training a neural network means changing its weights so that the loss goes down. To know which way to change each weight, you need its gradient: how much the loss would change if that weight moved a little. A network can have millions of weights, so computing each gradient separately would be hopeless. Backpropagation is the method that computes all of them in one backward sweep, at a cost about the same as one forward pass.

The method is the chain rule applied in the right order, with each intermediate value remembered from the forward pass. Once you see the pattern on a graph with three nodes, you can apply it to a network with a hundred layers. That is why this lesson starts with a scalar example and only then moves to matrices.

On the job, you will rarely write backpropagation by hand. Frameworks such as PyTorch build the graph for you and run the backward pass automatically. Still, the engineers who debug exploding gradients, frozen layers, NaN losses and custom operations are the ones who understand what the framework does underneath. You will use that understanding on Days 46 to 50.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1960s | Henry J. Kelley, Arthur E. Bryson and others describe the backward recursion for optimal control problems. | The first use of the reverse-mode idea behind backpropagation. |
| 1970 | Seppo Linnainmaa publishes the reverse mode of automatic differentiation in his master's thesis. | The general algorithm for computing gradients of any computational graph. |
| 1974 | Paul Werbos describes applying the method to multi-layer neural networks in his doctoral thesis. | The first clear link between the algorithm and neural networks. |
| 1986 | Rumelhart, Hinton and Williams publish the paper that brings backpropagation to wide attention. | The method that made deep networks trainable in practice. |
| 2015 onward | Frameworks such as Theano, TensorFlow and PyTorch implement reverse-mode automatic differentiation as a core feature. | You now rarely write the backward pass by hand. |

**Takeaway:** the idea is older than most of the tools that use it. The modern contribution is the software that builds the graph for you.

---

## Glossary

- **Computational graph**: a diagram in which each node is a value or an operation, and each edge shows that one value is an input to another.
- **Node**: a value in the graph, such as an input, a parameter or an intermediate result.
- **Operation**: a function that takes one or more nodes and produces a new node, such as addition, multiplication or tanh.
- **Forward pass**: computing every node from the inputs, in order, and storing the values.
- **Backward pass**: computing the gradient of the loss with respect to every node, in reverse order, using the chain rule.
- **Gradient**: the vector of partial derivatives of the loss with respect to a set of parameters.
- **Chain rule**: the rule that the derivative of a composition is the product of the derivatives along the path.
- **Reverse-mode automatic differentiation**: a method that computes the gradient of one output with respect to many inputs in one backward sweep.
- **Learning rate**: the factor that scales each parameter update.
- **Gradient descent**: the update rule that moves each parameter a small step opposite to its gradient.
- **Gradient check**: a comparison between an analytic gradient and a finite-difference estimate, used to find bugs.
- **Vanishing gradient**: a gradient that becomes so small that the early layers stop changing.

---

## Computational graphs

Picture a recipe written as a flowchart. Each step takes some ingredients, produces an intermediate dish, and passes it on. If someone tells you the final dish tastes too salty, you can trace back along the flowchart to see which step added the salt. A computational graph is that flowchart for arithmetic.

From first principles, take the expression L = (a + b) * c. You can split it into two steps: d = a + b, then L = d * c. The graph has nodes a, b, c, d and L. Edges run from a and b into d, and from d and c into L. Evaluating the graph from the inputs to the output is the forward pass. Tracing it in the other direction is the backward pass, and it is where the gradients come from. Frameworks record each operation as it happens, so the graph is built during the forward pass without you writing it down.

```
  a ---+
       (+) --> d ---+
  b ---+            (*) --> L
                    |
  c ----------------+
```

The limitation is that the graph is only as good as the operations it contains. A custom operation that does not define its backward rule breaks the chain, and the framework will either raise an error or treat the output as a constant. In practice, this is one of the most common causes of gradients that are silently zero.

Do not confuse the graph with the network diagram you draw on a whiteboard. The network diagram shows layers. The computational graph shows every arithmetic operation, including the transposes, additions and activations, and its size grows with the number of operations per example and per batch. On the job, a graph that is too large often shows up as a memory problem, not a speed problem, because every intermediate value is kept for the backward pass.

---

## Chain rule

Imagine a relay race where the baton passes through several runners. The speed of the baton at the finish depends on how each runner changes its speed. The chain rule says the same thing about functions: if y depends on u and u depends on x, the rate of change of y with respect to x is the rate of change of y with respect to u, times the rate of change of u with respect to x.

From first principles, if y = f(u) and u = g(x), then dy/dx = (dy/du) (du/dx). For a longer chain, multiply all the local derivatives along the path from the output back to the input. Each operation in the graph contributes one local derivative that depends only on its own inputs and output. The backward pass multiplies the incoming gradient by the local derivative at each node.

Here is the example from the first section, worked by hand. Let d = a + b and L = d c. Then dL/dd = c, dL/dc = d, dd/da = 1 and dd/db = 1. So dL/da = dL/dd times dd/da = c, and dL/db = c as well. With a = 2, b = 3 and c = 4, we get d = 5, L = 20, dL/da = 4 and dL/db = 4. Changing a by a tiny amount changes L by four times that amount, which you can check with a finite difference.

```python
a, b, c = 2.0, 3.0, 4.0
d = a + b
L = d * c

dL_dd = c
dL_da = dL_dd * 1.0
dL_db = dL_dd * 1.0
dL_dc = d
print(L, dL_da, dL_db, dL_dc)        # 20.0 4.0 4.0 5.0
```

Do not multiply local derivatives in the wrong order or forget one path. When a value feeds into two operations, its gradient is the sum of the gradients from both paths. Forgetting to add them is a common bug, and a Value class that uses `+=` for gradients avoids it.

In the real market, the chain rule is the mathematics behind every framework's autograd system. Knowing it is what lets you reason about a custom loss, a new layer or a numerical instability in a production model.

---

## Forward pass

A forward pass is the work of computing every node from the inputs. For a dense layer, it is the matrix product and the bias. For an activation, it is the nonlinear function. The forward pass is the easy part, and it has one extra job during training: it must store the intermediate values, because the backward pass uses them. The derivative of tanh, for example, is 1 - tanh(z)^2, so you must keep tanh(z) or z from the forward pass.

From first principles, a two-layer network with input X, hidden pre-activation Z1, hidden activation A1, output pre-activation Z2 and output A2 computes:

```
Z1 = X W1 + b1
A1 = tanh(Z1)
Z2 = A1 W2 + b2
A2 = sigmoid(Z2)
```

Each line is a node in the graph, and the matrices W1, b1, W2 and b2 are parameters. The output A2 is a probability, one per example. You store X, Z1, A1, Z2 and A2 for the backward pass.

The common mistake is to forget that the forward pass must be run in the training mode of the network, and to reuse a cached value from an earlier batch. Stale caches cause wrong gradients that are hard to find. In the real market, memory use during training is mostly the memory of these stored values, which is why engineers reduce batch size or use gradient checkpointing when a model does not fit.

---

## Loss

The loss compresses the output into one number. For a batch of N examples with labels y and probabilities A2, the binary cross-entropy from Day 42 is L = -(1/N) sum of (y log A2 + (1 - y) log(1 - A2)). The loss is a scalar, which is important: backpropagation computes the gradient of one number with respect to many parameters, and that is the efficient direction.

From first principles, the loss is the last node in the graph. Its gradient with respect to the output A2 is the first thing the backward pass computes. For binary cross-entropy, that gradient is (A2 - y) / (A2 (1 - A2)) divided by N. The backward pass then passes through the sigmoid, which multiplies by A2 (1 - A2), and the product simplifies to (A2 - y) / N. Starting from the logit gives the simpler form, which you will use in the code.

The limitation is that the loss is a proxy for what you want. The gradient is correct for the loss you chose, not necessarily for the business outcome. If the loss averages over examples, a rare class may contribute very little to the gradient. This is why the loss design on Day 42 matters for the backward pass too.

Do not compute the loss from a probability that has been clipped to exactly 0 or 1, because the gradient then becomes infinite or undefined. The fix is to clip with a small epsilon, or to compute the loss from logits, as Day 42 shows.

---

## Gradients

A gradient is a table of partial derivatives, one for each parameter. The partial derivative of L with respect to W1[i, j] tells you how the loss changes if that single weight increases. Collect all of them with the same shape as the parameter, and you have the gradient of that parameter. For a matrix W1 of shape (2, 4), the gradient dW1 also has shape (2, 4).

From first principles, the gradients of the two-layer network are obtained by applying the chain rule to each node, in reverse order. Write dZ2 for the gradient of L with respect to Z2. Then dW2 = A1^T dZ2, because each weight in W2 multiplies one hidden activation. The bias gradient db2 is the sum of dZ2 over the batch, because the bias is added to every example. Similarly, dA1 = dZ2 W2^T, dZ1 = dA1 times (1 - A1^2), dW1 = X^T dZ1, and db1 is the sum of dZ1 over the batch.

Each of these formulas has a shape check. If dW1 does not have the same shape as W1, something is wrong. Shape checks catch a large share of backpropagation bugs before any numbers are compared.

The limitation of an analytic gradient is that it can be wrong in a way that still looks plausible. A missing transpose, a forgotten sum over the batch or a sign error can produce gradients that move the loss in the right direction on some examples and the wrong direction on others. That is why the project checks every gradient against a finite difference.

Do not sum over the batch along the wrong axis. For a weight matrix, you sum over examples through the matrix product, but for a bias vector, you sum the batch dimension explicitly. In the real market, gradient checks are a standard step when someone writes a custom layer in a production framework, and the same finite-difference idea is used in unit tests for custom operations.

---

## Backward pass

The backward pass is the order in which the gradients are computed. It starts at the loss, moves to the output layer, then to the hidden layer, and it reuses the forward values at each step. Each node receives the gradient from its downstream nodes, multiplies by its local derivative, and passes the result to its inputs. When a node feeds several operations, the gradients from those operations are added.

From first principles, the procedure is the following. Compute dZ2 from the loss. Use it to compute the gradients for W2 and b2. Pass it back through W2 to get dA1. Multiply by the derivative of tanh at Z1 to get dZ1. Use dZ1 to compute the gradients for W1 and b1. Each step uses one matrix product or one elementwise product, which is why the backward pass costs about as much as the forward pass.

```python
import numpy as np

def backward(X, y, cache, params):
    W2 = params["W2"]
    A1, A2 = cache["A1"], cache["A2"]
    N = X.shape[0]

    dZ2 = (A2 - y) / N                       # gradient at the output logit
    dW2 = A1.T @ dZ2
    db2 = dZ2.sum(axis=0)
    dA1 = dZ2 @ W2.T
    dZ1 = dA1 * (1.0 - A1 ** 2)              # tanh derivative from the forward pass
    dW1 = X.T @ dZ1
    db1 = dZ1.sum(axis=0)
    return {"W1": dW1, "b1": db1, "W2": dW2, "b2": db2}
```

The limitation is that the backward pass needs the forward values, so memory grows with depth and batch size. Techniques such as recomputing activations during the backward pass trade compute for memory. Do not call the backward pass twice on the same graph without zeroing the old gradients. Frameworks keep gradients by accumulation, so you must reset them before each batch. Forgetting that is one of the most frequent bugs in early training loops, and you will write the reset line on Day 50.

---

## Parameter updates

Once you have the gradients, the update is simple. Move each parameter a small step in the direction that decreases the loss: W = W - lr * dW. The learning rate lr controls the size of the step. Too small and training is slow, too large and the loss can oscillate or diverge.

From first principles, gradient descent follows the negative gradient because the gradient points in the direction of steepest increase. A small step in the opposite direction decreases the loss to first order. The update is applied to every parameter in the network after each batch. Richer optimizers, such as momentum and Adam, change the direction and scale of the step, and you will study them on Day 44.

Here is the update applied to a dictionary of parameters:

```python
def update(params, grads, lr):
    for name in params:
        params[name] -= lr * grads[name]
    return params
```

The limitation is that gradient descent only sees the local slope. It can get stuck in saddle points or flat regions, and it can overshoot narrow valleys. A learning rate that works at the start of training may be too large later, which is why learning-rate schedules exist.

Do not update a parameter in place while it is still needed for the backward pass. If the forward cache refers to the same array as the parameter, an in-place update corrupts the cache. Copy the values when they are needed later, or structure the loop so that updates happen after the backward pass is complete. In industry, the same update step is run on many machines with distributed training, and the update must be consistent across them.

---

## Hands-on exercises

1. **Scalar graph by hand.** For L = (a + b) c with a = 1, b = -2 and c = 3, compute the forward values and the four gradients on paper. Then check your numbers in Python.
2. **Shared input.** For L = a * a + a with a = 2, compute dL/da by hand, using the fact that a feeds two operations. Explain why a naive backward pass that does not accumulate gradients would give the wrong answer.
3. **Shape audit.** For a network with input size 5, hidden size 3 and output size 1, write the shape of every intermediate value in the forward pass and of every gradient in the backward pass. Check your table against a run of your code.
4. **Finite difference by hand.** For f(x) = x^3 at x = 2, compute the central difference with h = 0.001 and compare it with the derivative 3x^2 = 12. Describe how the error changes when h is made smaller.
5. **Learning-rate sweep.** Train the XOR network with learning rates 0.01, 0.1, 1.0 and 10.0 for 2,000 epochs each, and record the final loss. Describe what you see for each learning rate.

---

## Project: a tiny autograd engine and a hand-written network

**Goal:** build a scalar autograd engine that builds a graph and runs the backward pass, then write the backward pass of a two-layer network by hand. Check every gradient against a finite difference, train the network on XOR, and document which seeds converge.

**Repo name:** `100-days-ai-day-43-backpropagation`

**Repo layout:**

```
100-days-ai-day-43-backpropagation/
├── README.md
├── requirements.txt
├── src/
│   └── backprop_lab/
│       ├── __init__.py
│       ├── value.py
│       ├── network.py
│       └── gradcheck.py
├── tests/
│   ├── test_value.py
│   └── test_network.py
└── experiments/
    └── xor_seeds.py
```

**Starter code.** Create `requirements.txt` with the pinned packages from the requirements section. Create `src/backprop_lab/value.py` with the scalar engine below.

```python
import math


class Value:
    def __init__(self, data, _children=(), _op=""):
        self.data = float(data)
        self.grad = 0.0
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op

    def __add__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data + other.data, (self, other), "+")

        def _backward():
            self.grad += out.grad
            other.grad += out.grad

        out._backward = _backward
        return out

    def __mul__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data * other.data, (self, other), "*")

        def _backward():
            self.grad += other.data * out.grad
            other.grad += self.data * out.grad

        out._backward = _backward
        return out

    def tanh(self):
        t = math.tanh(self.data)
        out = Value(t, (self,), "tanh")

        def _backward():
            self.grad += (1.0 - t * t) * out.grad

        out._backward = _backward
        return out

    def backward(self):
        topo, visited = [], set()

        def build(v):
            if v not in visited:
                visited.add(v)
                for child in v._prev:
                    build(child)
                topo.append(v)

        build(self)
        self.grad = 1.0
        for v in reversed(topo):
            v._backward()

    __radd__ = __add__
    __rmul__ = __mul__

    def __neg__(self):
        return self * -1.0

    def __sub__(self, other):
        return self + (-other if isinstance(other, Value) else Value(-other))
```

**Step-by-step tasks:**

1. Create the repo, initialize Git, create a virtual environment and install the pinned packages.
2. Write `tests/test_value.py` with three tests. First, `c = a * b + a` with a = 2 and b = -3 has `c.data == -4`, `a.grad == -2` and `b.grad == 2`. Second, a value used twice, `L = a * a + a` at a = 2, gives `a.grad == 5`. Third, a tanh node has the gradient 1 - tanh^2 at its input.
3. Write `src/backprop_lab/network.py` with a `TwoLayerNet` class. Its constructor takes an input size, a hidden size, a seed and a learning rate. It stores W1 with shape (input, hidden), b1, W2 with shape (hidden, 1) and b2, all as float64 arrays. Use `np.random.default_rng(seed)` and scale the weights by 0.5. Implement `forward(X)` that returns the predictions and a cache, and `backward(X, y, cache)` that returns a dictionary of gradients, using the formulas in the lesson.
4. Write `src/backprop_lab/gradcheck.py` with a function `numerical_gradient(loss_fn, param, h=1e-5)` that perturbs each entry of the parameter array in place, restores it and returns the finite-difference gradient with the same shape. Use float64 arrays and a small network, such as input 2, hidden 3 and four XOR examples.
5. Write `tests/test_network.py` to assert that each analytic gradient matches the numerical gradient within a relative error of 1e-5. Use the relative error `abs(a - n) / max(1e-8, abs(a) + abs(n))`.
6. Write `experiments/xor_seeds.py` that trains the network on XOR for 5,000 full-batch epochs for seeds 0 to 9, with learning rate 0.5. Record the final loss and the accuracy for each seed in a table, and save the loss curve for seed 0 to `results/xor_loss.png`.
7. Fill in the README with the seed table. Say which seeds converged, and say what you would try for a seed that did not.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the seed experiment
- [ ] The scalar graph example, with the gradients you computed by hand
- [ ] The gradient check output, with the largest relative error
- [ ] The seed table, with the final loss and accuracy for each seed
- [ ] A paragraph on what you would change if a seed did not converge

**Done when:**

- The scalar tests pass, and the shared-input test fails if you remove the `+=` accumulation and use `=` instead
- Every gradient in the network matches the numerical gradient within the tolerance
- The seed table is in the README, and every number in it comes from your run
- You can explain, without notes, why the gradient of a shared input is a sum
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the README with the gradient check and the seed table, the loss curve image, and a `NOTES.md` that records one bug you hit in the backward pass and the gradient check that found it. A convergent run with no bug is less useful evidence than one that shows the check working.

**Why this project matters:** a network that trains without a verified backward pass is a network you cannot trust. The gradient check is the habit that protects you when you later write custom layers in a framework.

---

## Check your understanding

1. In the expression L = a * a + a, why must the gradient of a be the sum of two contributions, and what would go wrong if the engine overwrote the gradient instead?
2. Explain why the backward pass costs about as much as the forward pass, in terms of the number of matrix products.
3. The gradient of binary cross-entropy with respect to the logit is (A2 - y) / N. Show how this follows from the chain rule through the sigmoid.
4. A teammate computes db1 by summing dZ1 over the features instead of the examples. What shape would the result have, and how would you catch the error before training?
5. You run a training loop and the loss becomes NaN after ten steps. Name two causes that the backward pass could produce and one check you would run first.
6. Why does a gradient check use a central difference rather than a one-sided difference, and what tolerance would you expect from float64 with h = 1e-5?

**Answers:**

1. Changing a changes both the product and the sum, so the total change in L is the sum of the two paths. The engine must add the two contributions. If it overwrote the gradient, only the last path would be kept, and the result would be wrong, with the gradient of a equal to one path alone.
2. Each layer needs one matrix product for the forward pass, and the backward pass needs about two matrix products per layer, one for the weight gradient and one for the gradient passed to the previous layer. So the cost is a small constant multiple of the forward cost, with the same order of magnitude.
3. With A2 = sigmoid(Z2), the derivative dA2/dZ2 is A2 (1 - A2). The gradient of the loss with respect to A2 is (A2 - y) / (A2 (1 - A2)) divided by N. Multiplying the two gives (A2 - y) / N.
4. The sum over features gives a vector of shape (N,) instead of (H,), which does not match the bias. Broadcasting may hide the error in some frameworks, so the shape check and a gradient check are the safe ways to catch it.
5. A large learning rate can make the parameters grow until the forward pass overflows, and a log of zero in the loss can produce NaN. A check to run first is the loss at each step with the learning rate lowered, and whether the clipped probabilities stay inside the epsilon range.
6. A central difference has error of order h^2, while a one-sided difference has error of order h. For float64 and h = 1e-5, the central difference error is small enough to detect a sign error or a missing term. A relative error near 1e-7 to 1e-9 is typical for a correct gradient, so 1e-5 is a conservative tolerance.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Backpropagation", for the algorithm and its history
- Wikipedia, "Chain rule", for the rule in its general form
- Wikipedia, "Automatic differentiation", for the forward and reverse modes
- Wikipedia, "Computational graph", for the graph view of a function
- Wikipedia, "Gradient descent", for the update rule and its variants
- Stanford CS231n course notes, "Backpropagation, Intuitions", at cs231n.github.io, which walks through a graph with gates

**Tutorials**

- PyTorch documentation, "Autograd: Automatic Differentiation", at pytorch.org/tutorials, which shows the same ideas in the framework you will use from Day 46
- Andrej Karpathy, "micrograd", the repository at github.com/karpathy/micrograd, which contains a scalar autograd engine in under one hundred lines
- NumPy documentation, "numpy.ndarray.sum", at numpy.org/doc/stable, for the axis behavior used in the bias gradients

**Watch**

- Andrej Karpathy, "The spelled-out intro to neural networks and backpropagation: building micrograd", on YouTube
- 3Blue1Brown, "What is backpropagation really doing? | Deep learning chapter 3", on YouTube
- 3Blue1Brown, "Backpropagation calculus | Deep learning chapter 4", on YouTube

**Practice**

- Build the micrograd example from Karpathy's video in a notebook, then extend the Value class with a power operation and check its gradient with the same tests you wrote today.

---

## Tomorrow

Day 44 keeps the gradients you can now compute and asks how to use them well. Plain gradient descent on the full dataset is slow and can be unstable, so you will compare batch gradient descent, stochastic gradient descent and mini-batches, then add momentum and Adam, and study learning-rate schedules. The training loop you build on Day 50 will call the backward pass you wrote today, and the optimizer you choose tomorrow decides how far each update moves the weights.

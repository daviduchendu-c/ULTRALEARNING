# Day 48: Building PyTorch Models

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** intermediate. You know tensors (Day 46) and autograd (Day 47). Today you package a network the way production code does: as a class that owns its layers, describes its computation in one method and can be saved, loaded and tested.

**By the end of this day, you will have:**

- A configurable multi-layer perceptron written as a subclass of `nn.Module`, with its layers registered and its forward method defined
- A training function that uses `nn.BCEWithLogitsLoss` and `torch.optim.Adam` correctly with the model's output logits
- Tests that check output shapes, parameter counts, train and eval behavior, and a save and load round trip
- A short comparison of a deep narrow network and a shallow wide network with about the same parameter count, with the numbers from your run

**What you need:**

- A computer with Python 3.11 or 3.12
- PyTorch 2.5.1 from Day 46, with the same environment
- pytest 8.3.3, as before
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the loss plot
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 39: the dense layer and its forward pass
- Day 42: binary cross-entropy and why it is paired with logits
- Day 45: dropout, batch normalization and the eval mode
- Day 46 and Day 47: tensors, devices and autograd

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [nn.Module](#nnmodule)
- [Layers](#layers)
- [Forward methods](#forward-methods)
- [Losses](#losses)
- [Optimizers](#optimizers)
- [Model architecture](#model-architecture)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a configurable MLP in PyTorch](#project-a-configurable-mlp-in-pytorch)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A model in PyTorch is a Python object. It holds its layers as attributes, it knows its parameters, it runs the computation in one method, and it can be moved to a device, switched between training and evaluation, saved to a file and loaded back. This structure is what turns a set of equations into code that other people can read, test and deploy.

The base class that makes this possible is `torch.nn.Module`. You subclass it, create the layers in `__init__`, and write the computation in `forward`. PyTorch then finds the parameters of every layer automatically, so the optimizer can receive `model.parameters()` and update them all. Most model code you will read in a job follows this pattern, from a small classifier to a large transformer.

On the job, the way a model is written determines how easy it is to test, change and ship. A model with hard-coded sizes cannot be reused for a new dataset. A model without a saved state cannot be reproduced. A model whose forward method hides a bug in its shapes will fail only in production. The habits of today, such as configurable sizes and explicit shape tests, prevent those failures.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1980s | Frameworks such as the Lisp-based and C-based neural network toolkits describe networks as data structures and code, with explicit layer objects. | The early idea of a network as a composable object. |
| 2002 | The Torch library, written in Lua, provides neural network modules with forward and backward methods. | The ancestor of the module design in PyTorch. |
| 2015 | Keras popularizes the `Sequential` and layer-object style in Python. | A high-level way to describe a model as a list of layers. |
| 2016 | Research frameworks such as Chainer describe models as Python classes whose forward method runs on each call. | The design you use today in `nn.Module`. |
| 2017 | PyTorch releases the `nn.Module` interface for training with autograd, with `state_dict` for saving and loading. | The pattern that makes models saveable and reproducible. |

**Takeaway:** the module pattern was an answer to the problem of describing networks that change shape and size. It is now the standard in research and production.

---

## Glossary

- **Module**: the base class `torch.nn.Module` for every layer and model in PyTorch.
- **Parameter**: a tensor that the module registers as learnable, such as a weight or a bias.
- **Buffer**: a tensor that belongs to a module but is not learned, such as the running mean of a batch normalization layer.
- **Forward method**: the method that defines the computation from input to output, called when you write `model(x)`.
- **Logits**: the raw outputs of the last layer, before a sigmoid or softmax converts them to probabilities.
- **State dict**: a dictionary that maps each parameter and buffer name to its tensor, used for saving and loading.
- **Sequential**: a container that runs its child modules one after another.
- **Architecture**: the arrangement of layers, their sizes and the connections between them.
- **Parameter count**: the total number of learnable numbers in a model, the most common size measure.
- **Train mode and eval mode**: the two states of a module that change dropout and batch normalization behavior, switched by `train()` and `eval()`.
- **Device placement**: the device a module's parameters live on, changed with `.to(device)`.

---

## nn.Module

Think of a model as a kit of parts with a manual. The parts are the layers, and the manual is the forward method. The kit knows which parts it contains, so you can label them, count them, paint them or pack them away as one unit. `nn.Module` is the box that holds the kit.

From first principles, when you assign a module or a parameter to an attribute of another module, PyTorch registers it in an internal dictionary. The method `parameters()` walks that structure and yields every parameter, including those of nested modules. Calling the module as a function, such as `model(x)`, runs the `__call__` method, which calls `forward` and runs any registered hooks. You should always call `model(x)`, not `model.forward(x)`, so that hooks run.

```python
import torch
import torch.nn as nn

class Tiny(nn.Module):
    def __init__(self):
        super().__init__()
        self.layer = nn.Linear(2, 1)

    def forward(self, x):
        return self.layer(x)

m = Tiny()
print([name for name, _ in m.named_parameters()])   # ['layer.weight', 'layer.bias']
```

The limitation of the module pattern is that a plain Python list of layers is not registered. If you store layers in a list and forget `nn.ModuleList`, their parameters are invisible to the optimizer, and the model silently fails to learn. The same applies to dictionaries, which need `nn.ModuleDict`.

Do not forget `super().__init__()` at the start of `__init__`. Without it, the module cannot register its attributes, and the error message is confusing. Make it the first line of every module, every time.

In the real market, a model class is the unit that a team reviews, tests and versions. The module boundary is where teams put shape assertions, documentation and configuration.

---

## Layers

Picture a set of standard building blocks, each with a label that says what it does. A dense layer, a dropout layer and an activation are the blocks you will use most often. PyTorch provides each one as a module, and you combine them in the constructor.

From first principles, `nn.Linear(in_features, out_features)` computes X W^T + b, with weights of shape (out, in). Note the transpose: PyTorch stores the weight in the output-by-input layout, which differs from the input-by-output layout of your NumPy code on Day 39. `nn.Tanh`, `nn.ReLU` and `nn.Sigmoid` are activation modules, and `nn.Dropout(p)` and `nn.BatchNorm1d(num_features)` are the regularizers from Day 45. Each layer creates its own parameters with a default initialization, which you can change if you need to.

```python
import torch
import torch.nn as nn

layer = nn.Linear(4, 3)
print(layer.weight.shape)       # torch.Size([3, 4]): out by in
print(layer.bias.shape)         # torch.Size([3])
print(sum(p.numel() for p in layer.parameters()))   # 15 = 4*3 + 3
```

The limitation of the default initialization is that it is chosen for typical activations, and a deep network with unusual activations may need a different scheme. The standard alternatives, such as Kaiming initialization for ReLU, are available in `torch.nn.init`, and you should choose them on purpose when a model trains poorly.

Do not confuse the weight shape with the NumPy layout. A common bug is to write `x @ layer.weight` and get a shape error, when the correct call is `layer(x)` or `x @ layer.weight.T`. Use the module call, which handles the transpose, and reserve direct matrix products for custom operations.

In industry, the standard layer types are shared across teams, and reviewers look for the layers that matter for the task: normalization, dropout, the activation choice and the output head.

---

## Forward methods

The forward method is the recipe. It takes the input tensor, passes it through the layers in order and returns the output. It is ordinary Python, so you can use conditionals, loops and helper functions inside it, as long as every operation is a differentiable tensor operation.

From first principles, the forward method runs once for each call of the module, and the autograd graph from Day 47 is built during that call. Because the graph is rebuilt on each call, the forward method can change its behavior between calls, for example by skipping a layer when a flag is set. It must return the tensor that the loss expects, and for a binary classifier that tensor should be the logit, not a probability.

```python
import torch
import torch.nn as nn

class TwoLayerNet(nn.Module):
    def __init__(self, n_in, n_hidden, n_out=1):
        super().__init__()
        self.hidden = nn.Linear(n_in, n_hidden)
        self.act = nn.Tanh()
        self.out = nn.Linear(n_hidden, n_out)

    def forward(self, x):
        return self.out(self.act(self.hidden(x)))   # logits, shape (batch, n_out)

net = TwoLayerNet(2, 8)
logits = net(torch.randn(5, 2))
print(logits.shape)              # torch.Size([5, 1])
```

The limitation of a forward method written in Python is that it runs slowly if it contains many small operations and Python loops over tensor elements. Compilation tools such as `torch.compile` can speed up the forward pass, but they require the code to be written in a compilable style, and not every model benefits.

Do not apply a sigmoid in the forward method and then pass the result to `BCEWithLogitsLoss`, which expects logits. The loss applies the sigmoid internally in a numerically stable way, so the double application makes training slower and less accurate. Keep the output as logits, and apply the sigmoid only at prediction time, outside the loss. In industry, this is one of the most common modeling errors, and it is caught by comparing the predicted probabilities with the training labels on a small batch.

---

## Losses

A loss module is a function that compares the model's output with the target. PyTorch provides loss modules that you create once and call on each batch. For binary classification, `nn.BCEWithLogitsLoss` combines a sigmoid and the binary cross-entropy from Day 42 in one stable operation.

From first principles, the combined form computes the loss from the logit z and the label y with a formula that avoids computing log of a probability near zero. The gradient with respect to the logit is sigmoid(z) minus y, as you showed on Day 42. For multi-class classification, `nn.CrossEntropyLoss` takes logits and integer class labels and applies log softmax internally. For regression, `nn.MSELoss` takes predictions and real targets. The input conventions differ between losses, so read each one's documentation before use.

```python
import torch
import torch.nn as nn

loss_fn = nn.BCEWithLogitsLoss()
logits = torch.tensor([[2.0], [-1.0]])
labels = torch.tensor([[1.0], [0.0]])
print(loss_fn(logits, labels).item())   # about 0.2201
```

The limitation of the loss module is that it is only one part of the objective. Weight decay, a penalty on outputs or a custom term must be added explicitly, and the optimizer's weight decay argument is not the same thing as adding a penalty to the loss for every parameter. Day 45 explained the difference.

Do not pass probabilities to `BCEWithLogitsLoss`, or logits to `BCELoss`. Both run without a clear error when the values fall in range, and the model trains to a wrong answer. Check the name and the expected input of each loss every time, and test with a small example whose loss you can compute by hand.

In the real market, the loss module is usually the only part of training that encodes the business objective, and teams review it with the same care as the model.

---

## Optimizers

An optimizer is an object that holds a reference to the parameters and updates them from their gradients. You construct it once with the list of parameters and the hyperparameters, and you call `step` after each backward pass. The optimizers you studied on Day 44 are available as classes in `torch.optim`.

From first principles, `torch.optim.Adam(model.parameters(), lr=1e-3)` creates an optimizer whose state includes the moment estimates for each parameter, which the update uses. Calling `optimizer.zero_grad()` clears the gradients, `loss.backward()` fills them and `optimizer.step()` applies the update. The order is the same as on Day 47. Learning-rate schedulers in `torch.optim.lr_scheduler` change the rate between steps, and you attach them to the optimizer.

```python
import torch
import torch.nn as nn

model = nn.Linear(2, 1)
optimizer = torch.optim.Adam(model.parameters(), lr=1e-2)
x = torch.randn(16, 2)
y = torch.randint(0, 2, (16, 1)).float()
loss_fn = nn.BCEWithLogitsLoss()

optimizer.zero_grad()
loss = loss_fn(model(x), y)
loss.backward()
optimizer.step()
print(loss.item())
```

The limitation of the optimizer object is that it stores state that must be saved with the model if you want to resume training exactly. Saving only the model's state dict loses the Adam moments, so a resumed run starts with different updates.

Do not create the optimizer before moving the model to a device, or create it on one model and train another. The optimizer holds references to specific tensors, and if the model's parameters are replaced, the optimizer updates the old ones. Move the model first, then create the optimizer from `model.parameters()`. In industry, a silent failure of this kind looks like a model that never learns, and the fix is to check that the optimizer's parameters are the model's parameters.

---

## Model architecture

Architecture is the plan of the model: how many layers, how wide each one is, which activations and regularizers appear, and how the input connects to the output. For a multi-layer perceptron, the architecture is a list of hidden sizes plus a choice of activation. Writing it as a parameter lets you test several architectures with one class.

From first principles, a model with hidden sizes [h1, h2, ...] has one linear layer for each consecutive pair of sizes, plus the input and output layers. The number of parameters of a linear layer with n inputs and m outputs is n m + m. For a model with a fixed budget of parameters, a deeper network has fewer units per layer, and a shallower one has more. The choice affects both how the function is shaped and how easily it trains.

```python
import torch.nn as nn

class ConfigurableMLP(nn.Module):
    def __init__(self, n_in, hidden_sizes, n_out=1, activation=nn.Tanh):
        super().__init__()
        layers, prev = [], n_in
        for h in hidden_sizes:
            layers += [nn.Linear(prev, h), activation()]
            prev = h
        layers.append(nn.Linear(prev, n_out))
        self.net = nn.Sequential(*layers)

    def forward(self, x):
        return self.net(x)
```

The limitation of configurable architectures is that a search over many shapes costs time and can overfit the validation set. Start from a reasonable default, change one thing at a time, and report the number of parameters with every result. A deep model that is much larger than the dataset will overfit, as Day 45 showed.

Do not compare two architectures with different parameter counts and call the larger one better. Compare them at a matched budget, or report both the count and the accuracy, so the comparison is fair. In the real market, architecture decisions are judged against the cost of training and serving, not only the accuracy on a leaderboard.

---

## Hands-on exercises

1. **Parameter count.** For `TwoLayerNet(2, 8)` with one output, compute the number of parameters by hand, then print the sum from `model.parameters()` and compare.
2. **Named parameters.** Print the names of all parameters of a `ConfigurableMLP(2, [16, 8])`. Explain why the names look the way they do.
3. **Loss by hand.** For logit 0 and label 1, compute `BCEWithLogitsLoss` by hand and compare with the module's output. Repeat for logit 3 with label 1 and explain why the loss is smaller.
4. **Forgotten registration.** Replace `nn.Sequential` with a plain Python list of layers, and check the number of parameters the optimizer receives. Explain what happens to training.
5. **Train one step.** Write a function that takes a model, a batch and an optimizer, and performs one training step with the zero, forward, loss, backward, step order. Print the loss before and after the step on the same batch.

---

## Project: a configurable MLP in PyTorch

**Goal:** write a configurable MLP as an `nn.Module`, train it on a synthetic circle dataset with the correct loss and optimizer, add tests for shapes, counts and saving, and compare a deep narrow model with a shallow wide one at a matched parameter count.

**Repo name:** `100-days-ai-day-48-building-pytorch-models`

**Repo layout:**

```
100-days-ai-day-48-building-pytorch-models/
├── README.md
├── requirements.txt
├── src/
│   └── model_lab/
│       ├── __init__.py
│       ├── data.py
│       ├── models.py
│       └── train.py
├── tests/
│   ├── test_models.py
│   └── test_save_load.py
├── experiments/
│   └── compare_architectures.py
└── results/
```

**Starter code.** Create `requirements.txt` with PyTorch 2.5.1, pytest 8.3.3 and Matplotlib 3.9.2 pinned. Create `src/model_lab/data.py`:

```python
import torch


def make_circles(n, seed=0):
    g = torch.Generator().manual_seed(seed)
    X = torch.rand(n, 2, generator=g) * 3.0 - 1.5
    y = ((X ** 2).sum(dim=1, keepdim=True) < 1.0).float()
    return X, y
```

Create `src/model_lab/models.py` with the `ConfigurableMLP` class from the lesson, and add a `count_parameters(model)` function that returns the number of parameters that require gradients.

Create `src/model_lab/train.py`:

```python
import torch
import torch.nn as nn


def train(model, X, y, epochs=300, lr=1e-2, seed=0):
    torch.manual_seed(seed)
    loss_fn = nn.BCEWithLogitsLoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=lr)
    history = []
    for _ in range(epochs):
        model.train()
        optimizer.zero_grad()
        loss = loss_fn(model(X), y)
        loss.backward()
        optimizer.step()
        history.append(loss.item())
    return history


@torch.no_grad()
def accuracy(model, X, y):
    model.eval()
    preds = (torch.sigmoid(model(X)) > 0.5).float()
    return (preds == y).float().mean().item()
```

**Step-by-step tasks:**

1. Create the repo, initialize Git, make a virtual environment and install the pinned packages.
2. Write `tests/test_models.py`. Check the output shape of `ConfigurableMLP(2, [16, 8])` for a batch of five inputs, the parameter count for `ConfigurableMLP(2, [8])` (it should be 2 times 8 plus 8, plus 8 plus 1, which is 33), and that two evaluations in eval mode give identical outputs for a model with dropout added.
3. Write `tests/test_save_load.py`. Save the state dict of a trained model to a temporary file, load it into a fresh model with `weights_only=True`, and assert that the outputs are equal for the same input.
4. Write `experiments/compare_architectures.py`. Train two models on 400 training points from `make_circles` with seed 0, and evaluate on 400 points with seed 1. The deep model is `[8, 8, 8]` and the wide model is one hidden layer with a width chosen so the parameter counts are within 10 percent of each other. Train each for 1,000 epochs with three seeds.
5. Record the parameter count, the mean and spread of the training accuracy, the test accuracy and the final loss for each model in a table in `results/architectures.md`.
6. Save a plot of the training loss for one run of each model to `results/loss_curves.png`.
7. Save the best model's state dict to `results/best_model.pt`, and add a line to the README that shows how to load it.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the comparison
- [ ] The architectures table with parameter counts
- [ ] The loss curves image with one paragraph on the difference
- [ ] The loading snippet for the saved model
- [ ] A statement of which architecture you would choose and why, citing the numbers

**Done when:**

- All model and save tests pass
- The parameter count for `ConfigurableMLP(2, [8])` is 33 as a sum of the two linear layers, and the test checks it
- The architectures table shows the parameter counts within 10 percent of each other
- The best model loads from the saved file and reproduces the test accuracy on the same data
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the README with the architectures table, the loss plot, the saved model file if it is under 5 MB, and a `NOTES.md` that records one place where the shape or the loss convention caught you. The notes show that the module structure and the loss conventions are now yours to check.

**Why this project matters:** a model that is a class with tests, a save path and a documented comparison is the unit of work that a team can review and ship. The habits here apply to every model you build from now on.

---

## Check your understanding

1. Why does a plain Python list of layers inside a module cause a silent training failure, and which container fixes it?
2. A model's forward method ends with a sigmoid, and the loss is `BCEWithLogitsLoss`. What is wrong, and what is the minimal change?
3. Explain why `nn.Linear(4, 3)` stores its weight with shape (3, 4), and how that affects a manual matrix product.
4. Why is it insufficient to save only the model's state dict when you want to resume training exactly?
5. A deep narrow model and a shallow wide model reach the same training loss, but the deep model does better on the test set. Give two possible reasons and one check you would make before claiming the deep model is better.
6. What is the difference between `model.train()` and `torch.no_grad()`, and why does an evaluation loop usually need both?

**Answers:**

1. The optimizer receives parameters only from registered modules, so layers held in a plain list are not in `model.parameters()`. They keep their random initial values, and the loss does not change through them. `nn.ModuleList` registers the layers, so the optimizer sees them.
2. The sigmoid is applied twice: once in the forward method and once inside the loss, which expects logits. The minimal change is to remove the sigmoid from the forward method and apply it only when producing predictions.
3. PyTorch stores the weight as (out features, in features), so `layer(x)` computes x times the transpose of the weight, plus the bias. A manual product must use `x @ layer.weight.T`, or the module call, which handles the transpose.
4. The state dict holds the parameters and buffers but not the optimizer's internal state, such as the Adam moments, or the random number generator state. A resumed run that loads only the state dict starts the optimizer fresh, so its updates differ from the original run.
5. The deep model may have a better inductive bias for the structure of the data, or the shallow model may have a worse optimization path, and the difference could also come from chance on a small test set. A check is to run several seeds for each model and compare the spread of the test accuracy, not a single run.
6. `model.train()` changes the behavior of layers such as dropout and batch normalization, and it does not stop gradient tracking. `torch.no_grad()` stops the graph from being recorded, which saves memory, and it does not change the layer behavior. An evaluation loop needs `model.eval()` for correct layer behavior and `no_grad` for efficiency.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Multilayer perceptron", for the architecture used in the project
- Wikipedia, "Deep learning", for the context of models built from layers
- PyTorch documentation, "torch.nn.Module", at pytorch.org/docs/stable, for the methods and attributes you used today
- PyTorch documentation, "torch.nn" layers page, at pytorch.org/docs/stable, for `Linear`, `Dropout`, `BatchNorm1d` and the loss modules
- PyTorch documentation, "torch.save" and "torch.load", for the `weights_only` argument

**Tutorials**

- PyTorch Tutorials, "Build the Neural Network", at pytorch.org/tutorials/beginner/basics/buildmodel_tutorial.html
- PyTorch Tutorials, "Save and Load the Model", at pytorch.org/tutorials/beginner/basics/saveloadrun_tutorial.html
- PyTorch Tutorials, "Optimizing Model Parameters", at pytorch.org/tutorials/beginner/basics/optimization_tutorial.html

**Watch**

- Andrej Karpathy, "Building makemore Part 2: MLP", from the "Neural Networks: Zero to Hero" playlist on YouTube, which builds a multi-layer perceptron in PyTorch step by step

**Practice**

- The PyTorch "Build the Neural Network" tutorial, with the FashionMNIST example run on your machine, then the same model rewritten as a `ConfigurableMLP` with a different number of hidden layers

---

## Tomorrow

Day 49 deals with the data that feeds the model. You will meet `torch.utils.data.Dataset` and `DataLoader`, learn how batches are formed, shuffled and transformed, and write a custom dataset that reads from files. The model you built today accepts a batch of tensors, and the loader is the component that produces those batches from raw data reliably and efficiently.

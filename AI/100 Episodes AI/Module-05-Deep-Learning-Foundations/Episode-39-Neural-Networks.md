# Day 39: Neural Networks

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3 hours
**Level:** beginner to intermediate. You know Python functions, NumPy arrays and the dot product from Days 8 and 9. You do not need calculus yet, because today is about what a network computes, not how it learns.

**By the end of this day, you will have:**

- A clear mental model of an artificial neuron as a weighted sum followed by a nonlinear function
- A NumPy implementation of one neuron and of a dense layer that processes a whole batch at once
- A tested forward pass that you can read line by line and explain to someone else
- A short note on which parts of a network are learned and which parts are chosen by a human

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the optional plot in the project
- A code editor such as VS Code, which is free
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 8: vectors, the dot product and magnitude
- Day 9: matrices, shapes and matrix multiplication
- Day 10: the idea of a linear transformation
- Basic Python: functions, lists, and running scripts from a terminal

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Biological inspiration versus computation](#biological-inspiration-versus-computation)
- [Artificial neurons](#artificial-neurons)
- [Inputs, weights and biases](#inputs-weights-and-biases)
- [Layers](#layers)
- [Activations](#activations)
- [The forward pass](#the-forward-pass)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a forward pass from scratch with NumPy](#project-a-forward-pass-from-scratch-with-numpy)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Modules 1 to 4 gave you the tools: vectors, matrices, derivatives, probability and statistics. Module 5 uses them to build the model family behind most modern AI, the neural network. Today you learn only the first half of the story, which is what a network computes when you give it an input. Day 43 explains how it learns.

The idea is simple enough to write on one line: a neural network is a stack of weighted sums, each followed by a bend in the function. Stack enough of those and you can approximate surprisingly complicated mappings from images to labels, from sentences to translations, or from sensor readings to motor commands. The complexity comes from repetition and from the shape of the stack, not from any single piece.

On the job, the forward pass is the part that runs in production. Every time a recommendation service scores a product, a speech model turns a chunk of audio into text, or a robot decides whether an object is in its gripper, a forward pass runs. When a model gives a wrong answer at 3 a.m., you debug it by asking what each layer computed, so you need to read a network's forward pass the way you read a function.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1943 | Warren McCulloch and Walter Pitts publish a simple model of a neuron as a threshold logic unit. | The first mathematical neuron, with binary inputs and a firing rule. |
| 1958 | Frank Rosenblatt introduces the perceptron, a neuron whose weights can be learned from examples. | The first learning machine built on the same ideas you study today. |
| 1986 | David Rumelhart, Geoffrey Hinton and Ronald Williams popularize backpropagation for training multi-layer networks. | The training method you will implement on Day 43. |
| 2012 | AlexNet, a deep convolutional network trained on GPUs, wins the ImageNet image classification challenge by a wide margin. | Deep networks became the default approach for perception. |
| 2017 | The transformer architecture is introduced in the paper "Attention Is All You Need". | Most large language models are stacks of layers like the ones you learn today. |

**Takeaway:** the forward pass has barely changed since the 1950s. What changed is the scale of the data, the speed of the hardware and the methods used to train the weights.

---

## Glossary

- **Neural network**: a function built from layers of simple units, where each unit computes a weighted sum of its inputs and applies a nonlinear function.
- **Artificial neuron**: the basic unit of a network, which takes numbers in, combines them with weights and a bias, and outputs one number.
- **Input**: a number the network receives, such as one pixel or one feature of a customer.
- **Weight**: a learned number that sets how strongly one input influences a neuron.
- **Bias**: a learned number added to the weighted sum, which shifts the neuron's threshold.
- **Layer**: a group of neurons that all read the same inputs and produce one output each.
- **Activation function**: the nonlinear function applied to a neuron's weighted sum.
- **Pre-activation**: the weighted sum before the activation function is applied, often written z.
- **Forward pass**: the computation that turns an input into an output by passing it through every layer in order.
- **Hidden layer**: any layer that is neither the input nor the output layer.
- **Parameters**: all the learned weights and biases in a network.
- **Batch**: several inputs processed together as rows of one matrix.
- **Shape**: the dimensions of an array, such as (32, 784) for 32 images of 784 pixels each.

---

## Biological inspiration versus computation

Think about how you recognize a friend's face in a crowd. Light hits your retina, cells in the visual cortex respond to edges and colors, and signals travel through layers of neurons that each respond to a slightly more complex pattern. No single cell knows what a face is. The understanding comes from the chain of responses. That picture is the inspiration for artificial networks.

From first principles, a biological neuron receives electrical signals through its dendrites, sums them in its cell body, and fires an electrical spike along its axon when the total crosses a threshold. The strength of each connection, called a synapse, changes with experience. Engineers took two ideas from this picture: many inputs are combined, and the strength of each connection can be adjusted.

The network you build today is a loose imitation. Artificial neurons output real numbers, not spikes. Their connections are numbers stored in matrices in memory. Learning happens by optimization over examples, a different process from biological plasticity. The analogy is useful for intuition, and it is misleading when you assume that a network thinks the way a brain does. A network is a differentiable function with many parameters.

A worked example makes the difference concrete. Suppose a neuron receives the number 0.8 and multiplies it by the weight 2.0. The output of the weighted part is 1.6. A biological neuron would answer with a firing rate that depends on many chemical and timing details. The artificial neuron answers with exactly 1.6, every time, which makes it easy to test and easy to differentiate.

A common mistake is to treat the brain comparison as evidence that a bigger network is closer to intelligence. Size and architecture matter, but the analogy does not tell you which design choice will help. You check a design by measuring it on data. On the job, product teams use neural networks for search ranking, fraud scoring and speech recognition, and none of those systems are judged by how brain-like they are.

---

## Artificial neurons

Consider a judge scoring a diving competition. Each judge looks at several aspects of the dive, such as height, entry and form. The judge gives more weight to some aspects than others, adds a personal offset, and produces one score. A neuron works the same way: it looks at its inputs, gives each one an importance, adds an offset, and reports a single number.

Written as an equation, a neuron with inputs x1 to xn computes z = w1 x1 + w2 x2 + ... + wn xn + b. The weights w and the bias b are the parameters. The output is then a = f(z), where f is the activation function. In vector form the sum is the dot product z = w . x + b, which you already know from Day 8.

```python
import numpy as np

x = np.array([0.5, -1.0, 2.0])   # three inputs
w = np.array([0.4, 0.3, -0.2])   # three weights
b = 0.1                          # one bias

z = np.dot(w, x) + b             # 0.2 - 0.3 - 0.4 + 0.1 = -0.4
print(z)                         # -0.4
```

The neuron has four parameters and one output. If you change any weight, the output changes, and that is the handle that learning will use on Day 43. Without an activation function, a neuron would be a linear model, which is why the next sections matter.

Confusing the bias with an input is a common error. The bias is a parameter, learned like a weight, and it is not multiplied by a feature. Forgetting it is one of the most common causes of a model that cannot fit a simple dataset, because the decision boundary is then forced through the origin.

In industry, a single neuron with a sigmoid activation is exactly what logistic regression is, and that model still runs in many fraud and credit scoring systems because it is easy to explain.

---

## Inputs, weights and biases

Inputs are the numbers that describe one example. For a house, the inputs might be the floor area, the number of bedrooms and the distance to the nearest station. For an image, they are the pixel values, often scaled to the range 0 to 1. The network does not see the house or the photo, only these numbers, so the way you encode the input decides what the network can learn.

Weights decide how much each input matters. A large positive weight means the neuron grows when that input grows. A large negative weight means it shrinks. A weight near zero means the neuron ignores that input. Because weights are learned, the network discovers which inputs to trust. You do not need to know in advance that floor area matters more than bedroom count, although you can help by scaling the inputs so that they have comparable sizes.

The bias moves the threshold. Imagine a neuron with weights (1, 1) and no bias, which fires when x1 + x2 is positive. Adding a bias of -3 makes it fire only when x1 + x2 is greater than 3. The weights control the direction of the boundary and the bias controls where it sits.

Here is a small experiment you can run. Set the weights to (1, 1) and compare the biases 0 and -3 on the inputs (1, 1), (2, 2) and (3, 3). The weighted sums are 2, 4 and 6. With bias 0 the pre-activations are 2, 4 and 6, and with bias -3 they are -1, 1 and 3. The bias did not change the inputs, only the threshold at which the neuron turns positive.

A common mistake is to scale inputs badly. If one feature is in the range 0 to 1 and another in the range 0 to 1,000,000, the large feature dominates the weighted sum, and learning becomes slow and unstable. The fix is normalization, which you will study in Module 6 for images and again in Day 45. Real data pipelines in companies normalize features with statistics computed on the training set only, so the test set does not leak information.

---

## Layers

One neuron can only draw one boundary in the input space. Real problems need many boundaries that work together. A layer solves this by placing many neurons side by side, each reading the same inputs with its own weights and bias. If a layer has 4 neurons and the input has 3 values, the layer has 4 weight vectors of length 3, which you can stack into a matrix W of shape (3, 4) if you let each column be one neuron.

The output of a layer is then a vector of length 4, one number per neuron. When you process a batch of 32 examples at once, the input becomes a matrix X of shape (32, 3), and the layer computes Z = X W + b, which has shape (32, 4). Each row is one example and each column is one neuron. This is the dense layer, also called a fully connected layer, and it is the most common building block in deep learning.

```
input (3 values)        layer of 4 neurons          output (4 values)

  x1 ----+----------->  neuron 1  ----------->  a1
  x2 ----+--+-------->  neuron 2  ----------->  a2
  x3 ----+--+--+----->  neuron 3  ----------->  a3
            ...        neuron 4  ----------->  a4
```

Stacking layers is what makes a network deep. The output of layer one becomes the input of layer two, and so on. The intermediate layers are called hidden because you never see their values directly in the training data. Each hidden layer can reuse the features computed by the layer before it, which is why deep networks can build complex features from simple ones.

Confusing the number of layers with the number of neurons is a common error. Ten layers of 2 neurons is a very different model from one layer of 20 neurons. Depth gives you composition, and width gives you more features per level. Both have costs. In industry, the choice of width and depth is usually made by trial on a validation set, and the default widths that engineers start with come from published architectures rather than from theory.

---

## Activations

Suppose every layer computed only a weighted sum with no activation function. Two linear layers, Z1 = X W1 + b1 followed by Z2 = Z1 W2 + b2, would combine into one linear map, because (X W1) W2 is X times the matrix W1 W2. Stacking ten such layers would still give you one matrix multiplication. The depth would be wasted.

An activation function breaks this collapse. It is a nonlinear function applied to each neuron's output, so the next layer does not see a simple linear combination of the previous one. That nonlinearity lets the network bend its decision boundary, carve out curved regions and approximate functions that a straight line cannot. Day 41 compares the common choices, including sigmoid, tanh, ReLU and GELU. For today, the example below is enough.

```python
import numpy as np

def relu(z):
    return np.maximum(0.0, z)

z = np.array([-2.0, -0.5, 0.0, 1.5])
print(relu(z))          # [0.  0.  0.  1.5]
```

The limit of this idea is that a nonlinearity alone does not guarantee that a network will learn. A poorly chosen activation can make gradients vanish or explode, and you will see that on Day 44. Also, the claim that networks with one hidden layer can approximate any continuous function, the universal approximation theorem, says the network could represent the function, not that training will find it.

A common mistake is to add an activation after the last layer by reflex. For regression to a real number, the output usually has no activation, because the target can be negative. For binary classification, a sigmoid turns the output into a probability. For multi-class classification, a softmax does. The right output activation depends on the loss function you choose on Day 42.

---

## The forward pass

The forward pass is the full computation from input to output. You feed in a batch, each layer computes its weighted sum and applies its activation, and the last layer produces a prediction. Nothing is learned during a forward pass. The weights are fixed, and the pass is arithmetic.

Think of it as an assembly line. Each station reads the output of the station before, applies its own transformation and passes the result on. The line does not change its machines while it runs. Training changes the machines, and you will learn that on Day 43.

From first principles, a network with L layers has parameters (W1, b1), (W2, b2) and so on. Set a1 = x. For each layer l from 1 to L, compute z_l = a_{l-1} W_l + b_l and a_l = f_l(z_l). The prediction is a_L. You store the intermediate z and a values, because the backward pass on Day 43 needs them.

```python
import numpy as np

def dense(X, W, b, f):
    return f(X @ W + b)

def relu(z):
    return np.maximum(0.0, z)

def identity(z):
    return z

rng = np.random.default_rng(seed=0)
X = rng.normal(size=(4, 3))          # batch of 4 examples, 3 features each
W1, b1 = rng.normal(size=(3, 5)), np.zeros(5)
W2, b2 = rng.normal(size=(5, 2)), np.zeros(2)

H = dense(X, W1, b1, relu)           # hidden layer, shape (4, 5)
Y = dense(H, W2, b2, identity)       # output layer, shape (4, 2)
print(H.shape, Y.shape)              # (4, 5) (4, 2)
```

The most common mistake in a forward pass is a shape mismatch. If the second layer expects 5 inputs and you pass 3, the matrix multiplication fails. Check shapes at every step while you learn. Print them, write them as comments, or assert them. In production, shape bugs are caught by tests, and a good habit is to write one test per layer that checks the output shape for a small batch.

The forward pass also shows up in deployment. When you export a model to ONNX or serve it through TorchScript or a managed inference endpoint, you are shipping the forward pass and nothing else. The training code, the optimizer and the loss are not needed at inference time.

---

## Hands-on exercises

1. **Neuron by hand.** On paper, compute the output of a neuron with inputs (1, -2), weights (0.5, 0.25), bias 0.1 and the ReLU activation. Then check your answer in Python. Your answer should be 0.1 for the pre-activation and 0.1 after ReLU. Write down the two numbers you got and whether they matched.
2. **Shape table.** For a dense network with layers of sizes 784, 128, 64 and 10, write the shape of W1, b1, W2, b2, W3 and b3. Then verify with `print` statements on random arrays for a batch of 32 examples. The output of the network must have shape (32, 10).
3. **Bias experiment.** Build a single neuron with weights (1, -1) and sweep the bias from -2 to 2 in steps of 0.5. For each bias, compute the output on the input (1, 0). Describe in one sentence how the bias changes the output.
4. **Collapse test.** Write two linear layers without activations, then compute their composition with a single matrix W1 @ W2. Confirm with `np.allclose` that the two results match for a random input. Then insert a ReLU between the layers and show that the match fails for some inputs.
5. **Plot a neuron.** Using Matplotlib, plot the output of a single sigmoid neuron on a grid of two inputs as a heat map. Change the weights and describe how the boundary moves.

---

## Project: a forward pass from scratch with NumPy

**Goal:** build a small library that computes neurons, dense layers and a two-layer network with NumPy only. Test each part against hand calculations and against a second implementation. Commit the code, the tests and a short README that explains the shapes.

**Repo name:** `100-days-ai-day-39-neural-networks`

**Repo layout:**

```
100-days-ai-day-39-neural-networks/
├── README.md
├── requirements.txt
├── src/
│   └── neural_net/
│       ├── __init__.py
│       ├── activations.py
│       ├── neuron.py
│       ├── layers.py
│       └── network.py
├── tests/
│   └── test_forward.py
└── examples/
    └── forward_demo.py
```

**Starter code.** Create `requirements.txt` with the three pinned packages from the requirements section. Then create the files below.

`src/neural_net/activations.py`:

```python
import numpy as np


def relu(z):
    return np.maximum(0.0, z)


def sigmoid(z):
    return 1.0 / (1.0 + np.exp(-z))


def identity(z):
    return z
```

`src/neural_net/neuron.py`:

```python
import numpy as np


def neuron_forward(x, w, b, activation=None):
    """Compute activation(w . x + b) for one neuron."""
    x = np.asarray(x, dtype=float)
    w = np.asarray(w, dtype=float)
    if x.shape != w.shape:
        raise ValueError(f"x and w must have the same shape, got {x.shape} and {w.shape}")
    z = float(np.dot(w, x) + b)
    return z if activation is None else activation(z)
```

`src/neural_net/layers.py`:

```python
import numpy as np


def dense_forward(X, W, b, activation=None):
    """X: (batch, n_in), W: (n_in, n_out), b: (n_out,). Returns (batch, n_out)."""
    X = np.asarray(X, dtype=float)
    W = np.asarray(W, dtype=float)
    b = np.asarray(b, dtype=float)
    if X.shape[1] != W.shape[0]:
        raise ValueError(f"X has {X.shape[1]} features but W expects {W.shape[0]}")
    Z = X @ W + b
    return Z if activation is None else activation(Z)
```

`tests/test_forward.py`:

```python
import numpy as np
import pytest

from neural_net.activations import relu, sigmoid
from neural_net.layers import dense_forward
from neural_net.neuron import neuron_forward


def test_neuron_matches_hand_calculation():
    # 2*1 + 1*(-1) = 1, plus bias 0.5 gives 1.5
    assert neuron_forward([2, 1], [1, -1], 0.5) == pytest.approx(1.5)


def test_relu_clips_negative_values():
    assert neuron_forward([1, 1], [-1, -1], 0.0, relu) == 0.0


def test_sigmoid_of_zero_is_half():
    assert neuron_forward([0, 0], [1, 1], 0.0, sigmoid) == pytest.approx(0.5)


def test_dense_layer_output_shape():
    X = np.ones((4, 3))
    W = np.ones((3, 2))
    b = np.zeros(2)
    out = dense_forward(X, W, b)
    assert out.shape == (4, 2)
    assert np.allclose(out, 3.0)


def test_dense_matches_neuron_by_neuron():
    rng = np.random.default_rng(0)
    X = rng.normal(size=(5, 3))
    W = rng.normal(size=(3, 2))
    b = rng.normal(size=2)
    out = dense_forward(X, W, b, relu)
    for i in range(5):
        for j in range(2):
            expected = neuron_forward(X[i], W[:, j], b[j], relu)
            assert out[i, j] == pytest.approx(expected)


def test_mismatched_shapes_raise():
    with pytest.raises(ValueError):
        dense_forward(np.ones((2, 4)), np.ones((3, 2)), np.zeros(2))
```

Create `src/neural_net/network.py` yourself. It should define a class `TwoLayerNetwork` whose constructor takes the layer sizes, creates random weights with a seeded NumPy generator and zero biases, and whose `forward` method applies a ReLU hidden layer followed by an identity output layer. Create `examples/forward_demo.py` to print the input shape, hidden shape and output shape for a batch of 8 random examples with 4 features.

**Step-by-step tasks:**

1. Create the repo folder, initialize Git and make a virtual environment with Python 3.11 or 3.12.
2. Install the pinned packages with `pip install -r requirements.txt`.
3. Create the files above and make `src` importable. Run the tests from the repo root with `PYTHONPATH=src pytest` on macOS or Linux, or `$env:PYTHONPATH="src"; pytest` in PowerShell.
4. Write `network.py` and a test that checks the output shape of `TwoLayerNetwork(4, 6, 2).forward(X)` for a batch of 8.
5. Add one test that compares your network to a manual computation for a batch of two examples, using numbers you calculate on paper.
6. Run `examples/forward_demo.py` and copy its output into the README.
7. In the README, write a three-line explanation of the shape of every parameter and every intermediate array.

**README checklist:**

- [ ] The one-line description of the project
- [ ] Setup steps with the exact Python and package versions
- [ ] How to run the tests and the demo
- [ ] A table of parameter shapes for the two-layer network
- [ ] The demo output pasted in as a code block
- [ ] One sentence on what this project does not do yet (no learning, no backpropagation)

**Done when:**

- `pytest` passes all tests on a clean virtual environment
- The shape test fails if you change the number of hidden units in `forward`, and you have seen it fail at least once
- The two-layer network's output matches your hand calculation to six decimal places
- The README explains every shape in plain words
- The repo has at least four commits with clear messages

**Evidence to commit to GitHub:** the source files, the tests, the `requirements.txt`, the README with the demo output, and a short `NOTES.md` that lists three shape bugs you hit and how you fixed them. If you did not hit any, write down the one that you think is most likely in your own code.

**Why this project is practical:** it is small, but it forces you to write the forward pass as a set of shapes, which is the skill you need before any framework will feel natural.

---

## Check your understanding

1. A neuron has three inputs, and you double every weight and the bias. Does its pre-activation double? Explain why, and say what happens to the output after a sigmoid.
2. Why does a deep network with no activation functions behave like a single linear layer? Use the matrix product to explain it.
3. A colleague says the bias is "just another input that is always one". Is that a correct view? What is the practical difference between the two views when you write code?
4. You feed a batch of 16 images of 28 by 28 pixels into a dense layer with 100 neurons. What is the shape of the output, and why does the number 784 appear in the weight matrix?
5. Explain why scaling input features to similar ranges can speed up learning, even though the network can learn any scale in principle.
6. A model has one hidden layer of 1,000 ReLU neurons, and its training accuracy is high while the test accuracy is low. Is the problem in the forward pass? What would you check first?

**Answers:**

1. Yes. Since z = w . x + b is linear in w and b, scaling all of them by two scales z by two. The output after a sigmoid does not double, because the sigmoid is nonlinear, so the output moves to a different value that depends on where z started.
2. Two layers give Z1 = X W1 + b1 and Z2 = Z1 W2 + b2. Substituting shows Z2 = X (W1 W2) + (b1 W2 + b2), which is one matrix multiplication plus one bias. Without a nonlinearity between the layers, the composition stays linear.
3. The view is mathematically convenient, because you can append a column of ones to the input and fold the bias into the weights. In code, however, the bias is a separate parameter with its own shape, and it is often stored separately so that regularization can skip it.
4. The output has shape (16, 100). The number 784 is 28 times 28, which is the number of input features after flattening each image into a vector. The weight matrix therefore has shape (784, 100), plus a bias of shape (100,).
5. Features on very different scales make the weighted sum dominated by the large features, so gradients for the small ones are tiny and the optimizer struggles. Scaling puts the features on comparable footing, which makes the loss surface easier to traverse.
6. It is probably not the forward pass. High training accuracy with low test accuracy points to overfitting, which Day 45 covers. First check whether the train and test splits are clean, and whether the model is much larger than the data supports.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Artificial neural network", for the overview and the history of the field
- Wikipedia, "Artificial neuron", for the mathematical definition used in this lesson
- Wikipedia, "Activation function", for a survey of the nonlinearities you will meet on Day 41
- Wikipedia, "Biological neuron model", for the biology that inspired the design
- Stanford CS231n course notes, "Neural Networks Part 1: Setting up the Architecture", at cs231n.github.io/neural-networks-1

**Tutorials**

- NumPy documentation, "NumPy quickstart", at numpy.org/doc/stable/user/quickstart.html
- GeeksforGeeks, search for "Introduction to Artificial Neural Networks", for a step-by-step walk through neurons and layers
- pytest documentation, "Get Started", at docs.pytest.org, for the testing tools used in the project

**Watch**

- 3Blue1Brown, "But what is a neural network? | Deep learning chapter 1" on YouTube
- StatQuest with Josh Starmer, "Neural Networks Pt. 1: Inside the Black Box" on YouTube

**Practice**

- TensorFlow Playground at playground.tensorflow.org, where you can change the layers, neurons and activations and watch the decision boundary change in your browser

---

## Tomorrow

Day 40 takes the single neuron you built today and asks what it cannot do. A perceptron is one neuron with a step activation, and it can only separate classes that a straight line divides. You will see why the XOR problem stopped early research in its tracks, and how a multi-layer perceptron solves it by stacking hidden layers. The forward pass you wrote today is the same computation you will use there, so the code you commit now becomes the base for tomorrow's experiments.

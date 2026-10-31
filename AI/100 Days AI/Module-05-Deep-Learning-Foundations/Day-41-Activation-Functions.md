# Day 41: Activation Functions

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3 hours
**Level:** intermediate. You can write a forward pass (Day 39) and you have seen why a perceptron with a step function cannot learn from its errors (Day 40). Today you need to differentiate simple functions, so basic calculus from Day 11 is enough.

**By the end of this day, you will have:**

- A NumPy module with six activation functions (sigmoid, tanh, ReLU, leaky ReLU, GELU and softmax), each with its derivative
- Tests that check every derivative against a finite-difference estimate
- A measured experiment showing how the product of derivatives shrinks through a deep sigmoid stack and how ReLU avoids that shrinkage
- A one-page decision table that says which activation to use at each position in a network and why

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the optional plot
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 11: derivatives, the chain rule in its simplest form and the meaning of slope
- Day 12: how a derivative tells you the direction of change
- Day 39: neurons, layers and the forward pass
- Day 40: why a step function gives no useful gradient

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Why nonlinearities matter](#why-nonlinearities-matter)
- [Sigmoid](#sigmoid)
- [Tanh](#tanh)
- [ReLU](#relu)
- [Leaky ReLU](#leaky-relu)
- [GELU concept](#gelu-concept)
- [Softmax](#softmax)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an activation library and a depth experiment](#project-an-activation-library-and-a-depth-experiment)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

An activation function is the one part of a neuron that is not a weighted sum. It takes the pre-activation z and returns a new value, and that small step is what lets a stack of layers represent curves, corners and combinations rather than straight lines. Day 40 showed that the step function can do this in principle, but its slope is zero everywhere except at the jump, so a learning rule based on slopes cannot use it. Modern networks need activations that are nonlinear and have useful derivatives.

Choosing an activation has real consequences. It decides how gradients flow backward during training, whether units can die and stop learning, and how stable the numbers stay as the network gets deeper. Two networks with the same layers can train in very different ways if you swap one activation for another. Many training failures that look like a bad learning rate are really activation problems.

On the job, you will pick activations in every model you build. A transformer language model uses GELU or a close relative inside its feed-forward blocks. A convolutional image model usually uses ReLU. A classifier uses softmax at the output, and a binary classifier uses sigmoid. Knowing the shape and the derivative of each function lets you debug a model that saturates, produces NaN values or stops learning after a few epochs.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1943 | McCulloch and Pitts use a threshold function as the neuron's output. | The step activation, which has no useful gradient. |
| 1980s | Logistic sigmoid and hyperbolic tangent units become standard in multi-layer networks trained with backpropagation. | The smooth functions that made gradient-based training possible. |
| 2000 | Richard Hahnloser and colleagues report rectified linear units with biological and computational arguments, and the idea spreads. | The origin of the ReLU family in modern practice. |
| 2011 | Xavier Glorot, Antoine Bordes and Yoshua Bengio show that ReLU networks train well in deep settings. | ReLU becomes the default hidden activation. |
| 2016 | Dan Hendrycks and Kevin Gimpel propose the Gaussian error linear unit (GELU). | GELU appears in transformer models such as BERT and GPT-2. |

**Takeaway:** the functions changed because the gradients did. Each new activation solved a specific problem with the one before it.

---

## Glossary

- **Nonlinearity**: a function whose output is not a scaled sum of its inputs. Activations are the source of nonlinearity in a network.
- **Saturation**: the regime in which a function is nearly flat, so its derivative is close to zero.
- **Vanishing gradient**: a training problem in which gradients shrink toward zero as they pass backward through many layers.
- **Dead unit**: a ReLU neuron that outputs zero for every input, and therefore passes no gradient.
- **Derivative**: the rate of change of a function, which determines how much a parameter should move.
- **Finite difference**: an approximation to a derivative, computed as (f(x + h) - f(x - h)) / (2h) for a small h.
- **Logit**: a raw score from the last layer, before it is turned into a probability.
- **Probability distribution**: a set of nonnegative numbers that sum to one.
- **Zero-centered**: an output whose average is near zero, which tends to help gradient descent move in both directions.
- **Gaussian distribution**: the bell-shaped probability distribution, whose cumulative function appears in the definition of GELU.

---

## Why nonlinearities matter

Start with a stack of linear layers. Each layer computes a matrix product and a bias. Two linear layers compose into one linear map, as you saw on Day 39. A stack of ten linear layers is still one matrix multiplication, so it can only draw a straight boundary, however deep you make it.

From first principles, if f is linear, then f(a x + b y) = a f(x) + b f(y). Composing two linear functions gives another linear function, so depth adds no new expressive power. An activation function g breaks this property, because g(a x + b y) is generally not a g(x) + b g(y). Each nonlinear layer folds the space in a new way, and the number of folds grows quickly with depth and width. That is the source of a deep network's ability to draw complicated regions.

```python
import numpy as np

x = np.array([-2.0, -0.5, 0.5, 2.0])

linear_twice = 2.0 * (3.0 * x)       # equals 6 * x: still linear
relu_then_scale = 2.0 * np.maximum(0.0, x)

print(linear_twice)                  # [-12.  -3.   3.  12.]
print(relu_then_scale)               # [ 0.  0.  1.  4.]
```

Not every nonlinearity is equally good. A nonlinearity that squashes all of its inputs into a tiny range can make learning slow, even if it is nonlinear in the mathematical sense. The useful question is whether the function is nonlinear and whether its derivative stays in a range that gradient descent can use.

In the real market, every production model combines these two properties. The nonlinearity makes the model expressive, and the derivative makes it trainable. When someone proposes a new activation, the first thing a serious team checks is what happens to the gradient in the regions where the inputs are large or negative.

---

## Sigmoid

Picture a dimmer switch that is never fully off or fully on. As you turn the knob, the light brightens slowly at the ends and quickly in the middle. The sigmoid is that kind of curve. It maps any real number to a value between 0 and 1, and it changes most rapidly near zero.

From first principles, the logistic sigmoid is sigma(z) = 1 / (1 + exp(-z)). At z = 0 it returns 0.5, at large positive z it approaches 1, and at large negative z it approaches 0. Its derivative is sigma'(z) = sigma(z) (1 - sigma(z)). That formula is convenient because the derivative can be computed from the output, which you will reuse on Day 43. The largest value of the derivative is 0.25, at z = 0.

```python
import numpy as np

def sigmoid(z):
    return 1.0 / (1.0 + np.exp(-z))

def sigmoid_grad(z):
    s = sigmoid(z)
    return s * (1.0 - s)

z = np.array([-4.0, 0.0, 4.0])
print(sigmoid(z))        # [0.018 0.5   0.982]
print(sigmoid_grad(z))   # [0.018 0.25  0.018]
```

The sigmoid has two limitations. Its output is never zero-centered, so all values in a layer are positive, which can make weight updates zigzag. Its derivative is at most 0.25, so when many sigmoid layers are stacked, the gradient shrinks by at least a factor of four per layer, and usually much more. The depth experiment in the project measures this effect.

The common mistake is to use sigmoid in hidden layers out of habit. Sigmoid is still correct at the output of a binary classifier, where the value means a probability, and inside a few specialized gates, such as the forget gate of an LSTM. In the real market, you see it in logistic regression baselines, in click-through prediction heads and in the output of binary classifiers in fraud and spam systems.

---

## Tanh

Consider the same dimmer switch, but now it can go below zero as well. The hyperbolic tangent is a sigmoid that is centered on zero and stretched to cover the range from -1 to 1. It has the same S shape and the same saturation at the ends.

From first principles, tanh(z) = (exp(z) - exp(-z)) / (exp(z) + exp(-z)). It is related to the sigmoid by tanh(z) = 2 sigma(2z) - 1. Its derivative is 1 - tanh(z)^2. At z = 0 the derivative is 1, which is four times the maximum slope of the sigmoid. That is why tanh often trains better than sigmoid in hidden layers.

```python
import numpy as np

def tanh_grad(z):
    return 1.0 - np.tanh(z) ** 2

z = np.array([-2.0, 0.0, 2.0])
print(np.tanh(z))        # [-0.964  0.     0.964]
print(tanh_grad(z))      # [ 0.0707 1.     0.0707]
```

Tanh still saturates. At z = 2 the derivative is already about 0.07, and at z = 4 it is about 0.0013. A deep stack of tanh layers can therefore suffer from vanishing gradients, although less severely than a sigmoid stack. Tanh is still used in recurrent networks and in some gating units, where its bounded output is useful.

Assuming that a zero-centered activation is always better is a mistake. It helps optimization in many cases, but saturation is the dominant problem in deep networks, and tanh does not solve it. In practice, tanh is common in the hidden state of older recurrent models, in some audio models and in gating units, and the output range is useful when you need signed values bounded to a fixed interval.

---

## ReLU

Think of a one-way valve in a water pipe. Water flows freely in one direction and is blocked in the other. The rectified linear unit, ReLU, is that valve applied to a number. It returns z if z is positive and zero otherwise.

From first principles, ReLU(z) = max(0, z). Its derivative is 1 for z greater than 0 and 0 for z less than 0. At exactly z = 0 the derivative is undefined, and frameworks choose a convention, usually 0. The derivative does not shrink for positive inputs, so gradients pass through a chain of ReLU layers without the factor-of-four loss that sigmoid causes. Computing ReLU is also cheap, since it is a comparison rather than an exponential.

```python
import numpy as np

def relu(z):
    return np.maximum(0.0, z)

def relu_grad(z):
    return (z > 0).astype(float)

z = np.array([-1.5, 0.0, 2.0])
print(relu(z))           # [0. 0. 2.]
print(relu_grad(z))      # [0. 0. 1.]
```

ReLU has its own failure mode. A unit whose pre-activation is always negative outputs zero and has zero gradient, so it never updates. This is called a dead unit. Dead units can appear after a large learning-rate step pushes a neuron's bias far negative, and a network can lose a large fraction of its units this way. Watching the fraction of activations that are exactly zero is a simple check.

The common mistake is to treat ReLU as a fix for every training problem. It is a good default for hidden layers in convolutional and feed-forward networks, but it does not make the output layer any easier, and it does not fix a bad initialization or a learning rate that is too high. On the job, ReLU appears in almost every image classifier that runs on a phone, because it is fast and it works well with the convolution operations on Day 55.

---

## Leaky ReLU

Imagine the valve from the last section with a tiny gap in it, so a trickle still flows backward. The leaky ReLU is that design. It returns z for positive inputs and a small multiple of z for negative inputs, so no unit is ever completely dead.

From first principles, LeakyReLU(z) = z if z > 0, and alpha z otherwise, where alpha is a small constant such as 0.01. Its derivative is 1 for positive inputs and alpha for negative inputs. The gradient is never zero, so a unit that starts in the negative region can still recover.

```python
import numpy as np

def leaky_relu(z, alpha=0.01):
    return np.where(z > 0, z, alpha * z)

def leaky_relu_grad(z, alpha=0.01):
    return np.where(z > 0, 1.0, alpha)

z = np.array([-3.0, 0.0, 3.0])
print(leaky_relu(z))           # [-0.03  0.    3.  ]
print(leaky_relu_grad(z))      # [0.01 1.   1.  ]
```

The limitation is that the small slope on the negative side is a hyperparameter, and the benefit over ReLU is often small. Some experiments show improvement, and others show no difference. The parametric version, where alpha is learned, adds parameters and can overfit small datasets. Do not switch to leaky ReLU without a measurement and then assume a gain.

In practice, leaky ReLU and its variants appear in generative adversarial networks, where gradient flow is delicate, and in some detection models. Many production teams prefer GELU or SiLU in transformers and ReLU elsewhere, and they keep leaky ReLU for specific reasons.

---

## GELU concept

Consider a gate that lets a value through in proportion to how likely it is to be important. A large positive input passes almost unchanged, a large negative input is almost blocked, and inputs near zero are scaled by a smooth fraction between those extremes. The Gaussian error linear unit, GELU, works this way.

From first principles, GELU(z) = z Phi(z), where Phi is the cumulative distribution function of the standard normal distribution. That means each input is multiplied by the probability that a standard normal variable is less than z. For large positive z, Phi is near 1, so GELU is close to z. For large negative z, Phi is near 0, so GELU is close to zero. Near zero the curve is smooth, which gives a clean derivative. A common and fast approximation is 0.5 z (1 + tanh(sqrt(2/pi) (z + 0.044715 z^3))).

```python
import numpy as np

def gelu_approx(z):
    c = np.sqrt(2.0 / np.pi)
    return 0.5 * z * (1.0 + np.tanh(c * (z + 0.044715 * z ** 3)))

z = np.array([-2.0, 0.0, 2.0])
print(gelu_approx(z))     # approximately [-0.0454  0.      1.9546]
```

GELU is not free. It requires a tanh or an error function, which costs more than ReLU, and its benefit depends on the model. The transformer papers that adopted it reported gains over ReLU on their tasks, but those gains do not transfer automatically to every architecture. Do not copy an activation from a transformer into a convolutional network without measuring.

In the real market, GELU appears inside the feed-forward blocks of many language and vision transformers. When you read a model's configuration file and see `hidden_act: "gelu"`, that choice is a deliberate design decision inherited from the research.

---

## Softmax

Imagine a panel of three judges who each give a raw score to three contestants. The scores can be any real numbers. To turn them into a shared share of the vote, you exponentiate each score so everything is positive, then divide by the total. That is softmax.

From first principles, softmax(z)_i = exp(z_i) / sum over j of exp(z_j). The outputs are positive and sum to one, so they form a probability distribution. Subtracting the maximum score from every entry before exponentiating gives the same result and avoids overflow for large values. The derivative is not a single number per entry, because each output depends on all of the inputs, but in practice you rarely write it by hand, because the cross-entropy loss on Day 42 combines the two cleanly.

```python
import numpy as np

def softmax(z):
    shifted = z - np.max(z, axis=-1, keepdims=True)
    e = np.exp(shifted)
    return e / np.sum(e, axis=-1, keepdims=True)

logits = np.array([2.0, 1.0, 0.1])
p = softmax(logits)
print(np.round(p, 4))          # [0.659  0.2424 0.0986]
print(p.sum())                 # 1.0
```

The limitation is that softmax is sensitive to the scale of its inputs. Large logits make the output nearly one-hot, which can hide uncertainty. A model that is overconfident often has large logits, and calibration methods such as temperature scaling adjust the logits before the softmax.

The common mistake is to apply softmax to a set of values that are not logits of one decision, for example to independent sigmoid outputs in a multi-label problem. Use sigmoid per label in that case, because each label has its own probability. Softmax is the right choice when exactly one class is true. In the real market, it is the output of almost every multi-class image classifier and of the next-token distribution in a language model.

---

## Hands-on exercises

1. **Derivative by hand.** Compute sigmoid(0), sigmoid'(0) and tanh'(0) on paper. Confirm each with a short Python call. Write the three values in your notes.
2. **Saturation table.** For z in the list [-6, -3, 0, 3, 6], compute the sigmoid derivative and the tanh derivative. Describe in two sentences where each derivative is largest and smallest.
3. **Dead unit simulation.** Create 1,000 random pre-activations with mean -1 and standard deviation 0.5. Count how many are exactly zero after ReLU. Then count how many are zero after leaky ReLU with alpha 0.01.
4. **Softmax stability.** Compute softmax of [1000, 1001, 1002] with and without the subtraction of the maximum. Report what you see without the subtraction and explain why.
5. **Probability check.** Draw ten random logit vectors of length 5 and confirm that each softmax output sums to one within 1e-12.

---

## Project: an activation library and a depth experiment

**Goal:** build an activation module with forward and derivative functions, test every derivative numerically, and measure how the product of derivatives changes with depth for sigmoid, tanh and ReLU.

**Repo name:** `100-days-ai-day-41-activation-functions`

**Repo layout:**

```
100-days-ai-day-41-activation-functions/
├── README.md
├── requirements.txt
├── src/
│   └── activations/
│       ├── __init__.py
│       └── functions.py
├── tests/
│   └── test_derivatives.py
└── experiments/
    └── depth_gradient.py
```

**Starter code.** Create `requirements.txt` with the pinned packages from the requirements section. Create `src/activations/functions.py` with the following content, then extend it.

```python
import numpy as np


def sigmoid(z):
    return 1.0 / (1.0 + np.exp(-z))


def sigmoid_grad(z):
    s = sigmoid(z)
    return s * (1.0 - s)


def tanh_grad(z):
    return 1.0 - np.tanh(z) ** 2


def relu(z):
    return np.maximum(0.0, z)


def relu_grad(z):
    return (z > 0).astype(float)


def leaky_relu(z, alpha=0.01):
    return np.where(z > 0, z, alpha * z)


def leaky_relu_grad(z, alpha=0.01):
    return np.where(z > 0, 1.0, alpha)


def gelu(z):
    c = np.sqrt(2.0 / np.pi)
    return 0.5 * z * (1.0 + np.tanh(c * (z + 0.044715 * z ** 3)))


def softmax(z):
    shifted = z - np.max(z, axis=-1, keepdims=True)
    e = np.exp(shifted)
    return e / np.sum(e, axis=-1, keepdims=True)
```

**Step-by-step tasks:**

1. Create the repo, initialize Git, make a virtual environment and install the pinned packages.
2. Add `gelu_grad` to `functions.py`. Use the derivative of the tanh approximation, or compute it numerically with a central difference and document that choice in the README.
3. Write `tests/test_derivatives.py`. For each of sigmoid, tanh, ReLU, leaky ReLU and GELU, pick ten points away from zero, compute the central difference with h = 1e-5, and assert that the analytic derivative matches within 1e-4. Skip ReLU points exactly at zero.
4. Add a test that softmax rows sum to one and that softmax of `[1000, 1001, 1002]` returns finite numbers.
5. Write `experiments/depth_gradient.py`. For depths from 1 to 20, compute the product of the derivative at z = 0.5 for each activation, repeated depth times. Print a table of the results.
6. Save a plot of the product against depth on a log scale to `results/depth_gradient.png`.
7. In the README, record the table and write three sentences on what the numbers show and what they do not show. The experiment ignores weights, so say so.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the experiment
- [ ] The depth table with the product for each activation
- [ ] The plot, saved in the results folder
- [ ] A paragraph that states what the experiment ignores (weights, biases, the interaction with the other layers)

**Done when:**

- All derivative tests pass, and one test fails if you deliberately flip the sign of a derivative
- The softmax test passes for the large-value input without NaN
- The depth table shows the sigmoid product falling far faster than the ReLU product
- The README explains why the experiment is a simplified model and not a full training run
- The repo has at least four commits

**Evidence to commit to GitHub:** the source files, the tests, the README with the table and the plot, and a `NOTES.md` that lists one activation you would not use in a given setting and the reason. The notes show that you understand the choice, not only the code.

**Why this project is practical:** finite-difference checks are a standard way to find derivative bugs in any model you write by hand, and the depth experiment shows why the choice of activation changes training.

---

## Check your understanding

1. Why does composing two linear layers give no benefit over one, and what does an activation change about that composition?
2. The sigmoid derivative is at most 0.25. Using that bound, explain why a stack of ten sigmoid layers can have a very small gradient at the first layer.
3. A ReLU network has 30 percent of its hidden units producing zero on every example in a batch. Name two causes and one change you would test first.
4. Why is tanh often preferred to sigmoid in hidden layers, and what limitation does it share with sigmoid?
5. You have a three-class image classifier and a multi-label tagger where each image can have several tags. Which output activation do you use for each, and why?
6. A colleague wants to switch from ReLU to GELU in a convolutional image model because a transformer paper reported gains. What would you need to measure before accepting the change?

**Answers:**

1. Two linear layers compute a matrix product, and the product of two matrices is one matrix. So the composed function is linear, and the second layer adds no new shape to the boundary. An activation between the layers breaks the linearity, so each layer can fold the space in a new way.
2. The gradient at the first layer is the product of the derivatives of all later layers, each at most 0.25. Even if every derivative were at its maximum, the product would be 0.25 to the tenth power, which is about one millionth. Real values are smaller, because most inputs are away from zero.
3. The causes include a learning rate large enough to push biases strongly negative, and an initialization that starts pre-activations in the negative range. The first change to test is a lower learning rate or a leaky ReLU, then compare the fraction of zeros.
4. Tanh is zero-centered, so its outputs can be positive or negative, and its derivative at zero is 1 rather than 0.25. Both functions saturate for large inputs, so both can suffer from vanishing gradients in deep stacks.
5. For the three-class problem, each image has exactly one class, so softmax is correct. For the multi-label tagger, each tag is an independent yes or no question, so each output should be a sigmoid, and the loss is binary cross-entropy per tag.
6. You would need to measure accuracy, training stability and inference cost on your own data, with the same training schedule and several random seeds. GELU is more expensive than ReLU, so the gain has to justify the cost on your task.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Activation function", for the overview and a table of common functions
- Wikipedia, "Sigmoid function", for the logistic curve and its properties
- Wikipedia, "Hyperbolic functions", for tanh and its relation to the exponential
- Wikipedia, "Rectifier (neural networks)", for ReLU, leaky ReLU and their variants
- Wikipedia, "Softmax function", for the definition and numerical notes
- Wikipedia, "Gaussian error linear unit", for GELU and the approximation
- Wikipedia, "Vanishing gradient problem", for the depth effect measured in the project
- Stanford CS231n course notes, "Neural Networks Part 1", at cs231n.github.io, which covers activation functions in detail

**Tutorials**

- PyTorch documentation, "torch.nn" non-linear activation functions, at pytorch.org/docs/stable, which lists the built-in versions you will use in later days
- NumPy documentation, "numpy.exp" and "numpy.where", at numpy.org/doc/stable, for the functions used in the project
- GeeksforGeeks, search for "Activation Functions in Neural Networks", for side-by-side plots and examples

**Watch**

- 3Blue1Brown, "But what is a neural network? | Deep learning chapter 1", on YouTube, for the role of the nonlinearity in a single neuron
- Stanford CS231n, the lecture on training neural networks, which covers activation choices, on YouTube

**Practice**

- TensorFlow Playground at playground.tensorflow.org. Switch the activation between Sigmoid, Tanh, ReLU and Linear and train the same network on the spiral dataset. Describe the difference you see in training speed and in the boundary.

---

## Tomorrow

Day 42 turns the outputs of the network into a number that tells the learning algorithm how wrong it was. You will compare mean squared error, binary cross-entropy, categorical cross-entropy and negative log likelihood, and you will see why the choice of loss must match the activation at the output. The softmax you wrote today appears again there, where it pairs with cross-entropy to give a clean gradient.

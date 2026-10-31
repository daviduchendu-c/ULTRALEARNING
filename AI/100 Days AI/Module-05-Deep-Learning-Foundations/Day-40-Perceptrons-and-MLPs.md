# Day 40: Perceptrons and MLPs

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You finished Day 39 and can compute a forward pass with NumPy. Today you add a learning rule for a single neuron and then see why one neuron is not enough.

**By the end of this day, you will have:**

- A working perceptron with a learning rule, trained on the AND function until it makes no errors
- A demonstration, with code and a counterexample, that a single perceptron cannot learn XOR
- A hand-wired two-layer multi-layer perceptron (MLP) that computes XOR, with a table showing the hidden layer's new representation of the data
- A short written note on what hidden layers buy you and what they cost

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the optional plot in the project
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 39: the neuron equation z = w . x + b, the forward pass and shapes
- Day 8: the dot product and vectors as points in space
- Day 10: lines and planes as the boundaries that a linear map can draw
- Basic Python: loops, functions and classes

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Perceptron](#perceptron)
- [Linear separability](#linear-separability)
- [Multi-layer perceptrons](#multi-layer-perceptrons)
- [Hidden layers](#hidden-layers)
- [Representation learning](#representation-learning)
- [Hands-on exercises](#hands-on-exercises)
- [Project: perceptron, XOR and a hand-wired MLP](#project-perceptron-xor-and-a-hand-wired-mlp)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Yesterday you wrote the forward pass of a network but left the weights as numbers you chose or drew at random. Today you go one step further in two directions. First you give a single neuron a way to correct its own weights from its mistakes, which is the perceptron. Then you find the problem that a single neuron cannot solve, and you see how stacking neurons into layers solves it. That problem, XOR, was once blamed for stalling research on neural networks for more than a decade.

The perceptron matters for two reasons. It is the first learning algorithm that adjusts a model from labeled examples, and it has the same structure as the modern training loop: predict, compare with the label, nudge the weights. Everything you learn on Days 42 to 50 is a refinement of that loop. The multi-layer perceptron matters because it is the simplest network that can represent functions a linear model cannot, and it is the ancestor of the dense layers you will use in every PyTorch model.

On the job, you will rarely train a bare perceptron. You will meet it in a few places: a linear classifier as the last layer of a network, a quick baseline that tells you whether your features separate the classes at all, and a reminder that a model with no hidden layers has a hard ceiling on what it can express. When a teammate asks why you need a deeper model, the honest answer is in this lesson.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1943 | Warren McCulloch and Walter Pitts propose a logical model of a neuron with threshold firing. | The first mathematical neuron, and the origin of the step activation. |
| 1958 | Frank Rosenblatt presents the perceptron and shows a learning rule for it, with a hardware machine built at the Cornell Aeronautical Laboratory. | The first learning rule for an artificial neuron. |
| 1969 | Marvin Minsky and Seymour Papert publish "Perceptrons", which proves that a single layer cannot compute XOR. | The result that slowed funding and interest in neural networks for years. |
| 1986 | Rumelhart, Hinton and Williams popularize backpropagation for multi-layer networks. | The training method that makes the hidden layers of an MLP learnable. |
| 1989 | George Cybenko and others prove that a single hidden layer with enough neurons can approximate continuous functions on bounded domains. | The universal approximation result, which says what hidden layers can represent, not how to find the weights. |

**Takeaway:** the perceptron learning rule is old and simple. The limitation it exposed is the reason multi-layer networks exist.

---

## Glossary

- **Perceptron**: a single neuron with a step activation and a learning rule that updates its weights when it makes a mistake.
- **Step function**: an activation that outputs 1 when its input is zero or positive, and 0 otherwise.
- **Learning rate**: a small positive number that controls how large each weight update is.
- **Epoch**: one full pass over the training set.
- **Linearly separable**: a dataset that a single straight line or hyperplane can split into its classes with no errors.
- **Decision boundary**: the set of inputs where the model switches from one class to the other.
- **XOR**: the exclusive-or function, which outputs 1 when exactly one of its two inputs is 1.
- **Multi-layer perceptron (MLP)**: a network of several dense layers, with a nonlinear activation between them.
- **Hidden layer**: a layer between the input and the output whose values are not part of the training labels.
- **Representation**: the way a model encodes its input internally, typically as the activations of a hidden layer.
- **Universal approximation theorem**: a result that says a network with one hidden layer can approximate a wide class of functions to any accuracy, given enough neurons.
- **Feature**: a number that describes an input, such as a pixel value or a sensor reading.

---

## Perceptron

Imagine a sorting machine on a factory line that sends red parts left and blue parts right. It looks at a few measurements of each part and adds them up with a weight for each measurement. If the total is above a threshold, it sends the part right. When the machine sends a part the wrong way, a technician turns the weights a little toward the correct answer. Over many parts, the machine gets better. A perceptron is that machine in code.

From first principles, a perceptron computes z = w . x + b, then outputs y_hat = step(z), where step returns 1 if z is zero or positive and 0 otherwise. Given a labeled example (x, y), the learning rule is simple. If the prediction is right, change nothing. If it is wrong, update the weights by w = w + lr (y - y_hat) x and the bias by b = b + lr (y - y_hat). The difference y minus y_hat is either +1, when the neuron should have fired and did not, or -1, when it fired and should not have. The weights move in the direction of the input, which pushes the output toward the correct value for that example.

```python
import numpy as np

def step(z):
    return np.where(z >= 0, 1, 0)

w = np.array([0.0, 0.0])
b = 0.0
lr = 0.1

x, y = np.array([1.0, 1.0]), 1       # one example labeled 1
y_hat = int(step(x @ w + b))         # the neuron outputs 1 at z = 0
print(y_hat, y - y_hat)              # 1 0, no mistake, no update
```

The perceptron convergence theorem says that if the data are linearly separable, this rule finds a separating line in a finite number of updates. The number of updates depends on the margin between the classes and the size of the inputs, not on the learning rate, which only scales the step.

The mistake most beginners make is to expect the perceptron to learn when the data are not separable. It will keep updating forever, flipping the same few examples back and forth, and the error count will never reach zero. The fix is to recognize the situation, not to train longer. You can detect it by watching the number of errors per epoch. A count that stops decreasing and never reaches zero is a signal that the model class is too simple.

In industry, the perceptron is rarely used alone, but its close relatives are everywhere. Logistic regression is a perceptron with a smooth activation and a different loss, and it still runs in credit scoring, spam filtering and A/B test analysis, where a linear decision boundary is often exactly the goal.

---

## Linear separability

Think about sorting marbles on a table with a ruler. If the red and blue marbles are on opposite sides of the ruler, you can split them with a single straight cut. If they are mixed in a checkerboard, no straight cut works, and you need a curved path or several cuts. A dataset is linearly separable when one straight cut is enough.

From first principles, a linear decision rule in two dimensions is the line w1 x1 + w2 x2 + b = 0. In d dimensions, it is a hyperplane, which is a flat surface one dimension lower than the space. A dataset is linearly separable if there exist weights w and bias b such that every positive example has w . x + b greater than zero and every negative example has it below zero. The AND function is separable: the single positive example (1, 1) lies on one side of the line x1 + x2 = 1.5, and the three negative examples lie on the other side. The XOR function is not separable, because the two positive examples (0, 1) and (1, 0) lie on opposite corners of the square, and any line that puts one on the positive side also puts the other on the negative side.

```
  x2
  1 |  0 *        * 1
    |
    |
  0 |  1 *        * 0
    +-------------------- x1
       0            1

  AND: one line separates the 1 from the three 0s.
  XOR: no single line separates the 1s from the 0s.
```

A useful test is to sketch the data on paper before training. If you can draw a straight line by eye, a perceptron will find one. If you cannot, you need a different model. The test is quick and it saves hours of training a model that cannot succeed.

The limitation is real, and it does not disappear with more data or more training time. A perceptron with ten thousand examples of XOR still fails. The fix is a richer model, and more training will not help. Do not treat the failure as a bug in the code. When your perceptron never converges, check the geometry before you check the implementation.

On the job, linear separability is a diagnostic. A baseline logistic regression that scores far below a random forest often means the classes overlap in the raw features in a nonlinear way, so you should add features or try a model that can bend its boundary.

---

## Multi-layer perceptrons

Consider the XOR problem again. The trick is to stop asking one neuron to do everything. An OR neuron is easy, since it is linearly separable. A NAND neuron is also easy. The AND of those two results is also easy. If you can compute OR and NAND of the inputs, a third neuron can combine them into XOR. Stacking neurons this way is exactly what a multi-layer perceptron does.

From first principles, an MLP with one hidden layer computes h = step(X W1 + b1) and then y = step(h W2 + b2), with W1 of shape (2, 2) and W2 of shape (2, 1) for two inputs and two hidden neurons. Each hidden neuron draws its own line in the input space. The output neuron then draws a line in the space of the hidden values, not in the original space. Because the hidden values are a nonlinear function of the inputs, a line in hidden space can be a curve in input space.

The forward pass is the one you wrote on Day 39, with a step activation in place of the smooth ones. The new idea is the composition. The output of one layer becomes the input of the next, and the parameters of every layer together define the function.

```python
import numpy as np

def step(z):
    return np.where(z >= 0, 1, 0)

# Hidden neuron 1 computes OR: fires when x1 + x2 >= 0.5
# Hidden neuron 2 computes NAND: fires when x1 + x2 <= 1.5
W1 = np.array([[1.0, -1.0],
               [1.0, -1.0]])        # shape (2 inputs, 2 hidden)
b1 = np.array([-0.5, 1.5])

# Output neuron computes AND of the two hidden values
W2 = np.array([[1.0],
               [1.0]])              # shape (2 hidden, 1 output)
b2 = np.array([-1.5])

X = np.array([[0, 0], [0, 1], [1, 0], [1, 1]])
H = step(X @ W1 + b1)
Y = step(H @ W2 + b2).ravel()
print(Y)                             # [0 1 1 0]
```

Read W1 by column. Each column holds the weights of one hidden neuron: the first column, (1, 1), is the OR neuron, and the second column, (-1, -1), is the NAND neuron. Each row holds the weights that one input sends to both hidden neurons. Printing `W1[:, 0]` and `W1[:, 1]` is a quick way to check that the layout is what you intended.

Do not assume that adding a hidden layer always helps. An MLP with too many hidden neurons can memorize the training set, and an MLP with too few cannot represent the boundary at all. In industry, the number of hidden units is chosen by experiment on a validation set, starting from a published architecture rather than from first principles. The universal approximation theorem tells you a large enough network can represent the function, but it does not tell you how many neurons to use or how to find the weights.

---

## Hidden layers

A hidden layer is any layer between the input and the output. Its values are never given as labels, so the network has to invent them as useful intermediate features. In the XOR example, the two hidden neurons invent the features "at least one input is on" and "not both inputs are on". Neither feature was in the data, and together they make the problem easy.

From first principles, each hidden neuron defines one boundary in the input space. Two hidden neurons define two lines, which divide the plane into up to four regions. The output neuron then uses those regions. The more hidden neurons you add, the more lines you have, and the more complicated the regions can become. This is the sense in which hidden layers bend the boundary.

The cost is capacity and training difficulty. Each hidden neuron has its own parameters, and the number of parameters grows with the width of the layers. More parameters mean more ways to fit noise. Deep networks with many hidden layers train slowly unless the activations, the initialization and the optimizer are chosen well, which you will study on Days 41, 44 and 45.

```
Hidden space, after OR and NAND                 Label

h2
1 |  (0,1)              (1,1)          0 at (0,1), 1 at (1,1)
  |                                    (both inputs 01 and 10 land here)
  |
0 |                     (1,0)          0 at (1,0)
  +----------------------------- h1
        0                  1
```

The sketch shows the idea. In the input space the two classes sit on opposite corners of a square, so no line separates them. In the hidden space the two points labeled 1 land on the same spot, (1, 1), and a single line such as h1 + h2 = 1.5 separates that spot from the two 0s. The table in the next section gives the exact coordinates.

A common mistake is to treat a hidden layer as a mysterious box. You can inspect it. Print the hidden activations for each input, plot them and see whether the classes separate. In a real project, the same check is done with a t-SNE or UMAP plot of hidden activations, which shows whether the network has learned a useful representation.

Hidden layers show up in every production network. A ResNet image classifier has dozens of them, a transformer has many attention and feed-forward blocks, and each one is a hidden layer by the definition above.

---

## Representation learning

Representation learning is the idea that a good model is one that transforms the raw input into a form where the task is easy. In the XOR example, the hidden layer does not change the labels. It changes the coordinates. The same four points that were arranged in a square become points that a single line separates. The output layer then solves an easy problem.

From first principles, the representation is the function that maps the input x to the hidden vector h. For the hand-wired network, h = (OR(x), NAND(x)). Here is the table you can compute by hand or by running the code:

| Input (x1, x2) | Hidden (h1, h2) | Output y | Label |
|---|---|---|---|
| (0, 0) | (0, 1) | 0 | 0 |
| (0, 1) | (1, 1) | 1 | 1 |
| (1, 0) | (1, 1) | 1 | 1 |
| (1, 1) | (1, 0) | 0 | 0 |

In the hidden space, the points labeled 1 both map to (1, 1), and the points labeled 0 map to (0, 1) and (1, 0). The line h1 + h2 = 1.5 separates them.

Representation learning becomes powerful when the network learns the hidden features from data rather than having them wired by hand. In that case, the features are not chosen by a person, and they can capture patterns no engineer would think of. The limitation is that learned representations are hard to interpret, and they can encode biases in the training data without anyone noticing. You will see both sides when you train networks on Days 50 and 51.

Do not assume that learned representations are automatically good. A network can achieve high accuracy by exploiting a shortcut in the data, such as a background color in an image, and its representation will reflect that shortcut. Checking the representation on new data and on slices of the data is part of the job.

In the real market, representation learning is the reason pretrained models are valuable. Companies fine-tune a model whose early layers already encode useful features, such as edges in images or word meanings in text, and they reuse those features on their own task. Transfer learning, which you will meet on Day 57, depends on this idea.

---

## Hands-on exercises

1. **Hand perceptron.** Start with weights (0, 0) and bias 0 on the AND data. Run one epoch of the learning rule by hand with learning rate 1. Write down each weight after each example. Check your final weights against the code in the project.
2. **Separability sketch.** Draw the four points of XOR on graph paper and try to draw a single line that separates the 1s from the 0s. Write one sentence on why you cannot.
3. **Error counts.** Train the perceptron on the OR function and print the number of errors per epoch. Confirm that it reaches zero. Then train it on XOR for 200 epochs and plot the error count. Describe the shape of the curve.
4. **Hidden neurons by hand.** Modify the hand-wired network so the second hidden neuron computes AND instead of NAND. Predict whether the output will still be XOR. Check your answer with code and explain the result.
5. **Ten-line MLP.** Write a function that takes a 2 by 2 weight matrix, a bias vector and an input matrix, and returns the hidden activations using the step function. Verify it on the four XOR inputs.

---

## Project: perceptron, XOR and a hand-wired MLP

**Goal:** implement the perceptron learning rule, show that it learns AND and fails on XOR, then build a hand-wired MLP that computes XOR and print the hidden representation. Test each claim, and write up what the failure and the fix teach you about model capacity.

**Repo name:** `100-days-ai-day-40-perceptrons-and-mlps`

**Repo layout:**

```
100-days-ai-day-40-perceptrons-and-mlps/
├── README.md
├── requirements.txt
├── src/
│   └── perceptron_lab/
│       ├── __init__.py
│       ├── data.py
│       ├── perceptron.py
│       └── mlp.py
├── tests/
│   ├── test_perceptron.py
│   └── test_mlp.py
└── experiments/
    └── xor_error_curve.py
```

**Starter code.** Create `requirements.txt` with the pinned packages from the requirements section. Then create the files below.

`src/perceptron_lab/data.py`:

```python
import numpy as np

X = np.array([[0, 0],
              [0, 1],
              [1, 0],
              [1, 1]], dtype=float)

Y_AND = np.array([0, 0, 0, 1])
Y_OR = np.array([0, 1, 1, 1])
Y_XOR = np.array([0, 1, 1, 0])
```

`src/perceptron_lab/perceptron.py`:

```python
import numpy as np


def step(z):
    return np.where(z >= 0, 1, 0)


class Perceptron:
    def __init__(self, n_features, lr=0.1):
        self.w = np.zeros(n_features)
        self.b = 0.0
        self.lr = lr

    def predict(self, X):
        return step(X @ self.w + self.b)

    def fit(self, X, y, epochs=100):
        """Return the number of mistakes made in each epoch."""
        mistakes_per_epoch = []
        for _ in range(epochs):
            mistakes = 0
            for xi, target in zip(X, y):
                pred = int(step(xi @ self.w + self.b))
                update = self.lr * (target - pred)
                if update != 0:
                    self.w = self.w + update * xi
                    self.b = self.b + update
                    mistakes += 1
            mistakes_per_epoch.append(mistakes)
            if mistakes == 0:
                break
        return mistakes_per_epoch
```

`src/perceptron_lab/mlp.py`:

```python
import numpy as np

from perceptron_lab.perceptron import step


def xor_mlp_forward(X):
    """Hand-wired two-layer network. Hidden neurons compute OR and NAND."""
    W1 = np.array([[1.0, -1.0],
                   [1.0, -1.0]])
    b1 = np.array([-0.5, 1.5])
    W2 = np.array([[1.0],
                   [1.0]])
    b2 = np.array([-1.5])
    H = step(X @ W1 + b1)
    Y = step(H @ W2 + b2).ravel()
    return H, Y
```

Before you go on, check `W1` by printing its columns. Column 0 should be the weights of the OR neuron, and column 1 should be the weights of the NAND neuron. If your hidden activations do not match the table in the lesson, fix the matrix first.

**Step-by-step tasks:**

1. Create the repo, initialize Git, create a virtual environment and install the pinned packages.
2. Write `tests/test_perceptron.py` with three tests: the perceptron reaches zero mistakes on AND within 100 epochs, the perceptron on OR reaches zero mistakes, and the perceptron on XOR still has at least one mistake after 500 epochs.
3. Write `tests/test_mlp.py` with one test that checks the output of `xor_mlp_forward` against `Y_XOR` for all four inputs, and one test that checks the hidden activations match the table in the lesson.
4. Run `pytest` from the repo root with `PYTHONPATH=src`.
5. Write `experiments/xor_error_curve.py` that trains on XOR for 200 epochs and saves a plot of mistakes per epoch to `results/xor_errors.png`. Describe the curve in the README.
6. Change the learning rate to 0.01 and then to 1.0 and check whether XOR still fails. Explain in one sentence why the answer does not change.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the experiment
- [ ] The hidden representation table, produced by your code
- [ ] The XOR error curve, with one paragraph describing it
- [ ] A short note on what the perceptron can and cannot learn, in your own words

**Done when:**

- The perceptron tests pass, and the XOR test fails if you change the expectation to zero mistakes
- The hidden layer test matches the representation table exactly
- The plot of XOR mistakes is saved and you can explain why it does not reach zero
- You can state, without notes, why a hidden layer makes XOR separable
- The repo has at least four commits

**Evidence to commit to GitHub:** the source, the tests, the README with the hidden representation table and the error curve image, and a `NOTES.md` file that records one wrong hypothesis you had about the failure and what disproved it. The image and the notes are the evidence that you understand the result, not only that the code runs.

**Why this project is practical:** it is a small experiment with a clear pass and fail, which is how you test any learning system before you trust it.

---

## Check your understanding

1. The perceptron learning rule makes no update when the prediction is correct. Why does this rule still move the decision boundary toward the correct answer for the misclassified examples?
2. Explain in your own words why the XOR function is not linearly separable, using the four points and a line.
3. Suppose you add a second output neuron to the hand-wired MLP. What would you need to change in the output layer, and what would the new network compute?
4. A teammate says that adding more epochs will eventually make the perceptron learn XOR. Give the argument that refutes this claim.
5. A hidden layer of ten neurons is trained on a dataset of 50 examples. Name two risks and one check you would run before trusting the model.
6. Describe a case in which a linear model is the right choice even though a deeper network could fit the data better.

**Answers:**

1. A misclassified example has a nonzero difference between the target and the prediction, so the update adds a multiple of the input to the weights. For a positive example that was predicted negative, the update increases the dot product with that input, which moves the boundary so the example falls on the correct side. The rule therefore corrects each mistake directly, and it stops when there are none.
2. For XOR, the two positive points (0, 1) and (1, 0) lie on opposite corners of the unit square, and the two negative points (0, 0) and (1, 1) lie on the other two corners. Any line that separates the positives from the negatives must have the two positives on one side and the negatives on the other. The midpoint of the two positives is (0.5, 0.5), which is also the midpoint of the two negatives, so no line can put the two groups on opposite sides.
3. You would need to split the output into two neurons, each with its own weights and bias, and you would then have two outputs for each input. Whether the new network computes something new depends on what the neurons learn, and with hand-set weights it would compute two boolean functions of the hidden values.
4. The perceptron convergence theorem only applies to separable data. For a dataset that is not separable, no weight vector gives zero errors, so the rule never reaches zero no matter how many epochs run. The errors keep cycling, and more epochs only add more cycling.
5. The risks are overfitting, since the network has many parameters for 50 examples, and an unstable estimate of performance, since the test set is tiny. A good check is to hold out part of the data with cross-validation and to compare against a linear model on the same split.
6. When the classes are separated by a line in the chosen features, a linear model is simpler, faster to train, easier to explain and less likely to overfit. Credit scoring teams often choose logistic regression for exactly those reasons, and they add features to capture nonlinearity instead of adding hidden layers.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Perceptron", for the definition, the learning rule and the convergence result
- Wikipedia, "Linear separability", for the formal definition used in this lesson
- Wikipedia, "Multilayer perceptron", for the architecture and its training
- Wikipedia, "XOR gate", for the logic function and its history
- Wikipedia, "Universal approximation theorem", for what hidden layers can represent
- Wikipedia, "Representation learning", for the broader idea behind the hidden representation

**Tutorials**

- GeeksforGeeks, search for "Perceptron Learning Algorithm" for a worked version of the update rule
- NumPy documentation, "numpy.where", at numpy.org/doc/stable/reference/generated/numpy.where.html, for the vectorized step function
- scikit-learn documentation, "Perceptron", at scikit-learn.org, which shows the same model implemented in a library

**Watch**

- MIT OpenCourseWare, 6.034 Artificial Intelligence, the lecture on neural nets taught by Patrick Winston
- StatQuest with Josh Starmer, "Neural Networks Pt. 1: Inside the Black Box", on YouTube

**Practice**

- TensorFlow Playground at playground.tensorflow.org. Choose the XOR dataset, remove all hidden layers, train, then add one hidden layer with two neurons and train again.

---

## Tomorrow

Day 41 asks what the step function should be replaced with. The step function you used today has zero slope everywhere except at the threshold, which makes learning from errors impossible in a deep network. Day 41 compares sigmoid, tanh, ReLU, leaky ReLU, GELU and softmax, and it shows why a smooth nonlinearity with a usable gradient is necessary once you stack many layers. The hidden layer you wired today becomes the place where those choices matter.

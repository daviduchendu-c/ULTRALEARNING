# Day 42: Loss Functions

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3 hours
**Level:** intermediate. You can compute a forward pass, apply softmax and take a logarithm. Today you need the derivative of a log and the idea that a loss is a number to minimize.

**By the end of this day, you will have:**

- A NumPy module with four losses (mean squared error, binary cross-entropy, categorical cross-entropy from logits, and negative log likelihood) and their gradients
- Tests that check each loss against a value you computed by hand and each gradient against a finite-difference estimate
- A measured comparison showing why cross-entropy keeps a large gradient for a confident wrong answer when mean squared error does not
- A table that matches each task type to an output activation and a loss

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 13: probability, the idea of a probability distribution and conditional probability
- Day 14: likelihood and maximum likelihood, at the level of the definitions
- Day 15: entropy and cross-entropy as measures of surprise
- Day 41: sigmoid, softmax and their derivatives

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [MSE](#mse)
- [Binary cross-entropy](#binary-cross-entropy)
- [Categorical cross-entropy](#categorical-cross-entropy)
- [Negative log likelihood](#negative-log-likelihood)
- [Task and loss alignment](#task-and-loss-alignment)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a loss library with gradient checks](#project-a-loss-library-with-gradient-checks)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A neural network produces numbers. A loss function turns those numbers into one score that says how bad the answer was, and training is the search for weights that make that score small. The loss is the only place where the task enters the learning process. If the loss does not describe what you want, the network will optimize something else very well.

That is why the loss is a design decision you have to make. Predicting a house price and predicting whether a transaction is fraudulent need different losses, even when the network architecture looks the same. The loss also shapes the gradient, which is the signal that tells each weight which way to move. A poorly chosen loss can produce a gradient that is too small to learn from when the model is most wrong, which is the failure this lesson measures.

On the job, the loss is often the first thing to check when a model trains without improving. Engineers look at the loss curve, compare the loss against a trivial baseline, and confirm that the output activation and the loss match the target format. Many subtle bugs come from a mismatch, such as applying softmax and then cross-entropy that applies softmax again, or using mean squared error for a yes-or-no label.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1795 to 1809 | Carl Friedrich Gauss and Adrien-Marie Legendre develop least squares for fitting orbits and measurements. | The origin of mean squared error and of the squared-error criterion that still drives regression. |
| 1948 | Claude Shannon publishes "A Mathematical Theory of Communication", which defines entropy as a measure of information. | The basis for cross-entropy, which measures the cost of coding one distribution with another. |
| 1950s to 1960s | Statisticians and engineers use logistic regression with the log loss to model binary outcomes. | The binary cross-entropy you use in classification today. |
| 1986 | Rumelhart, Hinton and Williams use squared error in their backpropagation experiments. | Mean squared error becomes the default loss for early neural networks. |
| 2010s | Cross-entropy with softmax becomes the standard loss for deep image classifiers. | The loss used in the ImageNet models that drove the modern deep learning wave. |

**Takeaway:** most losses come from two ideas, squared error from measurement and cross-entropy from information. Each suits a different kind of output.

---

## Glossary

- **Loss function**: a function that takes the prediction and the true label and returns a single number that training tries to minimize.
- **Cost function**: a loss averaged over a dataset or a batch, which is the value the optimizer reduces.
- **Residual**: the difference between the true value and the prediction, such as y minus y_hat.
- **Mean squared error (MSE)**: the average of the squared residuals.
- **Binary cross-entropy**: the loss for one yes-or-no output, given a probability p and a label y in {0, 1}.
- **Categorical cross-entropy**: the loss for one of several classes, given a probability distribution over the classes and a one-hot label.
- **One-hot vector**: a vector with a single 1 at the true class and zeros elsewhere.
- **Negative log likelihood (NLL)**: the negative logarithm of the probability the model assigns to the true class.
- **Logit**: the raw score before a sigmoid or softmax converts it to a probability.
- **Log-sum-exp**: the function log(sum of exp(z_i)), computed in a stable way to avoid overflow.
- **Clipping**: limiting a value to a range, such as keeping a probability between a small epsilon and one minus epsilon so the logarithm stays finite.
- **Calibration**: the property that a model's predicted probability matches the observed frequency of the outcome.

---

## MSE

Imagine you are measuring the distance to a lighthouse with a rangefinder. Each measurement is off by some amount. If you average the errors, positive and negative errors cancel, which hides a bad measurement. If you average the squares of the errors, every error counts and large errors count much more. Mean squared error does this.

From first principles, for n examples with predictions y_hat_i and true values y_i, MSE = (1/n) sum of (y_i - y_hat_i)^2. Its gradient with respect to a prediction is 2 (y_hat_i - y_i) / n. The gradient is linear in the residual, which is a convenient property. MSE is the right loss when the target is a real number and errors on the two sides are equally undesirable.

```python
import numpy as np

def mse(y_pred, y_true):
    return float(np.mean((y_pred - y_true) ** 2))

y_true = np.array([1.0, 2.0])
y_pred = np.array([0.0, 0.0])
print(mse(y_pred, y_true))           # (1 + 4) / 2 = 2.5
```

The limitation is that squared error is sensitive to outliers. A single label that is wrong by 100 contributes 10,000 to the sum, and it can dominate training. When outliers are expected, the Huber loss or the absolute error is safer, and the choice should be made deliberately.

Using MSE on a sigmoid output for a yes-or-no label is a serious error in classification. The loss is not wrong in principle, but its gradient vanishes when the sigmoid saturates, and a confident wrong prediction then barely moves the weights. The comparison later in this lesson shows the effect with numbers. In the real market, MSE is the default for regression tasks such as demand forecasting, sensor calibration and price prediction, where the target is continuous.

---

## Binary cross-entropy

Think about betting on a coin toss. If you say the chance of heads is 0.99 and the coin lands tails, you lose a lot, and the loss grows without limit as your stated probability approaches certainty and the answer is wrong. Binary cross-entropy is the penalty that makes this bet honest.

From first principles, for a predicted probability p in (0, 1) and a label y in {0, 1}, the loss for one example is -(y log p + (1 - y) log(1 - p)). When y = 1, the loss is -log p, which is zero at p = 1 and grows to infinity as p approaches 0. The mean over a batch is the average of these losses. The gradient with respect to p is (p - y) / (p (1 - p)), which is large when p is close to 0 or 1 and the label disagrees. The clean gradient appears when you take the derivative with respect to the logit z instead, where p = sigmoid(z): the gradient becomes p - y, as the project shows.

```python
import numpy as np

def binary_cross_entropy(p, y):
    p = np.clip(p, 1e-12, 1.0 - 1e-12)
    return float(-np.mean(y * np.log(p) + (1 - y) * np.log(1 - p)))

print(binary_cross_entropy(np.array([0.5]), np.array([1.0])))   # 0.6931 = ln 2
```

The limitation is that clipping is a hack. If p is exactly 0 or 1, the logarithm is undefined, so you clip, and the clip changes the gradient in that region. The stable approach is to compute the loss from logits with a log-sum-exp style formula, which frameworks provide as a single function. Use the logit form in production code.

Do not apply a sigmoid and then pass the probability to a loss that expects logits, or the reverse. Both are silent errors: the model trains, and the accuracy is wrong. Check the name of the loss in the documentation before you use it. In the real market, binary cross-entropy is the loss behind fraud detection, click prediction, spam filters and any model that outputs a yes-or-no probability.

---

## Categorical cross-entropy

Consider a quiz show where the contestant must name the one correct answer among several. Each answer gets a confidence score, and the score for the right answer is what matters. If the contestant put 0.01 on the right answer, the penalty is large. If the contestant put 0.9, the penalty is small. Categorical cross-entropy is that penalty for several classes.

From first principles, if the model produces a probability distribution p over K classes and the true class is c, the loss is -log p_c. With a one-hot label y, this is -sum over k of y_k log p_k, which is the general form. In practice you compute it from logits z, because the log of a softmax has a stable form: log p_c = z_c - log-sum-exp(z). The gradient with respect to the logits is p - y, the predicted distribution minus the one-hot label, divided by the batch size when the loss is averaged.

```python
import numpy as np

def log_softmax(z):
    shifted = z - z.max(axis=1, keepdims=True)
    return shifted - np.log(np.exp(shifted).sum(axis=1, keepdims=True))

def cross_entropy_from_logits(logits, labels):
    logp = log_softmax(logits)
    return float(-np.mean(logp[np.arange(len(labels)), labels]))

logits = np.zeros((1, 3))            # uniform over 3 classes
print(cross_entropy_from_logits(logits, np.array([0])))   # ln 3, about 1.0986
```

The limitation is that cross-entropy assumes the classes are mutually exclusive. Label smoothing, which replaces the one-hot target with a slightly softened distribution, is a common fix when labels are noisy, and it changes the minimum the model reaches. Do not apply softmax in the model and then call a loss that applies softmax again. The model will still learn, but the probabilities will be wrong and the loss will not decrease as expected.

On the job, categorical cross-entropy is used for every multi-class classifier, and it is the loss that language models use when they predict the next token from a vocabulary of tens of thousands of options.

---

## Negative log likelihood

Imagine you are a detective who asks, for each suspect, how probable the evidence would be if that person were guilty. The most likely suspect is the one who makes the observed evidence most probable. Maximum likelihood is that rule, and the negative log likelihood is the loss you minimize to apply it.

From first principles, if a model assigns probability p_i(y_i) to the true label of example i, the likelihood of the dataset is the product of those probabilities. Taking the logarithm turns the product into a sum, and taking the negative turns maximization into minimization. The result is the average negative log likelihood, -(1/n) sum of log p_i(y_i). Categorical cross-entropy with a one-hot label is exactly this quantity, which is why the two names are used interchangeably in many libraries. In PyTorch, the function `nn.NLLLoss` expects log probabilities as input, and `nn.CrossEntropyLoss` expects logits and applies log softmax for you.

```python
import numpy as np

probs = np.array([[0.7, 0.2, 0.1],
                  [0.1, 0.8, 0.1]])
labels = np.array([0, 1])
nll = -np.mean(np.log(probs[np.arange(2), labels]))
print(round(nll, 4))                 # about 0.2899
```

The limitation is the same as for cross-entropy: the loss only sees the probability of the true class, so it does not directly reward a well-spread distribution over the wrong classes. A model can improve its loss by making the true class slightly more probable and still be poorly calibrated. Calibration checks, such as reliability plots, are needed in addition.

Do not pass probabilities to a loss that expects log probabilities, or log probabilities to a loss that expects logits. Read the input convention of your framework's loss function. In industry, NLL is the workhorse of language modeling and of any system trained on a likelihood, including speech recognition and probabilistic forecasting.

---

## Task and loss alignment

A loss is aligned with a task when minimizing it makes the model better at the thing you care about. Alignment has three parts: the output format must match the loss, the loss must reflect the real cost of errors, and the metric you report must agree with the loss on a validation set.

From first principles, the output format is fixed by the loss. Regression with MSE expects a real number with no activation, or a linear output. Binary classification with binary cross-entropy expects a logit, which becomes a probability through a sigmoid. Multi-class classification with categorical cross-entropy expects one logit per class, which becomes a distribution through softmax. The cost of errors is a design choice: a missed fraud case may cost far more than a false alarm, and then a weighted loss or a threshold chosen on the validation set is needed.

| Task | Output | Activation | Loss |
|---|---|---|---|
| Predict a real value | one number | none | MSE, or Huber for outliers |
| Yes or no, one label | one logit | sigmoid | binary cross-entropy from logits |
| One of K classes | K logits | softmax | categorical cross-entropy |
| Several independent tags | K logits | sigmoid per tag | binary cross-entropy per tag |

A loss that does not match the reported metric is the hardest error to notice. A model trained with cross-entropy may still have poor accuracy at the chosen threshold, and a model trained with MSE on a probability may rank examples badly. Always report the metric you care about, compare it with the loss curve, and set the decision threshold on data you did not train on.

In the real market, alignment is one of the first questions in a model review: what is the loss, what is the metric, and what business cost does a wrong answer carry? A clear answer prevents months of training a model that scores well on paper and fails in production.

---

## Hands-on exercises

1. **Hand MSE.** Compute the MSE of predictions (2, 0, 1) against labels (1, 1, 1) by hand. Check your answer in Python.
2. **Hand BCE.** Compute the binary cross-entropy for p = 0.8 with y = 1 and for p = 0.8 with y = 0. Compare the two values and explain why they differ.
3. **Uniform guess.** For a K-class problem with uniform logits, the categorical cross-entropy equals ln K. Confirm this for K = 2, 5 and 10.
4. **Log-sum-exp stability.** Compute log_softmax of [1000, 1001, 1002] with the shift and without it. Report which version returns finite values.
5. **Alignment table.** For three real problems you know, write the output format, the activation and the loss. For each, name one cost of a wrong answer and say whether a symmetric loss is appropriate.

---

## Project: a loss library with gradient checks

**Goal:** implement four losses and their gradients, check each gradient numerically, and compare the gradient of MSE on a sigmoid output with the gradient of binary cross-entropy for a confident wrong prediction.

**Repo name:** `100-days-ai-day-42-loss-functions`

**Repo layout:**

```
100-days-ai-day-42-loss-functions/
├── README.md
├── requirements.txt
├── src/
│   └── losses/
│       ├── __init__.py
│       └── losses.py
├── tests/
│   ├── test_values.py
│   └── test_gradients.py
└── experiments/
    └── saturation_compare.py
```

**Starter code.** Create `requirements.txt` with the pinned packages from the requirements section. Create `src/losses/losses.py` with the following content:

```python
import numpy as np

EPS = 1e-12


def mse(y_pred, y_true):
    return float(np.mean((y_pred - y_true) ** 2))


def mse_grad(y_pred, y_true):
    return 2.0 * (y_pred - y_true) / y_pred.size


def binary_cross_entropy(p, y):
    p = np.clip(p, EPS, 1.0 - EPS)
    return float(-np.mean(y * np.log(p) + (1 - y) * np.log(1 - p)))


def binary_cross_entropy_grad(p, y):
    p = np.clip(p, EPS, 1.0 - EPS)
    return (p - y) / (p * (1.0 - p)) / p.size


def log_softmax(z):
    shifted = z - z.max(axis=1, keepdims=True)
    return shifted - np.log(np.exp(shifted).sum(axis=1, keepdims=True))


def cross_entropy_from_logits(logits, labels):
    logp = log_softmax(logits)
    return float(-np.mean(logp[np.arange(len(labels)), labels]))


def cross_entropy_from_logits_grad(logits, labels):
    p = np.exp(log_softmax(logits))
    p[np.arange(len(labels)), labels] -= 1.0
    return p / len(labels)
```

**Step-by-step tasks:**

1. Create the repo, initialize Git, create a virtual environment and install the pinned packages.
2. Write `tests/test_values.py`. Check that `mse([0, 0], [1, 2])` equals 2.5, that `binary_cross_entropy(0.5, 1)` equals ln 2, and that `cross_entropy_from_logits(zeros((1, 3)), [0])` equals ln 3.
3. Write `tests/test_gradients.py`. For each loss, use a central difference with h = 1e-6 on a small random input, and assert the analytic gradient matches within 1e-5 in relative terms. For the cross-entropy gradient, perturb one logit at a time.
4. Add a test that cross-entropy from logits stays finite for logits of size 1000.
5. Write `experiments/saturation_compare.py`. Use a single logit z = -4.6 with label y = 1. Compute p = sigmoid(z). Compare the gradient of MSE with respect to z, which is 2 (p - y) p (1 - p), with the gradient of binary cross-entropy with respect to z, which is p - y. Print both values and their ratio.
6. In the README, explain why the ratio is so large and what it means for training when the model is confidently wrong.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and the experiment
- [ ] The table of values from `test_values.py`
- [ ] The saturation comparison output, pasted as a code block
- [ ] A paragraph on task and loss alignment, with one example from your own work or from a public dataset

**Done when:**

- All value and gradient tests pass, and one test fails if you flip the sign of a gradient
- The saturation comparison prints the two gradients, and you can explain the ratio without notes
- The cross-entropy test passes for large logits without NaN
- The README contains the alignment paragraph
- The repo has at least four commits

**Evidence to commit to GitHub:** the source, the tests, the README with the value table and the saturation output, and a `NOTES.md` that records one alignment mistake you could imagine making on a real project and the test that would catch it. The notes are evidence that you can connect the loss to the task, not only compute it.

**Why this project is light on the model side:** the loss is a small function, and the project is about checking the math and the numerics. The full training loop comes on Day 50, where you will use these ideas in practice.

---

## Check your understanding

1. Why does squaring the residual make MSE sensitive to outliers, and what would you change in the loss if a few labels are known to be unreliable?
2. Using the chain rule, show that the gradient of binary cross-entropy with respect to the logit z, where p = sigmoid(z), is p - y.
3. A colleague applies softmax in the last layer and then passes the result to a loss that applies softmax again. What happens to the probabilities, and why might the model still train?
4. Explain why negative log likelihood and categorical cross-entropy give the same value when the label is one-hot.
5. A fraud model has 0.1 percent positive cases. A loss-based check shows the cross-entropy falling, but the team's recall on fraud is poor. What could explain the gap, and what would you change first?
6. Give an example where mean squared error is the right loss for a classification-like problem, and explain why it is still reasonable there.

**Answers:**

1. Squaring makes a large residual contribute much more than a small one. A label that is wrong by a large amount dominates the sum. For unreliable labels, use a robust loss such as Huber, drop or down-weight the suspect examples, or relabel them after review.
2. With p = sigmoid(z), dp/dz = p (1 - p). The loss derivative with respect to p is (p - y) / (p (1 - p)). Multiplying by dp/dz gives (p - y) p (1 - p) / (p (1 - p)) = p - y.
3. The second softmax is applied to a distribution that is already close to uniform or already normalized, so the outputs become flatter than the model intended. The loss still decreases because the gradient points in a consistent direction, but the probabilities are miscalibrated and the training signal is weaker than it should be.
4. For a one-hot label y, the sum of y_k log p_k has only one nonzero term, the true class. So the cross-entropy is minus log p of the true class, which is exactly the negative log likelihood of that label.
5. A class imbalance of 0.1 percent means that a model predicting no fraud at all has low cross-entropy. The loss is dominated by the majority class. The first change to test is class weighting or resampling, followed by choosing a decision threshold on a validation set that favors recall.
6. Predicting a score between 0 and 1 for a rating scale, such as a star rating, can use MSE, because the ordering of the categories matters and squared distance respects it. It is still a regression on an ordinal target, and the choice should be checked against a metric that rewards near misses.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Loss function", for the definition and common examples
- Wikipedia, "Mean squared error", for the formula and its properties
- Wikipedia, "Cross-entropy", for the information-theoretic view and the binary case
- Wikipedia, "Maximum likelihood estimation", for the link between likelihood and the loss
- Wikipedia, "Entropy (information theory)", for the background that cross-entropy builds on
- Stanford CS231n course notes, "Linear Classification" and "Optimization", at cs231n.github.io, for the softmax loss and its gradient

**Tutorials**

- PyTorch documentation, the torch.nn loss functions page, at pytorch.org/docs/stable, which names the input conventions of `CrossEntropyLoss`, `NLLLoss` and `BCEWithLogitsLoss`
- Google, "Machine Learning Crash Course", at developers.google.com/machine-learning/crash-course, the section on logistic regression and loss
- GeeksforGeeks, search for "Cross Entropy Loss" for a worked numeric example

**Watch**

- StatQuest with Josh Starmer, the video on cross entropy, found by searching YouTube for "StatQuest cross entropy"
- 3Blue1Brown, "Gradient descent, how neural networks learn | Deep learning chapter 2", on YouTube, for why the loss surface matters

**Practice**

- Google Machine Learning Crash Course exercises on logistic regression, which let you change the loss and see the effect on the model

---

## Tomorrow

Day 43 uses the loss you can now compute to find the gradient of every weight in a network. You will build a computational graph, apply the chain rule backward from the loss and update the parameters. The gradient of cross-entropy with respect to the logits, p minus y, is the first step of that backward pass, so today's project is the starting point for the tiny network you will write by hand tomorrow.

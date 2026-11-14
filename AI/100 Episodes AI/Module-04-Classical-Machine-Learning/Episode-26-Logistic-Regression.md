# Day 26: Logistic Regression

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You should have finished Day 25, since today reuses its weighted sum, its gradient and its training loop.

**By the end of this day, you will have:**

- An understanding of why a linear score needs the sigmoid function to produce a probability
- A from-scratch logistic regression trained with gradient descent on log loss
- A table of precision, recall and accuracy at three thresholds, showing how the decision changes
- A drawn decision boundary and a note on when a straight boundary is not enough

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the boundary plot
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 25: the linear model, the gradient descent loop and the ridge penalty
- Day 24: train, validation and test splits and the baseline habit
- The logarithm and its rules: `log(ab) = log a + log b` and the shape of `log x` near 0 and 1
- Basic probability: a probability lies between 0 and 1, and the probabilities of complementary events sum to 1

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Classification](#classification)
- [Logits](#logits)
- [Sigmoid](#sigmoid)
- [Probabilities](#probabilities)
- [Decision boundaries](#decision-boundaries)
- [Log loss](#log-loss)
- [Thresholds](#thresholds)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Logistic regression with threshold analysis](#project-logistic-regression-with-threshold-analysis)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Many business questions have a yes-or-no answer. Will this customer cancel? Is this transaction fraudulent? Will this patient be readmitted? A regression that outputs 0.4 or 2.7 is not useful for these, but a model that says "there is a 40 percent chance" is. Logistic regression is the standard tool for that kind of answer, and it is the default baseline for classification in most industries.

It matters beyond its direct use. Its output is a probability, its loss is log loss, and its decision rule is a threshold. Those three ideas return in neural networks, where the final layer of a classifier is a logistic function or its multi-class generalization, the softmax. A deep network that classifies images is, at its last step, doing what you will implement today.

On the job, logistic regression is valued because it is fast, stable and explainable. A risk team can read each coefficient as a change in log odds, and an engineer can deploy it on a small server with no GPU. The skill you need is the decision itself: which threshold to pick, and what the cost of each kind of error is.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1838 | Pierre-François Verhulst introduces the logistic function to model population growth. | The S-shaped curve has a name and a long history before it reached machine learning. |
| 1944 | Joseph Berkson proposes the logit model for binary data, and the term "logit" is coined in that work. | The idea of modeling the log odds, which is the core of today's model, comes from biostatistics. |
| 1958 | David Cox publishes work on the logistic regression model for binary outcomes. | The estimation methods used for it in statistics software are built on his framework. |
| 1980s to 1990s | Logistic regression is used widely in epidemiology and credit scoring, with maximum likelihood as the standard training method. | Credit scorecards at banks often still rely on it, because regulators can inspect it. |
| 2010s | Neural networks with a logistic or softmax output become standard, and logistic regression becomes the baseline for classification. | The model is the last layer of a deep classifier and the first baseline you should try. |

**Takeaway:** logistic regression began as a statistical model for populations and survival, and it became the reference classifier because its output can be read and trusted.

---

## Glossary

- **Binary classification**: predicting one of two classes, usually coded 0 and 1.
- **Logit**: the log of the odds, defined as `log(p / (1 - p))`, which maps a probability to the whole real line.
- **Odds**: the ratio of the probability of an event to the probability that it does not happen.
- **Sigmoid (logistic function)**: the function `sigma(z) = 1 / (1 + e^(-z))`, which maps any real number to a value between 0 and 1.
- **Probability estimate**: the model's output `p(y = 1 | x)`, a number between 0 and 1 for one row.
- **Decision boundary**: the set of inputs where the predicted probability equals the threshold.
- **Log loss (binary cross-entropy)**: the average negative log of the probability assigned to the true class.
- **Threshold**: the probability above which the model predicts class 1.
- **Precision**: among rows predicted positive, the share that really are positive.
- **Recall**: among really positive rows, the share the model finds.
- **Calibration**: how closely the predicted probabilities match observed frequencies.

---

### Classification

Consider a doctor who has to decide whether a patient has a condition. The doctor does not know for certain. A useful answer combines a sense of how likely the condition is with a rule for when to act on that sense. Classification has the same two parts: estimate a probability, then turn it into a label.

From first principles, a classifier maps features `x` to a class `y` in {0, 1}. A linear classifier computes a score from the features and then decides by checking the sign or size of that score. The score alone does not say how confident the model is, because a score of 3 and a score of 300 are both positive. Logistic regression fixes this by converting the score into a probability through the sigmoid, which is the central step of the lesson.

```python
import numpy as np

X = np.array([[0.5], [1.5], [3.0], [4.5]])
y = np.array([0, 0, 1, 1])
# A classifier that uses the sign of a linear score predicts 1 if w*x + b > 0.
w, b = 1.0, -2.0
print(((w * X + b).ravel() > 0).astype(int))  # [0 0 1 1]
```

The mistake to avoid is treating a linear classifier's score as a probability. A score of 0.9 from a support vector machine, which you will see on Day 31, is not a probability at all, and calibrating it requires an extra step. Logistic regression is designed to output something that behaves like a probability, although calibration can still drift. In industry, a classifier that flags suspicious transactions produces a score, and the bank sets a threshold to match its investigation capacity, which is classification plus a decision rule.

### Logits

Think of odds as the language of betting. Odds of 3 to 1 mean the event is three times as likely as the alternative. Odds are easy to multiply but awkward to add. Taking their logarithm turns them into a scale that a linear model can handle.

From first principles, the odds of an event with probability `p` are `p / (1 - p)`. The logit is the log of the odds: `logit(p) = log(p / (1 - p))`. It ranges over all real numbers. A probability of 0.5 has logit 0, a probability of 0.9 has logit about 2.2, and a probability of 0.1 has logit about minus 2.2. Logistic regression assumes that the logit is a linear function of the features, `log(p / (1 - p)) = w^T x + b`. The coefficients then have a direct meaning: a one-unit increase in a feature multiplies the odds by `e^w`.

```python
import numpy as np

p = np.array([0.1, 0.5, 0.9])
logit = np.log(p / (1 - p))
print(logit.round(2))         # [-2.2   0.    2.2]
print(np.exp(1.0).round(2))   # 2.72, the odds multiplier for a coefficient of 1
```

A common mistake is reading a coefficient as a change in probability. A coefficient of 1 changes the odds by a factor of about 2.7, but the change in probability depends on where you start. Near 0.5 a shift has a large effect on probability, and near 0 or 1 it has a small one. The limitation is that odds ratios are easy to misread in reports, so present them with a baseline probability in the same table. In industry, a credit model often reports odds ratios for each factor because the regulator expects it.

### Sigmoid

The sigmoid is a smooth switch. Far to the left it is almost 0, far to the right it is almost 1, and in the middle it passes through 0.5 with a steep slope. It converts the unbounded logit into a probability without a hard cutoff, which keeps the function differentiable.

From first principles, `sigma(z) = 1 / (1 + e^(-z))`. Its derivative has a neat form, `sigma'(z) = sigma(z) (1 - sigma(z))`, which makes the gradient easy to compute. The function is the inverse of the logit, so `sigma(logit(p)) = p`. Its steepness at 0.5 is the largest, which means the model is most sensitive to small changes in the score near the decision boundary.

```python
import numpy as np

def sigmoid(z):
    return 1.0 / (1.0 + np.exp(-z))

z = np.array([-4.0, 0.0, 4.0])
print(sigmoid(z).round(4))  # [0.0180 0.5    0.9820]
```

The practical issue is numerical overflow. For large negative `z`, `np.exp(-z)` becomes huge, and the expression can produce warnings or inaccurate results. Production code uses a stable form that branches on the sign of `z`, or relies on a library. The limitation of the sigmoid is that it saturates: for very large `|z|`, the gradient is almost zero, so the model learns slowly from confident mistakes. In the market, the sigmoid is the output layer of binary classifiers in deep learning, and the same saturation issue is one reason training deep networks needed better activations, which you will meet on Day 41.

### Probabilities

Imagine a weather app that says 70 percent chance of rain. A good forecast is not one that says 100 percent on every day that turned out wet. It is one whose stated probabilities match the observed frequencies over many days. Probability estimates are only useful if they are honest.

From first principles, after training, the model gives `p_i = sigma(w^T x_i + b)` for each row. You can read this as the estimated chance that row `i` belongs to class 1. The predicted class is a separate decision that uses a threshold, covered below. Probabilities depend on the training data's class balance. If you train on a sample where the positive class is artificially common, the probabilities are inflated, and you must correct them before using them for decisions.

```python
import numpy as np
from sklearn.linear_model import LogisticRegression

rng = np.random.default_rng(0)
X = rng.normal(size=(500, 2))
y = (X[:, 0] + 0.5 * X[:, 1] + rng.normal(0, 0.8, 500) > 0).astype(int)

clf = LogisticRegression().fit(X, y)
probs = clf.predict_proba(X)[:, 1]
print(probs[:5].round(3))
print("mean predicted", probs.mean().round(3), "observed", y.mean().round(3))
```

The mistake to avoid is trusting the probabilities without checking calibration. Plot the predicted probability against the observed fraction in bins, which scikit-learn can do with `calibration_curve`. A well-calibrated model lies close to the diagonal. Logistic regression is often reasonably calibrated, but not always, and it can drift when the population changes. The limitation is that calibration says nothing about discrimination, meaning how well the model ranks positives above negatives. A model can be well calibrated and still rank poorly. In industry, pricing and insurance teams check calibration before they use a probability to set a premium.

### Decision boundaries

Picture a fence drawn across a field. Points on one side are classed as one thing and points on the other side as another. For a linear model, that fence is a straight line in two dimensions, or a flat plane in higher dimensions.

From first principles, the model predicts class 1 when `p(y = 1 | x) >= t`. Since the sigmoid is increasing, this condition is equivalent to `w^T x + b >= logit(t)`. For the common threshold `t = 0.5`, the logit is zero, so the boundary is the set where `w^T x + b = 0`. Changing the threshold moves the boundary parallel to itself, without changing the learned weights. The boundary is linear in the features, which is the main limitation of the model.

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.linear_model import LogisticRegression

rng = np.random.default_rng(0)
X = rng.normal(size=(300, 2))
y = (X[:, 0] + X[:, 1] ** 2 - 0.5 > 0).astype(int)  # a curved true boundary

clf = LogisticRegression().fit(X, y)
xx, yy = np.meshgrid(np.linspace(-3, 3, 200), np.linspace(-3, 3, 200))
zz = clf.predict(np.c_[xx.ravel(), yy.ravel()]).reshape(xx.shape)
plt.contourf(xx, yy, zz, alpha=0.3)
plt.scatter(X[:, 0], X[:, 1], c=y, s=12, edgecolor="k")
plt.savefig("boundary.png")
```

The mistake is assuming logistic regression can learn a curved boundary by itself. It cannot, unless you add nonlinear features, such as the square of a feature or an interaction term. The data here has a curved true boundary, so the straight line fits poorly. The fix is either to add `X[:, 1] ** 2` as a feature or to use a nonlinear model such as a decision tree on Day 29. In industry, the choice between a linear boundary and a curved one is a trade-off between explainability and fit.

### Log loss

Think of a forecaster judged by the probability she gave to the outcome that actually happened. Giving 0.9 to a rain that occurred is good. Giving 0.01 to it is a serious error, and the penalty should grow sharply as the probability approaches zero. Log loss does exactly that.

From first principles, for one row with true label `y` in {0, 1} and predicted probability `p`, the log loss is `-(y log p + (1 - y) log(1 - p))`. Averaging over rows gives the training loss. The gradient with respect to the weights is `(1/n) X^T (p - y)`, which has the same shape as the linear regression gradient but uses probabilities instead of raw predictions. Minimizing log loss is the same as maximum likelihood estimation for the Bernoulli model, which is why the method is principled.

```python
import numpy as np

def log_loss(y, p, eps=1e-12):
    p = np.clip(p, eps, 1 - eps)
    return -np.mean(y * np.log(p) + (1 - y) * np.log(1 - p))

y = np.array([1, 0, 1])
print(log_loss(y, np.array([0.9, 0.1, 0.8])))   # small, good predictions
print(log_loss(y, np.array([0.01, 0.99, 0.2]))) # large, confident mistakes
```

The mistake to avoid is computing `log(p)` directly when `p` can be exactly 0 or 1, which gives infinity or NaN. Clip the probabilities, as the code does, or use the library. The limitation is that log loss punishes confident wrong answers very hard, so one mislabeled row can dominate the training loss, which is a reason to check label quality. In the market, log loss is the default training loss for click-through and fraud models, and it is also a metric you should report, because it measures probability quality, not only the label.

### Thresholds

A smoke detector has a sensitivity setting. Set it too high and it wakes the house every time someone makes toast. Set it too low and it misses a real fire. The threshold on a classifier has the same trade-off, and the right setting depends on the cost of each kind of error. A fixed default of 0.5 ignores that cost.

From first principles, changing the threshold `t` changes which rows are predicted positive, and therefore changes precision and recall. A lower threshold increases recall, because more real positives are caught, and reduces precision, because more of the flagged rows are false alarms. A higher threshold does the opposite. The threshold can be chosen on the validation set, by maximizing a metric or by matching a cost that the business can state, such as the cost of a missed fraud against the cost of a blocked customer.

```python
import numpy as np
from sklearn.metrics import precision_score, recall_score, accuracy_score

y_val = np.array([1, 0, 1, 1, 0, 0, 1, 0])
p_val = np.array([0.9, 0.4, 0.7, 0.45, 0.3, 0.6, 0.8, 0.1])

for t in (0.3, 0.5, 0.7):
    pred = (p_val >= t).astype(int)
    print(t,
          round(accuracy_score(y_val, pred), 2),
          round(precision_score(y_val, pred), 2),
          round(recall_score(y_val, pred), 2))
```

The mistake is leaving the threshold at 0.5 by habit. On an imbalanced problem, 0.5 is rarely optimal, and accuracy at 0.5 can look fine while recall is terrible. Pick the threshold on validation data, then report the test score once. The limitation is that the chosen threshold may not transfer if the class balance or the score distribution changes after deployment. In industry, medical screening teams often choose a low threshold to avoid missed cases, then accept more follow-up tests.

---

## Hands-on exercises

1. Compute the logit of 0.2, 0.5 and 0.8 by hand, using the natural log, and check your results with NumPy.
2. Plot the sigmoid from -6 to 6 and mark where it crosses 0.5. Describe on the plot where the slope is steepest.
3. Implement `sigmoid` in a numerically stable way that handles large negative inputs without overflow warnings. Test it with `np.array([-1000.0, 1000.0])`.
4. Train a logistic regression from scratch with gradient descent on a synthetic two-feature dataset. Track the log loss at each 100 steps and plot the curve.
5. Compare your weights with `sklearn.linear_model.LogisticRegression(C=1e6)`. Explain why a very large `C` roughly removes the regularization.
6. Choose three thresholds on the validation set and build a table of precision, recall and accuracy for each. Explain which threshold you would choose if a false negative costs five times as much as a false positive.
7. Plot a calibration curve with `sklearn.calibration.calibration_curve`. Describe in one sentence whether the model over- or under-predicts at high probabilities.

---

## Project: Logistic regression with threshold analysis

This is a medium-weight project. The model is short. The deliverable is a threshold analysis that a non-technical stakeholder can act on.

**Repo name:** `100-days-ai-engineering-day-26-logistic-regression`

**Goal:** train a logistic regression on a binary dataset, check its calibration, choose a threshold using a stated cost, and report the test result once.

**Repo layout:**

```
100-days-ai-engineering-day-26-logistic-regression/
├── README.md
├── requirements.txt
├── logreg.py
├── threshold.py
├── results/
│   ├── threshold_table.md
│   └── calibration.png
└── tests/
    └── test_logreg.py
```

**Starter code for `logreg.py`:**

```python
import numpy as np


def sigmoid(z):
    """Numerically stable logistic function."""
    raise NotImplementedError


def log_loss(y, p):
    """Mean binary cross-entropy, with probabilities clipped away from 0 and 1."""
    raise NotImplementedError


def fit(X, y, lr=0.1, steps=3000):
    """Gradient descent on log loss. Return (w, b)."""
    raise NotImplementedError


def predict_proba(X, w, b):
    """Return the probability of class 1 for each row."""
    raise NotImplementedError
```

**Starter code for `tests/test_logreg.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from logreg import sigmoid, log_loss, fit, predict_proba


def make_data(n=400, seed=0):
    rng = np.random.default_rng(seed)
    X = rng.normal(size=(n, 2))
    y = (X[:, 0] + 0.5 * X[:, 1] + rng.normal(0, 0.8, n) > 0).astype(int)
    return X, y


def test_sigmoid_is_stable_for_large_inputs():
    out = sigmoid(np.array([-1000.0, 0.0, 1000.0]))
    assert np.all(np.isfinite(out))
    assert out[1] == 0.5


def test_log_loss_is_small_for_correct_confident_predictions():
    y = np.array([1, 0])
    assert log_loss(y, np.array([0.99, 0.01])) < 0.05


def test_fit_beats_chance_on_training_data():
    X, y = make_data()
    w, b = fit(X, y)
    p = predict_proba(X, w, b)
    assert ((p >= 0.5) == y).mean() > 0.75


def test_probabilities_are_in_unit_interval():
    X, y = make_data()
    w, b = fit(X, y)
    p = predict_proba(X, w, b)
    assert p.min() >= 0.0 and p.max() <= 1.0
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib` and `pytest`.
2. Find a binary dataset with at least 1,000 rows. The UCI Breast Cancer Wisconsin (Diagnostic) dataset is a common choice, available through `sklearn.datasets.load_breast_cancer`.
3. Implement `sigmoid`, `log_loss`, `fit` and `predict_proba`. Run `pytest` until all four tests pass.
4. In `threshold.py`, split the data into train, validation and test parts with a fixed seed. Fit the model on the training part, choose a threshold on the validation part by minimizing a stated cost (for example, a missed case costs five times a false alarm), and write the table of precision, recall and accuracy at a grid of thresholds to `results/threshold_table.md`.
5. Save the calibration plot to `results/calibration.png` and describe it in the README.
6. Score the test part once, at the chosen threshold, and record the result in the README. Do not change the threshold after this step.
7. Write the README. State the cost assumption, the chosen threshold, the test precision and recall, and one sentence on what the business would need to decide.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python threshold.py`
- [ ] The cost assumption is written in plain words
- [ ] The threshold table and calibration plot are included
- [ ] The README states that the test set was used once

**Done when:**

- `pytest` passes all four tests
- The threshold table covers at least five values and the chosen one is justified by the stated cost
- The calibration plot is saved and described
- You can explain in one paragraph why the decision boundary is a straight line, and what you would change if it were not

**What to commit as evidence:** the source files, the tests, the threshold table, the calibration plot, and the README with the cost assumption and the single test result.

---

## Check your understanding

1. A coefficient of 0.7 on a feature is reported as "the odds of churn rise by 0.7." Correct the statement and explain why the change in probability depends on the starting point.
2. Explain why the sigmoid is used instead of a raw linear score when the output must be a probability. What goes wrong with the raw score?
3. Your classifier has 95 percent accuracy on a dataset where 95 percent of rows are negative. Explain why this result is not informative, and name the two metrics you would check instead.
4. Two models rank customers identically, but one has a log loss of 0.2 and the other 0.6. Explain how this is possible, and which model you would use if the probabilities feed a pricing formula.
5. You lower the threshold from 0.5 to 0.3. Predict what happens to precision and to recall, and explain the reasoning in terms of which rows change their predicted class.
6. A product manager asks for a classifier that works well on a curved boundary, and says that logistic regression is too simple. Explain what logistic regression can and cannot do here, and name two ways to extend it without giving up its interpretability.
7. Pairing the sigmoid with log loss gives the gradient `p - y` with respect to the score. Explain why this matters for a confident wrong prediction, and contrast it with what happens if you pair the sigmoid with squared error.

### Answers

1. A coefficient of 0.7 means that a one-unit increase in the feature multiplies the odds of churn by about 2.0, since `e^0.7` is about 2.01. The change in probability depends on the starting probability, because the sigmoid is steep near 0.5 and flat near 0 or 1.
2. A raw linear score can be any real number, so it cannot be read as a probability. The sigmoid maps it into the interval from 0 to 1, and it does so smoothly, which keeps the model differentiable for training.
3. A model that always predicts the majority class gets 95 percent accuracy, so the result does not show that the model learned anything. Check the precision and recall on the positive class, or the ROC AUC, which compares the ranking of positives and negatives.
4. The two models have the same ranking, which is what the AUC measures, but they assign different probabilities. The one with the lower log loss gives better-calibrated and more confident correct probabilities, and it is the better choice when those numbers feed a formula.
5. Lowering the threshold makes the model predict positive for more rows, so recall rises because more real positives are caught. Precision usually falls because some of the newly flagged rows are negatives.
6. Logistic regression can learn any linear boundary in the features it is given, but not a curved one unless curvature is encoded. Two extensions are adding polynomial or interaction features, and using a spline transformation of each feature, which keeps the model additive and explainable.
7. With log loss, the gradient with respect to the score is `p - y`. For a confident wrong prediction, such as `p = 0.99` when `y = 0`, this is close to 1, so the model receives a strong correction. Squared error on a sigmoid output multiplies that gradient by the sigmoid's slope, which is nearly zero at the extremes, so the same mistake gets a weak correction and training stalls. That is the main reason log loss is the standard pairing for logistic regression.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Logistic regression](https://en.wikipedia.org/wiki/Logistic_regression)
- Wikipedia: [Sigmoid function](https://en.wikipedia.org/wiki/Sigmoid_function)
- Wikipedia: [Logit](https://en.wikipedia.org/wiki/Logit)
- Wikipedia: [Cross-entropy](https://en.wikipedia.org/wiki/Cross-entropy)
- Wikipedia: [Precision and recall](https://en.wikipedia.org/wiki/Precision_and_recall)
- Stanford CS229 lecture notes by Andrew Ng, the section on logistic regression and the gradient for the Bernoulli model. Search "CS229 lecture notes logistic regression".
- scikit-learn user guide, "Logistic regression" and "Probability calibration", at scikit-learn.org under the stable documentation.

**Tutorials**

- scikit-learn documentation, the `LogisticRegression` and `calibration_curve` pages.
- GeeksforGeeks, search "logistic regression in machine learning" for a worked example with the sigmoid.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Logistic Regression Details Pt 1: Coefficients" and "Logistic Regression Details Pt 2: Maximum Likelihood".
- 3Blue1Brown (YouTube), "Gradient descent, how neural networks learn", for the gradient intuition that carries over to classifiers.

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course, which includes binary classification exercises you can run in the browser.

---

## Tomorrow

Tomorrow you meet a different way to classify: k-nearest neighbors. Instead of learning weights, the model stores the training data and decides by looking at the closest examples. You will work with distance metrics, choose k, and see why scaling features can change the answer completely. The contrast with today's model is useful, because it shows what a parametric model gains by summarizing data and what it loses.

**Next:** [Day 27: k-Nearest Neighbors](Day-27-k-Nearest-Neighbors.md)

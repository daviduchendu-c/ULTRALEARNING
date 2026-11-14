# Day 25: Linear Regression

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You need the Day 24 workflow and the vector and matrix basics from Day 8 and Day 9. The calculus is light, and you will check each derivative numerically.

**By the end of this day, you will have:**

- A working understanding of the linear model, residuals, mean squared error and the least-squares solution
- A from-scratch implementation of linear regression trained with gradient descent, with a ridge penalty
- A comparison table against scikit-learn's `LinearRegression` and `Ridge` that shows where your code and the library agree and why
- A short note explaining what the coefficients mean and when they should not be trusted

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause license
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the residual plot
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 8: dot product, vector norm and the idea of a projection
- Day 9: matrix multiplication, transpose and solving linear systems
- Day 24: train, validation and test splits, and the baseline habit
- Basic derivatives: the slope of a function and the chain rule at the level of one variable

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [The linear model](#the-linear-model)
- [Residuals and mean squared error](#residuals-and-mean-squared-error)
- [Least-squares intuition](#least-squares-intuition)
- [Gradient descent](#gradient-descent)
- [Regularization introduction](#regularization-introduction)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Linear regression from scratch](#project-linear-regression-from-scratch)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Linear regression is the oldest and most widely deployed predictive model in practice. It predicts a number as a weighted sum of inputs. Pricing teams use it to estimate demand, finance teams use it to explain costs, and engineers use it as the baseline that every other regression must beat.

Its lasting value is the machinery it teaches. The same loss function, the same gradient and the same regularization idea appear in logistic regression, in neural networks and in the training of large language models. If you can derive the update rule for a straight line and see why it works, the later days are variations on one theme.

On the job, the questions you will face are practical. Is the relationship roughly linear? Which features matter, and by how much? Is the model stable when the data changes? Can you explain the coefficient to a product manager? Linear regression gives direct answers to all four, which is why it is often the right first model even when a more complex one is later chosen.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1805 | Adrien-Marie Legendre publishes the method of least squares in an appendix on comets. | The core estimator of linear regression is over two hundred years old. |
| 1809 | Carl Friedrich Gauss publishes a probabilistic justification of least squares and claims he used it earlier. | Gauss links the method to the normal error distribution, which is still the usual assumption. |
| 1886 | Francis Galton introduces the word "regression" while studying the heights of parents and children. | The name refers to regression toward the mean, not to going backward. |
| 1950s to 1960s | Digital computers make the matrix solution practical for large problems, and statistical software standardizes the output. | Libraries such as SAS and later R and Python packages turned the method into a routine tool. |
| 1970 | Arthur Hoerl and Robert Kennard publish ridge regression, which adds a penalty to the coefficients. | The first widely used regularized model, which you will meet again in this lesson. |

**Takeaway:** the model is simple and old, and its tools, least squares and the penalty, are still central to modern training.

---

## Glossary

- **Target**: the number the model predicts, also called the response or dependent variable.
- **Coefficient or weight**: the number multiplied by a feature in the model.
- **Intercept or bias**: the constant term added to the weighted sum.
- **Prediction**: the model's output for one row, which is the intercept plus the weighted sum of its features.
- **Residual**: the actual value minus the predicted value for one row.
- **Mean squared error (MSE)**: the average of the squared residuals.
- **Loss function**: the single number a training procedure tries to make small.
- **Least squares**: the method of choosing coefficients that minimize the sum of squared residuals.
- **Gradient**: the vector of partial derivatives of the loss with respect to each coefficient.
- **Learning rate**: the step size used in each gradient descent update.
- **Regularization**: a penalty added to the loss that discourages large coefficients.
- **Ridge regression**: linear regression with an L2 penalty on the coefficients.

---

### The linear model

Think of a restaurant that prices a meal by adding a base fee to a charge per dish and a charge per guest. Each cost is a constant times a quantity, and the total is the sum. Linear regression works the same way: it multiplies each feature by a weight, adds them, and adds a base value.

From first principles, for a row with features `x_1` to `x_d`, the prediction is `y_hat = b + w_1 x_1 + ... + w_d x_d`, or in vector form `y_hat = b + w^T x`. Stacking many rows into a matrix `X` with shape `(n, d)` gives `y_hat = X w + b`. A common trick is to add a column of ones to `X` so the intercept becomes just another weight. The learning problem is to find the vector `w` and the scalar `b` that make the predictions close to the true targets.

```python
import numpy as np

X = np.array([[1.0, 2.0],
              [2.0, 1.0],
              [3.0, 4.0]])
w = np.array([0.5, 1.0])
b = 2.0
y_hat = X @ w + b
print(y_hat)  # [5.  4.5 8.5]
```

The model assumes that the effect of each feature is additive and that the effect does not depend on the other features. A feature that interacts with another, such as the effect of advertising depending on season, is missed unless you add an interaction term. The mistake to watch for is reading the coefficient as the causal effect of the feature. The coefficient is the association after holding the other features fixed, and it is causal only under strong assumptions about the data. In industry, a pricing team might use a linear model to estimate that each extra square metre adds a certain amount to rent, and the number is useful for forecasting even when the causal story is uncertain.

### Residuals and mean squared error

A residual is the gap between what happened and what the model said would happen. A weather forecaster who predicted 20 degrees and observed 23 has a residual of 3. Residuals are the only evidence the model receives about its own mistakes, so the training procedure is built around them.

From first principles, for each row `i`, the residual is `r_i = y_i - y_hat_i`. The mean squared error is `MSE = (1/n) sum_i r_i^2`. Squaring makes every error positive and penalizes large errors more than small ones. A model that is off by 10 contributes 100 to the sum, while two models that are each off by 5 contribute 50 in total. This property is useful when large errors are costly, and harmful when a few outliers should not dominate.

```python
import numpy as np

y = np.array([3.0, 5.0, 7.0])
y_hat = np.array([2.5, 5.5, 6.0])
residuals = y - y_hat
mse = np.mean(residuals ** 2)
print(residuals, mse)  # [ 0.5 -0.5  1.] 0.5
```

The mistake to avoid is quoting MSE without its unit. It is in squared units, so a target in dollars gives an MSE in dollars squared, which is hard to read. The root mean squared error, covered on Day 35, puts it back on the original scale. The limitation is that MSE rewards models that predict the average well and can hide systematic patterns in the residuals, which is why you should always plot them. A residual plot that curves or fans out tells you the linear model is wrong in a specific way.

### Least-squares intuition

Picture a set of points on a graph and a ruler you slide until the total squared vertical distance to the points is as small as possible. That ruler is the least-squares line. The procedure does not need a search if the problem has a closed form, and for linear regression it does.

From first principles, the loss is `L(w) = ||y - X w||^2`. Setting its gradient with respect to `w` to zero gives the normal equations `X^T X w = X^T y`. When `X^T X` is invertible, the solution is `w = (X^T X)^{-1} X^T y`. Geometrically, `X w` is the projection of `y` onto the column space of `X`, and the residual vector is perpendicular to that space. This is the orthogonality idea from Day 10, which explains why the residuals are uncorrelated with the features in the fitted model.

```python
import numpy as np

rng = np.random.default_rng(0)
X = np.column_stack([np.ones(50), rng.uniform(0, 10, 50)])
true_w = np.array([2.0, 3.0])
y = X @ true_w + rng.normal(0, 1.0, 50)

w_closed = np.linalg.solve(X.T @ X, X.T @ y)
print(w_closed)  # close to [2, 3]
```

Two warnings matter here. First, solving the normal equations with an explicit inverse is numerically fragile when features are correlated. Use `np.linalg.solve` or a QR-based solver instead, as the code does. Second, the closed form is not available when the number of features is larger than the number of rows, or when the design matrix is not full rank. The limitation of least squares is also its sensitivity to outliers, because squared errors let a few extreme points pull the line. In industry, most statistical packages use QR or SVD internally, not the textbook inverse, because they must run reliably on messy data.

### Gradient descent

Imagine walking down a hill in fog. You cannot see the bottom, but you can feel the slope under your feet. Taking a small step downhill, then feeling the slope again, eventually reaches a valley. Gradient descent does exactly that with the loss.

From first principles, the gradient of the MSE with respect to `w` is `(2/n) X^T (X w - y)`. The update is `w <- w - alpha * gradient`, where `alpha` is the learning rate. Each step moves the coefficients a little in the direction that reduces the loss most quickly. If `alpha` is too large, the steps overshoot and the loss can grow. If it is too small, training takes many more steps than needed. For linear regression the loss is convex, so gradient descent with a suitable learning rate reaches the same solution as the closed form.

```python
import numpy as np

def fit_gd(X, y, lr=0.05, steps=2000):
    w = np.zeros(X.shape[1])
    n = X.shape[0]
    for _ in range(steps):
        grad = (2.0 / n) * X.T @ (X @ w - y)
        w -= lr * grad
    return w

rng = np.random.default_rng(0)
X = np.column_stack([np.ones(50), rng.uniform(0, 10, 50)])
y = X @ np.array([2.0, 3.0]) + rng.normal(0, 1.0, 50)
print(fit_gd(X, y))  # close to the least-squares answer
```

The common mistake is not scaling the features. If one feature ranges from 0 to 1 and another from 0 to 100000, the loss surface is a long narrow valley, and gradient descent zigzags slowly. Standardize the features first, as you practiced on Day 24. Another mistake is stopping when the loss is small without checking the gradient, because a flat region can look like convergence when it is not. The limitation of plain gradient descent is that it uses the whole dataset each step, which is expensive for large data. Stochastic and mini-batch versions, covered on Day 44, handle that. In the market, every deep learning framework trains its models with variants of this loop, so the update rule you write today is the same idea at larger scale.

### Regularization introduction

A student who memorizes every number in a textbook can answer the exact exercises but fails when a question changes slightly. A student who learns the rules handles new questions better. Regularization pushes a model toward the second kind of learning by discouraging large, fragile coefficients.

From first principles, ridge regression adds a penalty `lambda ||w||^2` to the loss, leaving the intercept out of the penalty. The gradient gains a term `2 lambda w`, and the closed form becomes `w = (X^T X + lambda I)^{-1} X^T y`. Adding `lambda I` makes the matrix invertible even when features are correlated, and it shrinks coefficients toward zero. A larger `lambda` means stronger shrinkage, which reduces variance at the cost of some bias, the trade-off you met on Day 24.

```python
import numpy as np

def fit_ridge(X, y, lam):
    d = X.shape[1]
    I = np.eye(d)
    I[0, 0] = 0.0  # do not penalize the intercept column
    return np.linalg.solve(X.T @ X + lam * I, X.T @ y)

rng = np.random.default_rng(0)
X = np.column_stack([np.ones(50), rng.uniform(0, 10, 50)])
y = X @ np.array([2.0, 3.0]) + rng.normal(0, 1.0, 50)
for lam in (0.0, 1.0, 100.0):
    print(lam, fit_ridge(X, y, lam).round(3))
```

The mistake is choosing `lambda` by looking at the training loss, which always falls as `lambda` shrinks. Choose it with cross-validation, and standardize features before penalizing them, because the penalty depends on the units. The limitation is that ridge keeps every feature, shrinking them but not removing them. Lasso, an L1 penalty, can set coefficients exactly to zero, which is useful for selection but is not covered today. In industry, ridge is a standard default for linear models with many correlated inputs, such as sensor readings or marketing channels, where the plain least-squares answer swings wildly from one sample to the next.

---

## Hands-on exercises

1. Generate data with `rng = np.random.default_rng(0)` from the model `y = 2 + 3x + noise`, with noise standard deviation 1.0 and 50 rows. Plot the points and the fitted line with Matplotlib.
2. Compute the residuals by hand for three points of your choice, square them and average them. Check your result with `np.mean`.
3. Derive the gradient of the MSE on paper for a model with one feature and an intercept. Confirm it matches the formula `(2/n) X^T (X w - y)` by checking the shapes of each term.
4. Implement `fit_gd` from the section above. Plot the loss against the step number for learning rates 0.001, 0.05 and 0.5. Describe what happens to each curve and why.
5. Compare `fit_gd` with `np.linalg.lstsq` and with `sklearn.linear_model.LinearRegression`. Report the largest absolute difference in coefficients.
6. Create a dataset with two nearly identical features. Fit ordinary least squares and ridge with `lambda` equal to 1. Report the coefficients from both and explain the difference in one paragraph.
7. Use 5-fold cross-validation on the training data to choose `lambda` from the values 0.01, 0.1, 1, 10 and 100. Report the mean validation MSE for each.

Record each number in a notebook cell with a short comment. The comments are part of the evidence.

---

## Project: Linear regression from scratch

This is a medium-weight project. The model is short, so most of the value comes from testing the properties of the solution and comparing it with a library.

**Repo name:** `100-days-ai-engineering-day-25-linear-regression`

**Goal:** implement linear regression with a closed-form solver, a gradient descent solver and a ridge penalty, verify that the solvers agree with scikit-learn, and write a short note on the coefficients and residuals.

**Repo layout:**

```
100-days-ai-engineering-day-25-linear-regression/
├── README.md
├── requirements.txt
├── linreg.py
├── compare.py
├── results/
│   └── residuals.png
└── tests/
    └── test_linreg.py
```

**Starter code for `linreg.py`:**

```python
import numpy as np


def add_intercept(X):
    """Return X with a leading column of ones."""
    return np.column_stack([np.ones(X.shape[0]), X])


def fit_closed_form(X, y):
    """Least squares with np.linalg.solve on the normal equations. Return w with the intercept first."""
    raise NotImplementedError


def fit_gradient_descent(X, y, lr=0.05, steps=5000):
    """Gradient descent on the MSE. Return w with the intercept first."""
    raise NotImplementedError


def fit_ridge(X, y, lam):
    """Ridge regression that leaves the intercept unpenalized. Return w."""
    raise NotImplementedError


def predict(X, w):
    """Return X with an intercept column, multiplied by w."""
    raise NotImplementedError


def mse(y, y_hat):
    """Mean squared error."""
    raise NotImplementedError
```

**Starter code for `tests/test_linreg.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from linreg import add_intercept, fit_closed_form, fit_gradient_descent, fit_ridge, predict, mse


def make_data(n=200, seed=0):
    rng = np.random.default_rng(seed)
    X = rng.uniform(0, 10, (n, 2))
    y = 1.5 + 2.0 * X[:, 0] - 0.5 * X[:, 1] + rng.normal(0, 0.3, n)
    return X, y


def test_closed_form_recovers_known_weights():
    X, y = make_data()
    w = fit_closed_form(add_intercept(X), y)
    assert w == pytest.approx([1.5, 2.0, -0.5], abs=0.1)


def test_gradient_descent_matches_closed_form():
    X, y = make_data()
    Xi = add_intercept(X)
    assert np.allclose(fit_gradient_descent(Xi, y), fit_closed_form(Xi, y), atol=1e-2)


def test_ridge_with_zero_penalty_matches_closed_form():
    X, y = make_data()
    Xi = add_intercept(X)
    assert np.allclose(fit_ridge(Xi, y, 0.0), fit_closed_form(Xi, y), atol=1e-8)


def test_ridge_shrinks_coefficients():
    X, y = make_data()
    Xi = add_intercept(X)
    small = np.linalg.norm(fit_ridge(Xi, y, 100.0)[1:])
    large = np.linalg.norm(fit_closed_form(Xi, y)[1:])
    assert small < large


def test_mse_is_zero_for_perfect_predictions():
    y = np.array([1.0, 2.0, 3.0])
    assert mse(y, y) == 0.0
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib` and `pytest`.
2. Implement `add_intercept`, `fit_closed_form`, `predict` and `mse`. Run `pytest` and make the first tests pass.
3. Implement `fit_gradient_descent` with the update from the lesson. Pick a learning rate that converges on standardized features, and record the number of steps you needed.
4. Implement `fit_ridge`, leaving the intercept column unpenalized. Make the ridge tests pass.
5. Write `compare.py`. Fit your three solvers and `sklearn.linear_model.LinearRegression` on the same data, and print a table of coefficients and the maximum difference from scikit-learn.
6. In `compare.py`, also plot the residuals against the fitted values and save the figure to `results/residuals.png`. Look for curvature or a fan shape.
7. Write the README. Report the coefficients, the MSE on the training and validation parts, and the regularization path for three values of lambda.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python compare.py`
- [ ] The coefficient table from your solver and from scikit-learn is included
- [ ] The residual plot is embedded with a one-line description of what it shows
- [ ] The README names one assumption that the data may violate, and one place the model fails

**Done when:**

- `pytest` passes all five tests
- Your closed-form and gradient descent solvers agree with scikit-learn to at least three decimal places on the test data
- The residual plot is saved and described in the README
- You can explain in one paragraph why the residuals are orthogonal to the features at the least-squares solution

**What to commit as evidence:** the source files, the tests, the comparison output pasted into the README, the residual plot, and the written note.

---

## Check your understanding

1. A model predicts house prices with a coefficient of 300 on square metres. A colleague says that adding one square metre causes the price to rise by 300. What is the correct interpretation, and what assumption would be needed for the causal reading?
2. Your residual plot shows a clear curve, with positive residuals at the ends and negative residuals in the middle. What does this tell you about the model, and what would you try next?
3. Two features are almost identical. Explain why ordinary least squares can give large, opposite coefficients for them, and how ridge regression changes that outcome.
4. You run gradient descent and the loss goes up after each step. Name the most likely cause and the fix.
5. Explain why the least-squares residuals are uncorrelated with each feature in the fitted model, using the idea of projection.
6. A teammate adds a penalty to the intercept and reports that the model underpredicts every row. Explain why penalizing the intercept can cause a problem, and describe the fix.
7. Your model has a training MSE of 4.0 and a validation MSE of 4.2. Your colleague says the model is ready to ship. What else would you check before agreeing?

### Answers

1. The coefficient means that, holding the other features fixed, a one-unit increase in square metres is associated with a 300 unit increase in predicted price. The causal reading needs the assumption that no unobserved factor, such as location, drives both the size and the price, and that the feature can be changed without changing others.
2. The model misses a curved pattern in the data. A linear model cannot represent the curve, so the residuals show structure. Try adding a polynomial term, a transformed feature such as the logarithm of the target or the feature, or a nonlinear model.
3. When two features carry nearly the same information, many combinations of their coefficients fit the data equally well, and small noise can push the solution far in opposite directions. Ridge adds a penalty that favors the smaller and more stable solution, and it spreads the weight between the two correlated features.
4. The learning rate is most likely too large, so each step overshoots the minimum. Reduce the learning rate, and standardize the features if the scales differ widely.
5. At the least-squares solution, the prediction `X w` is the projection of `y` onto the column space of `X`. The residual is the part of `y` perpendicular to that space, so it is orthogonal to every column of `X`, which means it has zero inner product with every feature.
6. Penalizing the intercept shrinks the average prediction toward zero, which pulls every prediction down when the target has a nonzero mean. The fix is to leave the intercept column out of the penalty, as the code does.
7. A small gap between training and validation suggests the model generalizes, but it does not show the error is acceptable for the business. Check whether 4.0 MSE is good enough in the target's units, compare against the baseline from Day 24, look at the residuals for patterns, and confirm the test set score on data you have not used.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Linear regression](https://en.wikipedia.org/wiki/Linear_regression)
- Wikipedia: [Least squares](https://en.wikipedia.org/wiki/Least_squares)
- Wikipedia: [Gradient descent](https://en.wikipedia.org/wiki/Gradient_descent)
- Wikipedia: [Ridge regression](https://en.wikipedia.org/wiki/Ridge_regression)
- Wikipedia: [Mean squared error](https://en.wikipedia.org/wiki/Mean_squared_error)
- Stanford CS229 lecture notes by Andrew Ng, the first set on linear regression and least squares. Search "CS229 lecture notes Andrew Ng supervised learning".
- scikit-learn user guide, "Linear Models", at scikit-learn.org under the stable documentation.

**Tutorials**

- scikit-learn documentation, the `LinearRegression` and `Ridge` pages, with the parameters and worked examples.
- GeeksforGeeks, search "linear regression machine learning" for a worked example with a small table.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Linear Regression, Clearly Explained" and the video on the least-squares method.
- 3Blue1Brown (YouTube), "Gradient descent, how neural networks learn", for the intuition behind the update rule.
- MIT OpenCourseWare, 6.036 "Introduction to Machine Learning", the lectures on linear regression.

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course, and the "Linear regression" examples in Kaggle notebooks, which you can run and modify in the browser.

---

## Tomorrow

Tomorrow you keep the linear model but change its output. Logistic regression predicts a probability for a yes-or-no label by passing a weighted sum through the sigmoid function, and it is trained with log loss instead of squared error. You will meet the logit, the decision threshold and the decision boundary, and you will see that the training loop you wrote today carries over almost unchanged.

**Next:** [Day 26: Logistic Regression](Day-26-Logistic-Regression.md)

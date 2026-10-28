# Day 16: Mathematics Project

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 4 hours
**Level:** intermediate. This is the capstone of Module 2. You need the earlier days of this module and enough Python to write a few functions and a test. It is for readers who want proof that they can turn the mathematics into code that works.

**By the end of this day, you will have:**

- A `mathlab` package that implements vector and matrix operations, mean squared error, its gradient, linear regression with gradient descent and a closed-form check, all written without machine learning frameworks
- A test suite that compares each function with a trusted reference and passes
- A training run whose learned weights match the true weights used to generate the data, and whose loss falls at every step
- A README that explains each result and records the learning rates you tried

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`). NumPy is used only for arrays, random numbers and the reference check. You may not use scikit-learn, PyTorch, TensorFlow, JAX or any other machine learning framework in the project code.
- pytest 7 or newer (`pip install pytest`)
- Git and a free GitHub account for the repository
- Optional: matplotlib 3.7 or newer, if you want to plot the loss curve

**Prerequisites:**

- Day 7: functions, summation notation and square roots
- Day 8: the dot product and the magnitude of a vector
- Day 9: matrix shapes, multiplication and transposes
- Day 10: the idea that the solution to a least squares problem is a projection
- Day 11: derivatives and the chain rule, and gradients
- Day 12: gradient descent, the learning rate and the loop that repeats the update
- Day 13 to Day 15: probability, statistics and cross-entropy, used here to read the loss and the evaluation

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Background and history](#background-and-history)
- [Glossary](#glossary)
- [Vectors and matrices from scratch](#vectors-and-matrices-from-scratch)
- [Linear regression](#linear-regression)
- [Loss calculation](#loss-calculation)
- [Gradient descent from scratch](#gradient-descent-from-scratch)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: mathlab, linear regression from scratch](#project-mathlab-linear-regression-from-scratch)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Over the last eight days, you have met the mathematics behind machine learning one piece at a time. Vectors hold the features of an example. Matrices hold a whole dataset. Derivatives tell you which way to move the parameters. Probability and information theory tell you what a loss means. Today those pieces meet in one program. You will fit a line to data, measure how wrong it is, and walk the parameters downhill until the error stops shrinking.

On the job, this is the most useful skill in the module. Teams that can read a library's source, reproduce a model by hand and check it against a reference are the ones who debug training runs instead of guessing. A model that trains with a NumPy loop you wrote yourself is also a model you can explain to a client, a reviewer or a regulator.

The big idea of this day is that linear regression is the smallest complete machine learning system. It has data, a model with parameters, a loss, a way to compute the gradient of the loss and an update rule. Every neural network you will train later repeats that same loop with more layers and more parameters. Once you have built it once, the frameworks stop looking like magic.

---

## Background and history

| Period | What happened | Why it matters today |
|---|---|---|
| 1805 | Adrien-Marie Legendre publishes the method of least squares for fitting orbits of comets. | The first published fit of a line to noisy measurements by minimizing squared error. |
| 1809 | Carl Friedrich Gauss publishes his own least squares work, and claims to have used it since 1795. | Gauss's version is the basis of the theory of linear regression. |
| 1886 | Francis Galton describes "regression towards mediocrity" in hereditary stature, where extreme parents tend to have less extreme children. | The name for the model you are building today comes from a biology study. |
| 1950s | Computers make it practical to solve least squares problems for many variables, using matrix methods. | The closed-form solution becomes a routine calculation in statistics software. |
| 1960s | Widrow and Hoff introduce the LMS algorithm, an incremental update rule that is essentially gradient descent on squared error. | The first practical learning rule for a single adaptive unit, from which neural network training descends. |

**Takeaway:** linear regression is about two centuries old. The loop you will write today is the direct ancestor of the training code in modern deep learning.

---

## Glossary

- **Feature**: one measured input to a model, such as the number of rooms in a house.
- **Target**: the value the model is trying to predict, such as the price of the house.
- **Design matrix**: a matrix with one row per example and one column per feature. A column of ones is added for the intercept.
- **Parameters (weights)**: the numbers the model learns. In linear regression, these are the intercept and one coefficient per feature.
- **Prediction**: the model's output for an example, computed as the dot product of the features and the weights.
- **Residual**: the difference between the target and the prediction for one example.
- **Mean squared error (MSE)**: the average of the squared residuals.
- **Gradient**: the vector of partial derivatives of the loss with respect to each weight.
- **Learning rate**: the size of each step in gradient descent.
- **Epoch or iteration**: one update of the weights. In this project, a full-batch iteration uses every example.
- **Closed form**: a formula that gives the answer directly, here `w = (X^T X)^-1 X^T y`, instead of by repeated steps.
- **Normal equations**: the linear system `(X^T X) w = X^T y` whose solution is the least squares answer.
- **Reference implementation**: a trusted library function, such as `np.linalg.solve`, that you compare your code against.
- **Unit test**: a small automated check that a single function gives the expected answer.
- **Regression test**: a check that a result did not change after you edited the code.

---

## Vectors and matrices from scratch

A vector is an ordered list of numbers, and in machine learning it usually holds the features of one example. A house with 3 rooms, 120 square meters and a garden is the vector `[3, 120, 1]`. A matrix stacks many vectors into rows. A dataset of 200 houses is a 200 by 3 matrix.

The two operations you need most are the dot product and matrix multiplication. The dot product multiplies matching entries and adds the results:

```
[1, 2, 3] · [4, -5, 6] = 1*4 + 2*(-5) + 3*6 = 4 - 10 + 18 = 12
```

Matrix multiplication computes one dot product for each pair of a row from the first matrix and a column from the second. If `A` has shape 2 by 3 and `B` has shape 3 by 2, the product `AB` has shape 2 by 2, and each entry is a dot product of a row of `A` with a column of `B`. The inner dimensions must match, which is the shape rule from Day 9.

```python
A = [[1, 2, 3], [4, 5, 6]]
B = [[1, 0], [0, 1], [1, 1]]

# entry (0, 0) is the dot product of row 0 of A with column 0 of B
print(1 * 1 + 2 * 0 + 3 * 1)   # 4
# entry (1, 1) is row 1 of A dotted with column 1 of B
print(4 * 0 + 5 * 1 + 6 * 1)   # 11
```

Matrix multiplication is not commutative. `AB` and `BA` are generally different, and `BA` may not even exist. Transposing a matrix swaps its rows and columns, which is written `A^T`. The transpose appears in the linear regression formula, because `X^T X` is how you combine every feature with every other feature.

The common mistake is to multiply shapes that do not line up and to misread the error. If `X` is 200 by 3 and `w` has length 3, then `X @ w` is valid and gives 200 predictions. If you accidentally pass a column vector of shape 200 by 1, you get a shape error or, worse, a silent broadcast that gives the wrong answer. Print the shapes of every array before you combine them.

In the market, vectors and matrices are the native data type of machine learning. Recommendation systems store users and items as matrices, search engines store documents as vectors and GPUs are built to multiply matrices quickly. Every ML library is, at its core, a set of optimized vector and matrix operations.

---

## Linear regression

Linear regression is the model where the prediction is a weighted sum of the features plus an intercept. For one feature, it is the familiar line `y = w0 + w1 * x`. For several features, it is the dot product of the weight vector and the feature vector:

```
prediction = w0 + w1 * x1 + w2 * x2 + ... = X @ w
```

The trick that makes the notation clean is to add a column of ones to the data. Then the intercept `w0` multiplies a constant 1, and the whole model becomes a single matrix-vector product. The design matrix `X` has a leading column of ones.

Here is a tiny example. Suppose three houses have one feature, the size in square meters, and the prices are 150, 210 and 270 thousand. The design matrix has rows `[1, 50]`, `[1, 70]` and `[1, 90]`. If the weights are `[30, 3]`, the predictions are `30 + 3 * 50 = 180`, `30 + 3 * 70 = 240` and `30 + 3 * 90 = 300`. The model is close, and the residuals are `-30`, `-30` and `-30`, which is the same error for every house.

Linear regression finds the weights that make the predictions as close as possible to the targets, measured by squared error. There are two ways to get the answer. The closed form solves the normal equations `(X^T X) w = X^T y` in one step, which is exact and fast for small problems. Gradient descent starts from any guess and improves the weights step by step, which scales to large data and works for models with no closed form. This project implements both, so you can check one against the other.

The common mistake is to think a linear model cannot learn curves. It is linear in its weights, not in its inputs. If you add a feature `x^2` as a column, the model fits a parabola, and the same math applies. The honest limitation is that a linear model can only express relationships that are linear in the features you give it. If the true relationship has interactions or thresholds, the residuals will show a pattern, and the model needs new features or a different model.

On the job, linear regression is the baseline that every serious forecasting, pricing and risk project starts with. A complicated model that cannot beat a well-built linear regression on the same features is a model that needs more work, not a success.

---

## Loss calculation

A loss function turns the difference between predictions and targets into one number. For linear regression, the standard loss is mean squared error:

```
MSE = (1 / n) * sum over i of (y_i - prediction_i)^2
```

The squared residual makes large errors count much more than small ones. An error of 10 contributes 100, and an error of 1 contributes 1. The square also makes the function smooth, which is what gradient descent needs. The average over `n` examples keeps the loss from growing just because the dataset got bigger.

Here is the calculation for the house example above, where the residuals are `-30` each:

```python
import numpy as np

y = np.array([150.0, 210.0, 270.0])
X = np.array([[1.0, 50.0], [1.0, 70.0], [1.0, 90.0]])
w = np.array([30.0, 3.0])

residuals = y - X @ w
mse = np.mean(residuals ** 2)
print(residuals)   # [-30. -30. -30.]
print(mse)         # 900.0
```

MSE is in squared units. The loss of 900 means "thousands squared," which is hard to interpret, so people also report the root mean squared error, which is the square root of the MSE. Here it is 30, which says the typical error is 30 thousand. Reporting the RMSE is a choice that makes results readable to the people who will act on them.

The common mistake is to compare losses across datasets with different scales. An MSE of 900 is excellent for house prices in thousands and terrible for a quantity that varies between 0 and 1. A loss only means something relative to the spread of the target. Compare it with the variance of the target, or report R-squared, which is `1 - SS_res / SS_tot`, the fraction of variance the model explains.

The honest limitation of squared error is that it is sensitive to outliers. One house listed at ten times its normal price can dominate the loss and drag the weights toward it. Robust losses such as the absolute error or the Huber loss handle outliers better, at the cost of a less smooth gradient. For this project, MSE is the right choice, and the outlier issue is worth noting in your README.

In the market, MSE is the default loss for regression in nearly every library and in most forecasting tools. The choice of loss, and whether the team checked it against the business cost of errors, is one of the most important decisions in a regression project.

---

## Gradient descent from scratch

Gradient descent improves the weights by moving them opposite to the gradient of the loss. For MSE with a linear model, the gradient has a clean form. The derivative of the loss with respect to the weights is:

```
gradient = (2 / n) * X^T (X w - y)
```

Read it piece by piece. `X w - y` is the vector of residuals (predictions minus targets). Multiplying by `X^T` combines each residual with the feature values that produced it. Dividing by `n` gives an average over the examples, and the 2 comes from the derivative of the square. The update rule is:

```
w_new = w_old - learning_rate * gradient
```

Here is one full step on a tiny dataset, worked by hand and then checked in code. The design matrix is `X = [[1, 1], [1, 2]]` and the targets are `y = [2, 3]`. Start from `w = [0, 0]`. The predictions are all zero, so the residuals are `[-2, -3]`. Then `X^T` times the residuals is `[1*(-2) + 1*(-3), 1*(-2) + 2*(-3)]`, which is `[-5, -8]`. With `n = 2`, the gradient is `(2/2) * [-5, -8] = [-5, -8]`. With a learning rate of 0.1, the new weights are `[0.5, 0.8]`.

```python
import numpy as np

X = np.array([[1.0, 1.0], [1.0, 2.0]])
y = np.array([2.0, 3.0])
w = np.zeros(2)

gradient = (2 / len(y)) * X.T @ (X @ w - y)
w = w - 0.1 * gradient
print(gradient)   # [-5. -8.]
print(w)          # [0.5 0.8]
```

The loop repeats this step many times, and the loss should fall at every step when the learning rate is small enough. Day 12 showed what happens when it is too large. A useful check is to record the loss at each iteration and confirm that it is decreasing. If it grows, the learning rate is too big, and if it is flat from the start, the gradient may be wrong.

The common mistake is to get the sign or the scale of the gradient wrong and not notice, because the loss still falls a little. The fix is a finite difference check. Compute `(loss(w + h) - loss(w - h)) / (2h)` for a small `h` and compare it with your analytic gradient. The tests in this project do exactly that, and they catch the errors that a falling loss would hide.

The honest limitation is that full-batch gradient descent reads the whole dataset at every step. For a dataset with millions of rows, that is slow, and the stochastic methods from Day 12 are used instead. For this project, the dataset is small enough that full batch is the right choice.

In the market, you will rarely write the gradient by hand for a production model. Frameworks compute it automatically. But the engineers who can derive the gradient for a small model, check it with a finite difference and reproduce the update loop are the ones who can tell when an automatic gradient is wrong, or when a custom loss has a bug.

---

## Hands-on exercises (about 60 minutes)

Create a folder called `day16-warmup` and write your code in `warmup.py`. Do each step on paper first, then check it in Python. The exercises prepare you for the project.

1. **Dot product by hand (5 minutes).** Compute `[2, -1, 4] · [3, 5, 0]` on paper. Then write a function `dot(a, b)` using a loop, and confirm the result with `np.dot`. The answer is `6 - 5 + 0 = 1`.

2. **Matrix product by hand (10 minutes).** Multiply `A = [[1, 2], [0, 1], [3, 4]]` (shape 3 by 2) by `B = [[2, 0, 1], [1, 1, 0]]` (shape 2 by 3) on paper. Write the result as a 3 by 3 matrix. Check with `A @ B` in NumPy. Then try `B @ A` and note its shape.

3. **MSE by hand (10 minutes).** Targets are `y = [5, 7, 9]` and predictions are `[4, 7, 11]`. Compute the residuals, the squared residuals and the MSE on paper. Confirm with NumPy. Then write one sentence on which example contributes most to the loss and why.

4. **One gradient step by hand (15 minutes).** Use `X = [[1, 1], [1, 2]]`, `y = [2, 3]`, `w = [0, 0]` and a learning rate of 0.1. Compute the gradient and the new weights by hand, as shown in the lesson. Check your numbers with the code in this lesson.

5. **Finite difference check (10 minutes).** For the same data, write a function that computes the loss at `w`. Compute the numerical gradient by perturbing each weight by `h = 1e-6`, and compare it with the analytic gradient `[-5, -8]` at `w = [0, 0]`. Confirm that the two agree to about six decimal places.

6. **Predict a line (10 minutes).** Use the three houses from the lesson. Set `w = [30, 3]` and confirm the residuals are `-30` each and the MSE is 900. Then change `w` to `[0, 3]` and compute the new MSE. The answer is 0, because these three houses lie exactly on the line `y = 3x`. Write one sentence on whether the intercept or the slope mattered more for this change.

**Evidence to commit:** `warmup.py` with the functions and checks from steps 1, 4 and 5, and a file called `notes.md` with the hand calculations and your sentences from steps 3 and 6.

---

## Project: mathlab, linear regression from scratch

This project is practical and it is the module's capstone. You will implement the core of a small machine learning library, with no ML framework, and test every piece against a trusted reference. Name the repository `100-days-ai-engineering-day-16-mathematics-project`.

The starter code gives you the data generator, the test suite and the main script. The functions you must write are marked `NotImplementedError`. Every test fails until you implement the matching function, and every test should pass when you are done.

**Repo layout**

```
100-days-ai-engineering-day-16-mathematics-project/
├── README.md
├── requirements.txt
├── main.py
├── mathlab/
│   ├── __init__.py
│   ├── data.py
│   ├── vectors.py
│   ├── losses.py
│   └── regression.py
└── tests/
    └── test_mathlab.py
```

**Starter code**

`requirements.txt`:

```
numpy>=1.24
pytest>=7.0
```

`mathlab/__init__.py` is empty.

`mathlab/data.py`:

```python
"""Synthetic data with a known answer, so you can check your code."""
import numpy as np


def make_linear_data(n_samples=200, true_w=(4.0, -2.0, 0.5), noise=0.5, seed=0):
    """Return (X, y, true_w). X has a leading column of ones for the intercept."""
    rng = np.random.default_rng(seed)
    true_w = np.asarray(true_w, dtype=float)
    n_features = len(true_w) - 1
    raw = rng.uniform(-3.0, 3.0, size=(n_samples, n_features))
    X = np.c_[np.ones(n_samples), raw]
    y = X @ true_w + rng.normal(0.0, noise, size=n_samples)
    return X, y, true_w
```

`mathlab/vectors.py` (you implement every function):

```python
"""Vector and matrix operations. Vectors are lists of floats; matrices are lists of rows."""
import math


def dot(a, b):
    """Return the dot product sum(a_i * b_i). Raise ValueError if lengths differ."""
    raise NotImplementedError


def add(a, b):
    """Return the element-wise sum. Raise ValueError if lengths differ."""
    raise NotImplementedError


def scale(c, a):
    """Multiply every element of a by the scalar c."""
    raise NotImplementedError


def norm(a):
    """Return the Euclidean length of a. Use dot and math.sqrt."""
    raise NotImplementedError


def unit(a):
    """Return a divided by its norm. Raise ValueError for the zero vector."""
    raise NotImplementedError


def transpose(A):
    """Swap rows and columns of A, given as a list of rows."""
    raise NotImplementedError


def matmul(A, B):
    """Multiply A (m by n) by B (n by p). Raise ValueError if the inner sizes differ."""
    raise NotImplementedError
```

`mathlab/losses.py` (you implement every function, using NumPy arrays):

```python
"""Loss functions and their gradients. Arrays are NumPy arrays."""


def predict(X, w):
    """Return X @ w."""
    raise NotImplementedError


def mse(y_true, y_pred):
    """Mean of the squared residuals."""
    raise NotImplementedError


def mse_gradient(X, y_true, w):
    """Gradient of mse(y, X @ w) with respect to w, equal to (2 / n) * X^T (X w - y)."""
    raise NotImplementedError
```

`mathlab/regression.py` (you implement `fit`, `predict` and `fit_normal_equations`):

```python
"""Linear regression trained with gradient descent."""
import numpy as np

from .losses import mse, mse_gradient


class LinearRegression:
    def __init__(self, learning_rate=0.05, n_iters=2000):
        self.learning_rate = learning_rate
        self.n_iters = n_iters
        self.w = None
        self.loss_history = []

    def fit(self, X, y):
        """Start with zero weights. Each iteration, record mse, then step against the gradient.
        Store the weights in self.w and the losses in self.loss_history. Return self."""
        raise NotImplementedError

    def predict(self, X):
        """Return X @ self.w."""
        raise NotImplementedError


def fit_normal_equations(X, y):
    """Closed-form answer w = (X^T X)^-1 X^T y. Solve (X^T X) w = X^T y with np.linalg.solve."""
    raise NotImplementedError
```

`main.py`:

```python
from mathlab.data import make_linear_data
from mathlab.losses import mse
from mathlab.regression import LinearRegression, fit_normal_equations

X, y, true_w = make_linear_data()
model = LinearRegression(learning_rate=0.05, n_iters=2000).fit(X, y)
print("true   ", true_w.round(3))
print("gd     ", model.w.round(3))
print("closed ", fit_normal_equations(X, y).round(3))
print("final mse", round(mse(y, model.predict(X)), 4))
```

`tests/test_mathlab.py`:

```python
import numpy as np
import pytest

from mathlab.vectors import dot, add, scale, norm, unit, matmul, transpose
from mathlab.losses import mse, mse_gradient
from mathlab.data import make_linear_data
from mathlab.regression import LinearRegression, fit_normal_equations


def test_dot_matches_numpy():
    a, b = [1.0, 2.0, 3.0], [4.0, -5.0, 6.0]
    assert dot(a, b) == pytest.approx(np.dot(a, b))


def test_dot_rejects_mismatched_lengths():
    with pytest.raises(ValueError):
        dot([1, 2], [1, 2, 3])


def test_norm_and_unit():
    assert norm([3.0, 4.0]) == pytest.approx(5.0)
    assert norm(unit([3.0, 4.0])) == pytest.approx(1.0)


def test_matmul_matches_numpy():
    A = [[1.0, 2.0], [3.0, 4.0], [5.0, 6.0]]
    B = [[1.0, 0.0, 2.0], [0.0, 1.0, 3.0]]
    assert np.allclose(matmul(A, B), np.array(A) @ np.array(B))
    assert transpose(A) == [[1.0, 3.0, 5.0], [2.0, 4.0, 6.0]]


def test_mse_gradient_matches_finite_difference():
    X, y, _ = make_linear_data(n_samples=50)
    w = np.array([0.5, -1.0, 2.0])
    eps = 1e-6
    numeric = np.zeros_like(w)
    for j in range(len(w)):
        step = np.zeros_like(w)
        step[j] = eps
        numeric[j] = (mse(y, X @ (w + step)) - mse(y, X @ (w - step))) / (2 * eps)
    assert np.allclose(mse_gradient(X, y, w), numeric, atol=1e-5)


def test_gradient_descent_matches_closed_form():
    X, y, _ = make_linear_data()
    model = LinearRegression(learning_rate=0.05, n_iters=2000).fit(X, y)
    assert np.allclose(model.w, fit_normal_equations(X, y), atol=1e-3)


def test_loss_decreases():
    X, y, _ = make_linear_data()
    model = LinearRegression(learning_rate=0.05, n_iters=500).fit(X, y)
    assert model.loss_history[-1] < model.loss_history[0]
```

Running `python main.py` after you finish should print values close to these. The learned weights match the true weights `[4, -2, 0.5]` within noise, and the closed form matches gradient descent:

```
true    [ 4.  -2.   0.5]
gd      [ 4.004 -1.972  0.554]
closed  [ 4.004 -1.972  0.554]
final mse 0.2353
```

**Step-by-step tasks**

1. Create the repository, the folder tree and a virtual environment. Install the requirements. Run `pytest` and confirm that all seven tests fail with `NotImplementedError`. This tells you the tests are connected to the code.
2. Implement `dot`, `add`, `scale`, `norm` and `unit` in `vectors.py`. Run `pytest -k "dot or norm"` after each function.
3. Implement `transpose` and `matmul` in `vectors.py`. Use `dot` inside `matmul` and `zip` inside `transpose`. Run `pytest -k matmul`.
4. Implement `predict`, `mse` and `mse_gradient` in `losses.py`. Run `pytest -k gradient`. If the finite difference test fails, check the factor of 2 and the division by `n`.
5. Implement `fit` and `predict` in `regression.py`. Start from zero weights. Store the loss before each update so that `loss_history` has one entry per iteration. Run `pytest -k "loss_decreases or closed"`.
6. Implement `fit_normal_equations` with `np.linalg.solve`. Run the full test suite and `python main.py`.
7. Add a function `r_squared(y_true, y_pred)` in `losses.py` that returns `1 - SS_res / SS_tot`. Add a test that checks it equals 1.0 when the predictions are exact and 0.0 when they always equal the mean of the targets.
8. Write `README.md` with the checklist below. Paste the final `pytest` line and the output of `main.py`. Add a table of learning rates `0.01`, `0.05`, `0.1` and `0.3` with the final MSE from each run, and say which ones diverge.

**README checklist**

- [ ] The project installs and runs with `python main.py` on a fresh environment
- [ ] `pytest` passes with at least eight tests, including the one you added for `r_squared`
- [ ] No ML framework is imported anywhere in the `mathlab` package
- [ ] The README contains the learning rate table from task 8, and it explains any divergence in one sentence
- [ ] The README explains in your own words why the closed form and gradient descent agree, and when you would prefer each
- [ ] The README names one limitation of MSE that applies to this dataset

**Done when**

- Every function in `mathlab` is implemented, and no `NotImplementedError` remains
- All tests pass, and the gradient test passes against the finite difference check
- `main.py` prints weights within 0.1 of the true weights `[4, -2, 0.5]`, and the closed form and gradient descent agree to three decimal places
- A reader can clone the repo, run three commands and see the same output you pasted in the README
- The repository is committed to GitHub with the code, tests and README

**Evidence to commit:** the full `mathlab` package, the tests, `main.py`, the README with the pytest output, the `main.py` output and the learning rate table.

---

## Check your understanding

1. A colleague says linear regression cannot fit a curved relationship. Is this correct? Explain with the design matrix, and give an example of a feature that would let the model fit a parabola.
2. Your gradient descent loss goes down for 50 iterations, then rises and turns into `nan`. What is the most likely cause? Which single change would you try first, and why would a larger learning rate make it worse?
3. Your gradient check shows that the analytic gradient differs from the finite difference by about 2 times in every component. What is the most likely mistake in the code, and where would you look?
4. The closed form and gradient descent give almost the same weights on the synthetic data. A teammate says gradient descent is therefore useless. Give two situations where you would still use gradient descent.
5. A house price dataset contains one listing at 100 times the normal price. The fitted slope changes a lot when you include it. Explain the mechanism using the squared residual, and name one change you could make to the loss or the data.
6. You report an MSE of 4.0 for a model that predicts a target with a standard deviation of 2.0. Is the model good? Use the relation between MSE and variance to justify your answer.

**Answers**

1. The statement is wrong. Linear regression is linear in its weights, not in its features. If you add a column `x^2` to the design matrix, the prediction `X @ w` is a parabola in `x`, and the same normal equations or gradient descent fit it. The limitation is that the model can only use the features and interactions you give it.
2. The most likely cause is a learning rate that is too large, so each step overshoots the valley and the loss grows until it overflows. Try reducing the learning rate by a factor of 10. A larger rate makes the overshoot bigger at every step, so the growth is faster. Also check the scale of the features, since large features make the gradient large.
3. The factor of 2 is the most likely mistake, because the derivative of the square produces a 2 that is easy to drop or double. Check the formula `(2 / n) X^T (X w - y)`, and check that the finite difference uses `h` of a sensible size and the central difference `(f(w + h) - f(w - h)) / (2h)`.
4. First, when the data is too large to fit in memory, so the normal equations cannot be formed. Second, when the model has no closed form, such as a neural network or a model with a nonlinear loss, where gradient descent is the only option. A third reason is a streaming setting, where data arrives over time and the weights must be updated as it comes in.
5. The squared residual grows with the square of the error, so the outlier's residual, which is huge, produces a loss term that dominates the sum. The gradient is proportional to the residual, so the fit is pulled toward the outlier. You could remove or cap the outlier after checking it is a data error, or switch to a robust loss such as the Huber loss.
6. Not enough information to say yes. The variance of the target is 2.0 squared, which is 4.0. An MSE of 4.0 means the model explains none of the variance, because it is as bad as predicting the mean. The model is no better than a constant. Compute R-squared to confirm, and compare with a simple baseline.

---

## Sources

**Read**

- Wikipedia: [Linear regression](https://en.wikipedia.org/wiki/Linear_regression)
- Wikipedia: [Least squares](https://en.wikipedia.org/wiki/Least_squares)
- Wikipedia: [Mean squared error](https://en.wikipedia.org/wiki/Mean_squared_error)
- Wikipedia: [Gradient descent](https://en.wikipedia.org/wiki/Gradient_descent)
- Wikipedia: [Matrix multiplication](https://en.wikipedia.org/wiki/Matrix_multiplication)
- Wikipedia: [Coefficient of determination](https://en.wikipedia.org/wiki/Coefficient_of_determination)
- Stanford University, CS229 Machine Learning course notes by Andrew Ng, the lecture notes on linear regression and the LMS algorithm. Search "CS229 lecture notes" on the course website.
- MIT OpenCourseWare, 18.06 Linear Algebra by Gilbert Strang, the lectures on matrix multiplication and least squares. Search "MIT OpenCourseWare 18.06 least squares".

**Tutorials**

- NumPy documentation: "NumPy quickstart" and the reference for `numpy.linalg.solve`. Search these titles on numpy.org.
- pytest documentation: "Get started" and "How to parametrize tests". Search these titles on docs.pytest.org.
- GeeksforGeeks: search "Linear Regression Python Implementation" on geeksforgeeks.org

**Watch**

- 3Blue1Brown, "Gradient descent, how neural networks learn" (Deep Learning series, chapter 2)
- 3Blue1Brown, "Matrix multiplication as composition" (Essence of Linear Algebra series)
- StatQuest with Josh Starmer, "Linear Regression, Clearly Explained" and "R-squared, Clearly Explained"

**Practice**

- Google Colab (colab.research.google.com). Create a notebook, paste the `mathlab` functions, and plot the loss history for three learning rates. Google Colab is free to use with a Google account.

---

## Tomorrow

Tomorrow begins Module 3, Data Engineering for AI, with Day 17 on NumPy. You now have the mathematics for a learning system: vectors and matrices to hold data, derivatives to move the weights, probability and information theory to define the loss and statistics to judge the result. Day 17 turns the array ideas you used in `mathlab` into a working toolkit, covering ndarrays, shapes and dimensions, so that the data is in the right form before any model sees it. The `mathlab` package you wrote today is the reference you can return to whenever a library function does something you do not understand.

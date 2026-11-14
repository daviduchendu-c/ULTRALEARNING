# Day 16: Mathematics Project

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 4 hours
**Level:** intermediate. This is the capstone of Module 2. You need the earlier days of this module and enough Python to write functions and tests. It is for readers who want proof that they can turn the mathematics into working code.

**By the end of this day, you will have:**

- A `mathlab` package with vector and matrix operations, mean squared error, its gradient, linear regression trained by gradient descent and a closed-form check, written without machine learning frameworks
- A test suite that compares each function with a trusted reference, and passes
- A training run whose weights match the true weights used to generate the data
- A README that explains the results and records the learning rates you tried

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`). NumPy is used only for arrays, random numbers and the reference check. You may not use scikit-learn, PyTorch, TensorFlow, JAX or any other machine learning framework in the project code.
- pytest 7 or newer (`pip install pytest`)
- Git and a free GitHub account for the repository

**Prerequisites:**

- Day 7: functions, summation notation and square roots
- Day 8: the dot product and the length of a vector
- Day 9: matrix shapes, multiplication and transposes
- Day 11: derivatives, the chain rule and gradients
- Day 12: gradient descent, the learning rate and the update loop
- Day 13 to Day 15: probability, statistics and cross-entropy, used to read the loss and the results

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Vectors and matrices from scratch](#vectors-and-matrices-from-scratch)
- [Linear regression](#linear-regression)
- [Loss calculation](#loss-calculation)
- [Gradient descent from scratch](#gradient-descent-from-scratch)
- [Hands-on exercises](#hands-on-exercises)
- [Project: mathlab, linear regression from scratch](#project-mathlab-linear-regression-from-scratch)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Over the last eight days you have met the mathematics behind machine learning one piece at a time. Vectors hold the features of an example. Matrices hold a dataset. Derivatives tell you which way to move the parameters. Probability and information theory define and interpret the loss. Today those pieces meet in one program. You will fit a line to data, measure how wrong it is, and walk the parameters downhill until the error stops shrinking.

On the job, engineers who can reproduce a model by hand and check it against a reference are the ones who debug training runs instead of guessing. A model you can trace to its equations is also a model you can explain to a client or a reviewer.

The big idea is that linear regression is the smallest complete learning system. It has data, a model with parameters, a loss, a gradient and an update rule. The training loops in deep learning repeat that same pattern with more layers and more parameters. Once you have built it once, the frameworks stop looking like magic.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1805 | Adrien-Marie Legendre publishes the method of least squares for fitting the orbits of comets. | The first published fit of a line to noisy measurements by minimizing squared error. |
| 1886 | Francis Galton describes "regression towards mediocrity" in hereditary stature. | The name of the model comes from a biology study. |
| 1960 | Bernard Widrow and Marcian Hoff introduce the LMS algorithm, an incremental update rule for adaptive linear units. | An early learning rule that is essentially gradient descent on squared error. |

**Takeaway:** linear regression is about two centuries old. The loop you write today is the ancestor of modern training code.

---

## Glossary

- **Feature**: one measured input to a model, such as the size of a house.
- **Target**: the value the model predicts, such as its price.
- **Design matrix**: one row per example and one column per feature, with a leading column of ones for the intercept.
- **Weights (parameters)**: the numbers the model learns, one per column of the design matrix.
- **Residual**: the target minus the prediction for one example.
- **Mean squared error (MSE)**: the average of the squared residuals.
- **Gradient**: the vector of partial derivatives of the loss with respect to the weights.
- **Closed form**: a formula that gives the answer directly, here `w = (X^T X)^-1 X^T y`.
- **Normal equations**: the linear system `(X^T X) w = X^T y` whose solution is the least squares answer.
- **Reference implementation**: a trusted library function that you compare your code against.
- **Unit test**: an automated check that one function gives the expected answer.
- **Finite difference**: an approximate derivative, computed by nudging an input by a small amount `h`.

---

## Vectors and matrices from scratch

A vector is an ordered list of numbers, and in machine learning it usually holds the features of one example. A house with 3 rooms, 120 square meters and a garden is `[3, 120, 1]`. A matrix stacks many vectors into rows, so 200 houses form a 200 by 3 matrix.

The dot product multiplies matching entries and adds the results. Matrix multiplication computes one dot product for each row of the first matrix and each column of the second. If `A` is 2 by 3 and `B` is 3 by 2, then `AB` is 2 by 2, and the inner sizes (3 and 3) must match.

```python
A = [[1, 2, 3], [4, 5, 6]]
B = [[1, 0], [0, 1], [1, 1]]

# entry (0, 0): row 0 of A dotted with column 0 of B
print(1 * 1 + 2 * 0 + 3 * 1)   # 4
# entry (1, 1): row 1 of A dotted with column 1 of B
print(4 * 0 + 5 * 1 + 6 * 1)   # 11
```

Matrix multiplication is not commutative, and transposing swaps rows and columns. The transpose `X^T` appears in the regression formulas, because `X^T X` combines every feature with every other feature.

The common mistake is to multiply shapes that do not line up. A 200 by 3 matrix times a length-3 vector gives 200 predictions. A 200 by 1 column vector in the same place produces a shape error or, worse, a silent broadcast with the wrong answer. Print the shapes before you combine arrays.

In the market, matrices are the native data type of machine learning. Recommendation systems store users and items as matrices, and GPUs are built to multiply them quickly.

---

## Linear regression

Linear regression predicts with a weighted sum of the features plus an intercept. In matrix form the prediction is `X @ w`, where the first column of `X` is all ones so that `w0` acts as the intercept.

Here is a tiny example. Three houses have sizes 50, 70 and 90 square meters and prices 150, 210 and 270 thousand. With weights `[30, 3]`, the predictions are `30 + 3 * 50 = 180`, `240` and `300`. Each residual is `-30`, so the model is consistently 30 thousand too low.

There are two ways to find the best weights. The closed form solves the normal equations `(X^T X) w = X^T y` in one step. Gradient descent starts from any guess and improves the weights step by step, which scales to large data and to models with no closed form. This project implements both so you can check one against the other.

Thinking a linear model cannot learn curves is a common mistake. It is linear in its weights, not its inputs. Adding a column `x^2` lets the same math fit a parabola. The honest limitation is that the model can only use the features and interactions you give it. If the residuals show a pattern, the model needs new features or a different model.

In the market, linear regression is the baseline for forecasting, pricing and risk work. A complicated model that cannot beat a well-built linear regression on the same features needs more work.

---

## Loss calculation

The standard loss for linear regression is mean squared error:

```
MSE = (1 / n) * sum over i of (y_i - prediction_i)^2
```

Squaring makes large errors count much more, and it makes the function smooth, which gradient descent needs. Here is the calculation for the house example:

```python
import numpy as np

y = np.array([150.0, 210.0, 270.0])
X = np.array([[1.0, 50.0], [1.0, 70.0], [1.0, 90.0]])
w = np.array([30.0, 3.0])

residuals = y - X @ w
print(residuals)            # [-30. -30. -30.]
print(np.mean(residuals ** 2))   # 900.0
```

MSE is in squared units. The root mean squared error, 30 here, is easier to read because it is in the units of the target. The common mistake is to compare losses across datasets with different scales. An MSE of 900 is excellent for prices in thousands and poor for a quantity between 0 and 1. Report R-squared, `1 - SS_res / SS_tot`, which is the fraction of variance the model explains.

The honest limitation of squared error is its sensitivity to outliers. One listing at ten times the normal price can dominate the loss and drag the weights toward it. A robust loss such as the Huber loss handles outliers better, at the cost of a less smooth gradient.

In the market, MSE is the default regression loss in most libraries, and choosing it without checking the business cost of errors is a common and costly shortcut.

---

## Gradient descent from scratch

The gradient of MSE for a linear model has a clean form:

```
gradient = (2 / n) * X^T (X w - y)
```

`X w - y` is the vector of residuals. Multiplying by `X^T` combines each residual with the features that produced it. Dividing by `n` averages over examples, and the 2 comes from differentiating the square. The update is `w_new = w_old - learning_rate * gradient`.

Here is one full step on a small dataset. With `X = [[1, 1], [1, 2]]`, `y = [2, 3]` and `w = [0, 0]`, the residuals are `[-2, -3]`. Then `X^T` times the residuals is `[-5, -8]`, and with `n = 2` the gradient is `[-5, -8]`. A learning rate of 0.1 gives `[0.5, 0.8]`.

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

Repeating the step is the training loop. Record the loss at each iteration. If it grows, the learning rate is too large (Day 12). If it is flat from the start, the gradient may be wrong. The common mistake is a sign or scale error that the falling loss hides. A finite difference check catches it: compute `(loss(w + h) - loss(w - h)) / (2h)` for a small `h` and compare it with the analytic gradient. The tests in this project do exactly that.

The honest limitation is that full-batch gradient descent reads every row at each step, which is slow for millions of rows. Day 12's mini-batch methods are used then. In the market, frameworks compute gradients automatically, but engineers who can derive and check a gradient by hand are the ones who notice when an automatic gradient or a custom loss is wrong.

---

## Hands-on exercises

Create a folder called `day16-warmup` and write your code in `warmup.py`. Do each step on paper first.

1. **Dot product.** Compute `[2, -1, 4] · [3, 5, 0]` by hand. Write a loop-based `dot(a, b)` and confirm it matches `np.dot`. The answer is 1.

2. **Matrix product.** Multiply `A = [[1, 2], [0, 1], [3, 4]]` by `B = [[2, 0, 1], [1, 1, 0]]` on paper, then check with `A @ B`. Predict the shape of `B @ A` before computing it.

3. **MSE.** Targets `[5, 7, 9]` and predictions `[4, 7, 11]`. Compute the MSE by hand and confirm with NumPy. Write one sentence on which example contributes most.

4. **One gradient step.** Use `X = [[1, 1], [1, 2]]`, `y = [2, 3]`, `w = [0, 0]` and a learning rate of 0.1. Compute the gradient and the new weights by hand, then check them in code.

5. **Finite difference.** For the same data, compute the loss at `w = [0, 0]` and the numerical gradient by nudging each weight by `h = 1e-6`. Confirm it matches `[-5, -8]` to about six decimal places.

6. **Predict a line.** Use the three houses from the lesson with `w = [30, 3]` and confirm the MSE is 900. Then set `w = [0, 3]`. The new MSE is 0, because the houses lie exactly on `y = 3x`. Write one sentence on whether the intercept or the slope mattered more.

**Evidence to commit:** `warmup.py` and `notes.md` with your hand calculations and sentences.

---

## Project: mathlab, linear regression from scratch

This project is practical and is the module's capstone. You will implement a small machine learning library with no framework and test every piece against a trusted reference. Name the repository `100-days-ai-engineering-day-16-mathematics-project`.

The starter gives you the data generator, the tests and the main script. The functions you must write raise `NotImplementedError`. At the start, all seven tests fail, and when you are done they should all pass.

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

`mathlab/vectors.py` (implement every function, with lists as vectors and lists of rows as matrices):

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

`mathlab/losses.py` (implement with NumPy arrays):

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

`mathlab/regression.py` (implement `fit`, `predict` and `fit_normal_equations`):

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
    """Closed-form answer. Solve (X^T X) w = X^T y with np.linalg.solve."""
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

With a correct implementation, `python main.py` prints values close to these. The weights match the true weights `[4, -2, 0.5]` within noise, and the closed form matches gradient descent:

```
true    [ 4.  -2.   0.5]
gd      [ 4.004 -1.972  0.554]
closed  [ 4.004 -1.972  0.554]
final mse 0.2353
```

**Step-by-step tasks**

1. Create the repository, the folder tree and a virtual environment. Install the requirements and run `pytest`. All seven tests should fail with `NotImplementedError`.
2. Implement `dot`, `add`, `scale`, `norm` and `unit`. Run `pytest -k "dot or norm"` after each one.
3. Implement `transpose` and `matmul`. Use `dot` inside `matmul`. Run `pytest -k matmul`.
4. Implement `predict`, `mse` and `mse_gradient`. Run `pytest -k gradient`. If the finite difference test fails, check the factor of 2 and the division by `n`.
5. Implement `fit` and `predict`. Record the loss before each update so that `loss_history` has one entry per iteration. Run `pytest -k "loss_decreases or closed"`.
6. Implement `fit_normal_equations` with `np.linalg.solve`. Run the full suite and `python main.py`.
7. Add `r_squared(y_true, y_pred)` to `losses.py` that returns `1 - SS_res / SS_tot`. Add a test that it equals 1.0 for exact predictions and 0.0 when the predictions always equal the mean of the targets.
8. Write `README.md` with the checklist below. Paste the final pytest line and the `main.py` output. Add a table of learning rates `0.01`, `0.05`, `0.1` and `0.3` with the final MSE for each, and say which ones diverge.

**README checklist**

- [ ] The project runs with `python main.py` on a fresh environment
- [ ] `pytest` passes with at least eight tests, including the `r_squared` test
- [ ] No ML framework is imported anywhere in the `mathlab` package
- [ ] The README contains the learning rate table and explains any divergence in one sentence
- [ ] The README explains in your own words why the closed form and gradient descent agree, and when you would prefer each
- [ ] The README names one limitation of MSE that applies to this dataset

**Done when**

- Every function in `mathlab` is implemented, and no `NotImplementedError` remains
- All tests pass, including the finite difference check
- `main.py` prints weights within 0.1 of `[4, -2, 0.5]`, and the closed form and gradient descent agree to three decimal places
- A reader can clone the repo, run three commands and see the output you pasted in the README
- The repository is committed to GitHub with the code, tests and README

**Evidence to commit:** the `mathlab` package, the tests, `main.py`, and the README with the pytest output, the `main.py` output and the learning rate table.

---

## Check your understanding

1. A colleague says linear regression cannot fit a curved relationship. Is this correct? Explain using the design matrix.
2. Gradient descent loss falls for 50 iterations, then rises and turns into `nan`. What is the likely cause, and what would you change first?
3. Your analytic gradient differs from the finite difference by a factor of about 2 in every component. What is the most likely mistake?
4. The closed form and gradient descent give almost the same weights on this data, so a teammate says gradient descent is useless. Give two situations where you would still use it.
5. One house listing is 100 times the normal price, and the fitted slope changes a lot when it is included. Explain the mechanism using the squared residual, and name one fix.
6. A model has an MSE of 4.0 on a target with standard deviation 2.0. Is it good? Justify with the relation between MSE and variance.

**Answers**

1. Incorrect. The model is linear in its weights, not its features. Adding a column `x^2` makes `X @ w` a parabola in `x`, and the same normal equations or gradient descent fit it.
2. A learning rate that is too large. Each step overshoots, the loss grows and overflows. Reduce the rate by about a factor of 10. A larger rate makes the growth faster. Also check for unscaled features.
3. The factor of 2 from the derivative of the square, which is easy to drop or double. Check `(2 / n) X^T (X w - y)` and the central difference formula.
4. When the data is too large for the normal equations to be formed in memory, and when the model has no closed form, as in neural networks. A third case is streaming data, where the weights must be updated as examples arrive.
5. The squared residual of the outlier is huge, so its term dominates the sum, and the gradient is proportional to the residual, which pulls the fit toward it. Check whether the listing is a data error and remove it if so, or switch to a robust loss such as Huber.
6. Not enough to say it is good. The variance of the target is `2.0^2 = 4.0`, so an MSE of 4.0 means the model is no better than predicting the mean. Compute R-squared to confirm it is zero.

---

## Sources

**Read**

- Wikipedia: [Linear regression](https://en.wikipedia.org/wiki/Linear_regression)
- Wikipedia: [Least squares](https://en.wikipedia.org/wiki/Least_squares)
- Wikipedia: [Mean squared error](https://en.wikipedia.org/wiki/Mean_squared_error)
- Wikipedia: [Gradient descent](https://en.wikipedia.org/wiki/Gradient_descent)
- Wikipedia: [Matrix multiplication](https://en.wikipedia.org/wiki/Matrix_multiplication)
- Wikipedia: [Coefficient of determination](https://en.wikipedia.org/wiki/Coefficient_of_determination)
- Stanford University, CS229 Machine Learning lecture notes by Andrew Ng, the notes on linear regression and the LMS algorithm. Search "CS229 lecture notes" on the course website.
- MIT OpenCourseWare, 18.06 Linear Algebra by Gilbert Strang, the lectures on matrix multiplication and least squares. Search the course title on ocw.mit.edu.

**Tutorials**

- NumPy documentation: "NumPy quickstart" and the reference for `numpy.linalg.solve`. Search these titles on numpy.org.
- pytest documentation: "Get started" and "How to parametrize tests". Search these titles on docs.pytest.org.
- GeeksforGeeks: search "Linear Regression Python Implementation" on geeksforgeeks.org

**Watch**

- 3Blue1Brown, "Gradient descent, how neural networks learn" (Deep Learning series, chapter 2)
- 3Blue1Brown, "Matrix multiplication as composition" (Essence of Linear Algebra series)
- StatQuest with Josh Starmer, "Linear Regression, Clearly Explained" and "R-squared, Clearly Explained"

**Practice**

- Google Colab (colab.research.google.com), a free notebook environment with a Google account. Paste the `mathlab` functions and plot the loss history for three learning rates.

---

## Tomorrow

Tomorrow begins Module 3, Data Engineering for AI, with Day 17 on NumPy. You now have the mathematics for a learning system: vectors and matrices to hold data, derivatives to move the weights, probability and information theory to define the loss, and statistics to judge the result. Day 17 turns the array ideas you used in `mathlab` into a working toolkit for ndarrays, shapes and dimensions, so the data is in the right form before any model sees it.

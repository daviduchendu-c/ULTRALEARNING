# Day 11: Calculus I

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** intermediate. You need Day 7's functions and Day 8's vectors. You do not need prior calculus, but you should be willing to read a limit as a process and not as a result you look up.

**By the end of this day, you will have:**

- A working definition of a derivative that you can explain from the idea of a limit and a slope
- A Python function that estimates derivatives numerically with a central difference, and a check that it agrees with the rules you derive by hand
- A small gradient calculation for a linear model's mean squared error, verified against the numerical estimate, plus one step of gradient descent that lowers the loss

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 2.0 or newer (`pip install numpy`), free and open source under the BSD license
- pytest 8.0 or newer (`pip install pytest`)
- A code editor such as VS Code (free) or any editor you already use
- Git and a free GitHub account for the project section
- Paper, a pencil and graph paper for the slope sketches

**Prerequisites:**

- Day 7: functions, exponents, logarithms and summation
- Day 8: vectors and the dot product
- Day 9: matrices, at least as a way of storing many numbers at once
- Basic Python: functions, loops and floating-point numbers

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Limits concept](#limits-concept)
- [Slopes](#slopes)
- [Derivatives](#derivatives)
- [Partial derivatives](#partial-derivatives)
- [Chain rule](#chain-rule)
- [Gradients](#gradients)
- [Connecting derivatives to learning](#connecting-derivatives-to-learning)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a numerical gradient checker](#project-a-numerical-gradient-checker)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Training a model is a search. You start with a set of weights, measure how wrong the model is, and adjust the weights so the error drops. The question that drives the adjustment is simple to state: if I nudge this weight a little, does the error go up or down, and by how much? Calculus is the language that answers it. The rate at which an output changes as an input changes is a derivative, and the collection of those rates for every weight at once is a gradient.

You do not need to compute derivatives by hand to use them. Libraries such as PyTorch and JAX compute gradients automatically. But when a training run diverges, stalls or gives a gradient of `nan`, you need to know what the number means, which way it points and how to check it. A gradient checker, which compares an automatic gradient against a finite-difference estimate, is one of the most useful debugging tools in a deep learning codebase, and today you build one.

On the job, derivatives appear when you write a custom loss function, when you tune a learning rate by looking at how fast the loss falls, when you read a paper that defines a training objective, and when a model's gradients explode and you need to know why. This day gives you the first-principles picture that every one of those tasks depends on.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1660s | Isaac Newton develops his method of fluxions, which treats quantities as flowing in time, and uses it to study motion. | The first systematic calculus of rates of change. |
| 1684 | Gottfried Wilhelm Leibniz publishes his differential calculus, with the notation `dy/dx` that is still used. | The notation you will use for derivatives and partial derivatives. |
| 1687 | Newton publishes the *Principia*, which applies calculus to planetary motion. | Showed that rates of change explain physical laws. |
| 1821 | Augustin-Louis Cauchy publishes a course that defines the limit carefully, as the basis for derivatives and integrals. | The definition of a limit that this day uses. |
| 1986 | David Rumelhart, Geoffrey Hinton and Ronald Williams publish a paper that popularizes backpropagation, which applies the chain rule to train multi-layer networks. | The algorithm that trains nearly every neural network today. |

**Takeaway:** the idea of a derivative took about two centuries to become rigorous, and the algorithm that trains today's models is the chain rule applied systematically, which Leibniz would recognize.

---

## Glossary

- **Function of one variable**: a rule that takes one number and returns one number, such as `f(x) = x^2`.
- **Limit**: the value a function approaches as its input approaches a point, written `lim as h goes to 0`.
- **Secant line**: a line through two points on a curve.
- **Tangent line**: the line that touches a curve at one point and has the same slope there as the curve.
- **Slope**: the rise divided by the run, the change in output per unit change in input.
- **Difference quotient**: the ratio `(f(x + h) - f(x)) / h`, the slope of the secant line over an interval of width `h`.
- **Derivative**: the limit of the difference quotient as `h` goes to 0. It is the slope of the tangent line, and it is written `f'(x)` or `df/dx`.
- **Partial derivative**: the derivative of a function of several variables with respect to one of them, with the others held fixed. Written `∂f/∂x`.
- **Chain rule**: the rule for the derivative of a composition, `d/dx f(g(x)) = f'(g(x)) g'(x)`.
- **Gradient**: the vector of all partial derivatives of a function of several variables.
- **Finite difference**: an approximation of a derivative from function values at nearby points, without a formula for the derivative.
- **Central difference**: a finite difference that uses points on both sides of `x`, `(f(x + h) - f(x - h)) / (2h)`, which is more accurate than the one-sided version.
- **Loss function**: a function that measures how wrong a model's predictions are. Training minimizes it.
- **Gradient descent**: the method of repeatedly moving a parameter in the direction opposite to its gradient.

---

### Limits concept

Suppose you are approaching a wall and want to know how close you can get. You take steps that are half as long as the last one, and you see where you are heading. A limit asks what value a function is heading toward as its input gets closer and closer to a point, without asking what the function does exactly at that point.

We write `lim as x goes to a of f(x) = L` when `f(x)` can be made as close to `L` as we like by taking `x` close enough to `a` but not equal to `a`. The limit ignores the value at `a` itself, which is why it can handle expressions that look undefined at that point, such as `0/0`. Calculus needs limits because the derivative is defined by one: it asks what happens to a slope over an interval as the interval shrinks to nothing.

Here is the function `f(x) = x^2` near `x = 3`, with the interval shrinking:

```python
f = lambda x: x ** 2
x = 3.0
for h in [0.5, 0.1, 0.01, 0.001]:
    quotient = (f(x + h) - f(x)) / h
    print(h, round(quotient, 4))
# 0.5 6.5, 0.1 6.1, 0.01 6.01, 0.001 6.001
```

The values approach 6. That is the limit of the difference quotient at `x = 3`, and it will turn out to be the derivative.

Treating a limit as a plug-in is a common mistake. Substituting `h = 0` into `(f(x + h) - f(x)) / h` gives `0/0`, which is not an answer. The limit is the value the expression approaches, found by simplifying first. The second mistake is trusting floating-point arithmetic near zero. In Python, `(3 + 1e-16)**2 - 9` returns `0.0`, because `3 + 1e-16` rounds to `3.0` in double precision, so the numerical quotient is zero, not 6. The step size must be small but not so small that the arithmetic loses the signal. In the real market, every automatic differentiation system avoids this problem by applying the rules of calculus symbolically rather than by taking tiny steps, which is why gradient checkers are a check on the machinery and not a replacement for it.

### Slopes

A hill you can walk up has a slope. On a gentle hill, each step forward changes your height a little. On a steep hill, the same step changes your height a lot. The slope is the ratio of the change in height to the distance you walked.

From first principles, for two points `(a, f(a))` and `(b, f(b))` on a curve, the secant slope is `(f(b) - f(a)) / (b - a)`. A straight line has the same slope everywhere, `m = (y2 - y1) / (x2 - x1)`. A curve does not. Its slope changes from point to point, and the slope at one point is the slope of the tangent line there. Moving `b` closer to `a` turns the secant into the tangent, which is the limit idea from the previous section. The derivative is the function that returns the tangent slope at every point.

```python
f = lambda x: x ** 2
a = 3.0
for b in [4.0, 3.5, 3.1, 3.01]:
    print(b, (f(b) - f(a)) / (b - a))
# 7.0, 6.5, 6.1, 6.01 -- the secant slopes approach 6, the tangent slope at 3
```

One mistake is reading a slope off a graph with unequal axes. A curve that looks steep can be gentle once you account for the scale, so always label both axes and check the units. Another mistake is assuming that a steep secant means a steep tangent. The secant is an average over an interval, and the tangent is the rate at one point. On a function that rises and then falls, the average over a wide interval can be zero while the slope at the peak is also zero, which hides the actual shape. In the real market, the slope of a loss curve over the last hundred training steps is the first thing an engineer checks when deciding whether to keep training, and it is a secant slope over a window.

### Derivatives

A derivative is the instantaneous rate of change. A car that covers 60 kilometers in an hour has an average speed of 60 kilometers per hour, but its speedometer shows its speed at each moment. The derivative of position with respect to time is the speedometer reading.

The derivative of `f` at `x` is `f'(x) = lim as h goes to 0 of (f(x + h) - f(x)) / h`, when the limit exists. Repeating this for every `x` gives a new function `f'`. Some rules come straight from the definition and save you from computing limits by hand. The power rule says `d/dx x^n = n x^(n-1)`. The constant multiple rule says `d/dx (c f) = c f'`. The sum rule says `d/dx (f + g) = f' + g'`. The derivative of `e^x` is `e^x`, and the derivative of `ln(x)` is `1/x`.

For `f(x) = 3x^2 + 2x + 1`, the rules give `f'(x) = 6x + 2`. At `x = 2` the derivative is 14. You can check that with the difference quotient:

```python
def f(x):
    return 3 * x**2 + 2 * x + 1

def fprime(x):
    return 6 * x + 2

h = 1e-6
print(fprime(2.0))                   # 14.0
print((f(2.0 + h) - f(2.0)) / h)     # about 14.000003
```

A common mistake is applying the power rule to a constant or treating `x^n` and `n^x` as the same thing. The derivative of `2^x` is `2^x ln(2)`, not `x 2^(x-1)`. Another mistake is confusing the derivative with the function value. At `x = 2`, `f(2) = 17` and `f'(2) = 14`. The value tells you where you are, and the derivative tells you how fast you are moving. In industry, the derivative of an activation function, such as the sigmoid or ReLU, is computed in every backward pass of a neural network, and a bug in that derivative is a common cause of a model that trains slowly or not at all.

### Partial derivatives

Think of the price of a meal as a function of two inputs, the number of guests and the price per dish. To ask how the cost changes when you add a guest, you hold the price per dish fixed. That is a partial derivative: the rate of change with respect to one input, with the others held constant.

For `f(x, y)`, the partial derivative with respect to `x` is `∂f/∂x`, computed by treating `y` as a constant and differentiating in `x` as usual. The partial derivative with respect to `y` treats `x` as constant. Each is a slope along one axis of a surface, and together they describe how the surface tilts in each direction. For `f(x, y) = x^2 y + 3y`, we get `∂f/∂x = 2xy` and `∂f/∂y = x^2 + 3`. At `(2, 1)`, these equal 4 and 7.

```python
def F(x, y):
    return x**2 * y + 3 * y

x, y = 2.0, 1.0
h = 1e-6
print((F(x + h, y) - F(x, y)) / h)  # about 4.0, the partial in x
print((F(x, y + h) - F(x, y)) / h)  # about 7.0, the partial in y
```

Forgetting to hold the other variables fixed is a common mistake. If you differentiate `x^2 y` with respect to `x` and treat `y` as if it changes too, the answer is wrong. Another mistake is assuming that the partial derivatives describe the slope in every direction. They describe the slope along the two axes only. A surface can have zero partial derivatives at a point and still rise steeply in a diagonal direction, and a point where both partials are zero may be a maximum, a minimum or a saddle. In the market, every parameter in a neural network has its own partial derivative of the loss, and the training code computes millions of them in one pass.

### Chain rule

A relay race is a useful picture. The runner at the end of the course moves as fast as the runner before them, times the speed of the handoff. The chain rule works the same way for functions that are nested inside other functions. The total rate of change is the product of the rates along the chain.

If `y = f(u)` and `u = g(x)`, then `dy/dx = (dy/du) (du/dx)`. In function notation, `d/dx f(g(x)) = f'(g(x)) g'(x)`. The derivative of the outer function is evaluated at the inner function's value, and then multiplied by the derivative of the inner function. For `f(x) = (3x + 1)^2`, the outer function is `u^2` and the inner function is `3x + 1`. The outer derivative is `2u`, the inner derivative is 3, so `f'(x) = 2(3x + 1) * 3 = 6(3x + 1)`. At `x = 2`, that gives `6 * 7 = 42`.

```python
def g(x):
    return (3 * x + 1) ** 2

x = 2.0
h = 1e-6
analytic = 6 * (3 * x + 1)          # chain rule result
numeric = (g(x + h) - g(x)) / h     # finite-difference check
print(analytic, round(numeric, 4))  # 42.0 42.0
```

A common mistake is forgetting the inner derivative. Many beginners differentiate `(3x + 1)^2` as `2(3x + 1)` and stop, which is the most frequent error in hand-derived gradients and one that a numerical check catches immediately. The chain rule is also why deep networks are hard to train: the gradient of the loss with respect to an early layer is a product of many derivatives, and if each factor is much smaller than one, the product vanishes, while if each is much larger than one, it explodes. That phenomenon is called the vanishing or exploding gradient problem, and it is the main reason later architectures added normalization and residual connections. In the market, backpropagation is the chain rule applied in reverse order over the computation graph, and every framework's autograd engine does exactly that.

### Gradients

Stand on a hillside in fog. You cannot see the valley, but you can feel the ground under your feet and tell which direction is steepest uphill. The gradient is that direction. It is a vector that collects the partial derivatives, and it points in the direction in which the function increases fastest.

The gradient of `f(x, y)` is `∇f = (∂f/∂x, ∂f/∂y)`. For a function of `n` variables it is a vector with `n` entries, one for each input. Its direction is the direction of steepest increase, and its magnitude is the rate of increase in that direction. The rate of change in any unit direction `u` is the dot product `∇f . u`, which is the connection to Day 8. The dot product is largest when `u` points along the gradient. Moving in the direction of `-∇f` decreases the function fastest, and that is the basis of gradient descent.

At `(2, 1)` for `f(x, y) = x^2 y + 3y`, the gradient is `(4, 7)`. Moving in the direction `(-4, -7)` lowers the function fastest, and a small step of size `0.01` in that direction changes `f` by about `-0.01 * ||(4, 7)||^2`, which is `-0.01 * 65 = -0.65`, to first order. That approximation holds only when the step is small.

```python
import numpy as np

def F(v):
    x, y = v
    return x**2 * y + 3 * y

def grad_F(v):
    x, y = v
    return np.array([2 * x * y, x**2 + 3])

v = np.array([2.0, 1.0])
print(grad_F(v))                       # [4. 7.]
step = 0.01
print(F(v), F(v - step * grad_F(v)))   # the function drops after a step downhill
```

A common mistake is assuming the gradient points toward the minimum. It points uphill, and gradient descent moves against it. Another mistake is using a gradient computed at one point as if it described the whole function. The gradient is a local linear approximation. Far from the point, the function can curve away from the linear prediction, so a step that is too large can overshoot and raise the loss. A third mistake is confusing a zero gradient with a minimum. A zero gradient marks a flat spot, which can be a minimum, a maximum or a saddle point. In industry, the gradient vector is what a training framework returns from `loss.backward()` in PyTorch, and the optimizer uses it to decide each weight update, so understanding the gradient's direction is understanding the update.

### Connecting derivatives to learning

Learning, in the sense used by machine learning, is adjusting parameters to lower a loss. The derivative of the loss with respect to a parameter tells you how that parameter affects the loss. If the derivative is positive, increasing the parameter increases the loss, so you should decrease the parameter. If it is negative, you should increase it. The size of the derivative tells you how strongly to respond.

Here is the simplest real case. Fit a line `y_hat = w x + b` to three points, `(1, 3)`, `(2, 5)` and `(3, 7)`, by minimizing the mean squared error `L(w, b) = mean((w x + b - y)^2)`. The partial derivatives are `∂L/∂w = mean(2 (w x + b - y) x)` and `∂L/∂b = mean(2 (w x + b - y))`. At `w = 0` and `b = 0`, the predictions are all zero, and the gradient is `(-22.67, -10)`. Both derivatives are negative, so increasing `w` and `b` lowers the loss. A learning rate of 0.1 turns that into a step: `w` becomes `0 - 0.1 * (-22.67) = 2.27` and `b` becomes `0 - 0.1 * (-10) = 1.0`. The loss falls from about 27.67 to about 0.33.

```python
import numpy as np

xs = np.array([1.0, 2.0, 3.0])
ys = np.array([3.0, 5.0, 7.0])

def loss(w, b):
    return np.mean((w * xs + b - ys) ** 2)

def grad(w, b):
    pred = w * xs + b
    dw = np.mean(2 * (pred - ys) * xs)
    db = np.mean(2 * (pred - ys))
    return dw, db

w, b = 0.0, 0.0
print(round(loss(w, b), 2))            # 27.67
dw, db = grad(w, b)
w, b = w - 0.1 * dw, b - 0.1 * db
print(round(w, 4), round(b, 4))        # 2.2667 1.0
print(round(loss(w, b), 2))            # 0.33
```

The limitation is that one step does not produce the best fit. The loss is still falling, and the right line is `y = 2x + 1`, which this data follows exactly. The learning rate of 0.1 is a guess, and a larger one could overshoot. Day 12 turns this single step into a loop and studies how the learning rate affects convergence.

Reading the gradient's sign as the amount to change the parameter is a common mistake. The gradient gives direction and relative size, and the learning rate sets the actual step. Skipping the learning rate, or setting it too large, is the usual cause of a loss that jumps up and down. In industry, every fine-tuning job, every regression that is solved iteratively and every custom loss in a research repository reduces to this pattern.

---

## Hands-on exercises

Work the first four on paper, then check each one numerically.

1. Compute the difference quotient `(f(x + h) - f(x)) / h` by hand for `f(x) = x^3` at `x = 2` with `h = 0.1`. Then simplify the expression algebraically as far as you can, and take the limit as `h` goes to 0. Confirm the result is 12.
2. Find the derivative of `f(x) = 4x^3 - 5x + 7` by the power, constant multiple and sum rules. Evaluate it at `x = 1` and compare with a central difference using `h = 1e-5`.
3. For `f(x, y) = 3x y^2 + x`, find `∂f/∂x` and `∂f/∂y` by hand. Evaluate both at `(1, 2)`, then verify with finite differences.
4. Differentiate `h(x) = sqrt(x^2 + 1)` with the chain rule. Evaluate at `x = 1` and check numerically. Then explain why the inner derivative must be included.
5. Write a function `central_diff(f, x, h=1e-5)` that returns the central difference. Test it on `f(x) = x^2` at `x = 3`, where the answer is 6, and compare its error with the one-sided version `(f(x + h) - f(x)) / h`. Report which is more accurate and why.
6. Using the linear model from the lesson, compute the gradient at `w = 2` and `b = 1`. Then take one gradient step with learning rate 0.05 and report the new loss. Explain whether it went down.
7. Test the effect of the learning rate. Starting at `w = 0` and `b = 0`, run 20 steps of gradient descent with learning rates 0.01, 0.1 and 0.5, and print the final loss for each. Describe which rate converges, which is slow and which diverges, if any.

Record any place where your hand result and the numerical result disagree, and explain the gap. Most gaps come from step sizes that are too large or too small.

---

## Project: a numerical gradient checker

This is a light project, and the reason is deliberate. Today's skill is understanding the derivative and checking it, not building a training loop. The code is a small set of functions, and the value is in the tests that compare the analytic gradient with the numerical one. Gradient descent as a full loop arrives on Day 12 and the complete implementation on Day 16.

**Repo name:** `100-days-ai-engineering-day-11-calculus-i`

**Goal:** write a central-difference derivative estimator, derive the gradient of a mean squared error loss by hand, implement it, and test that the two agree.

**Repo layout:**

```
100-days-ai-engineering-day-11-calculus-i/
├── README.md
├── requirements.txt
├── derivatives.py
├── mse_gradient.py
└── tests/
    └── test_gradients.py
```

**Starter code for `derivatives.py`:**

```python
import numpy as np


def central_diff(f, x, h=1e-5):
    """Return the central-difference estimate of f'(x) for a scalar function f."""
    raise NotImplementedError


def numerical_gradient(f, v, h=1e-5):
    """Return the numerical gradient of f at the vector v, one central difference per entry."""
    raise NotImplementedError
```

**Starter code for `mse_gradient.py`:**

```python
import numpy as np


def mse_loss(w, b, x, y):
    """Return mean((w x + b - y)^2)."""
    raise NotImplementedError


def mse_grad(w, b, x, y):
    """Return the analytic gradient (dL/dw, dL/db) of mse_loss.

    Derive it by hand first: dL/dw = mean(2 (pred - y) x) and dL/db = mean(2 (pred - y)).
    """
    raise NotImplementedError
```

**Starter code for `tests/test_gradients.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from derivatives import central_diff, numerical_gradient
from mse_gradient import mse_loss, mse_grad

XS = np.array([1.0, 2.0, 3.0])
YS = np.array([3.0, 5.0, 7.0])


def test_central_diff_of_square_at_3():
    assert central_diff(lambda x: x**2, 3.0) == pytest.approx(6.0, abs=1e-6)


def test_numerical_gradient_matches_known_answer():
    g = numerical_gradient(lambda v: v[0] ** 2 * v[1] + 3 * v[1], np.array([2.0, 1.0]))
    assert np.allclose(g, [4.0, 7.0], atol=1e-4)


def test_mse_grad_at_zero_matches_hand_calculation():
    dw, db = mse_grad(0.0, 0.0, XS, YS)
    assert dw == pytest.approx(-68.0 / 3.0)
    assert db == pytest.approx(-10.0)


def test_analytic_grad_matches_numerical_grad():
    w, b = 1.3, -0.4
    analytic = np.array(mse_grad(w, b, XS, YS))
    numeric = numerical_gradient(lambda v: mse_loss(v[0], v[1], XS, YS), np.array([w, b]))
    assert np.allclose(analytic, numeric, atol=1e-4)


def test_one_step_lowers_loss():
    w, b = 0.0, 0.0
    dw, db = mse_grad(w, b, XS, YS)
    assert mse_loss(w - 0.1 * dw, b - 0.1 * db, XS, YS) < mse_loss(w, b, XS, YS)
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy` and `pytest`.
2. On paper, derive `dL/dw` and `dL/db` for the mean squared error. Write the derivation in the README, with each step. Use the chain rule and state where it applies.
3. Implement `central_diff` as `(f(x + h) - f(x - h)) / (2 h)`, and `numerical_gradient` as a loop over the entries of `v`, perturbing one entry at a time.
4. Implement `mse_loss` and `mse_grad` from your derivation.
5. Run `pytest`. If the analytic and numerical gradients disagree, first check the chain rule factor in `mse_grad`, which is the most common mistake.
6. Add a test of your own that checks `central_diff` against the power rule for `x**3` at `x = 2`, where the answer is 12.
7. Write the README with the derivation, the test output, a note on why the central difference is more accurate than the one-sided version, and one sentence on why a gradient checker is useful in a training codebase.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is written in one sentence
- [ ] Install and run commands are listed (`pip install -r requirements.txt`, `pytest`)
- [ ] The derivation of `dL/dw` and `dL/db` is written out step by step
- [ ] The README explains why the central difference error is smaller than the one-sided error
- [ ] One limitation of finite differences is named, such as the cost of perturbing every parameter and the sensitivity to the step size

**Done when:**

- `pytest` passes all six tests
- `mse_grad` at `(0, 0)` returns `(-22.67, -10.0)` to two decimal places
- The analytic and numerical gradients agree within `1e-4` at a random point
- The README derivation matches the code line by line
- The README states what happens when `h` is set too small and too large

**What to commit as evidence:** the two source files, the test file, the pytest output pasted into the README, and the derivation.

---

## Check your understanding

1. A student computes the derivative of `f(x) = (2x - 5)^3` as `3(2x - 5)^2`. Explain the mistake, give the correct derivative, and describe how a numerical check would reveal the error.
2. The difference quotient at `x = 1` for `f(x) = x^2` is `(2 + h)` for small `h`. Explain why the limit is 2, and explain why setting `h = 0` directly gives an undefined expression.
3. A function `f(x, y)` has `∂f/∂x = 0` and `∂f/∂y = 0` at a point. Can you conclude that the point is a minimum? Name the three possibilities and give a reason the test is not enough.
4. The gradient of a loss at the current weights is `(3, -4)`. Which direction should you move the weights, and what is the rate of increase of the loss in the direction of the gradient?
5. A training run shows a loss of `nan` after a few steps. Describe two causes that are visible in the derivative and one check you would run first.
6. Explain why the chain rule is needed to compute the gradient of a loss with respect to an early layer's weights in a deep network, and say what the vanishing gradient problem is in terms of a product of derivatives.
7. You set the learning rate to 5 for a loss whose gradient at the start is `(-22.67, -10)`. Predict qualitatively what happens over the first few steps, and explain using the gradient direction and the size of the step.

### Answers

1. The student forgot the inner derivative. The derivative of `(2x - 5)^3` is `3(2x - 5)^2 * 2 = 6(2x - 5)^2`. A central-difference check at any point, for example `x = 3`, would give a value that differs from the student's answer by a factor of 2, which exposes the error.
2. The difference quotient `((1 + h)^2 - 1) / h` simplifies to `(2h + h^2) / h = 2 + h`, which approaches 2 as `h` goes to 0. Setting `h = 0` directly gives `0/0`, which has no value, so the limit must be found by simplifying first.
3. No. A zero gradient marks a critical point, which may be a minimum, a maximum or a saddle point. The gradient alone cannot tell them apart, and you need second-order information or the values of the function nearby to decide.
4. Move the weights in the direction opposite to the gradient, so the change is proportional to `(-3, 4)`, to reduce the loss fastest. The rate of increase of the loss in the direction of the gradient is the magnitude of the gradient, `sqrt(3^2 + 4^2) = 5`, per unit step.
5. Two common causes are an exploding gradient, where the derivatives grow very large because the chain rule multiplies many factors above one, and a division by zero or a logarithm of zero in the loss or its derivative. A first check is to print the gradient norm at each step and look for the step where it jumps, then check the learning rate and any functions with domain limits.
6. An early layer affects the loss through every layer after it, so its derivative is a product of the derivatives of each stage, which is the chain rule. The vanishing gradient problem occurs when many of those factors are much smaller than one, so the product becomes tiny and early layers learn almost nothing.
7. The step is `5` times the gradient, which is about `(-113, -50)`, so the weights jump far past the point where the loss is minimized. The loss will probably rise rather than fall, and it may oscillate or diverge over the first few steps, because the step overshoots the local linear approximation.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Limit of a function](https://en.wikipedia.org/wiki/Limit_of_a_function)
- Wikipedia: [Derivative](https://en.wikipedia.org/wiki/Derivative)
- Wikipedia: [Partial derivative](https://en.wikipedia.org/wiki/Partial_derivative)
- Wikipedia: [Chain rule](https://en.wikipedia.org/wiki/Chain_rule)
- Wikipedia: [Gradient](https://en.wikipedia.org/wiki/Gradient)
- Wikipedia: [Gradient descent](https://en.wikipedia.org/wiki/Gradient_descent)
- Wikipedia: [Numerical differentiation](https://en.wikipedia.org/wiki/Numerical_differentiation)
- MIT OpenCourseWare, "18.01 Single Variable Calculus" and "18.02 Multivariable Calculus", search the course titles on ocw.mit.edu for the lectures on derivatives and the chain rule.

**Tutorials**

- Khan Academy, "Differential calculus" course, the units on the limit definition of the derivative and on the chain rule, with practice exercises.
- GeeksforGeeks, search "partial derivatives" and "chain rule in calculus" for worked examples.

**Watch**

- 3Blue1Brown (YouTube), "The paradox of the derivative | Essence of calculus, chapter 2". It explains the limit and the derivative with pictures of shrinking intervals.
- 3Blue1Brown (YouTube), "The chain rule | Essence of calculus, chapter 4". It explains why the derivatives multiply.
- 3Blue1Brown (YouTube), "Gradient descent, how neural networks learn | Deep learning, chapter 2". It shows the gradient as a way to lower a loss.
- MIT OpenCourseWare, "18.01 Single Variable Calculus" lectures by Prof. David Jerison on derivatives, linked from the course page.

**Practice**

- NumPy documentation, "numpy.gradient" on numpy.org, to compare the library's numerical derivative with yours.

---

## Tomorrow

Tomorrow you turn the single gradient step into an optimization loop. You will study objective functions, minima and maxima, the difference between local and global minima, and why a convex function is easy to optimize. Then you will implement gradient descent by hand, study how the learning rate decides whether training converges, and meet stochastic gradient descent, which estimates the gradient from a small batch of data. The gradient you computed today is the input to every step.

**Next:** [Day 12: Calculus II: Optimization](Day-12-Calculus-II-Optimization.md)

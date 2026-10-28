# Day 12: Calculus II: Optimization

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** intermediate. You have seen derivatives and gradients on Day 11. This day turns them into a method for finding the best parameters of a model. It is for anyone who wants to understand what "training" does.

**By the end of this day, you will have:**

- A hand-written gradient descent function that you have run on a one-variable problem and a linear regression problem
- A table showing how the learning rate changes convergence, slow progress, oscillation and divergence
- A short explanation of why a non-convex loss can trap gradient descent in a local minimum
- A small optimization lab repository with tests and an experiment script

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`)
- pytest 7 or newer for the tests (`pip install pytest`)
- The free Desmos graphing calculator at desmos.com, which runs in a browser without signing in
- Git and a free GitHub account for the project

**Prerequisites:**

- Day 11: derivatives, partial derivatives, the chain rule and gradients
- Day 7: functions, equations and graphs
- Day 8 and Day 9: NumPy-style vectors and matrix products
- Basic Python: functions, loops and lists

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Background and history](#background-and-history)
- [Glossary](#glossary)
- [Objective functions](#objective-functions)
- [Minima and maxima](#minima-and-maxima)
- [Local and global minima](#local-and-global-minima)
- [Convexity intuition](#convexity-intuition)
- [Gradient descent](#gradient-descent)
- [Learning rate](#learning-rate)
- [Stochastic optimization](#stochastic-optimization)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: an optimization lab](#project-an-optimization-lab)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Every machine learning model has adjustable numbers, called parameters. A linear model has a slope and an intercept. A neural network can have billions. Training means choosing those numbers so the model makes small mistakes, where "mistakes" is measured by one number called the loss. Finding the parameters that make the loss small is an optimization problem, and calculus gives the tool: the derivative tells you which direction makes a function go down.

On the job, a training run that goes wrong usually shows one of three symptoms. A loss that will not drop, a loss that oscillates, or a loss that suddenly becomes `nan` all come from the optimizer. If you understand the learning rate and the shape of the loss surface, you can read a training curve and guess the fix in a minute.

The big idea is simple. Start somewhere. Compute the slope of the loss at your current point. Take a small step downhill. Repeat until the slope is close to zero. That loop is gradient descent, and nearly every model in this series is trained with some version of it.

---

## Background and history

| Period | What happened | Why it matters today |
|---|---|---|
| 1847 | Augustin-Louis Cauchy describes a method that moves against the gradient to solve equations. | The core idea is almost 180 years old. |
| 1951 | Herbert Robbins and Sutton Monro publish "A Stochastic Approximation Method", which updates a solution from noisy measurements. | The root of stochastic gradient descent. |
| 1986 | Rumelhart, Hinton and Williams popularize backpropagation for training multi-layer neural networks. | Gradient descent becomes the standard way to train neural networks. |

**Takeaway:** the method is old. What changed is the scale: millions or billions of parameters, and datasets too large to process at once.

---

## Glossary

- **Objective function**: the function you want to make as small (or large) as possible. In machine learning it is usually the loss.
- **Loss**: a number that measures how wrong a model is on the training data. Lower is better.
- **Critical point**: a point where the derivative (or gradient) is zero. Minima, maxima and saddle points are all critical points.
- **Saddle point**: a critical point that is a minimum in some directions and a maximum in others.
- **Gradient**: the vector of partial derivatives. It points in the direction in which the function increases fastest.
- **Gradient descent**: the iterative method that steps in the direction opposite to the gradient.
- **Learning rate**: the size of each step in gradient descent.
- **Convex function**: a bowl-shaped function where any local minimum is also the global minimum.
- **Local minimum**: a point lower than its neighbors but not necessarily the lowest overall.
- **Global minimum**: the lowest value the function takes anywhere.
- **Epoch**: one full pass through the training data.
- **Mini-batch**: a small random subset of the training data used to estimate the gradient for one step.
- **Stochastic gradient descent (SGD)**: gradient descent where each step uses a random sample or mini-batch.

---

## Objective functions

Think of tuning a shower. Too cold is bad, too hot is bad, and comfort is somewhere in between. You score each setting by how far it is from comfortable, then try to make the score small. The score is the objective function, and the temperature is the variable you can change.

For a model, the objective is built from data. Suppose three points, (1, 2), (2, 4) and (3, 7), and a model `y = w * x` with one parameter `w`. A natural objective is the sum of squared errors:

```python
import numpy as np

x = np.array([1.0, 2.0, 3.0])
y = np.array([2.0, 4.0, 7.0])

def loss(w):
    return np.sum((w * x - y) ** 2)

print(loss(1.0))       # 21.0
print(loss(2.0))       # 1.0
print(loss(31 / 14))   # about 0.357, the best w (31 / 14 is about 2.214)
```

Training is the search for the `w` with the smallest loss. The optimizer does not care how the model was written. It only needs a number for each set of parameters.

The common mistake is to optimize a number that does not match the goal. If you minimize training loss but the real goal is accuracy on new customers, the optimizer will happily overfit. For any loss, check whether a lower value here means a better product.

In the market, every training job, from a bank's fraud model to a streaming service's recommender, is defined by an objective someone chose on purpose.

---

## Minima and maxima

A minimum is the bottom of a valley on a graph, and a maximum is the top of a hill. Both occur where the slope is zero. For a function of one variable, `f'(x) = 0` at an extremum. This is necessary but not sufficient, because a flat spot such as `x^3` at `x = 0` is neither.

The second derivative separates them. If `f''(x) > 0`, the curve bends upward and the point is a minimum. For `f(x) = (x - 3)^2`, the first derivative `2(x - 3)` is zero at `x = 3`, and the second derivative `2` is positive, so `x = 3` is a minimum.

For many parameters, a point is critical when every partial derivative is zero, which means the gradient vector is zero. Solving that exactly is usually impossible for complicated models, which is why we walk toward it instead.

A frequent error is to solve `gradient = 0` and stop without checking whether the point is a minimum or a saddle point. In deep networks, saddle points are common, and an optimizer can stall near one for a long time. If the loss has stopped falling while the gradient is still large, you are on a slope, not a valley floor.

In the market, anyone who fits a logistic regression is solving a maximum likelihood problem, which is a maximization turned into a minimization by flipping the sign of the objective.

---

## Local and global minima

A local minimum is a valley lower than everything next to it. The global minimum is the lowest valley anywhere. A hiker in fog who always walks downhill reaches some valley, not necessarily the deepest one.

Consider `g(x) = x^4 - 3x^2 + x`. It has a deep valley near `x = -1.3` and a shallow one near `x = 1.13`. Here is gradient descent from two starting points:

```python
def g(x):
    return x**4 - 3 * x**2 + x

def dg(x):
    return 4 * x**3 - 6 * x + 1

def descend(start, lr=0.01, steps=200):
    x = start
    for _ in range(steps):
        x -= lr * dg(x)
    return x

for start in [-2.0, 2.0]:
    x = descend(start)
    print(f"start {start:+.1f} -> x = {x:+.4f}, g(x) = {g(x):+.4f}")

# start -2.0 -> x = -1.3008, g(x) = -3.5139
# start +2.0 -> x = +1.1309, g(x) = -1.0702
```

Both runs stop where the gradient is zero. Only the starting point decided which valley they reached. The common mistake is to assume a flattened loss curve has found the best answer. Rerun from several starts and compare the final losses. If they differ, the problem has more than one valley.

On the job, this is why weight initialization in neural networks is an engineering decision. Teams run several seeds and report the spread, not one lucky number.

---

## Convexity intuition

A function is convex if the line segment between any two points on its graph lies on or above the graph. It looks like a bowl, and a bowl has no false valleys: every local minimum is the global minimum.

A convexity check for one variable is the second derivative. It must never be negative. For `(x - 3)^2` it is always 2. For `g(x)` it is `12x^2 - 6`, which is negative between about -0.71 and 0.71, so `g` is not convex.

Linear regression with mean squared error is convex, and so is logistic regression with log loss. Gradient descent therefore finds the single answer reliably. Deep networks have no such guarantee.

The honest limitation is that convexity is a property of the whole function, and a small change can break it. Adding a non-convex penalty or changing a link function can turn a bowl into a landscape with traps. Re-check the shape when you move from a textbook model to a custom one.

In the market, convex models are one reason classical models in banks and insurers are easy to audit: the optimizer's answer is the answer.

---

## Gradient descent

Gradient descent is a hiker's rule: feel which way is downhill and step that way. The gradient points uphill, so the update subtracts it:

```
x_new = x_old - learning_rate * f'(x_old)
```

For many variables, `w_new = w_old - learning_rate * gradient(w_old)`.

Each step is proportional to the slope. Far from the minimum the slope is large and the steps are big. Near the bottom the steps shrink on their own, so no extra logic is needed.

```python
def f(x):
    return (x - 3) ** 2

def grad_f(x):
    return 2 * (x - 3)

def gradient_descent_1d(start, lr, steps):
    x = start
    history = [x]
    for _ in range(steps):
        x = x - lr * grad_f(x)
        history.append(x)
    return x, history

x_final, history = gradient_descent_1d(start=0.0, lr=0.1, steps=20)
print(round(x_final, 4))   # 2.9654, very close to the minimum at x = 3
```

The same rule works for several parameters. Here is linear regression on 100 points generated from `y = 2 + 3x + noise`:

```python
import numpy as np

rng = np.random.default_rng(0)
X = np.c_[np.ones(100), rng.uniform(0, 10, 100)]   # column of ones for the intercept
y = X @ np.array([2.0, 3.0]) + rng.normal(0, 1, 100)

def mse_gradient(X, y, w):
    return (2 / len(y)) * X.T @ (X @ w - y)

w = np.zeros(2)
for _ in range(2000):
    w = w - 0.01 * mse_gradient(X, y, w)

print(w.round(3))                                       # about [1.992 2.987]
print(np.linalg.solve(X.T @ X, X.T @ y).round(3))       # closed form: [1.993 2.987]
```

A frequent error is to stop after a fixed number of steps chosen at random. Stop when the loss has stopped changing meaningfully or the gradient is tiny, and check the loss history: if it is still falling steadily, you have not converged.

In the market, every network trained in PyTorch or TensorFlow runs this update at its core. The framework adds automatic differentiation, but the loop is the same.

---

## Learning rate

The learning rate is the setting you choose directly, and it has the largest effect on whether training works. Too small and training is slow. Too large and the steps overshoot until the loss grows without bound.

For `f(x) = (x - 3)^2`, each step multiplies the distance from the minimum by `(1 - 2 * learning_rate)`:

```python
for lr in [0.01, 0.1, 0.5, 1.1]:
    x_final, _ = gradient_descent_1d(start=0.0, lr=lr, steps=20)
    print(f"lr={lr:<5} x after 20 steps = {x_final:.4f}")

# lr=0.01  x after 20 steps = 0.9972     (too slow)
# lr=0.1   x after 20 steps = 2.9654     (good)
# lr=0.5   x after 20 steps = 3.0000     (exact in one step)
# lr=1.1   x after 20 steps = -112.0128  (diverges)
```

With `lr = 0.5` the factor is zero, so one step lands on the minimum. With `lr = 1.1` the factor is `-1.2`: each step flips sides and grows the distance by 20 percent. For a quadratic with curvature 2, the threshold for divergence is a learning rate of 1. Real problems have many directions with different curvatures, so there is no universal safe value. In practice, try 0.1, 0.01 and 0.001 and watch the loss.

The common mistake is to tune the learning rate on unscaled features. If one column ranges from 0 to 1 and another from 0 to 1,000,000, the curvature differs between directions and no single rate fits both. Scaling the features often fixes what looks like a learning rate problem. The honest limitation of schedules, which start high and shrink the rate, is that each adds settings that can be tuned badly.

In the market, a learning rate sweep is one of the first experiments a machine learning engineer runs on a new model.

---

## Stochastic optimization

The full gradient reads every training example before one step. For ten million rows, that is expensive. Stochastic gradient descent estimates the gradient from a random mini-batch. It is like a survey: a random sample gives a noisy estimate that is cheap and points roughly the right way.

```python
def sgd(X, y, lr=0.01, epochs=200, batch_size=10, seed=1):
    rng = np.random.default_rng(seed)
    w = np.zeros(X.shape[1])
    n = len(y)
    for _ in range(epochs):
        order = rng.permutation(n)
        for start in range(0, n, batch_size):
            batch = order[start:start + batch_size]
            w = w - lr * mse_gradient(X[batch], y[batch], w)
    return w

print(sgd(X, y).round(3))   # about [1.996 3.003]
```

With 100 points and batch size 10, each epoch makes ten updates. Batch size 1 is pure SGD, and a batch equal to the dataset is full-batch gradient descent. Most practical work uses something in between.

The common mistake is to judge a stochastic run by its last step, which can be slightly off because of noise. Average the loss over a window of steps before you read the trend.

The honest limitation is that with a fixed learning rate, SGD hovers around the minimum and does not settle on it. A decreasing schedule or averaging the iterates fixes this, at the cost of more tuning.

In the market, nearly every large-scale model is trained with mini-batch SGD or a variant such as Adam. Mini-batches also suit GPUs, which process a batch with one large matrix multiplication.

---

## Hands-on exercises (about 60 minutes)

Create a folder called `day12-practice` and write your code in `practice.py`.

1. **Plot the bowl (10 minutes).** In Desmos, plot `y = (x - 3)^2` and `y = 2(x - 3)`. Confirm the minimum is at `x = 3` and that the derivative crosses zero there. Write both observations in a comment.

2. **Implement gradient descent (10 minutes).** Copy `gradient_descent_1d`. Run it from `start = 0.0` with `lr = 0.1` for 20 steps and confirm the result is about `2.9654`.

3. **Find the divergence threshold (10 minutes).** Loop over learning rates from 0.1 to 1.2 in steps of 0.05. Find the smallest rate for which the run is more than 0.01 away from 3 after 20 steps. Compare it with the prediction of 1.0.

4. **Try a non-convex function (10 minutes).** Run gradient descent on `g(x)` from 10 starts between -2 and 2. Count how many runs end in each valley and explain the result in one sentence.

5. **Linear regression by hand (10 minutes).** Rebuild the dataset from this lesson. Run 2000 steps with `lr = 0.01` and compare with `np.linalg.solve`. Confirm they agree to two decimal places.

6. **Compare batch sizes (10 minutes).** Run `sgd` with batch sizes 1, 10 and 100 for 50 epochs. Print the final weights and the number of updates for each. Write one sentence on which batch size was most stable.

**Evidence to commit:** `practice.py` with all six steps, and `notes.md` with the threshold from step 3, the valley counts from step 4 and the batch size comparison from step 6.

---

## Project: an optimization lab

This project is practical. It turns the hand-written gradient descent into a small tested library and an experiment script that you could reuse for any loss. Name the repository `100-days-ai-engineering-day-12-optimization`.

**Repo layout**

```
100-days-ai-engineering-day-12-optimization/
├── README.md
├── requirements.txt
├── optimizers.py
├── experiments.py
└── tests/
    └── test_optimizers.py
```

**Starter code**

`requirements.txt`:

```
numpy>=1.24
pytest>=7.0
```

`optimizers.py`:

```python
"""Gradient descent written by hand."""


def gradient_descent(grad, start, learning_rate=0.1, steps=100):
    """Repeatedly move against the gradient. Return (final_x, history)."""
    x = start
    history = [x]
    for _ in range(steps):
        x = x - learning_rate * grad(x)
        history.append(x)
    return x, history


def objective(x):
    """A convex bowl with its minimum at x = 3."""
    return (x - 3) ** 2


def objective_grad(x):
    """Derivative of the objective: 2 * (x - 3)."""
    return 2 * (x - 3)
```

`experiments.py`:

```python
from optimizers import gradient_descent, objective, objective_grad

print("learning_rate  final_x   objective")
for lr in [0.01, 0.1, 0.5, 0.9, 1.1]:
    x, _ = gradient_descent(objective_grad, start=0.0, learning_rate=lr, steps=20)
    print(f"{lr:<14} {x:<9.4f} {objective(x):.6f}")
```

Running `python experiments.py` prints:

```
learning_rate  final_x   objective
0.01           0.9972    4.011304
0.1            2.9654    0.001196
0.5            3.0000    0.000000
0.9            2.9654    0.001196
1.1            -112.0128 13227.944112
```

The rates `0.9` and `0.1` land equally far from the minimum after 20 steps. For `0.9` the factor is `-0.8`, so the iterate oscillates across 3 while shrinking at the same rate. Two paths can look equally good at a fixed step count, so always inspect the path.

`tests/test_optimizers.py`:

```python
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))
from optimizers import gradient_descent, objective_grad


def test_converges_to_minimum():
    x, _ = gradient_descent(objective_grad, start=0.0, learning_rate=0.1, steps=200)
    assert abs(x - 3.0) < 1e-6


def test_history_length():
    _, history = gradient_descent(objective_grad, start=0.0, learning_rate=0.1, steps=10)
    assert len(history) == 11
```

**Step-by-step tasks**

1. Create the repository, the folder tree and a virtual environment. Install the requirements, run `python experiments.py` and confirm the table matches the one above.
2. Run `pytest` and make sure both tests pass.
3. Add a tolerance argument `tol=1e-8` to `gradient_descent`. Stop early when `abs(grad(x)) < tol`, and return the number of steps taken as a third value. Add a test for the early stop.
4. Add `gradient_descent_nd(grad, start, learning_rate, steps)` that works on NumPy arrays. Use it on the linear regression loss from the lesson, and add a test that compares its answer with `np.linalg.solve`.
5. Plot the loss history for three learning rates, or save the values to a CSV file if matplotlib is not installed.
6. Write `README.md` with the checklist below, a table of the learning rates you tried and their final results, and the pytest output.

**README checklist**

- [ ] The project runs with `python experiments.py` on a fresh environment
- [ ] `pytest` passes with at least four tests
- [ ] The README lists each learning rate tried and its result
- [ ] The README gives one example where gradient descent stops in a local minimum
- [ ] The README states the stopping rule and why you chose it

**Done when**

- The repository contains the code, tests and README, and is committed to GitHub
- `gradient_descent_nd` matches the closed-form answer to within `1e-3` on the test dataset
- The README explains the oscillation at `0.9` from the table
- A reader can clone the repo and reproduce the results in three commands

**Evidence to commit:** the code, the pytest output, the learning rate table and the loss plot or CSV.

---

## Check your understanding

1. A colleague says a run is finished because the loss has been flat for 500 steps. What three checks would you run before agreeing?
2. The loss falls for ten steps, then rises steadily until it reaches `nan`. Which setting do you change first, and why must it go down rather than up?
3. Two runs from different seeds end with different training losses. Can you conclude the lower one is better? What would you check?
4. For `f(x) = x^2 + 4x + 7`, find the minimum analytically and explain why gradient descent finds it from any start with a reasonable learning rate.
5. You switch to mini-batches of size 32 and the loss curve becomes noisy. Is training broken? What would you look at?
6. A teammate adds a non-convex penalty to a linear model to encourage sparsity. What changes, and what would you check after training?

**Answers**

1. Check the gradient norm, not just the loss, since a flat loss at a saddle point still has a large gradient in some direction. Check validation loss, since a flat training loss can hide overfitting. Rerun from another seed, since a plateau may be escaped from a different start.
2. The learning rate, reduced by about a factor of 10. A large step overshoots the valley, the loss grows, and it can overflow. A larger rate would make each overshoot worse. Also check whether the features are unscaled, since large inputs produce large gradients.
3. No. The runs may have landed in different local minima, or the difference may come from luck with the start. Compare the spread over several seeds and check the validation metric, not only training loss.
4. The derivative is `2x + 4`, zero at `x = -2`. The second derivative is 2, which is positive, so the point is a minimum. The function is convex, and with a learning rate below 1 (since the curvature is 2) gradient descent converges from any start.
5. Not necessarily. Noise is expected with mini-batches. Look at the loss averaged over many steps and at the validation loss. If the average falls steadily, training works. If it is flat or rising, lower the learning rate.
6. The problem is no longer convex, so the solution can depend on the start and gradient descent can stop in a local minimum. Run several seeds, compare the solutions, and check that the sparsity actually appeared in the weights.

---

## Sources

**Read**

- Wikipedia: [Mathematical optimization](https://en.wikipedia.org/wiki/Mathematical_optimization)
- Wikipedia: [Gradient descent](https://en.wikipedia.org/wiki/Gradient_descent)
- Wikipedia: [Maxima and minima](https://en.wikipedia.org/wiki/Maxima_and_minima)
- Wikipedia: [Convex function](https://en.wikipedia.org/wiki/Convex_function)
- Wikipedia: [Stochastic gradient descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent)
- Stanford University, CS229 Machine Learning lecture notes by Andrew Ng. Search "CS229 lecture notes" on the course website.
- Stephen Boyd and Lieven Vandenberghe, *Convex Optimization*, Cambridge University Press. The first chapters cover convex functions, and a free PDF is available from the authors' Stanford site.

**Tutorials**

- GeeksforGeeks: search "Gradient Descent in Machine Learning" on geeksforgeeks.org
- Google Developers, Machine Learning Crash Course, the section "Gradient Descent". Search the title on developers.google.com

**Watch**

- 3Blue1Brown, "Gradient descent, how neural networks learn" (Deep Learning series, chapter 2)
- 3Blue1Brown, "The paradox of the derivative" (Essence of Calculus series)
- MIT OpenCourseWare, 18.065 Matrix Methods in Data Analysis, the lecture on least squares. Search the course title on ocw.mit.edu.

**Practice**

- Desmos graphing calculator at desmos.com. Plot the functions in this lesson and drag the parameters to see how the minimum moves.

---

## Tomorrow

Tomorrow is Day 13, Probability. Today you learned to move downhill on a loss surface. Probability gives you the language of uncertainty: what the data might be, how likely a result is, and how to update a belief when new evidence arrives. Those ideas underpin the loss functions in Day 15 and the statistical reasoning in Day 14.

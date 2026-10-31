# Day 44: Optimization

*100 Days of AI Engineering, Module 5: Deep Learning Foundations*

**Time:** about 3.5 hours
**Level:** intermediate. You can compute gradients (Day 43) and you know what a learning rate does. Today you learn how to choose the step, not only the direction.

**By the end of this day, you will have:**

- Working NumPy implementations of stochastic gradient descent with and without momentum, and of Adam, written from the update equations
- A demonstration, on a simple two-dimensional problem, that a step size can make training diverge and that Adam and momentum behave differently on a badly scaled problem
- A comparison of batch, stochastic and mini-batch training on a small network, with a learning-rate schedule and gradient clipping
- A short report that recommends a default optimizer and schedule for a given problem, backed by your own numbers

**What you need:**

- A computer with Python 3.11 or 3.12
- NumPy 2.1.3 (`pip install numpy==2.1.3`), free and open source under the BSD license
- pytest 8.3.3 (`pip install pytest==8.3.3`), free and open source under the MIT license
- Matplotlib 3.9.2 (`pip install matplotlib==3.9.2`), for the loss curves
- The `TwoLayerNet` class from your Day 43 repository, with its `forward` and `backward` methods and a `params` dictionary holding W1, b1, W2 and b2
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 11 and Day 12: derivatives, gradients and the idea of a minimum
- Day 13: the idea of a random sample and its mean
- Day 43: the backward pass and the update step `params -= lr * grads`

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Batch gradient descent](#batch-gradient-descent)
- [SGD](#sgd)
- [Mini-batches](#mini-batches)
- [Momentum](#momentum)
- [Adam](#adam)
- [Learning-rate schedules](#learning-rate-schedules)
- [Optimization instability](#optimization-instability)
- [Hands-on exercises](#hands-on-exercises)
- [Project: optimizers from the equations](#project-optimizers-from-the-equations)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Day 43 told you which way to move each weight. Today you decide how far to move and how to combine the gradients from many examples. Those choices decide whether a network trains in minutes or never converges, and whether the final model generalizes. Two engineers with the same architecture and data can get very different results because one of them picked a learning rate of 0.1 and the other picked 0.001 with a warm-up.

The core tension is that the gradient of the full dataset is accurate but expensive, and the gradient of one example is cheap but noisy. Modern training sits between those extremes, using mini-batches and an adaptive method such as Adam. The rest of the methods, momentum and schedules, correct the problems that appear in the middle of training, when the loss surface is narrow in some directions and flat in others.

On the job, the optimizer and the schedule are some of the first hyperparameters a team tunes. Many published results fix them to a known recipe, and a team that wants to reproduce a result must use the same recipe, or the comparison is meaningless. You will be asked why you chose a learning rate, and the answer should cite the training curve.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1847 | Augustin-Louis Cauchy describes the method of steepest descent for solving equations. | The root of gradient descent, which is still the basic step. |
| 1951 | Herbert Robbins and Sutton Monro publish the stochastic approximation method. | The theory behind updating with noisy gradients from random samples. |
| 1964 | Boris Polyak proposes the heavy-ball method, which adds a fraction of the previous step. | The idea behind momentum. |
| 1983 | Yurii Nesterov proposes an accelerated gradient method with better worst-case rates. | The variant that is often used in convex optimization and is related to momentum in deep learning. |
| 2014 | Diederik Kingma and Jimmy Ba publish Adam, which adapts the step size for each parameter. | The default optimizer for most deep learning work today. |

**Takeaway:** the core ideas are more than a century old. Adam is the newest of the standard methods and it is still treated with care, because its defaults are not always the best choice.

---

## Glossary

- **Objective**: the function that training minimizes, usually the average loss over the training data.
- **Batch**: the set of examples used to compute one gradient estimate.
- **Epoch**: one pass over the entire training set, in whatever batches you choose.
- **Iteration or step**: one parameter update, computed from one batch.
- **Gradient estimate**: the gradient computed from a batch, which approximates the gradient of the full objective.
- **Momentum**: a running average of past gradients that carries the update through noise and narrow valleys.
- **Adaptive learning rate**: a step size that is scaled for each parameter, based on the history of its gradients.
- **Second moment**: the running average of squared gradients, used by Adam to scale the step.
- **Schedule**: a rule that changes the learning rate over training, such as a decay or a warm-up.
- **Warm-up**: a schedule that starts with a small learning rate and raises it over the first steps.
- **Saddle point**: a point where the gradient is zero but the objective goes up in some directions and down in others.
- **Gradient clipping**: rescaling the gradient when its norm is larger than a chosen threshold.
- **Condition number**: the ratio of the largest to the smallest curvature of the objective, which measures how badly scaled the problem is.

---

## Batch gradient descent

Imagine finding the lowest point in a valley by surveying the whole valley before each step. Each survey is accurate, so every step goes in the right direction, but surveying takes a long time. Batch gradient descent is that method. Each update uses the gradient of the loss over all training examples.

From first principles, the update is theta = theta - lr times the average gradient over all N examples. The expected direction is correct, and with a small enough learning rate the loss decreases at every step on a smooth convex problem. The cost of one step is proportional to N, which becomes prohibitive for large datasets. A dataset of ten million images would need ten million forward and backward passes for each single update.

```python
import numpy as np

def batch_gd(theta, grad_fn, lr, steps):
    for _ in range(steps):
        theta = theta - lr * grad_fn(theta)   # grad_fn uses ALL examples
    return theta
```

The limitation is that full-batch steps are slow and can fall into poor regions of a nonconvex loss, where the gradient is small but the solution is bad. A more accurate gradient per step does not mean batch gradient descent finds a better model. On a deep network, the noise of smaller batches often helps generalization.

In the real market, full-batch methods still appear in classical models such as logistic regression on small tabular data, and in second-order methods used in scientific computing. For deep networks, they are used rarely.

---

## SGD

Think of a hiker who checks the slope under their own feet instead of surveying the valley. The direction is less reliable, but the hiker takes many quick steps. Stochastic gradient descent, SGD, is that method. Each update uses the gradient of one randomly chosen example.

From first principles, if the example index is chosen at random, the expected value of its gradient equals the full gradient. That is what makes SGD an unbiased estimate of batch gradient descent, although each individual step is noisy. The noise has a benefit: it helps escape saddle points and flat regions, and it acts as a regularizer that tends to find solutions with small generalization gap. The noise also makes the loss curve jagged, and the learning rate must be smaller than for batch descent to keep the noise under control.

```python
import numpy as np

def sgd(theta, grad_fn_single, X, y, lr, epochs, rng):
    n = len(X)
    for _ in range(epochs):
        for i in rng.permutation(n):       # shuffle once per epoch
            theta = theta - lr * grad_fn_single(theta, X[i], y[i])
    return theta
```

The limitation is that single-example steps waste the hardware. A GPU is built for matrix operations on many examples at once, and a batch of one leaves most of it idle. The noise can also make the final loss oscillate, so the last learned weights may be worse than the average of recent ones.

Do not forget to shuffle the data between epochs. If the examples are always in the same order, the updates follow the same path each time, and the model can learn the order of the data. In industry, pure SGD with batch size one is almost never used, but the same idea with mini-batches is the default.

---

## Mini-batches

Consider taking a sample of ten people from a crowd to estimate the average height, instead of measuring everyone or just one person. The sample is noisy but cheap, and its average is a reasonable estimate. A mini-batch is that sample for the gradient.

From first principles, if you take a batch of B examples, the average of their gradients has the same expected value as the full gradient, and its variance shrinks as 1 over B. Larger batches give smoother estimates, but each step costs more. The best batch size balances the noise against the time per step, and it also interacts with the learning rate. A common rule of thumb is to scale the learning rate with the batch size, but the rule is approximate and should be checked.

```python
import numpy as np

def minibatches(X, y, batch_size, rng):
    idx = rng.permutation(len(X))
    for start in range(0, len(X), batch_size):
        j = idx[start:start + batch_size]
        yield X[j], y[j]
```

The limitation is that the batch size is a hyperparameter with a real effect on generalization. Very large batches can reach a sharper minimum that generalizes worse, and very small batches can be too noisy to converge. Large-batch training often needs a warm-up and a careful learning rate to match the small-batch result.

Do not treat the last incomplete batch as if it were full. If the loss is an average, a small final batch gives a noisier estimate with the same weight, so some implementations drop it. Either choice is valid, but make it on purpose. In industry, mini-batches are the default unit of work for GPU training, and batch sizes of several hundred to several thousand are common for large image and language models.

---

## Momentum

Picture a ball rolling down a bumpy hill. The ball does not stop at every small bump, and it builds speed in a consistent direction. Momentum gives the optimizer the same behavior: each update adds a fraction of the previous update, so consistent gradient directions accumulate and noisy ones cancel.

From first principles, classical momentum keeps a velocity v with v = mu v + g, where g is the gradient and mu is a number such as 0.9. The parameters move by theta = theta - lr v. If the gradient points the same way for many steps, the velocity approaches g divided by one minus mu, which is ten times g when mu is 0.9. In a narrow valley, the gradients across the valley alternate in sign and cancel, while the gradients along the valley agree and add up. That is why momentum speeds up training on badly scaled problems.

```python
import numpy as np

def momentum_step(theta, v, grad, lr, mu=0.9):
    v = mu * v + grad
    theta = theta - lr * v
    return theta, v
```

The limitation is that momentum can overshoot. Too large a momentum value or too large a learning rate can make the ball roll past the minimum repeatedly, and the loss oscillates. Do not treat mu as a constant that is always good. The value 0.9 is a common default, and it is a starting point, not a law.

In practice, SGD with momentum is still a strong baseline for image classification, where it often matches or beats adaptive methods when the schedule is tuned well. Many published vision results use it.

---

## Adam

Imagine adjusting the volume knob for each instrument in a band separately. A loud instrument needs a smaller turn, and a quiet one needs a bigger turn, so every part of the mix moves at a similar pace. Adam does this for every parameter. It keeps a running average of the gradient and a running average of its square, and it divides the first by the square root of the second.

From first principles, Adam maintains m = beta1 m + (1 - beta1) g and v = beta2 v + (1 - beta2) g^2. Because both averages start at zero, they are biased toward zero early in training, so Adam corrects them: m_hat = m / (1 - beta1^t) and v_hat = v / (1 - beta2^t). The update is theta = theta - lr m_hat / (sqrt(v_hat) + eps). The defaults in the original paper are lr = 0.001, beta1 = 0.9, beta2 = 0.999 and eps = 1e-8. Parameters with large gradients take smaller steps, and parameters with small gradients take larger steps, which helps when the scales of the parameters differ.

```python
import numpy as np

def adam_step(theta, m, v, grad, t, lr=1e-3, beta1=0.9, beta2=0.999, eps=1e-8):
    m = beta1 * m + (1 - beta1) * grad
    v = beta2 * v + (1 - beta2) * grad * grad
    m_hat = m / (1 - beta1 ** t)
    v_hat = v / (1 - beta2 ** t)
    theta = theta - lr * m_hat / (np.sqrt(v_hat) + eps)
    return theta, m, v
```

The limitation is that Adam's adaptive scaling does not always generalize as well as SGD with momentum. Some studies find that Adam converges quickly to solutions that generalize slightly worse on image tasks, although the gap depends on the task and the tuning. Adam also has a weight-decay interaction that is subtle: adding an L2 penalty to the gradient is not the same as decoupled weight decay, and AdamW was proposed to separate them.

Do not treat Adam's default learning rate as correct for every problem. A learning rate of 0.001 is a good starting point for many networks, but a transformer may need a warm-up and a smaller value, and a small network may need a larger one. In the real market, Adam or AdamW is the default optimizer for language models and most new deep learning projects, and teams tune the learning rate and the schedule rather than the other constants.

---

## Learning-rate schedules

Think of driving toward a destination. You accelerate to a good speed, cruise, and slow down as you approach, so you do not overshoot. A learning-rate schedule plans that trajectory for the optimizer.

From first principles, the simplest schedule is a step decay, which multiplies the learning rate by a factor such as 0.1 at chosen epochs. A cosine schedule decreases the rate smoothly from a maximum to a minimum over a fixed number of steps, using lr_t = lr_min + 0.5 (lr_max - lr_min) (1 + cos(pi t / T)). A warm-up raises the rate linearly from near zero over the first few hundred steps, which helps when the early gradients are large or unreliable. Warm-up followed by cosine decay is a common pattern in transformer training.

```python
import numpy as np

def cosine_lr(step, total_steps, lr_max, lr_min=0.0):
    return lr_min + 0.5 * (lr_max - lr_min) * (1 + np.cos(np.pi * step / total_steps))

def warmup_then_cosine(step, warmup_steps, total_steps, lr_max):
    if step < warmup_steps:
        return lr_max * (step + 1) / warmup_steps
    return cosine_lr(step - warmup_steps, total_steps - warmup_steps, lr_max)
```

The limitation is that a schedule has its own hyperparameters, such as the decay factor, the step epochs or the warm-up length, and they interact with the optimizer. A cosine schedule that ends too early can stop learning before the model has converged, and a step schedule with a bad epoch can stall the loss for many epochs.

Do not pick a schedule first and then tune the learning rate to fit it. Start with a constant learning rate that is stable, then add a schedule that reduces it late in training. In industry, a schedule is often part of the recipe that a team inherits, and the recipe is checked on a validation set before it is changed.

---

## Optimization instability

A bridge with a resonance frequency will shake apart if you push it at that frequency. Optimization has a similar failure: a step that is too large in a steep direction makes the loss grow, the gradient grows and the next step is even larger. The parameters diverge.

From first principles, consider the quadratic f(x, y) = 0.5 (x^2 + 100 y^2). Its gradient is (x, 100 y), and its curvature is 1 in the x direction and 100 in the y direction. For gradient descent with step lr, the y coordinate is multiplied by (1 - 100 lr) at every step. If lr is greater than 0.02, the magnitude of that factor is greater than one, and y grows without bound. The x coordinate, with factor 1 - lr, shrinks slowly. The stable step size is set by the steepest direction, which slows learning in the flat direction. That is the condition number problem.

```python
import numpy as np

def run_sgd_quadratic(lr, steps=20):
    x, y = 1.0, 1.0
    for _ in range(steps):
        x, y = x - lr * x, y - lr * 100.0 * y
    return x, y

print(run_sgd_quadratic(0.005))   # both coordinates shrink
print(run_sgd_quadratic(0.03))    # y grows: the factor is -2 per step
```

Other failures are exploding gradients in deep recurrent networks and dying units in ReLU networks, which you met on Day 41. Gradient clipping limits the norm of the gradient before the update, so a single large gradient cannot throw the parameters far away. A clip of 1.0 is a common starting point in language models.

The limitation of clipping is that it changes the direction when it triggers, and a frequent clip hides a real problem, such as a bad learning rate or a data error. Do not turn on clipping and assume the instability is gone. Log how often clipping triggers. If it triggers on most steps, the learning rate is too high.

In the real market, the first sign of optimization instability is a loss that becomes NaN or that spikes on one batch. Teams react with a smaller learning rate, a warm-up, clipping and a check of the data for outliers, in that order.

---

## Hands-on exercises

1. **Stability by hand.** For f(x, y) = 0.5 (x^2 + 100 y^2), compute the factor by which the y coordinate changes each step under gradient descent with lr = 0.01, 0.02 and 0.03. Say which learning rates converge.
2. **Momentum effect.** Using the momentum update with mu = 0.9 and a constant gradient g = 1, compute the velocity after one, two and ten steps. Compare it with the limit g / (1 - mu).
3. **Adam first step.** For a single parameter with gradient 0.5 on the first step, compute m, v, m_hat and v_hat with the default constants. Show that the first update has size close to lr, regardless of the gradient size.
4. **Schedule plot.** Plot the warm-up then cosine schedule for 1,000 steps with a warm-up of 100 steps and a maximum of 0.001. Describe the shape in one sentence.
5. **Batch sweep.** Using the `TwoLayerNet` from Day 43 on the circle dataset, train with batch sizes 1, 16, 128 and the full dataset for the same number of epochs. Record the time per epoch and the final loss.

---

## Project: optimizers from the equations

**Goal:** implement SGD, SGD with momentum, and Adam from their update equations, check them on a badly scaled quadratic, then compare batch sizes, optimizers and schedules on a small network trained on a synthetic circle dataset. Write a recommendation backed by your numbers.

**Repo name:** `100-days-ai-day-44-optimization`

**Repo layout:**

```
100-days-ai-day-44-optimization/
├── README.md
├── requirements.txt
├── src/
│   └── optim_lab/
│       ├── __init__.py
│       ├── optimizers.py
│       ├── schedules.py
│       ├── data.py
│       └── network.py
├── tests/
│   └── test_optimizers.py
└── experiments/
    ├── quadratic_paths.py
    └── circle_compare.py
```

**Starter code.** Create `requirements.txt` with the pinned packages. Copy your `TwoLayerNet` from Day 43 into `src/optim_lab/network.py`. It must expose a `params` dictionary with the keys W1, b1, W2 and b2, and it must return gradients in a dictionary with the same keys. Then create `src/optim_lab/optimizers.py`:

```python
import numpy as np


class SGD:
    def __init__(self, lr=0.1, momentum=0.0):
        self.lr = lr
        self.momentum = momentum
        self.velocity = {}

    def step(self, params, grads):
        for k in params:
            g = grads[k]
            if self.momentum:
                v = self.velocity.get(k, np.zeros_like(g))
                v = self.momentum * v + g
                self.velocity[k] = v
                params[k] -= self.lr * v
            else:
                params[k] -= self.lr * g


class Adam:
    def __init__(self, lr=1e-3, beta1=0.9, beta2=0.999, eps=1e-8):
        self.lr = lr
        self.beta1 = beta1
        self.beta2 = beta2
        self.eps = eps
        self.t = 0
        self.m = {}
        self.v = {}

    def step(self, params, grads):
        self.t += 1
        for k in params:
            g = grads[k]
            m = self.m.get(k, np.zeros_like(g))
            v = self.v.get(k, np.zeros_like(g))
            m = self.beta1 * m + (1 - self.beta1) * g
            v = self.beta2 * v + (1 - self.beta2) * g * g
            self.m[k], self.v[k] = m, v
            m_hat = m / (1 - self.beta1 ** self.t)
            v_hat = v / (1 - self.beta2 ** self.t)
            params[k] -= self.lr * m_hat / (np.sqrt(v_hat) + self.eps)


def clip_grads(grads, max_norm):
    total = np.sqrt(sum(float(np.sum(g * g)) for g in grads.values()))
    if total > max_norm:
        scale = max_norm / (total + 1e-12)
        for k in grads:
            grads[k] = grads[k] * scale
    return total
```

`src/optim_lab/schedules.py` holds `cosine_lr` and `warmup_then_cosine` from the lesson. `src/optim_lab/data.py` should generate N points uniformly in the square from -1.5 to 1.5 on each axis, with label 1 when x1^2 + x2^2 < 1 and 0 otherwise, using `np.random.default_rng(seed)`.

**Step-by-step tasks:**

1. Create the repo, initialize Git, create a virtual environment and install the pinned packages.
2. Write `tests/test_optimizers.py`. On f(x) = 0.5 (x1^2 + 100 x2^2) with starting point (1, 1), assert three things: SGD with lr = 0.01 reduces f after each of the first 100 steps, SGD with momentum 0.9 and lr = 0.001 reaches f below 1e-3 within 1,000 steps, and Adam with lr = 0.01 reduces f below 1 percent of its starting value within 1,000 steps. If the Adam test fails with your numbers, record the actual value and adjust the tolerance with a comment that explains why.
3. Write `experiments/quadratic_paths.py` that runs SGD with lr = 0.03 and records the path, so that you can see the divergence described in the lesson. Save a plot of the paths to `results/quadratic_paths.png`.
4. Write `experiments/circle_compare.py`. For each combination of optimizer (SGD, momentum, Adam), batch size (16 and full batch) and schedule (constant and warm-up then cosine), train the network for 200 epochs on the circle dataset with 400 training points and 400 test points. Record the final training loss, the test accuracy and whether the loss ever became NaN.
5. Save a table of the results to `results/circle_table.md`, and save the loss curves for the three optimizers with batch size 16 and the cosine schedule to `results/circle_curves.png`.
6. Run one experiment with and without clipping at max_norm 1.0 and report how often clipping triggered.

**README checklist:**

- [ ] A one-line description of the project
- [ ] Setup steps with exact versions
- [ ] How to run the tests and both experiments
- [ ] The quadratic divergence plot, with one paragraph explaining it
- [ ] The results table, with the test accuracy for each configuration
- [ ] The clipping result and the number of steps where clipping triggered
- [ ] A recommendation for the default optimizer and schedule, in three sentences, that cites your numbers

**Done when:**

- The optimizer tests pass, and you have read the Adam tolerance comment and can defend it
- The quadratic plot shows the SGD path diverging at lr = 0.03 and converging at a smaller step
- The results table covers all twelve configurations and every number comes from your run
- Your recommendation cites at least one number from the table
- The repo has at least five commits

**Evidence to commit to GitHub:** the source, the tests, the README with the table and the two plots, and a `NOTES.md` that states one result that surprised you and one that confirmed a claim in the lesson. The notes show that the numbers informed the recommendation.

**Why this project is practical:** choosing an optimizer is a hyperparameter decision that you will make in every project. A small grid with a fixed budget and a written recommendation is the habit you need.

---

## Check your understanding

1. Why does the stable learning rate for gradient descent on a quadratic depend on the largest curvature, not the average curvature?
2. Explain why SGD with a single example per step still converges in expectation, even though each step is noisy.
3. Momentum with mu = 0.9 can increase the effective step by a factor of about ten. Under what conditions would that be helpful, and under what conditions would it cause oscillation?
4. Adam divides by the square root of the second moment. What happens to a parameter whose gradients are always very large, and what happens to one whose gradients are almost always zero?
5. A training run with a warm-up and cosine schedule reaches a good training loss, but the validation accuracy is poor. Name two causes that the optimizer could be responsible for and one check you would make before changing the model.
6. Your loss spikes to NaN at step 800 once every few runs. List the changes you would try, in the order you would try them, and say why.

**Answers:**

1. On a quadratic, the update in each direction multiplies the error by one minus the learning rate times the curvature in that direction. To keep every direction from growing, the learning rate must be small enough for the steepest direction, which is the largest curvature. The flat direction then learns slowly.
2. The expected value of a single-example gradient equals the full gradient, so the average of the updates follows the full gradient direction. The noise averages out over many steps, provided the learning rate decays or is small enough that the noise does not dominate.
3. Momentum helps when the gradients in the same direction agree over many steps, such as along a narrow valley, and when the learning rate is small enough for the accumulated step to be stable. It causes oscillation when the learning rate is too large or mu is too close to one, so the accumulated velocity carries the parameters past the minimum.
4. A parameter with always large gradients has a large second moment, so its step is divided by a large number and becomes small. A parameter with almost always zero gradients has a small second moment, so its step can be relatively large when a gradient finally appears. Adam therefore gives each parameter a step of comparable size.
5. The optimizer can cause poor validation accuracy if the learning rate is too large late in training, or if the schedule ends before convergence, or if the model is overfitting the training loss. A check is to plot the validation curve against the training curve and to compare the constant-rate and scheduled runs on the same seed before changing the architecture.
6. First, check the data for a batch with extreme values or a label error. Second, lower the learning rate. Third, add a warm-up. Fourth, enable gradient clipping and log how often it triggers. The order goes from the cheapest check to the most invasive change, and each change should be tested alone so you know which one fixed the problem.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, "Gradient descent", for the update rule and convergence conditions
- Wikipedia, "Stochastic gradient descent", for the mini-batch version and its history
- Wikipedia, "Adam (optimizer)", for the algorithm and the bias correction
- Wikipedia, "Condition number", for the measure behind the instability example
- Diederik P. Kingma and Jimmy Ba, "Adam: A Method for Stochastic Optimization", arXiv 1412.6980, for the original method and its defaults
- Ilya Loshchilov and Frank Hutter, "SGDR: Stochastic Gradient Descent with Warm Restarts", arXiv 1608.03983, for the cosine schedule

**Tutorials**

- PyTorch documentation, "torch.optim", at pytorch.org/docs/stable, for SGD, Adam and AdamW and their argument names
- Gabriel Goh, "Why Momentum Really Works", Distill, at distill.pub/2017/momentum, an interactive article with clear plots
- Andrej Karpathy, "A Recipe for Training Neural Networks", at karpathy.github.io, for a practical checklist of optimization issues

**Watch**

- 3Blue1Brown, "Gradient descent, how neural networks learn | Deep learning chapter 2", on YouTube
- Stanford CS231n lectures on training neural networks, which cover optimizers and schedules, on YouTube

**Practice**

- The Distill momentum article above, with its interactive plots, and the quadratic experiment in this project. Change the momentum value and watch the path change.

---

## Tomorrow

Day 45 asks what happens after the optimizer has done its work. A network that fits the training set perfectly can still fail on new data, and that gap is called overfitting. You will study weight decay, dropout, early stopping, data augmentation and batch normalization, and you will measure each one on the circle dataset you used today. The optimizer and schedule you chose here will be part of every comparison tomorrow.

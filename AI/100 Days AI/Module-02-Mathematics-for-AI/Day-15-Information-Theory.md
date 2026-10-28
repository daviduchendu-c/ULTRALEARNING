# Day 15: Information Theory

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** intermediate. You need Day 13 (probability) and the logarithm rules from Day 7. It is for readers who want to understand the loss functions behind classifiers and language models, and who want to stop treating them as black boxes.

**By the end of this day, you will have:**

- Working implementations of surprisal, entropy, cross-entropy and KL divergence, checked against the identity that links them
- A demonstration that cross-entropy gives a strong learning signal when a model is confidently wrong, while squared error on probabilities gives almost none
- A clear explanation of why a classifier's loss is cross-entropy and what perplexity means
- A small information theory lab repository with tests and an experiment script

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`)
- pytest 7 or newer (`pip install pytest`)
- Git and a free GitHub account for the project

**Prerequisites:**

- Day 7: logarithms, exponents and summation notation
- Day 8: the dot product
- Day 11: derivatives and the chain rule, used once for the gradient comparison
- Day 13: probability distributions and the idea that a model outputs a probability for each class
- Day 14: the mean, which is the average of per-example losses

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Information](#information)
- [Entropy](#entropy)
- [Cross-entropy](#cross-entropy)
- [KL divergence concept](#kl-divergence-concept)
- [Why cross-entropy appears in classification and language models](#why-cross-entropy-appears-in-classification-and-language-models)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an information theory lab](#project-an-information-theory-lab)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Information theory asks how much a message tells you. "The sun rose this morning" tells you almost nothing. "It snowed in July in Cairo" tells you a lot. The amount depends on how surprising the message is. Claude Shannon turned this into mathematics in 1948, and the result, measured in bits, underpins data compression, error-correcting codes and the training of almost every classifier in use today.

On the job, these ideas appear in concrete places. A classifier is trained by minimizing cross-entropy, so the loss in your training log is an information measure. A language model is evaluated by perplexity, which is the exponential of cross-entropy. Distillation uses KL divergence to measure how far a smaller model has drifted from a larger one. If you cannot read these numbers, you cannot tell whether a high loss means a bad model or bad labels.

The big idea is that a loss is a measure of surprise. A good model is rarely surprised by the true outcome. Cross-entropy is the average surprise, and training makes the true labels less surprising to the model.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1928 | Ralph Hartley defines information as the logarithm of the number of equally likely outcomes. | The first step toward a logarithmic measure of information. |
| 1948 | Claude Shannon publishes "A Mathematical Theory of Communication", defining entropy as the average information of a source. | The foundation of the field, and the origin of the bit. |
| 1951 | Solomon Kullback and Richard Leibler introduce the measure of divergence between two distributions. | The formal tool behind KL divergence. |

**Takeaway:** the mathematics dates from the 1940s and 1950s. What is new is that it now serves as the training objective for systems with billions of parameters.

---

## Glossary

- **Information (surprisal)**: the surprise in one outcome, equal to the negative log of its probability.
- **Bit**: the unit of information with base-2 logarithms. One fair coin flip carries exactly one bit.
- **Nat**: the unit with natural logarithms. Machine learning libraries usually use nats.
- **Entropy**: the average information of outcomes drawn from a distribution.
- **Cross-entropy**: the average surprisal when outcomes come from one distribution but are judged by another.
- **KL divergence**: the extra average surprisal from using the wrong distribution. It is zero only when the two distributions match.
- **Logits**: the raw, unnormalized scores a model outputs before softmax.
- **Softmax**: a function that turns logits into positive probabilities that add to 1.
- **One-hot vector**: a vector with a 1 at the true class and 0 elsewhere.
- **Perplexity**: the exponential of the cross-entropy in nats, which can be read as an effective number of equally likely choices.
- **Calibration**: the property that predicted probabilities match observed frequencies.

---

## Information

Weather in a desert city is almost always dry, so "no rain today" tells you nothing. In a rainforest, the same report tells you something. Information measures how much an outcome tells you, and that depends on its probability before you see it:

```
information(outcome) = -log(probability of outcome)
```

A certain outcome carries `-log(1) = 0`. A fair coin flip carries `-log2(0.5) = 1` bit. A roll of 6 on a fair die carries `-log2(1/6)`, about 2.58 bits. Rare events carry more.

```python
import numpy as np

def surprisal(p, base=2):
    return -np.log(p) / np.log(base)

print(surprisal(0.5))              # 1.0 bit, a fair coin
print(round(surprisal(1 / 6), 2))  # 2.58 bits, one face of a die
print(round(surprisal(0.01), 2))   # 6.64 bits, a rare event
```

Information adds for independent events, because the logarithm turns products of probabilities into sums. The common mistake is to think information belongs to the message. It belongs to the probability model. "Heads" carries 1 bit for a fair coin and almost nothing for a coin that always lands heads.

In the market, the same principle drives data compression. ZIP and gzip give short codes to common patterns and long codes to rare ones, because common patterns carry little information each.

---

## Entropy

Entropy is the average information of a distribution, weighted by how often each outcome occurs:

```
H(p) = sum over outcomes x of p(x) * (-log p(x))
```

Entropy is highest when all outcomes are equally likely and zero when one outcome is certain. A fair coin has 1 bit, and a coin with probability 0.9 for heads has about 0.47 bits. For `p = [0.5, 0.25, 0.125, 0.125]`, the surprisals are 1, 2, 3 and 3 bits, and their weighted average is `0.5*1 + 0.25*2 + 0.125*3 + 0.125*3 = 1.75` bits.

```python
import numpy as np

def entropy(p, base=2):
    p = np.asarray(p, dtype=float)
    p = p[p > 0]                  # 0 * log(0) is taken to be 0
    return float(-np.sum(p * np.log(p)) / np.log(base))

print(entropy([0.5, 0.25, 0.125, 0.125]))   # 1.75
print(entropy([0.25, 0.25, 0.25, 0.25]))    # 2.0, the maximum for four outcomes
```

For `n` equally likely outcomes the entropy is `log2(n)` bits, the number of yes-or-no questions needed in the best case. The common mistake is to compute entropy from continuous values without deciding the outcome set. The result depends on how you bin the values, so state the outcomes first.

The honest limitation is that entropy describes a known distribution. Estimated from counts, it is biased low when rare outcomes are unseen. In the market, entropy chooses splits in decision trees and measures label diversity.

---

## Cross-entropy

Cross-entropy asks what happens when the truth follows `p` but your model believes `q`. The average surprisal is measured with `q`:

```
H(p, q) = sum over outcomes x of p(x) * (-log q(x))
```

If `q` matches `p`, cross-entropy equals entropy. Otherwise it is larger. For a classifier, the truth is a one-hot vector, so only the true class contributes and the loss becomes `-log q(true class)`.

Consider logits `[2.0, 1.0, 0.1]` with the true class first. The softmax gives about `[0.659, 0.242, 0.099]`, and the loss is `-log(0.659)`, about 0.417 nats.

```python
import numpy as np

def softmax(logits):
    z = np.asarray(logits, dtype=float)
    z = z - z.max()                       # subtract the max to avoid overflow
    e = np.exp(z)
    return e / e.sum()

probs = softmax([2.0, 1.0, 0.1])
print(probs.round(4))                     # [0.659  0.2424 0.0986]
print(round(-np.log(probs[0]), 4))        # 0.417 nats

print(round(-np.log(softmax([0.1, 3.0, 0.1])[0]), 4))   # 3.0044 nats, confidently wrong
```

When the model is confidently wrong, the loss grows toward infinity as the probability of the true class approaches zero. The common mistake is computing `log(0)`, which produces `inf` and then `nan` in a training run. Libraries avoid this by computing log-softmax directly from the logits. If you write your own loss, do the same or clip probabilities at a small value such as `1e-12`.

In the market, cross-entropy is the default classification loss in PyTorch and TensorFlow, and it is what you minimize when you fit logistic regression.

---

## KL divergence concept

KL divergence is the extra surprise from using `q` instead of the truth `p`. It is the cross-entropy minus the entropy:

```
KL(p || q) = H(p, q) - H(p) = sum over x of p(x) * log(p(x) / q(x))
```

It is never negative, and it is zero only when the distributions are identical. It is not symmetric. Take `p = [0.9, 0.1]` and `q = [0.5, 0.5]`. Then `KL(p || q)` is about 0.531 bits and `KL(q || p)` is about 0.737 bits.

```python
import numpy as np

def kl_divergence(p, q, base=2):
    p = np.asarray(p, dtype=float)
    q = np.asarray(q, dtype=float)
    mask = p > 0
    return float(np.sum(p[mask] * np.log(p[mask] / q[mask])) / np.log(base))

print(round(kl_divergence([0.9, 0.1], [0.5, 0.5]), 3))   # 0.531
print(round(kl_divergence([0.5, 0.5], [0.9, 0.1]), 3))   # 0.737
```

A frequent error is treating KL divergence as a distance and swapping its arguments freely. The first argument is the truth, and the order changes the answer. The honest limitation is that KL is infinite when `q` gives zero probability to an outcome that `p` allows, which is why smoothing matters in practice. In the market, it appears in knowledge distillation, variational autoencoders and drift monitoring.

---

## Why cross-entropy appears in classification and language models

Cross-entropy comes from maximum likelihood. If each label is drawn from the model's predicted distribution, the probability of the training set is a product of probabilities. Maximizing that product is the same as minimizing the sum of negative log probabilities, which is cross-entropy. The average over examples is the loss you see in a training log.

The gradient explains why it trains well. With softmax and cross-entropy, the gradient with respect to the logits is the predicted probabilities minus the one-hot target. Here is a confidently wrong example, with the true class first:

```python
import numpy as np

def softmax(logits):
    z = np.asarray(logits, dtype=float)
    z = z - z.max()
    e = np.exp(z)
    return e / e.sum()

logits = np.array([-4.0, 4.0, 0.0])   # the model strongly prefers class 1
y = np.array([1.0, 0.0, 0.0])         # the true class is 0

p = softmax(logits)
ce_grad = p - y                        # gradient of cross-entropy with respect to the logits

J = np.diag(p) - np.outer(p, p)        # derivative of softmax
mse_grad = J.T @ (2 * (p - y))         # gradient of squared error on the probabilities

print(ce_grad.round(4))                # [-0.9997  0.9817  0.018 ]
print(mse_grad.round(4))               # [-0.0013  0.0353 -0.034 ]
```

The cross-entropy gradient for the true class is about -1, a strong push to raise that logit. The squared error gradient is about -0.001. The softmax is saturated, so its derivative is nearly zero, and the model barely learns from a confident mistake. That is why cross-entropy trains classifiers faster than squared error on probabilities.

For language models, the same loss applies to each next token, and perplexity is its exponential. A model that gives probability 0.25 to the correct token at every step has an average loss of `-log(0.25)`, about 1.386 nats, and a perplexity of 4. That means it is as uncertain as a uniform choice among four tokens. Perplexity compares models only when they share a vocabulary, tokenizer and evaluation text.

The honest limitation is that low cross-entropy does not mean a useful model. A model can be well fitted on common cases and miscalibrated on rare ones, so check calibration and task metrics alongside the loss. In the market, every classifier that reports probabilities is trained with this loss, and every language model release reports a loss or perplexity on held-out text.

---

## Hands-on exercises

Create a folder called `day15-practice` and write your code in `practice.py`. Use the Python prompt for arithmetic checks, and write down each result.

1. **Surprisal.** Compute the information in bits of a fair coin landing heads, one specific card from 52, and a roll of 6 on a fair die. Check each with `math.log2`.

2. **Entropy.** Write `entropy(p)` in bits. Compute it for `[1/3, 1/3, 1/3]`, `[0.7, 0.2, 0.1]` and `[0.98, 0.01, 0.01]`. The first should equal `log2(3)`, about 1.585 bits. Write one sentence on which distribution is most predictable.

3. **Wrong model.** For `p = [0.5, 0.25, 0.125, 0.125]`, compute the cross-entropy against `q = [0.25, 0.25, 0.25, 0.25]` and against `q = [0.1, 0.1, 0.4, 0.4]`. The second should be larger.

4. **Check the identity.** For the same pair, confirm that `cross_entropy = entropy + KL` to within `1e-9`. Swap the arguments of KL and note whether the value changes.

5. **Logits and loss.** With true class 0, compute the cross-entropy in nats for logits `[2.0, 1.0, 0.1]`, `[10.0, 1.0, 0.1]` and `[-5.0, 1.0, 0.1]`. Explain in one sentence why the loss changes as it does.

6. **Gradients.** Compare the cross-entropy and squared error gradients twice: with logits `[-4, 4, 0]` and true class 1 (the model is correct), and with logits `[4, -4, 0]` and true class 1 (the model is confidently wrong). Write one sentence on which gradient is large in the wrong case.

**Evidence to commit:** `practice.py` and `notes.md` with your numbers and sentences.

---

## Project: an information theory lab

This project is light, because the topic is conceptual. The goal is a small repository with the measures implemented, the identities tested and one experiment on the gradient gap. Name it `100-days-ai-engineering-day-15-information-theory`.

**Repo layout**

```
100-days-ai-engineering-day-15-information-theory/
├── README.md
├── requirements.txt
├── info_theory.py
├── experiment.py
└── tests/
    └── test_info_theory.py
```

**Starter code**

`requirements.txt`:

```
numpy>=1.24
pytest>=7.0
```

`info_theory.py`:

```python
"""Information measures in bits (log base 2) and nats (natural log)."""
import numpy as np


def softmax(logits):
    z = np.asarray(logits, dtype=float)
    z = z - z.max()                      # subtract the max for numerical stability
    e = np.exp(z)
    return e / e.sum()


def surprisal(p, base=2):
    """Information in one outcome that has probability p."""
    return -np.log(p) / np.log(base)


def entropy(p, base=2):
    """Average surprisal of outcomes drawn from distribution p."""
    p = np.asarray(p, dtype=float)
    p = p[p > 0]                         # 0 * log(0) is taken to be 0
    return float(-np.sum(p * np.log(p)) / np.log(base))


def cross_entropy(p, q, base=2):
    """Average surprisal when outcomes come from p but are coded with q."""
    p = np.asarray(p, dtype=float)
    q = np.asarray(q, dtype=float)
    mask = p > 0
    return float(-np.sum(p[mask] * np.log(q[mask])) / np.log(base))


def kl_divergence(p, q, base=2):
    """Extra bits paid for using q when the truth is p."""
    return cross_entropy(p, q, base) - entropy(p, base)
```

`experiment.py`:

```python
import numpy as np
from info_theory import softmax, entropy, cross_entropy, kl_divergence

p = [0.5, 0.25, 0.125, 0.125]
q = [0.25, 0.25, 0.25, 0.25]
print("H(p) =", round(entropy(p), 4), "H(p,q) =", round(cross_entropy(p, q), 4), "KL(p||q) =", round(kl_divergence(p, q), 4))

a, b = [0.9, 0.1], [0.5, 0.5]
print("KL(a||b) =", round(kl_divergence(a, b), 4), "KL(b||a) =", round(kl_divergence(b, a), 4))

# One example, true class 0, confidently wrong logits.
logits = np.array([-4.0, 4.0, 0.0])
y = np.array([1.0, 0.0, 0.0])
p_hat = softmax(logits)
J = np.diag(p_hat) - np.outer(p_hat, p_hat)           # softmax Jacobian
ce_grad = p_hat - y                                     # d(CE)/d(logits)
mse_grad = J.T @ (2 * (p_hat - y))                      # d(MSE on probabilities)/d(logits)
print("probs", p_hat.round(4))
print("CE grad wrt logits  ", ce_grad.round(4))
print("MSE grad wrt logits ", mse_grad.round(4))
```

Running `python experiment.py` prints:

```
H(p) = 1.75 H(p,q) = 2.0 KL(p||q) = 0.25
KL(a||b) = 0.531 KL(b||a) = 0.737
probs [3.000e-04 9.817e-01 1.800e-02]
CE grad wrt logits   [-0.9997  0.9817  0.018 ]
MSE grad wrt logits  [-0.0013  0.0353 -0.034 ]
```

`tests/test_info_theory.py`:

```python
import os, sys
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))
import numpy as np
import pytest
from info_theory import softmax, entropy, cross_entropy, kl_divergence


def test_fair_coin_entropy_is_one_bit():
    assert entropy([0.5, 0.5]) == pytest.approx(1.0)


def test_cross_entropy_equals_entropy_plus_kl():
    p, q = [0.5, 0.25, 0.125, 0.125], [0.25, 0.25, 0.25, 0.25]
    assert cross_entropy(p, q) == pytest.approx(entropy(p) + kl_divergence(p, q))


def test_kl_is_zero_only_when_distributions_match():
    p = [0.2, 0.3, 0.5]
    assert kl_divergence(p, p) == pytest.approx(0.0, abs=1e-12)
    assert kl_divergence(p, [1/3, 1/3, 1/3]) > 0


def test_softmax_is_stable_for_large_logits():
    probs = softmax([1000.0, 1001.0, 1002.0])
    assert np.isfinite(probs).all()
    assert probs.sum() == pytest.approx(1.0)


def test_one_hot_cross_entropy_is_negative_log_probability():
    probs = softmax([2.0, 1.0, 0.1])
    assert -np.log(probs[0]) == pytest.approx(0.417, abs=1e-3)
```

**Step-by-step tasks**

1. Create the repository, the folder tree and a virtual environment. Install the requirements, run `python experiment.py` and compare with the output above.
2. Run `pytest` and make sure all five tests pass.
3. Add `cross_entropy_from_logits(logits, true_index)` that returns the loss in nats using a stable log-softmax. Test it against `-np.log(softmax(logits)[true_index])` for three sets of logits, one of them near 1000.
4. Add `perplexity(token_probs)` that returns the perplexity from the probability given to each true token. A list of four 0.25 values should give 4.0.
5. Write `README.md` explaining in your own words why the cross-entropy gradient for the true class is near -1 when the model is confidently wrong, and why the squared error gradient is not. Include the experiment output.

**README checklist**

- [ ] The project runs with `python experiment.py` on a fresh environment
- [ ] `pytest` passes with at least seven tests
- [ ] The README states the identity `cross_entropy = entropy + KL` with numbers
- [ ] The README explains the gradient gap in your own words
- [ ] The README names one case where a low cross-entropy would still hide a bad model

**Done when**

- Every function matches its reference formula in the tests within `1e-9`
- The stable loss from logits returns finite values for logits near 1000
- The README includes the experiment output and the perplexity check
- A reader can clone the repo and reproduce the experiment in three commands

**Evidence to commit:** the code, the pytest output and the README.

---

## Check your understanding

1. A classifier has a training loss of 0.02 on a dataset where most examples are easy. Is this good? What would you check first?
2. Model A gives the true class probability 0.9 on every example, and Model B gives 0.5. Which has the lower cross-entropy, and by how much per example?
3. A teammate says KL divergence is a distance, so the order of its arguments does not matter. Explain why this is wrong, with numbers.
4. A language model reports perplexity 20 on validation data, and another team reports 18 on what they say is the same data. Can you say the second model is better?
5. You train with squared error on softmax outputs, and the model learns slowly on confident mistakes. Using the gradient argument, explain why, and name the loss you would switch to.
6. A distillation pipeline minimizes `KL(teacher || student)`, and the student gives zero probability to a class the teacher rates at 0.01. What happens, and how would you fix it?

**Answers**

1. Not necessarily. Most examples are easy, so the average can be low while the hard examples and rare classes are poor. Check validation loss, per-class results and calibration.
2. Model A, with loss `-log(0.9)`, about 0.105 nats. Model B has `-log(0.5)`, about 0.693 nats. The difference is about 0.59 nats per example.
3. KL divergence weights the log ratio by the probabilities of the first argument, so swapping the arguments changes the weights. For `[0.9, 0.1]` and `[0.5, 0.5]`, the two orders give 0.531 and 0.737 bits.
4. Not necessarily. Perplexity is comparable only when the vocabulary, tokenizer, evaluation text and log base match. Confirm these before comparing.
5. The squared error gradient is multiplied by the softmax derivative, which is nearly zero when the model is confident. Cross-entropy cancels that factor, so the gradient becomes predicted probability minus target. Switch to cross-entropy.
6. The term `p * log(p / q)` becomes infinite where `q` is zero, so the loss is infinite. Add smoothing to the student's probabilities, or compute the loss from logits with a stable log-softmax.

---

## Sources

**Read**

- Wikipedia: [Information theory](https://en.wikipedia.org/wiki/Information_theory)
- Wikipedia: [Entropy (information theory)](https://en.wikipedia.org/wiki/Entropy_(information_theory))
- Wikipedia: [Cross-entropy](https://en.wikipedia.org/wiki/Cross-entropy)
- Wikipedia: [Kullback-Leibler divergence](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence) (search "Kullback-Leibler divergence" if the link does not open)
- Wikipedia: [Softmax function](https://en.wikipedia.org/wiki/Softmax_function)
- Wikipedia: [Perplexity](https://en.wikipedia.org/wiki/Perplexity)
- Claude Shannon, "A Mathematical Theory of Communication", *Bell System Technical Journal*, 1948. Search the title for a free copy.
- Chris Olah, "Visual Information Theory", on colah.github.io. A short visual introduction to surprisal, entropy and cross-entropy.
- Stanford University, CS229 Machine Learning lecture notes by Andrew Ng, the sections on logistic regression. Search "CS229 lecture notes" on the course website.

**Tutorials**

- GeeksforGeeks: search "Entropy in Information Theory" and "Cross Entropy Loss" on geeksforgeeks.org
- Google Developers, Machine Learning Crash Course, "Logistic regression: loss and regularization". Search the title on developers.google.com

**Watch**

- 3Blue1Brown, "But what is a GPT?" (Deep Learning series), for how a language model outputs a distribution over tokens
- Stanford University, CS224N Natural Language Processing with Deep Learning, the lecture on language models. Search the course website for the lecture title.
- StatQuest with Josh Starmer, the videos on cross-entropy and log loss

**Practice**

- Google Colab (colab.research.google.com), a free notebook environment. Copy the functions from this lesson and plot cross-entropy as the probability of the true class moves from 0 to 1.

---

## Tomorrow

Tomorrow is Day 16, the Mathematics Project. You will build vectors, matrices, a linear regression model, the mean squared error loss and a gradient descent loop from scratch, using NumPy arrays but no machine learning framework. Every piece comes from Days 8 to 15: the vector and matrix operations, the derivative of the loss, the optimization loop and the loss measures you have just studied.

**Next:** [Day 16: Mathematics Project](Day-16-Mathematics-Project.md)

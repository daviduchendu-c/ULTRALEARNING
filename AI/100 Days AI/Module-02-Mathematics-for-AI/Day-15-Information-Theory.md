# Day 15: Information Theory

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** intermediate. You need Day 13 (probability) and the logarithm rules from Day 7. It is for readers who want to understand the loss functions used to train classifiers and language models, and who want to stop treating them as black boxes.

**By the end of this day, you will have:**

- A working implementation of surprisal, entropy, cross-entropy and KL divergence, checked against the identity that links them
- A numerical demonstration that cross-entropy with softmax gives a strong learning signal when a model is confidently wrong, while squared error on probabilities gives almost none
- An explanation, in your own words, of why a classifier's loss is cross-entropy and what perplexity means for a language model
- A small information theory lab repository with tests and an experiment script

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`)
- pytest 7 or newer (`pip install pytest`)
- A calculator, or the Python prompt, for the logarithm checks
- Git and a free GitHub account for the project

**Prerequisites:**

- Day 7: logarithms, exponents and summation notation
- Day 8: the dot product, which appears in the softmax and in the loss
- Day 11: derivatives and the chain rule, used once in the gradient comparison
- Day 13: probability distributions, conditional probability and the idea that a model outputs a probability for each class
- Day 14: the mean, which is the average of the per-example losses

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Background and history](#background-and-history)
- [Glossary](#glossary)
- [Information](#information)
- [Entropy](#entropy)
- [Cross-entropy](#cross-entropy)
- [KL divergence concept](#kl-divergence-concept)
- [Why cross-entropy appears in classification and language models](#why-cross-entropy-appears-in-classification-and-language-models)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: an information theory lab](#project-an-information-theory-lab)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Information theory answers a question that sounds simple: how much does a message tell you? If a friend says "the sun rose this morning," you learn almost nothing. If they say "it snowed in July in Cairo," you learn a lot. The amount of information depends on how surprising the message is. Claude Shannon turned this idea into mathematics in 1948, and the result, measured in bits, now underpins data compression, error-correcting codes and the training of almost every classifier in use today.

On the job, the ideas from this day show up in a few concrete places. A classification model is trained by minimizing cross-entropy, so the loss in your training log is an information measure. A language model is evaluated by perplexity, which is the exponential of the cross-entropy. Model compression and distillation use KL divergence to measure how far a smaller model has drifted from a larger one. If you cannot read these numbers, you cannot tell whether a loss is high because the model is bad or because the labels are wrong.

The big idea of this day is that a loss function is a measure of surprise. A good model is one that is rarely surprised by the true outcome. Cross-entropy is the average surprise, and training is the process of making the true labels less surprising to the model.

---

## Background and history

| Period | What happened | Why it matters today |
|---|---|---|
| 1928 | Ralph Hartley defines a measure of information as the logarithm of the number of equally likely outcomes. | The first step toward counting information in logarithmic units. |
| 1948 | Claude Shannon publishes "A Mathematical Theory of Communication", defining entropy as the average information of a source. | The foundation of the field, and the source of the bit as a unit. |
| 1951 | Solomon Kullback and Richard Leibler introduce the measure of divergence that is now called the Kullback-Leibler divergence. | The formal tool for comparing two distributions, used in statistics and machine learning. |
| 1990s | Cross-entropy becomes the standard loss for neural network classifiers, paired with the softmax output. | Almost every modern classifier is trained with this loss. |
| 2020 onward | Large language models are trained to minimize cross-entropy on next-token prediction, and their quality is reported as perplexity. | The loss you see in a language model training run is an information measure. |

**Takeaway:** the mathematics is from the 1940s and 1950s. What is new is that it is now the training objective for systems with billions of parameters.

---

## Glossary

- **Information (surprisal)**: the amount of surprise in one outcome, equal to the negative log of its probability. Rare outcomes carry more information.
- **Bit**: the unit of information when the logarithm is base 2. One fair coin flip carries exactly one bit.
- **Nat**: the unit of information when the logarithm is natural (base e). Machine learning libraries usually use nats.
- **Entropy**: the average information of outcomes drawn from a distribution. It measures how unpredictable the distribution is.
- **Cross-entropy**: the average surprisal when the outcomes come from one distribution but are judged by another. It is the usual classification loss.
- **KL divergence**: the extra average surprisal you pay for using the wrong distribution. It is zero only when the two distributions are identical.
- **Logits**: the raw, unnormalized scores that a model outputs before the softmax.
- **Softmax**: a function that turns a vector of logits into probabilities that are positive and add to 1.
- **One-hot vector**: a vector with a 1 in the position of the true class and 0 everywhere else.
- **Perplexity**: the exponential of the cross-entropy, measured in nats. It can be read as the effective number of equally likely choices the model is considering.
- **Calibration**: the property that a model's predicted probabilities match the observed frequencies of outcomes.
- **Log loss**: another name for the cross-entropy loss in binary classification.

---

## Information

Imagine a weather report. In a desert city, "no rain today" is almost certain, so the report tells you almost nothing. In a rainforest, "no rain today" would be surprising and worth noting. Information measures how much you learn from an outcome, and that depends on how probable the outcome was before you saw it.

The definition is:

```
information(outcome) = -log(probability of outcome)
```

The minus sign makes the value positive, because the logarithm of a probability between 0 and 1 is negative. A certain outcome, with probability 1, has information `-log(1) = 0`. An unlikely outcome has a large value. Using base 2 gives bits. A fair coin flip has probability 0.5, so its information is `-log2(0.5) = 1` bit. A roll of 6 on a fair die has probability 1/6, so it carries `-log2(1/6)`, which is about 2.58 bits.

```python
import numpy as np

def surprisal(p, base=2):
    return -np.log(p) / np.log(base)

print(surprisal(0.5))              # 1.0 bit, a fair coin
print(round(surprisal(1 / 6), 2))  # 2.58 bits, one face of a die
print(round(surprisal(0.01), 2))   # 6.64 bits, a rare event
```

Information adds up for independent events. If two coin flips are independent, the surprise from seeing heads twice is the sum of the surprise from each flip, which is 2 bits. The logarithm turns the multiplication of probabilities into a sum, which is one reason logs appear everywhere in this subject.

The common mistake is to think of information as a property of the message's content. In information theory it is a property of the probability model. The same word "heads" carries 1 bit in a fair coin and almost no information in a coin that always lands heads. If you change the model, you change the information.

In the market, the same idea appears in data compression. A file compressor gives short codes to common patterns and long codes to rare ones, because common patterns carry less information each. ZIP, gzip and the Huffman codes inside them rely on this principle.

---

## Entropy

Entropy is the average information of a distribution. You compute the surprisal of each outcome and weight it by how often that outcome occurs:

```
H(p) = sum over outcomes x of p(x) * (-log p(x))
```

Entropy is highest when every outcome is equally likely, because then no outcome is predictable. It is zero when one outcome is certain, because nothing can surprise you. For two outcomes, a fair coin has entropy 1 bit and a coin with probability 0.9 for heads has entropy about 0.47 bits.

Here is a four-outcome example. Let `p = [0.5, 0.25, 0.125, 0.125]`. The surprisal of the outcomes is 1, 2, 3 and 3 bits. Their average, weighted by the probabilities, is `0.5 * 1 + 0.25 * 2 + 0.125 * 3 + 0.125 * 3 = 1.75` bits.

```python
import numpy as np

def entropy(p, base=2):
    p = np.asarray(p, dtype=float)
    p = p[p > 0]                  # 0 * log(0) is defined to be 0
    return float(-np.sum(p * np.log(p)) / np.log(base))

print(entropy([0.5, 0.25, 0.125, 0.125]))   # 1.75
print(entropy([0.25, 0.25, 0.25, 0.25]))    # 2.0, the maximum for four outcomes
print(round(entropy([0.9, 0.1]), 4))        # 0.469
```

Entropy has a useful reading. For `n` equally likely outcomes, the entropy is `log2(n)` bits, which is the number of yes-or-no questions you would need on average to identify the outcome in the best case. That is why a uniform distribution over 1,024 options has entropy 10 bits.

The common mistake is to compute entropy from a sample of values without deciding what the outcomes are. If you use a continuous feature, the entropy depends on how you bin it, and the number changes with the bin width. Always state the outcome set before you compute the number.

The honest limitation is that entropy describes a known distribution. In practice you estimate the probabilities from counts, and with few samples the estimate is biased low, because rare outcomes are often not seen at all. Use it for distributions you can count reliably.

In the market, entropy is used in decision tree algorithms to choose splits, in feature selection and in measuring the diversity of a set of labels. It is also the number that a language model's uncertainty is compared against, which is the main idea behind perplexity later today.

---

## Cross-entropy

Cross-entropy asks what happens when the true outcomes come from distribution `p`, but your model believes they come from distribution `q`. The average surprisal is then measured with `q`:

```
H(p, q) = sum over outcomes x of p(x) * (-log q(x))
```

If `q` matches `p`, the cross-entropy equals the entropy. If `q` is different, the cross-entropy is larger, because the model is more surprised than it should be. This is what makes cross-entropy a useful loss: it is smallest when the model's probabilities match the true ones.

For a classification problem, the true distribution `p` is a one-hot vector. If the true class is the second of three classes, then `p = [0, 1, 0]`, and the sum has only one term. The cross-entropy becomes `-log q(true class)`. The loss is simply the negative log of the probability the model gave to the correct answer.

Consider a model that outputs logits `[2.0, 1.0, 0.1]` for three classes, and the true class is the first. The softmax gives probabilities of about `[0.659, 0.242, 0.099]`. The loss is `-log(0.659)`, which is about 0.417 nats.

```python
import numpy as np

def softmax(logits):
    z = np.asarray(logits, dtype=float)
    z = z - z.max()                     # subtract the max to avoid overflow
    e = np.exp(z)
    return e / e.sum()

probs = softmax([2.0, 1.0, 0.1])
print(probs.round(4))                   # [0.659  0.2424 0.0986]
print(round(-np.log(probs[0]), 4))      # 0.417 nats

probs_wrong = softmax([0.1, 3.0, 0.1])  # confidently predicts class 1
print(round(-np.log(probs_wrong[0]), 4))  # 3.0044 nats for true class 0
```

The second example shows the important property. When the model is confidently wrong, the loss is large, around 3 nats, and it grows without limit as the probability of the true class approaches zero. A small probability for the correct answer is punished hard.

The common mistake is to compute `log(0)` when a probability is exactly zero. The loss becomes infinite, and the training run produces `nan`. Libraries avoid this by computing the log of softmax directly from the logits, using a stable formula. If you write your own loss, do the same, or clip the probabilities by a small amount such as `1e-12`.

On the job, cross-entropy is the default loss for classifiers in PyTorch and TensorFlow, and it is what you minimize when you fit a logistic regression. Its values are reported in the training logs of almost every model you will see.

---

## KL divergence concept

KL divergence measures the extra surprise you pay by using `q` instead of the true `p`. It is the cross-entropy minus the entropy:

```
KL(p || q) = H(p, q) - H(p) = sum over x of p(x) * log(p(x) / q(x))
```

KL divergence is always zero or positive. It is zero only when `p` and `q` are identical. It is not symmetric: `KL(p || q)` is generally different from `KL(q || p)`. That makes it a divergence, not a distance in the strict mathematical sense.

Here is the asymmetry with two distributions. Let `p = [0.9, 0.1]` and `q = [0.5, 0.5]`. Then `KL(p || q)` is about 0.531 bits and `KL(q || p)` is about 0.737 bits. The direction matters.

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

Check the identity for the first example. The entropy of `[0.5, 0.25, 0.125, 0.125]` is 1.75 bits, and its cross-entropy with the uniform distribution is 2.0 bits, so the KL divergence is 0.25 bits. The identity holds exactly.

The common mistake is to read KL divergence as a distance and swap its arguments without thinking. The direction determines which distribution plays the role of the truth. In variational methods and in distillation, the choice of direction changes whether the model covers all the modes of the truth or concentrates on one.

The honest limitation is that KL divergence is infinite when `q` gives zero probability to something that `p` allows. A model that assigns zero probability to a possible outcome has an infinite divergence from any distribution that includes it. This is why smoothing and careful handling of zeros matter in practice.

In the market, KL divergence appears in knowledge distillation, where a small model is trained to match the output distribution of a larger one. It also appears in variational autoencoders and in monitoring pipelines that check whether the distribution of live inputs has drifted from the training data.

---

## Why cross-entropy appears in classification and language models

Cross-entropy is not an arbitrary choice. It comes from maximum likelihood. If you assume each label was drawn from the model's predicted distribution, the probability of your whole training set is a product of probabilities. Maximizing that product is the same as minimizing the sum of negative log probabilities, which is the cross-entropy summed over the examples. The average over examples is the mean cross-entropy loss that you see in a training log.

This is why the loss is the right one for classification. A classifier that outputs calibrated probabilities is rewarded for giving high probability to the correct class, and it is penalized in proportion to how surprised it was. The loss keeps pushing the probability of the true class up, rather than only getting the top prediction right.

The gradient shows why this matters in practice. When you combine softmax and cross-entropy, the gradient with respect to the logits is simply the predicted probabilities minus the one-hot target. Here is a single example with the true class first and a confidently wrong model:

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

print(p.round(4))                      # [0.0003 0.9817 0.018 ]
print(ce_grad.round(4))                # [-0.9997  0.9817  0.018 ]
print(mse_grad.round(4))               # [-0.0013  0.0353 -0.034 ]
```

The cross-entropy gradient for the true class is close to -1. The model receives a strong push to raise that logit. The squared error gradient is about -0.001 for the same class. The softmax is saturated, so its derivative is almost zero, and the model barely learns from a confident mistake. That is the reason cross-entropy trains classifiers faster than squared error does on probabilities.

For a language model, the same loss is applied to each next token. At each position, the true token is a one-hot target over the vocabulary. The average cross-entropy over a long text is the training loss, and the perplexity is its exponential:

```python
import numpy as np

# A model that gives probability 0.25 to the correct token at every step
avg_cross_entropy = -np.log(0.25)              # about 1.386 nats per token
perplexity = np.exp(avg_cross_entropy)
print(round(perplexity, 2))                    # 4.0
```

A perplexity of 4 means the model is, on average, as uncertain as choosing uniformly among four tokens. A perplexity of 1 would mean perfect prediction. The number is only comparable between models that use the same vocabulary and the same tokenizer, which is a common source of confusion when people compare reported scores.

The honest limitation is that cross-entropy measures how well the probabilities match the labels, not how useful the model is. A model can have low cross-entropy and still be poorly calibrated for a downstream decision, or be strong on the common cases and weak on rare ones. Check calibration and task metrics alongside the loss.

In the market, every classifier that reports probabilities, from spam filters to clinical risk scores, is trained with this loss. Every large language model release reports a loss or perplexity on a held-out set, and the team behind it uses the same measure to decide whether a new training run is better.

---

## Hands-on exercises (about 60 minutes)

Create a folder called `day15-practice` and write your code in `practice.py`. Use the Python prompt for the arithmetic checks, and write down each result.

1. **Surprisal by hand (10 minutes).** Compute the information in bits of these events: a fair coin landing heads, one specific card from a 52-card deck, and a roll of 6 on a fair die. Write each as `-log2(p)` on paper, then check the values with `math.log2` in Python.

2. **Entropy of three distributions (10 minutes).** Write a function `entropy(p)` in bits. Compute it for `[1/3, 1/3, 1/3]`, `[0.7, 0.2, 0.1]` and `[0.98, 0.01, 0.01]`. Check that the first is `log2(3)`, about 1.585 bits. Write one sentence on which distribution is most predictable.

3. **Cross-entropy with a wrong model (10 minutes).** Take the true distribution `p = [0.5, 0.25, 0.125, 0.125]`. Compute the cross-entropy against `q = [0.25, 0.25, 0.25, 0.25]`, then against `q = [0.1, 0.1, 0.4, 0.4]`. Confirm that the cross-entropy against the second `q` is larger. Record both values.

4. **Check the identity (10 minutes).** For the two distributions in step 3, compute `entropy(p)`, `cross_entropy(p, q)` and `kl_divergence(p, q)`. Confirm that `cross_entropy = entropy + KL` to within `1e-9`. Repeat with the arguments swapped and note whether the KL divergence changes.

5. **Softmax and loss (10 minutes).** Take logits `[2.0, 1.0, 0.1]` and true class 0. Compute the softmax and the cross-entropy in nats. Then change the logits to `[10.0, 1.0, 0.1]` and `[-5.0, 1.0, 0.1]`. Record the loss in each case and explain in one sentence why the loss changes as it does.

6. **Compare gradients (10 minutes).** Repeat the gradient comparison from this lesson twice. First, use logits `[-4, 4, 0]` with the true class set to 1, where the model is correct. Second, use logits `[4, -4, 0]` with the true class set to 1, where the model is confidently wrong. Print both gradients each time. Write one sentence on whether the cross-entropy gradient for the true class is large in the confidently wrong case, and whether the squared error gradient is.

**Evidence to commit:** `practice.py` with all six steps, and a file called `notes.md` with the numbers from each step and the sentences you wrote.

---

## Project: an information theory lab

This project is light because the topic is conceptual at its core. The goal is a small repository that implements the measures from scratch, tests the identities, and runs one experiment that shows why cross-entropy trains classifiers better than squared error. Name the repository `100-days-ai-engineering-day-15-information-theory`.

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

The first printed line shows the identity from the lesson: 2.0 equals 1.75 plus 0.25. The last two lines show the gradient gap. The cross-entropy gradient for the true class is close to -1, while the squared error gradient is close to zero.

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

1. Create the repository, the folder tree and a virtual environment. Install the requirements.
2. Run `python experiment.py` and compare the output with the block above.
3. Run `pytest` and make sure all five tests pass.
4. Add a function `cross_entropy_from_logits(logits, true_index)` that computes the loss directly from the logits using `log(softmax)` with a stable formula, and returns it in nats. Test it against `-np.log(softmax(logits)[true_index])` for three sets of logits, including one with values around 1000.
5. Add a function `perplexity(token_probs)` that takes the probability the model gave to each true token and returns the perplexity. Test it with a list of four 0.25 values, which should give 4.0.
6. Write a `README.md` that explains in your own words why the cross-entropy gradient for the true class is close to -1 when the model is confidently wrong, and why the squared error gradient is not. Include the output of `experiment.py` in the README.

**README checklist**

- [ ] The project runs with `python experiment.py` on a fresh environment
- [ ] `pytest` passes with at least seven tests
- [ ] The README states the identity `cross_entropy = entropy + KL` and shows it with numbers
- [ ] The README explains the gradient gap in your own words, with no copied sentences
- [ ] The README names one case where a low cross-entropy would still hide a bad model

**Done when**

- Every function matches its reference formula in the tests, within `1e-9`
- The stable loss from logits does not return `nan` or `inf` for logits near 1000
- The README includes the numbers from the experiment and the perplexity of 4.0 check
- A reader can clone the repo, install the requirements and reproduce the experiment in three commands

**Evidence to commit:** the code, the tests, the pytest output and the README with the experiment output and your explanation.

---

## Check your understanding

1. A classifier has a training loss of 0.02 on a dataset where most examples are easy. Is this a good result? What would you check before you report it?
2. Two models predict a three-class problem. Model A gives the correct class probability 0.9 for every example. Model B gives 0.5 for every example. Which has the lower cross-entropy, and by roughly how much in nats?
3. A teammate says "KL divergence is a distance, so we can use it in either order." Explain why this is not correct, and give an example where the two orders give different numbers.
4. A language model reports a perplexity of 20 on a validation set. A second team reports 18 on what they say is the same data. Can you say the second model is better? What would you need to confirm?
5. You train a classifier with squared error on softmax outputs and it learns slowly on examples it gets wrong with high confidence. Using the gradient argument from the lesson, explain why, and name the loss you would switch to.
6. A distillation pipeline minimizes `KL(teacher || student)`. The student assigns zero probability to a class that the teacher gives 0.01 probability. What happens to the loss, and how would you fix the pipeline?

**Answers**

1. It is not enough to call it good. A low average loss can come from a dataset where most examples are easy, so check the loss on the hard examples and on a validation set. Also check calibration and the per-class results, since a model can have a low average loss and still be poor on the rare class.
2. Model A has the lower cross-entropy. Its loss per example is `-log(0.9)`, about 0.105 nats. Model B's loss is `-log(0.5)`, about 0.693 nats. The difference is about 0.59 nats per example.
3. KL divergence is not symmetric. `KL(p || q)` weights the log ratio by the probabilities of `p`, so swapping the arguments changes the weights. For `p = [0.9, 0.1]` and `q = [0.5, 0.5]`, the two orders give 0.531 and 0.737 bits. The truth must be placed first.
4. Not necessarily. Perplexity is only comparable when the vocabulary, the tokenizer and the evaluation text are the same, because the number of tokens changes the average. Confirm the tokenizer, the text and the normalization, and check that both teams used the same log base and the same handling of the end of text.
5. With squared error on softmax outputs, the gradient for the true class is multiplied by the softmax derivative, which is close to zero when the model is confident. The model barely changes after a confident mistake. Cross-entropy with softmax cancels that factor, so the gradient is the predicted probability minus the target. Switch to cross-entropy.
6. If the student gives zero probability to a class the teacher gives 0.01, the term `p * log(p / q)` is infinite, so the loss becomes infinite. Fix it by adding smoothing to the student's probabilities, or by computing the loss from the student's logits with a stable log-softmax and clipping the probability before the log.

---

## Sources

**Read**

- Wikipedia: [Information theory](https://en.wikipedia.org/wiki/Information_theory)
- Wikipedia: [Entropy (information theory)](https://en.wikipedia.org/wiki/Entropy_(information_theory))
- Wikipedia: [Cross-entropy](https://en.wikipedia.org/wiki/Cross-entropy)
- Wikipedia: [Kullback-Leibler divergence](https://en.wikipedia.org/wiki/Kullback%E2%80%93Leibler_divergence) (search "Kullback-Leibler divergence" if the link does not open)
- Wikipedia: [Softmax function](https://en.wikipedia.org/wiki/Softmax_function)
- Wikipedia: [Perplexity](https://en.wikipedia.org/wiki/Perplexity)
- Claude Shannon, "A Mathematical Theory of Communication", *Bell System Technical Journal*, 1948. Search the title to find a free copy.
- Chris Olah, "Visual Information Theory", on colah.github.io. A short visual introduction to surprisal, entropy and cross-entropy.
- Stanford University, CS229 Machine Learning course notes by Andrew Ng, the sections on logistic regression. Search "CS229 lecture notes" for the current copy.

**Tutorials**

- GeeksforGeeks: search "Entropy in Information Theory" and "Cross Entropy Loss" on geeksforgeeks.org
- Google Developers, Machine Learning Crash Course, the section on "Logistic regression: loss and regularization" (search the title on developers.google.com)

**Watch**

- 3Blue1Brown, "But what is a GPT?" (the Deep Learning series), for how a language model outputs a probability distribution over tokens
- Stanford University, CS224N Natural Language Processing with Deep Learning, the lecture on language models. Search the course website for the lecture title.
- StatQuest with Josh Starmer, the videos on cross-entropy and log loss. Search "StatQuest cross entropy".

**Practice**

- Google Colab (colab.research.google.com), a free notebook environment. Create a notebook, copy the functions from this lesson, and plot the cross-entropy as the predicted probability of the true class moves from 0 to 1.

---

## Tomorrow

Tomorrow is Day 16, the Mathematics Project. You will build vectors, matrices, a linear regression model, the mean squared error loss and a gradient descent loop from scratch, with NumPy arrays but without any machine learning framework. Every piece you need comes from Days 8 to 15: the vector and matrix operations, the derivative of the loss, the optimization loop and the loss measures you have just studied. The project is where the whole module becomes working code.

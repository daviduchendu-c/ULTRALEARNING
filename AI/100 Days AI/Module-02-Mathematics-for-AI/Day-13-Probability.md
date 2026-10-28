# Day 13: Probability

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** beginner to intermediate. You need arithmetic, fractions and a little algebra. It is for readers who want to reason about uncertain data and who have heard of Bayes' theorem without knowing what it does.

**By the end of this day, you will have:**

- A correct conditional probability and Bayes update, checked against a simulation
- A test of whether two events are independent, done exactly and with NumPy
- A sample drawn from each of three distributions (Bernoulli, binomial, normal), checked against its theoretical mean
- A small probability lab repository that compares exact answers with simulated ones

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`)
- pytest 7 or newer (`pip install pytest`)
- Paper and a pencil for the hand calculations
- Git and a free GitHub account for the project

**Prerequisites:**

- Day 7: fractions, exponents and summation notation
- Day 8: the idea of a vector of numbers
- Day 12: the idea that a model is a function with parameters fitted to data
- Basic Python: functions, loops and list comprehensions

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Background and history](#background-and-history)
- [Glossary](#glossary)
- [Events](#events)
- [Random variables](#random-variables)
- [Conditional probability](#conditional-probability)
- [Bayes' theorem](#bayes-theorem)
- [Independence](#independence)
- [Probability distributions](#probability-distributions)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a probability lab](#project-a-probability-lab)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Machine learning models do not know anything for certain. A spam filter assigns a chance that an email is spam. A medical model gives a risk score. A language model gives a probability for every possible next word. Probability is the language that lets you state these outputs precisely, check whether they are right, and explain them to someone who has to act on them.

On the job, probability shows up in two places. The first is interpretation: a fraud model that says "95 percent" is only useful if you know how often it is right when it says 95 percent. The second is design: many loss functions are probability ideas in disguise, as Day 15 shows.

The big idea is that probability is counting. List what can happen, count the outcomes you care about, and divide. Bayes' theorem and the rest are careful bookkeeping on top of that.

---

## Background and history

| Period | What happened | Why it matters today |
|---|---|---|
| 1654 | Blaise Pascal and Pierre de Fermat exchange letters about dividing the stakes of an unfinished game of chance. | The first systematic probability calculations. |
| 1763 | Thomas Bayes's essay on inverse probability is published after his death, edited by Richard Price. | The root of Bayes' theorem, which updates a belief with evidence. |
| 1933 | Andrey Kolmogorov publishes the axioms that define probability in modern mathematical terms. | The rigorous foundation that all later probability builds on. |

**Takeaway:** the rules are old and stable. What changed is that computers can now count outcomes across millions of simulated cases, which makes the rules easy to check.

---

## Glossary

- **Experiment**: any process with uncertain results, such as rolling a die.
- **Sample space**: the set of all possible outcomes.
- **Event**: a set of outcomes you care about.
- **Probability**: a number from 0 to 1 measuring how likely an event is. The probabilities of all outcomes add to 1.
- **Random variable**: a rule that assigns a number to each outcome.
- **Conditional probability**: the probability of A given B, written P(A | B).
- **Independence**: two events are independent when knowing one does not change the probability of the other.
- **Prior**: your belief about an event before seeing new evidence.
- **Likelihood**: how probable the evidence is if a hypothesis is true.
- **Posterior**: your updated belief after seeing the evidence.
- **Base rate**: how common a condition is before any test.
- **Probability distribution**: a rule listing how probability is spread over the values of a random variable.
- **Bernoulli distribution**: a distribution for one yes-or-no trial.
- **Binomial distribution**: the number of successes in a fixed number of independent yes-or-no trials.
- **Normal distribution**: a bell-shaped distribution defined by a mean and a standard deviation.
- **Monte Carlo simulation**: estimating a probability by generating many random outcomes and counting.

---

## Events

Draw a marble from a bag of 10 marbles, 3 of them red. The sample space is the 10 marbles, and the event "red" has probability 3/10. A probability is always between 0 and 1, and the probabilities of all outcomes add to 1. For a fair die, the event "even" is {2, 4, 6}, so its probability is 3/6.

Counting is the method, and exact fractions keep the logic visible:

```python
from fractions import Fraction

sample_space = [1, 2, 3, 4, 5, 6]
even = [x for x in sample_space if x % 2 == 0]
greater_than_4 = [x for x in sample_space if x > 4]

print(Fraction(len(even), len(sample_space)))            # 1/2
print(Fraction(len(greater_than_4), len(sample_space)))  # 1/3
```

The common mistake is to count outcomes that are not equally likely as if they were. Two coin tosses give 0, 1 or 2 heads, but the probabilities are 1/4, 1/2 and 1/4. List the basic outcomes and their probabilities before you count.

In practice, the sample space of a spam classifier has two labels, and a language model's space has about 50,000 tokens. The logic is the same, only the list is longer.

---

## Random variables

A random variable is a function on the sample space, not a random number in itself. Take two dice and let `S` be the sum of the faces. `S` takes values 2 to 12, but not equally often. Only (1, 1) gives 2, so P(S = 2) = 1/36. Six pairs give 7, so P(S = 7) = 6/36.

```python
from collections import Counter

outcomes = [(a, b) for a in range(1, 7) for b in range(1, 7)]
sums = Counter(a + b for a, b in outcomes)
print(sums[7] / 36)   # 0.1667
```

A discrete random variable takes countable values, such as the number of defective items in a box. A continuous one takes any value in an interval, such as a response time. For a continuous variable, you describe the probability of an interval, because any exact value has probability zero.

The common mistake is to treat the mean of a random variable as its most likely value. For a skewed variable like household income, they can be far apart. On the job, every feature and every model output is a random variable, and a summary such as its mean says nothing about how much individual values vary.

---

## Conditional probability

The conditional probability P(A | B) is the probability of A once you know B happened:

```
P(A | B) = P(A and B) / P(B)
```

You ignore every outcome outside B, and what remains is the part of B where A also holds. Take a deck of 52 cards, with A = "king" and B = "face card" (jack, queen or king). There are 12 face cards and 4 kings, so P(A | B) = 4/12 = 1/3. Before you knew the card was a face card, P(king) was 4/52, about 8 percent.

```python
p_face = 12 / 52
p_king_and_face = 4 / 52
print(round(p_king_and_face / p_face, 4))   # 0.3333
```

The common mistake is to swap the two sides. P(A | B) is not P(B | A). Most people with a passport are adults, but most adults do not hold a passport. Confusing the two directions causes many false alarms in medicine and fraud detection. Always name the condition explicitly.

In the market, every classifier estimates conditional probabilities. A spam model estimates P(spam | words in the email), and a credit model estimates P(default | income, history).

---

## Bayes' theorem

Bayes' theorem turns a conditional probability around. If you know P(evidence | hypothesis), it gives P(hypothesis | evidence):

```
P(H | E) = P(E | H) * P(H) / P(E)
```

The denominator is the total probability of the evidence: `P(E) = P(E | H) * P(H) + P(E | not H) * P(not H)`.

A disease affects 1 in 100 people. A test catches 99 percent of people who have it and gives a false positive for 5 percent of people who do not. A patient tests positive. Most people guess about 95 percent. The calculation gives:

```
P(disease | positive) = 0.99 * 0.01 / (0.99 * 0.01 + 0.05 * 0.99) = 0.1667
```

About 17 percent. The test is strong, but the disease is rare, so most positives come from the large healthy group. Here is the formula and a simulation of one million people:

```python
import numpy as np

def bayes_posterior(prior, sensitivity, false_positive_rate):
    p_positive = sensitivity * prior + false_positive_rate * (1 - prior)
    return sensitivity * prior / p_positive

print(round(bayes_posterior(0.01, 0.99, 0.05), 4))   # 0.1667

rng = np.random.default_rng(42)
n = 1_000_000
has_disease = rng.random(n) < 0.01
tests_positive = np.where(has_disease, rng.random(n) < 0.99, rng.random(n) < 0.05)
print(round(has_disease[tests_positive].mean(), 4))  # about 0.1668
```

The simulation agrees with the formula. Change only the base rate: with a prior of 0.001 the posterior falls to about 0.019, and with a prior of 0.1 it rises to about 0.69.

The common mistake is the base rate fallacy: focusing on the test's accuracy and ignoring how rare the condition is. Write the prior down before you look at the likelihood, and count people in each group.

In the market, Bayes' theorem drives spam filters, screening tools and fraud alerts. Any system that updates a score when new evidence arrives applies this rule.

---

## Independence

Two events are independent when knowing one tells you nothing about the other. Formally, `P(A and B) = P(A) * P(B)`. With two fair dice, let A = "first die even" and B = "second die 5 or 6". P(A) = 0.5, P(B) = 1/3 and P(A and B) = 1/6, which equals 0.5 * 1/3. They are independent.

Now let B = "sum is 8". P(sum is 8) = 5/36, so the product is about 0.0694. But P(A and sum is 8) = 3/36, about 0.0833, because only (2, 6), (4, 4) and (6, 2) have an even first die and a sum of 8. The product rule fails, so the events depend on each other.

```python
outcomes = [(a, b) for a in range(1, 7) for b in range(1, 7)]

def check(event_a, event_b):
    n = len(outcomes)
    p_a = sum(event_a(a, b) for a, b in outcomes) / n
    p_b = sum(event_b(a, b) for a, b in outcomes) / n
    p_ab = sum(event_a(a, b) and event_b(a, b) for a, b in outcomes) / n
    return p_a * p_b, p_ab

print(check(lambda a, b: a % 2 == 0, lambda a, b: b >= 5))       # (0.1667, 0.1667)
print(check(lambda a, b: a % 2 == 0, lambda a, b: a + b == 8))   # (0.0694, 0.0833)
```

The common mistake is to assume independence because the events seem unrelated. Real data often breaks this. Two readings from the same sensor a second apart, or two words in one sentence, are dependent. The Naive Bayes classifier assumes words are independent given the class, which is often accurate enough for spam but makes its probabilities too extreme.

The honest limitation is that the product test needs true probabilities. With a small sample, dependent events can look independent by chance, so use a statistical test or a larger sample before relying on the assumption.

---

## Probability distributions

A probability distribution is the complete description of how probability is spread over a random variable's values. Three distributions recur in this series.

The **Bernoulli distribution** describes one yes-or-no trial with success probability `p`. Its mean is `p` and its variance is `p(1 - p)`. A coin flip has `p = 0.5`, and an ad click has a small `p`.

The **binomial distribution** counts successes in `n` independent Bernoulli trials. Its mean is `n * p` and its variance is `n * p * (1 - p)`.

The **normal distribution** is the bell curve, defined by a mean and a standard deviation. About 68 percent of its probability lies within one standard deviation of the mean, and about 95 percent within two.

```python
import numpy as np

rng = np.random.default_rng(1)

bern = rng.binomial(1, 0.3, size=100_000)
print("bernoulli mean", bern.mean().round(3), "theory 0.3")

binom = rng.binomial(10, 0.3, size=100_000)
print("binomial mean", binom.mean().round(3), "theory 3.0")
print("binomial var", binom.var().round(3), "theory 2.1")

normal = rng.normal(loc=50, scale=10, size=100_000)
print("share within one sigma", round(np.mean(np.abs(normal - 50) < 10), 3), "theory 0.683")
```

The common mistake is to assume a normal distribution because a histogram looks roughly symmetric. Incomes, waiting times and document lengths are often skewed or heavy-tailed, and a normal model would mislead you. Check the shape and compare the mean with the median before you choose a distribution.

In the market, the distribution you choose sets the loss you optimize. Regression with normal noise leads to squared error, and binary outcomes lead to a Bernoulli likelihood. Count data, such as support tickets per day, is often modeled with a Poisson distribution.

---

## Hands-on exercises (about 60 minutes)

Create a folder called `day13-practice` and write your code in `practice.py`. Do steps 1 and 2 on paper first.

1. **Count outcomes (10 minutes).** List the sample space for two dice and count the outcomes with a sum of 8. Write P(sum = 8) as a fraction and a decimal, then confirm with a list comprehension.

2. **Bayes update (10 minutes).** A spam filter sees the word "free" in 60 percent of spam and 5 percent of legitimate email. Assume 20 percent of email is spam. Compute P(spam | "free") on paper, then write `spam_posterior(prior, p_word_given_spam, p_word_given_ham)` and confirm it returns 0.75.

3. **Base rates (10 minutes).** Using sensitivity 0.99 and false positive rate 0.05, compute the posterior for base rates 0.001, 0.01, 0.1 and 0.5. Print a table and write one sentence on why the answer changes so much.

4. **Independence (10 minutes).** For the dice, test three pairs: "first die even" with "second die 5 or 6", "first die even" with "sum is 8", and "first die is 1" with "second die is 1". Record which pairs are independent.

5. **Sample distributions (10 minutes).** With `np.random.default_rng(0)`, draw 100,000 values from a binomial with `n = 20` and `p = 0.4`, and from a normal with mean 0 and standard deviation 2. Compare the sample mean and variance with the theory: 8 and 4.8 for the binomial, and 4 for the variance of the normal.

6. **Find a dependence (10 minutes).** Take any small CSV file with two binary columns. Compare the joint frequency with the product of the marginal frequencies, and write two sentences on whether the independence assumption is reasonable.

**Evidence to commit:** `practice.py` with all six steps, and `notes.md` with your hand calculations, the base rate table and the two sentences.

---

## Project: a probability lab

This project is light, because the topic is mostly conceptual. The goal is a repository that puts exact calculations beside simulations and shows that they agree. Name it `100-days-ai-engineering-day-13-probability`.

**Repo layout**

```
100-days-ai-engineering-day-13-probability/
├── README.md
├── requirements.txt
├── probability_lab.py
├── report.md
└── tests/
    └── test_probability_lab.py
```

**Starter code**

`requirements.txt`:

```
numpy>=1.24
pytest>=7.0
```

`probability_lab.py`:

```python
"""Exact probabilities compared with simulated ones."""
import numpy as np


def bayes_posterior(prior, sensitivity, false_positive_rate):
    """P(disease | positive test) using Bayes' theorem."""
    p_positive = sensitivity * prior + false_positive_rate * (1 - prior)
    return sensitivity * prior / p_positive


def simulate_posterior(prior, sensitivity, false_positive_rate, n=1_000_000, seed=42):
    """Estimate P(disease | positive test) by counting simulated people."""
    rng = np.random.default_rng(seed)
    has_disease = rng.random(n) < prior
    tests_positive = np.where(
        has_disease,
        rng.random(n) < sensitivity,
        rng.random(n) < false_positive_rate,
    )
    return has_disease[tests_positive].mean()


def dice_events():
    """Exact probabilities for two fair dice: A = first die even, B = second die 5 or 6."""
    outcomes = [(a, b) for a in range(1, 7) for b in range(1, 7)]
    total = len(outcomes)
    p_a = sum(a % 2 == 0 for a, _ in outcomes) / total
    p_b = sum(b >= 5 for _, b in outcomes) / total
    p_ab = sum((a % 2 == 0) and (b >= 5) for a, b in outcomes) / total
    return p_a, p_b, p_ab


if __name__ == "__main__":
    print(f"Bayes exact:     {bayes_posterior(0.01, 0.99, 0.05):.4f}")
    print(f"Bayes simulated: {simulate_posterior(0.01, 0.99, 0.05):.4f}")
    p_a, p_b, p_ab = dice_events()
    print(f"P(A) = {p_a:.4f}, P(B) = {p_b:.4f}, P(A and B) = {p_ab:.4f}, P(A)P(B) = {p_a * p_b:.4f}")
```

Running `python probability_lab.py` prints:

```
Bayes exact:     0.1667
Bayes simulated: 0.1668
P(A) = 0.5000, P(B) = 0.3333, P(A and B) = 0.1667, P(A)P(B) = 0.1667
```

`tests/test_probability_lab.py`:

```python
import os, sys
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))
import pytest
from probability_lab import bayes_posterior, simulate_posterior, dice_events


def test_bayes_matches_hand_calculation():
    assert bayes_posterior(0.01, 0.99, 0.05) == pytest.approx(0.1667, abs=1e-4)


def test_simulation_close_to_exact():
    exact = bayes_posterior(0.01, 0.99, 0.05)
    simulated = simulate_posterior(0.01, 0.99, 0.05, n=200_000)
    assert simulated == pytest.approx(exact, abs=0.01)


def test_dice_events_are_independent():
    p_a, p_b, p_ab = dice_events()
    assert p_ab == pytest.approx(p_a * p_b)
```

**Step-by-step tasks**

1. Create the repository, the folder tree and a virtual environment. Install the requirements, run `python probability_lab.py` and compare with the output above.
2. Run `pytest` and make sure all three tests pass.
3. Add `dependent_dice_events()` that returns P(A), P(B) and P(A and B) for "first die even" and "sum is 8". Add a test that shows they are not independent.
4. Add `posterior_table(base_rates)` that returns the posterior for each base rate, and use it to fill a table in `report.md` for 0.001, 0.01, 0.1 and 0.5.
5. Write `report.md` with the table and two sentences on why the base rate matters.

**README checklist**

- [ ] The project runs with `python probability_lab.py` on a fresh environment
- [ ] `pytest` passes with at least four tests
- [ ] `report.md` contains the base rate table
- [ ] The README states which assumption the dice test checks and why the product rule matters
- [ ] The README names one real situation where the independence assumption is likely to fail

**Done when**

- The repository is committed to GitHub with the code, tests, report and README
- The simulated Bayes answer is within 0.01 of the exact one, and a test shows it
- The report explains the base rate effect with numbers from your own table
- A reader can clone the repo and reproduce the numbers in three commands

**Evidence to commit:** the code, the pytest output, and `report.md`.

---

## Check your understanding

1. A screening test has 95 percent sensitivity and 95 percent specificity, and the condition has a base rate of 1 in 1,000. Is a positive result more likely to be a true case or a false alarm? Give a number.
2. Customers who buy product A also buy product B 70 percent of the time. Does A cause B purchases? Which probability statement is the observation?
3. Two sensors fail on the same day 3 percent of the time, and each fails on its own 5 percent of days. Are the failures independent? What does this mean for a redundancy plan?
4. A model's predicted probability is 0.99 for every email it marks as spam. Why could this be a problem even with high accuracy on the test set?
5. You must choose between a Bernoulli and a binomial model for weekly clicks. What decides the choice?

**Answers**

1. A false alarm. Of 100,000 people, 100 have the condition and 95 test positive. Of the 99,900 healthy people, 4,995 test positive as well. So a positive result is a true case only about 1.9 percent of the time, since 95 / (95 + 4,995) is about 0.019.
2. No. The observation is the conditional probability P(B | A) = 0.7. Shared customer types or shared causes can explain it. A claim about the effect of promoting A needs a controlled experiment.
3. They are not independent. Independence would give a joint failure rate of 0.05 * 0.05 = 0.25 percent, but the observed rate is 3 percent. The failures are correlated, perhaps through shared power. The redundancy plan needs separate power or a different measurement method.
4. A probability of 0.99 for every prediction suggests the model is overconfident. Accuracy measures whether the top label is right, not whether the probabilities mean anything. Check calibration by comparing predicted probabilities with the observed frequency of spam in each bin.
5. A Bernoulli model describes one yes-or-no trial. A binomial model fits a count over a fixed number of independent trials with the same probability. If the number of opportunities varies or events depend on each other, neither fits, and a different count model is needed.

---

## Sources

**Read**

- Wikipedia: [Probability theory](https://en.wikipedia.org/wiki/Probability_theory)
- Wikipedia: [Probability axioms](https://en.wikipedia.org/wiki/Probability_axioms)
- Wikipedia: [Conditional probability](https://en.wikipedia.org/wiki/Conditional_probability)
- Wikipedia: [Bayes' theorem](https://en.wikipedia.org/wiki/Bayes%27_theorem)
- Wikipedia: [Independence (probability theory)](https://en.wikipedia.org/wiki/Independence_(probability_theory))
- Wikipedia: [Random variable](https://en.wikipedia.org/wiki/Random_variable)
- Wikipedia: [Base rate fallacy](https://en.wikipedia.org/wiki/Base_rate_fallacy)
- MIT OpenCourseWare, 6.041 Probabilistic Systems Analysis and Applied Probability. Search the course title on ocw.mit.edu.

**Tutorials**

- GeeksforGeeks: search "Bayes Theorem" and "Conditional Probability" on geeksforgeeks.org
- W3Schools: search "Statistics Probability" on w3schools.com

**Watch**

- 3Blue1Brown, "Bayes theorem, the geometry of changing beliefs" (search the title on YouTube)
- StatQuest with Josh Starmer, the videos on conditional probability and Bayes' theorem
- Stanford University, CS109 Probability for Computer Scientists. Search the course website for the lecture on conditional probability.

**Practice**

- Seeing Theory, a free visual introduction to probability and statistics from Brown University. Search "Seeing Theory Brown".

---

## Tomorrow

Tomorrow is Day 14, Statistics. Today you reasoned forward from a model to the data it produces. Statistics runs the other way: you have data and want to infer the model. You will summarize data with the mean, median and variance, measure how two variables move together, and judge whether a sample can be trusted. Those tools rest on the distributions and conditional reasoning from today.

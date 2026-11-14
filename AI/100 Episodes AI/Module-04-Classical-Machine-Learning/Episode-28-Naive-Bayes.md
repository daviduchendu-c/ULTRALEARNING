# Day 28: Naive Bayes

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3 hours
**Level:** beginner to intermediate. You need the conditional probability rules from Module 2 and basic counting. The model is small enough to compute by hand, which you should do once.

**By the end of this day, you will have:**

- A worked calculation of Bayes' theorem on a small example, done by hand and then in code
- A from-scratch multinomial Naive Bayes text classifier with Laplace smoothing
- A comparison of your classifier with scikit-learn's `MultinomialNB` and a logistic regression baseline
- A written note on when the independence assumption fails and why the model can still rank well

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, which includes the 20 Newsgroups loader
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section
- Paper and a pencil for the Bayes calculation in the exercises

**Prerequisites:**

- Day 24: train, validation and test splits, and the baseline habit
- Day 26: probabilities, log loss and thresholds
- Basic text preprocessing: lowercasing, splitting into words and counting them
- Basic counting: the probability of an event is the number of favorable outcomes over the total
- The logarithm rules from Day 26, because you will sum log probabilities to avoid underflow

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Bayes' theorem](#bayes-theorem)
- [Conditional probability](#conditional-probability)
- [Independence assumption](#independence-assumption)
- [Text-classification applications](#text-classification-applications)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Multinomial Naive Bayes for spam filtering](#project-multinomial-naive-bayes-for-spam-filtering)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A spam filter can be useful without understanding an email. It only needs to notice that certain words appear far more often in spam than in ordinary mail. Naive Bayes is the classic model built on that observation, and it was a working spam filter long before large models existed.

The model matters for three reasons. It is one of the fastest classifiers to train, so it is a sensible baseline for text. It is built directly on Bayes' theorem, so it teaches how evidence updates belief, which is the same reasoning used in medical testing, search ranking and fraud scoring. And it is a clear example of a model that is wrong about its own assumptions and still useful, which is a lesson you will need for every model in this module.

On the job, Naive Bayes shows up in text triage, in sentiment baselines, in routing support tickets and in the first pass of content moderation. A team that proposes a large language model for a simple routing task should be asked what a Naive Bayes baseline scored, because it costs minutes to build and gives a number to beat.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1763 | Thomas Bayes's essay, published posthumously by Richard Price, introduces the method of inverse probability. | The theorem that bears his name is the core of this model. |
| 1812 | Pierre-Simon Laplace develops Bayesian probability in *Théorie analytique des probabilités* and popularizes the rule of succession. | The additive smoothing you will use today is a direct descendant of the rule of succession. |
| 1961 | M. E. Maron publishes "Automatic indexing: an experimental inquiry" in the Journal of the ACM, which applies probabilistic classification to document indexing. | Early work on the idea of classifying documents by word probabilities. |
| 1998 | David Lewis reviews Naive Bayes for text classification and shows it is a strong baseline with word-count features. | The paper that made the multinomial text model a standard reference. |
| 1998 | Mehran Sahami and colleagues publish "A Bayesian Approach to Filtering Junk E-Mail", a Naive Bayes spam filter, and the approach spreads through email systems. | The model's first mass-market success was a spam filter, a use that persists today. |

**Takeaway:** the model is more than two centuries old in its core idea, and its success on text came from its simplicity and from the counts it needs.

---

## Glossary

- **Prior**: the probability of a class before looking at the features, such as the share of spam in all mail.
- **Likelihood**: the probability of observing the features given a class, such as the chance that the word "free" appears in spam.
- **Posterior**: the probability of a class after looking at the features, which is what the classifier reports.
- **Evidence**: the overall probability of the observed features, which is the same for all classes and so is often ignored when comparing them.
- **Conditional probability**: the probability of one event given that another has happened, written `P(A | B)`.
- **Independence assumption**: the assumption that features are conditionally independent given the class, which is what makes the model "naive".
- **Bag of words**: a representation of a document as the counts of each word, ignoring order.
- **Multinomial model**: a Naive Bayes variant for count features, where each feature is the number of times a word occurs.
- **Laplace smoothing**: adding a small constant to every count so that no word has zero probability in any class.
- **Log-sum**: computing sums of log probabilities instead of products, to avoid numerical underflow.

---

### Bayes' theorem

Imagine a medical test that is 99 percent accurate, and a disease that affects 1 person in 1,000. If you test positive, the chance you have the disease is not 99 percent. Most positives come from the large healthy group, so the answer is much lower. Bayes' theorem is the rule that gets this right.

From first principles, Bayes' theorem says that `P(C | x) = P(x | C) P(C) / P(x)`. Here `P(C)` is the prior, `P(x | C)` is the likelihood, and `P(C | x)` is the posterior. The denominator `P(x)` is the same for every class, so for classification you compare the numerators `P(x | C) P(C)` across classes and choose the largest. Working the medical example: out of 100,000 people, 100 have the disease and 99 of them test positive, while 99,900 do not and 1 percent of those, or 999, test positive. The chance of disease given a positive test is 99 / (99 + 999), about 9 percent. The prior dominated the result.

```python
prior = 1 / 1000
sensitivity = 0.99        # P(positive | disease)
false_positive = 0.01     # P(positive | no disease)
evidence = sensitivity * prior + false_positive * (1 - prior)
posterior = sensitivity * prior / evidence
print(round(posterior, 4))  # about 0.0901
```

The mistake this example exposes is base-rate neglect: treating the likelihood as if it were the posterior. Naive Bayes avoids it only if the priors are estimated from data and the likelihoods are estimated correctly. The limitation is that the priors are only as good as the training data. If the spam rate changes after deployment, the model keeps its old prior until it is retrained. In the market, a fraud team that uses a rare-event prior must check it monthly, because a small change in the base rate changes the alert volume a great deal.

### Conditional probability

Think of a bag of marbles. The chance of pulling a red marble is one thing. The chance of pulling a red marble given that you already know the bag contains only marbles from the second box is another. The second number depends on the condition, and conditional probability measures exactly that dependence.

From first principles, `P(A | B) = P(A and B) / P(B)`, whenever `P(B) > 0`. For text, the key quantity is the probability that a word appears given a class, estimated by counting. If 40 of 200 spam emails contain the word "free", then `P(free | spam)` is 0.2. Estimating these probabilities for every word and every class is the entire training procedure. For a multinomial model, the probability of a whole document is proportional to a product of word probabilities, each raised to the number of times the word appears.

```python
import numpy as np

# word counts in 200 spam and 800 ham emails, for two words
spam_free, spam_total = 40, 200
ham_free, ham_total = 20, 800
p_free_spam = spam_free / spam_total
p_free_ham = ham_free / ham_total
print(p_free_spam, p_free_ham)  # 0.2 0.025
print("likelihood ratio", round(p_free_spam / p_free_ham, 1))  # 8.0
```

The mistake is estimating a conditional probability from too few examples. A word that appears twice in spam and never in ham gets a probability of zero in ham, which zeroes out the whole product for any email containing it. Smoothing fixes this, and you will see it in the project. The limitation of count-based estimates is that they need enough data for each word and class, which is why rare words are noisy. In industry, teams building keyword-based routing often discover this problem only when a new product name, unseen in training, appears in production.

### Independence assumption

Picture a jury that hears two witnesses. Naive Bayes treats each witness as if their testimony were unrelated to the other, once you know the verdict. In reality, the witnesses may have talked to each other. Ignoring that link makes the combined evidence look stronger than it is.

From first principles, the naive assumption is that the features are conditionally independent given the class, so `P(x_1, ..., x_d | C) = product_j P(x_j | C)`. This makes the likelihood easy to estimate, because each feature is handled alone. For text, it means the presence of "machine" is treated as unrelated to the presence of "learning", even though these words co-occur in ways that carry meaning. The assumption is almost always false. Yet the model often ranks documents correctly, because the errors in the individual factors tend to point the same way and the ranking depends only on the order of scores.

```python
import numpy as np

# two perfectly correlated features: naive Bayes counts the evidence twice
p_a = 0.3   # P(feature A | class)
p_b = 0.3   # P(feature B | class), identical to A
naive_evidence = p_a * p_b
true_joint = p_a            # because B always occurs with A
print("naive product:", naive_evidence, " true joint:", true_joint)
```

The mistake is treating the probabilities that Naive Bayes outputs as calibrated. Because the independence assumption overcounts correlated evidence, the probabilities tend to be too extreme, close to 0 or 1. The ranking can still be good, which is why the model is useful for filtering and triage, but the probabilities should not feed a pricing formula without calibration. The limitation is clear: when correlated features carry the signal, such as the combination of two symptoms, a model that respects the dependence will do better. In the market, many Naive Bayes systems are used for ranking or thresholding, and teams check their outputs with a calibration step before using the number for anything else.

### Text-classification applications

Consider an email with the words "free", "prize" and "winner". Each word, on its own, is evidence for spam. Naive Bayes adds up that evidence, weighted by how often each word appears in each class, and reports the class with the larger total. That is all the model does, and it is fast enough to run on millions of messages.

From first principles, the multinomial model has one parameter per word and class, `theta_{c,w} = P(w | c)`, estimated as the total count of word `w` in class `c` divided by the total number of words in class `c`, with Laplace smoothing added to both. The prediction for a document is the class with the largest `log P(c) + sum_w count_w log theta_{c,w}`. Summing logs prevents numerical underflow when a document has thousands of words. The model is also naturally online, because counts can be updated as new labeled emails arrive.

```python
import numpy as np

def train_nb(counts, labels, alpha=1.0):
    classes = np.unique(labels)
    log_prior, log_theta = {}, {}
    for c in classes:
        rows = counts[labels == c]
        log_prior[c] = np.log(rows.shape[0] / counts.shape[0])
        word_totals = rows.sum(axis=0) + alpha
        log_theta[c] = np.log(word_totals / word_totals.sum())
    return classes, log_prior, log_theta

def predict_nb(x, classes, log_prior, log_theta):
    scores = {c: log_prior[c] + x @ log_theta[c] for c in classes}
    return max(scores, key=scores.get)
```

The limitations are practical. Word order is lost, so "not good" and "good not" look the same. Negation and context are weak. The model needs a clean vocabulary, so tokenization and stop-word choices matter. And it struggles with new vocabulary, because unseen words receive only the smoothing mass. In industry, a support team that routes tickets to queues often starts with Naive Bayes, measures the accuracy per queue, and only moves to a larger model if the errors cost enough to justify it.

---

## Hands-on exercises

1. Work the medical test example by hand with a prevalence of 1 in 500, a sensitivity of 0.95 and a false positive rate of 0.02. Compute the posterior for a positive test, then check your answer in code.
2. Given 300 spam and 700 ham messages where "offer" appears in 90 spam and 35 ham messages, compute `P(offer | spam)`, `P(offer | ham)` and the likelihood ratio. State which class the word favors.
3. Implement `train_nb` and `predict_nb` from the section above, and test them on a tiny hand-made count matrix where you can predict the labels by hand.
4. Train your classifier on the 20 Newsgroups dataset restricted to two categories, using `CountVectorizer` for the counts. Report validation accuracy with smoothing `alpha` equal to 1 and to 0.01.
5. Compare your classifier's predictions with `sklearn.naive_bayes.MultinomialNB` on the same split, and report any disagreement.
6. Compare Naive Bayes with logistic regression on the same features. Record accuracy and training time for each, and write one sentence on which you would ship and why.
7. Plot the predicted probabilities from Naive Bayes against the observed fraction of positives in bins. Describe whether the probabilities are too extreme, and link your answer to the independence assumption.

---

## Project: Multinomial Naive Bayes for spam filtering

This is a medium-weight project. The model is short, so the value is in the evaluation, the smoothing study and the honest comparison with a baseline.

**Repo name:** `100-days-ai-engineering-day-28-naive-bayes`

**Goal:** build a multinomial Naive Bayes spam filter from scratch, check it against scikit-learn, study the effect of smoothing, and write up where the model fails.

**Repo layout:**

```
100-days-ai-engineering-day-28-naive-bayes/
├── README.md
├── requirements.txt
├── nb.py
├── compare.py
├── data/
│   └── .gitkeep
├── results/
│   └── smoothing_table.md
└── tests/
    └── test_nb.py
```

**Starter code for `nb.py`:**

```python
import numpy as np


def fit(X_counts, y, alpha=1.0):
    """Return a dict with keys 'classes', 'log_prior' and 'log_theta'.
    X_counts is a dense or sparse matrix of word counts (n by vocabulary size).
    Use Laplace smoothing with the value alpha.
    """
    raise NotImplementedError


def predict(model, X_counts):
    """Return the predicted class for each row, using the log-sum rule."""
    raise NotImplementedError
```

**Starter code for `tests/test_nb.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from nb import fit, predict


def test_predicts_the_word_that_only_appears_in_one_class():
    X = np.array([[2, 0], [3, 0], [0, 2], [0, 4]])
    y = np.array([0, 0, 1, 1])
    model = fit(X, y, alpha=1.0)
    assert predict(model, np.array([[1, 0], [0, 1]])).tolist() == [0, 1]


def test_log_prior_sums_to_one_in_probability_space():
    X = np.array([[1, 0], [0, 1], [1, 1], [0, 2]])
    y = np.array([0, 0, 1, 1])
    model = fit(X, y)
    assert np.isclose(np.exp(model["log_prior"]).sum(), 1.0)


def test_smoothing_gives_no_zero_probabilities():
    X = np.array([[5, 0], [4, 0], [0, 3]])
    y = np.array([0, 0, 1])
    model = fit(X, y, alpha=1.0)
    assert np.all(np.exp(model["log_theta"]) > 0)
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `pandas` and `pytest`.
2. Get a labeled spam dataset. The SMS Spam Collection from the UCI Machine Learning Repository is a good choice. Check its license before you commit any file, and store it under `data/` only if the license allows it, otherwise download it in the script.
3. Tokenize with `CountVectorizer` from scikit-learn, which is only used to produce counts. Keep your own `fit` and `predict` for the classifier.
4. Implement `fit` and `predict`. Make the three tests pass.
5. In `compare.py`, run the classifier with `alpha` set to 1, 0.1 and 0.01, and record validation accuracy, precision and recall for the spam class in `results/smoothing_table.md`.
6. Compare your predictions with `MultinomialNB` and with a logistic regression on the same features. Report any disagreement.
7. Write the README. State the best smoothing value, the spam-class precision and recall, and the most common kind of error you saw in the false positives.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python compare.py`
- [ ] The dataset source, its license and the download method are stated
- [ ] The smoothing table is included with the spam-class precision and recall
- [ ] The README names one example of a false positive and one of a false negative, with the words that caused them

**Done when:**

- `pytest` passes all three tests
- Your classifier agrees with `MultinomialNB` on at least 95 percent of the validation messages
- The smoothing study shows which alpha works best and why very small alpha can hurt
- You can explain in one paragraph why the independence assumption is wrong for text and why the model still ranks well

**What to commit as evidence:** the source files, the tests, the smoothing table, the comparison output, and the README with the error examples.

---

## Check your understanding

1. A test has 95 percent sensitivity and 95 percent specificity for a condition with prevalence 1 percent. Estimate the chance of disease given a positive result, and explain why it is far below 95 percent.
2. Explain why Naive Bayes multiplies the likelihoods of each word, and what the independence assumption claims about the words.
3. A word appears in 50 spam emails and never in ham. Without smoothing, what probability does the model assign to a new email that contains only that word, and why is this a problem?
4. Naive Bayes outputs a probability of 0.999 for a message that logistic regression scores at 0.7. Explain why this can happen even if the two rank messages the same way.
5. Your team adds a feature that duplicates an existing word count. Predict how Naive Bayes changes its decisions, and explain why logistic regression would react differently.
6. Explain why the model works with log probabilities, and what would happen with a document of 2,000 words if you multiplied the raw probabilities.
7. A product manager asks whether a neural network will beat Naive Bayes on routing support tickets. Describe the test you would run before answering, and what result would justify the extra cost.

### Answers

1. Among 10,000 people, 100 have the condition and 95 test positive. Of the 9,900 without it, 5 percent, or 495, test positive. The chance of disease given a positive is 95 / (95 + 495), about 16 percent. The low prevalence makes most positives come from the healthy group.
2. The model estimates the probability of each word given the class and assumes that, once the class is known, the words do not inform each other. The product of the individual likelihoods is then the joint likelihood. The claim is false for language, but the ranking often still works.
3. The probability of the word given spam is nonzero, but the probability given ham is zero, so the email gets zero probability as ham and is classified as spam regardless of other evidence. Laplace smoothing gives every word a small nonzero probability in each class, which prevents a single unseen word from deciding the result.
4. The independence assumption counts correlated evidence multiple times, which pushes the probabilities toward 0 and 1. Logistic regression, trained with log loss, tends to produce less extreme probabilities. The two can rank messages the same way while disagreeing on the numbers.
5. Naive Bayes counts the duplicated feature's evidence twice, so the decision shifts toward the class that the word favors. Logistic regression would split the weight between the two copies, so the effect is smaller, because it fits coefficients jointly.
6. Multiplying 2,000 probabilities, each typically well below 1, produces a number far smaller than the smallest positive float, so it underflows to zero. Summing log probabilities keeps the values in a safe range and gives the same ordering of classes.
7. I would run both models on the same validation split with the same labels, measure accuracy, the per-queue errors and the cost of each mistake, and time training and prediction. The neural network would need to beat Naive Bayes by a margin that justifies its cost in compute, maintenance and explainability.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Naive Bayes classifier](https://en.wikipedia.org/wiki/Naive_Bayes_classifier)
- Wikipedia: [Bayes' theorem](https://en.wikipedia.org/wiki/Bayes%27_theorem)
- Wikipedia: [Conditional probability](https://en.wikipedia.org/wiki/Conditional_probability)
- Wikipedia: [Additive smoothing](https://en.wikipedia.org/wiki/Additive_smoothing)
- Wikipedia: [Bag-of-words model](https://en.wikipedia.org/wiki/Bag-of-words_model)
- Stanford NLP course notes by Dan Jurafsky and Christopher Manning on Naive Bayes and text classification, from Stanford's online course materials. Search "Stanford Naive Bayes text classification Jurafsky Manning".
- scikit-learn user guide, "Naive Bayes", at scikit-learn.org under the stable documentation.

**Tutorials**

- scikit-learn documentation, `MultinomialNB` and `CountVectorizer`, with worked examples on text.
- GeeksforGeeks, search "Naive Bayes classifiers" for a worked example with a table of probabilities.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Naive Bayes, Clearly Explained!!!" and the video on Bayes' theorem.
- 3Blue1Brown (YouTube), "Bayes theorem, the geometry of changing beliefs", for the visual intuition of updating a prior.

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course and a notebook on text classification with Naive Bayes, which you can run and change in the browser.

---

## Tomorrow

Tomorrow you move from probabilities to rules that look like questions. A decision tree splits the data one feature at a time, choosing each split to make the groups purer. You will meet impurity measures such as Gini and entropy, compute information gain by hand, and see how a tree overfits when it grows too deep. The tree is the building block for the ensemble methods on Day 30.

**Next:** [Day 29: Decision Trees](Day-29-Decision-Trees.md)

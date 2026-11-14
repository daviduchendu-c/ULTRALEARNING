# Day 34: Model Evaluation: Classification

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 26 (probabilities and thresholds) and Day 24 (splits). Metrics are simple to compute and easy to misread, so most of the time goes to choosing and explaining them.

**By the end of this day, you will have:**

- A confusion matrix you can read by hand, and every standard rate derived from its four cells
- A metric report for an imbalanced problem that shows why accuracy alone is misleading
- ROC and precision-recall curves, with the area under each, and an explanation of when each one is the honest choice
- A threshold chosen from a stated cost on validation data, and reported once on the test set

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, which includes all metric functions used here
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the curves
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 24: train, validation and test splits, and the rule that the test set is used once
- Day 26: predicted probabilities, thresholds, precision and recall introduced briefly
- Day 28: conditional probability, which the rates below rely on
- Basic algebra with fractions

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Confusion matrix](#confusion-matrix)
- [Accuracy](#accuracy)
- [Precision](#precision)
- [Recall](#recall)
- [F1](#f1)
- [Specificity concept](#specificity-concept)
- [ROC and AUC](#roc-and-auc)
- [PR curves](#pr-curves)
- [Threshold selection](#threshold-selection)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Classification metrics report with a cost-based threshold](#project-classification-metrics-report-with-a-cost-based-threshold)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A classifier's score is a claim about the world, and the metric is the test of that claim. The same model can look excellent or useless depending on which number you read. A fraud detector that flags nothing can be 99 percent accurate on a dataset where 1 percent of transactions are fraud, and it is worthless. Choosing the right metric is therefore a substantive decision. It is how the business's costs enter the model.

The metrics are simple to compute once you have the four cells of the confusion matrix. The difficulty is in choosing which cell matters most, and in explaining the choice to a person who will act on the number. Doctors, underwriters, moderators and engineers all make decisions with thresholds, and each threshold encodes a trade-off between false alarms and missed cases.

On the job, the metric is the first thing a reviewer asks about. A good report names the baseline, the class balance, the threshold, the cost assumption and the uncertainty. Today you build that report step by step, and you use the same discipline you practiced on Day 24.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1940s | Radar engineers develop signal detection theory, which separates the ability to detect a signal from the decision to report it. | The idea behind the ROC curve started with radar operators deciding whether a blip was an aircraft. |
| 1951 to 1954 | W. Peterson, T. Birdsall and W. Fox publish the theory of signal detectability, and the term receiver operating characteristic enters use. | ROC analysis is named after the receiver operators whose decisions were studied. |
| 1960s to 1970s | Medical diagnosis adopts ROC curves to compare tests and choose cutoffs. | The area under the curve becomes a standard measure of diagnostic accuracy. |
| 1990s | Machine learning and pattern recognition researchers adopt ROC curves and the area under them to compare classifiers. | Machine learning inherits the curve and the area under it as evaluation tools. |
| 2000s | Precision-recall curves and their advantage for imbalanced data are emphasized in information retrieval and machine learning research. | Teams working on rare events now prefer precision-recall curves for their reports. |

**Takeaway:** the metrics came from radar and diagnostic medicine, where the cost of each error was explicit. Your job is to make the same costs explicit in your own problem.

---

## Glossary

- **Confusion matrix**: a table of counts that compares true labels with predicted labels, with four cells for binary problems.
- **True positive (TP)**: a positive row predicted positive.
- **True negative (TN)**: a negative row predicted negative.
- **False positive (FP)**: a negative row predicted positive, also called a false alarm or a Type I error.
- **False negative (FN)**: a positive row predicted negative, also called a miss or a Type II error.
- **Accuracy**: the share of all rows that were predicted correctly.
- **Precision**: TP divided by TP plus FP, the share of positive predictions that are right.
- **Recall (sensitivity, true positive rate)**: TP divided by TP plus FN, the share of real positives that were found.
- **Specificity (true negative rate)**: TN divided by TN plus FP, the share of real negatives that were correctly cleared.
- **F1 score**: the harmonic mean of precision and recall, `2PR / (P + R)`.
- **Receiver operating characteristic (ROC) curve**: a plot of recall against the false positive rate as the threshold varies.
- **False positive rate (FPR)**: FP divided by FP plus TN, which equals one minus specificity.
- **AUC (area under the curve)**: the area under the ROC curve, which equals the probability that a random positive is ranked above a random negative.
- **Average precision**: a summary of the precision-recall curve, computed as a weighted mean of precision over recall steps.
- **Base rate (prevalence)**: the share of positive rows in the data.

---

### Confusion matrix

Think of a sorting machine that puts letters into two bins, "urgent" and "not urgent". After a day, you can count four things: urgent letters in the urgent bin, not-urgent letters in the not-urgent bin, not-urgent letters wrongly in the urgent bin, and urgent letters wrongly in the not-urgent bin. Those four counts contain every rate you will ever need.

From first principles, for a binary classifier with a fixed threshold, each row falls into exactly one of the four cells. The rows of the matrix are the true labels and the columns are the predicted labels, which is the convention scikit-learn uses. Every metric in this lesson is a ratio of these cells, and the choice of metric is the choice of which ratio matters for the decision.

```python
import numpy as np
from sklearn.metrics import confusion_matrix

y_true = np.array([1, 0, 1, 1, 0, 0, 1, 0, 0, 0])
y_pred = np.array([1, 0, 0, 1, 0, 1, 1, 0, 0, 0])
tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()
print(f"TP={tp} FP={fp} FN={fn} TN={tn}")  # TP=3 FP=1 FN=1 TN=5
```

The mistake is reading the matrix with the rows and columns swapped, which flips precision and recall. Check the convention every time, or write the cell names beside the printed table. The limitation of the confusion matrix is that it depends on a threshold. A model's ranking can be excellent while its matrix at one threshold is poor, which is why the curves later in the lesson exist. In the market, every serious metric report starts with the matrix, because the raw counts let a stakeholder see how many cases each error affects.

### Accuracy

Accuracy is the first number people ask for, and it is simple: correct predictions divided by all predictions. It is a fair summary when the classes are roughly balanced and the cost of each error is the same. It fails in other settings.

From first principles, accuracy equals `(TP + TN) / (TP + TN + FP + FN)`. Its weakness is that it mixes the two kinds of correct answer. On a dataset with a base rate of 2 percent positives, a classifier that always predicts negative gets 98 percent accuracy and finds no positives at all. The number is correct and the model is useless. Accuracy is still worth reporting, as long as it sits next to the baseline and the base rate.

```python
import numpy as np
from sklearn.metrics import accuracy_score

y = np.array([1] * 2 + [0] * 98)     # 2 percent positives
always_no = np.zeros_like(y)
print("accuracy of always-negative:", accuracy_score(y, always_no))  # 0.98
```

The mistake is comparing accuracy across datasets with different base rates. Two models with 90 percent accuracy on different datasets may be doing very different jobs. The limitation is that accuracy has no notion of cost: a missed cancer and a false alarm count the same. In industry, accuracy is rarely the decision metric for rare events, though it is useful for balanced tasks such as routing among similar categories.

### Precision

Precision answers the question "when the model says yes, how often is it right?" It matters when a false alarm is expensive, such as sending a fraud investigation to an innocent customer or blocking a legitimate payment.

From first principles, precision is `TP / (TP + FP)`, computed only over rows the model predicted positive. A model that predicts positive rarely and only for obvious cases can have high precision with low recall. Precision is undefined when the model predicts no positives, and scikit-learn returns zero with a warning in that case.

```python
from sklearn.metrics import precision_score

y_true = [1, 0, 1, 1, 0, 0, 1, 0, 0, 0]
y_pred = [1, 0, 0, 1, 0, 1, 1, 0, 0, 0]
print("precision:", round(precision_score(y_true, y_pred), 3))  # 3 / 4 = 0.75
```

The mistake is reporting precision without recall. A model that flags one case and gets it right has precision 1.0 and misses almost everything. The limitation is that precision depends on the base rate: when positives are rare, even a good model can have low precision, because false alarms from the large negative group add up. In the market, a content moderation team tracks precision on flagged items to protect users from wrongful removal, and reports it to a policy group.

### Recall

Recall answers the opposite question: "of the real positives, how many did the model find?" It matters when a miss is expensive, such as failing to flag a fraudulent transaction or missing a disease.

From first principles, recall is `TP / (TP + FN)`, computed over the rows that are truly positive. A model that predicts everything positive has recall 1.0 and precision equal to the base rate. Recall alone is therefore easy to game, which is why it is always reported with precision or with a cost.

```python
from sklearn.metrics import recall_score

y_true = [1, 0, 1, 1, 0, 0, 1, 0, 0, 0]
y_pred = [1, 0, 0, 1, 0, 1, 1, 0, 0, 0]
print("recall:", round(recall_score(y_true, y_pred), 3))  # 3 / 4 = 0.75
```

The mistake is treating recall as a free improvement by lowering the threshold. Each extra positive prediction costs precision or workload, and the trade-off must be made explicitly. The limitation of recall is that it says nothing about the negatives. In industry, screening programs set high recall targets and then manage the false alarms through a second test, which is a deliberate two-stage design.

### F1

F1 combines precision and recall into one number, which is useful for comparing models quickly. It is the harmonic mean, so it is high only when both are high. A model with precision 1.0 and recall 0.0 gets an F1 of 0, not 0.5.

From first principles, `F1 = 2PR / (P + R)`, or equivalently `2TP / (2TP + FP + FN)`. The harmonic mean penalizes imbalance between the two numbers more than an arithmetic mean would. F1 treats precision and recall as equally important, which is rarely true in practice. The F-beta score generalizes it with a weight that favors recall when beta is greater than 1, and precision when beta is less than 1.

```python
from sklearn.metrics import f1_score, fbeta_score

y_true = [1, 0, 1, 1, 0, 0, 1, 0, 0, 0]
y_pred = [1, 0, 0, 1, 0, 1, 1, 0, 0, 0]
print("F1:", round(f1_score(y_true, y_pred), 3))              # 0.75
print("F2 (recall-weighted):", round(fbeta_score(y_true, y_pred, beta=2), 3))
```

The mistake is comparing F1 across models with different thresholds without noting the threshold, or using F1 when the costs of the errors are very different. The limitation is that F1 ignores true negatives entirely, which is a problem when the negatives carry information. In the market, F1 is a common headline metric in information extraction and in search, where it balances finding relevant items with keeping results clean.

### Specificity concept

Specificity is the mirror of recall. It answers "of the real negatives, how many did the model correctly clear?" It matters when a false alarm causes harm, such as a diagnostic test that frightens a healthy patient, or a fraud rule that blocks a good customer.

From first principles, specificity is `TN / (TN + FP)`, and the false positive rate is `FP / (FP + TN)`, which equals one minus specificity. Specificity and recall together describe how a classifier performs on each class. A common report pairs sensitivity and specificity, because medical tests are compared this way. Specificity does not depend on the base rate, which makes it stable across populations, but the predictive value of a test does depend on the base rate, as you saw on Day 28.

```python
from sklearn.metrics import confusion_matrix

y_true = [1, 0, 1, 1, 0, 0, 1, 0, 0, 0]
y_pred = [1, 0, 0, 1, 0, 1, 1, 0, 0, 0]
tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()
print("specificity:", round(tn / (tn + fp), 3))  # 5 / 6 = 0.833
```

The mistake is quoting specificity from a study with one base rate and applying the predictive value to another population. The limitation is the same as for recall: specificity alone says nothing about how many positives are found. In industry, teams that deploy screening rules pair specificity with recall in a single summary table and state the population the numbers come from.

### ROC and AUC

Imagine a judge who gives each defendant a risk score, and a court that decides on a cutoff. Changing the cutoff changes how many guilty people are caught and how many innocent people are wrongly flagged. The ROC curve shows that trade-off for every possible cutoff, and the AUC summarizes the whole curve in one number.

From first principles, sweep the threshold from high to low. At each step, compute the true positive rate (recall) and the false positive rate. Plot the points: the curve starts at (0, 0), ends at (1, 1), and bows toward the top-left corner for good classifiers. The AUC is the area under this curve, and it has a useful interpretation: it equals the probability that a randomly chosen positive gets a higher score than a randomly chosen negative. A random classifier has AUC 0.5. The AUC does not depend on the base rate, which makes it useful for comparing ranking quality, but it can look optimistic when positives are very rare.

```python
import numpy as np
from sklearn.metrics import roc_curve, roc_auc_score

y_true = np.array([0, 0, 1, 1, 0, 1, 0, 1])
scores = np.array([0.1, 0.4, 0.35, 0.8, 0.2, 0.9, 0.5, 0.6])
fpr, tpr, thresholds = roc_curve(y_true, scores)
print("AUC:", round(roc_auc_score(y_true, scores), 3))  # 0.875 on this toy data
```

The mistake is comparing AUC with accuracy as if they measured the same thing, or reading an AUC of 0.9 as a 90 percent chance of being correct at a given threshold. AUC measures ranking, not a single decision. The limitation is that on heavily imbalanced data the ROC curve can look excellent while precision is poor, because the false positive rate is computed against a large number of negatives. In industry, teams report AUC for model comparison and then choose a threshold with precision and recall.

### PR curves

Picture a search engine. The user wants the first results to be relevant, and the rest matter less. A precision-recall curve focuses on the positive class and shows how precision falls as recall rises, which is the view that matters when positives are rare.

From first principles, the PR curve plots precision against recall as the threshold varies. A random classifier gives a horizontal line at the base rate. A good classifier stays high on precision while recall increases. Average precision summarizes the curve as the weighted mean of precision at each recall level, with the weights given by the increase in recall. Unlike ROC AUC, average precision depends on the base rate, so it is not comparable across datasets with different prevalence, but within one dataset it is a sharp measure of how well the top-ranked items are positives.

```python
import numpy as np
from sklearn.metrics import precision_recall_curve, average_precision_score

y_true = np.array([0, 0, 1, 1, 0, 1, 0, 1])
scores = np.array([0.1, 0.4, 0.35, 0.8, 0.2, 0.9, 0.5, 0.6])
precision, recall, thresholds = precision_recall_curve(y_true, scores)
print("average precision:", round(average_precision_score(y_true, scores), 3))
```

The mistake is comparing average precision across datasets with different base rates, as if a value of 0.6 meant the same thing everywhere. The limitation is that the PR curve tells you nothing about true negatives, so it cannot show how many negatives you wrongly flag in absolute terms. In the market, precision-recall curves are common in fraud, ads and information retrieval, because the positives are rare and the cost lies in the top of the ranked list.

### Threshold selection

A threshold is a business decision that looks like a statistical one. It sets how many alerts the team will handle, how many misses it will accept and how much each error costs. The right threshold depends on those costs, so you should state them before looking at the test set.

From first principles, for each candidate threshold on the validation set, compute the number of false positives and false negatives, multiply each by its cost, and sum the expected cost. The threshold that minimizes the total is the cost-optimal choice under the assumed costs. If the costs are uncertain, report a range of thresholds and the cost at each, rather than a single number. Choose the threshold on the validation part, record the choice, and score the test part once.

```python
import numpy as np

def best_threshold(y, scores, cost_fn=5.0, cost_fp=1.0):
    best_t, best_cost = None, np.inf
    for t in np.unique(scores):
        pred = scores >= t
        fn = np.sum((~pred) & (y == 1))
        fp = np.sum(pred & (y == 0))
        cost = cost_fn * fn + cost_fp * fp
        if cost < best_cost:
            best_t, best_cost = t, cost
    return best_t, best_cost

y = np.array([0, 0, 1, 1, 0, 1, 0, 1])
s = np.array([0.1, 0.4, 0.35, 0.8, 0.2, 0.9, 0.5, 0.6])
print(best_threshold(y, s))
```

The mistake is choosing the threshold on the test set, which makes the reported score optimistic. Another is treating the costs as known when they are guesses. The limitation is that a threshold tuned on one period may fail when the score distribution shifts, so monitoring the score distribution and the alert volume is part of the job. In industry, a fraud team sets its threshold from the number of analysts available, then checks the cost each month against actual losses and false alarms.

---

## Hands-on exercises

1. Build the confusion matrix for ten predictions by hand, then check it with `confusion_matrix`. Compute accuracy, precision, recall and F1 by hand from the four cells.
2. For a dataset with 5 percent positives, compute the accuracy of a model that always predicts negative. Then compute its precision, recall and F1, and explain why the accuracy is misleading.
3. Train a logistic regression on an imbalanced dataset that you generate with `make_classification` and `weights=[0.95, 0.05]`. Report accuracy, precision, recall, F1, specificity and AUC on the validation set.
4. Plot the ROC curve and the precision-recall curve on the same dataset, side by side. Describe where the two curves disagree and why.
5. Compute the average precision and the ROC AUC for two models with the same AUC but different precision at the top of the ranking. Explain which one you would deploy for a rare-event alert.
6. Choose the cost-optimal threshold on the validation set for three cost ratios: 1 to 1, 5 to 1 and 20 to 1, using the `best_threshold` function. Report how the threshold changes.
7. Report the test-set metrics once, at the threshold chosen with the 5 to 1 ratio, and write a short paragraph that states the costs, the threshold and the trade-off in plain words.

---

## Project: Classification metrics report with a cost-based threshold

This is a medium-weight project. The code computes standard metrics, so the deliverable is a careful report that a non-technical stakeholder could act on.

**Repo name:** `100-days-ai-engineering-day-34-model-evaluation-classification`

**Goal:** produce a metrics report for an imbalanced binary classifier, choose a threshold from stated costs on validation data, and report the test result once with its context.

**Repo layout:**

```
100-days-ai-engineering-day-34-model-evaluation-classification/
├── README.md
├── requirements.txt
├── metrics_report.py
├── threshold.py
├── results/
│   ├── report.md
│   ├── roc_pr.png
│   └── threshold_costs.md
└── tests/
    └── test_metrics.py
```

**Starter code for `metrics_report.py`:**

```python
import numpy as np


def confusion_counts(y_true, y_pred):
    """Return (tp, fp, fn, tn) as integers."""
    raise NotImplementedError


def summary(y_true, y_pred, scores):
    """Return a dict with accuracy, precision, recall, specificity, F1, ROC AUC and average precision.
    Use the functions from scikit-learn for AUC and average precision. Guard against division by zero.
    """
    raise NotImplementedError
```

**Starter code for `tests/test_metrics.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from metrics_report import confusion_counts, summary


def test_confusion_counts_match_hand_count():
    y_true = np.array([1, 0, 1, 1, 0, 0])
    y_pred = np.array([1, 0, 0, 1, 0, 1])
    assert confusion_counts(y_true, y_pred) == (2, 1, 1, 2)


def test_summary_keys_present():
    y_true = np.array([1, 0, 1, 0])
    y_pred = np.array([1, 0, 0, 0])
    scores = np.array([0.9, 0.2, 0.4, 0.1])
    out = summary(y_true, y_pred, scores)
    assert {"accuracy", "precision", "recall", "specificity", "f1", "roc_auc", "average_precision"} <= set(out)


def test_precision_is_zero_when_nothing_is_predicted_positive():
    y_true = np.array([1, 0, 1])
    y_pred = np.array([0, 0, 0])
    scores = np.array([0.1, 0.2, 0.3])
    assert summary(y_true, y_pred, scores)["precision"] == 0.0
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib` and `pytest`.
2. Create or download an imbalanced binary dataset with a base rate near 5 percent. The `make_classification` function with `weights` works for a practice run. For a real dataset, use one with a clear license, such as UCI's Bank Marketing dataset, and state its source.
3. Implement `confusion_counts` and `summary`. Make the tests pass.
4. In `threshold.py`, split the data into train, validation and test parts. Fit a logistic regression on the training part. Report the baseline accuracy of always predicting negative.
5. Choose a threshold on the validation part using the cost function from the lesson, for at least three cost ratios. Save the table to `results/threshold_costs.md`.
6. Choose one cost ratio, state it in the README, and compute the test-set summary once at its threshold.
7. Save a figure with the ROC and precision-recall curves side by side to `results/roc_pr.png`.
8. Write `results/report.md`, which includes the confusion matrix, the summary table, the threshold and the costs, and the test score.
9. Write the README and commit the files. Push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python threshold.py`
- [ ] The dataset source, license and base rate are stated
- [ ] The cost ratio is written in plain words, with who bears each cost
- [ ] The threshold table and the ROC and PR figure are included
- [ ] The README states that the test set was used once

**Done when:**

- `pytest` passes all three tests
- The report shows the always-negative baseline accuracy next to the model's accuracy, and explains why accuracy is not enough
- The threshold is chosen on validation data and the test set is scored once
- You can explain in one paragraph why the ROC AUC can look high while precision is low on rare events

**What to commit as evidence:** the source files, the tests, the threshold cost table, the figure, the report and the README with the cost assumption.

---

## Check your understanding

1. A model has 40 true positives, 10 false positives, 20 false negatives and 930 true negatives. Compute accuracy, precision, recall, specificity and F1, and say which one a team investigating fraud should watch first.
2. Explain why a classifier that predicts every row as positive has recall 1.0 but precision equal to the base rate.
3. Two models have AUC 0.92 and 0.89 on a dataset with a 1 percent base rate. Explain why the higher AUC may still produce more false alarms in practice, and name the curve you would check.
4. Explain what the AUC means in terms of random pairs of positives and negatives, and why a random classifier has AUC 0.5.
5. A colleague chooses the threshold that maximizes F1 on the test set and reports that value. Describe the bias this introduces and the correct procedure.
6. Your fraud model has a precision of 0.2 at the chosen threshold. Explain whether this is necessarily a failure, using the cost of a false alarm and the cost of a missed fraud.
7. A medical test has high specificity and low sensitivity. Explain which kind of error it makes more often, and what that implies for its use as a screening test.

### Answers

1. Accuracy is `(40 + 930) / 1000 = 0.97`. Precision is `40 / 50 = 0.8`. Recall is `40 / 60`, about 0.667. Specificity is `930 / 940`, about 0.989. F1 is `2 * 0.8 * 0.667 / (0.8 + 0.667)`, about 0.727. A fraud team should watch recall first, since missed fraud is the main cost, while keeping precision acceptable so that investigations stay manageable.
2. Predicting every row as positive means every real positive is found, so recall is 1.0. The predictions are positive for every row, so precision equals the share of positives in the data, which is the base rate.
3. On a dataset with a 1 percent base rate, the 99 percent negatives produce many false positives even at a low false positive rate. The ROC curve compares rates against the large negative group, so it can look good. The precision-recall curve is the one to check, because it shows how many of the flagged rows are actually positive.
4. The AUC equals the probability that a randomly chosen positive receives a higher score than a randomly chosen negative. A random classifier ranks positives and negatives in random order, so that probability is one half.
5. Choosing the threshold on the test set uses the test labels to make a decision, so the reported score is optimistic. The correct procedure is to choose the threshold on the validation set, using a stated cost or objective, and then score the test set once at that threshold.
6. Not necessarily. If a missed fraud costs far more than a false alarm, a precision of 0.2 may be the right trade-off, because each alarm is cheap to review and each caught fraud is valuable. The check is to compare the expected cost of the alarms with the expected loss avoided, using real figures.
7. A test with high specificity makes few false positives and produces mostly correct negatives, but low sensitivity means it misses many real cases, so it makes more false negatives. As a screening test, it is weak at finding the condition, and a positive result is useful, but a negative result does not rule it out.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Confusion matrix](https://en.wikipedia.org/wiki/Confusion_matrix)
- Wikipedia: [Precision and recall](https://en.wikipedia.org/wiki/Precision_and_recall)
- Wikipedia: [Receiver operating characteristic](https://en.wikipedia.org/wiki/Receiver_operating_characteristic)
- Wikipedia: [Sensitivity and specificity](https://en.wikipedia.org/wiki/Sensitivity_and_specificity)
- Wikipedia: [F-score](https://en.wikipedia.org/wiki/F-score)
- scikit-learn user guide, "Model evaluation: quantifying the quality of predictions", at scikit-learn.org under the stable documentation, including the sections on the ROC curve and the precision-recall curve.

**Tutorials**

- scikit-learn documentation, the examples "Receiver Operating Characteristic (ROC) with cross validation" and "Precision-Recall", from the gallery.
- GeeksforGeeks, search "confusion matrix in machine learning" for worked examples with the four cells.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Confusion Matrix, Clearly Explained" and "ROC and AUC, Clearly Explained!!!".
- StatQuest with Josh Starmer (YouTube), "Precision and Recall" if the search finds it, or the same channel's classification series.

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course, and a notebook that computes these metrics on a public imbalanced dataset.

---

## Tomorrow

Tomorrow you turn to regression. The classification metrics are about yes-or-no decisions, but many problems predict a number, such as a price or a delivery time. You will meet mean absolute error, mean squared error, root mean squared error, R-squared, and the residual analysis that tells you where a regression fails. The same habits, a baseline, a stated cost and a single test score, carry over directly.

**Next:** [Day 35: Model Evaluation: Regression](Day-35-Model-Evaluation-Regression.md)

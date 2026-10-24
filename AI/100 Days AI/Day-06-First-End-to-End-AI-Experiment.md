# Day 6: First End-to-End AI Experiment

*100 Days of AI Engineering, Module 1: AI Foundations*

**Time:** about 4 to 5 hours
**You will build:** a complete, reproducible experiment on a real medical dataset (569 breast tumor samples). You will load, inspect, preprocess, split, beat a baseline, measure with the right metrics, save predictions, read every mistake, and write it up as an experiment report. This is the first repo in the series that you could show an employer as-is.
**You should know:** Days 1 to 5. In particular: train vs test (Day 2), the lifecycle and baselines (Day 3), classification and why accuracy can mislead (Day 4), and your pinned environment (Day 5).
**You will install:** Python 3.11 or newer with `numpy`, `pandas`, `scipy`, `scikit-learn` and `matplotlib` at the versions from Day 5.

---

## 1. The big idea

For five days you have studied the pieces separately. Today you connect them into one experiment, the unit of work an ML engineer delivers dozens of times a year. An experiment is a chain of decisions, each one written down: where the data came from, what it looks like, how it was split, what the baseline scored, which metric matters and why, what the model got wrong, and what you would try next.

Anyone should be able to clone your repo, run three commands, get your exact numbers, and understand from your report whether to trust them. The report is the part that turns a tutorial copy into a portfolio project.

---

## 2. The dataset and the stage map

The [Breast Cancer Wisconsin (Diagnostic)](https://scikit-learn.org/stable/datasets/toy_dataset.html) dataset was created at the University of Wisconsin in the early 1990s by Dr. William H. Wolberg, W. Nick Street and Olvi L. Mangasarian. Each row describes cell nuclei seen in a digitized image of a fine needle aspirate (a small tissue sample taken with a needle) from a breast mass. Ten properties of the nuclei (radius, texture, perimeter, area, smoothness, compactness, concavity, concave points, symmetry, fractal dimension) are each summarized three ways: the mean, the standard error, and the "worst" (the mean of the three largest values). That gives 30 numeric features. The label is the diagnosis: **malignant** (cancerous) or **benign**.

It ships with scikit-learn, so you need no download, and it is small enough to inspect by eye. It is a teaching dataset. **Nothing you build today is a medical tool**, and the report in section 6 says why.

| Stage (Day 3) | Today's version |
|---|---|
| Problem definition | Flag tumors likely to be malignant for further review |
| Data collection | Already done: 569 rows from one hospital, early 1990s |
| Exploration | `inspect_data.py` |
| Cleaning and features | Check for missing values and duplicates; scale features |
| Baseline | `DummyClassifier` |
| Training, testing | Logistic regression, one held-out test set |
| Evaluation | Metrics, confusion matrix, error inspection |
| Deployment, monitoring | Out of scope today (Modules 10 and 11) |

---

## 3. The experiment, in one picture

```
load ──► inspect ──► split (stratified, seed 42) ──┬──► baseline (DummyClassifier) ──┐
                                                   │                                 ├──► metrics on TEST
                                                   └──► pipeline: scaler → logreg ───┘        │
                                                                                              ▼
            REPORT.md ◄── error inspection ◄── predictions.csv (true, predicted, p_malignant)
```

Every arrow is a file or function in the repo. If a number appears in `REPORT.md`, you can trace it back to the script that produced it.

---

## 4. The nine steps, and the ideas behind them

The project in section 5 has code for each step. This section explains what each step is for and where people go wrong.

### 4.1 Loading

**Intuition.** Before cooking, take every ingredient out and put it on the counter where you can see it.

**What it is.** Reading the data into a structure your code can work with, in one place, the same way every time. `load_breast_cancer(as_frame=True)` returns a pandas table. The project wraps this in one function, `load_dataframe()`, that every script calls, and it turns the 0/1 target into text.

**Trap.** In this dataset **0 means malignant and 1 means benign**, the opposite of what most people guess. If you assume 1 is malignant, every precision and recall you report is for the wrong class. Converting the numbers to words on load stops this mistake from spreading.

### 4.2 Inspection

**Intuition.** Look at a patient before you treat them.

**What it is.** Checking shape, types, missing values, duplicates, class balance and feature ranges before any modeling. You will find 569 rows, 31 columns (30 features plus the diagnosis), no missing values, no duplicates, and a 63% / 37% split between benign and malignant. That balance means "always predict benign" already scores about 63% accuracy.

**Trap.** Skipping inspection because the dataset looks clean. Even clean data hides surprises such as the 0/1 encoding above. Also check the feature scales: `mean area` goes up to 2,501, while `mean smoothness` stays below 0.17.

### 4.3 Preprocessing

**Intuition.** Converting everyone's height to the same unit before comparing them.

**What it is.** Transforming features so the model can use them well. Here the transform is standardization: each feature gets mean 0 and standard deviation 1, so a feature measured in thousands (area) does not dominate one measured in hundredths (smoothness). `StandardScaler` learns each feature's mean and standard deviation from the training rows, then applies `(value - mean) / std`. Putting it inside `make_pipeline(StandardScaler(), LogisticRegression())` makes sure the test rows are scaled with training statistics, never their own.

**Trap.** Scaling the whole dataset before splitting, which is the small leak from Days 2 and 3. Scaling the target in classification, which you should not do because labels are categories. Assuming every model needs scaling. Tree models (Day 29) do not care about feature scale, but logistic regression's optimizer converges much better with it.

### 4.4 Train/test split

**Intuition.** Lock the final exam in a drawer before you start studying.

**What it is.** Holding out a share of rows (here 25%, which is 143 rows) that the model never sees during training. `stratify=y` keeps the 63/37 balance in both parts. `random_state=42` makes the split identical on every run. The result is 426 training rows and 143 test rows. The test set has 53 malignant and 90 benign tumors.

**Trap.** Splitting after preprocessing. Re-splitting with new seeds until the test score looks good. Forgetting that 143 rows is small. Each test row is worth 0.7 percentage points of accuracy, so a gap of one or two points between models can be pure luck. When rows have time order, or several rows belong to one patient, split by time or by group instead (Module 4).

### 4.5 Baseline model

**Intuition.** A thermometer that always says 20 °C is right in some rooms. You need to know how often before you praise a real thermometer.

**What it is.** The simplest predictor. Here it is `DummyClassifier(strategy="most_frequent")`, which predicts "benign" for every tumor. It scores 0.629 accuracy and 0.0 recall on malignant. It never finds a single cancer. Any accuracy below about 63% is worse than doing nothing, and a model with high accuracy but low recall is close to useless for this task.

**Trap.** Skipping it. Without the baseline, "96% accuracy" sounds amazing. With it, you know the model adds about 34 points over doing nothing, and you know accuracy alone is the wrong headline.

### 4.6 Metrics

**Intuition.** A smoke alarm that never rings has no false alarms and is still useless. You have to count both kinds of mistake.

**What it is.** The **positive class** is the class you are trying to detect, here **malignant**. Precision and recall are computed for it. A **confusion matrix** counts four outcomes:

- **True positive (TP):** malignant, predicted malignant.
- **False negative (FN):** malignant, predicted benign. A missed cancer.
- **False positive (FP):** benign, predicted malignant. A false alarm.
- **True negative (TN):** benign, predicted benign.

From those counts:

- **Precision** = TP / (TP + FP). Of everything flagged malignant, the share that truly was.
- **Recall** (sensitivity) = TP / (TP + FN). Of all truly malignant tumors, the share that was flagged.
- **F1 score** is the harmonic mean of precision and recall. It is low if either one is low.
- **Accuracy** is the share of all rows that were right. It is context here, not the headline.

Today's test results make this concrete:

```
                     predicted malignant   predicted benign
actually malignant          TP = 49             FN = 4
actually benign             FP = 1              TN = 89
```

- Precision = 49 / (49 + 1) = 0.980. When the model says malignant, it is right 98% of the time.
- Recall = 49 / (49 + 4) = 0.925. It finds 92.5% of cancers and misses 4 of 53.
- Accuracy = (49 + 89) / 143 = 0.965.

**Trap.** Leading with accuracy on an imbalanced problem. Computing precision and recall for the wrong class. scikit-learn's default is `pos_label=1`. With text labels that raises an error. With 0/1 labels it would silently score "benign" here, so always set `pos_label` explicitly. Treating the two error types as equal. In screening, a missed cancer (FN) is usually far worse than a false alarm (FP), which is why a flagged case gets an extra test rather than a diagnosis.

The model also outputs a **predicted probability** for each row, and a **decision threshold** (0.5 by default) turns that probability into a label. Section 6 changes that threshold on purpose.

### 4.7 Predictions

**Intuition.** Keep the marked exam papers as well as the final grade.

**What it is.** Saving, for every test row, the true label, the predicted label and the predicted probability. The file is the raw material for error analysis and for anyone who wants to check your metrics. With the probabilities saved, you can test other thresholds without retraining.

**Trap.** Saving only labels and losing the probabilities. Losing the row ID, so you cannot join predictions back to the original features. Production systems log every prediction with an ID and the model version, which is what makes monitoring (Day 93) possible.

### 4.8 Error inspection

**Intuition.** A coach reviews the tape of the goals the team let in, as well as the final score.

**What it is.** Reading the misclassified rows one by one, comparing their features with the class averages, and checking how confident the model was. Today's model makes 5 errors: 4 malignant tumors predicted benign and 1 benign tumor predicted malignant. The missed cancers are smaller than a typical malignant tumor. Their mean radius (13.8 to 15.1) sits between the benign average (12.1) and the malignant average (17.5). Three of the five errors had probabilities between 0.3 and 0.7, so the model was unsure. The fourth missed cancer (row 73) got a malignant probability of only 0.056. The model was confidently wrong, and that is the row a clinician would worry about most.

**Trap.** Stopping at the metric. Looking only at errors and never at correct predictions for comparison. Drawing big conclusions from 5 rows. Error analysis gives you hypotheses to test, not proof. ML teams use error analysis to choose the next step, such as more data of a certain kind, a new feature, or a different threshold. Day 37 is devoted to it.

### 4.9 Documentation

**Intuition.** A lab experiment that is not written down did not happen, as far as anyone else is concerned.

**What it is.** A README for how to run the code, and an experiment report (`REPORT.md`) for what you found: question, data, method, results, errors, limitations and next steps. For models other people may use, a **model card** adds intended use and out-of-scope uses. Mitchell et al. introduced model cards in 2019, and the Hugging Face Hub now hosts them as standard.

**Trap.** Writing the report from memory instead of from the saved results. Hiding limitations. Claiming more than the data supports. "Detects cancer with 96.5% accuracy" is a claim about a medical device. "Logistic regression reached 0.925 recall on a 143-row held-out set" is a claim about an experiment.

---

## 5. Hands-on exercises (about 60 minutes)

Do these before the project. They use the same dataset.

**Exercise 1: confusion matrix by hand (about 15 minutes).** A different model on a different split gives TP = 45, FN = 8, FP = 3, TN = 87. Compute precision, recall and accuracy by hand. Is this model better or worse than today's for screening, and why? (Answer: precision 45/48 = 0.938, recall 45/53 = 0.849, accuracy 132/143 = 0.923. It is worse, because it misses 8 cancers instead of 4.)

**Exercise 2: find the label trap (about 15 minutes).** Run:

```python
from sklearn.datasets import load_breast_cancer
data = load_breast_cancer()
print(data.target_names, data.target[:5])
```

Read `target_names` and work out which number means malignant. Then print the dataset description with `print(data.DESCR)` and find the "Class Distribution" line. Write down both, in your own words, in your notes.

**Exercise 3: precision vs recall in your own words (about 30 minutes).** For each scenario, say whether you would favor precision or recall, and the cost of each error type: (a) cancer screening, (b) an email spam filter, (c) a system that automatically blocks bank accounts for suspected fraud, (d) a search engine's first page of results. (Suggested answers: (a) recall, because a missed cancer is worse; (b) precision, because losing a real email is worse than seeing some spam; (c) precision, because blocking innocent customers is very costly, so use review queues instead; (d) precision at the top of the list.)

---

## 6. Project: a reproducible end-to-end experiment

**Goal:** a repo that anyone can clone and run with three commands to reproduce your numbers, with an experiment report that explains them.

Suggested repo name: `100-days-ai-engineering-day-06-first-experiment`.

### Repository layout

```
day-06-first-experiment/
├── README.md                # how to set up and run
├── REPORT.md                # the experiment report (step 7)
├── .gitignore               # from Day 5
├── requirements.txt
├── env_check.py             # optional, copied from Day 5
├── src/
│   ├── data.py              # loading and splitting, shared by every script
│   ├── inspect_data.py      # step 2: inspection
│   ├── train.py             # steps 3 to 7: preprocess, baseline, model, metrics, predictions
│   ├── inspect_errors.py    # step 8: error inspection
│   └── thresholds.py        # stretch: other decision thresholds
└── results/                 # created by train.py, committed so readers can see your numbers
    ├── metrics.json
    ├── predictions.csv
    └── confusion_matrix.png
```

### Step 1: set up

```bash
mkdir day-06-first-experiment
cd day-06-first-experiment
mkdir src
python -m venv .venv
```

Activate it, copy `.gitignore` (and optionally `env_check.py`) from Day 5, then create `requirements.txt`. This project does not need PyTorch or Jupyter, so the list is shorter than Day 5's, with the same versions:

```text
numpy==2.2.6
pandas==2.2.3
scipy==1.15.2
scikit-learn==1.6.1
matplotlib==3.10.0
```

Install with `python -m pip install -r requirements.txt`. **Run every script from the repo root** (the folder that contains `src/`), for example `python src/train.py`, so that the `results/` folder is created in the right place.

### Step 2: loading and splitting (`src/data.py`)

```python
"""Loading and splitting. Every other script gets its data from here, so the split never changes."""

import pandas as pd
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split

SEED = 42
TARGET = "diagnosis"


def load_dataframe() -> pd.DataFrame:
    """Return the 569 tumors as one table, with a readable text label."""
    bunch = load_breast_cancer(as_frame=True)
    df = bunch.frame.rename(columns={"target": TARGET})
    # In this dataset 0 means malignant and 1 means benign. Make that impossible to forget.
    df[TARGET] = df[TARGET].map({0: "malignant", 1: "benign"})
    return df


def split(df: pd.DataFrame):
    """Hold out 25% of rows as a test set, keeping the class balance the same in both parts."""
    X = df.drop(columns=[TARGET])
    y = df[TARGET]
    return train_test_split(X, y, test_size=0.25, stratify=y, random_state=SEED)
```

Every script imports `load_dataframe` and `split` from here. If you change the seed or the split size, the change applies everywhere at once, and no script can quietly use a different split.

### Step 3: inspection (`src/inspect_data.py`)

```python
"""Stage 2: look at the data before modeling anything."""

from data import TARGET, load_dataframe


def main():
    df = load_dataframe()

    print(f"shape: {df.shape[0]} rows, {df.shape[1]} columns")
    print(f"missing values: {int(df.isna().sum().sum())}")
    print(f"duplicate rows: {int(df.duplicated().sum())}")

    print("\nclass balance:")
    print(df[TARGET].value_counts().to_string())
    print(df[TARGET].value_counts(normalize=True).round(3).to_string())

    print("\nfeature ranges (first 5 features):")
    print(df.iloc[:, :5].describe().loc[["min", "mean", "max"]].round(2).to_string())

    print("\nmean of 3 features per class:")
    cols = ["mean radius", "mean concave points", "worst area"]
    print(df.groupby(TARGET)[cols].mean().round(3).to_string())


if __name__ == "__main__":
    main()
```

Run `python src/inspect_data.py`. Expected output:

```text
shape: 569 rows, 31 columns
missing values: 0
duplicate rows: 0

class balance:
diagnosis
benign       357
malignant    212
diagnosis
benign       0.627
malignant    0.373

feature ranges (first 5 features):
      mean radius  mean texture  mean perimeter  mean area  mean smoothness
min          6.98          9.71           43.79     143.50             0.05
mean        14.13         19.29           91.97     654.89             0.10
max         28.11         39.28          188.50    2501.00             0.16

mean of 3 features per class:
           mean radius  mean concave points  worst area
diagnosis                                              
benign          12.147                0.026     558.899
malignant       17.463                0.088    1422.286
```

Three findings go into your report. There are no missing values or duplicates, so no cleaning is needed. The classes are imbalanced (63/37). The features have very different scales (area in the thousands, smoothness below 0.2), so the model needs scaling. Malignant tumors are larger on average, with more concave points.

### Step 4: baseline, model, metrics and predictions (`src/train.py`)

```python
"""Stages 3 to 7: preprocess, train a baseline and a real model, measure, save predictions."""

import json
from pathlib import Path

import matplotlib

matplotlib.use("Agg")  # draw to files, no window needed
import matplotlib.pyplot as plt
import pandas as pd
from sklearn.dummy import DummyClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import (
    ConfusionMatrixDisplay,
    accuracy_score,
    classification_report,
    precision_score,
    recall_score,
)
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

from data import SEED, load_dataframe, split

RESULTS = Path("results")
POSITIVE = "malignant"  # the class we care most about catching


def evaluate(name, y_true, y_pred):
    scores = {
        "accuracy": round(accuracy_score(y_true, y_pred), 3),
        "precision_malignant": round(
            precision_score(y_true, y_pred, pos_label=POSITIVE, zero_division=0), 3
        ),
        "recall_malignant": round(recall_score(y_true, y_pred, pos_label=POSITIVE), 3),
    }
    print(f"{name:<10} {scores}")
    return scores


def main():
    RESULTS.mkdir(exist_ok=True)
    X_train, X_test, y_train, y_test = split(load_dataframe())
    print(f"train rows: {len(X_train)}, test rows: {len(X_test)}\n")

    # Baseline: always guess the most common class. Any real model must beat this.
    baseline = DummyClassifier(strategy="most_frequent").fit(X_train, y_train)

    # Real model: scale every feature, then logistic regression.
    # The scaler sits inside the pipeline, so it learns its means from training rows only.
    model = make_pipeline(StandardScaler(), LogisticRegression(max_iter=1000, random_state=SEED))
    model.fit(X_train, y_train)

    metrics = {
        "baseline": evaluate("baseline", y_test, baseline.predict(X_test)),
        "logreg": evaluate("logreg", y_test, model.predict(X_test)),
    }
    print("\n" + classification_report(y_test, model.predict(X_test), digits=3))

    # Predictions: keep the label, the true answer and the model's confidence for every test row.
    malignant_col = list(model.classes_).index(POSITIVE)
    predictions = pd.DataFrame(
        {
            "row_id": X_test.index,
            "true": y_test.to_numpy(),
            "predicted": model.predict(X_test),
            "p_malignant": model.predict_proba(X_test)[:, malignant_col].round(3),
        }
    )
    predictions.to_csv(RESULTS / "predictions.csv", index=False)
    (RESULTS / "metrics.json").write_text(json.dumps(metrics, indent=2))

    ConfusionMatrixDisplay.from_estimator(
        model, X_test, y_test, labels=["malignant", "benign"], cmap="Blues"
    )
    plt.title("Logistic regression, test set")
    plt.savefig(RESULTS / "confusion_matrix.png", dpi=120, bbox_inches="tight")
    print("saved results/metrics.json, results/predictions.csv, results/confusion_matrix.png")


if __name__ == "__main__":
    main()
```

A few lines deserve a comment:

- `zero_division=0` in the precision call: the baseline never predicts malignant, so its precision is 0 / 0. Without this argument, scikit-learn warns and uses 0 anyway. With it, the choice is explicit.
- `model.classes_` lists the classes in the order `predict_proba` uses (alphabetical here: benign, malignant). Looking up the column by name is safer than assuming it is column 1.
- `labels=["malignant", "benign"]` puts the positive class first in the confusion matrix, so the top-left cell is TP, matching section 4.6.
- Logistic regression is a classification model despite its name. It learns one weight per feature and outputs a probability between 0 and 1. Day 26 explains how.

Run `python src/train.py`. Expected output:

```text
train rows: 426, test rows: 143

baseline   {'accuracy': 0.629, 'precision_malignant': 0.0, 'recall_malignant': 0.0}
logreg     {'accuracy': 0.965, 'precision_malignant': 0.98, 'recall_malignant': 0.925}

              precision    recall  f1-score   support

      benign      0.957     0.989     0.973        90
   malignant      0.980     0.925     0.951        53

    accuracy                          0.965       143
   macro avg      0.968     0.957     0.962       143
weighted avg      0.966     0.965     0.965       143

saved results/metrics.json, results/predictions.csv, results/confusion_matrix.png
```

Open `results/confusion_matrix.png`. The top row should read 49 and 4, and the bottom row 1 and 89. Run `python src/train.py` a second time and confirm that `results/metrics.json` is unchanged. That is your reproducibility check.

### Step 5: inspect the errors (`src/inspect_errors.py`)

```python
"""Stage 8: read the mistakes one by one."""

import pandas as pd

from data import load_dataframe

COLUMNS = ["mean radius", "mean concave points", "worst area", "worst concave points"]


def main():
    predictions = pd.read_csv("results/predictions.csv")
    errors = predictions[predictions["true"] != predictions["predicted"]]
    print(f"{len(errors)} wrong out of {len(predictions)} test rows\n")

    # Join the mistakes back to their original features so you can see what the model saw.
    df = load_dataframe()
    detail = errors.merge(df[COLUMNS], left_on="row_id", right_index=True)
    print(detail.sort_values("p_malignant").to_string(index=False))

    print("\nclass averages for the same columns, for comparison:")
    print(df.groupby("diagnosis")[COLUMNS].mean().round(3).to_string())

    # How confident was the model when it was wrong?
    unsure = predictions["p_malignant"].between(0.3, 0.7)
    print(f"\ntest rows with p_malignant between 0.3 and 0.7: {int(unsure.sum())}")
    print(f"errors inside that band: {int((unsure & (predictions['true'] != predictions['predicted'])).sum())}")


if __name__ == "__main__":
    main()
```

Run `python src/inspect_errors.py`. Expected output:

```text
5 wrong out of 143 test rows

 row_id      true predicted  p_malignant  mean radius  mean concave points  worst area  worst concave points
     73 malignant    benign        0.056        13.80              0.05069       812.4               0.13830
    190 malignant    benign        0.177        14.22              0.06618       762.4               0.17720
    215 malignant    benign        0.353        13.86              0.05602       750.1               0.16540
    205 malignant    benign        0.476        15.12              0.04079       989.5               0.12520
    455    benign malignant        0.538        13.38              0.03264       705.6               0.07763

class averages for the same columns, for comparison:
           mean radius  mean concave points  worst area  worst concave points
diagnosis                                                                    
benign          12.147                0.026     558.899                 0.074
malignant       17.463                0.088    1422.286                 0.182

test rows with p_malignant between 0.3 and 0.7: 4
errors inside that band: 3
```

Read the table row by row. All four missed cancers are smaller than the average malignant tumor: radius 13.8 to 15.1 against an average of 17.5, and worst area 750 to 990 against 1,422. Their concave-points values sit between the two class averages, so they look more like benign tumors. That is a hypothesis for the report: **the model struggles with small malignant tumors**. Only 4 test rows fall in the "unsure" band of 0.3 to 0.7, and 3 of them are errors. So the model's uncertainty carries information. Row 73 is the most serious case: a cancer scored at 5.6% probability.

### Step 6 (stretch): change the decision threshold (`src/thresholds.py`)

The model predicts malignant when `p_malignant` is at or above 0.5. For screening, you might accept more false alarms to miss fewer cancers.

```python
"""Stretch: what happens if we flag a tumor as malignant at a lower probability?"""

import pandas as pd

predictions = pd.read_csv("results/predictions.csv")
is_malignant = predictions["true"] == "malignant"

print(f"{'threshold':>9} {'caught':>7} {'missed':>7} {'false alarms':>13}")
for threshold in [0.5, 0.3, 0.1]:
    flagged = predictions["p_malignant"] >= threshold
    caught = int((flagged & is_malignant).sum())
    missed = int((~flagged & is_malignant).sum())
    false_alarms = int((flagged & ~is_malignant).sum())
    print(f"{threshold:>9} {caught:>7} {missed:>7} {false_alarms:>13}")
```

Expected output:

```text
threshold  caught  missed  false alarms
      0.5      49       4             1
      0.3      51       2             1
      0.1      52       1             6
```

Lowering the threshold to 0.3 catches 2 more cancers with no extra false alarms on this test set. Going to 0.1 catches one more but adds 5 false alarms.

**Caveat:** this threshold was picked by looking at the test set, which is the Day 3 mistake. To choose a threshold properly, pick it on a validation set (or with cross-validation on the training rows), then report its test result once. Put this caveat in your report.

### Step 7: write `REPORT.md`

Use this structure. Every number must come from your `results/` files or script outputs.

```markdown
# Experiment report: logistic regression on Breast Cancer Wisconsin (Diagnostic)

## Question
Can a simple linear model flag malignant tumors from 30 nucleus measurements,
and how many cancers does it miss?

## Data
- Source: scikit-learn `load_breast_cancer` (UCI Breast Cancer Wisconsin Diagnostic)
- 569 rows, 30 numeric features, no missing values, no duplicates
- 357 benign (62.7%), 212 malignant (37.3%)
- Label encoding in the original data: 0 = malignant, 1 = benign (converted to text on load)

## Method
- Stratified 75/25 train/test split, seed 42 (426 train, 143 test)
- Baseline: always predict the most frequent class
- Model: StandardScaler + LogisticRegression (max_iter=1000), in one pipeline
- Positive class for metrics: malignant

## Results (test set)
| Model | Accuracy | Precision (malignant) | Recall (malignant) |
|---|---|---|---|
| Baseline | 0.629 | 0.000 | 0.000 |
| Logistic regression | 0.965 | 0.980 | 0.925 |

Confusion matrix: TP 49, FN 4, FP 1, TN 89. See results/confusion_matrix.png.

## Error analysis
- 5 errors: 4 missed cancers, 1 false alarm
- Missed cancers are small for malignant tumors (radius 13.8 to 15.1 vs class mean 17.5)
- 3 of 5 errors had p_malignant between 0.3 and 0.7; row 73 was confidently wrong (0.056)

## Limitations
- 143 test rows: one row is about 0.7 points of accuracy; results are noisy
- One dataset from one institution in the early 1990s; no evidence it generalizes elsewhere
- Threshold exploration was done on the test set (biased)
- Not a medical device; not validated for any clinical use

## Next steps
- Cross-validation to measure the spread of results across splits
- Choose the threshold on validation data
- Try a tree-based model and compare errors (Module 4)
```

### Step 8: write the README

The README is for someone who wants to run your code. Include a two-sentence description, the setup commands, the run commands in order, and where the results land:

```bash
python -m pip install -r requirements.txt
python src/inspect_data.py
python src/train.py
python src/inspect_errors.py
python src/thresholds.py   # optional stretch
```

Then add a short "Environment" section (paste the `env_check.py` output, or at least your Python version) and a link to `REPORT.md`.

### Done when

- A fresh clone runs end to end with the commands in your README, and `results/metrics.json` matches the expected numbers (or differs only because of library versions, which you have noted).
- Running `python src/train.py` twice produces an identical `results/metrics.json`.
- `REPORT.md` has all seven sections, and every number in it can be traced to a file in `results/` or a script output.
- The report names at least one limitation you found yourself, beyond the ones in the template.
- You can explain, without notes, why the baseline has 0.629 accuracy and 0.0 recall, and why that makes accuracy the wrong headline metric here.

---

## 7. Check your understanding

1. Your model has 0.965 accuracy. A classmate's has 0.972 on the same 143-row test set. Is theirs better? What would you need to decide?
2. Why does the scaler live inside the pipeline instead of being applied to `X` before `split`?
3. You forgot that 0 means malignant and computed recall for class 1. What number would you have reported, and why is it misleading?
4. Row 73 was malignant with `p_malignant = 0.056`. Why is a confidently wrong prediction more worrying than an unsure one?
5. Lowering the threshold to 0.3 looked free on the test set. Why can you not just adopt 0.3 and report the improved recall?
6. Give two reasons this model should not be used on patients today, even with 0.98 precision.
7. Name one piece of evidence in your repo that would convince a reviewer your numbers are reproducible.

**Answers (check after you try):**

1. Not necessarily. 0.007 is one test row. You would need repeated splits or cross-validation to see whether the difference is larger than the noise. You should also compare recall on malignant, the metric that matters here, not accuracy.
2. Fitting the scaler on all rows lets test-set statistics influence training (a leak). Inside the pipeline, the scaler is fit on training rows only, and the same fitted scaler is applied to test rows and future inputs.
3. You would have reported benign recall (0.989). That sounds better and says nothing about how many cancers were missed, which was the question.
4. An unsure prediction can be routed to a human or a second test. A confident wrong one looks trustworthy and gets acted on, so the patient may not receive further review.
5. You chose 0.3 by looking at the test results, so the improved recall is optimistic. That is the same selection bias as picking the best of many models on the test set. Choose the threshold on validation data and test it once.
6. Any two of these: the data comes from one institution and era and may not match today's patients or equipment; 143 test rows is too few to estimate rare failure rates; the model has not gone through clinical validation or regulatory approval; the features come from a specific image-processing method that a new hospital may not reproduce exactly.
7. A pinned `requirements.txt`, a fixed seed in `data.py`, a committed `results/metrics.json` that a rerun reproduces exactly, and a README listing the exact run commands.

---

## 8. Sources to read and watch

**Read (reference):**
- [scikit-learn: load_breast_cancer](https://scikit-learn.org/stable/modules/generated/sklearn.datasets.load_breast_cancer.html), and the dataset description in the [toy datasets page](https://scikit-learn.org/stable/datasets/toy_dataset.html)
- UCI Machine Learning Repository: search "Breast Cancer Wisconsin (Diagnostic)" on archive.ics.uci.edu for the original dataset page
- [Wikipedia: Confusion matrix](https://en.wikipedia.org/wiki/Confusion_matrix)
- [Wikipedia: Precision and recall](https://en.wikipedia.org/wiki/Precision_and_recall)
- [Wikipedia: Sensitivity and specificity](https://en.wikipedia.org/wiki/Sensitivity_and_specificity)
- [Wikipedia: Logistic regression](https://en.wikipedia.org/wiki/Logistic_regression) (read the introduction only; Day 26 covers the rest)
- [scikit-learn: Metrics and scoring](https://scikit-learn.org/stable/modules/model_evaluation.html)
- [scikit-learn: Pipelines and composite estimators](https://scikit-learn.org/stable/modules/compose.html)
- "Model Cards for Model Reporting," Mitchell et al., 2019. Search the title on arxiv.org.

**Tutorials:**
- GeeksforGeeks: search "Confusion Matrix in Machine Learning" and "Precision and Recall in Machine Learning" on geeksforgeeks.org.
- Google Machine Learning Crash Course: the "Classification" module, especially the pages on thresholds, the confusion matrix, and accuracy, precision and recall, at [developers.google.com/machine-learning/crash-course](https://developers.google.com/machine-learning/crash-course).

**Watch:**
- StatQuest with Josh Starmer: "Machine Learning Fundamentals: The Confusion Matrix" and "Machine Learning Fundamentals: Sensitivity and Specificity." Search the titles on the [StatQuest channel](https://www.youtube.com/@statquest).
- StatQuest: "Logistic Regression" (the first video in the series), for intuition before Day 26.

**Practice:**
- [Kaggle: Titanic, Machine Learning from Disaster](https://www.kaggle.com/competitions/titanic): repeat today's nine steps on a dataset that needs real cleaning (missing ages, text columns). Submit one prediction file and compare your score with your baseline.

---

## 9. Tomorrow

Day 7 begins Module 2, **Mathematics for AI**, with **Mathematical Foundations**: functions, variables, equations, exponents, logarithms, summation notation, function composition and graphs. Every piece of math you used this week without proof lives there. The MSE on Day 2 is a summation. The gradient descent update is a function applied over and over. Logistic regression, which produced today's `p_malignant`, is a composition of two functions: a weighted sum, then a squashing function built from an exponent. Tomorrow you start making those ideas precise, so that by Day 12 you can derive the gradient you coded by hand on Day 2 instead of copying it.

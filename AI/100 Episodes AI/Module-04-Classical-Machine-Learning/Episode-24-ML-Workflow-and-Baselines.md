# Day 24: ML Workflow and Baselines

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3 hours
**Level:** beginner to intermediate. You can write Python functions and load a CSV file. Today is about habits, so the code is short but the reasoning is the point.

**By the end of this day, you will have:**

- A clear mental model of the supervised learning workflow, from raw data to a model you can defend
- A split of a real dataset into train, validation and test sets that you can explain line by line
- Two baseline models and a report that shows whether a fancier model is worth its cost
- A cross-validation run that reports a mean score and its spread, not a single lucky number

**What you need:**

- Python 3.11 or newer
- scikit-learn 1.5 or newer (`pip install scikit-learn`), free and open source under the BSD 3-Clause license
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- Matplotlib 3.8 or newer (`pip install matplotlib`), optional, for one plot
- A code editor such as VS Code (free) or Jupyter (free)
- Git and a free GitHub account for the project section
- The Adult Income dataset from the UCI Machine Learning Repository, or any tabular CSV with a yes-or-no target column

**Prerequisites:**

- Day 7: summation notation and the idea of a function
- Day 8 and Day 9: vectors and matrices at the level of shapes and dot products
- Module 2 statistics: the mean, the variance and the idea of a random sample
- Comfort with Python lists, dictionaries and functions

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Baseline models](#baseline-models)
- [Train, validation and test splits](#train-validation-and-test-splits)
- [Generalization](#generalization)
- [Underfitting and overfitting](#underfitting-and-overfitting)
- [Bias and variance](#bias-and-variance)
- [Cross-validation](#cross-validation)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Baseline-first classifier workflow](#project-baseline-first-classifier-workflow)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most machine learning failures in industry come from the workflow around the algorithm. A model is trained on data that already contained the answer, it is scored on the same rows it learned from, or it is compared with nothing at all. The result looks impressive in a notebook and fails in production within a week.

Today you learn the order of operations that avoids those failures. You will define a question, split the data before you look at it too closely, build the simplest model that could possibly work, and only then try anything more complicated. The discipline is more valuable than any single algorithm, and it applies to every model you will meet from Day 25 onward.

On the job, a data scientist or ML engineer is judged on whether a model beats a sensible baseline on data it has never seen, and whether the team can reproduce that claim next month. Those two things depend on the workflow you practice today.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1950s | Arthur Samuel's checkers program learns from play, and the term "machine learning" spreads through the field. | Early learning systems were judged against simple rules and against human play. |
| 1967 | Thomas Cover and Peter Hart publish the nearest neighbor rule with a proof of its error bounds. | A simple method, used today as a baseline, that set the standard for a theory of generalization. |
| 1970s to 1980s | Vladimir Vapnik and colleagues develop statistical learning theory, which formalizes generalization from finite samples. | The vocabulary of training error, test error and capacity comes from this period. |
| 2000s | Cross-validation and held-out test sets become standard in competitions and in published work. | The habit of separating data for tuning and for final scoring is now expected in any serious report. |
| 2010s to today | Open libraries such as scikit-learn make a baseline a one-line call. | The cheapest step in the workflow is also the one most often skipped. |

**Takeaway:** the methods change, but the discipline of splitting data and comparing against a baseline has stayed the same for decades.

---

## Glossary

- **Supervised learning**: learning a mapping from inputs to known outputs (labels) from example pairs.
- **Feature**: one measured input column, such as age or income.
- **Label or target**: the output the model should predict, such as "bought" or "did not buy".
- **Baseline**: the simplest reasonable prediction method, used as the bar that a real model must clear.
- **Training set**: the rows the model fits its parameters to.
- **Validation set**: rows held out during development to compare models and choose settings.
- **Test set**: rows held out until the very end, used once to report the final score.
- **Generalization**: how well a model performs on new data drawn from the same process as the training data.
- **Underfitting**: a model too simple to capture the real pattern, so it is poor on both training and new data.
- **Overfitting**: a model that memorizes noise in the training data, so it is good on training data and poor on new data.
- **Bias**: error from wrong assumptions in the model, which makes it systematically miss the pattern.
- **Variance**: error from sensitivity to the particular training sample, which makes predictions change a lot when the data changes.
- **Cross-validation**: a method that splits the training data into several folds, trains on some and scores on the rest, then averages the results.
- **Data leakage**: information from outside the training data, often from the test set or from the future, that makes a score look better than it will be in production.

---

### Baseline models

Think of a new cook who has to make a dish that customers will like. Before inventing a recipe, the cook serves plain rice. If the fancy dish does not beat plain rice, the effort is wasted. A baseline plays the role of the plain rice. It is cheap, easy to explain, and hard to beat by accident.

From first principles, a baseline is a function that maps any input to a prediction without using the input in a sophisticated way. For classification, the two classic baselines are predicting the most frequent class and predicting at random in proportion to the class frequencies. For regression, the baseline is predicting the mean of the training targets. Scikit-learn provides both as `DummyClassifier` and `DummyRegressor`.

```python
from sklearn.dummy import DummyClassifier
from sklearn.metrics import accuracy_score

# y_train and y_test are arrays of 0 and 1
baseline = DummyClassifier(strategy="most_frequent")
baseline.fit(X_train, y_train)
pred = baseline.predict(X_test)
print("baseline accuracy:", accuracy_score(y_test, pred))
```

If 76 percent of rows belong to class 0, this baseline gets 76 percent accuracy without learning anything. That number is the bar. A model with 78 percent accuracy looks like a small gain, but its precision on the minority class may be worse than the baseline, which is why Day 34 introduces metrics beyond accuracy.

A common mistake is skipping the baseline because the real model "obviously" works. The fix is to write the baseline score in the first line of your report. Another mistake is choosing a weak baseline, such as a model with no features at all, and then claiming a large improvement. Use a baseline that a competent colleague would accept as fair. In the market, teams at companies such as banks and retailers routinely compare a new scorecard to the existing rule-based process, which is the baseline in practice.

The limitation is that a dummy model says nothing about the difficulty of the problem. If the majority class dominates, accuracy is misleading, so you need several baselines and several metrics.

### Train, validation and test splits

Picture a student preparing for an exam. Practice questions are for learning, mock exams are for choosing a study method, and the real exam is the only honest measure of what the student knows. Using the real exam to choose the study method ruins its value as a measurement. Machine learning splits data for the same reason.

From first principles, you divide your rows into three disjoint groups. The training set fits the parameters. The validation set compares candidate models and settings. The test set gives the final score and must be touched only once. A common split is 70, 15 and 15 percent, though the right numbers depend on how much data you have. Rows must be assigned randomly with a fixed seed, and if the data has a time order or groups such as the same patient appearing many times, the split must respect that structure.

```python
from sklearn.model_selection import train_test_split

X_temp, X_test, y_temp, y_test = train_test_split(
    X, y, test_size=0.15, stratify=y, random_state=42
)
X_train, X_val, y_train, y_val = train_test_split(
    X_temp, y_temp, test_size=0.1765, stratify=y_temp, random_state=42
)
# 0.1765 of the remaining 85 percent is about 15 percent of the whole
```

The `stratify` argument keeps the class proportions the same in each part, which matters when the positive class is rare. The mistake to watch for is fitting a scaler, an imputer or a feature selector on the full dataset before splitting. Those steps learn statistics such as the mean, and if the mean includes validation or test rows, the model has seen information it should not have. Wrap every fitted step in a pipeline so it is learned only on the training part. A limitation of the three-way split is that a small dataset leaves each part too small to trust, which is the reason cross-validation exists.

In industry, time-based splits are common in fraud detection and demand forecasting. You train on last year and validate on the most recent months, because a random split would let the model peek at the future.

### Generalization

Generalization is the question you actually care about. A student who memorized last year's exam answers will do well on last year's paper and poorly on this year's. What matters is performance on the next exam, which is drawn from the same subject but contains new questions.

From first principles, if the training rows and future rows come from the same distribution, then the gap between training error and error on new data measures how much the model has learned that transfers. The gap is the generalization error. Keeping the test set separate is the only reliable way to estimate it, because training error is always optimistic. A model that scores 99 percent on its training rows and 70 percent on validation rows has a large gap, and the fix is usually more data, a simpler model or regularization, which Day 25 introduces.

```python
from sklearn.metrics import accuracy_score

train_acc = accuracy_score(y_train, model.predict(X_train))
val_acc = accuracy_score(y_val, model.predict(X_val))
print(f"train {train_acc:.3f}  validation {val_acc:.3f}  gap {train_acc - val_acc:.3f}")
```

A common mistake is reporting the training score as if it were the expected performance. Another is reading a small validation gap as proof of generalization when the validation set was used to choose the model many times. Each reuse of the validation set leaks a little information into your choices. The limitation is that generalization is only estimated, and it holds only as long as the future looks like the past. In the market, a churn model that generalized well last year can fail after a pricing change, and no split can detect a shift that has not happened yet.

### Underfitting and overfitting

Imagine fitting a curve through points that roughly follow a bend. A straight line misses the bend, which is underfitting. A curve that passes through every point, including the random jitter in the measurements, is overfitting. The right curve follows the bend and ignores the jitter.

From first principles, model complexity is the number of ways a model can adapt to the data, such as the number of parameters or the depth of a tree. Underfitting happens when complexity is too low to represent the pattern, so both training and validation errors stay high. Overfitting happens when complexity is high enough to memorize noise, so training error falls while validation error rises after a point. Plotting both errors against complexity shows the sweet spot between them.

```python
import numpy as np
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures
from sklearn.pipeline import make_pipeline

rng = np.random.default_rng(0)
x = np.sort(rng.uniform(-3, 3, 40)).reshape(-1, 1)
y = np.sin(x).ravel() + rng.normal(0, 0.2, 40)

for degree in (1, 3, 12):
    model = make_pipeline(PolynomialFeatures(degree), LinearRegression())
    model.fit(x, y)
    print(degree, round(model.score(x, y), 3))
# training R-squared rises with degree, but it is not a measure of quality
```

The printed training score rises with degree, which is exactly the trap. The mistake is judging a model by the number that keeps going up. The check is to compare against validation error and to look for the point where it stops falling. The limitation is that the threshold between underfitting and overfitting depends on the data size and on the noise level, so no single rule works across problems. In industry, a team that deploys a 200-feature gradient boosted model to predict a rare event often finds it overfits the training period, which is why production monitoring tracks validation performance over time.

### Bias and variance

Consider an archer. If every arrow lands in the same spot but away from the bullseye, the archer has a bias problem: the aim is systematically off. If the arrows scatter widely around the target, the archer has a variance problem: the shots are inconsistent. A good archer is both accurate on average and consistent.

From first principles, for a prediction the expected squared error can be split into three parts: bias squared, variance and irreducible noise. Bias is how far the average prediction across possible training sets is from the truth. Variance is how much predictions change from one training set to another. Noise is the randomness in the target that no model can remove. Simple models such as a straight line have high bias and low variance. Flexible models such as deep trees have low bias and high variance. The total error is lowest at a middle level of flexibility.

```python
# Bias-variance intuition in code: refit on many bootstrap samples
import numpy as np
from sklearn.tree import DecisionTreeRegressor

rng = np.random.default_rng(1)
x_grid = np.linspace(-3, 3, 100).reshape(-1, 1)
preds = []
for _ in range(50):
    idx = rng.integers(0, 40, 40)
    x = np.linspace(-3, 3, 40).reshape(-1, 1)[idx]
    y = np.sin(x).ravel() + rng.normal(0, 0.2, 40)
    preds.append(DecisionTreeRegressor(max_depth=8).fit(x, y).predict(x_grid))
preds = np.array(preds)
print("variance across refits:", preds.var(axis=0).mean().round(4))
```

The mistake is treating bias and variance as properties of the algorithm alone. They depend on the algorithm, the data size and the features together. Adding more training rows usually reduces variance and leaves bias unchanged, while adding features may reduce bias and raise variance. The limitation is that the decomposition is clean for squared error and becomes approximate for other losses. In practice, engineers diagnose the problem by comparing training and validation errors rather than by computing the decomposition. In the market, a credit model that swings a lot when retrained on a new quarter is showing high variance, and the response is usually regularization or more stable features.

### Cross-validation

A single validation split can be unlucky. If the rows that landed in validation happen to be easy, the score is too high, and if they are hard, it is too low. Cross-validation reduces that luck by repeating the split.

From first principles, k-fold cross-validation divides the training data into k equal folds. For each fold, it trains on the other k minus 1 folds and scores on the held-out fold. The k scores are averaged and their standard deviation is reported. Five folds is a common default. The mean estimates the expected performance, and the spread tells you how stable that estimate is. For classification with rare classes, stratified folds keep the class balance in each fold.

```python
from sklearn.model_selection import StratifiedKFold, cross_val_score
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
model = make_pipeline(StandardScaler(), LogisticRegression(max_iter=1000))
scores = cross_val_score(model, X_train, y_train, cv=cv, scoring="roc_auc")
print(f"ROC AUC {scores.mean():.3f} +/- {scores.std():.3f}")
```

Notice that the scaler sits inside the pipeline. Scikit-learn refits it on each training fold, which prevents leakage. The mistake to avoid is scaling or selecting features on the whole training set before calling `cross_val_score`, which inflates the score. Another is using cross-validation on time-ordered data without a time-aware splitter such as `TimeSeriesSplit`. The limitation is cost: five folds means five trainings, which matters for large models, and cross-validation does not replace the final test set. In industry, teams use cross-validation for model selection during development and keep a sealed test set for the report that goes to management.

---

## Hands-on exercises

Work through these in order, in one notebook or script, and keep the output of each step.

1. Load a tabular dataset with pandas and print the shape, the column types and the class balance of the target. Write down the majority-class share in one sentence.
2. Split the data into train, validation and test sets with stratification and a fixed random seed. Print the class balance of each part and confirm it is close to the full-dataset balance.
3. Fit `DummyClassifier(strategy="most_frequent")` and `DummyClassifier(strategy="stratified")`. Record the accuracy and the ROC AUC of each on the validation set. The ROC AUC of a constant predictor is 0.5, so explain why the most-frequent baseline's AUC is 0.5.
4. Fit a logistic regression pipeline with a scaler, using the training set only. Record training and validation accuracy and the gap.
5. Run 5-fold stratified cross-validation on the training set with the same pipeline. Report the mean and standard deviation of the scores. Then compare the mean with the single validation score from step 4.
6. Deliberately leak information by fitting the scaler on the full dataset before splitting, then rerun step 5. Record how much the cross-validation score changes and write one sentence explaining why.
7. Write a one-paragraph summary that states the baseline score, the model score on the held-out test set (computed once), and whether the model clears the baseline by a margin larger than the cross-validation standard deviation.

Keep a short table of each number. The table is evidence that you followed the workflow, which the code alone does not show.

---

## Project: Baseline-first classifier workflow

This is a practical, medium-weight project. The code is short. The main deliverable is a workflow you can rerun from a clean clone, with a report that compares a baseline against a model using cross-validation and a sealed test set.

**Repo name:** `100-days-ai-engineering-day-24-ml-workflow-and-baselines`

**Goal:** build a reproducible script that splits data, reports baselines, trains one pipeline model with cross-validation, and scores the held-out test set exactly once.

**Repo layout:**

```
100-days-ai-engineering-day-24-ml-workflow-and-baselines/
├── README.md
├── requirements.txt
├── data/
│   └── .gitkeep
├── src/
│   ├── __init__.py
│   ├── data.py
│   └── workflow.py
├── reports/
│   └── results.md
└── tests/
    └── test_workflow.py
```

**Starter code for `src/data.py`:**

```python
import pandas as pd


def load_csv(path: str, target: str):
    """Return (X, y) from a CSV file. The target column is binary (0 or 1)."""
    df = pd.read_csv(path)
    y = df[target]
    X = df.drop(columns=[target])
    return X, y
```

**Starter code for `src/workflow.py`:**

```python
from sklearn.compose import make_column_selector, make_column_transformer
from sklearn.dummy import DummyClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score
from sklearn.model_selection import StratifiedKFold, cross_val_score, train_test_split
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler


def build_preprocessor():
    """Scale numeric columns and one-hot encode text columns."""
    return make_column_transformer(
        (StandardScaler(), make_column_selector(dtype_include="number")),
        (OneHotEncoder(handle_unknown="ignore"), make_column_selector(dtype_include="object")),
    )


def split(X, y, seed=42):
    """Return train, validation and test parts, stratified, with the test set held out."""
    raise NotImplementedError


def run(X, y, seed=42):
    """Return a dict with baseline validation AUC, model CV mean and std, and test AUC.
    The test set must be scored once, at the end.
    """
    raise NotImplementedError
```

**Starter code for `tests/test_workflow.py`:**

```python
import numpy as np
import pandas as pd

from src.workflow import split


def make_frame(n=300):
    rng = np.random.default_rng(0)
    return pd.DataFrame({
        "a": rng.normal(size=n),
        "b": rng.normal(size=n),
        "y": (rng.random(n) < 0.3).astype(int),
    })


def test_split_sizes_add_up():
    df = make_frame()
    X, y = df[["a", "b"]], df["y"]
    X_tr, X_va, X_te, y_tr, y_va, y_te = split(X, y)
    assert len(X_tr) + len(X_va) + len(X_te) == len(df)


def test_split_keeps_class_balance():
    df = make_frame()
    X, y = df[["a", "b"]], df["y"]
    _, _, _, y_tr, _, y_te = split(X, y)
    assert abs(y_tr.mean() - y.mean()) < 0.03
    assert abs(y_te.mean() - y.mean()) < 0.05
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `pandas`, `numpy`, `scikit-learn` and `pytest`.
2. Find a binary-classification CSV with at least 1,000 rows. The UCI Adult Income dataset works well. Place it under `data/`, and do not commit files over a few megabytes. Add the path to `.gitignore` if needed.
3. Implement `split` with two calls to `train_test_split`, stratified, with the seed. Make the test tests pass.
4. Implement `run`. Score the two dummy baselines on the validation part. Build the model pipeline with `build_preprocessor` and `LogisticRegression(max_iter=1000)`, run 5-fold stratified cross-validation on the training part, and then fit on the training part and score validation.
5. Score the test set once, after you have chosen the final configuration. Write the number to `reports/results.md` and do not change the configuration afterwards.
6. Run `pytest` until the tests pass, then run the full script from a clean terminal to confirm it reproduces the same numbers.
7. Write the README. Explain the split sizes, the baseline score, the model score, and whether the difference is larger than the cross-validation standard deviation.
8. Commit the code, the tests, the report and the README, then push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pip install -r requirements.txt` and `pytest`
- [ ] The dataset source and license are named, with a link to where it was downloaded
- [ ] The split sizes and class balance of each part are reported in a table
- [ ] The baseline and model scores are both reported, with cross-validation mean and standard deviation
- [ ] The README says in one line whether the test score was computed once

**Done when:**

- `pytest` passes all tests
- The script runs from a clean clone and prints the same numbers twice in a row
- The test score appears once in the report, and the configuration was fixed before it was computed
- The model clears the majority-class baseline by more than the cross-validation standard deviation, or the README explains honestly why it does not

**What to commit as evidence:** the source files, the tests, the results file, the README with the tables, and one commit message that names the seed used.

This project is a good place to practice the discipline. A correct workflow with a modest model is worth more to an employer than a leaky workflow with an impressive number.

---

## Check your understanding

1. A colleague reports 94 percent accuracy on a fraud dataset where 93 percent of rows are legitimate. Explain why this number may tell you little, and name one baseline and one metric you would ask for instead.
2. You scale your features using the mean of the full dataset, then split into train and test. Describe the leak, and say how the pipeline pattern prevents it.
3. Your model scores 98 percent on training data and 71 percent on validation data. Diagnose the problem using the words bias and variance, and suggest one change that targets the variance.
4. Two models have the same mean cross-validation score, but one has a standard deviation of 0.01 and the other 0.08. Which would you prefer for deployment, and why?
5. A time-series dataset of daily sales is split randomly into train and test. Explain what the test score measures and why it may be too optimistic.
6. Your validation score has improved over fifteen experiments, and you now want to report it to a client. Explain why the validation score is no longer an unbiased estimate, and what should be used instead.
7. A teammate says the training error should go to zero for a good model. Explain why that is wrong, and describe the pattern you would expect to see in a model that has learned real structure.

### Answers

1. On a dataset where 93 percent of rows are legitimate, a model that always predicts legitimate scores 93 percent accuracy. The 94 percent figure is barely better than that. Ask for the majority-class baseline score and for precision and recall on the fraud class, or for the ROC AUC, which does not depend on the class balance in the same way.
2. The scaler learns the mean and variance from test rows, so the training features carry information about the test set. The leak can inflate the score. A pipeline fits the scaler only inside each training fold or on the training set, and applies the learned values to the test set unchanged.
3. The gap between 98 and 71 percent is a large gap, which points to high variance: the model fits the particular training sample too closely. One change that targets variance is regularization, such as a smaller C value in logistic regression, or simply more training data.
4. The one with the standard deviation of 0.01 is the better choice, because its score is stable across folds. A large spread means the estimate depends heavily on which rows were chosen, so the deployed model may behave unpredictably on new data.
5. The random split lets the model see future patterns while it trains, so the test rows are not truly new. The test score measures interpolation within the observed period, which is usually easier than forecasting. A time-aware split is more honest.
6. The validation set has been used to choose among fifteen configurations, so the best score is partly the result of selection. The unbiased estimate comes from a test set that was not used in any choice, scored once at the end.
7. A training error of zero usually signals memorization, not learning, especially for flexible models with noisy data. A model that has learned real structure shows a training error that is low but not zero, and a validation error that is close to it.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Supervised learning](https://en.wikipedia.org/wiki/Supervised_learning)
- Wikipedia: [Overfitting](https://en.wikipedia.org/wiki/Overfitting)
- Wikipedia: [Bias–variance tradeoff](https://en.wikipedia.org/wiki/Bias%E2%80%93variance_tradeoff)
- Wikipedia: [Cross-validation (statistics)](https://en.wikipedia.org/wiki/Cross-validation_(statistics))
- Wikipedia: [Leakage (machine learning)](https://en.wikipedia.org/wiki/Leakage_(machine_learning))
- scikit-learn user guide, "Cross-validation: evaluating estimator performance" and "Common pitfalls and recommended practices", at scikit-learn.org under the stable user guide.
- James, Witten, Hastie and Tibshirani, *An Introduction to Statistical Learning*, the chapter on resampling. A free PDF is linked from statlearning.com.

**Tutorials**

- scikit-learn documentation, "Dummy estimators" and "Pipelines and composite estimators", for the exact API used today.
- GeeksforGeeks, search "train test validation split in machine learning" for a worked example with a table.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Machine Learning Fundamentals" playlist, the videos on bias and variance and on cross-validation.
- Google Machine Learning Crash Course, the sections on generalization and on the training and test split, at developers.google.com/machine-learning/crash-course.

**Practice**

- Kaggle Learn, the free "Intro to Machine Learning" course, which has a notebook on model validation and a notebook on cross-validation.

---

## Tomorrow

Tomorrow you build your first real model: linear regression. You will see the straight-line model as a sum of weighted features, measure its errors with the mean squared error, derive the least-squares solution, and then train it by gradient descent. You will also meet regularization, the first tool you use to reduce the variance you diagnosed today. Bring the baseline table from this project, because tomorrow's model has to beat it.

**Next:** [Day 25: Linear Regression](Day-25-Linear-Regression.md)

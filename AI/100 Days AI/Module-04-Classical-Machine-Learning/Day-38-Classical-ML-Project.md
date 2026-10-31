# Day 38: Classical ML Project

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 6 hours, spread over two sittings
**Level:** intermediate. This is the capstone for Module 4. You should have finished Days 24 to 37, since every stage of the build uses one of them.

**By the end of this day, you will have:**

- A repository with a data pipeline that validates its input and splits it once, in a fixed and documented way
- A baseline, four candidate models and a logged experiment table with cross-validated results
- A final model selected by validation, a threshold chosen to meet a stated recall target, and one test-set evaluation
- An error-slice report and a short model card that states what the model is for and where it fails
- A command-line inference interface that loads the saved model and predicts from a JSON file

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- joblib 1.4 or newer (`pip install joblib`), BSD 3-Clause, for saving the fitted model
- pytest 8.0 or newer (`pip install pytest`)
- An internet connection the first time you run the script, so that scikit-learn can download the Adult Census Income dataset from OpenML
- Git and a free GitHub account

**Prerequisites:**

- Days 24 to 27: workflow, linear models, logistic regression and nearest neighbors
- Days 28 to 32: naive Bayes, trees, ensembles, SVMs and clustering, at least at the level of knowing when each one fits
- Days 33 to 37: dimensionality reduction, classification and regression metrics, hyperparameter search and error analysis
- Comfort with pipelines and `ColumnTransformer` from Day 36

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Data pipeline](#data-pipeline)
- [Baseline](#baseline)
- [Several models](#several-models)
- [Experiment comparison](#experiment-comparison)
- [Evaluation](#evaluation)
- [Error analysis](#error-analysis)
- [Model selection](#model-selection)
- [Simple inference interface](#simple-inference-interface)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Adult income classifier as a product](#project-adult-income-classifier-as-a-product)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

The earlier days taught you the parts. This day puts them into one product: a repository that takes a raw table, checks it, trains a set of models under the same rules, picks one for stated reasons, reports one honest test score, explains where the model fails, and answers a request for a prediction. A reviewer should be able to clone it, run one command, and get the same numbers you report.

The project is deliberately modest. You will predict whether a person earns more than 50,000 dollars a year from census-style features, which is a real public-sector and marketing question with clear ethical stakes. The model matters less than the discipline: a split that happens once, a baseline that everything must beat, a validation set that chooses the threshold, a test set that is opened once, and an error analysis that names the groups where the model is weakest.

On the job, most of the value of a classical ML project sits in these habits, and the last few points of accuracy matter less. A team that can rerun the pipeline, explain the choice of model and state its limits will be trusted with the next model. A team that cannot will spend its time explaining why last quarter's number did not reproduce.

---

## Short history

This project does not add a new algorithm, so the history here is about the tools the project relies on.

| Period | What happened | Why it matters today |
|---|---|---|
| 1987 | The UCI Machine Learning Repository is founded at the University of California, Irvine, and becomes a common source of benchmark datasets. | The Adult Census Income dataset was extracted from a 1994 census database and donated to this repository. |
| 2007 | David Cournapeau starts the scikit-learn project as a Google Summer of Code project, building on SciPy. | The pipeline, cross-validation and metrics you use today come from this project's design. |
| 2013 | OpenML is launched as an open platform for sharing datasets and experiments with machine-readable metadata. | Downloading the dataset through a stable identifier makes the project reproducible. |
| 2008 onward | The joblib library, built for efficient persistence of large NumPy-based objects, becomes a standard way to save fitted scikit-learn estimators. | Saving the fitted pipeline as a single file is what makes the inference step possible. |
| 2019 | Mitchell and colleagues publish "Model Cards for Model Reporting", proposing a standard document for a model's intended use, data, metrics and limitations. | The model card you write today follows this proposal. |

**Takeaway:** the parts are mature and shared. The project is about assembling them carefully and reporting them honestly.

---

## Glossary

- **Data pipeline**: the code path from raw data to a validated, split dataset.
- **Schema check**: a test that the input has the expected columns, types and no missing target values.
- **Candidate model**: a model considered for the final choice, with its settings fixed before the comparison.
- **Experiment log**: a table that records each model's cross-validated results, time and settings.
- **Final model**: the one model chosen for deployment, refit on the training and validation data after the choice.
- **Operating threshold**: the probability cutoff used for the final decisions, chosen on validation data.
- **Model card**: a short document stating a model's purpose, data, metrics, slices and limitations.
- **Inference interface**: the code that loads the saved model and returns predictions for new inputs.
- **Contract**: the set of columns and types an inference request must provide.

---

## Data pipeline

A data pipeline is the first thing that goes wrong in most projects, and the first thing a reviewer checks. The rule for this project is that raw data is loaded once, validated against a schema, split once with a fixed seed, and then never reshuffled. Every later step reads from those splits.

From first principles, the pipeline has four stages. Load the raw table from a stable source. Clean what must be cleaned, such as converting categorical columns to a plain type and mapping the target to 0 and 1. Validate the result against the contract: the target exists, has no missing values, and has the expected classes. Split into train, validation and test parts, with stratification on the target. The split is 60, 20 and 20 percent. Keep the test part out of every decision until the final evaluation.

Here is the data module. Its job is to make the rules explicit.

```python
# src/data.py
import numpy as np
import pandas as pd
from sklearn.datasets import fetch_openml
from sklearn.model_selection import train_test_split

TARGET = "income"


def load_adult():
    """Download the Adult Census Income dataset from OpenML and return a clean DataFrame."""
    bunch = fetch_openml(name="adult", version=2, as_frame=True)
    df = bunch.frame.rename(columns={"class": TARGET}).copy()
    for col in df.select_dtypes(include="category").columns:
        df[col] = df[col].astype(object)          # plain objects for the imputers
    df[TARGET] = df[TARGET].astype(str).str.strip().str.startswith(">").astype(int)
    return df


def check_schema(df):
    """Raise ValueError if the frame does not meet the contract."""
    if TARGET not in df.columns:
        raise ValueError(f"missing target column: {TARGET}")
    if df[TARGET].isna().any():
        raise ValueError("target contains missing values")
    if set(df[TARGET].unique()) != {0, 1}:
        raise ValueError("target must contain both classes 0 and 1")
    if len(df) < 1000:
        raise ValueError("dataset is too small for a meaningful split")
    return True


def split_frame(df, seed=42):
    """Return (train, validation, test) with a 60/20/20 stratified split."""
    train_val, test = train_test_split(df, test_size=0.2, stratify=df[TARGET], random_state=seed)
    train, val = train_test_split(train_val, test_size=0.25, stratify=train_val[TARGET], random_state=seed)
    return train, val, test
```

The mistake to avoid is cleaning the whole dataset before splitting, for example by filling in medians from the full table, which leaks information into validation and test. The imputation belongs in the pipeline, as the next section shows. Another mistake is splitting again each time you run the script, which means the numbers change without any change in the model. The fixed seed prevents that. The limitation is that a random split assumes the census-style data is not time-ordered, which is reasonable for this dataset but would be wrong for, say, monthly sales.

---

## Baseline

Every model in this project must beat a baseline, and the baseline is the first row of the experiment table. For a classification task with a 25 percent positive rate, predicting the majority class gives 75 percent accuracy and an ROC AUC of 0.5. Any model that does not clear that bar on the metric that matters has no reason to exist.

The baseline should be trivial, fitted inside the same cross-validation folds, and reported with the same metrics as everything else. Use `DummyClassifier` so the baseline is run by the same machinery as the models.

```python
# src/models.py (the baseline is the first entry in candidate_models)
from sklearn.dummy import DummyClassifier

def candidate_models(seed=0):
    return {"baseline_most_frequent": DummyClassifier(strategy="most_frequent"), ...}
```

The mistake is to compare a model against an invented number, such as "accuracy above 80 percent is good", without checking the base rate. The limitation of a dummy baseline is that it says nothing about how hard the problem is for a simple rule. A useful second baseline is a single-feature model, such as a logistic regression on age alone, which tells you how much the other features add. In the market, a team that skips the baseline ends up arguing about whether 0.84 is good, and the baseline settles it in a line.

---

## Several models

The candidate models should be chosen for different reasons, not at random. You want one linear model, which is explainable and fast. You want one bagged ensemble, which is strong on tabular data and needs little tuning. You want one boosted model, which often wins on tabular data and handles missing values natively. Naive Bayes and the SVM are left out here: naive Bayes needs a separate treatment of the numeric columns to add anything on this mixed-type data, and the SVM is slow on tens of thousands of rows with one-hot columns. That judgment is yours to defend, and you should be able to explain it. Write down the reason for every model you include, and for every model you leave out.

```python
# src/models.py
from sklearn.dummy import DummyClassifier
from sklearn.ensemble import HistGradientBoostingClassifier, RandomForestClassifier
from sklearn.linear_model import LogisticRegression


def candidate_models(seed=0):
    """Return fresh, unfitted estimators. Every call returns new objects with fixed settings."""
    return {
        "baseline_most_frequent": DummyClassifier(strategy="most_frequent"),
        "logistic_regression": LogisticRegression(C=1.0, max_iter=2000),
        "random_forest": RandomForestClassifier(
            n_estimators=300, min_samples_leaf=5, n_jobs=-1, random_state=seed
        ),
        "hist_gradient_boosting": HistGradientBoostingClassifier(
            learning_rate=0.1, max_iter=300, random_state=seed
        ),
    }
```

The mistake is to add models until one wins by chance. Four well-chosen candidates with a fixed setting are better than twenty with a search that you cannot defend. The preprocessing for all of them comes from one shared function, which keeps the comparison fair.

```python
# src/features.py
from sklearn.compose import make_column_selector, make_column_transformer
from sklearn.impute import SimpleImputer
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler


def build_preprocessor():
    """Impute and scale numeric columns; impute and one-hot encode categorical columns."""
    numeric = make_pipeline(SimpleImputer(strategy="median"), StandardScaler())
    categorical = make_pipeline(
        SimpleImputer(strategy="most_frequent"),
        OneHotEncoder(handle_unknown="ignore"),
    )
    return make_column_transformer(
        (numeric, make_column_selector(dtype_include="number")),
        (categorical, make_column_selector(dtype_exclude="number")),
    )
```

The limitation of one-hot encoding is that high-cardinality categories create many columns, and `handle_unknown="ignore"` silently drops categories that were not seen in training. That is the right default for an inference service, but it should be logged, because an unseen category is a signal of data drift. In industry, the most common cause of a silent failure in a deployed tabular model is a category that appeared after training, and the code here at least makes the choice explicit.

---

## Experiment comparison

An experiment comparison is a table with one row per model and the same columns for each: cross-validated ROC AUC with its standard deviation, accuracy and fit time. It is the record that lets someone reproduce your choice, and it is the first thing to inspect if the final model disagrees with expectations.

The comparison uses 5-fold stratified cross-validation on the training part only. The validation part is not used here, so that it stays free for the threshold and the final check. Every model runs through the same pipeline, and the results go to a CSV file that is committed to the repository.

```python
# src/experiment.py
import time
from pathlib import Path

import pandas as pd
from sklearn.model_selection import StratifiedKFold, cross_validate
from sklearn.pipeline import make_pipeline

from src.data import TARGET
from src.features import build_preprocessor
from src.models import candidate_models


def run_experiments(train, out_csv, seed=0):
    """Cross-validate every candidate on the training part and write a CSV log."""
    X = train.drop(columns=[TARGET])
    y = train[TARGET]
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=seed)
    rows = []
    for name, estimator in candidate_models(seed).items():
        pipe = make_pipeline(build_preprocessor(), estimator)
        start = time.perf_counter()
        res = cross_validate(pipe, X, y, cv=cv, scoring=["roc_auc", "accuracy"])
        rows.append({
            "model": name,
            "roc_auc_mean": round(res["test_roc_auc"].mean(), 4),
            "roc_auc_std": round(res["test_roc_auc"].std(), 4),
            "accuracy_mean": round(res["test_accuracy"].mean(), 4),
            "fit_seconds_total": round(time.perf_counter() - start, 1),
        })
    table = pd.DataFrame(rows).sort_values("roc_auc_mean", ascending=False)
    Path(out_csv).parent.mkdir(parents=True, exist_ok=True)
    table.to_csv(out_csv, index=False)
    return table
```

The mistake is to rerun the comparison until the numbers look right and to overwrite the earlier file. Commit the log with the code, and if you change a setting, add a new row instead of replacing the old one. The limitation of a single cross-validation comparison is that differences smaller than the standard deviation across folds are not meaningful, and you should say so in the report.

---

## Evaluation

Evaluation has two parts with different purposes. Choosing the threshold uses the validation part, which the model has not trained on. Reporting the final number uses the test part, which is opened once.

The threshold is chosen from a stated objective. For this project, the objective is a recall target: the team wants to identify at least 80 percent of people who earn above 50,000 dollars, for a targeting campaign, and it accepts the precision that follows. Among thresholds that meet the target on validation data, the code keeps the highest one, which gives the most precise decisions while still meeting the target.

```python
# src/evaluate.py
import numpy as np
import pandas as pd
from sklearn.metrics import (accuracy_score, f1_score, precision_score,
                             recall_score, roc_auc_score)

from src.data import TARGET


def choose_threshold(model, val, min_recall=0.80):
    """Return the highest threshold on the validation part that still reaches min_recall."""
    X = val.drop(columns=[TARGET])
    y = val[TARGET].to_numpy()
    proba = model.predict_proba(X)[:, 1]
    chosen = 0.5                       # fallback if no threshold meets the target
    for t in np.linspace(0.05, 0.95, 91):
        if recall_score(y, proba >= t, zero_division=0) >= min_recall:
            chosen = float(t)
    return chosen


def evaluate_on_test(model, test, threshold):
    """Score the test part once, at the given threshold. Return a dict of metrics."""
    X = test.drop(columns=[TARGET])
    y = test[TARGET].to_numpy()
    proba = model.predict_proba(X)[:, 1]
    pred = (proba >= threshold).astype(int)
    majority = max(y.mean(), 1 - y.mean())
    return {
        "threshold": round(threshold, 3),
        "roc_auc": round(roc_auc_score(y, proba), 4),
        "accuracy": round(accuracy_score(y, pred), 4),
        "precision": round(precision_score(y, pred, zero_division=0), 4),
        "recall": round(recall_score(y, pred, zero_division=0), 4),
        "f1": round(f1_score(y, pred, zero_division=0), 4),
        "majority_baseline_accuracy": round(float(majority), 4),
    }
```

The mistake is to pick the threshold on the test set because it gives a better number there. That is leakage through the choice, as Day 36 showed. The second mistake is to report the validation score as the final result, which is optimistic because the threshold was chosen on it. The limitation of this design is that the final model is refit on training and validation together, so the threshold chosen with the smaller model may shift slightly. The project reports this as a known limitation rather than hiding it, and the README includes the threshold and the test result from the refit model.

---

## Error analysis

The test score says how often the model is right. The error analysis says where it is wrong, which matters for fairness, for the business and for the next round of work. Slice the test part by several sensitive or operationally important columns, such as sex, race, education and marital status, and report the accuracy, the positive rate and the predicted positive rate in each group. Flag small groups, because their numbers are noisy.

The aim is to make the differences visible, so that the team can decide what to do. A large gap in recall between groups is a finding that needs a check on the data, the features and the threshold, not a footnote.

```python
# src/evaluate.py (continued)
def error_slices(model, test, columns, threshold, min_rows=30):
    """Return one row per group in each column, with accuracy and rates, sorted by accuracy."""
    X = test.drop(columns=[TARGET])
    y = test[TARGET].to_numpy()
    pred = (model.predict_proba(X)[:, 1] >= threshold).astype(int)
    frame = test[columns].copy()
    frame["_correct"] = (pred == y).astype(int)
    frame["_y"] = y
    frame["_pred"] = pred
    rows = []
    for col in columns:
        for group, sub in frame.groupby(col):
            rows.append({
                "slice": f"{col}={group}",
                "rows": len(sub),
                "accuracy": round(sub["_correct"].mean(), 3),
                "actual_positive_rate": round(sub["_y"].mean(), 3),
                "predicted_positive_rate": round(sub["_pred"].mean(), 3),
                "small": len(sub) < min_rows,
            })
    return pd.DataFrame(rows).sort_values("accuracy").reset_index(drop=True)
```

The mistake is to read slice differences as causal, or to act on a single slice without checking it on other data. The limitation is that slicing one column at a time misses interactions, such as the combination of two groups. A follow-up slice on the combination is a reasonable next step, but it multiplies the number of comparisons, so treat any new pattern as a hypothesis for a second sample. In the market, a credit or hiring model with a gap across a protected group is a regulatory and reputational risk, and the error slice is the first evidence a review will ask for.

---

## Model selection

Model selection is the step that turns the experiment table into one choice. The rule should be written down before the results are seen. For this project the rule is: choose the candidate with the highest mean cross-validated ROC AUC. If the top two candidates are within one standard deviation of each other, the difference is not meaningful, so prefer the simpler model, which has fewer moving parts to explain and maintain, and write that decision in the README. The code makes the first step automatic and leaves the tie-break as a recorded human decision.

Once the model is chosen on training data, fit it on the training part, choose the threshold on validation, refit on training plus validation, and then evaluate on the test part once. The final model is the refit, and the saved metadata must record the seed, the chosen model, the threshold and the test result.

```python
# src/train.py
import json
from pathlib import Path

import joblib
import pandas as pd
from sklearn.pipeline import make_pipeline

from src.data import TARGET, check_schema, load_adult, split_frame
from src.evaluate import choose_threshold, error_slices, evaluate_on_test
from src.experiment import run_experiments
from src.features import build_preprocessor
from src.models import candidate_models

SEED = 42
ROOT = Path(__file__).resolve().parents[1]
SLICE_COLUMNS = ["sex", "race", "education"]


def select_model(table):
    """Pick the best non-baseline model by mean ROC AUC. A near-tie within one standard
    deviation is resolved by hand, and the decision is recorded in the README."""
    contenders = table[table["model"] != "baseline_most_frequent"]
    return contenders.sort_values(["roc_auc_mean"], ascending=False).iloc[0]["model"]


def main():
    df = load_adult()
    check_schema(df)
    train, val, test = split_frame(df, seed=SEED)

    table = run_experiments(train, ROOT / "experiments" / "results.csv", seed=SEED)
    print(table.to_string(index=False))
    best = select_model(table)

    # Fit on training data, choose the threshold on validation data.
    dev = make_pipeline(build_preprocessor(), candidate_models(SEED)[best])
    dev.fit(train.drop(columns=[TARGET]), train[TARGET])
    threshold = choose_threshold(dev, val, min_recall=0.80)

    # Refit on training plus validation with the same settings, then evaluate once on test.
    trainval = pd.concat([train, val])
    final = make_pipeline(build_preprocessor(), candidate_models(SEED)[best])
    final.fit(trainval.drop(columns=[TARGET]), trainval[TARGET])
    metrics = evaluate_on_test(final, test, threshold)
    slices = error_slices(final, test, SLICE_COLUMNS, threshold)

    (ROOT / "models").mkdir(exist_ok=True)
    (ROOT / "reports").mkdir(exist_ok=True)
    joblib.dump(final, ROOT / "models" / "model.joblib")
    contract = [c for c in df.columns if c != TARGET]
    meta = {"model": best, "threshold": threshold, "seed": SEED,
            "columns": contract, "test_metrics": metrics}
    (ROOT / "models" / "metadata.json").write_text(json.dumps(meta, indent=2))
    slices.to_csv(ROOT / "reports" / "error_slices.csv", index=False)
    print(json.dumps(metrics, indent=2))


if __name__ == "__main__":
    main()
```

The mistake is to let the test result change the choice of model after it is seen. If the first test result is disappointing, the honest response is to report it, state what was tried, and decide whether to collect more data, not to rerun the selection with the test set in view. The limitation of this process is that a single held-out test set gives one estimate with its own uncertainty, and a bootstrap interval on the test metrics is a good addition for the model card.

---

## Simple inference interface

A model that cannot be called is not a product. The inference interface loads the saved pipeline and the metadata, checks that an incoming request has the columns the model was trained on, and returns a probability and a label at the saved threshold. The interface is a command-line tool here. A web service can wrap the same function later.

The contract is what keeps the service safe, whichever framework runs it. If a request is missing a column, the service should fail with a clear message rather than guess. If it has an extra column, the pipeline ignores it. Both behaviors are tested.

```python
# src/predict.py
import argparse
import json
from pathlib import Path

import joblib
import pandas as pd

ROOT = Path(__file__).resolve().parents[1]


def load_artifacts(model_dir=ROOT / "models"):
    model = joblib.load(Path(model_dir) / "model.joblib")
    meta = json.loads((Path(model_dir) / "metadata.json").read_text())
    return model, meta


def validate_records(records, columns):
    """Raise ValueError if any record is missing a required column."""
    for i, rec in enumerate(records):
        missing = [c for c in columns if c not in rec]
        if missing:
            raise ValueError(f"record {i} is missing columns: {missing}")


def predict_records(records, model, meta):
    """Return a list of dicts with the probability and the label at the saved threshold."""
    validate_records(records, meta["columns"])
    frame = pd.DataFrame(records)[meta["columns"]]
    proba = model.predict_proba(frame)[:, 1]
    return [{"probability": round(float(p), 4),
             "label": int(p >= meta["threshold"])} for p in proba]


def main():
    parser = argparse.ArgumentParser(description="Predict income above 50K from a JSON file.")
    parser.add_argument("input", help="path to a JSON file containing a list of records")
    args = parser.parse_args()
    records = json.loads(Path(args.input).read_text())
    model, meta = load_artifacts()
    print(json.dumps(predict_records(records, model, meta), indent=2))


if __name__ == "__main__":
    main()
```

The mistake is to load the model once per request in a service, which is slow, or to skip the contract check, which lets a missing column become a silent imputation. The limitation of this interface is that it trusts the input format and has no authentication, rate limiting or logging of requests, so it is a development tool. Add those before you expose it. In industry, the same pattern, a saved artifact and a contract check, is the start of every serving system, and the framework is chosen later.

---

## Hands-on exercises

1. Run `python -m src.train` once, record the experiment table and the test metrics, then run it again and confirm the numbers are identical. Explain why they should be.
2. Add a single-feature logistic regression on `age` to the candidate models as a second baseline. Report whether it is close to the full models, and what that says about the other features.
3. Change the recall target from 0.80 to 0.90 and report how the threshold, precision and test accuracy change. State which target you would defend to the business and why.
4. Write a schema test that fails when the target has only one class, and check that `check_schema` raises the error.
5. Write one new slice, for example the combination of sex and a high-education group, and compare its accuracy with the single-column slices. Mark it as a hypothesis for a second sample.
6. Compute a 95 percent bootstrap interval for the test ROC AUC by resampling the test rows 1,000 times. Report the interval and say whether the difference between the top two models is inside it.
7. Call `predict_records` on three hand-written records, one of them with a missing column, and record the error message. Explain why the message is more useful than a silent imputation.

---

## Project: Adult income classifier as a product

This is the capstone project. It is larger than the earlier projects, and the goal is a repository that a colleague can clone, run and review in an afternoon.

**Repo name:** `100-days-ai-engineering-day-38-classical-ml-project`

**Goal:** build a reproducible classifier for the Adult Census Income dataset with a data contract, a baseline, four candidate models, a logged experiment table, a threshold chosen for a recall target, one test evaluation, an error-slice report and a command-line inference interface.

**Repo layout:**

```
100-days-ai-engineering-day-38-classical-ml-project/
├── README.md
├── MODEL_CARD.md
├── requirements.txt
├── .gitignore
├── src/
│   ├── __init__.py
│   ├── data.py
│   ├── features.py
│   ├── models.py
│   ├── experiment.py
│   ├── evaluate.py
│   ├── train.py
│   └── predict.py
├── experiments/
│   └── results.csv          (generated, committed)
├── reports/
│   └── error_slices.csv     (generated, committed)
├── models/                  (generated, ignored by git)
│   ├── model.joblib
│   └── metadata.json
├── examples/
│   └── sample_request.json
└── tests/
    ├── test_data.py
    └── test_pipeline.py
```

**Starter code for `requirements.txt`:**

```
numpy>=2.0
pandas>=2.2
scikit-learn>=1.5
joblib>=1.4
pytest>=8.0
```

**Starter code for `tests/test_data.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pandas as pd
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from src.data import TARGET, check_schema, split_frame


def make_frame(n=500, seed=0):
    rng = np.random.default_rng(seed)
    return pd.DataFrame({
        "age": rng.normal(40, 10, n),
        "hours": rng.normal(40, 8, n),
        TARGET: rng.integers(0, 2, n),
    })


def test_split_is_60_20_20():
    train, val, test = split_frame(make_frame(), seed=1)
    assert (len(train), len(val), len(test)) == (300, 100, 100)


def test_schema_rejects_missing_target():
    df = make_frame().drop(columns=[TARGET])
    with pytest.raises(ValueError):
        check_schema(df)


def test_schema_rejects_one_class():
    df = make_frame()
    df[TARGET] = 0
    with pytest.raises(ValueError):
        check_schema(df)
```

**Starter code for `tests/test_pipeline.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pandas as pd
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from sklearn.pipeline import make_pipeline

from src.data import TARGET, split_frame
from src.evaluate import choose_threshold
from src.features import build_preprocessor
from src.models import candidate_models
from src.predict import predict_records, validate_records


def make_frame(n=600, seed=0):
    rng = np.random.default_rng(seed)
    df = pd.DataFrame({
        "age": rng.normal(40, 10, n),
        "hours": rng.normal(40, 8, n),
        "workclass": pd.Series(rng.choice(["private", "gov"], n)).where(rng.random(n) > 0.1),
    })
    df[TARGET] = ((df["age"] > 40) ^ (rng.random(n) < 0.2)).astype(int)
    return df


def test_every_candidate_fits_with_missing_values():
    df = make_frame()
    train, val, _ = split_frame(df, seed=0)
    for name, est in candidate_models(0).items():
        pipe = make_pipeline(build_preprocessor(), est)
        pipe.fit(train.drop(columns=[TARGET]), train[TARGET])
        assert pipe.predict(val.drop(columns=[TARGET])).shape == (len(val),), name


def test_threshold_meets_recall_target_on_validation():
    df = make_frame()
    train, val, _ = split_frame(df, seed=0)
    model = make_pipeline(build_preprocessor(), candidate_models(0)["logistic_regression"])
    model.fit(train.drop(columns=[TARGET]), train[TARGET])
    t = choose_threshold(model, val, min_recall=0.80)
    from sklearn.metrics import recall_score
    proba = model.predict_proba(val.drop(columns=[TARGET]))[:, 1]
    assert recall_score(val[TARGET], proba >= t) >= 0.80 or t == 0.5


def test_validate_records_reports_missing_column():
    with pytest.raises(ValueError):
        validate_records([{"age": 30}], ["age", "hours"])


def test_predict_records_returns_probability_and_label():
    df = make_frame()
    train, _, _ = split_frame(df, seed=0)
    columns = [c for c in df.columns if c != TARGET]
    model = make_pipeline(build_preprocessor(), candidate_models(0)["logistic_regression"])
    model.fit(train[columns], train[TARGET])
    meta = {"columns": columns, "threshold": 0.5}
    out = predict_records([{"age": 50, "hours": 40, "workclass": "gov"}], model, meta)
    assert 0.0 <= out[0]["probability"] <= 1.0
    assert out[0]["label"] in (0, 1)
```

**Starter code for `MODEL_CARD.md`:**

```markdown
# Model card: Adult income classifier

## Intended use
Ranking people by the probability that their income exceeds 50,000 dollars, for a
targeting exercise in a development setting. Not for decisions about individuals
such as credit, hiring or pricing.

## Data
UCI Adult Census Income, retrieved from OpenML (version 2). Check the license on
the dataset page before reuse. Fill in the date you downloaded it.

## Metrics
Test-set metrics from models/metadata.json, reported once.

## Slices
The table in reports/error_slices.csv. Name the groups with the lowest accuracy
and the recall gap, and say what you did or did not verify.

## Limitations
Fill in: the dataset is from an older census, the target is a threshold on income,
the threshold was chosen on validation data, and the model was refit on training
plus validation before the final test evaluation.
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it, and add the layout above. Add a `.gitignore` that ignores `models/`, `__pycache__/` and `.venv/`.
2. Implement `src/data.py`, `src/features.py` and `src/models.py` from the sections above. Run `pytest tests/test_data.py` and make those tests pass.
3. Implement `src/experiment.py` and `src/evaluate.py`. Run `pytest tests/test_pipeline.py` and make those tests pass.
4. Implement `src/train.py`. Run `python -m src.train` from the repository root. Confirm that `experiments/results.csv`, `reports/error_slices.csv`, `models/model.joblib` and `models/metadata.json` are created.
5. Run the training script a second time and confirm that the experiment table and the test metrics are the same.
6. Implement `src/predict.py`. Put a sample request in `examples/sample_request.json` with two or three records, and run `python -m src.predict examples/sample_request.json`.
7. Fill in `MODEL_CARD.md` with the test metrics, the worst slice and the limitations. Keep each statement tied to a file or a number.
8. Write the README. Include the one-command run, the experiment table, the test metrics with the threshold, the worst error slice, and a paragraph that says what the model is for and what it must not be used for.
9. Commit the code, the tests, the experiment table, the error slices, the model card, the README and the example request. Do not commit `models/`. Push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] The one-command run is listed: `pip install -r requirements.txt`, `pytest`, `python -m src.train`, and `python -m src.predict examples/sample_request.json`
- [ ] The dataset source and its license check are stated
- [ ] The experiment table includes the baseline and all four candidates, with standard deviations
- [ ] The threshold, the recall target and the test metrics are reported once
- [ ] The worst error slice is named with its row count and accuracy
- [ ] The README links to MODEL_CARD.md and lists what the model must not be used for
- [ ] The README states that the threshold was chosen on validation data and the final model was refit on training plus validation

**Done when:**

- All tests pass with `pytest`
- Two runs of `python -m src.train` produce identical experiment and test numbers
- The final model beats the majority baseline on ROC AUC, and its recall on the test part is at or above the target, or the README explains the shortfall
- The inference command returns a probability and a label for each record in the sample request, and rejects a request with a missing column
- A reviewer can answer three questions from the README alone: what the model is for, where it is weakest, and what would make you retrain it

**What to commit as evidence:** the source code, the tests, the experiment table, the error-slice report, the model card, the README, the example request and the output of the inference command, pasted into the README.

---

## Check your understanding

1. Why does the pipeline fit the imputer and the scaler inside each fold, and what would happen to the cross-validated score if you filled in the medians on the full dataset first?
2. The baseline has a ROC AUC of 0.5 and an accuracy equal to the majority rate. Explain why the baseline is still useful even though it never predicts a positive row.
3. Two candidate models have ROC AUC means of 0.905 and 0.902, with standard deviations of 0.004. Apply the selection rule from this lesson and explain your choice.
4. You chose the threshold on validation data to reach a recall of 0.80, and the test recall is 0.76. Explain the likely causes, and say what you would report.
5. An error slice shows accuracy of 0.62 for one group with 45 rows, and the overall accuracy is 0.84. Explain why you should not conclude that the model is biased against that group yet, and what the next step is.
6. The refit model uses more data than the development model. Explain why the threshold chosen with the development model may not transfer exactly, and how you would check it.
7. A product manager asks to deploy the model with a threshold of 0.5 instead of the recall-based threshold, because it gives a higher accuracy. Explain what the trade-off is, and what you would ask them to decide before you change the code.

### Answers

1. The imputer and scaler learn statistics from the rows they are fitted on. If they are fitted on the full dataset, the held-out fold contributes to the statistics used to transform the training rows, which inflates the score. The pipeline refits them inside each training fold, so the held-out rows influence only the evaluation.
2. The baseline is the reference that every model must clear on the metric that matters, and it shows how much of the accuracy comes from the class balance alone. A model with accuracy 0.84 against a baseline of 0.76 is a modest gain, while the same model against a baseline of 0.5 would look far stronger, so the comparison is only meaningful with the baseline shown.
3. The gap between the two means is 0.003, which is smaller than the standard deviation of 0.004, so the difference is not meaningful. The rule says to prefer the simpler model, which is the one with fewer moving parts. I would choose the simpler of the two models, and state that the choice is a tie on this evidence.
4. The likely causes are the difference between the validation and test samples, the refit on more data that shifted the score distribution, and sampling noise in a test set of this size. I would report the test recall next to the target, give the interval if I can compute one, and say that the target was missed by this margin on this sample.
5. A group of 45 rows gives a noisy accuracy, so the gap may be chance, and the slice has not been checked on a second sample. The next step is to compute an interval for the slice, look at the error types for those rows, and check whether the pattern holds on new data before drawing any conclusion about bias.
6. The refit model sees more data and may produce slightly different probabilities, so a threshold tuned on the smaller model may give a different recall. I would check the recall of the refit model on the validation part, which it has seen, and I would report that the check is not an independent estimate. The honest test is the single test evaluation already in the pipeline, and I would state the shift if it exists.
7. A threshold of 0.5 maximizes overall accuracy on a dataset with a 25 percent positive rate, but it may miss many positives, so recall falls below the target. The decision belongs to the product owner: they need to state the cost of a missed high earner against the cost of a false alarm, and whether the recall target is still the goal. I would ask for that decision in writing before changing the code, and I would record the new threshold and the reason in the metadata.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Machine learning](https://en.wikipedia.org/wiki/Machine_learning)
- Mitchell et al., "Model Cards for Model Reporting" (2019), available on arXiv. Search the title.
- scikit-learn user guide, "Common pitfalls and recommended practices", at scikit-learn.org under the stable documentation.
- UCI Machine Learning Repository, the "Adult" dataset page, for the description of the variables and the license.

**Tutorials**

- scikit-learn documentation, "Column Transformer with Mixed Types", which shows the preprocessing pattern used in this project.
- joblib documentation, "Persistence", at joblib.readthedocs.io, for saving and loading the fitted pipeline.

**Watch**

- Google Machine Learning Crash Course, the sections on production ML systems and on data, at developers.google.com/machine-learning/crash-course.
- StatQuest with Josh Starmer (YouTube), the videos on the topics from Days 24 to 37 you want to review before the project.

**Practice**

- Kaggle, the "Adult Census Income" dataset page and its public notebooks. Compare your experiment table with the public results for the same dataset, and explain any difference.

---

## Tomorrow

Tomorrow starts Module 5, deep learning foundations, where the classical tools you have built are the baseline the neural networks must beat. You will meet the artificial neuron, the weights and biases that you have seen as coefficients, the layers that stack them and the forward pass that computes a prediction. Keep this repository open. The logistic regression you trained today is a single neuron with a sigmoid, and the same evaluation discipline applies to every network that follows.

**Next:** [Day 39: Neural Networks](../Module-05-Deep-Learning-Foundations/Day-39-Neural-Networks.md)

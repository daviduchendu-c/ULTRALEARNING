# Day 23: Data Project

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 6 hours, spread over two sessions if you can
**Level:** intermediate. This is the module project. It asks you to take a messy dataset you did not design and carry it through every step of Days 17 to 22.

**By the end of this day, you will have:**

- A cleaned dataset with a documented rule for every change
- An EDA report with at least five findings, each backed by a number or a plot
- A preprocessing pipeline that is fitted only on training data and saved with its settings
- A train, validation and test strategy, applied once, with the test set reserved for the final check
- A written list of assumptions, each marked as a decision you made or a fact you checked

**What you need:**

- A computer with Python 3.10 or newer
- pandas 2.2 or newer, NumPy 2.0 or newer, matplotlib 3.8 or newer
- scikit-learn 1.5 or newer (`pip install "scikit-learn>=1.5"`), free under the BSD 3-Clause license
- pytest 8.0 or newer (`pip install "pytest>=8.0"`)
- The UCI Adult dataset, free from the UCI Machine Learning Repository (search "adult UCI" and download `adult.data`, the training file). The dataset is distributed for research and education. Read the dataset page for its license and citation.
- Git and a free GitHub account for the project section

**Prerequisites:**

- Days 17 to 22: arrays, DataFrames, cleaning, EDA, feature engineering, splits and reproducibility
- Day 14: the basics of a classification metric, such as accuracy and a confusion matrix
- Day 13: conditional probability, so that you can read a predicted probability

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Scoping the question](#scoping-the-question)
- [Assumptions and decisions](#assumptions-and-decisions)
- [Train, validation and test strategy](#train-validation-and-test-strategy)
- [Baselines and metrics](#baselines-and-metrics)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an end-to-end data project on the UCI Adult dataset](#project-an-end-to-end-data-project-on-the-uci-adult-dataset)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every earlier day in this module gave you a tool on a clean, small example. Real work does not arrive that way. A file comes from another team with question marks where numbers should be, labels that disagree with the documentation, columns whose meaning nobody remembers, and a request that says only "predict who will earn more than fifty thousand". Your job is to turn that situation into a sequence of decisions that you can defend.

This day is practice in that sequence. You will not be told which rows to remove or which features to build. You will make those choices, record them, test them and explain them. The skill you are building is the habit of asking, at every step, what the data says, what you assumed, and how you would know if the assumption were wrong.

On the job, a data project is judged by more than its model score. A reviewer asks whether the test set was touched early, whether the cleaning rules were applied the same way to every split, and whether the assumptions are written down. A project that answers those questions well is one the team can rely on, extend and hand over. That is the standard for this day.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1994 | Income records are extracted from the 1994 United States Census database. | The Adult dataset comes from this census extract. |
| 1996 | Ronny Kohavi and Barry Becker donate the extract to the UCI Machine Learning Repository, the public archive of benchmark datasets. Kohavi's 1996 work on a decision-tree and Naive Bayes hybrid uses it. | It becomes one of the most-used tabular benchmarks in teaching and in research. |
| 2000s | The dataset appears in many papers that compare classification methods, which makes its scores a shared reference point. | Comparing your result with published results requires the same split and metric, which is why the protocol matters. |
| 2010s | Researchers document fairness problems in income prediction from census-style data, including the use of sensitive attributes. | Modern projects must state which attributes they use and why. |
| 2020s | Teaching courses use the dataset for end-to-end practice, with cleaning, pipelines and reproducibility as the focus. | It is a good training ground for the workflow in this module. |

**Takeaway:** a benchmark dataset is a shared yardstick. Its value depends on everyone using the same split and reporting the same metric, and on everyone remembering that the data describes people.

---

## Glossary

- **Scope**: the precise question the project answers, including the target, the population and the decision the answer supports.
- **Target**: the column the model predicts. In this project, it is a binary flag for income above 50,000 dollars.
- **Assumption**: a statement you accept without proof, such as "exact duplicate rows are repeated records", written down so that others can challenge it.
- **Stratified split**: a split that keeps the share of each class the same in every part.
- **Validation set**: rows used to compare models and choose settings.
- **Test set**: rows used once, at the end, to estimate performance on unseen data.
- **Baseline**: the simplest sensible prediction, such as always predicting the most common class, against which any model is compared.
- **ROC AUC**: the area under the receiver operating characteristic curve. It measures how well a model ranks positive examples above negative ones, from 0.5 for random ranking to 1.0 for perfect ranking.
- **Average precision**: a summary of the precision-recall curve, useful when the positive class is rare.
- **Sensitive attribute**: a characteristic such as sex or race, which is protected in many settings and needs careful handling in any model.

---

### Scoping the question

A project without a precise question wanders. Before you touch the data, write one sentence that names the target, the population, and the decision the prediction supports. "Predict whether a person earns more than 50,000 dollars a year, using census records of working adults, to support a research estimate of income patterns" is a scope. "Predict income" is not, because it does not say who the rows describe or what the answer is for.

From first principles, a good scope prevents three kinds of mistake. It stops you from including columns that are not available at the time the prediction would be made. It stops you from comparing your results with a different target. And it tells you which metric matters, because a fraud decision and a research estimate have different costs of error. Write the scope at the top of the README before you write any code, and check it again when the project ends.

The honest limitation of a scope is that it is a decision too. In this project, the scope mentions research, which is a choice that makes some uses, such as deciding who gets a loan, inappropriate. Say so in the README.

---

### Assumptions and decisions

An assumption is something you accept without checking it, and a decision is a choice you make about the data. Both must be written down. An unwritten assumption is the one that breaks a project six months later, when someone asks why a row disappeared and nobody can answer.

From first principles, keep two lists. The first lists facts you checked, such as the number of rows, the number of missing values in each column and the meaning of each code in the documentation. The second lists assumptions and decisions, each with a reason and a count of the rows it affected. A line in the second list might read: "Exact duplicate rows are removed. Reason: the file has no record ID, so identical rows cannot be told apart. Affected: 24 rows, which is 0.1 percent of the file." Such a line lets a reviewer decide whether they agree, and it tells you what to recheck if the file changes.

```python
import pandas as pd

def decision_log(entries: list[dict]) -> pd.DataFrame:
    """Each entry: {'step', 'rule', 'reason', 'rows_affected'}."""
    return pd.DataFrame(entries, columns=["step", "rule", "reason", "rows_affected"])
```

The common mistake is to write the decisions only in code comments, which nobody reads. Put them in a table in the README and generate that table from the same counts the pipeline prints. A practical habit is to keep a data card or a datasheet next to each dataset, describing its source, collection method, known gaps and intended uses. Writing a datasheet is the most useful habit a junior data practitioner can build.

---

### Train, validation and test strategy

A strategy for splitting the data is a plan for how each part will be used, and in what order. The training set fits the model. The validation set compares models and settings. The test set gives the final number, and it is opened once. The strategy is a promise that the final number is honest.

From first principles, use a stratified split when the target is imbalanced, so that every part has the same share of positives. Use a fixed random seed so the split can be reproduced. Fit every preprocessing step on the training rows only, as you learned in Day 19. Choose the model on the validation rows, and then, after the choice is made, evaluate once on the test rows. If you look at the test score and then change the model, the test set has become a second validation set, and your final number is no longer honest.

```python
from sklearn.model_selection import train_test_split

def three_way_split(df, target="target", seed=42):
    train_val, test = train_test_split(df, test_size=0.2,
                                       stratify=df[target], random_state=seed)
    train, val = train_test_split(train_val, test_size=0.25,
                                  stratify=train_val[target], random_state=seed)
    return train, val, test
```

The numbers 0.25 and 0.2 produce a 60, 20 and 20 split, because 0.25 of 80 percent is 20 percent. A common mistake is to split once into train and test, then split the training part again into train and validation, and then tune on the test set anyway, because it is the largest number that looks convenient. Another mistake is to split after cleaning with a rule that uses the whole dataset, such as a median computed before the split. The honest order is split first, then fit everything else on the training part. A separate team often holds the test set, or it is locked in a file until release, to stop this temptation.

---

### Baselines and metrics

A baseline gives a number that any model must beat. Without one, a score of 0.85 means nothing. With a baseline that always predicts the majority class and scores 0.76 accuracy, the same 0.85 may be a real improvement, or it may be a small one. Choose the baseline before you train anything, and report it next to every model.

From first principles, choose a metric that matches the question. Accuracy counts how often the prediction is right, and it is misleading when one class is rare. ROC AUC measures how well the model ranks positives above negatives across all thresholds, and it does not depend on the threshold you pick. Average precision focuses on the positive class and is more informative when positives are rare. Report at least two metrics, and state which one you optimized.

```python
import numpy as np
from sklearn.metrics import roc_auc_score, average_precision_score

def report(y_true, proba):
    return {
        "roc_auc": round(roc_auc_score(y_true, proba), 4),
        "average_precision": round(average_precision_score(y_true, proba), 4),
        "positive_rate": round(float(np.mean(y_true)), 4),
    }
```

The mistake is to compare a model's validation score with a published test score from another paper, which uses a different split. Compare only numbers computed with the same protocol. A model is reported with its baseline, its confidence interval if available, and the population it was measured on, so that a reader can judge whether the improvement matters.

---

## Hands-on exercises

Work through these before you start the project, using the first thousand rows of the file if the full dataset is slow.

1. **Read the documentation.** Read the dataset page, the column descriptions and the note about missing values. Write three facts into a table with the source line for each.
2. **Load with missing values.** Load the file with the column names given and with `"?"` read as missing. Count the missing values per column. Report the three columns with the most missing values.
3. **Duplicate count.** Count the exact duplicate rows. Decide whether they are records or errors, write the decision in one sentence, and then apply it.
4. **Target check.** Count the target classes. Compute the baseline accuracy. State the positive rate and explain why it matters for the metric.
5. **Two EDA findings.** Find one numeric relationship with the target and one categorical relationship with the target. Support each with a table of rates or means and a plot.
6. **Redundancy check.** Find two columns that encode the same information. Decide which one to keep and write the reason.
7. **Split protocol.** Write the split function, run it with a fixed seed, and confirm that the positive rate is nearly the same in all three parts.

---

## Project: an end-to-end data project on the UCI Adult dataset

### Goal

Carry the UCI Adult census income dataset through every step of this module. Clean it with documented rules, explore it, build a preprocessing pipeline that is fitted on training data only, compare two models on validation data, and evaluate the chosen model once on the test set. Produce a cleaned file, an EDA report, the pipeline and a README with the assumptions. The test checks protect the steps that break most often.

Suggested repository name: `100-days-ai-day-23-data-project`

### Repo layout

```text
100-days-ai-day-23-data-project/
├── README.md
├── data/
│   └── adult.data            (download, do not commit)
├── project/
│   ├── __init__.py
│   ├── load.py
│   ├── clean.py
│   ├── eda.py
│   ├── pipeline.py
│   └── evaluate.py
├── tests/
│   ├── fixtures/adult_sample.data
│   └── test_project.py
├── outputs/
│   └── .gitkeep
├── run_project.py
├── requirements.txt
└── .gitignore
```

Add `data/` to `.gitignore`, because the dataset file should not be committed.

### Starter code

`requirements.txt`:

```text
pandas>=2.2
numpy>=2.0
scikit-learn>=1.5
matplotlib>=3.8
pytest>=8.0
```

`project/load.py`:

```python
from pathlib import Path
import pandas as pd

COLUMNS = [
    "age", "workclass", "fnlwgt", "education", "education-num",
    "marital-status", "occupation", "relationship", "race", "sex",
    "capital-gain", "capital-loss", "hours-per-week", "native-country", "income",
]


def load_adult(path: Path) -> pd.DataFrame:
    return pd.read_csv(path, names=COLUMNS, na_values="?", skipinitialspace=True)
```

`project/clean.py`:

```python
import pandas as pd

RENAMES = {c: c.replace("-", "_") for c in [
    "education-num", "marital-status", "capital-gain", "capital-loss",
    "hours-per-week", "native-country",
]}


def clean(raw: pd.DataFrame) -> pd.DataFrame:
    """Each rule is documented in the README decision table."""
    out = raw.rename(columns=RENAMES)
    out["target"] = (out["income"].str.strip() == ">50K").astype(int)
    out = out.drop(columns=["income", "education"])       # education-num carries the same meaning
    out = out.drop_duplicates()                           # no record ID exists, so identical rows are treated as repeats
    return out.reset_index(drop=True)
```

`project/pipeline.py`:

```python
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

NUM = ["age", "fnlwgt", "education_num", "capital_gain", "capital_loss", "hours_per_week"]
CAT = ["workclass", "marital_status", "occupation", "relationship",
       "race", "sex", "native_country"]


def build_pipeline(model):
    numeric = make_pipeline(SimpleImputer(strategy="median"), StandardScaler())
    categorical = make_pipeline(
        SimpleImputer(strategy="constant", fill_value="missing"),
        OneHotEncoder(handle_unknown="ignore", sparse_output=False),
    )
    pre = ColumnTransformer([("num", numeric, NUM), ("cat", categorical, CAT)])
    return make_pipeline(pre, model)
```

`project/evaluate.py`:

```python
import pandas as pd
from sklearn.ensemble import HistGradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import average_precision_score, roc_auc_score
from project.pipeline import build_pipeline

MODELS = {
    "logistic_regression": lambda: LogisticRegression(max_iter=2000),
    "hist_gradient_boosting": lambda: HistGradientBoostingClassifier(random_state=0),
}


def validation_scores(train: pd.DataFrame, val: pd.DataFrame) -> dict:
    X_tr, y_tr = train.drop(columns="target"), train["target"]
    X_va, y_va = val.drop(columns="target"), val["target"]
    results = {"baseline_positive_rate": round(float(y_tr.mean()), 4)}
    for name, make in MODELS.items():
        pipe = build_pipeline(make()).fit(X_tr, y_tr)
        proba = pipe.predict_proba(X_va)[:, 1]
        results[name] = {
            "roc_auc": round(roc_auc_score(y_va, proba), 4),
            "average_precision": round(average_precision_score(y_va, proba), 4),
        }
    return results


def final_test(model_name: str, train_val: pd.DataFrame, test: pd.DataFrame) -> dict:
    """Run once, after the model choice is fixed on validation data."""
    pipe = build_pipeline(MODELS[model_name]()).fit(
        train_val.drop(columns="target"), train_val["target"])
    proba = pipe.predict_proba(test.drop(columns="target"))[:, 1]
    return {"model": model_name,
            "roc_auc": round(roc_auc_score(test["target"], proba), 4),
            "average_precision": round(average_precision_score(test["target"], proba), 4)}
```

`project/eda.py`:

```python
from pathlib import Path
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import pandas as pd


def eda_text(df: pd.DataFrame) -> str:
    parts = [
        f"rows: {len(df)}",
        "target share:\n" + df["target"].value_counts(normalize=True).round(4).to_string(),
        "mean target by sex:\n" + df.groupby("sex")["target"].mean().round(4).to_string(),
        "mean target by education_num:\n"
        + df.groupby("education_num")["target"].mean().round(3).to_string(),
        "numeric summary:\n" + df[["age", "hours_per_week", "capital_gain"]].describe().round(1).to_string(),
    ]
    return "\n\n".join(parts)


def save_eda(df: pd.DataFrame, out_dir: Path) -> None:
    out_dir.mkdir(parents=True, exist_ok=True)
    fig, ax = plt.subplots(figsize=(7, 4))
    ax.hist(df["age"], bins=40, color="#4C72B0")
    ax.set_title("Age distribution")
    ax.set_xlabel("age (years)")
    ax.set_ylabel("people")
    fig.tight_layout()
    fig.savefig(out_dir / "age_hist.png", dpi=120)
    plt.close(fig)
```

`project/__init__.py`: leave it empty.

`run_project.py`:

```python
from pathlib import Path
from project.load import load_adult
from project.clean import clean
from project.eda import eda_text, save_eda
from project.evaluate import validation_scores, final_test
from project.pipeline import build_pipeline
from project.evaluate import MODELS
from sklearn.model_selection import train_test_split

DATA = Path("data/adult.data")
OUT = Path("outputs")

if __name__ == "__main__":
    OUT.mkdir(exist_ok=True)
    raw = load_adult(DATA)
    df = clean(raw)
    df.to_csv(OUT / "cleaned.csv", index=False, lineterminator="\n")
    (OUT / "eda_report.txt").write_text(eda_text(df), encoding="utf-8")
    save_eda(df, OUT)

    train_val, test = train_test_split(df, test_size=0.2, stratify=df["target"], random_state=42)
    train, val = train_test_split(train_val, test_size=0.25, stratify=train_val["target"], random_state=42)

    scores = validation_scores(train, val)
    print("validation:", scores)
    best = max(MODELS, key=lambda m: scores[m]["roc_auc"])
    print("chosen on validation:", best)
    print("final test, run once:", final_test(best, train_val, test))
```

`tests/fixtures/adult_sample.data`: write these six lines exactly, with a trailing newline:

```text
39, State-gov, 77516, Bachelors, 13, Never-married, Adm-clerical, Not-in-family, White, Male, 2174, 0, 40, United-States, <=50K
50, Self-emp-not-inc, 83311, Bachelors, 13, Married-civ-spouse, Exec-managerial, Husband, White, Male, 0, 0, 13, United-States, >50K
38, ?, 215646, HS-grad, 9, Divorced, ?, Not-in-family, White, Male, 0, 0, 40, United-States, <=50K
53, Private, 234721, 11th, 7, Married-civ-spouse, Handlers-cleaners, Husband, Black, Male, 0, 0, 40, ?, <=50K
28, Private, 338409, Bachelors, 13, Married-civ-spouse, Prof-specialty, Wife, Black, Female, 0, 0, 40, Cuba, <=50K
28, Private, 338409, Bachelors, 13, Married-civ-spouse, Prof-specialty, Wife, Black, Female, 0, 0, 40, Cuba, <=50K
52, Self-emp-not-inc, 209642, HS-grad, 9, Married-civ-spouse, Exec-managerial, Husband, White, Male, 0, 0, 45, United-States, >50K
31, Private, 45781, Masters, 14, Never-married, Prof-specialty, Not-in-family, White, Female, 14084, 0, 50, United-States, >50K
```

`tests/test_project.py`:

```python
from pathlib import Path
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split
from project.load import load_adult
from project.clean import clean
from project.pipeline import build_pipeline, NUM, CAT
from project.evaluate import validation_scores, MODELS

FIXTURE = Path(__file__).parent / "fixtures" / "adult_sample.data"


def test_question_marks_become_missing():
    raw = load_adult(FIXTURE)
    assert raw["workclass"].isna().sum() == 1
    assert raw["native-country"].isna().sum() == 1


def test_clean_builds_binary_target_and_drops_duplicates():
    df = clean(load_adult(FIXTURE))
    assert set(df["target"].unique()) <= {0, 1}
    assert len(df) == 7                       # eight raw rows, one exact duplicate
    assert "income" not in df.columns


def test_imputer_statistics_come_from_training_rows_only():
    df = clean(load_adult(FIXTURE))
    train = df.iloc[:5]
    holdout = df.iloc[5:].copy()
    holdout["age"] = 999.0                        # an extreme value that would move a median
    pipe = build_pipeline(MODELS["logistic_regression"]()).fit(
        train.drop(columns="target"), train["target"])
    stat = pipe[0].named_transformers_["num"][0].statistics_[NUM.index("age")]
    assert np.isclose(stat, train["age"].median())
    contaminated = pd.concat([train, holdout])
    assert not np.isclose(contaminated["age"].median(), stat)


def test_pipeline_handles_unseen_category():
    df = clean(load_adult(FIXTURE))
    pipe = build_pipeline(MODELS["logistic_regression"]()).fit(
        df.drop(columns="target"), df["target"])
    new = df.drop(columns="target").head(2).copy()
    new["occupation"] = "Astronaut"
    proba = pipe.predict_proba(new)
    assert proba.shape == (2, 2)


def test_validation_scores_have_expected_keys():
    df = clean(load_adult(FIXTURE))
    train, val = train_test_split(df, test_size=0.25, stratify=df["target"], random_state=0)
    scores = validation_scores(train, val)
    assert "baseline_positive_rate" in scores
    assert set(MODELS) <= set(scores)
```

### Step-by-step tasks

1. Download `adult.data` into `data/`. Create the repository, the environment and the install. Create the files as shown, and write the fixture file with the eight lines given above.
2. Run `pytest -q`. All five tests should pass on the fixture. If the stratified split in the last test fails on the small fixture, explain why in the README and keep the test, because small stratified splits need at least two rows per class.
3. Write `scope` at the top of the README, in one sentence, before you run the full pipeline. Include the target, the population and the decision the prediction supports, and one use you will not support.
4. Run the cleaning step on the full file. Build the decision table in the README: each rule, its reason and the number of rows it affected. The duplicate count should match the number you counted in the exercises.
5. Run `python run_project.py`. Copy the validation scores, the chosen model and the final test result into the README. Do not change the model after you see the test result. If you want to try something else, write it as a future step.
6. Open `outputs/eda_report.txt` and `outputs/age_hist.png`. Write five findings, each with the number or plot that supports it and one sentence on what it might mean for the model.
7. Write the assumptions list. Mark each item as a fact you checked or a decision you made, and give the number of rows it affected.
8. Write a short section on fairness. Name the sensitive attributes in the data, say whether you used them as features, and explain what the model would do with them removed. Keep the explanation to a paragraph.

### README checklist

- [ ] The scope sentence, the target, the population and one use you will not support
- [ ] The dataset source, the license note and the citation from the dataset page
- [ ] The decision table: each cleaning rule, its reason and the rows affected
- [ ] The split strategy, the seed, and the final sizes of train, validation and test
- [ ] The baseline positive rate and the validation scores for both models
- [ ] The final test result, reported once, with the model name
- [ ] Five EDA findings, each with its evidence
- [ ] The assumptions list, each marked as a fact checked or a decision made
- [ ] The fairness paragraph
- [ ] Two limitations, for example that the data is from 1994 and that the test set is a single split

### Done when

- `pytest -q` passes all five tests on the fixture
- `run_project.py` runs end to end on the full file and writes `outputs/cleaned.csv`, `outputs/eda_report.txt` and the age plot
- The imputer test shows that the fitted statistics come from training rows only
- The test set is used once, in `final_test`, after the model is chosen on validation rows
- Every decision is in the README decision table with a count
- The README states the scope, the baseline, both metrics and the fairness note

### Evidence to commit to GitHub

Commit the package, the tests and the fixture, the runner, the README with the decision table, the assumptions, the validation and test scores, and the EDA outputs. Do not commit `data/adult.data`, which should be listed in `.gitignore`. Commit the cleaned file only if the license allows redistribution, and say so in the README if you leave it out. The commit history should show the scope commit before the first model commit, so the reader can see the question was fixed first. The reader should be able to show a green `pytest -q` run, the one-time test result, and the decision table.

---

## Check your understanding

1. You split the data into train and test, then clean the full training file with a rule that uses the median of the whole file, and then split again for validation. Name the leak, explain why the test score is no longer an honest estimate, and describe the order of steps that avoids it.
2. Your validation accuracy is 0.84 and the baseline accuracy is 0.76. Is the model good? Name the metric you would add and explain why accuracy alone is not enough when the positive class is about a quarter of the data.
3. A teammate says the test score improved after they tried a new feature, so they keep the feature. Explain why this choice spends the test set, and describe what they should do instead.
4. Two duplicate rows in a census file have identical values. One reviewer says they are errors, and another says they are real people with the same profile. Write the decision you would make, and the check that would change your mind.
5. A model scores well on the validation set but poorly on a subgroup defined by a sensitive attribute. Name two things you would check before deciding whether the model is unfair, and state which of them you cannot fix by changing the code alone.
6. Describe what a data card or datasheet would contain for this project, and explain which section a future user would need most if they wanted to reuse the model on a different country's census.

Work each answer out in writing before you read the answers below.

**Answers**

1. The median was computed over the full file, so the test rows influenced the values used to clean the training rows. The test score is optimistic. Split first into train, validation and test, compute the median on the training rows only, and apply that value to the other two parts.
2. An accuracy of 0.84 against a 0.76 baseline is better, but with about 24 percent positives the margin is small in terms of the minority class. Add ROC AUC and average precision, and compare average precision with the positive rate, which is its baseline. Accuracy alone can hide a model that misses most positives.
3. Choosing the feature by the test score turns the test set into a second validation set, so the reported number is optimistic. Decide on validation scores or on cross-validation within the training data. If the decision is changed after a peek, say so in the README, and get a fresh test set if the project allows it.
4. The file has no record ID, so identical rows cannot be shown to belong to one person, and the reviewer's concern is real. A reasonable decision is to remove exact duplicates and report the count, because repeated rows can inflate the weight of one profile and put copies on both sides of the split. You would change your mind if the file had a unique ID or a survey weight that separates people, or if removing the rows changed the class balance a lot.
5. Check whether the gap is larger than the noise, using a confidence interval and the subgroup size. Check whether the subgroup has a different label rate or different feature distributions, and whether the errors are of the same type. You cannot fix the historical differences in the labels by changing the code, because they reflect social conditions, and a decision about use is a policy choice for people.
6. A datasheet would record the source, the 1994 collection method, the population, the missing value convention, the labeling, the license, the intended and out-of-scope uses, and the sensitive attributes. For reuse in another country, the population and collection sections matter most, because the categories, the income threshold and the time period do not transfer.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "Census", for what census microdata is and how it is collected.
- Wikipedia, article "Receiver operating characteristic", for the ROC curve and the meaning of the area under it.
- Wikipedia, article "Precision and recall", for the definitions used in the average precision score.
- Wikipedia, article "Algorithmic bias", for the ways a model can treat groups unevenly.
- Timnit Gebru and colleagues, "Datasheets for Datasets", a 2018 paper that proposes the datasheet practice. Search the title to find the arXiv version.
- scikit-learn documentation, User Guide sections on "Pipelines and composite estimators" and "Model evaluation", which cover the tools used in this project. Search the titles on scikit-learn.org.
- UCI Machine Learning Repository, the dataset page for "Adult", which describes the columns, the missing value convention and the citation.

**Tutorials**

- GeeksforGeeks, "Titanic Survival Prediction" and similar end-to-end classification tutorials, read for the workflow rather than for the code.
- W3Schools, "Machine Learning" in the Python section of w3schools.com, for the basic vocabulary of training and testing.

**Watch**

- StatQuest with Josh Starmer, on YouTube, for the videos on ROC curves and on precision and recall.
- Andrew Ng, "Machine Learning Specialization" on Coursera, for the module on model evaluation and on the train, validation and test workflow.
- Search YouTube for "fairness in machine learning" and choose a talk from a research group or a university course.

**Practice**

- Kaggle, "Adult Census Income" data and the public notebooks written for it. Read two notebooks and note where they differ in cleaning, splitting and evaluation.

---

## Tomorrow

Tomorrow starts the next module, which goes deeper into classical machine learning, beginning with linear regression and logistic regression. You will fit models on the kind of features you built today and read their coefficients. Everything you did here, the split, the pipeline, the baseline and the record of decisions, carries forward unchanged.

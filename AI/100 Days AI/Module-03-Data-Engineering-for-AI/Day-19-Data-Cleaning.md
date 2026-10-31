# Day 19: Data Cleaning

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 4 hours
**Level:** intermediate. You need pandas from Day 18 and a basic idea of train and test sets from Day 14. The project is the longest so far, so plan for the full four hours.

**By the end of this day, you will have:**

- A clear method for finding and handling missing values, duplicates, invalid records, outliers, wrong data types, leakage, label errors and data quality problems
- A tested cleaning module and a data quality report generated from a messy customer file
- A list of cleaning decisions, each with the reason it was made and the rows it affected

**What you need:**

- A computer with Python 3.10 or newer
- pandas 2.2 or newer (`pip install "pandas>=2.2"`), free under the BSD 3-Clause license
- NumPy 2.0 or newer
- scikit-learn 1.5 or newer (`pip install "scikit-learn>=1.5"`), free under the BSD 3-Clause license
- pytest 8.0 or newer (`pip install "pytest>=8.0"`)
- A code editor such as VS Code (free) or Jupyter Lab
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 18: DataFrames, filtering, merging and `fillna`
- Day 14: the mean, the median, quartiles and the idea of a train and test split
- Day 13: the idea of a probability and a conditional probability

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Missing values](#missing-values)
- [Duplicates](#duplicates)
- [Invalid records](#invalid-records)
- [Outliers](#outliers)
- [Data types](#data-types)
- [Leakage](#leakage)
- [Label errors](#label-errors)
- [Data quality](#data-quality)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a data cleaning audit with a quality report](#project-a-data-cleaning-audit-with-a-quality-report)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most of a machine learning project is spent on data, and most of the time spent on data is spent on cleaning it. Practitioners often report that the cleaning step takes far longer than the modeling step, and they are usually right. A model cannot tell that a customer's age is 230, that the same order appears twice, or that a label was typed by a tired annotator on the wrong row. It learns whatever pattern is in the file, errors included.

Cleaning is therefore where you decide what the data means. Every cleaning step changes the population your model learns from, so every step is a modeling decision. Dropping a row with a missing income drops the customers who are most likely to be wealthy or most likely to be hiding information. Capping an outlier changes the average. The goal is a table you can explain and defend, with a record of what was changed.

On the job, data cleaning shows up as a validation step in an ingestion pipeline, a notebook that documents what was removed, a data quality dashboard that alerts when the null rate jumps, and a review meeting where someone asks why 3 percent of the labels disappeared. This lesson gives you the vocabulary and a repeatable method for those tasks.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1960s to 1970s | Statisticians develop methods for missing data and for robust estimates, such as the median and trimmed means, to resist bad values. | Many cleaning rules still come from classical statistics. |
| 1980s | Relational databases make integrity constraints, such as unique keys and not-null columns, a standard part of data design. | Many of the checks you write in pandas are the same rules a database enforces. |
| 1987 | Donald Rubin publishes the foundation of multiple imputation, a method that fills missing values several times to show the uncertainty they add. | The idea of imputing with uncertainty is still the careful way to handle missing data. |
| 2000s | Data quality becomes a field of its own in business intelligence, with dimensions such as accuracy, completeness and consistency. | The quality dimensions in this lesson come from that work. |
| 2010s to 2020s | Researchers show that large public benchmark datasets contain label errors, and tools such as cleanlab appear to find them automatically. | Label quality is now treated as a first-class part of model development. |

**Takeaway:** the rules are old, but the scale is new. A check you once ran by eye now has to run on every batch.

---

## Glossary

- **Missing value**: an entry that holds no data, often written as `NaN`, `null`, an empty string or a placeholder such as `-1`.
- **Imputation**: filling a missing value with an estimated value.
- **Duplicate**: a record that repeats another record in every field, or in the fields that define its identity.
- **Invalid record**: a record that breaks a rule the data must follow, such as a negative age or a date that does not exist.
- **Outlier**: a value that is far from where most values sit. It may be an error or a real and rare event.
- **Winsorizing**: replacing extreme values with the nearest acceptable value, rather than removing the row.
- **Data type**: the kind of value a column holds, such as integer, float, text, date or category.
- **Leakage**: information about the target that reaches the model through a feature or a preprocessing step, which makes the test score look better than the real performance.
- **Label error**: an example whose recorded label is wrong for its features.
- **Data quality dimension**: a property you measure, such as completeness, validity, uniqueness, consistency or timeliness.
- **Null rate**: the share of rows where a column is missing.

---

### Missing values

A missing value is a blank in a form. A blank can mean "not applicable", "not known", "refused to answer", or "the system failed to record it". The correct handling depends on which one it is, and the data itself rarely tells you. Before you choose a fix, look at where the blanks are and what they correlate with.

From first principles, missingness falls into three broad patterns. It can be missing completely at random, where the gap has no link to any value. It can be missing at random, where the gap depends on other observed columns, such as older customers skipping an optional field. Or it can be missing not at random, where the gap depends on the missing value itself, such as high earners declining to report income. Only the first pattern is safe to ignore. The other two can bias your results.

```python
import numpy as np
import pandas as pd

df = pd.DataFrame({
    "age": [25, np.nan, 40, np.nan, 31],
    "income": [50_000, 62_000, np.nan, 90_000, 48_000],
    "churned": [0, 1, 0, 1, 0],
})

print(df.isna().mean())                         # share missing per column
print(df.groupby("churned")["age"].apply(lambda s: s.isna().mean()))
```

The second print shows whether the missingness in `age` differs between churned and retained customers. If it does, dropping those rows will change the class balance and possibly the answer. A common mistake is to fill a column with its mean before checking the pattern, which hides the signal. Keep an indicator column that records which values were filled. The missingness indicator is often a useful feature in its own right, because a blank field is itself information.

---

### Duplicates

Duplicates are the same real-world thing recorded more than once. Think of a guest list where one person was typed twice, once as "Ada Lovelace" and once as "A. Lovelace". An exact duplicate is the easy case, where every field matches. A near-duplicate is harder, because the fields differ slightly.

From first principles, you need a key that defines identity. A customer ID, an order number or a combination of fields such as name, date of birth and postal code can serve as that key. Once you choose the key, `duplicated` marks the repeats, and `drop_duplicates` removes them, keeping the first or last occurrence.

```python
import pandas as pd

df = pd.DataFrame({
    "customer_id": ["c1", "c2", "c2", "c3"],
    "age": [25, 40, 40, 31],
    "note": ["a", "b", "b2", "c"],
})

print(df.duplicated().sum())                         # exact duplicates across all columns: 0
print(df.duplicated(subset="customer_id").sum())     # repeats of the key: 1
clean = df.drop_duplicates(subset="customer_id", keep="first")
print(len(clean))                                    # 3
```

The first print returns zero, even though `c2` appears twice, because the `note` column differs. This is the classic trap. Decide the key first, then check duplicates on that key. The second mistake is to keep the first occurrence without thinking about which one is correct. If the duplicates carry different timestamps, keep the most recent one. Duplicated rows double the revenue in a report or the positive examples in a training set, and both errors are hard to notice without a count.

---

### Invalid records

An invalid record breaks a rule that the real world imposes. An age cannot be negative. An order date cannot be in the future. A postal code has a fixed format in each country. A rule is a predicate, a yes or no test on each row. You write the rules down and count the failures.

The useful habit is to separate two actions: flagging and removing. A flag is a column that marks which rows fail a rule. Removal is a decision you make after you have seen the flags. Flagging keeps the audit trail and lets you test how sensitive your results are to the rule.

```python
import numpy as np
import pandas as pd

df = pd.DataFrame({"age": [25.0, -1.0, 230.0, np.nan, 40.0]})

valid_age = df["age"].between(18, 110)     # NaN fails the test, so it becomes False
df["age_invalid"] = ~valid_age & df["age"].notna()
df["age_clean"] = df["age"].where(valid_age)

print(df)
```

The `where` call keeps values where the condition holds and replaces the rest with `NaN`, so an invalid age becomes a missing value that you can then impute on purpose. The mistake to avoid is treating the boundary values as safe. A rule of 18 to 110 keeps a 110-year-old who is probably a typing error, while a rule of 0 to 120 keeps a 0 that might be a placeholder. Write the rule with a source or a reason next to it. Validation rules are written as code in an ingestion step, and a batch that fails too many rules is held back rather than loaded into the warehouse.

---

### Outliers

An outlier is a value far from the bulk of the data. Imagine a classroom where most students are between 15 and 17 years old and one is 45. The 45-year-old may be a teacher who was counted by mistake, or may be a real and interesting student.
From first principles, the common detection rules are the interquartile range (IQR) rule and the z-score. The IQR rule marks values below the first quartile minus 1.5 times the IQR, or above the third quartile plus 1.5 times the IQR. The z-score measures how many standard deviations a value sits from the mean, which assumes a roughly bell-shaped distribution. Both are sensitive to the choice of threshold, and the IQR rule is safer on skewed data such as income.

```python
import numpy as np
import pandas as pd

income = pd.Series([50_000, 62_000, 48_000, 90_000, 55_000, 5_000_000])

q1, q3 = income.quantile([0.25, 0.75])
iqr = q3 - q1
low, high = q1 - 1.5 * iqr, q3 + 1.5 * iqr
is_outlier = (income < low) | (income > high)

print(is_outlier.sum())           # 1, the 5 million value
winsorized = income.clip(lower=low, upper=high)
print(winsorized.max())           # high fence, not 5,000,000
```

Two choices appear here. Clipping keeps the row and limits its influence, which is winsorizing. Dropping the row removes it entirely. Clipping is usually safer, because it preserves the record while stopping one value from dominating a mean or a linear model. The mistake is to remove outliers from the test set and keep them in the training set, which makes the test look cleaner than the real data. Apply the same fences to every split, and compute the fences from the training data only. Outlier handling matters in fraud detection, where the outliers are the target, and removing them would delete the very cases you want to catch.

---

### Data types

A data type decides what operations are valid. A column of numbers stored as text will sort alphabetically, so `"9"` comes after `"10"`. A postal code stored as an integer loses its leading zero. A date stored as a string cannot be subtracted to get the number of days. Type errors are quiet. The code runs and the answer is wrong.

From first principles, every column should have the dtype that matches its meaning. Use `astype` or `pd.to_numeric` for numbers, `pd.to_datetime` with an explicit format for dates, `astype("string")` for text that may be missing, and `astype("category")` for a small set of labels such as a plan name. Choosing an explicit date format is safer than letting pandas guess, because an ambiguous string like `03/04/2024` means different dates in different countries.

```python
import pandas as pd

raw = pd.DataFrame({
    "signup": ["2024-03-01", "01/03/2024", "not a date"],
    "zip": ["00123", "02110", "10001"],
    "plan": ["pro", "basic", "pro"],
})

iso = pd.to_datetime(raw["signup"], format="%Y-%m-%d", errors="coerce")
dmy = pd.to_datetime(raw["signup"], format="%d/%m/%Y", errors="coerce")
raw["signup_date"] = iso.fillna(dmy)

raw["zip"] = raw["zip"].astype("string")          # keep the leading zeros
raw["plan"] = raw["plan"].astype("category")

print(raw.dtypes)
print(raw)
```

The `errors="coerce"` argument turns anything that fails to parse into a missing value, so you can count the failures instead of stopping on the first one. The common mistake is to convert a column with `astype(float)` and see an error that you silence by coercing everything to missing. Count the coerced values before and after the conversion. Date and identifier columns are where most silent corruption happens, which is why a good ingestion step declares the schema explicitly.

---

### Leakage

Leakage happens when the model learns from information it would not have at prediction time. It is the most dangerous problem in this lesson, because it produces a score that looks excellent and then fails in production. Think of a student who sees the answer key before the exam. The score is high, and the student has learned nothing about the subject.

Leakage comes in two common forms. Target leakage happens when a feature is recorded after the outcome, such as a "cancellation reason" field that is filled in only after a customer leaves. Preprocessing leakage happens when a step learns from the whole dataset before the split, such as a scaler or an imputer fitted on all rows, including the test rows.

```python
import numpy as np
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

rng = np.random.default_rng(0)
X = rng.normal(size=(200, 3))
X[rng.random(X.shape) < 0.1] = np.nan
y = (X[:, 0] > 0).astype(int)

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.25, random_state=0)

model = make_pipeline(SimpleImputer(strategy="median"), StandardScaler(),
                      LogisticRegression(max_iter=1000))
model.fit(X_train, y_train)           # imputer and scaler learn from training rows only
print(model.score(X_test, y_test))
```

Putting the imputer and scaler inside the pipeline means `fit` learns their statistics from the training rows, and `score` applies them to the test rows without refitting. The mistake this prevents is common: calling `fillna(X.median())` on the full table and only then splitting. The fix is to split first and fit preprocessing second. Leakage is the usual reason a model that scored 0.95 in a notebook scores 0.60 in production, and a pipeline object is the standard protection against it.

---

### Label errors

A label is the answer a model is trained to predict. Labels come from people, from rules or from older systems, and each source makes mistakes. Benchmark datasets that are used to compare models have been shown to contain a measurable share of wrong labels, which means that a model's score is partly a measure of how well it matches the errors.

From first principles, one way to find suspicious labels is to train a model on the data with cross-validation, so that each example is predicted by a model that never saw it. If the model gives the recorded label a very low probability, the label is a candidate error. This is the core idea behind confident learning, which the open-source library cleanlab implements. The library is licensed under AGPL-3.0 for its open-source version, so check the license before you use it in a commercial product.

```python
import numpy as np
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import StratifiedKFold, cross_val_predict
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

def label_suspects(X: np.ndarray, y: np.ndarray, threshold: float = 0.1) -> np.ndarray:
    model = make_pipeline(SimpleImputer(strategy="median"), StandardScaler(),
                          LogisticRegression(max_iter=1000))
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)
    proba = cross_val_predict(model, X, y, cv=cv, method="predict_proba")
    given = proba[np.arange(len(y)), y]     # probability the model gives the recorded label
    return given < threshold
```

The imputer sits inside the pipeline, so the cross-validation folds do not leak. The output is a flag that marks a row for review. A flagged row may be an error, or it may be a hard example that the model has not learned yet. Look at flagged rows by hand before you change any label. The mistake is to relabel everything the function flags, which can remove the rare cases that the model was supposed to learn. Label review is a manual step where a domain expert decides, and a tool like this only sorts the review queue.

---

### Data quality

Data quality is the measured state of a dataset along named dimensions. The usual set includes completeness (how much is present), validity (how much follows the rules), uniqueness (how much is not repeated), consistency (how much agrees across tables and fields), accuracy (how much matches the real world, which is the hardest to measure) and timeliness (how current it is). Each dimension gets a number, and the numbers are tracked over time.

The point of a quality report is to turn a vague worry into a checklist. "The data looks fine" describes a feeling. "Income is missing for 2.1 percent of rows, 14 ages fail the validity rule, and the signup date parses for 98.5 percent of records" is a measurement that another person can reproduce. Tools such as Great Expectations, an open-source Python library under the Apache 2.0 license, and pandera, an open-source library for DataFrame schemas, let you write these checks as code. Use them in a real project, but learn the checks in plain pandas first.

```python
import pandas as pd

def quality_report(df: pd.DataFrame) -> dict:
    return {
        "rows": len(df),
        "null_rate": df.isna().mean().round(4).to_dict(),
        "duplicate_ids": int(df["customer_id"].duplicated().sum()),
    }
```

The mistake is to produce a report once and never again. A quality number is useful when it is compared with the last batch, so the report should be stored with its date and the data version.

---

## Hands-on exercises

Work through these in a notebook first, then move them into the project.

1. **Missingness pattern.** Create a DataFrame with 200 rows, a numeric column and a binary label. Remove 15 percent of the numeric values at random, then remove another 15 percent only from rows where the label is 1. Compare the null rate by label. Explain in one paragraph what kind of missingness you created.
2. **Key-based duplicates.** Take a table of 50 orders with an order ID and some other fields. Copy 5 rows and change one field in each copy. Show that `duplicated()` on all columns misses them, and that `duplicated(subset="order_id")` catches them. Keep the most recent record per key.
3. **Rule table.** Write a table of at least five validation rules for a dataset you choose, each with a column, a test and a one-line reason. Implement each as a boolean column. Count the failures per rule.
4. **IQR fences.** Compute the IQR fences for a skewed income column. Count how many values fall outside them. Winsorize the column and compare the mean before and after. Explain why the mean moved.
5. **Date parsing.** Build a column with dates in three formats and one impossible value. Parse it with explicit formats and `errors="coerce"`. Print the number of values that failed. Find the cause of each failure.
6. **Leakage experiment.** Fit a scaler on the full dataset, then on the training split only, and compare the test score of a logistic regression under each. Explain whether the difference is large enough to matter on your data.
7. **Label check.** Introduce 10 flipped labels into a synthetic binary dataset. Run `label_suspects` and count how many of the 10 flipped rows are flagged, and how many clean rows are flagged too. Report both numbers.

---

## Project: a data cleaning audit with a quality report

### Goal

Build a cleaning module that takes a messy customer file, applies a documented set of rules, and writes two outputs: a cleaned dataset and a data quality report that records every decision with its count. The project uses a seeded generator, so the messy file is reproducible.

Suggested repository name: `100-days-ai-day-19-data-cleaning`

### Repo layout

```text
100-days-ai-day-19-data-cleaning/
├── README.md
├── cleaning/
│   ├── __init__.py
│   ├── generate.py
│   ├── clean.py
│   └── report.py
├── tests/
│   └── test_clean.py
├── outputs/
│   └── .gitkeep
├── requirements.txt
└── .gitignore
```

### Starter code

`requirements.txt`:

```text
pandas>=2.2
numpy>=2.0
scikit-learn>=1.5
pytest>=8.0
```

`cleaning/generate.py`:

```python
import numpy as np
import pandas as pd


def make_messy_customers(n: int = 300, seed: int = 7) -> pd.DataFrame:
    rng = np.random.default_rng(seed)
    tenure = rng.integers(0, 120, size=n)
    churn_prob = 0.25 + 0.2 * (tenure < 6)
    df = pd.DataFrame({
        "customer_id": [f"c{i:04d}" for i in range(n)],
        "age": rng.integers(18, 80, size=n).astype(float),
        "income": np.round(rng.lognormal(mean=10.5, sigma=0.4, size=n), 2),
        "tenure_months": tenure,
        "signup_date": pd.date_range("2022-01-01", periods=n, freq="D").strftime("%Y-%m-%d"),
        "plan": rng.choice(["basic", "plus", "pro"], size=n),
        "churned": (rng.random(n) < churn_prob).astype(int),
    })
    df.loc[rng.choice(n, 20, replace=False), "age"] = np.nan
    df.loc[rng.choice(n, 5, replace=False), "age"] = -1.0
    df.loc[rng.choice(n, 5, replace=False), "age"] = 230.0
    spikes = rng.choice(n, 6, replace=False)
    df.loc[spikes, "income"] = df.loc[spikes, "income"] * 100
    flips = rng.choice(n, 12, replace=False)
    df.loc[flips, "churned"] = 1 - df.loc[flips, "churned"]
    df = pd.concat([df, df.sample(10, random_state=seed)], ignore_index=True)
    day_fmt = rng.choice(len(df), 30, replace=False)
    df.loc[day_fmt, "signup_date"] = (
        pd.to_datetime(df.loc[day_fmt, "signup_date"]).dt.strftime("%d/%m/%Y")
    )
    return df
```

`cleaning/clean.py`:

```python
import numpy as np
import pandas as pd


AGE_MIN, AGE_MAX = 18, 110


def drop_duplicate_ids(df: pd.DataFrame) -> pd.DataFrame:
    return df.drop_duplicates(subset="customer_id", keep="last")


def clean_age(df: pd.DataFrame) -> pd.DataFrame:
    out = df.copy()
    valid = out["age"].between(AGE_MIN, AGE_MAX)
    out["age_invalid"] = out["age"].notna() & ~valid
    out["age"] = out["age"].where(valid)
    return out


def parse_signup(df: pd.DataFrame) -> pd.DataFrame:
    out = df.copy()
    iso = pd.to_datetime(out["signup_date"], format="%Y-%m-%d", errors="coerce")
    dmy = pd.to_datetime(out["signup_date"], format="%d/%m/%Y", errors="coerce")
    out["signup_date"] = iso.fillna(dmy)
    return out


def flag_income_outliers(df: pd.DataFrame) -> pd.DataFrame:
    out = df.copy()
    q1, q3 = out["income"].quantile([0.25, 0.75])
    iqr = q3 - q1
    low, high = q1 - 1.5 * iqr, q3 + 1.5 * iqr
    out["income_outlier"] = (out["income"] < low) | (out["income"] > high)
    out["income"] = out["income"].clip(lower=low, upper=high)
    return out


def cast_types(df: pd.DataFrame) -> pd.DataFrame:
    out = df.copy()
    out["customer_id"] = out["customer_id"].astype("string")
    out["plan"] = out["plan"].astype("category")
    out["churned"] = out["churned"].astype("int8")
    return out


def clean(df: pd.DataFrame) -> pd.DataFrame:
    out = drop_duplicate_ids(df)
    out = clean_age(out)
    out = parse_signup(out)
    out = flag_income_outliers(out)
    return cast_types(out)
```

`cleaning/report.py`:

```python
import pandas as pd


def quality_report(raw: pd.DataFrame, clean: pd.DataFrame) -> dict:
    return {
        "rows_raw": len(raw),
        "rows_clean": len(clean),
        "duplicate_ids_removed": int(raw["customer_id"].duplicated().sum()),
        "age_invalid_flagged": int(clean["age_invalid"].sum()),
        "age_missing_after_clean": int(clean["age"].isna().sum()),
        "signup_unparsed": int(clean["signup_date"].isna().sum()),
        "income_outliers_flagged": int(clean["income_outlier"].sum()),
        "churn_rate": round(float(clean["churned"].mean()), 4),
    }
```

`tests/test_clean.py`:

```python
import pandas as pd
from cleaning.generate import make_messy_customers
from cleaning.clean import clean, drop_duplicate_ids, parse_signup
from cleaning.report import quality_report


def test_duplicate_ids_are_removed():
    raw = make_messy_customers()
    out = drop_duplicate_ids(raw)
    assert out["customer_id"].is_unique


def test_invalid_ages_become_missing_and_are_flagged():
    raw = make_messy_customers()
    out = clean(raw)
    assert out["age"].dropna().between(18, 110).all()
    assert out["age_invalid"].sum() >= 1


def test_both_date_formats_parse():
    raw = make_messy_customers()
    out = parse_signup(raw)
    assert out["signup_date"].notna().all()


def test_income_outliers_are_clipped_not_dropped():
    raw = make_messy_customers()
    out = clean(raw)
    assert len(out) == raw["customer_id"].nunique()
    assert out["income_outlier"].sum() >= 1


def test_report_counts_are_consistent():
    raw = make_messy_customers()
    out = clean(raw)
    report = quality_report(raw, out)
    assert report["rows_clean"] == raw["customer_id"].nunique()
    assert report["duplicate_ids_removed"] == len(raw) - report["rows_clean"]
```

### Step-by-step tasks

1. Create the repository, the virtual environment and the requirements install.
2. Create the files as shown. Run `pytest -q` and make sure all five tests pass.
3. In a script, generate the messy file, run `clean`, and print `quality_report(raw, clean_df)`. Save the output as `outputs/quality_report.txt` by writing the dictionary's items, one per line.
4. Count the rows in `raw` where `signup_date` is in the day-first format. Use `pd.to_datetime` with each format and compare the counts. You should find exactly 30 day-first rows, because an ISO string such as `2022-01-05` can never parse as day-first, so no row matches both formats.
5. Find the six rows with inflated income. Confirm that they are flagged, that their clipped values are below the upper fence, and that the rows still exist.
6. Add the `label_suspects` function from this lesson in a new file `cleaning/labels.py`. Run it on the cleaned data with `X = clean_df[["age", "income", "tenure_months"]].to_numpy(dtype=float)` and `y = clean_df["churned"].to_numpy()`. Count the flagged rows and write one paragraph on whether the flags look like the 12 flipped labels. The generator flips 12 labels among the original 300 customers before it appends duplicates, so the flipped labels are in the data, but the file does not say which rows they are. To recover them, add a line to the generator that saves a copy of the `churned` column before the flip, and compare the two columns.
7. Write a `docs/decisions.md` file, or a section of the README, listing each rule, its reason, and the count it affected. Each row should match a number in your quality report.

### README checklist

- [ ] Install and run instructions with the versions you tested
- [ ] The command that generates the messy file and the command that writes the report
- [ ] The quality report output, pasted as text
- [ ] A table of cleaning rules with the reason for each and the count it affected
- [ ] The label check results, including how many flagged rows were real flips
- [ ] The leakage risk you avoided, and how the pipeline prevents it
- [ ] One limitation, for example that clipping outliers changes the income distribution

### Done when

- `pytest -q` passes all five tests
- The quality report counts agree with the tests and with your hand checks
- Each cleaning rule has a stated reason and a count
- The six inflated incomes are flagged and clipped, not removed
- The label check runs and the README explains what its flags mean and do not mean
- The repository has at least four commits, including one commit that adds the label check after the cleaning rules

### Evidence to commit to GitHub

Commit the package, the tests, the README with the decisions table and the quality report output. Do not commit large data files, but commit `outputs/quality_report.txt` so the reader can compare it with a fresh run. The reader should be able to show a green `pytest -q` run, the report, and a short paragraph on which rule they changed their mind about and why.

---

## Check your understanding

1. A column has 4 percent missing values, and the missing values are concentrated in customers who later churned. Explain why dropping those rows biases the model, and describe one feature you could add to record the gap.
2. Your dataset has no exact duplicate rows, yet two records describe the same order. What is the likely cause, and how would you choose the key that finds them?
3. A colleague removes every row with an income above the 99th percentile and says the outliers are gone. Name two things that this removal changes and one case where it would be the wrong thing to do.
4. A model scores 0.97 on a test split after you imputed the missing values on the full table. Describe the mechanism that could inflate the score, and say what you would check first.
5. A feature called `refund_issued` is 1 for almost every churned customer and 0 for nearly all others. Is this a strong feature or a leak? Explain the difference in terms of when the value is known.
6. You run a label check and it flags 9 percent of the rows. Give two explanations for the high rate and describe how you would decide between them without relabeling everything.

Work each answer out in writing before you read the answers below.

**Answers**

1. The gap is likely related to churn itself, so the rows are not missing at random. Dropping them removes churners unevenly and changes the class balance and the relationships the model learns. Keep the rows, add an indicator column such as `income_missing`, and use it as a feature if the gap predicts the target.
2. The key you chose does not identify the real-world thing. The two records may differ by a status update, a timestamp or a typo in a string such as a trailing space or different capitalization. Normalize the strings first, then check duplicates on a key built from business identity, such as order ID with customer ID and order date. Inspect a sample by hand before you drop anything.
3. Removing the top 1 percent changes the tail of the distribution, lowers the mean and variance, and removes the cases the model most needs to see. In an income task, the high earners are often the positive class, so the model would learn to be unsure about them. It would be the wrong choice when the outliers are the phenomenon you want to predict, as with fraud or high-value customers.
4. The most likely cause is a preprocessing step, such as a median or a scaler, fitted before the split, so test-row information entered training. Check the order of operations first. Move every fitted step into a pipeline and rerun the evaluation. Also check for the same customer appearing in both train and test, which inflates scores in a different way.
5. It is a leak if the value is written after the churn decision, for example a refund that is issued because the customer cancelled. A feature is valid only if it is known at the moment the prediction would be made. Compare the timestamp of the feature with the timestamp of the outcome before you decide.
6. Two explanations are common: the model cannot yet separate those examples, which means the features are missing signal, or the labeling rule differs from the one the model was given. To decide without relabeling everything, spot-check about 30 flagged rows by hand, check whether the flags cluster by date or by source, which would point to a pipeline or labeling batch, and see whether the flag rate falls when you use a stronger model.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "Data cleansing", for the overview of cleaning steps and their purpose.
- Wikipedia, article "Missing data", for the three patterns of missingness and why they matter.
- Wikipedia, article "Outlier", for the definitions and detection methods.
- Wikipedia, article "Data quality", for the dimensions used in the quality report.
- Wikipedia, article "Leakage (machine learning)", for the formal description of target and preprocessing leakage.
- scikit-learn documentation, User Guide sections on "Imputation of missing values" and "Common pitfalls and recommended practices", which covers leakage. Search "scikit-learn common pitfalls" on scikit-learn.org.
- Northcutt, Jiang and Chuang, "Confident Learning: Estimating Uncertainty in Dataset Labels". The paper was first posted to arXiv in 2019 and published in the *Journal of Artificial Intelligence Research* in 2021. Search the title to find it. Read the introduction and the method summary.

**Tutorials**

- GeeksforGeeks, "Data Cleaning in Python" and the articles on handling outliers and handling missing values.
- W3Schools, "Pandas Cleaning Data" in the Python section of w3schools.com.

**Watch**

- StatQuest with Josh Starmer, on YouTube, for the videos on bias, variance and the train and test split, which explain why leakage hurts.
- Search YouTube for "data leakage in machine learning" and choose a talk that uses a real example.
- DeepLearning.AI, "Data-Centric AI" short course, on the DeepLearning.AI platform. It covers label quality and data cleaning with a hands-on notebook.

**Practice**

- Kaggle Learn, "Data Cleaning" micro-course, free with a Kaggle account, with browser notebooks and immediate feedback.

---

## Tomorrow

Tomorrow is exploratory data analysis. Today you decided which rows were trustworthy and recorded every decision. Day 20 asks what the cleaned data actually says: how each variable is distributed, which features move together, how imbalanced the labels are, and what patterns deserve a hypothesis before any model is trained. The quality report you built today is the starting point for that investigation.

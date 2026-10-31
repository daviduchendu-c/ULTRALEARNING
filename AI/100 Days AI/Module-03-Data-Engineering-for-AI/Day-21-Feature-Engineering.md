# Day 21: Feature Engineering

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 4 hours
**Level:** intermediate. You need the cleaning and EDA habits from Days 19 and 20 and the basics of a train and test split. The project trains a small model, so a working scikit-learn install is part of the day.

**By the end of this day, you will have:**

- A working method for turning raw columns into model inputs: numerical transformations, categorical encoding, scaling, normalization, interactions and feature selection
- A reusable scikit-learn preprocessing pipeline that is fitted inside cross-validation, so that it does not leak information
- A written comparison of model scores with and without engineered features, with the reasons for each feature kept or dropped

**What you need:**

- A computer with Python 3.10 or newer
- pandas 2.2 or newer, NumPy 2.0 or newer
- scikit-learn 1.5 or newer (`pip install "scikit-learn>=1.5"`), free under the BSD 3-Clause license
- pytest 8.0 or newer (`pip install "pytest>=8.0"`)
- A code editor such as VS Code (free) or Jupyter Lab
- Git and a free GitHub account for the project section
- The `make_subscriptions` function from Day 20, which the project copies into its own repository

**Prerequisites:**

- Day 19: missing values, outliers and leakage, because feature code is where leakage most often enters
- Day 20: distributions, skewness, correlations and the class balance
- Day 14: the mean and standard deviation, and what standardization means

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Numerical features](#numerical-features)
- [Categorical encoding](#categorical-encoding)
- [Scaling](#scaling)
- [Normalization](#normalization)
- [Transformations](#transformations)
- [Feature interactions](#feature-interactions)
- [Feature selection](#feature-selection)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a leak-free feature pipeline with a measured gain](#project-a-leak-free-feature-pipeline-with-a-measured-gain)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A model reads numbers. Your data contains plans, dates, free text, counts, prices and missing entries. Feature engineering is the work of translating the real-world information in the table into numbers a model can use, in a form that makes the pattern easy to learn. It is often the step that decides how well a model performs. A simple model with well-chosen features frequently beats a complicated model with raw columns.

The work has two halves. One half is representation: deciding what a number should mean, such as "tickets per month" rather than "total tickets", or "is this customer on the pro plan" rather than the word "pro". The other half is discipline: making sure every transformation is learned from the training data and applied unchanged to the test data and to live data. The second half is where most production bugs come from.

On the job, feature engineering takes up a large share of a machine learning engineer's time. Feature code is shared between training jobs and serving systems, so a feature must be computed the same way in both places. Teams that maintain a feature store, such as Feast, an open-source project under the Apache 2.0 license, are solving exactly this problem at scale. Today you build the same pattern on a small scale.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1930s to 1964 | Statisticians use logarithms and other power transforms to make skewed data closer to symmetric. Box and Cox publish a parametric power family in 1964. | Log transforms are still the first move for skewed positive variables. |
| 1960s | Dummy coding, which turns a category into a set of 0 and 1 columns, becomes standard in regression analysis. | It is the origin of one-hot encoding. |
| 1990s | Data mining textbooks treat feature construction and feature selection as central tasks, separate from the choice of algorithm. | The idea that data preparation matters as much as the model is established here. |
| 2000s to 2010s | Libraries such as scikit-learn put preprocessing into reusable transformer objects that can be chained into pipelines. | Leak-free pipelines become a routine part of practice. |
| 2010s to today | Feature stores and feature platforms appear in large companies, so one feature definition serves both training and serving. | Feature code moves from notebooks into shared infrastructure. |

**Takeaway:** the transformations are old and well understood. The modern challenge is to apply them the same way every time, in every place the model runs.

---

## Glossary

- **Feature**: an input column, or a value derived from one or more input columns, that a model uses to predict.
- **Numerical feature**: a feature that holds quantities on a numeric scale.
- **Categorical feature**: a feature that holds a label from a fixed set of values, such as a plan name.
- **One-hot encoding**: turning a category into one binary column per value, with a 1 in the column for the row's value and 0 elsewhere.
- **Ordinal encoding**: turning a category into integers that respect a natural order, such as small, medium and large.
- **Standardization**: subtracting the mean and dividing by the standard deviation, so the result has mean 0 and standard deviation 1.
- **Normalization**: rescaling values into a fixed range, such as 0 to 1, or rescaling each row to unit length. The word is used in several senses, so check which one a function applies.
- **Log transform**: replacing a value with its logarithm. The `log1p` version computes the logarithm of one plus the value, which handles zeros.
- **Interaction**: a feature that multiplies or combines two or more features, so the model can learn that the effect of one depends on another.
- **Feature selection**: choosing a subset of features to keep, based on their usefulness or on their redundancy.
- **Pipeline**: a sequence of transformations and a model, joined into one object that is fitted and applied as a unit.
- **Mutual information**: a measure of how much knowing one variable reduces the uncertainty about another. It is zero for independent variables.

---

### Numerical features

A numerical feature is a number that already has a meaning. Think of a car's odometer reading. The number is useful, but a rate such as miles per day or fuel per trip often says more about how the car is used. Feature engineering starts by asking which quantities matter and whether the raw number or a derived number carries the signal.

From first principles, a numerical feature can be used as it is, combined with others into a ratio, counted, binned into ranges, or passed through a function. Ratios are often the most informative. A count of support tickets says little about a customer who has been with the company for a day and a lot about one who has been with it for three years, so `tickets / tenure` captures the rate. Binning turns a continuous value into bands, which helps when the relationship with the target is not a straight line.

```python
import numpy as np
import pandas as pd

df = pd.DataFrame({
    "support_tickets": [0, 2, 5, 1],
    "tenure_months": [24, 3, 60, 1],
})

df["tickets_per_month"] = df["support_tickets"] / df["tenure_months"].clip(lower=1)
df["tenure_band"] = pd.cut(df["tenure_months"], bins=[0, 6, 24, np.inf],
                           labels=["new", "established", "long-term"])
print(df)
```

The common mistake is to create a ratio whose denominator can be zero, or to create a ratio from a column that is missing for many rows. The clip on the denominator is a simple guard, and the missing values need a rule of their own. A second mistake is to bin too finely, which gives the model many small groups with little data in each. Rates and recency features are central to churn models, fraud models and recommendation systems, because the raw totals grow with time and hide the behavior.

---

### Categorical encoding

A model cannot multiply a plan name by a weight. It needs numbers. Encoding is the translation from categories to numbers. The choice of encoding matters because a poor one can give the model a false sense of order or make it blind to a category that matters.

From first principles, one-hot encoding is the safe default for nominal categories, where the labels have no natural order. Each category becomes its own binary column, so the model learns a separate effect for each. Ordinal encoding is appropriate only when the categories have a real order, such as service tiers that are known to rank from smallest to largest. Encoding a nominal category as integers, such as basic equals 0, plus equals 1 and pro equals 2, tells the model that pro is twice plus, which is usually false.

```python
import pandas as pd
from sklearn.preprocessing import OneHotEncoder

plans = pd.DataFrame({"plan": ["basic", "plus", "pro", "plus"]})

encoder = OneHotEncoder(handle_unknown="ignore")
encoded = encoder.fit_transform(plans)
print(encoder.get_feature_names_out())    # ['plan_basic' 'plan_plus' 'plan_pro']
print(encoded.toarray())

new_plan = pd.DataFrame({"plan": ["enterprise"]})
print(encoder.transform(new_plan).toarray())   # all zeros, no error
```

The `handle_unknown="ignore"` argument matters in production. A new category, such as a plan launched next month, produces a row of zeros instead of an error that stops the service. The row then falls back to the baseline for all known categories, which is a reasonable default, but you should log how often it happens. The mistake to avoid is one-hot encoding a column with thousands of distinct values, such as a free-text city field, because the result has thousands of columns and most of them are rarely used. For high-cardinality categories, use target encoding with careful cross-validation, or group the rare values into an "other" category first. The encoder is part of the saved model, because the category list must match the one used in training.

---

### Scaling

Scaling changes the units of a numerical feature so that features measured in different units are comparable. A feature measured in thousands of currency units and a feature measured in counts of 0 to 10 are on very different ranges. Many algorithms, including logistic regression, support vector machines and neural networks, treat a large-valued feature as more important simply because its numbers are bigger, unless they are scaled.

From first principles, standardization computes `z = (x - mean) / std` using the training set's mean and standard deviation. The result has mean 0 and standard deviation 1 on the training data. Tree-based models such as random forests and gradient boosting are less sensitive to scale, because they split on thresholds, but the split points are still easier to interpret on scaled values.

```python
import numpy as np
from sklearn.preprocessing import StandardScaler

X_train = np.array([[1000.0, 2.0], [2000.0, 4.0], [3000.0, 6.0]])
X_test = np.array([[2500.0, 5.0]])

scaler = StandardScaler().fit(X_train)     # learn mean and std from training rows only
print(scaler.mean_)                        # [2000. 4.]
print(scaler.transform(X_test).round(4))   # [[0.6124 0.6124]]
```

`fit` must see only the training rows. If you fit the scaler on the full table, the test set's mean and spread leak into the training features. The common mistake is to call `fit_transform` on everything before splitting. The pipeline pattern shown in Day 19 avoids this automatically. Also note that a scaler does not change the shape of a skewed distribution. A feature with a long right tail stays skewed after standardization, which is why the log transform below is often needed too. In practice, standardized inputs are the default for linear and neural models, and the scaler's learned values are saved with the model so that the live system uses the same numbers.

---

### Normalization

Normalization is a word used in several ways, so be precise. In this lesson it means rescaling values into a fixed range, most often from 0 to 1 using the minimum and maximum of the training data. Think of rescaling a set of exam marks so that the lowest possible score is 0 and the highest is 1. The shape of the distribution is kept, but the range is fixed.

From first principles, min-max scaling computes `x' = (x - min) / (max - min)` with the training minimum and maximum. It is sensitive to outliers, because a single extreme value sets the maximum and squeezes all other values toward 0. A separate meaning of normalization is row-wise scaling to unit length, which is common for text vectors and embeddings, where the direction of a vector matters more than its size.

```python
import numpy as np
from sklearn.preprocessing import MinMaxScaler, normalize

X = np.array([[1.0, 100.0], [2.0, 300.0], [4.0, 500.0]])

mm = MinMaxScaler().fit(X)
print(mm.transform(X))                     # each column now runs from 0 to 1

rows = normalize(X, norm="l2")             # each row has length 1
print(np.linalg.norm(rows, axis=1))        # [1. 1. 1.]
```

The mistake is to apply min-max scaling to a feature with a single extreme outlier and then wonder why the other values are all bunched near zero. Clip or winsorize the outlier first, as in Day 19. The second mistake is to assume that "normalize" means the same thing in every library. The scikit-learn function `normalize` works row by row, while `MinMaxScaler` works column by column. Min-max scaling is common for image pixels and for neural network inputs with bounded ranges, while unit-length normalization is common for embeddings used in similarity search.

---

### Transformations

A transformation changes the shape of a distribution as well as its scale. The most common one is the logarithm, which compresses a long right tail so that large values do not dominate. Think of a bank statement that lists small purchases and one very large one. On a log scale, the large purchase is no longer thousands of times bigger than the small ones, just a few steps further along.

From first principles, the log transform replaces `x` with `log(x)`, and it is defined only for positive values. The `log1p` version computes `log(1 + x)`, which is defined at zero and is safe for counts and spending values that can be zero. Other options include the square root, which is milder, and the Box-Cox and Yeo-Johnson transforms, which search for the power that makes the distribution most symmetric.

```python
import numpy as np
import pandas as pd

spend = pd.Series([0.0, 9.0, 19.0, 49.0, 480.0, 5000.0])

print(np.log1p(spend).round(2).tolist())   # the large values shrink toward the rest
print(np.sqrt(spend).round(2).tolist())    # a milder compression
```

Check the effect of a transformation by plotting the distribution before and after, as you learned in Day 20. The mistake is to apply a log to a variable that contains negative values, which gives `NaN` or an error, or to interpret a coefficient from a log-transformed model as if it were on the original scale. A one-unit change in `log1p(x)` is not a one-unit change in `x`. Log transforms are standard for prices, incomes, page views and any quantity that grows multiplicatively.

---

### Feature interactions

An interaction is a feature that combines two others, so that the model can learn an effect that depends on both. Picture a recipe where salt and heat each have an effect on flavor, but the effect of salt depends on how hot the pan is. A linear model that only adds the two ingredients cannot capture that dependence unless you give it the product.

From first principles, the simplest interaction is the product of two numerical features, such as `tenure * tickets`. The model then learns a weight for that product. Interactions can also be built from a numerical and a categorical feature, such as a separate slope for each plan, which is the form most useful in practice. Tree-based models find many interactions automatically, which is one reason they often do well on tabular data without manual work.

```python
import pandas as pd

df = pd.DataFrame({
    "tickets_per_month": [0.0, 0.2, 1.0],
    "plan": ["basic", "plus", "pro"],
})

df["tickets_x_pro"] = df["tickets_per_month"] * (df["plan"] == "pro").astype(float)
print(df)
```

The mistake is to create many interactions without a reason, which multiplies the number of features and invites overfitting. Create an interaction when there is a reason to believe the effect depends on the second feature, and check it on the validation data. A second caution is that interactions can be hard to interpret. The interactions that survive are usually ones that a domain expert can explain, such as a discount that matters only for new customers.

---

### Feature selection

Feature selection chooses which features to keep. More features do not always help. Each additional column adds noise that a model may fit by chance, adds computation at serving time and makes the model harder to explain. Selection is the discipline of keeping what helps and documenting what was removed.

From first principles, three families of methods exist. Filter methods score each feature on its own, using a statistic such as correlation or mutual information with the target. Wrapper methods train models on subsets of features and keep the subset that scores best, which is accurate but expensive. Embedded methods, such as L1-regularized regression, let the model shrink weak features to zero during training. Mutual information measures how much one variable tells you about another, and it captures nonlinear dependence that correlation can miss.

```python
import numpy as np
from sklearn.feature_selection import mutual_info_classif

rng = np.random.default_rng(0)
x_useful = rng.normal(size=500)
x_noise = rng.normal(size=500)
y = (x_useful + 0.5 * rng.normal(size=500) > 0).astype(int)
X = np.column_stack([x_useful, x_noise])

scores = mutual_info_classif(X, y, random_state=0)
print(scores.round(3))            # the useful column scores clearly higher
```

The common mistake is to select features using the full dataset, including the rows that will be used for testing. Selection is part of the model, so it belongs inside the cross-validation loop. A second mistake is to drop a feature because it is correlated with another without asking which of the two carries more signal for the target. Feature selection reduces the cost of serving a model and makes it easier to defend to a regulator or a customer who asks why a decision was made.

---

## Hands-on exercises

Work in a notebook. Before each step, write down what you expect the output to show.

1. **Ratio feature.** Using a dataset with a count and an exposure time, create a rate feature. Show a case where the raw count and the rate rank customers differently. Explain which ranking is more useful for a churn model and why.
2. **Encoding trap.** Encode a three-level plan column as integers 0, 1 and 2. Fit a logistic regression with that encoding and with one-hot encoding on the same data. Compare the coefficients and the cross-validated AUC. Explain the difference in one paragraph.
3. **Scaling leak.** Fit a `StandardScaler` on the full dataset and then on the training split only. Compare the two sets of means. Explain which one a deployed model could reproduce.
4. **Transformation effect.** Take a right-skewed column of positive values. Compute its skewness before and after `np.log1p`. Plot the histograms and state the change in one sentence.
5. **Interaction test.** Create a synthetic target where the effect of a feature exists only for one group. Fit a model without the interaction and one with it. Compare the cross-validated scores.
6. **Selection inside CV.** Use `SelectKBest` with mutual information inside a pipeline, and cross-validate the whole pipeline. Compare with selecting once on the full data before cross-validation. Note the difference in the score.
7. **Unknown category.** Fit a one-hot encoder on three categories. Transform a fourth category and check that the output is all zeros. Write a sentence on how a serving system would detect and log this.

---

## Project: a leak-free feature pipeline with a measured gain

### Goal

Build a feature pipeline for the synthetic subscription data from Day 20. The pipeline includes a log transform, scaling, one-hot encoding and one engineered ratio. Measure whether the engineered feature improves the cross-validated AUC, and write up the result with the evidence. Every transformation is learned inside the cross-validation loop, so the score is honest.

Suggested repository name: `100-days-ai-day-21-features`

### Repo layout

```text
100-days-ai-day-21-features/
├── README.md
├── features/
│   ├── __init__.py
│   ├── generate.py
│   ├── build.py
│   └── evaluate.py
├── tests/
│   └── test_build.py
├── run_features.py
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

`features/generate.py`: copy the `make_subscriptions` function from your Day 20 repository, unchanged, into this file. Keep the planted bug from Day 20 in it, because the pipeline should handle it. If you do not have Day 20, use the function below, which reproduces the same relationships.

```python
import numpy as np
import pandas as pd


def make_subscriptions(n: int = 3000, seed: int = 3) -> pd.DataFrame:
    rng = np.random.default_rng(seed)
    tenure = np.clip(np.round(rng.gamma(shape=2.0, scale=12.0, size=n)), 1, 72)
    plan = rng.choice(["basic", "plus", "pro"], p=[0.5, 0.35, 0.15], size=n)
    price_map = {"basic": 9.0, "plus": 19.0, "pro": 49.0}
    price = np.array([price_map[p] for p in plan])
    spend = price * rng.uniform(0.9, 1.2, size=n)
    tickets = rng.poisson(lam=np.where(plan == "basic", 2.0, 1.0))
    logit = -2.5 - 0.04 * tenure + 0.5 * tickets
    churned = (rng.random(n) < 1 / (1 + np.exp(-logit))).astype(int)
    signup = pd.Timestamp("2024-01-01") + pd.to_timedelta(rng.integers(0, 365, size=n), unit="D")
    df = pd.DataFrame({
        "customer_id": [f"s{i:05d}" for i in range(n)],
        "signup_date": signup,
        "plan": plan,
        "tenure_months": tenure.astype(int),
        "monthly_spend": np.round(spend, 2),
        "support_tickets": tickets,
        "churned": churned,
    })
    after = df["signup_date"] >= pd.Timestamp("2024-07-01")
    bug = after & (rng.random(n) < 0.6)
    df.loc[bug, "support_tickets"] = 0
    return df
```

`features/build.py`:

```python
import numpy as np
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import FunctionTransformer, OneHotEncoder, StandardScaler
import pandas as pd

LOG_COLS = ["monthly_spend"]
PLAIN_COLS = ["tenure_months", "support_tickets"]
RATIO_COL = ["tickets_per_month"]
CAT_COLS = ["plan"]


def add_ratio(df: pd.DataFrame) -> pd.DataFrame:
    """Row-level transform: safe to apply before splitting because it uses no other rows."""
    out = df.copy()
    out["tickets_per_month"] = out["support_tickets"] / out["tenure_months"].clip(lower=1)
    return out


def build_preprocessor(use_ratio: bool) -> ColumnTransformer:
    log_pipe = make_pipeline(
        FunctionTransformer(np.log1p),
        SimpleImputer(strategy="median"),
        StandardScaler(),
    )
    num_pipe = make_pipeline(SimpleImputer(strategy="median"), StandardScaler())
    numeric = [("log", log_pipe, LOG_COLS), ("num", num_pipe, PLAIN_COLS)]
    if use_ratio:
        numeric.append(("ratio", num_pipe, RATIO_COL))
    return ColumnTransformer(
        numeric + [("cat", OneHotEncoder(handle_unknown="ignore"), CAT_COLS)]
    )


def build_model(use_ratio: bool):
    return make_pipeline(build_preprocessor(use_ratio),
                         LogisticRegression(max_iter=2000))


def feature_columns(use_ratio: bool) -> list[str]:
    cols = LOG_COLS + PLAIN_COLS + CAT_COLS
    return cols + RATIO_COL if use_ratio else cols
```

`features/evaluate.py`:

```python
import pandas as pd
from sklearn.model_selection import StratifiedKFold, cross_val_score
from features.build import add_ratio, build_model, feature_columns


def compare(df: pd.DataFrame, seed: int = 0) -> dict:
    data = add_ratio(df)
    y = data["churned"]
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=seed)
    results = {}
    for use_ratio in (False, True):
        X = data[feature_columns(use_ratio)]
        scores = cross_val_score(build_model(use_ratio), X, y, cv=cv, scoring="roc_auc")
        results["with_ratio" if use_ratio else "without_ratio"] = scores
    return results
```

`features/__init__.py`: leave it empty.

`tests/test_build.py`:

```python
import numpy as np
import pandas as pd
from features.generate import make_subscriptions
from features.build import add_ratio, build_model, build_preprocessor, feature_columns


def test_ratio_handles_zero_tenure():
    df = pd.DataFrame({"support_tickets": [3], "tenure_months": [0]})
    out = add_ratio(df)
    assert np.isfinite(out["tickets_per_month"]).all()


def test_unknown_plan_does_not_fail():
    df = make_subscriptions(n=200)
    pre = build_preprocessor(use_ratio=False).fit(df[feature_columns(False)])
    new = df[feature_columns(False)].head(3).copy()
    new["plan"] = "enterprise"
    out = pre.transform(new)
    assert out.shape[0] == 3


def test_model_fits_and_predicts_probabilities():
    df = add_ratio(make_subscriptions(n=300))
    model = build_model(use_ratio=True).fit(df[feature_columns(True)], df["churned"])
    proba = model.predict_proba(df[feature_columns(True)])
    assert proba.shape == (300, 2)
    assert np.allclose(proba.sum(axis=1), 1.0)
```

`run_features.py`:

```python
from features.generate import make_subscriptions
from features.evaluate import compare

if __name__ == "__main__":
    df = make_subscriptions()
    results = compare(df)
    for name, scores in results.items():
        print(f"{name}: mean AUC {scores.mean():.4f}, std {scores.std():.4f}")
        print("  folds:", [round(s, 4) for s in scores])
```

### Step-by-step tasks

1. Create the repository, the virtual environment and the install. Create the files as shown, and add the `features` and `tests` folders with an empty `__init__.py` in each.
2. Run `pytest -q`. All three tests should pass.
3. Run `python run_features.py`. Copy the two result lines into your README.
4. Explain the gap, or lack of a gap, between the two AUC means in one paragraph. If the difference is smaller than the fold-to-fold standard deviation, say so and do not claim an improvement.
5. Move the `add_ratio` call inside the pipeline as a `FunctionTransformer`, then rerun the comparison. Confirm the scores do not change, and explain why a row-level transform is safe either way while a fitted scaler would not be.
6. Break the pipeline on purpose. Fit the `StandardScaler` on the full dataset before `cross_val_score` and compare the scores. Record the difference and explain which score you would trust in a report.
7. Write the README with the sections in the checklist.

### README checklist

- [ ] Install and run instructions with the versions you tested
- [ ] The two AUC results, with mean and standard deviation across folds
- [ ] A one-paragraph explanation of the result, including whether the gain is larger than the noise
- [ ] A table of every feature, its transformation and the reason for it
- [ ] The result of the deliberate leak experiment in step 6, and why it matters
- [ ] One feature you considered and dropped, with the reason
- [ ] One limitation, for example that the data is synthetic and the planted bug affects the ratio feature

### Done when

- `pytest -q` passes all three tests
- `run_features.py` prints both AUC results with fold details
- Every fitted transformer is inside the pipeline, so cross-validation refits it on each training fold
- The README states whether the ratio feature helps, using the fold spread as the measure of noise
- The leak experiment result is recorded and explained
- The planted bug's effect on the ratio feature is described in the README

### Evidence to commit to GitHub

Commit the package, the tests, the runner, the README with the results table, and the comparison output as a text file in the repository. The reader should be able to show a green `pytest -q` run, the two AUC results, and a commit history that records the leak experiment as a separate commit that was then reverted. Do not commit a virtual environment folder.

---

## Check your understanding

1. A feature has values from 0 to 1,000,000, and a second feature has values from 0 to 10. Without scaling, which feature will a regularized logistic regression treat as more important, and why might that be the wrong conclusion?
2. A colleague encodes a nominal category as integers and says the model "learned the order." What is the mistake, and what would you use instead?
3. Your scaler was fitted on the full dataset and the model scores 0.02 AUC higher in cross-validation than a clean version. Explain which information leaked and why the score is not trustworthy.
4. A ratio feature `tickets / tenure` has a large value for a customer with one month of tenure and two tickets. Describe the risk and propose a fix that does not delete the customer.
5. A tree-based model does well without scaling, while a neural network does poorly. Explain the difference in how the two model families use feature values.
6. A feature selection step keeps the 10 best of 200 features by mutual information on the whole dataset, and the cross-validated score is excellent. Explain why the score may be optimistic and how you would rerun the experiment to check.

Work each answer out in writing before you read the answers below.

**Answers**

1. Coefficients depend on units. The feature that ranges from 0 to 10 needs a large coefficient to matter, and the one that ranges up to a million needs a tiny one, so raw coefficient sizes make the 0-to-10 feature look more important. A regularized model also penalizes the coefficients unequally. Scale the features first, then compare the coefficients.
2. The integer encoding invents an order and equal spacing. It tells the model that pro is a fixed step above plus, which is usually false for a nominal category. Use one-hot encoding, which gives each plan its own weight without an order.
3. The scaler's mean and standard deviation were computed with test rows included, so the training features carry information about the test distribution. The 0.02 gain is an artifact of that leak. Trust the version where the scaler is fitted inside each training fold.
4. A ratio with a one-month denominator can be very large, and one such row can dominate a linear model. Keep the row, but stabilize the denominator with a floor such as `clip(lower=1)`, or use `tickets / (tenure + 1)`, or take the logarithm of the ratio after scaling.
5. Tree models split on thresholds, and a monotone rescaling does not change which rows fall on each side of a threshold, so their results barely change. Neural networks and linear models multiply inputs by weights and rely on gradient steps, so unscaled inputs make training unstable or slow.
6. Choosing the 10 features on the full data lets the selection see the test rows, so the score is optimistic. Put the selector inside the pipeline, so each training fold chooses its own features, and report that cross-validated score. Trust the nested version.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "Feature engineering", for the overview and the common transformations.
- Wikipedia, article "One-hot", for the encoding and its use in machine learning.
- Wikipedia, article "Feature selection", for the three families of methods.
- Wikipedia, article "Mutual information", for the definition and the intuition.
- Wikipedia, article "Box-Cox transformation", for the family of power transforms and how the power is chosen.
- scikit-learn documentation, User Guide sections on "Preprocessing data", "Encoding categorical features", "Feature selection" and "Column transformer". Search the titles on scikit-learn.org.
- Alice Zheng and Amanda Casari, *Feature Engineering for Machine Learning*, O'Reilly, 2018. The book is practical and covers most of today's topics.

**Tutorials**

- GeeksforGeeks, "Feature Engineering" and the articles on feature scaling and encoding categorical data.
- W3Schools, "Machine Learning" in the Python section of w3schools.com, for the basics of preprocessing.

**Watch**

- StatQuest with Josh Starmer, on YouTube, for the explanations of logistic regression and of standardization.
- Search YouTube for "feature engineering tutorial Python" and choose a talk that shows a before and after comparison of a model score.
- DeepLearning.AI, "Machine Learning Specialization" by Andrew Ng, on Coursera. The course covers feature scaling and feature engineering in its early modules.

**Practice**

- Kaggle Learn, "Feature Engineering" micro-course, free with a Kaggle account, with browser notebooks and immediate feedback.

---

## Tomorrow

Tomorrow is datasets and data pipelines. Today you built a pipeline that transforms one table in one run. Day 22 asks what happens when the same transformations must run on new data, on the next month's file, and on live requests, with the same results every time. You will split datasets, batch and shuffle them, version them, and check that training and serving compute the same features.

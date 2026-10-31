# Day 20: Exploratory Data Analysis

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 3.5 hours
**Level:** intermediate. You can already load, filter, group and clean a table with pandas. Today the goal is to ask questions of the data and to test the answers, before a model is involved.

**By the end of this day, you will have:**

- A method for describing a variable's distribution, the relationships between variables, the correlations among them and the balance of the target label
- The habit of forming a testable hypothesis from a plot or a table, and checking it with a simple statistical summary
- A reproducible EDA script and a written report on a synthetic subscription dataset that contains a planted data bug for you to find

**What you need:**

- A computer with Python 3.10 or newer
- pandas 2.2 or newer (`pip install "pandas>=2.2"`), free under the BSD 3-Clause license
- NumPy 2.0 or newer
- matplotlib 3.8 or newer (`pip install "matplotlib>=3.8"`), free under a BSD-style license
- pytest 8.0 or newer (`pip install "pytest>=8.0"`), for the optional test step
- Jupyter Lab (optional, free) or VS Code with the Jupyter extension for the exercises
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 14: distributions, the mean, the median, the standard deviation and the idea of a confidence interval
- Day 18: pandas grouping, resampling by date and filtering
- Day 19: cleaning rules, so that you know what a valid value looks like

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Distributions](#distributions)
- [Relationships](#relationships)
- [Correlations](#correlations)
- [Class imbalance](#class-imbalance)
- [Visualization](#visualization)
- [Hypothesis formation](#hypothesis-formation)
- [Data-driven debugging](#data-driven-debugging)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an EDA report that finds a planted bug](#project-an-eda-report-that-finds-a-planted-bug)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Exploratory data analysis, usually shortened to EDA, is the stage where you get to know the data before you try to predict anything with it. A good analyst looks at each variable, asks how the variables relate, checks the target, and writes down what surprised them. The output is not a model. It is a set of facts and questions that shape the model you build later.

EDA is also the cheapest place to catch problems. A histogram can show that a column is bimodal because two systems record the same field in different units. A time series plot can show that a value dropped to zero on a single day because a job failed. A correlation table can show that two features are the same number in disguise. Finding those things in an afternoon of plotting is far cheaper than finding them after a model has learned from them and produced a confident wrong answer.

On the job, EDA happens at the start of nearly every project and again whenever a metric moves unexpectedly. Product analysts use it to explain a drop in sign-ups. Machine learning engineers use it to decide which features to build. Data scientists use it to write the first draft of a report that other people will rely on. The discipline is the same in each case: describe, compare, question and check.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1880s to 1900s | Francis Galton introduces regression and correlation, and Karl Pearson formalizes the correlation coefficient. | The scatter plot and the correlation coefficient are still the first tools for relationships. |
| 1962 | John W. Tukey publishes "The Future of Data Analysis" and argues that data analysis deserves to be treated as a science in its own right. | Tukey's essay is the usual origin of the term exploratory data analysis. |
| 1977 | Tukey's book *Exploratory Data Analysis* introduces the box plot, the stem-and-leaf plot and the five-number summary. | The box plot in your library is Tukey's invention. |
| 1983 | Hoaglin, Mosteller and Tukey publish *Understanding Robust and Exploratory Data Analysis*, a textbook that extends the methods. | It shows how robust summaries such as the median protect against outliers. |
| 2000s to today | Open tools such as R, matplotlib and pandas make the plots and summaries of EDA available to anyone with a laptop. | EDA moves from specialist work to a standard first step in every data project. |

**Takeaway:** EDA was named and popularized by Tukey, and the tools you use today are the same ideas made fast and automatic.

---

## Glossary

- **Distribution**: the pattern of how often each value occurs, or how values spread across a range.
- **Histogram**: a bar chart that groups a numeric variable into bins and shows how many values fall into each bin.
- **Skewness**: the tendency of a distribution to have a long tail on one side. Income is usually right-skewed.
- **Quartile**: a value that splits sorted data into four equal parts. The 25th and 75th percentiles are the first and third quartiles.
- **Relationship**: a pattern in which the value of one variable tends to go with the value of another.
- **Correlation**: a number between -1 and 1 that measures how strongly two variables move together in a straight-line or ranked sense.
- **Pearson correlation**: the usual correlation, which measures linear association between numeric variables.
- **Spearman correlation**: the correlation of ranks, which measures monotonic association and is less affected by outliers.
- **Class imbalance**: a situation where one label is much more common than the others.
- **Baseline**: a simple prediction, such as always predicting the most common class, against which a model must be compared.
- **Hypothesis**: a specific, testable statement about the data that could turn out to be false.
- **Confidence interval**: a range of values that is likely to contain the true quantity, given the data and the method.
- **Bootstrap**: a method of estimating uncertainty by resampling the data with replacement many times.

---

### Distributions

A distribution is the shape of your data. Think of a crowd at a stadium. Some sections are packed and others are empty. A distribution tells you where the crowd sits, how spread out it is, and whether a few people stand far away from everyone else.

From first principles, describing a distribution takes four questions. Where is the center, measured by the mean or the median? How spread out is it, measured by the standard deviation or the interquartile range? Is it symmetric or skewed? Are there several peaks, which would suggest that two groups are mixed together? The mean and the median agree on symmetric data and split apart on skewed data, so comparing them is itself a diagnostic.

```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(0)
income = pd.Series(rng.lognormal(mean=10.5, sigma=0.6, size=1000))

print(income.describe(percentiles=[0.05, 0.25, 0.5, 0.75, 0.95]).round(0))
print("mean - median:", round(income.mean() - income.median(), 0))
```

For a right-skewed variable like income, the mean is larger than the median. If you report only the mean, you tell the reader that a typical customer earns more than most customers actually do. The fix is to report the median and a spread measure, or to transform the variable with a logarithm before modeling. A common mistake is to read `describe()` and stop. The useful habit is to plot the histogram and look at its shape before trusting any single number. Skewed monetary and usage variables are the most common cause of models that are dominated by a few heavy users.

---

### Relationships

A relationship is a pattern between two variables. Picture two friends who are always seen together. When one of them is in a room, the other is probably nearby. A relationship only tells you that knowing one says something about the other. Whether one causes the other is a separate question.

From first principles, a relationship between two numeric variables is visible in a scatter plot. Each point is a row, placed at its values on the two axes. A cloud that slopes up says the variables rise together. A curved cloud says the relationship is nonlinear, which a correlation coefficient may miss. Between a numeric variable and a category, the equivalent view is a box plot or a grouped mean, one per category.

```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(1)
tenure = rng.integers(1, 60, size=500)
spend = 10 + 0.8 * tenure + rng.normal(0, 5, size=500)
df = pd.DataFrame({"tenure_months": tenure, "monthly_spend": spend})

df["tenure_band"] = pd.cut(df["tenure_months"], bins=[0, 12, 24, 60],
                           labels=["under 1 year", "1 to 2 years", "2 to 5 years"])
print(df.groupby("tenure_band", observed=True)["monthly_spend"].agg(["mean", "std", "size"]))
```

Binning a numeric variable into bands and comparing group means is a simple and robust way to see a relationship. It also shows the sample size in each band, which matters because a band with 8 customers gives an unreliable mean. The mistake is to read a relationship as causal. In the example, longer tenure goes with higher spend, but that may be because loyal customers upgrade, because the spend rose over time, or because a third factor drives both. Product teams use these tables to decide which segment to target, and each relationship needs a separate check before it becomes a business decision.

---

### Correlations

A correlation coefficient compresses a relationship into one number from -1 to 1. A value near 1 means the two variables rise together in a straight line. A value near -1 means one rises as the other falls. A value near 0 means no straight-line association, which does not rule out a curved or more complicated pattern.

From first principles, Pearson correlation is the covariance of the two variables divided by the product of their standard deviations. It is sensitive to outliers and it measures only linear association. Spearman correlation applies Pearson's formula to the ranks of the values, so it measures whether one variable tends to increase when the other does, even when the increase is curved. Pandas computes both with `corr(method=...)`.

```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(2)
x = rng.normal(size=300)
df = pd.DataFrame({
    "x": x,
    "linear": 2 * x + rng.normal(scale=0.5, size=300),
    "curved": x ** 2 + rng.normal(scale=0.1, size=300),
    "noise": rng.normal(size=300),
})

print(df.corr(method="pearson").round(2))
print(df.corr(method="spearman").round(2))
```

The `curved` column has a near-zero Pearson correlation with `x`, even though the two are tightly linked by a formula. This is the main limitation to remember. A correlation matrix can hide a relationship that is clear on a scatter plot. Always plot the pairs you care about. A second mistake is to treat high correlation between two features as a reason to drop one of them without checking whether both carry the same information for the target. Correlation tables are the first step in spotting redundant features and in spotting a feature that is almost a copy of the label, which is a sign of leakage from Day 19.

---

### Class imbalance

Class imbalance means one label is far more common than the others. A fraud dataset may have one fraudulent transaction for every thousand normal ones. A churn dataset may have 8 churned customers for every 100 who stay. The imbalance changes how you should read every metric, because a model that predicts the majority class for everyone can look accurate.

From first principles, the first step is to count the labels and compute the share of each. The second step is to compute the baseline score: the accuracy of always predicting the majority class. Any model must beat that baseline to be useful. The third step is to decide which metric matches the cost of each error, which is a business question.

```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(3)
y = pd.Series(rng.random(1000) < 0.08).astype(int)   # about 8 percent positive

print(y.value_counts(normalize=True).round(3))
majority_accuracy = y.value_counts(normalize=True).max()
print("baseline accuracy:", round(majority_accuracy, 3))  # about 0.92
```

A model that reports 92 percent accuracy here has learned nothing if it predicts "stay" for everyone. The common mistake is to compare a model to a coin flip, which is a 50 percent baseline that is much easier to beat. The fix is to compare to the majority class and to report precision, recall or a similar metric that looks at the minority class. Imbalance appears in fraud, medical screening, equipment failure and churn, and the cost of missing a rare positive case is often far greater than the cost of a false alarm.

---

### Visualization

A plot is a claim about the data that the eye can check. A good plot makes one comparison easy, labels its axes and states its units, and uses color only when it adds information. A bad plot hides the comparison behind decoration, uses a truncated axis to exaggerate a difference, or plots so many series that none of them can be read.

The matplotlib library is the standard low-level plotting tool in Python. It works well with pandas. The pattern is to create a figure and an axis, draw the data, label everything and save the image to a file, so that the plot can be reproduced and attached to a report.

```python
import numpy as np
import pandas as pd
import matplotlib
matplotlib.use("Agg")                     # save to files without opening a window
import matplotlib.pyplot as plt

rng = np.random.default_rng(4)
income = pd.Series(rng.lognormal(mean=10.5, sigma=0.6, size=1000))

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(10, 4))
ax1.hist(income, bins=40, color="#4C72B0")
ax1.set_title("Income, raw scale")
ax1.set_xlabel("income (currency units)")
ax1.set_ylabel("count")

ax2.hist(np.log(income), bins=40, color="#55A868")
ax2.set_title("Income, log scale")
ax2.set_xlabel("log(income)")

fig.tight_layout()
fig.savefig("income_hist.png", dpi=120)
```

The two panels show the same data. The raw scale is squeezed to the left with a long tail, while the log scale is roughly symmetric. The choice of scale changes what the viewer sees, so label it clearly. The mistake is to show a plot without saying what the scale is, or to compare two plots that use different axis limits without noticing. Use the same limits when you compare groups. Dashboards and reports are judged on whether a non-specialist can read them, and an unlabeled axis is one of the most common reasons a correct analysis is misunderstood.

---

### Hypothesis formation

EDA should end with hypotheses, not conclusions. A hypothesis is a statement that is specific enough to be wrong. "Customers with more support tickets churn more often" is a hypothesis. "Support matters" is not, because nothing could show it false.

From first principles, a good hypothesis names the variables, the direction of the effect and the population. You then test it with a comparison that has a stated uncertainty. A confidence interval for a difference in rates tells you the range of effects consistent with the data. If the interval sits entirely above zero, the data supports a positive difference. If it spans zero, the data does not separate the groups.

```python
import numpy as np

def bootstrap_diff(a: np.ndarray, b: np.ndarray, n_boot: int = 2000, seed: int = 0):
    """95 percent bootstrap interval for mean(a) - mean(b)."""
    rng = np.random.default_rng(seed)
    diffs = np.empty(n_boot)
    for i in range(n_boot):
        diffs[i] = rng.choice(a, size=len(a)).mean() - rng.choice(b, size=len(b)).mean()
    return np.percentile(diffs, [2.5, 97.5])

rng = np.random.default_rng(5)
churn_many_tickets = rng.binomial(1, 0.15, size=400)
churn_few_tickets = rng.binomial(1, 0.09, size=600)
print(bootstrap_diff(churn_many_tickets, churn_few_tickets))
```

The interval is the answer to "how sure are we?" If it is wide, collect more data before making a decision. The common mistake is to test many hypotheses on the same data and report the one that looks significant. With twenty comparisons at the 5 percent level, one will look significant by chance. Write the hypotheses before you look at the test results, or correct for the number of comparisons. This is why experiment plans are written before a test starts, and why an analyst who finds a surprising pattern is expected to check it on fresh data.

---

### Data-driven debugging

Some of the most useful EDA results are bugs. A pipeline changed on a certain date. A column is filled with a default value for a subset of rows. A value that should be positive is zero for an entire week. These problems look like real behavior until you plot them against time or against the source system.

From first principles, debugging with EDA means asking "what would the data look like if the process were working?" and checking for the difference. Plot each key variable over time, compare the share of zeros and missing values across periods, and check the values that appear more often than the rest. A sudden step change on one date is a strong sign of a code or configuration change, not a change in customer behavior.

```python
import pandas as pd

def monthly_profile(df: pd.DataFrame) -> pd.DataFrame:
    df = df.assign(month=df["signup_date"].dt.to_period("M"))
    return df.groupby("month").agg(
        rows=("customer_id", "size"),
        mean_tickets=("support_tickets", "mean"),
        zero_share=("support_tickets", lambda s: (s == 0).mean()),
    )
```

If `zero_share` jumps in one month while `mean_tickets` drops, the feature may have been recorded wrongly, and the model would learn that customers stopped contacting support. The mistake is to explain the change in behavior before ruling out a bug. When a variable shifts suddenly, check the pipeline first. This habit prevents costly decisions based on a logging error, such as a pricing change made because a conversion metric appeared to drop when a tracking script had stopped firing.

---

## Hands-on exercises

Work in a notebook. For each step, write a one-sentence prediction before you run the code, then a one-sentence conclusion after.

1. **Distribution.** Generate 1000 values from a normal distribution and 1000 from a lognormal distribution with the same median. Plot both histograms on the same figure. Print the mean and the median of each. Explain why the lognormal mean is larger.
2. **Grouped relationship.** Create a dataset with a numeric feature and a numeric target that rises with the feature. Bin the feature into five quantile groups with `pd.qcut`. Report the mean target, the standard deviation and the count per group. Note any group with fewer than 30 rows.
3. **Correlation trap.** Generate a variable `x` and a variable `y = x**2`. Compute the Pearson and Spearman correlations. Plot the scatter. Explain why the Pearson value is near zero.
4. **Imbalance baseline.** Create a binary label with 5 percent positives. Compute the accuracy of always predicting the majority class. Then fit a logistic regression on one feature that is weakly related to the label and compare its accuracy to the baseline. Report precision and recall for the positive class.
5. **Plot critique.** Find or create one plot that uses a truncated axis or an unlabeled scale. Rewrite it so that the comparison is honest, and write two sentences explaining what changed.
6. **Hypothesis test.** Write one specific hypothesis about your data before looking at any result. Compute a bootstrap interval for the difference it predicts. State whether the interval supports the hypothesis.
7. **Planted bug.** Take a time series of a daily count and insert a week of zeros in the middle. Show that a monthly mean hides the problem, and that a daily plot or a zero-share table reveals it.

---

## Project: an EDA report that finds a planted bug

### Goal

Build a reproducible exploratory analysis of a synthetic subscription dataset. The dataset contains a real relationship, a class imbalance and one bug in a single column after a known date. Your job is to describe the data, form and test hypotheses, find the bug, and report what you found with the evidence.

Suggested repository name: `100-days-ai-day-20-eda`

### Repo layout

```text
100-days-ai-day-20-eda/
├── README.md
├── eda/
│   ├── __init__.py
│   ├── generate.py
│   ├── profile.py
│   └── plots.py
├── tests/
│   └── test_generate.py
├── outputs/
│   ├── .gitkeep
├── run_eda.py
├── requirements.txt
└── .gitignore
```

### Starter code

`requirements.txt`:

```text
pandas>=2.2
numpy>=2.0
matplotlib>=3.8
pytest>=8.0
```

`eda/generate.py`:

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
    # Planted bug: from July 2024 on, the ticket counter is written as 0 for 60 percent of rows.
    after = df["signup_date"] >= pd.Timestamp("2024-07-01")
    bug = after & (rng.random(n) < 0.6)
    df.loc[bug, "support_tickets"] = 0
    return df
```

`eda/profile.py`:

```python
import pandas as pd


def describe_numeric(df: pd.DataFrame) -> pd.DataFrame:
    cols = ["tenure_months", "monthly_spend", "support_tickets"]
    return df[cols].describe(percentiles=[0.05, 0.25, 0.5, 0.75, 0.95]).round(2)


def churn_balance(df: pd.DataFrame) -> pd.Series:
    return df["churned"].value_counts(normalize=True).round(4)


def correlations(df: pd.DataFrame) -> pd.DataFrame:
    cols = ["tenure_months", "monthly_spend", "support_tickets", "churned"]
    return df[cols].corr(method="spearman").round(3)


def churn_by_tickets(df: pd.DataFrame) -> pd.DataFrame:
    return (
        df.groupby("support_tickets")
        .agg(rate=("churned", "mean"), rows=("churned", "size"))
        .round(3)
    )


def monthly_profile(df: pd.DataFrame) -> pd.DataFrame:
    month = df["signup_date"].dt.to_period("M")
    return df.groupby(month).agg(
        rows=("customer_id", "size"),
        mean_tickets=("support_tickets", "mean"),
        zero_ticket_share=("support_tickets", lambda s: round((s == 0).mean(), 3)),
    )
```

`eda/plots.py`:

```python
from pathlib import Path
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import pandas as pd


def save_plots(df: pd.DataFrame, out_dir: Path) -> None:
    out_dir.mkdir(parents=True, exist_ok=True)

    fig, ax = plt.subplots(figsize=(7, 4))
    ax.hist(df["monthly_spend"], bins=40, color="#4C72B0")
    ax.set_title("Monthly spend")
    ax.set_xlabel("monthly spend (currency units)")
    ax.set_ylabel("customers")
    fig.tight_layout()
    fig.savefig(out_dir / "spend_hist.png", dpi=120)
    plt.close(fig)

    monthly = df.groupby(df["signup_date"].dt.to_period("M"))["support_tickets"].mean()
    fig, ax = plt.subplots(figsize=(7, 4))
    ax.plot(monthly.index.astype(str), monthly.values, marker="o", color="#C44E52")
    ax.set_title("Mean support tickets by signup month")
    ax.set_xlabel("signup month")
    ax.set_ylabel("mean tickets")
    ax.tick_params(axis="x", rotation=45)
    fig.tight_layout()
    fig.savefig(out_dir / "tickets_by_month.png", dpi=120)
    plt.close(fig)
```

`run_eda.py`:

```python
from pathlib import Path
import pandas as pd
from eda.generate import make_subscriptions
from eda.profile import (describe_numeric, churn_balance, correlations,
                         churn_by_tickets, monthly_profile)
from eda.plots import save_plots

OUT = Path("outputs")

if __name__ == "__main__":
    df = make_subscriptions()
    OUT.mkdir(exist_ok=True)
    with open(OUT / "eda_report.txt", "w", encoding="utf-8") as f:
        for title, obj in [
            ("Numeric summary", describe_numeric(df)),
            ("Churn balance", churn_balance(df)),
            ("Spearman correlations", correlations(df)),
            ("Churn rate by ticket count", churn_by_tickets(df)),
            ("Monthly profile", monthly_profile(df)),
        ]:
            f.write(f"== {title} ==\n{obj}\n\n")
    save_plots(df, OUT)
    print(open(OUT / "eda_report.txt", encoding="utf-8").read())
```

`tests/test_generate.py`:

```python
import pandas as pd
from eda.generate import make_subscriptions


def test_same_seed_same_data():
    a = make_subscriptions(seed=3)
    b = make_subscriptions(seed=3)
    assert a.equals(b)


def test_churn_is_imbalanced():
    df = make_subscriptions()
    rate = df["churned"].mean()
    assert 0.03 < rate < 0.25


def test_bug_is_only_after_july():
    df = make_subscriptions()
    before = df[df["signup_date"] < pd.Timestamp("2024-07-01")]
    assert (before["support_tickets"] == 0).mean() < 0.5
```

### Step-by-step tasks

1. Create the repository, the environment and the install. Run `python run_eda.py` and read the printed report.
2. Before reading the monthly profile closely, write three hypotheses in the README about how churn relates to tenure, plan and tickets. Each must name a direction.
3. Using `churn_by_tickets`, state the churn rate at each ticket count. Use the bootstrap function from the lesson to put an interval on the difference between customers with zero tickets and those with two or more tickets.
4. Look at the `churn_balance` output. Write the baseline accuracy in the README and state why accuracy alone would be misleading.
5. Open `outputs/tickets_by_month.png`. Describe what you see. Then check `monthly_profile` and identify the month where the zero-ticket share changes. Explain why this is a data-quality problem and not a change in customer behavior.
6. Write a short fix for the planted bug: a function that flags rows after the change date where the ticket count is suspicious. Do not delete those rows. Write one test that the flag is raised for the bug rows and not for the rows before the date.
7. Write the README report with the sections in the checklist below.

### README checklist

- [ ] Install and run instructions with the versions you tested
- [ ] The three hypotheses, written before the results
- [ ] The numeric summary and the churn balance, copied from the report
- [ ] The bootstrap interval for the ticket effect, with one sentence on what it means
- [ ] The planted bug: the month, the evidence and the fix you proposed
- [ ] One plot, with a sentence explaining what the viewer should see in it
- [ ] One limitation of the analysis, for example that the dataset is synthetic and the relationships are designed

### Done when

- `python run_eda.py` runs from a clean clone and writes `outputs/eda_report.txt` and two PNG files
- The three hypotheses are in the README and were written before the results section
- The README states the baseline accuracy and explains why it matters
- The planted bug is found, dated, and explained as a pipeline problem
- The flag function and its test pass with `pytest -q`
- Every number in the README matches the generated report

### Evidence to commit to GitHub

Commit the package, the script, the tests, the README and the generated `eda_report.txt`. Commit the two PNG files too, because the reader should be able to see the plots without running the code. The commit history should show the hypotheses file or README section committed before the results section, if you write them in separate commits. The reader should be able to show a green `pytest -q` run and point to the exact month of the bug.

---

## Check your understanding

1. A variable has a mean of 50 and a median of 30. Describe the shape of its distribution, and say which summary you would report to a non-specialist and why.
2. Two features have a Pearson correlation of 0.02. A colleague concludes that they are unrelated and drops one. Give a reason this could be wrong and the plot you would check first.
3. A model predicts churn with 91 percent accuracy in a dataset where 9 percent of customers churned. Is that good? Compute the baseline and name the metric you would examine next.
4. You test three hypotheses on the same dataset and one has a p-value under 0.05. Explain the risk of reporting only that one, and describe how you would structure the analysis to reduce it.
5. A monthly average of a feature drops sharply in one month and then recovers. Name two causes that are about the data pipeline and one cause that is about real behavior, and say which check you would run first.
6. You are asked whether a relationship between tenure and churn is causal. Explain what EDA can and cannot show, and name one additional kind of evidence that would help.

Work each answer out in writing before you read the answers below.

**Answers**

1. A mean well above the median means the distribution is right-skewed, with a long tail of large values pulling the mean up. Report the median with the interquartile range, because most readers take the typical value to be the middle one, and the mean is pulled by a few large values.
2. A Pearson value of 0.02 means there is no straight-line association, which does not mean the features are unrelated. A curved or threshold relationship can hide behind a near-zero Pearson value. Plot the scatter first and check the Spearman correlation. Do not drop a feature on the basis of a correlation alone.
3. A baseline that always predicts "stay" is about 91 percent accurate, so 91 percent accuracy is no better than the baseline. Look at precision, recall and average precision for the churn class, because those show how the model does on the rare positive cases.
4. With three independent tests at the 5 percent level, the chance that at least one looks significant by luck is about 14 percent (1 minus 0.95 to the power of 3). Reporting only the significant one overstates the evidence. Reduce the risk by stating the hypotheses before the test, correcting for the number of comparisons, for example by dividing the threshold by three, or by confirming the result on new data.
5. Two pipeline causes are a logging change or a failed job that writes default values for a period, and a change in how a field is recorded, such as a new unit or a new definition. A real behavior change, such as a promotion or seasonality, is the third cause. Check the pipeline change log and the zero or missing share by day around the change before you explain it as customer behavior.
6. EDA shows association, not cause. It cannot rule out a confounder, such as plan type affecting both tenure and churn. To support a causal claim you need a controlled experiment with random assignment, or a causal method that states its assumptions clearly, such as adjusting for the confounders you can measure.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "Exploratory data analysis", for the overview and the history of the term.
- Wikipedia, article "Histogram", and the article "Box plot", which covers Tukey's five-number summary.
- Wikipedia, article "Pearson correlation coefficient", and the article "Spearman's rank correlation coefficient".
- Wikipedia, article "Bootstrapping (statistics)", for the resampling method used in the hypothesis step.
- Wikipedia, article "Simpson's paradox", for why a relationship can reverse when a third variable is added.
- John W. Tukey, *Exploratory Data Analysis*, Addison-Wesley, 1977. The book is a classic; search for a library copy or a reputable reprint.
- pandas documentation, User Guide sections on group by and time series resampling, and the matplotlib tutorials on plotting histograms. Search for each on pandas.pydata.org and matplotlib.org.

**Tutorials**

- GeeksforGeeks, "Exploratory Data Analysis in Python" and the articles on correlation and visualization with matplotlib.
- W3Schools, "Matplotlib Tutorial" in the Python section of w3schools.com.

**Watch**

- StatQuest with Josh Starmer, on YouTube, for the videos on correlation, the Pearson coefficient and bootstrapping.
- Search YouTube for "Simpson's paradox" to find a short explainer with a worked example.
- Search YouTube for "EDA in Python" and choose a video that works with a real dataset and shows each plot's purpose.

**Practice**

- Kaggle Learn, "Data Visualization" micro-course, free with a Kaggle account. It runs in a browser notebook and uses real data.

---

## Tomorrow

Tomorrow is feature engineering. Today you learned what your data looks like, which variables move together, how imbalanced the target is, and where the data can lie to you. Day 21 turns those observations into inputs a model can use. You will encode categories, scale numbers, create interactions between features, and choose which features to keep. Each of those steps should be justified by something you saw in today's plots and tables.

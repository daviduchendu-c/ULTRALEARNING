# Day 18: pandas

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 3.5 hours
**Level:** beginner to intermediate. You need Python functions and NumPy arrays from Day 17. If you can index a NumPy array with a boolean mask, you are ready.

**By the end of this day, you will have:**

- A working grasp of Series and DataFrames, label and position indexing, filtering, grouping, merging, missing data and column transformations
- A small reporting pipeline, `pandas_orders`, that loads two CSV files, joins them, handles missing values and writes a summary, with tests that check each step
- A short written note on which of your pandas operations copied data and which changed the original, with the reason for each

**What you need:**

- A computer with Python 3.10 or newer
- pandas 2.2 or newer (`pip install "pandas>=2.2"`). pandas is free and open source under the BSD 3-Clause license. The examples also run on pandas 3.x.
- NumPy 2.0 or newer, which pandas installs as a dependency if it is missing
- pytest 8.0 or newer (`pip install "pytest>=8.0"`)
- A code editor such as VS Code (free) or Jupyter Lab for the exercises
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 17: ndarrays, shapes, boolean masks and broadcasting
- Day 14: summary statistics such as sums, means and medians
- Basic Python: dictionaries, functions and reading files

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Series](#series)
- [DataFrames](#dataframes)
- [Indexing](#indexing)
- [Filtering](#filtering)
- [Grouping](#grouping)
- [Merging](#merging)
- [Missing data](#missing-data)
- [Transformations](#transformations)
- [Copy and view rules in pandas](#copy-and-view-rules-in-pandas)
- [Hands-on exercises](#hands-on-exercises)
- [Project: pandas_orders, a tested reporting pipeline](#project-pandas_orders-a-tested-reporting-pipeline)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A NumPy array is a block of numbers with positions. A real dataset is different. It has a column called `customer_id` that holds text, a column called `order_date` that holds dates, a column called `discount` where some rows are blank, and a set of rows that need to be matched to another table before they mean anything. pandas is the Python library built for that kind of data. Its central object, the DataFrame, is a table with named columns that can each hold a different type, and it gives you a vocabulary for the work that fills most of a data scientist's day: select, filter, group, join, fill and reshape.

The library is a standard tool in industry. Before a model is trained, someone usually loads the raw export into pandas, checks it, joins it to labels, and turns it into the matrix the model reads. The quality of that step decides whether the model learns the pattern or learns an accident of the data.

On the job, pandas appears in batch reporting scripts, in notebooks that document an experiment, in feature pipelines that run before training, and in the glue code between a database and a model. Two habits matter most. The first is to check the shape and types after every step. The second is to avoid operations that silently change or duplicate rows, which is where most pandas bugs come from.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 2008 | Wes McKinney begins writing pandas while working at AQR Capital Management, to analyze financial time series. | The library grew out of a practical need for labeled data in finance. |
| 2009 to 2011 | pandas gains the DataFrame and its label-based indexing, and becomes open source. | The table model you use today dates from this period. |
| 2012 | McKinney's book *Python for Data Analysis* is published by O'Reilly. | It made pandas the default teaching tool for data work in Python. |
| 2020 | pandas 1.0 is released, the first major version with a stable public API. | Code written against 1.x and later versions is broadly compatible. |
| 2023 to 2026 | pandas 2.x adds the Arrow-backed dtypes and an opt-in copy-on-write mode, and pandas 3.0 makes copy-on-write and a dedicated string dtype the default. | The behavior of assignment and copying changed, so check the version you run. |

**Takeaway:** pandas is mature, but its copying rules changed in recent versions. Learn to check what each operation returns.

---

## Glossary

- **Series**: a one-dimensional labeled array. It is a single column with an index.
- **DataFrame**: a two-dimensional table of labeled columns. Each column is a Series that shares the same index.
- **Index**: the labels for the rows of a Series or DataFrame. It can be numbers, dates or text.
- **Label-based indexing**: selecting by the row label or column name, using `.loc`.
- **Position-based indexing**: selecting by integer position, using `.iloc`.
- **Boolean mask**: a Series of True and False values that keeps the rows where the value is True.
- **Group**: a set of rows that share the same value in one or more columns, produced by `groupby`.
- **Aggregation**: a function that reduces a group to one value, such as a sum, mean or count.
- **Join or merge**: combining two tables by matching the values of key columns.
- **Missing value**: an entry with no data. pandas represents it as `NaN` for numbers and `NA` or `None` for other types, depending on the dtype.
- **Dtype**: the data type of a column, such as `int64`, `float64`, `string` or `datetime64[ns]`.
- **Copy-on-write**: a rule that makes a derived object behave as a copy, so changing it cannot silently change the object it came from.

---

### Series

Think of a Series as one column of a paper ledger, where each line has a label written in the margin. The values are the amounts and the labels are the names of the accounts. Unlike a plain list, you can ask for an amount by its account name, and the arithmetic lines up the labels for you.

From first principles, a Series is a NumPy array with an index attached and some extra metadata such as a name and a dtype. When you add two Series, pandas matches the labels first and then adds the values. Labels that are present in only one of the two results become missing values, not errors.

```python
import pandas as pd

prices = pd.Series([3.5, 2.0, None], index=["apple", "bread", "milk"], name="price")

print(prices.dtype)          # float64, because the None became NaN
print(prices["bread"])       # 2.0
print(prices.isna().sum())   # 1

discounts = pd.Series([0.5, 0.2], index=["apple", "cheese"])
print(prices + discounts)    # apple 4.0, bread NaN, milk NaN, cheese NaN
```

The label alignment in the last line is useful and also a trap. If your labels are slightly different, for example `"Apple"` and `"apple"`, the sum becomes mostly missing values without any error. Normalize labels before you combine data. Series are what a single feature looks like inside a pipeline, and label alignment is how a time-stamped feature gets matched to a time-stamped label.

---

### DataFrames

A DataFrame is a set of Series that share one index, placed side by side like the columns of a spreadsheet. Each column has its own dtype, so one column can hold integers, another text and another dates. That is the main difference from a NumPy array, which holds one type throughout.

The DataFrame is what you get from almost every real data source. `pd.read_csv` reads a file, `pd.read_sql` reads a database query, and `pd.read_parquet` reads a columnar file. Each returns a table you can inspect with `.head()`, `.info()` and `.describe()`.

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["Lagos", "Nairobi", "Lagos", "Accra"],
    "sales": [120, 90, 150, None],
    "units": [4, 3, 5, 2],
})

print(df.dtypes)
print(df.shape)         # (4, 3)
df.info()               # column types and non-null counts
print(df.describe())    # summary statistics for numeric columns
```

In `dtypes`, `sales` became `float64` once the missing value appeared, even though every other value is a whole number. pandas upgrades integer columns to floats when they contain a missing entry, because NumPy has no integer missing value. Use the nullable `Int64` dtype if you need integers with missing values. The mistake is to trust a column that looks like integers but has been silently converted. `info()` is the first line of nearly every notebook, because it tells you what the file actually contained rather than what you expected.

---

### Indexing

Indexing answers the question "which rows and columns do I want?" pandas offers two modes. Label-based indexing with `.loc` uses the index labels and column names. Position-based indexing with `.iloc` uses integer positions, counting from zero, the way NumPy does.

Think of a hotel. You can ask for room 214 by its label, or you can ask for the third room on the second floor by its position. Both work, but they give different answers if the rooms are renumbered. Label-based selection is safer for code that must survive a change in row order. Position-based selection is simpler when you only want the first or last rows.

```python
import pandas as pd

df = pd.DataFrame(
    {"city": ["Lagos", "Nairobi", "Lagos", "Accra"],
     "sales": [120.0, 90.0, 150.0, None],
     "units": [4, 3, 5, 2]},
    index=["o1", "o2", "o3", "o4"],
)

print(df.loc["o2", "sales"])          # 90.0, by label
print(df.iloc[1, 1])                  # 90.0, by position
print(df.loc["o1":"o3", ["city", "units"]])   # label slices include the end
print(df.iloc[0:2, [0, 2]])           # position slices exclude the end
```

Note the difference in slicing. `.loc["o1":"o3"]` includes `o3`, while `.iloc[0:2]` excludes position 2. This is the most common indexing bug in pandas code. The rule to remember is that label slices are inclusive and position slices are not. Write `.loc` when the index is meaningful and `.iloc` when you truly mean "the first n rows". Real pipelines use `.loc` for assignment, such as `df.loc[mask, "flag"] = True`, because it sets values on the original table and avoids the chained-assignment problems described in the next sections.

---

### Filtering

Filtering keeps the rows that satisfy a condition. It works like a sieve. You describe what should pass, pandas builds a boolean mask, and only the rows where the mask is True come through.

From first principles, a comparison such as `df["sales"] > 100` returns a Series of True and False, one per row. Combine conditions with `&` for and, `|` for or and `~` for not. Wrap each condition in parentheses, because the operators bind more tightly than comparisons in Python.

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["Lagos", "Nairobi", "Lagos", "Accra"],
    "sales": [120.0, 90.0, 150.0, None],
})

big_lagos = df[(df["city"] == "Lagos") & (df["sales"] > 100)]
print(big_lagos)                       # rows 0 and 2

known = df[df["sales"].notna()]        # drop rows with missing sales
print(len(known))                      # 3

df.loc[df["sales"] > 100, "tier"] = "high"
print(df)
```

The common mistake is writing `df["city"] == "Lagos" & df["sales"] > 100` without parentheses. Python evaluates `&` first and raises an error or returns the wrong result. Another mistake is filtering to make a copy and then assigning to it, which triggers a `SettingWithCopyWarning` in older versions. Using `.loc` for the assignment avoids most of this. Filters define the population a metric is computed on, so an error in a filter silently changes every number in a report.

---

### Grouping

Grouping splits rows into sets that share a key, applies a function to each set and combines the results. Picture a teacher sorting exam papers into piles by class, then computing the average score of each pile. The split, apply and combine steps are the same whatever the function.

The pattern is `df.groupby(key)` followed by an aggregation. Named aggregation lets you name each output column, which makes the result readable and easy to test.

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["Lagos", "Nairobi", "Lagos", "Accra"],
    "sales": [120.0, 90.0, 150.0, None],
    "units": [4, 3, 5, 2],
})

by_city = df.groupby("city").agg(
    total_sales=("sales", "sum"),
    orders=("units", "size"),
    avg_units=("units", "mean"),
)
print(by_city)
#          total_sales  orders  avg_units
# city
# Accra           0.0       1        2.0
# Lagos         270.0       2        4.5
# Nairobi        90.0       1        3.0
```

Accra's total sales is 0.0, not missing, because the default `sum` skips missing values and adds up nothing. That number is misleading: the city had no recorded sales, not zero sales. Use `min_count=1` in `sum` to get `NaN` when every value is missing. The mistake is to report that total as a fact. Grouping also drops rows whose key is missing, by default, so a city column with blanks can make your totals disagree with the row count. Check the count of rows before and after every grouping. Grouped aggregations are how almost every dashboard metric is produced, from daily active users to revenue per region.

---

### Merging

Merging joins two tables on shared key columns, the same idea as a join in SQL. Think of matching two lists of guests by their badge numbers. Each row of one list is paired with the rows of the other that share the badge number.

The main choice is the `how` argument. An inner join keeps only keys present in both tables. A left join keeps every row of the left table and fills in blanks when the right side has no match. An outer join keeps everything from both sides. The `validate` argument checks the relationship between the keys, which catches many silent duplications.

```python
import pandas as pd

orders = pd.DataFrame({
    "city": ["Lagos", "Nairobi", "Lagos", "Accra"],
    "sales": [120.0, 90.0, 150.0, None],
})
regions = pd.DataFrame({
    "city": ["Lagos", "Nairobi"],
    "region": ["West", "East"],
})

merged = orders.merge(regions, on="city", how="left", validate="many_to_one")
print(merged)
#      city   sales region
# 0   Lagos   120.0   West
# 1 Nairobi    90.0   East
# 2   Lagos   150.0   West
# 3   Accra     NaN    NaN
```

The `validate="many_to_one"` argument says that each city must appear at most once in `regions`. If someone adds a duplicate city to the lookup table, pandas raises an error instead of quietly doubling your orders. The common mistake is an inner join that drops Accra without telling you, so the report looks complete when it is not. Compare the row count before and after every merge. A join against a customer or product table is the place where most duplicated revenue bugs are born.

---

### Missing data

Missing data is information you do not have. It can mean a sensor failed, a user skipped a form field, or two systems do not share a key. The right response depends on why the value is missing, and pandas gives you tools to find, count, drop and fill it.

The first step is to count. `isna()` marks each missing cell, and `sum()` counts them per column. The second step is to decide. You can drop rows with `dropna`, fill with a constant or a statistic with `fillna`, or interpolate for ordered data. Each choice changes the result.

```python
import pandas as pd

df = pd.DataFrame({
    "city": ["Lagos", "Nairobi", "Lagos", "Accra"],
    "sales": [120.0, 90.0, 150.0, None],
})

print(df.isna().sum())                      # missing count per column

median_sales = df["sales"].median()         # 120.0, ignores NaN
df["sales_filled"] = df["sales"].fillna(median_sales)
df["sales_known"] = df["sales"].notna()

print(df)
```

Filling with the median is a reasonable default for a skewed numeric column, but it is still an invented number. It makes the Accra row look like an average city, which may be untrue. Keep the indicator column `sales_known` so downstream models and reports can see that the value was imputed. A more serious mistake is computing the median on the full dataset before you split it into train and test sets, which lets information from the test set leak into training. Compute fill values from the training data only, and store them so the same values are applied to new data. Missing values also appear in merges, as the Accra row showed, so check for them after every join.

---

### Transformations

Transformations create new columns or reshape the table. They take the form of arithmetic on columns, string methods, date parsing, conditional logic and functions applied to groups. Think of a recipe card where each step takes the ingredients on the table and produces a new mixture, and the original ingredients stay on the counter.

The method `assign` returns a new DataFrame with added columns, which keeps the code in one readable expression. The method `pipe` passes a DataFrame through a function, which is the usual way to structure a pipeline into named steps.

```python
import numpy as np
import pandas as pd

def add_revenue(frame: pd.DataFrame, fx_rate: float) -> pd.DataFrame:
    return frame.assign(
        revenue_usd=lambda d: d["sales"] * fx_rate,
        city=lambda d: d["city"].str.strip().str.title(),
        big_order=lambda d: np.where(d["units"] >= 5, "yes", "no"),
    )

df = pd.DataFrame({
    "city": ["  lagos", "NAIROBI ", "lagos", "accra"],
    "sales": [120.0, 90.0, 150.0, None],
    "units": [4, 3, 5, 2],
})

clean = df.pipe(add_revenue, fx_rate=0.0006)
print(clean)
```

The function returns a new table. The input `df` is unchanged, which makes each step testable on its own. The common mistake is writing `df["revenue"] = ...` in several places in a long script, so that the columns depend on the order in which cells were run in a notebook. Put transformations in functions with clear inputs and outputs, and call them in a fixed order. A `pipe` chain is a common shape for feature code, and it is what lets a team reuse the same transformations for training and for live predictions.

---

### Copy and view rules in pandas

Some operations return views and some return copies, and recent pandas versions changed the default. You do not need to memorize every case, but you should understand the principle. A derived table should not change the table it came from unless you ask it to. Methods such as `assign`, `merge`, `groupby` and `fillna` return new objects. Assignment with `.loc` changes the original on purpose. Chained assignment, such as `df[df.x > 0]["y"] = 1`, may or may not change `df`, and it is the reason for the warnings in older code. Write `df.loc[df.x > 0, "y"] = 1` instead.

---

## Hands-on exercises

Work in a notebook or a script. For each step, write one sentence predicting the output before you run the code.

1. **Build and inspect.** Create a DataFrame with ten rows and four columns of your own invention, with at least one missing value and one text column. Print `info()`, `describe()` and `isna().sum()`. Note which column changed dtype because of the missing value.
2. **Label versus position.** Set the index to a column of your own ID strings. Select one row by label with `.loc` and the same row by position with `.iloc`. Then select a range with each and explain the difference in what is included.
3. **Filter with three conditions.** Keep rows where a numeric column is above a threshold, a text column equals one value, and another column is not missing. Print the row count after each condition is added, so you can see how the filter narrows the table.
4. **Grouped report.** Group your data by one categorical column and compute three aggregates with named aggregation: a sum, a count and a mean. Check that the counts add up to the number of rows you started with, minus any rows with a missing key.
5. **Join safely.** Create a lookup table with one row per category. Merge it onto your data with `validate="many_to_one"`. Then add a duplicate row to the lookup table and confirm that pandas raises an error.
6. **Impute with discipline.** Pick a numeric column with missing values. Split the data into the first 70 percent and the last 30 percent by row order. Compute the median on the first part only, fill both parts with that number, and add an indicator column for imputed values.
7. **Pipe.** Write a function that takes a DataFrame and returns a new one with two derived columns. Chain it with `pipe` and show that the original DataFrame is unchanged afterward.

---

## Project: pandas_orders, a tested reporting pipeline

### Goal

Build a small pipeline that reads two CSV files, cleans them, joins them, fills missing values with a documented rule, and writes a regional summary. The project uses data generated by a seeded script, so the repository is self-contained and every run produces the same files.

Suggested repository name: `100-days-ai-day-18-pandas`

### Repo layout

```text
100-days-ai-day-18-pandas/
├── README.md
├── pandas_orders/
│   ├── __init__.py
│   ├── generate.py
│   └── pipeline.py
├── data/
│   └── .gitkeep
├── tests/
│   └── test_pipeline.py
├── requirements.txt
└── .gitignore
```

### Starter code

`requirements.txt`:

```text
pandas>=2.2
numpy>=2.0
pytest>=8.0
```

`pandas_orders/__init__.py`:

```python
from .pipeline import load_orders, clean_orders, join_regions, summarize

__all__ = ["load_orders", "clean_orders", "join_regions", "summarize"]
```

`pandas_orders/generate.py`:

```python
from pathlib import Path
import numpy as np
import pandas as pd


def write_sample_data(out_dir: Path, seed: int = 42, n_orders: int = 200) -> None:
    rng = np.random.default_rng(seed)
    cities = ["Lagos", "Nairobi", "Accra", "Kigali", "Dakar"]
    orders = pd.DataFrame({
        "order_id": [f"o{i:04d}" for i in range(n_orders)],
        "city": rng.choice(cities, size=n_orders),
        "units": rng.integers(1, 10, size=n_orders),
        "unit_price": np.round(rng.uniform(5.0, 50.0, size=n_orders), 2),
    })
    # Make some data messy on purpose: blank prices and stray spaces in city names.
    orders.loc[rng.choice(n_orders, size=15, replace=False), "unit_price"] = np.nan
    orders.loc[rng.choice(n_orders, size=10, replace=False), "city"] = (
        orders["city"].str.upper() + " "
    )
    out_dir.mkdir(parents=True, exist_ok=True)
    orders.to_csv(out_dir / "orders.csv", index=False)

    regions = pd.DataFrame({
        "city": ["Lagos", "Nairobi", "Accra", "Kigali", "Dakar"],
        "region": ["West", "East", "West", "East", "West"],
    })
    regions.to_csv(out_dir / "regions.csv", index=False)


if __name__ == "__main__":
    write_sample_data(Path("data"))
```

`pandas_orders/pipeline.py`:

```python
from pathlib import Path
import pandas as pd


def load_orders(path: Path) -> pd.DataFrame:
    return pd.read_csv(path)


def clean_orders(orders: pd.DataFrame, fill_price: float | None = None) -> pd.DataFrame:
    """Normalize city names, add revenue, and mark and fill missing prices.

    If fill_price is None, the median of the known prices is used.
    """
    out = orders.copy()
    out["city"] = out["city"].str.strip().str.title()
    out["price_was_missing"] = out["unit_price"].isna()
    if fill_price is None:
        fill_price = out["unit_price"].median()
    out["unit_price"] = out["unit_price"].fillna(fill_price)
    out["revenue"] = out["units"] * out["unit_price"]
    return out


def join_regions(orders: pd.DataFrame, regions: pd.DataFrame) -> pd.DataFrame:
    return orders.merge(regions, on="city", how="left", validate="many_to_one")


def summarize(joined: pd.DataFrame) -> pd.DataFrame:
    return (
        joined.groupby("region", dropna=False)
        .agg(
            orders=("order_id", "size"),
            revenue=("revenue", "sum"),
            imputed_prices=("price_was_missing", "sum"),
        )
        .reset_index()
    )


def run(data_dir: Path, out_path: Path) -> pd.DataFrame:
    orders = clean_orders(load_orders(data_dir / "orders.csv"))
    regions = pd.read_csv(data_dir / "regions.csv")
    summary = summarize(join_regions(orders, regions))
    summary.to_csv(out_path, index=False)
    return summary
```

`tests/test_pipeline.py`:

```python
from pathlib import Path
import pandas as pd
from pandas_orders.generate import write_sample_data
from pandas_orders.pipeline import load_orders, clean_orders, join_regions, summarize


def make_inputs(tmp_path: Path):
    write_sample_data(tmp_path, seed=42, n_orders=200)
    return load_orders(tmp_path / "orders.csv"), pd.read_csv(tmp_path / "regions.csv")


def test_clean_strips_and_uppercases_cities(tmp_path):
    orders, _ = make_inputs(tmp_path)
    cleaned = clean_orders(orders)
    assert cleaned["city"].str.contains(" ").sum() == 0
    assert cleaned["city"].isin(["Lagos", "Nairobi", "Accra", "Kigali", "Dakar"]).all()


def test_missing_prices_are_filled_and_flagged(tmp_path):
    orders, _ = make_inputs(tmp_path)
    cleaned = clean_orders(orders)
    assert cleaned["unit_price"].isna().sum() == 0
    assert cleaned["price_was_missing"].sum() == orders["unit_price"].isna().sum()


def test_join_keeps_row_count(tmp_path):
    orders, regions = make_inputs(tmp_path)
    cleaned = clean_orders(orders)
    joined = join_regions(cleaned, regions)
    assert len(joined) == len(cleaned)
    assert joined["region"].notna().all()


def test_summary_revenue_matches_total(tmp_path):
    orders, regions = make_inputs(tmp_path)
    cleaned = clean_orders(orders)
    summary = summarize(join_regions(cleaned, regions))
    assert abs(summary["revenue"].sum() - cleaned["revenue"].sum()) < 1e-6
    assert summary["orders"].sum() == len(cleaned)
```

### Step-by-step tasks

1. Create the folders, initialize Git, create a virtual environment, and install the requirements.
2. Create the files as shown. Run `python -m pandas_orders.generate` from the repository root. Open `data/orders.csv` and confirm that you can see blank prices and stray spaces.
3. Run `pytest -q`. All four tests should pass.
4. Run the pipeline from a short script or a notebook: `from pathlib import Path; from pandas_orders.pipeline import run; print(run(Path("data"), Path("data/summary.csv")))`. Read the printed summary and check one region's total by hand using a filter in a notebook.
5. Change the fill rule to use zero instead of the median, by passing `fill_price=0.0`. Explain in the README what changes in the revenue total and why zero is a poor default.
6. Add a test that an unknown city, such as "Abuja", does not disappear from the output when it is joined to the regions table. Decide what the correct behavior is, implement it, and document the choice.
7. Write the README with the sections listed in the checklist below.

### README checklist

- [ ] Install and run instructions with the pandas version you tested
- [ ] The command that generates the sample data and the command that runs the pipeline
- [ ] The printed regional summary, pasted as text
- [ ] A description of each cleaning rule and why it exists
- [ ] The result of your fill-rule experiment in step 5
- [ ] A short note on the join: which key, which validation, and what happens to unknown cities
- [ ] One limitation, for example that the pipeline reads the whole file into memory

### Done when

- `pytest -q` passes with no failures
- Running the generator twice produces byte-identical CSV files
- The summary has one row per region, and the sum of orders equals the number of input rows
- The join uses `validate="many_to_one"` and the tests confirm that the row count is unchanged
- Every imputed price is flagged in the cleaned data and counted in the summary
- The README explains each decision, including the unknown-city choice

### Evidence to commit to GitHub

Commit the package, the tests, the generator, the README and the requirements file. Do not commit the large generated CSV files, but do commit the small `summary.csv` so the reader can compare it with a fresh run. The reader should be able to show a green `pytest -q` run, a summary file, and a commit history that records the unknown-city decision.

---

## Check your understanding

1. You run `df.groupby("city")["sales"].sum()` and Accra shows 0. Explain why this may be the wrong number to report, and write the one-line change that makes missing groups show as missing.
2. A left merge returns more rows than the left table. List two causes and say which argument would have caught each one early.
3. A teammate fills missing ages with the mean of the whole dataset before splitting into train and test sets. Describe the leak and the order of operations that avoids it.
4. You write `df[df["score"] > 50]["grade"] = "pass"` and nothing changes in `df`. Explain what pandas did and rewrite the line so it works.
5. Two analysts compute "monthly revenue" from the same file and get different totals. Name three decisions in a pandas pipeline that could explain the difference, and explain how you would make the definition reproducible.
6. Your pipeline works on a 10,000-row file and runs out of memory on a 50-million-row file. Which parts of this lesson would you change first, and what would you check before choosing between a chunked read, a different dtype and a database query?

Work each answer out in writing before you read the answers below.

**Answers**

1. `sum` skips missing values, so an all-missing group adds up to 0. That means "no recorded sales", not "zero sales", so the report overstates how well Accra did. Use `.sum(min_count=1)`, which returns `NaN` when a group has no values at all.
2. A left merge gains rows when the lookup table has more than one row for a key. Pass `validate="many_to_one"` to raise an error at the merge. Duplicates already in the left table also multiply nothing, but they inflate totals, so check `duplicated()` on the left key before the join.
3. The fill value is computed from every row, including the test rows, so information from the test set reaches the training features. Split first, compute the median on the training rows only, and apply that value to the test rows. A pipeline object such as `SimpleImputer` inside a cross-validation loop does this for you.
4. `df[mask]` may return a copy, so the assignment writes to the copy and the original is unchanged. Write `df.loc[df["score"] > 50, "grade"] = "pass"` instead, which assigns to the original table.
5. The decisions are which rows are filtered (cancelled or test orders, duplicates), which date defines the month (order date or payment date), how refunds and missing prices are counted, and which join type is used. Make the definition reproducible by writing it as one documented function with tests, so every analyst runs the same code.
6. First, measure the data: `df.memory_usage(deep=True).sum()` and the number of rows against your RAM. Then reduce the footprint with `usecols` for only the columns you need, `category` or smaller numeric dtypes, and `chunksize` in `read_csv` for aggregations that can be combined chunk by chunk. If the work needs the whole table at once, use a database query or a columnar format such as Parquet rather than a loop in pandas.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "Pandas (software)", for the library's history and design.
- Wikipedia, article "Relational database" and the section on joins, for the idea behind merging tables.
- Wikipedia, article "Missing data", for the statistical meaning of missing values and why the reason for the gap matters.
- pandas documentation, the User Guide sections on indexing, group by, merge, join and concatenate, and missing data. Search "pandas user guide" on pandas.pydata.org.
- Wes McKinney, *Python for Data Analysis*, O'Reilly, third edition. The early chapters cover the same ground as this lesson.

**Tutorials**

- GeeksforGeeks, "Pandas Tutorial" and the articles on pandas groupby, merge and handling missing values.
- W3Schools, "Pandas Tutorial" in the Python section of w3schools.com.

**Watch**

- Search YouTube for "pandas groupby explained" and choose a video that uses a real table and shows the output of each step.
- Search YouTube for "pandas copy on write" to find the recorded explanation of the newer copy rules.
- Stanford CS229, "Machine Learning" lectures by Andrew Ng, on YouTube, when you want to see how prepared features feed a model. Search the course name to find the official playlist.

**Practice**

- Kaggle Learn, "Pandas" micro-course, free with a Kaggle account. It runs in a browser notebook and checks your answers as you go.

---

## Tomorrow

Tomorrow is data cleaning. Today you filled, joined and grouped a dataset that you generated yourself, with the mess placed where you expected it. Real data is messier and less polite: duplicate records, impossible ages, dates written in three formats, and labels that are wrong. Day 19 turns the cleaning steps you saw here into a systematic checklist, and it asks whether the data is right at all, beyond whether it has the right shape.

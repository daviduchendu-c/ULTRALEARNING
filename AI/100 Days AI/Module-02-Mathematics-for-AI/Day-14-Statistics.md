# Day 14: Statistics

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** beginner to intermediate. You need Day 13 (probability) and comfort with formulas that use sums. It is for readers who will summarize datasets, judge whether a result is real and report uncertainty to people who do not read math.

**By the end of this day, you will have:**

- Hand-written descriptive statistics (mean, median, variance, standard deviation, covariance, correlation), each checked against NumPy
- A sampling experiment that shows how a biased sample gives a wrong answer
- A coverage simulation that tests whether a 95 percent confidence interval behaves as claimed
- A short report on one real dataset

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 1.24 or newer (`pip install numpy`)
- pytest 7 or newer (`pip install pytest`)
- Any CSV file with a numeric column for the exercises
- Git and a free GitHub account for the project

**Prerequisites:**

- Day 13: events, random variables, conditional probability and the normal and binomial distributions
- Day 7: summation notation and square roots
- Basic Python: functions, loops and list comprehensions

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Mean](#mean)
- [Median](#median)
- [Variance](#variance)
- [Standard deviation](#standard-deviation)
- [Covariance](#covariance)
- [Correlation](#correlation)
- [Sampling](#sampling)
- [Bias](#bias)
- [Confidence concepts](#confidence-concepts)
- [Statistical distributions](#statistical-distributions)
- [Hands-on exercises](#hands-on-exercises)
- [Project: statistics from scratch](#project-statistics-from-scratch)
- [Check your understanding](#check-your-understanding)
- [Sources](#sources)
- [Tomorrow](#tomorrow)

---

## Introduction

Probability starts from a model and asks what data will look like. Statistics runs the other way: you start with data and ask what it says about the world it came from. "Conversion went up 4 percent." "The new model is better." "Our average delivery time is 2.3 days." Each claim can be true, false or meaningless, depending on how the numbers were collected and summarized.

On the job, statistics protects you from two expensive errors. The first is reading noise as signal, such as celebrating a metric that moved by chance. The second is trusting a summary that hides the shape of the data, such as an average dominated by a few extreme values.

The big idea is that every statistic is a summary, and every summary throws information away. The mean keeps the total and drops the shape. Correlation keeps the direction of a relationship and drops its cause. A confidence interval keeps an honest measure of what you do not know. Your job is to pick the summary that fits the question and say what it leaves out.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1810s | Gauss and Laplace develop least squares and the normal distribution to handle measurement error. | Squared error and the normal distribution come from this work. |
| 1908 | William Sealy Gosset, writing as "Student", publishes the t distribution for small samples. | Explains why small samples need a wider multiplier than 1.96. |
| 1979 | Bradley Efron introduces the bootstrap, which estimates uncertainty by resampling the data. | Gives confidence intervals for statistics that have no formula. |

**Takeaway:** the core summaries are over a century old. Newer tools such as the bootstrap answer the same questions when the old formulas do not apply.

---

## Glossary

- **Population**: the full set of things you want to learn about.
- **Sample**: the subset you actually measured.
- **Parameter**: a true value of the population, usually unknown.
- **Statistic**: a value computed from a sample, which estimates a parameter.
- **Variance**: the average squared distance of values from their mean.
- **Standard deviation**: the square root of the variance, in the same units as the data.
- **Covariance**: how two variables move together, in units of the two variables multiplied.
- **Correlation**: covariance rescaled to lie between -1 and 1.
- **Sampling**: the process that decides which members of the population end up in your data.
- **Bias**: a systematic error, where the average of an estimate across samples is not the true value.
- **Standard error**: the standard deviation of a statistic across samples. For a mean, it is the standard deviation divided by the square root of `n`.
- **Confidence interval**: a range computed from a sample, designed to contain the true parameter at a stated rate over repeated samples.
- **Central limit theorem**: the result that the distribution of a sample mean approaches a normal distribution as the sample grows.
- **Bootstrap**: estimating uncertainty by resampling the observed data with replacement.

---

## Mean

The mean is the average. If five friends split a restaurant bill evenly, the mean is each person's share. Formally, it is the sum of the values divided by their count. For `[2, 3, 3, 4, 5, 7, 9, 40]` the sum is 73 and the count is 8, so the mean is 9.125. The single value 40 pulls the mean well above most of the data.

```python
values = [2, 3, 3, 4, 5, 7, 9, 40]
print(sum(values) / len(values))   # 9.125
```

A frequent error is reporting the mean of a skewed quantity without showing its shape. Incomes, page load times and order sizes are skewed, and one outlier can make the mean look typical when most people are far below it. Report the median next to it and look at the largest values before trusting a single average.

In the market, the mean is the default summary in most reports, and mean squared error (Day 16) is a mean, which is why it is sensitive to outliers too.

---

## Median

The median is the middle value of the sorted data. For an even count, it is the average of the two middle values. For the same eight numbers, the middle pair is 4 and 5, so the median is 4.5. The mean was 9.125. The outlier moved the mean a lot and the median hardly at all.

```python
import numpy as np

print(np.median([2, 3, 3, 4, 5, 7, 9, 40]))   # 4.5
```

The honest limitation is that the median ignores most of the information in the data. Two datasets with the same median can have very different spreads, and you cannot compute the median of a combined dataset from the medians of its parts. Use it to describe the typical value and pair it with a spread measure.

In the market, medians are standard for income, house prices and latency, where a few extreme values would otherwise dominate. Service-level targets often pair the median with the 95th or 99th percentile.

---

## Variance

Variance measures spread. Take each value's distance from the mean, square it so negatives do not cancel positives, and average the squares. The population formula divides by `n`. When the data is a sample and you want an unbiased estimate of the population variance, divide by `n - 1`. NumPy's default is `ddof=0`, and `ddof=1` gives the sample version.

```python
import numpy as np

values = np.array([2, 3, 3, 4, 5, 7, 9, 40], dtype=float)
print(round(np.var(values), 4))           # 140.8594, divide by n
print(round(np.var(values, ddof=1), 4))   # 160.9821, divide by n - 1
```

The common mistake is to mix the two versions without saying which one you used. The gap is small for large samples and large for small ones. Also note that variance is in squared units, so a variance of 140 dollars squared is hard to picture. That is why the standard deviation is usually reported.

In the market, variance sets risk limits in finance, helps choose sample sizes for experiments and is the quantity behind squared error losses.

---

## Standard deviation

The standard deviation is the square root of the variance, in the same units as the data. For the sample above, it is 11.87 with `ddof=0` and 12.69 with `ddof=1`. For normal data, about 68 percent of values lie within one standard deviation of the mean. That rule does not hold for skewed data. Here the standard deviation of 12 is pulled up by one value, while most values sit within 5 of the mean.

The common mistake is to report a mean and standard deviation for a skewed variable and let the reader assume a bell curve. Report the median and the interquartile range (the 25th to 75th percentile) alongside it.

In the market, standard deviation is used in quality control charts, for volatility in finance and for standardization, which subtracts the mean and divides by the standard deviation before a model is trained.

---

## Covariance

Covariance measures whether two variables rise and fall together. For each observation, multiply the two deviations from their means and average the products. Positive products mean both move in the same direction.

Take hours of study `[1, 2, 3, 4, 5]` and exam scores `[52, 58, 66, 71, 80]`. The means are 3 hours and 65.4 points. Every product of deviations is positive, so the covariance is positive. With the population formula it is 13.8, and with the sample formula it is 17.25.

```python
import numpy as np

hours = np.array([1, 2, 3, 4, 5.0])
score = np.array([52, 58, 66, 71, 80.0])
print(np.cov(hours, score, bias=True)[0, 1])   # 13.8
print(np.cov(hours, score)[0, 1])               # 17.25
```

The problem with covariance is its units. Hours times points means nothing to most readers, and the size depends on the scale of each variable. That is why correlation is usually reported instead.

The common mistake is to read covariance as strength. A covariance of 100 measured in thousands of dollars is not stronger than a covariance of 2 measured in dollars. Changing units changes the number.

In the market, covariance matrices are the basis of portfolio risk models, and they are the starting point for principal components (Day 10).

---

## Correlation

Correlation divides covariance by both standard deviations, which removes the units and the scale. The result is always between -1 and 1. Values near 1 mean a straight-line rise, values near -1 mean a straight-line fall, and values near 0 mean no straight-line relationship.

For the study data the correlation is 0.9968, very close to a straight line. That does not mean studying causes higher scores. It says the two move together in this sample.

```python
import numpy as np

print(round(np.corrcoef([1, 2, 3, 4, 5.0], [52, 58, 66, 71, 80.0])[0, 1], 4))   # 0.9968
```

The common mistake is to read correlation as causation or as the whole relationship. Pearson correlation measures only straight-line association. The points `(x, x^2)` for x from -3 to 3 have correlation zero, even though y is entirely determined by x. A single outlier can also create a strong correlation, so plot the scatter every time.

In the market, correlation drives feature selection and data quality checks that flag duplicate information. Strongly correlated inputs can make some models unstable, which is why teams check them before training.

---

## Sampling

Almost all data is a sample. Sampling decides which members of the population end up in the data, and that decision determines whether your summaries describe the population or only the people you reached. Simple random sampling gives every member an equal chance of selection, and under that design the sample mean is a fair estimate of the population mean.

```python
import numpy as np

rng = np.random.default_rng(7)
population = rng.normal(100, 15, 100_000)

random_sample = rng.choice(population, size=1_000, replace=False)
print(round(random_sample.mean(), 2))     # close to the population mean, 99.98

biased_sample = population[population > 90][:1_000]
print(round(biased_sample.mean(), 2))     # 105.53, far from the population mean
```

The common mistake is to believe a large sample fixes a bad sampling method. A million responses from people who already use your product still describe the wrong population. Size reduces random error. It does not reduce bias.

In the market, sampling decisions are made when a team chooses which users join an experiment, which logs feed model training and which examples go to annotators. Those choices decide what the model can learn.

---

## Bias

An estimator is biased when its average across many samples is not the true parameter. Bias is a consistent error, unlike noise, which is random. An unbiased but noisy estimate is right on average, while a biased one is wrong in a fixed direction.

The clearest example is the variance. Dividing by `n` underestimates the population variance on average, because the sample mean is computed from the same data. Dividing by `n - 1` removes that bias.

```python
import numpy as np

rng = np.random.default_rng(3)
est_n, est_n1 = [], []
for _ in range(20_000):
    sample = rng.normal(0, 2.0, size=5)        # true variance is 4
    est_n.append(np.var(sample))
    est_n1.append(np.var(sample, ddof=1))

print(round(np.mean(est_n), 2))     # about 3.2, too small
print(round(np.mean(est_n1), 2))    # about 4.0, unbiased
```

Assuming an estimate is good because it is precise is a common error. A precise but biased estimate is confidently wrong. Ask how the data was collected and whether the estimator has a known bias before you report it.

In the market, bias is behind many failed analytics projects, such as a churn model trained only on customers who stayed long enough to be measured.

---

## Confidence concepts

A point estimate, such as a sample mean of 9.1, does not say how far to trust it. A 95 percent confidence interval for a mean uses the standard error and the multiplier 1.96:

```
mean ± 1.96 * (sample standard deviation / square root of n)
```

The promise is about the method. If you repeated sampling and built the interval the same way each time, about 95 percent of the intervals would contain the true value. For a single interval, the true value is either inside or outside, so "95 percent chance it is inside" is the wrong reading.

This simulation draws 10,000 samples of size 30 from a population with mean 100 and standard deviation 15, and counts how often the interval covers 100:

```python
import numpy as np

rng = np.random.default_rng(0)
hits, trials, n = 0, 10_000, 30
for _ in range(trials):
    sample = rng.normal(100, 15, n)
    se = sample.std(ddof=1) / np.sqrt(n)
    lo, hi = sample.mean() - 1.96 * se, sample.mean() + 1.96 * se
    hits += lo <= 100 <= hi
print(hits / trials)   # about 0.94
```

The coverage is about 94 percent, not 95. For `n = 30`, the correct multiplier from the t distribution is 2.045, and 1.96 is slightly too narrow. This is the honest limitation of the normal shortcut for small samples.

The bootstrap is the alternative when no formula fits. Resample the data with replacement thousands of times, compute the statistic each time, and take the 2.5th and 97.5th percentiles. It needs no distribution assumption, but it can fail for tiny samples or very extreme statistics.

The common mistake is to say a confidence interval gives the probability of a result. It describes how reliable the procedure is. Another mistake is to treat overlapping intervals as proof of no difference. Compute the interval for the difference directly.

In the market, confidence intervals appear in A/B test dashboards, clinical trial reports and any model evaluation with error bars. A metric without an interval is a guess presented as a fact.

---

## Statistical distributions

Every summary assumes something about shape. Statistical distributions describe that shape, and Day 13 introduced the main families. The practical question is what a real dataset looks like. Compare the mean with the median, look at the histogram and check the tails. For a right-skewed variable such as income, the mean is larger than the median.

The central limit theorem explains why the normal distribution appears so often. The mean of many independent values tends toward a normal distribution, even when the values are not normal. This is why the confidence interval formula works for means of samples, not for individual raw values.

```python
import numpy as np

rng = np.random.default_rng(5)
skewed = rng.exponential(scale=2.0, size=(10_000, 50))
sample_means = skewed.mean(axis=1)

print(round(np.median(skewed), 2))        # 1.39, below the mean of 2.0
print(round(sample_means.std(), 3))       # about 0.283, the standard error for n = 50
```

The raw data is clearly skewed, but the means of 50 values each are close to symmetric. The theorem needs independent observations and finite variance, so heavy-tailed data with infinite variance does not satisfy it.

The common mistake is to fit a normal model because the sample mean looks normal. The raw values may still be skewed, and a prediction interval for a single value built on the normal assumption will be too narrow.

In the market, distribution checks come before any model is trained. Fraud amounts and server response times are rarely normal, and teams that log-transform them or use robust statistics avoid predictions that look accurate in the middle and fail in the tails.

---

## Hands-on exercises

Create a folder called `day14-practice` and write your code in `practice.py`. Use any CSV file with a numeric column, or the study-hours data from this lesson.

1. **Summarize one column.** Print the mean, median, sample standard deviation (`ddof=1`), minimum and maximum. Write one sentence on whether the mean and median agree.

2. **Stress the summaries.** Multiply the largest value by 100 and recompute the mean and median. Record how much each moved and explain why in one sentence.

3. **Units and correlation.** For two numeric columns, compute the covariance and correlation. Multiply one column by 1000 and compute both again. Record which number changed and which did not, and explain why.

4. **Biased sampling.** From a normal population with mean 100 and standard deviation 15, draw a random sample of 500 and a biased sample of 500 from values above 95. Print both means and the population mean. Then repeat the biased version with 5,000 values and note whether the bias shrinks.

5. **Coverage check.** Change the coverage simulation to `n = 100`, run 10,000 trials and record the coverage. Then use the t multiplier 1.984 in place of 1.96 and record the new coverage. Explain the change in one sentence.

6. **Describe your dataset.** Write a paragraph on your column: its shape, the summary you would report to a manager, one caveat about the sample, and one statistic you would not report and why.

**Evidence to commit:** `practice.py` and `notes.md` with the results of each step and your sentences.

---

## Project: statistics from scratch

This project is practical. It implements the descriptive statistics from this lesson from their formulas and tests every function against NumPy. Name the repository `100-days-ai-engineering-day-14-statistics`.

**Repo layout**

```
100-days-ai-engineering-day-14-statistics/
├── README.md
├── requirements.txt
├── stats_from_scratch.py
├── report.py
└── tests/
    └── test_stats.py
```

**Starter code**

`requirements.txt`:

```
numpy>=1.24
pytest>=7.0
```

`stats_from_scratch.py`:

```python
"""Descriptive statistics written from their formulas, checked against NumPy."""
import math


def mean(values):
    if not values:
        raise ValueError("mean of an empty sequence is undefined")
    return sum(values) / len(values)


def median(values):
    ordered = sorted(values)
    n = len(ordered)
    mid = n // 2
    if n % 2 == 1:
        return ordered[mid]
    return (ordered[mid - 1] + ordered[mid]) / 2


def variance(values, ddof=0):
    """Average squared distance from the mean. ddof=1 gives the sample variance."""
    n = len(values)
    if n - ddof <= 0:
        raise ValueError("not enough values for this ddof")
    m = mean(values)
    return sum((v - m) ** 2 for v in values) / (n - ddof)


def std(values, ddof=0):
    return math.sqrt(variance(values, ddof))


def covariance(xs, ys, ddof=0):
    """How much two variables move together, in the units of x times y."""
    if len(xs) != len(ys):
        raise ValueError("inputs must have the same length")
    mx, my = mean(xs), mean(ys)
    total = sum((x - mx) * (y - my) for x, y in zip(xs, ys))
    return total / (len(xs) - ddof)


def correlation(xs, ys):
    """Covariance divided by both standard deviations. Always between -1 and 1."""
    return covariance(xs, ys) / (std(xs) * std(ys))
```

`report.py`:

```python
import numpy as np
from stats_from_scratch import mean, median, variance, std, covariance, correlation

DATA = [2, 3, 3, 4, 5, 7, 9, 40]
print("mean", mean(DATA), "median", median(DATA))
print("var ddof0", round(variance(DATA), 4), "var ddof1", round(variance(DATA, 1), 4))
print("std ddof0", round(std(DATA), 4), "std ddof1", round(std(DATA, 1), 4))
hours = [1, 2, 3, 4, 5.0]
score = [52, 58, 66, 71, 80.0]
print("cov", round(covariance(hours, score), 3), "corr", round(correlation(hours, score), 4))

rng = np.random.default_rng(0)
hits, trials, n = 0, 10_000, 30
for _ in range(trials):
    sample = rng.normal(100, 15, n)
    se = sample.std(ddof=1) / np.sqrt(n)
    lo, hi = sample.mean() - 1.96 * se, sample.mean() + 1.96 * se
    hits += lo <= 100 <= hi
print("coverage of 95% CI with n=30:", round(hits / trials, 4))
```

Running `python report.py` prints values close to these. The coverage line changes slightly with the random seed:

```
mean 9.125 median 4.5
var ddof0 140.8594 var ddof1 160.9821
std ddof0 11.8684 std ddof1 12.6879
cov 13.8 corr 0.9968
coverage of 95% CI with n=30: 0.9385
```

`tests/test_stats.py`:

```python
import os, sys
sys.path.insert(0, os.path.join(os.path.dirname(__file__), ".."))
import numpy as np
import pytest
from stats_from_scratch import mean, median, variance, std, covariance, correlation

DATA = [2, 3, 3, 4, 5, 7, 9, 40]


def test_mean_median():
    assert mean(DATA) == pytest.approx(np.mean(DATA))
    assert median(DATA) == pytest.approx(np.median(DATA))


def test_variance_and_std():
    assert variance(DATA) == pytest.approx(np.var(DATA))
    assert variance(DATA, ddof=1) == pytest.approx(np.var(DATA, ddof=1))
    assert std(DATA, ddof=1) == pytest.approx(np.std(DATA, ddof=1))


def test_covariance_and_correlation():
    hours = [1, 2, 3, 4, 5.0]
    score = [52, 58, 66, 71, 80.0]
    assert covariance(hours, score) == pytest.approx(np.cov(hours, score, bias=True)[0, 1])
    assert correlation(hours, score) == pytest.approx(np.corrcoef(hours, score)[0, 1])


def test_correlation_bounds():
    xs = [1.0, 2.0, 3.0, 4.0]
    assert correlation(xs, [2 * x + 1 for x in xs]) == pytest.approx(1.0)
    assert correlation(xs, [-x for x in xs]) == pytest.approx(-1.0)
```

**Step-by-step tasks**

1. Create the repository, the folder tree and a virtual environment. Install the requirements, run `python report.py` and compare with the output above.
2. Run `pytest` and make sure all four tests pass.
3. Add `percentile(values, q)` with linear interpolation between sorted values. Test it against `np.percentile` for 25, 50 and 75.
4. Add `iqr(values)` and report the spread of `DATA` next to its standard deviation. Explain in the README why the IQR resists the value 40.
5. Add `bootstrap_ci(values, statistic, n_resamples=5000, seed=0)` that returns a 95 percent interval for any statistic. Test that it brackets the mean of `DATA`.
6. Write the README with the checklist below, and paste the pytest output.

**README checklist**

- [ ] The project runs with `python report.py` on a fresh environment
- [ ] `pytest` passes with at least seven tests
- [ ] The README states whether variance is population or sample, and why
- [ ] The README explains the gap between the mean and median of `DATA`
- [ ] The README names one statistic that would mislead on this data

**Done when**

- Every function matches its NumPy counterpart within `1e-9` in the tests
- The bootstrap interval for the mean of `DATA` is reported and brackets the sample mean
- The README has a table with the mean, median, standard deviation, IQR and bootstrap interval
- A reader can clone the repo and reproduce the report in three commands

**Evidence to commit:** the code, the tests, the pytest output and the summary table.

---

## Check your understanding

1. A feature lowers average response time from 220 ms to 200 ms, and the median stays at 120 ms. What should you report, and what would you check before saying the feature helped?
2. Two columns have a correlation of 0.02, yet a scatter plot shows a clear U-shape. What does correlation measure, and what does it miss?
3. A survey of 50,000 people is sent by email only to customers who opened the newsletter last month. Is sample size the main problem? What would you do first?
4. A 95 percent confidence interval for churn is 3.1 to 4.9 points. A colleague says there is a 95 percent chance the true value is in that range. Correct the statement.
5. With 12 values, one teammate uses `ddof=0` and another uses `ddof=1`. Which standard deviation is larger, by how much, and which estimates the population spread?
6. Transaction amounts have a mean of 52 and a median of 9. Would you fit a normal model? What would you check instead?

**Answers**

1. Report the median and a high percentile such as the 95th, because a mean can move because of a few slow requests. Before claiming the feature helped, compute a confidence interval for the difference between the before and after groups, and check that the change is not driven by a few outliers.
2. Pearson correlation measures straight-line association only. A U-shape is a strong relationship with no straight-line trend, because the rising and falling parts cancel. Plot the data, and use a nonlinear measure if the pattern is curved.
3. Sample size is not the main problem. The sampling frame is biased, because only engaged customers were reached, and a larger sample from the same frame would give a more precise wrong answer. First find out who was excluded and whether their preferences differ, and compare the sample with known customer demographics.
4. The interval describes the procedure. If you repeated the sampling and the calculation many times, about 95 percent of the intervals would contain the true churn rate. The true value is either inside this interval or not.
5. The `ddof=1` version is larger, by a factor of `sqrt(12/11)`, about 4.5 percent. It is the unbiased estimate of the population variance when the data is a sample, so use it for the population spread.
6. No. A mean of 52 with a median of 9 means a few large transactions dominate and the data is heavily right-skewed. Check a histogram and a log scale, consider modeling the log of the amounts, and use robust statistics such as the median and percentiles.

---

## Sources

**Read**

- Wikipedia: [Statistics](https://en.wikipedia.org/wiki/Statistics)
- Wikipedia: [Descriptive statistics](https://en.wikipedia.org/wiki/Descriptive_statistics)
- Wikipedia: [Variance](https://en.wikipedia.org/wiki/Variance)
- Wikipedia: [Covariance](https://en.wikipedia.org/wiki/Covariance)
- Wikipedia: [Pearson correlation coefficient](https://en.wikipedia.org/wiki/Pearson_correlation_coefficient)
- Wikipedia: [Sampling bias](https://en.wikipedia.org/wiki/Sampling_bias)
- Wikipedia: [Confidence interval](https://en.wikipedia.org/wiki/Confidence_interval)
- Wikipedia: [Central limit theorem](https://en.wikipedia.org/wiki/Central_limit_theorem)
- Wikipedia: [Bootstrapping (statistics)](https://en.wikipedia.org/wiki/Bootstrapping_(statistics))
- MIT OpenCourseWare, 18.05 Introduction to Probability and Statistics. Search the course title on ocw.mit.edu.

**Tutorials**

- GeeksforGeeks: search "Statistics for Machine Learning" on geeksforgeeks.org
- NumPy documentation: search "NumPy statistics functions" on numpy.org

**Watch**

- StatQuest with Josh Starmer, "Confidence Intervals, Clearly Explained" and "The Central Limit Theorem, Clearly Explained"
- StatQuest with Josh Starmer, "Standard Deviation, Clearly Explained"
- Khan Academy, the statistics and probability course, for a slower walkthrough of the mean, median and standard deviation

**Practice**

- Seeing Theory from Brown University, which has an interactive section on descriptive statistics, sampling and the central limit theorem. Search "Seeing Theory Brown".

---

## Tomorrow

Tomorrow is Day 15, Information Theory. Today you summarized data and checked whether a summary can be trusted. Information theory asks how much an observation tells you, and how many bits a random outcome needs. You will meet entropy, cross-entropy and KL divergence, which explain why classifiers are trained with cross-entropy loss. The variance and probability ideas from this day and Day 13 are the foundation for that work.

**Next:** [Day 15: Information Theory](Day-15-Information-Theory.md)

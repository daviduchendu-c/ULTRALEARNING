# Day 35: Model Evaluation: Regression

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3 hours
**Level:** intermediate. You need Day 25 (the residual and the mean squared error) and Day 34 (the habit of reporting several metrics with a baseline). The plots are where most of the insight comes from.

**By the end of this day, you will have:**

- A calculation of MAE, MSE, RMSE and R-squared by hand on a small table, and in code
- A baseline comparison that shows what a model's error means relative to predicting the mean
- A residual analysis with four plots that reveal curvature, changing spread, outliers and bias
- A short report that picks one metric from a stated cost, and says what the others add

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, for the metrics and the example models
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the residual plots
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 24: train, validation and test splits, and the baseline habit
- Day 25: residuals, mean squared error and the linear model
- Day 34: reporting several metrics with a stated purpose
- Square roots and the idea of an average of absolute values

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [MAE](#mae)
- [MSE](#mse)
- [RMSE](#rmse)
- [R-squared and its limits](#r-squared-and-its-limits)
- [Residual analysis](#residual-analysis)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Regression evaluation report with residual diagnostics](#project-regression-evaluation-report-with-residual-diagnostics)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A regression model predicts a quantity, such as the price of a house, the number of units a store will sell next week or the time a parcel takes to arrive. Its quality is not a single number. An error of 10 dollars can be trivial for a house and disastrous for a grocery item. The evaluation has to say what size of error matters, and how the errors are spread across the data.

Metric choice matters for the same reason it does in classification. MAE, MSE and RMSE measure error on the original scale in different ways, and R-squared measures the share of variation explained, which can mislead when the data is not what you expect. None of them tells you where the model is wrong. That is the job of residual analysis, which looks at the errors row by row and by group.

On the job, the question from a manager is usually "how far off will we be, and is that good enough?" Answering it needs a baseline, a unit, a residual plot and a statement of the cost of a typical error. Today you build that answer.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1805 | Adrien-Marie Legendre and, in 1809, Carl Friedrich Gauss develop least squares, which makes the squared error the natural criterion. | The squared error is the default loss because of its mathematical convenience and its link to the normal distribution. |
| 1887 | Francis Edgeworth discusses fitting by the sum of absolute deviations, an alternative to least squares. | MAE and least absolute deviations are older alternatives that are less sensitive to outliers. |
| 1890s | Karl Pearson develops the correlation coefficient in his series on the mathematical theory of evolution. | Correlation and the coefficient of determination, which R-squared generalizes, come from this line of work. |
| 1920s | Sewall Wright introduces path analysis, which decomposes the variation of a variable into the contributions of its causes. | The idea that R-squared measures the share of variance explained by a model has roots in this decomposition. |
| Today | Libraries report MAE, MSE, RMSE and R-squared by default, and residual plots remain the standard diagnostic. | Most model reports now include all four metrics and at least one residual plot. |

**Takeaway:** the squared error won because it was convenient to optimize, but the absolute error is often the better measure of how far off a prediction typically is.

---

## Glossary

- **Residual**: the actual value minus the predicted value for one row.
- **Mean absolute error (MAE)**: the average of the absolute residuals.
- **Mean squared error (MSE)**: the average of the squared residuals.
- **Root mean squared error (RMSE)**: the square root of the MSE, which returns the error to the target's units.
- **R-squared (coefficient of determination)**: one minus the ratio of the residual sum of squares to the total sum of squares.
- **Total sum of squares (SST)**: the sum of squared differences between the actual values and their mean.
- **Residual sum of squares (SSE or RSS)**: the sum of squared residuals.
- **Baseline**: the simplest reasonable prediction, here usually the mean or the median of the training targets.
- **Heteroscedasticity**: a pattern in which the spread of the residuals changes with the size of the prediction.
- **Outlier**: a row whose residual is far larger than most others.
- **Bias**: the average residual, which is zero when the model is unbiased on average.

---

### MAE

Imagine a delivery company that tells customers their parcel will arrive in a certain number of hours. A customer cares how far off the estimate is on a typical day. The mean absolute error answers that question directly: it averages the size of the miss, ignoring its sign.

From first principles, `MAE = (1/n) sum_i |y_i - y_hat_i|`. It has the same units as the target, and each error counts in proportion to its size. A single error of 10 contributes 10 to the sum, and two errors of 5 contribute 10 in total. MAE is also the loss that a median predictor minimizes, which is why it is less influenced by a few extreme values than the mean. It is not differentiable at zero, which is a minor issue for optimization but a reason some training methods prefer the squared error.

```python
import numpy as np
from sklearn.metrics import mean_absolute_error

y = np.array([10.0, 12.0, 15.0, 20.0])
y_hat = np.array([11.0, 10.0, 15.0, 27.0])
print("MAE by hand:", np.mean(np.abs(y - y_hat)))             # 2.5
print("MAE sklearn:", mean_absolute_error(y, y_hat))           # 2.5
```

The mistake is reporting MAE without the baseline. A MAE of 2.5 hours is excellent for a long-haul shipment and poor for a same-day one. The limitation of MAE is that it treats all errors linearly, so it does not punish a few very large misses as much as a business might. In industry, MAE is a common choice for demand and delivery forecasts, because the planners think in units of typical error.

### MSE

The mean squared error squares each residual before averaging. A miss of 2 counts as 4, and a miss of 10 counts as 100. That emphasis on large errors is useful when a big miss is much worse than several small ones, such as a power grid forecast that must avoid large shortfalls.

From first principles, `MSE = (1/n) sum_i (y_i - y_hat_i)^2`. Its gradient is simple, which is why least squares and gradient-based training use it. Its unit is the square of the target's unit, which makes it hard to read directly. Because it grows quadratically, a single outlier can dominate the average. This is a property of the metric and the data, not a bug, and it should be checked before the metric is used for selection.

```python
import numpy as np
from sklearn.metrics import mean_squared_error

y = np.array([10.0, 12.0, 15.0, 20.0])
y_hat = np.array([11.0, 10.0, 15.0, 27.0])
print("MSE by hand:", np.mean((y - y_hat) ** 2))              # (1 + 4 + 0 + 49) / 4 = 13.5
print("MSE sklearn:", mean_squared_error(y, y_hat))
```

The mistake is comparing MSE between two datasets with different target scales, as if the number meant the same thing in both. The limitation is the sensitivity to outliers, which can make a model that is usually good look bad because of one bad row. Check the largest residuals before drawing a conclusion. In the market, MSE is standard in training and in internal reports, and it is often translated into RMSE for presentation.

### RMSE

The root mean squared error takes the square root of the MSE. It keeps the squared emphasis on large errors but returns the result to the target's units. That makes it the most common headline number in regression reports.

From first principles, `RMSE = sqrt(MSE)`. Since the square root is monotonic, RMSE and MSE rank models in the same order on the same data, so choosing between them is about readability rather than selection. RMSE is roughly the size of a typical large error, not the typical error, because the squaring weights large errors more than small ones. That subtlety often gets lost in reports, so describe RMSE as a measure of error with extra weight on large misses.

```python
import numpy as np
from sklearn.metrics import root_mean_squared_error

y = np.array([10.0, 12.0, 15.0, 20.0])
y_hat = np.array([11.0, 10.0, 15.0, 27.0])
print("RMSE:", round(float(np.sqrt(np.mean((y - y_hat) ** 2))), 3))  # sqrt(13.5), about 3.674
print("RMSE sklearn:", round(root_mean_squared_error(y, y_hat), 3))
```

The mistake is describing RMSE as the "average error". Here the MAE is 2.5 and the RMSE is about 3.67, and the gap comes from the one large miss of 7. That gap is informative. The limitation is that RMSE depends on the scale of the target, so a model cannot be compared across targets with different units without normalizing. In industry, a team that reports RMSE should also report MAE, so that readers can see whether the error is driven by a few large misses.

### R-squared and its limits

R-squared asks what fraction of the variation in the target the model explains. It compares the model's squared errors with the squared errors of predicting the mean. A value of 1 means perfect predictions, 0 means no better than the mean, and a negative value means worse than the mean.

From first principles, `R^2 = 1 - SSE / SST`, where `SSE` is the sum of squared residuals and `SST` is the sum of squared deviations of the target from its mean. For a linear model fitted with an intercept on training data, R-squared is between 0 and 1 and equals the squared correlation between the fitted values and the target. On new data, R-squared can be negative, because the model's predictions can be worse than the mean of the new targets. R-squared does not say whether the predictions are biased, whether the errors are large in absolute terms, or whether the model is well calibrated.

```python
import numpy as np
from sklearn.metrics import r2_score

y = np.array([10.0, 12.0, 15.0, 20.0])
y_hat = np.array([11.0, 10.0, 15.0, 27.0])
sse = np.sum((y - y_hat) ** 2)
sst = np.sum((y - y.mean()) ** 2)
print("R^2 by hand:", round(1 - sse / sst, 3))   # 1 - 54 / 56.75, about 0.049
print("R^2 sklearn:", round(r2_score(y, y_hat), 3))
```

The example makes the point: this model explains only about 5 percent of the variation in the target, which is barely better than predicting the mean. A model that is worse than the mean gives a negative value. The mistake is reporting a high R-squared as proof of a useful model. A model can have R-squared 0.9 and still miss by a large absolute amount, or have a high R-squared because the data spans a wide range, while the errors are unacceptable at the scale of interest. The limitation is that R-squared increases with more features even when they are noise, which is why adjusted R-squared or validation error is more reliable for comparing models. In industry, a team should report R-squared only with a baseline and an absolute error.

### Residual analysis

A single metric hides the pattern of the errors. Residual analysis looks at the residuals as a whole. A healthy model has residuals that look like random noise: centered on zero, with the same spread across the range of predictions, and no curve or cluster. Any structure in the residuals is a clue about what the model is missing.

From first principles, plot the residuals against the fitted values. A curve means the model misses a nonlinear pattern. A fan shape, where the spread grows with the prediction, means the errors are not constant, and a log transform of the target or a weighted model may help. A few points far from zero are outliers, which need inspection before deletion. Plot the residuals against each feature as well, to see whether the error depends on a variable the model does not use. Compare the average residual on each group, such as region or product, to check for bias.

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

rng = np.random.default_rng(0)
x = rng.uniform(1, 10, 300).reshape(-1, 1)
y = 2 * x.ravel() + 0.5 * x.ravel() ** 2 + rng.normal(0, 1, 300)  # a curved truth

model = LinearRegression().fit(x, y)
pred = model.predict(x)
resid = y - pred
plt.scatter(pred, resid, s=8)
plt.axhline(0, color="k")
plt.xlabel("predicted")
plt.ylabel("residual")
plt.savefig("residuals.png")
```

The plot will show a clear U shape, which means the linear model misses the curvature. The mistake is checking the metrics and not the residuals, and the second is removing outliers without reading their features first, which can remove the cases the business cares about most. The limitation of residual analysis is that it is a visual judgment for the most part, so compare plots across folds and keep a note of what you saw. In the market, a residual plot is often the first thing a model reviewer requests, because it shows at a glance whether the model is reliable across the range of the target.

---

## Hands-on exercises

1. Compute MAE, MSE, RMSE and R-squared by hand for five pairs of actual and predicted values of your choice. Check each result with scikit-learn.
2. Build two predictions with the same MAE and different RMSE. Explain what difference in the residuals causes the gap.
3. Compute the R-squared of a model that always predicts the training mean on a test set with a different mean. Explain why the result is negative.
4. Fit a linear regression to the curved data from this lesson, then to a log-transformed target. Compare the residual plots side by side and describe what the transform fixes.
5. Build a baseline that predicts the training median, and compare its MAE with a linear model on a validation split. Report the improvement as a percentage.
6. Add an outlier to the data and recompute MAE, RMSE and R-squared. Describe which metric changed the most and why.
7. Group the residuals by a categorical feature and compute the mean residual for each group. State whether the model is biased for any group, and how you would check that the bias is not a chance result.

---

## Project: Regression evaluation report with residual diagnostics

This is a medium-weight project. The metrics are short to compute, so the deliverable is a complete report with a baseline, a metric table and residual diagnostics that a reviewer can read in a few minutes.

**Repo name:** `100-days-ai-engineering-day-35-model-evaluation-regression`

**Goal:** evaluate a regression model against a baseline, report MAE, RMSE and R-squared with the error in units that matter, and diagnose its residuals by plot and by group.

**Repo layout:**

```
100-days-ai-engineering-day-35-model-evaluation-regression/
├── README.md
├── requirements.txt
├── evaluate.py
├── diagnose.py
├── results/
│   ├── metrics.md
│   └── residuals.png
└── tests/
    └── test_evaluate.py
```

**Starter code for `evaluate.py`:**

```python
import numpy as np


def regression_summary(y_true, y_pred):
    """Return a dict with MAE, MSE, RMSE and R-squared. Handle empty input by raising ValueError."""
    raise NotImplementedError


def baseline_mean_predictions(y_train, n):
    """Return an array of length n filled with the training mean."""
    raise NotImplementedError


def skill_score(model_mae, baseline_mae):
    """Return 1 - model_mae / baseline_mae. A positive value means the model beats the baseline."""
    raise NotImplementedError
```

**Starter code for `tests/test_evaluate.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from evaluate import regression_summary, baseline_mean_predictions, skill_score


def test_perfect_predictions_have_zero_error_and_r2_one():
    y = np.array([1.0, 2.0, 3.0])
    out = regression_summary(y, y)
    assert out["mae"] == 0.0
    assert out["rmse"] == 0.0
    assert out["r2"] == pytest.approx(1.0)


def test_rmse_is_sqrt_of_mse():
    y = np.array([10.0, 12.0, 15.0, 20.0])
    yh = np.array([11.0, 10.0, 15.0, 27.0])
    out = regression_summary(y, yh)
    assert out["rmse"] == pytest.approx(np.sqrt(out["mse"]))


def test_skill_score_positive_when_model_is_better():
    assert skill_score(model_mae=2.0, baseline_mae=4.0) == pytest.approx(0.5)


def test_baseline_predicts_training_mean():
    base = baseline_mean_predictions(np.array([1.0, 3.0]), n=4)
    assert np.allclose(base, 2.0)
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib`, `pandas` and `pytest`.
2. Choose a regression dataset with a clear license. The California housing dataset that ships with scikit-learn is a common choice, and the UCI Bike Sharing dataset is another.
3. Implement `regression_summary`, `baseline_mean_predictions` and `skill_score`. Make the tests pass.
4. In `diagnose.py`, split the data into train, validation and test parts. Fit a linear regression and a gradient boosting regressor on the training part, and compare each with the mean baseline on the validation part.
5. For the chosen model, plot the residuals against the predictions and save the figure to `results/residuals.png`. Describe any curve or fan shape in the README.
6. Compute the mean residual for the groups of one categorical or binned feature, and report any group with a clear bias.
7. Score the test set once with the chosen model, and record the summary in `results/metrics.md`, together with the baseline and the skill score.
8. Write the README, state the cost of a typical error in units the stakeholder understands, and commit the files. Push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python diagnose.py`
- [ ] The dataset source and license are stated
- [ ] The metric table includes MAE, RMSE, R-squared and the baseline MAE
- [ ] The residual plot is embedded with a description of what it shows
- [ ] The README states that the test set was used once

**Done when:**

- `pytest` passes all four tests
- The chosen model beats the mean baseline by a skill score above 0.2 on validation data, or the README explains why it does not
- The residual analysis names at least one pattern or bias, or states clearly that none was found
- You can explain in one paragraph why RMSE and MAE can give different rankings, and which one you would report to the business

**What to commit as evidence:** the source files, the tests, the metric table, the residual plot and the README with the diagnosis.

---

## Check your understanding

1. A model has an MAE of 3 and an RMSE of 3. Explain what this tells you about the size of its errors, and what you would expect to see if RMSE were much larger than MAE.
2. Two models have the same MSE on a dataset. Explain whether they must have the same MAE, and give a simple example.
3. A model reports an R-squared of 0.85 on the training set and -0.2 on the test set. Explain what the negative value means, and name the likely cause.
4. A residual plot shows a funnel shape, with small residuals at low predictions and large residuals at high predictions. Name the problem and suggest two fixes.
5. A colleague removes the three rows with the largest residuals and reports that RMSE dropped by 40 percent. Explain why this is a problem, and what you would do instead.
6. Explain why R-squared does not tell you whether the predictions are biased, and name a check that does.
7. A delivery company asks which metric to report to operations managers. Choose one, justify it with the cost of a typical error, and explain what the second metric would add.

### Answers

1. An MAE equal to the RMSE means that every absolute error is the same size, since the RMSE is always at least the MAE and equals it only when all errors have equal magnitude. If RMSE were much larger than MAE, a few large errors would dominate the average, and you would check the residual plot for those cases.
2. No. Two datasets of errors can have the same mean squared error and different mean absolute errors if one has a few large errors and the other has many moderate ones. For example, errors of 0, 0, 0, 6 and errors of 3, 3, 3, 3 have MSE 9 in both cases but MAE 1.5 and 3, so the comparison depends on how the errors are distributed.
3. A negative R-squared on the test set means the model's predictions are worse than predicting the test set's own mean. The likely cause is overfitting, or a shift in the data between training and test, where the model learned a pattern that does not hold on the new data.
4. The funnel shape means the spread of the residuals grows with the prediction, which is heteroscedasticity. Two fixes are to log-transform the target so the spread becomes more constant, or to use a model that accounts for varying variance, such as a weighted regression.
5. Removing the largest residuals changes the metric by removing the rows the model handles worst, which hides the model's real failures. Instead, investigate those rows to see whether they are errors in the data or real cases the model misses, and report the metric with and without them, with the reason stated.
6. R-squared compares the squared error with the variance of the target, so a model can have a high value while predicting consistently too high or too low. A check for bias is to compute the mean residual overall and within groups, and to look at the residual plot for a shift away from zero.
7. I would report MAE, because it is the typical error in the units managers use, and they can compare it directly with the cost of a late delivery. The second metric, RMSE, would add the information about large misses, which matters for operations that must avoid extreme delays, so it would appear as a secondary figure with an explanation.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Mean squared error](https://en.wikipedia.org/wiki/Mean_squared_error)
- Wikipedia: [Mean absolute error](https://en.wikipedia.org/wiki/Mean_absolute_error)
- Wikipedia: [Root-mean-square deviation](https://en.wikipedia.org/wiki/Root-mean-square_deviation)
- Wikipedia: [Coefficient of determination](https://en.wikipedia.org/wiki/Coefficient_of_determination)
- Wikipedia: [Heteroscedasticity](https://en.wikipedia.org/wiki/Heteroscedasticity)
- scikit-learn user guide, "Metrics and scoring: quantifying the quality of predictions", at scikit-learn.org under the stable documentation, including the regression metrics.

**Tutorials**

- scikit-learn documentation, `mean_absolute_error`, `root_mean_squared_error` and `r2_score`, with their notes on the interpretation.
- GeeksforGeeks, search "residual plot in regression" for worked examples of residual diagnostics.

**Watch**

- StatQuest with Josh Starmer (YouTube), "R-squared, Clearly Explained" and "Linear Regression, Clearly Explained" for the residual and variance intuition.
- MIT OpenCourseWare, 18.650 "Statistics for Applications", for the theory of least squares and residuals.

**Practice**

- Kaggle Learn, the "Intermediate Machine Learning" course, and the regression exercises that compare MAE with RMSE on a public dataset.

---

## Tomorrow

Tomorrow you tune the model rather than measure it. Hyperparameters such as the depth of a tree, the penalty in ridge regression or the number of neighbors are set before training, and the choice changes the score. You will separate parameters from hyperparameters, compare grid search with random search, and see how leakage and a careless validation loop can make a tuned model look better than it is. The pipeline pattern from Day 24 is the tool that keeps the process honest.

**Next:** [Day 36: Hyperparameters and Model Selection](Day-36-Hyperparameters-and-Model-Selection.md)

# Day 37: Interpretability and Error Analysis

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 25 (coefficients), Day 29 and Day 30 (tree importances), Day 34 and Day 35 (metrics and residuals). The explanation tools are powerful and easy to misread, so you will test each one against a case where you know the truth.

**By the end of this day, you will have:**

- A comparison of three importance measures on a model with known structure, and a note on where each misleads
- A partial dependence plot and an explanation of the assumption it makes
- A SHAP analysis on one model, with local and global views, and a list of its limits
- An error-slice report that finds the group where the model fails and a written diagnosis that separates data problems from model problems

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, for permutation importance and partial dependence
- SHAP 0.46 or newer (`pip install shap`), MIT license, for Shapley-value explanations
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the plots
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 25: coefficients and what they mean under the linear model
- Day 30: random forests and the impurity-based importances
- Day 34 and Day 35: metrics and residual analysis
- Day 36: pipelines, so that explanations use the same features the model used

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Feature importance](#feature-importance)
- [Model coefficients](#model-coefficients)
- [Partial dependence concept](#partial-dependence-concept)
- [SHAP concept](#shap-concept)
- [Error slices](#error-slices)
- [Failure analysis](#failure-analysis)
- [Dataset problems vs model problems](#dataset-problems-vs-model-problems)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Explaining and diagnosing a classifier](#project-explaining-and-diagnosing-a-classifier)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A model that scores well is not yet a model that a team can trust. Someone will ask why it made a decision, which group it fails for, and whether the failures are the model's fault or the data's. Those questions cannot be answered by a single metric. They need tools that open the model up, and a habit of checking what those tools say against cases where the answer is known.

Interpretability tools are easy to misuse. Feature importance can rank a noise feature highly. A coefficient can change sign when a correlated feature is added. A partial dependence curve assumes the features are independent. A SHAP value explains the model's output, not the real world. Each tool answers a specific question, and the skill is in matching the question to the tool and knowing what the answer does not say.

On the job, the strongest use of these tools is to find the next thing to fix. An error slice shows that a model fails on new customers in one region, and the diagnosis shows whether the cause is a missing feature, a label problem or a real change in the population. Each answer leads to a different action, so the analysis earns its keep only if it leads to one.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1950s to 1960s | Early work on sensitivity analysis and on the effect of inputs on outputs in regression and simulation models. | The question of which input drives an output is older than machine learning. |
| 2001 | Leo Breiman publishes "Random Forests" and describes the impurity-based importance and the permutation idea in the same work. | Permutation importance and the forest's own importance measure both come from this period. |
| 2001 | Jerome Friedman publishes the gradient boosting machine, which later makes partial dependence plots a standard tool for boosted trees. | The partial dependence plot is a common view of boosted model behavior. |
| 2010 | Erik Štrumbelj and Igor Kononenko introduce Shapley-based explanations for classifiers, applying a game-theory idea to features. | The Shapley approach to feature attribution was adapted to machine learning models in this period. |
| 2017 | Scott Lundberg and Su-In Lee publish "A Unified Approach to Interpreting Model Predictions", which introduces SHAP. | SHAP became the standard tool for local and global explanations of tree models and others. |

**Takeaway:** interpretability methods are older than machine learning, and the newest ones still answer narrow questions. Treat them as evidence that still needs checking.

---

## Glossary

- **Feature importance**: a score that ranks features by how much they contribute to predictions, under a specific definition.
- **Impurity-based importance**: a tree model's importance, computed as the total impurity decrease from splits on a feature.
- **Permutation importance**: the drop in a metric when a feature's values are randomly shuffled on held-out data.
- **Coefficient**: the weight of a feature in a linear model, which is interpreted holding the others fixed.
- **Partial dependence**: the average prediction as one feature varies, with the other features averaged over their values in the data.
- **Individual conditional expectation (ICE)**: the prediction curve for one row as one feature varies, before averaging.
- **Shapley value**: a way to divide a prediction among features, based on their average marginal contribution over all orderings.
- **SHAP**: a family of methods that compute Shapley-based attributions for a model's output.
- **Local explanation**: an explanation of one prediction.
- **Global explanation**: a summary of how the model behaves across many predictions.
- **Error slice**: a subset of rows defined by a feature value or a range, used to compare the model's performance across groups.
- **Failure analysis**: a structured review of the rows the model gets wrong, to find their common causes.

---

### Feature importance

Think of a team of engineers, each credited for part of a project's success. Importance is a way of splitting the credit. The split depends on the rules you choose. Different rules give different credits, and you should know which one you used.

From first principles, impurity-based importance adds up the impurity decrease from every split on each feature, across all trees, and normalizes the total. It is cheap, but it favors features with many distinct values and is computed on the training data, so it can reward features that the model memorized. Permutation importance measures how much a metric drops when one feature is shuffled on held-out data, which breaks its link to the target. It is more honest, but it is slower, and when two features are correlated, shuffling one still leaves the other to carry the information, so both look less important than they are.

```python
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.inspection import permutation_importance
from sklearn.model_selection import train_test_split

rng = np.random.default_rng(0)
X = rng.normal(size=(600, 5))
y = (X[:, 0] + 0.5 * X[:, 1] + rng.normal(0, 0.5, 600) > 0).astype(int)
X[:, 4] = rng.normal(size=600)                        # a pure-noise feature
X_tr, X_va, y_tr, y_va = train_test_split(X, y, random_state=0)

rf = RandomForestClassifier(n_estimators=300, random_state=0).fit(X_tr, y_tr)
print("impurity:   ", rf.feature_importances_.round(3))
perm = permutation_importance(rf, X_va, y_va, n_repeats=20, random_state=0)
print("permutation:", perm.importances_mean.round(3))
```

The mistake is to rank features by impurity importance and present the ranking as causal. The noise feature may get a non-zero score, and the ranking may change with the random seed. The limitation of permutation importance is that it depends on the metric and the data, and that correlated features share credit. Report the method, the metric, the data split and the seed. In industry, a team that presents importances should also show a noise feature as a control, so readers can see the floor of the scale.

### Model coefficients

A linear model's coefficient looks like the simplest explanation. It says how much the prediction changes when one feature increases by one unit and the others stay fixed. That "holding the others fixed" part is where most mistakes happen.

From first principles, in a linear model, the coefficient `w_j` is the partial effect of feature `j` on the prediction, given the other features in the model. If two features are correlated, the coefficients can be unstable and can flip sign when one is added or removed, because the model splits the shared signal between them in a way that depends on the data. Coefficients are also in the units of their features, so they must be compared on standardized inputs before their sizes are interpreted. A coefficient is not a causal effect, unless the study design supports that claim.

```python
import numpy as np
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import StandardScaler

rng = np.random.default_rng(0)
x1 = rng.normal(size=500)
x2 = x1 + 0.1 * rng.normal(size=500)     # almost a copy of x1
y = 2 * x1 + rng.normal(size=500)

only_x1 = LinearRegression().fit(StandardScaler().fit_transform(x1[:, None]), y)
both = LinearRegression().fit(StandardScaler().fit_transform(np.column_stack([x1, x2])), y)
print("x1 alone:", only_x1.coef_.round(2))
print("x1 with x2:", both.coef_.round(2))   # the signal is split between the two
```

The mistake is reading a coefficient for one of two correlated features as its effect on its own. The limitation of coefficients is that they describe the linear model, not the world, so a nonlinear relationship can produce misleading coefficients. In the market, a finance team that explains a pricing model by its coefficients should check for correlated inputs and for interactions before presenting them.

### Partial dependence concept

Imagine studying how the price of a house depends on its size. The price also depends on the neighborhood, the age and other features. A partial dependence plot holds the other features at their observed values, averages the predictions over them, and plots the average against the feature of interest. It answers the question "on average, what does the model predict as this feature changes?"

From first principles, for a feature `x_S` of interest and the other features `x_C`, the partial dependence is `f_S(x_S) = E over x_C of f(x_S, x_C)`, estimated by setting `x_S` to a grid value for every row, predicting, and averaging. The method assumes the feature is roughly independent of the others. When features are correlated, the averaging creates combinations of values that never occur in the data, and the curve can describe a population that does not exist. Individual conditional expectation curves show one line per row and reveal whether the average hides different behaviors for different rows.

```python
import numpy as np
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.inspection import PartialDependenceDisplay
import matplotlib.pyplot as plt

rng = np.random.default_rng(0)
X = rng.uniform(0, 10, (500, 2))
y = np.sin(X[:, 0]) + 0.1 * X[:, 1] + rng.normal(0, 0.1, 500)
model = GradientBoostingRegressor(random_state=0).fit(X, y)

PartialDependenceDisplay.from_estimator(model, X, features=[0, 1], kind="both")
plt.savefig("partial_dependence.png")
```

The mistake is reading the average curve as the effect for every row. If the curve is flat but half the rows go up and half go down, the average hides the pattern, and the ICE lines show it. The limitation is the independence assumption, which fails in many business datasets. In industry, partial dependence is often shown to non-technical reviewers, and it is safer when the features are roughly independent and when the ICE lines are shown next to the average.

### SHAP concept

Picture a team project where the reward must be split among the members fairly. Shapley values come from game theory and give each member the average of the extra value they bring across all possible orders of joining the team. SHAP applies this idea to a model: each feature gets a share of the difference between a prediction and the average prediction.

From first principles, for a prediction `f(x)` and a baseline value `E[f]`, the SHAP values `phi_j` satisfy `f(x) = E[f] + sum_j phi_j`. Each `phi_j` is the average marginal effect of feature `j` over the subsets of other features, with the missing features filled in from a background dataset. For tree models, a fast exact algorithm exists, which is why SHAP is practical for boosted trees and forests. A summary plot of the SHAP values across rows gives a global view, and the values for one row give a local explanation. The values describe how the model computes its output. They say nothing about the causal structure of the world.

```python
import numpy as np
import shap
from sklearn.ensemble import RandomForestClassifier

rng = np.random.default_rng(0)
X = rng.normal(size=(400, 4))
y = (X[:, 0] + 0.5 * X[:, 1] > 0).astype(int)
rf = RandomForestClassifier(n_estimators=200, random_state=0).fit(X, y)

explainer = shap.TreeExplainer(rf)
sv = explainer.shap_values(X)
# the output format changed across shap versions: take the positive class if a list is returned
pos = sv[1] if isinstance(sv, list) else (sv[:, :, 1] if sv.ndim == 3 else sv)
print("mean |SHAP| per feature:", np.abs(pos).mean(axis=0).round(3))
```

The mistake is treating SHAP values as the true effect of a feature. They are the model's attribution, and they depend on the background data and on correlations among the features. The limitation is that SHAP is slower for models without a fast algorithm, and that a local explanation can look convincing while the model is wrong. In industry, SHAP is widely used in credit and risk reviews, and teams check that the attributions match domain expectations before they trust them.

### Error slices

An average error hides the groups where the model is much worse. Error slices split the evaluation by a feature, such as region, age group or device type, and report the metric within each slice. A model with 90 percent accuracy overall may have 60 percent on a small group that matters.

From first principles, define a slice by a condition on one or more features, select the rows that satisfy it, and compute the metric on those rows only. Compare each slice with the overall metric and with the baseline on the same slice. Small slices have noisy metrics, so report the number of rows and a confidence interval, or at least a warning, next to each one. Slices should be chosen before looking at the results when possible, to avoid finding a pattern by chance, and any slice found after the fact should be checked on new data.

```python
import numpy as np
import pandas as pd
from sklearn.metrics import accuracy_score

rng = np.random.default_rng(0)
df = pd.DataFrame({
    "region": rng.choice(["north", "south", "east"], 600, p=[0.6, 0.3, 0.1]),
    "y": rng.integers(0, 2, 600),
})
df["pred"] = np.where(df["region"] == "east", 1 - df["y"], df["y"])   # fails on the east slice
df["correct"] = (df["pred"] == df["y"]).astype(int)

print("overall", round(accuracy_score(df["y"], df["pred"]), 3))
print(df.groupby("region").agg(rows=("correct", "size"), accuracy=("correct", "mean")).round(3))
```

The mistake is reporting only the overall metric. Another mistake is searching through dozens of slices and reporting the one that looks bad, without a correction or a check on new data. The limitation is sample size: a slice with 20 rows can show any accuracy by chance. In industry, error slices are the first thing a fairness and quality review asks for, and they form the basis of a model card's performance section.

### Failure analysis

Think of a mechanic who opens the engine after a breakdown and writes down what they see. Failure analysis works the same way: you collect the wrong predictions, read them, and group them by cause. The count of errors matters less than the reasons behind them.

From first principles, sample a set of misclassified or badly predicted rows from the validation or test part. Read each one with its features and its label, and assign it to a category such as missing information, mislabeled data, an unusual case, a shift in the population, or a pattern the model cannot represent. Count the categories. The largest category points to the next action. Sampling should be random within the error set, so that the categories reflect the real errors and not only the most extreme ones.

```python
import numpy as np
import pandas as pd

rng = np.random.default_rng(1)
errors = pd.DataFrame({
    "id": range(200),
    "true": rng.integers(0, 2, 200),
})
errors["pred"] = 1 - errors["true"]          # pretend every row here is an error
sample = errors.sample(n=30, random_state=0)  # random sample of errors to read by hand
print(sample.head())
# next: write a category for each row in a column called 'cause' and tabulate them
```

The mistake is reading only the worst errors, which may all be outliers with a single unusual cause, and missing the common category of small errors. The limitation of failure analysis is that it is manual and slow, and the categories depend on the reviewer. Two reviewers should code a sample independently and compare. In the market, a support or moderation team does exactly this, because the categories tell the product team where to invest.

### Dataset problems vs model problems

A failing model can be fixed in two very different ways. If the data is wrong, a new model will repeat the mistake. If the model is too weak, more data will not help as much as a better model would. The first job is to find out which problem you have.

From first principles, a dataset problem shows up as mislabeled rows, missing or leaking features, a shift between the training and deployment populations, or a label that does not match the business question. A model problem shows up as underfitting, a systematic pattern in the residuals or errors that the model could represent but does not, or an overfit that memorized noise. Test the diagnosis with cheap experiments: relabel a sample and check whether the errors change, train on the same data with a stronger model and see whether the gap closes, and compare the training and deployment distributions with simple summaries.

```python
import numpy as np
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score

rng = np.random.default_rng(0)
X = rng.normal(size=(800, 2))
y = ((X[:, 0] * X[:, 1]) > 0).astype(int)   # a pattern a linear model cannot represent

for name, model in [("linear", LogisticRegression()), ("forest", RandomForestClassifier(random_state=0))]:
    print(name, round(cross_val_score(model, X, y, cv=5).mean(), 3))
# a large gap suggests a model problem; similar scores for both suggest the data itself limits accuracy
```

The mistake is changing the model when the labels are wrong, or cleaning the data when the model is simply too simple. The limitation of this diagnosis is that the experiments are only as good as the controls you design. In industry, the decision to retrain, to relabel or to add features follows from this diagnosis, and it is worth writing down before any work begins.

---

## Hands-on exercises

1. Compute the impurity and permutation importances for a random forest on a dataset with one noise feature. Report whether the noise feature ranks above zero in each method.
2. Fit a linear regression on two correlated features and then on one of them. Explain in one paragraph how the coefficient changed and why.
3. Make a partial dependence plot with ICE lines for a gradient boosting model on a dataset with an interaction. Describe what the average hides.
4. Compute SHAP values for a tree model on a dataset of your choice. Plot the global summary and pick one row to explain locally. State whether the local explanation matches your intuition about that row.
5. Build an error-slice table for a classifier on a real dataset with at least two categorical features. Report the number of rows in each slice and flag any slice with fewer than 30 rows.
6. Sample 30 misclassified rows, write a cause for each one, and tabulate the causes. Name the most common cause and the action it suggests.
7. Run the dataset-versus-model experiment from this lesson with a mix of noise labels, for example flipping 10 percent of the labels. Describe how the gap between the models changes, and what that tells you.

---

## Project: Explaining and diagnosing a classifier

This is a medium-weight project. The tools are library calls. The main deliverable is a diagnosis, written in plain language, that names the largest problem and the action it calls for.

**Repo name:** `100-days-ai-engineering-day-37-interpretability-and-error-analysis`

**Goal:** explain a tree-based classifier with permutation importance, partial dependence and SHAP, find the error slices where it fails, and write a diagnosis that separates data problems from model problems.

**Repo layout:**

```
100-days-ai-engineering-day-37-interpretability-and-error-analysis/
├── README.md
├── requirements.txt
├── explain.py
├── slices.py
├── results/
│   ├── importance.md
│   ├── partial_dependence.png
│   ├── shap_summary.png
│   └── slices.md
└── tests/
    └── test_slices.py
```

**Starter code for `slices.py`:**

```python
import numpy as np
import pandas as pd


def error_table(df, group_col, y_col="y", pred_col="pred"):
    """Return a DataFrame with one row per group: rows, accuracy and error rate.
    Sort by error rate, highest first.
    """
    raise NotImplementedError


def flag_small_slices(table, min_rows=30):
    """Return a copy of the table with a boolean column 'small' that is True when rows < min_rows."""
    raise NotImplementedError
```

**Starter code for `tests/test_slices.py`:**

```python
import sys
from pathlib import Path

import pandas as pd

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from slices import error_table, flag_small_slices


def make_frame():
    return pd.DataFrame({
        "group": ["a", "a", "b", "b", "b", "c"],
        "y":     [1, 0, 1, 1, 0, 1],
        "pred":  [1, 0, 0, 1, 0, 0],
    })


def test_error_table_sorted_by_error_rate():
    t = error_table(make_frame(), "group")
    assert t.iloc[0]["group"] == "c"
    assert t.iloc[0]["error_rate"] == 1.0


def test_small_slices_are_flagged():
    t = flag_small_slices(error_table(make_frame(), "group"), min_rows=3)
    assert t.loc[t["group"] == "c", "small"].item() is True
    assert t.loc[t["group"] == "b", "small"].item() is False
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `pandas`, `scikit-learn`, `shap`, `matplotlib` and `pytest`.
2. Choose a classification dataset with at least 1,000 rows and at least two categorical or binned features. Use one with a clear license, and state its source in the README.
3. Implement `error_table` and `flag_small_slices`. Make the tests pass.
4. In `explain.py`, train a random forest on a training split. Compute the impurity and permutation importances on a validation split, and add a noise feature as a control. Save the importance table to `results/importance.md`.
5. Make a partial dependence plot for the top two features and save it to `results/partial_dependence.png`. Add one sentence on whether the average hides different behavior.
6. Compute the SHAP summary for the validation rows and save it to `results/shap_summary.png`. Note any feature whose SHAP ranking disagrees with the permutation importance.
7. In `slices.py`, compute the error table for each categorical feature on the test split, and save the flagged table to `results/slices.md`. Choose the worst slice with at least 30 rows.
8. Sample 30 errors from the worst slice and categorize each one. Write the categories in the README.
9. Write the diagnosis in the README: name the largest problem, say whether it is a data problem or a model problem, and name the one action you would take first.
10. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest`, `python explain.py` and `python slices.py`
- [ ] The dataset source and license are stated
- [ ] The importance table includes the noise control feature
- [ ] The partial dependence plot and SHAP summary are embedded with captions
- [ ] The worst error slice is named with its row count and error rate
- [ ] The diagnosis separates data problems from model problems and names one first action

**Done when:**

- `pytest` passes both tests
- The noise feature is shown to rank low in permutation importance, and the README explains any disagreement with the impurity ranking
- The worst error slice has at least 30 rows, and its error rate is clearly above the overall rate
- You can explain in one paragraph why the average partial dependence curve can mislead, and how the SHAP summary helps

**What to commit as evidence:** the source files, the tests, the importance table, the figures, the slice table, the categorized error sample and the written diagnosis.

---

## Check your understanding

1. A feature ranks second in impurity importance and near the bottom in permutation importance. Explain two reasons this can happen, and which result you would trust more for a decision.
2. Two features are highly correlated. One has a large positive coefficient in a model with only that feature, and a coefficient near zero when the other is added. Explain what happened, and what you would report.
3. A partial dependence curve is flat, but the model's predictions vary a lot across rows. Explain how this is possible, and name the plot that reveals it.
4. The SHAP values for one row say that a feature increased the prediction. Explain what this claim means, and what it does not mean.
5. An error slice of 12 rows has an error rate of 50 percent, while the overall rate is 10 percent. Explain why you should not yet act on this slice, and what you would do next.
6. A failure analysis finds that most errors come from rows with a missing value in one column. Is this a data problem or a model problem? Name the first action.
7. A model's training accuracy is 0.97 and its validation accuracy is 0.71. Name the diagnosis, and say which experiment would distinguish a data problem from a model problem.

### Answers

1. The impurity importance may reward a feature with many distinct values or one the trees used to memorize training noise, while permutation importance on held-out data measures how much the model depends on it for new rows. For a decision, I would trust the permutation importance on held-out data more, while checking it for correlated features that share credit.
2. The two features carry nearly the same information, so the model splits the shared signal between them, and the individual coefficients depend on which other features are present. I would report that the group of correlated features carries the signal, and I would not interpret either coefficient alone.
3. The model's effect for the feature may cancel out across rows, so the average is flat while individual rows go up and down. The individual conditional expectation plot shows one line per row and reveals this pattern.
4. The claim means that, in this model, that feature moved the prediction for this row upward relative to the average prediction, given the other features. It does not mean the feature caused the outcome, and it depends on the background data used to fill in missing features.
5. A group of 12 rows is too small to estimate an error rate reliably, so the 50 percent may be noise. I would collect more rows for that slice, or compute a confidence interval, and check whether the pattern holds on new data before acting.
6. This is most likely a data problem, because the model is receiving incomplete inputs for those rows. The first action is to check how missing values arise, whether they are informative, and whether an imputation or a separate flag improves the errors in the validation data.
7. The gap indicates overfitting, which is a model problem in the sense that the model memorized the training data. To separate a data problem from a model problem, I would train a simpler model with stronger regularization, check for leakage and label errors in a sample, and compare the training and deployment distributions. If the gap closes with simpler models, it was a model problem, and if it persists, the data may be the limit.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Feature importance](https://en.wikipedia.org/wiki/Feature_importance)
- Wikipedia: [Shapley value](https://en.wikipedia.org/wiki/Shapley_value)
- Wikipedia: [Partial dependence plot](https://en.wikipedia.org/wiki/Partial_dependence_plot)
- Wikipedia: [Explainable artificial intelligence](https://en.wikipedia.org/wiki/Explainable_artificial_intelligence)
- Lundberg and Lee (2017), "A Unified Approach to Interpreting Model Predictions", at arxiv.org. Search the title to find the paper.
- scikit-learn user guide, "Inspection" (permutation importance, partial dependence and ICE), at scikit-learn.org under the stable documentation.

**Tutorials**

- SHAP documentation, the introduction and the tree explainer guide, at shap.readthedocs.io.
- scikit-learn documentation, the example "Permutation Importance vs Random Forest Feature Importance (MDI)", which shows the bias of impurity importance with a noise feature.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Shapley values" if the search finds it, or the same channel's videos on feature importance.
- Google's Machine Learning Crash Course, the section on fairness and model evaluation, at developers.google.com/machine-learning/crash-course.

**Practice**

- Kaggle Learn, the "Machine Learning Explainability" course, which has notebooks on permutation importance, partial dependence and SHAP on public datasets.

---

## Tomorrow

Tomorrow you put the whole module into one project. You will build a complete classical machine learning product: a data pipeline, a baseline, several models, an experiment comparison, an evaluation with the metrics from Day 34 and Day 35, an error analysis from today, a model selection step from Day 36, and a simple inference interface. Every earlier day returns in that build, and the goal is a system you could hand to a colleague with confidence.

**Next:** [Day 38: Classical ML Project](Day-38-Classical-ML-Project.md)

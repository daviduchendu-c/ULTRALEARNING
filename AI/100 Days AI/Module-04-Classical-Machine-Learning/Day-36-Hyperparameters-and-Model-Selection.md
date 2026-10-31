# Day 36: Hyperparameters and Model Selection

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 24 (validation and cross-validation), Day 34 (metrics) and Day 30 (ensembles, which have many settings to tune). Most of the risk today is methodological, so read the leakage section twice.

**By the end of this day, you will have:**

- A clear list of the parameters a model learns and the hyperparameters you choose, for three model families
- A grid search and a random search on the same budget, with the results compared
- A nested cross-validation estimate that separates model selection from model evaluation
- A leakage check that shows how a careless tuning loop inflates a score, and how a pipeline prevents it

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, which includes `GridSearchCV`, `RandomizedSearchCV` and the pipeline tools
- SciPy 1.13 or newer (`pip install scipy`), BSD 3-Clause, for the distributions used in random search
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the search plots
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 24: the three-way split and cross-validation
- Day 29 and Day 30: decision trees and ensembles, and the settings they expose
- Day 34: choosing a metric and a threshold on validation data
- Basic logarithms, since many hyperparameters are searched on a log scale

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Parameters vs hyperparameters](#parameters-vs-hyperparameters)
- [Grid search](#grid-search)
- [Random search](#random-search)
- [Cross-validation](#cross-validation)
- [Leakage](#leakage)
- [Pipeline-based evaluation](#pipeline-based-evaluation)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Tuning and nested evaluation with an honest score](#project-tuning-and-nested-evaluation-with-an-honest-score)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every model has two kinds of settings. The first kind the training process fills in: the coefficients of a linear model, the splits of a tree, the weights of a neural network. The second kind you choose before training: how deep the tree may grow, how strong the penalty is, how many neighbors to use. Those are hyperparameters, and choosing them well is the difference between a model that works and one that only looks good in a notebook.

The risk is that tuning is itself a form of learning. If you try two hundred settings and report the best validation score, that score is biased upward, because you selected the setting that happened to fit the validation rows. If the tuning process also sees the test rows, or if preprocessing uses the full dataset, the reported number is misleading in a way the business will eventually discover. Good model selection is therefore mostly about keeping the evaluation honest.

On the job, the question to ask of any tuned model is "what did the search see, and what did it never see?" Today you learn the tools for tuning and the discipline that keeps the score meaningful.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1930s | Ronald Fisher's work on experimental design introduces the idea of comparing treatments on separate data and randomizing assignment. | The discipline of separating the choice of design from the measurement is the root of honest model selection. |
| 1970 | Arthur Hoerl and Robert Kennard publish ridge regression, which makes the penalty strength a tuning choice. | Many modern hyperparameters are regularization strengths chosen by this kind of search. |
| 1974 | M. Stone publishes "Cross-Validatory Choice and Assessment of Statistical Predictions", which uses held-out data to estimate prediction error. | Cross-validation became the standard way to choose settings without a separate test set. |
| 2012 | James Bergstra and Yoshua Bengio publish "Random Search for Hyper-Parameter Optimization", showing random search often beats grid search with the same budget. | Random search is now the default first step for tuning in many libraries and teams. |
| 2010s onward | Bayesian optimization and successive halving tools such as Optuna and Hyperband become common for expensive models. | Search budgets are now managed, and tuning is treated as a costed step rather than free exploration. |

**Takeaway:** the search methods are simple, and the hard part is the design of the evaluation that wraps them.

---

## Glossary

- **Parameter**: a value that training learns from data, such as a regression coefficient or a tree split.
- **Hyperparameter**: a setting chosen before training, such as tree depth, regularization strength or learning rate.
- **Search space**: the set of candidate values for each hyperparameter.
- **Grid search**: trying every combination of a finite set of values for the hyperparameters.
- **Random search**: sampling combinations from distributions for the hyperparameters, with a fixed budget.
- **Budget**: the number of configurations or training runs the search may use.
- **Cross-validated score**: the average score across folds for one configuration.
- **Nested cross-validation**: an outer loop that estimates the performance of the whole tuning procedure, and an inner loop that does the tuning.
- **Leakage**: information from outside the training data entering the model, which inflates the score.
- **Pipeline**: a sequence of preprocessing and modeling steps fitted together, so that each step learns only from the training part.
- **Selection bias (optimism)**: the upward bias in a score caused by choosing the best of many candidates on the same data.

---

### Parameters vs hyperparameters

Think of a chef and an oven. The chef chooses the temperature and the cooking time before the dish goes in, and those choices are hyperparameters. The dish's texture and browning are what happens during cooking, which is what the parameters capture. Both matter, but they are set at different moments.

From first principles, a parameter is a quantity whose best value the training algorithm finds by optimizing a loss on the training data: the coefficients `w` and intercept `b` in linear regression, or the split thresholds in a tree. A hyperparameter controls how the training runs or how flexible the model is, and the training algorithm does not optimize it on the same loss. The regularization strength `lambda` in ridge regression, for example, cannot be chosen by minimizing the training loss, because the training loss always falls as `lambda` shrinks toward zero. That is why hyperparameters are chosen on held-out data.

```python
from sklearn.linear_model import Ridge
from sklearn.tree import DecisionTreeClassifier

ridge = Ridge(alpha=1.0)            # alpha is a hyperparameter
tree = DecisionTreeClassifier(max_depth=5, min_samples_leaf=10)   # both are hyperparameters
print(ridge.get_params()["alpha"], tree.get_params()["max_depth"])
```

The mistake is treating a hyperparameter as if it were learned from the training loss, which always favors more flexibility. The limitation of the split is that some settings blur the line: a neural network's learning rate is a hyperparameter, but the number of epochs can be chosen by early stopping on validation data, which is a kind of tuning. In industry, teams keep a clear record of which values were chosen, how, and on which data, so that a model can be reproduced.

### Grid search

Imagine testing a new recipe by trying every combination of three oven temperatures and three cooking times. That is a grid. It is thorough and easy to explain, but the number of combinations multiplies with each new setting, so the grid grows fast.

From first principles, grid search defines a finite set of values for each hyperparameter, evaluates every combination with cross-validation, and picks the one with the best mean score. For `m` hyperparameters with `v` values each, the search needs `v^m` configurations, multiplied by the number of folds. The method is deterministic and easy to audit. Its weakness is that many combinations are wasted on settings that do not matter, and the important hyperparameter may be sampled at only a few values.

```python
from sklearn.model_selection import GridSearchCV, StratifiedKFold
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
grid = {"max_depth": [3, 6, None], "min_samples_leaf": [1, 5, 10]}
search = GridSearchCV(
    RandomForestClassifier(n_estimators=200, random_state=0),
    grid,
    cv=StratifiedKFold(5, shuffle=True, random_state=0),
    scoring="roc_auc",
    n_jobs=-1,
)
search.fit(X, y)
print(search.best_params_, round(search.best_score_, 3))
```

The mistake is building a grid with values that are too coarse or that sit at the edge of the range, so the best value is at a boundary and the search has not really found an optimum. Check whether the best value is at the edge, and extend the grid if it is. The limitation of grid search is cost, and it is wasteful when only one hyperparameter matters. In industry, a grid is a good choice for a small number of hyperparameters with known ranges, and a random search is preferred when the space is larger.

### Random search

Suppose you have one knob that matters a lot and two that barely matter. A grid wastes most of its tries on the two unimportant knobs. Random search samples each knob independently, so every trial tests a new value of the important one.

From first principles, random search draws configurations from a distribution for each hyperparameter. Continuous values such as the regularization strength are best sampled on a log scale, so that each order of magnitude gets equal attention. For a fixed budget, random search explores more distinct values of each hyperparameter than a grid of the same size, which is why it often finds good settings faster when only a few hyperparameters matter. It gives no guarantee of finding the global optimum, but it is simple and easy to run in parallel.

```python
from scipy.stats import loguniform, randint
from sklearn.model_selection import RandomizedSearchCV, StratifiedKFold
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
pipe = make_pipeline(StandardScaler(), LogisticRegression(max_iter=5000))
dist = {"logisticregression__C": loguniform(1e-3, 1e2)}
search = RandomizedSearchCV(
    pipe, dist, n_iter=30, cv=StratifiedKFold(5, shuffle=True, random_state=0),
    scoring="roc_auc", random_state=0, n_jobs=-1,
)
search.fit(X, y)
print(round(search.best_params_["logisticregression__C"], 4), round(search.best_score_, 3))
```

The mistake is sampling a parameter uniformly across a wide range, such as from 0 to 100, which puts almost all samples near the top and misses the small values that matter. Use a log scale. The limitation of random search is that it can miss a narrow optimum, and that the result depends on the random seed and the budget. Report both, and rerun with a different seed to see whether the best value is stable. In the market, random search is often the first tuning tool a team uses, and a Bayesian method is added when training is expensive enough to justify the extra complexity.

### Cross-validation

Cross-validation is the tool that gives each hyperparameter setting a fair score. Without it, a setting can look good only because it happened to suit one validation split. With it, the setting must work across several splits, which reduces the luck in the choice.

From first principles, for each candidate configuration, split the training part into `k` folds, train on `k - 1` folds, score on the held-out fold, and average the scores. The configuration with the best average is selected. This estimates the expected performance of that configuration. But the estimate is optimistic if you also use it to report the final performance, because the same folds were used to choose the configuration. That is the reason for a separate test set, or for nested cross-validation, which the project uses.

```python
from sklearn.model_selection import cross_val_score, KFold
from sklearn.tree import DecisionTreeClassifier
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
cv = KFold(n_splits=5, shuffle=True, random_state=0)
for depth in (2, 4, 8):
    scores = cross_val_score(DecisionTreeClassifier(max_depth=depth, random_state=0), X, y, cv=cv)
    print(depth, round(scores.mean(), 3), round(scores.std(), 3))
```

The mistake is reporting the cross-validated score of the best configuration as the final performance. That number is selected and therefore optimistic. The limitation of cross-validation is cost, which multiplies with the grid size, and the fact that the folds must respect the structure of the data, such as time order or repeated patients. In industry, the final number reported to a stakeholder comes from a test set or an outer loop that the tuning never saw.

### Leakage

Leakage is the most common reason a tuned model fails in production. It happens when information that would not be available at prediction time leaks into training or tuning. The model's score then reflects the leak, not the real signal. Leakage can be subtle, and it is easiest to spot by asking what the model would know at the moment it makes a prediction.

From first principles, there are two main kinds. Target leakage occurs when a feature is computed using the target or a value that is only known after the event, such as a diagnosis code entered after treatment. Train-test contamination occurs when preprocessing, such as scaling, imputation or feature selection, is fitted on the full dataset before splitting. Both inflate the score during tuning, and they often pass unnoticed because the validation score looks excellent. The demonstration below uses a scaler fitted on the full data, which leaks the statistics of the held-out rows.

```python
import numpy as np
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_val_score, StratifiedKFold
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline
from sklearn.feature_selection import SelectKBest, f_classif

rng = np.random.default_rng(0)
X = rng.normal(size=(100, 2000))     # 2000 noise features
y = rng.integers(0, 2, 100)          # labels are random: the true score is 0.5

cv = StratifiedKFold(5, shuffle=True, random_state=0)
# leaky: select the top features using all rows, then cross-validate
Xk = SelectKBest(f_classif, k=20).fit_transform(X, y)
leaky = cross_val_score(LogisticRegression(max_iter=1000), Xk, y, cv=cv).mean()
# honest: selection happens inside each fold
honest_pipe = make_pipeline(SelectKBest(f_classif, k=20), LogisticRegression(max_iter=1000))
honest = cross_val_score(honest_pipe, X, y, cv=cv).mean()
print("leaky", round(leaky, 3), " honest", round(honest, 3))
```

The leaky version scores well above chance on pure noise, while the honest version stays near 0.5. The mistake is to trust the leaky number and to tune on it. The limitation of this check is that you only see the leak if you have a control such as random labels, so use one when a score looks too good. In industry, leakage is responsible for many models that impress in review and fail at deployment, and the defense is a pipeline plus a clear statement of what was fitted on which rows.

### Pipeline-based evaluation

A pipeline is a recipe that keeps every fitted step inside the training fold. The scaler, the imputer, the feature selector and the model are fitted together, and each held-out fold is transformed using what was learned from its training fold only. The pipeline turns honest evaluation from a discipline you must remember into a structure the code enforces.

From first principles, a pipeline is a sequence of transformers followed by an estimator. When it is fitted, each transformer learns from the training data and passes its output to the next. When it predicts, it applies the learned transformations to new data. Passing a pipeline to `GridSearchCV` or `cross_val_score` means that every fold refits all the steps, so the leakage in the previous section cannot occur. Hyperparameters of any step are addressed with the step name and a double underscore, as in `selectkbest__k`.

```python
from sklearn.model_selection import GridSearchCV, StratifiedKFold
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
pipe = make_pipeline(StandardScaler(), SelectKBest(f_classif), LogisticRegression(max_iter=5000))
grid = {"selectkbest__k": [5, 10, 20], "logisticregression__C": [0.1, 1.0, 10.0]}
search = GridSearchCV(pipe, grid, cv=StratifiedKFold(5, shuffle=True, random_state=0), scoring="roc_auc")
search.fit(X, y)
print(search.best_params_, round(search.best_score_, 3))
```

The mistake is building the pipeline and then calling `fit_transform` on the full data outside it, which brings back the leak. Another mistake is forgetting that `GridSearchCV` refits the best model on all the data it was given, so the held-out test set must be kept outside the search. The limitation of a pipeline is that it does not stop you from choosing the grid after looking at the test set, which is a human leak. In the market, a team that uses pipelines still needs a rule about when the test set is opened, and the rule must be written down before the first run.

---

## Hands-on exercises

1. List the parameters and hyperparameters of a logistic regression with an L2 penalty, a decision tree and a random forest. Mark each one and say how you would choose it.
2. Run a grid search over `max_depth` and `min_samples_leaf` for a decision tree on a dataset of your choice. Report the best setting and check whether it is at the edge of the grid.
3. Run a random search with 30 iterations over the same space, using the same cross-validation folds. Compare the best score with the grid search, and state which one used fewer evaluations.
4. Repeat the random search with three different seeds and report the spread of the best score. Explain what the spread tells you about the search.
5. Run the leakage demonstration from this lesson with 20, 200 and 2000 noise features. Describe how the leaky score changes and why.
6. Build a nested cross-validation loop with an outer 5-fold and an inner 3-fold grid search. Report the outer mean score and compare it with the best inner score from one full search.
7. Create a pipeline with a scaler, a feature selector and a logistic regression. Search over the number of selected features and the regularization strength, then write one sentence on what you would report as the final score and why.

---

## Project: Tuning and nested evaluation with an honest score

This is a medium-weight project. The code is short. The main deliverable is an honest report that shows how much the tuned score was inflated by selection, and what the test score says about the model.

**Repo name:** `100-days-ai-engineering-day-36-hyperparameters-and-model-selection`

**Goal:** tune a pipeline with grid search and random search on the same budget, compare them, estimate the score with nested cross-validation, and show the size of the selection bias.

**Repo layout:**

```
100-days-ai-engineering-day-36-hyperparameters-and-model-selection/
├── README.md
├── requirements.txt
├── pipeline.py
├── tune.py
├── results/
│   ├── search_comparison.md
│   └── nested_vs_naive.md
└── tests/
    └── test_pipeline.py
```

**Starter code for `pipeline.py`:**

```python
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler


def build_pipeline(seed=0):
    """Return an unfitted pipeline: StandardScaler, SelectKBest(f_classif), LogisticRegression.
    Use random_state=seed on the model and max_iter=5000.
    """
    raise NotImplementedError


def search_space():
    """Return a dict for grid search and a dict of distributions for random search.
    The grid uses k in {5, 10, 20} and C in {0.1, 1.0, 10.0}.
    The distributions use k as a randint and C as a log-uniform distribution.
    Return (grid, dists).
    """
    raise NotImplementedError
```

**Starter code for `tests/test_pipeline.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from pipeline import build_pipeline, search_space


def test_pipeline_has_three_named_steps():
    pipe = build_pipeline()
    assert list(pipe.named_steps) == ["standardscaler", "selectkbest", "logisticregression"]


def test_grid_keys_match_pipeline_steps():
    grid, dists = search_space()
    assert "selectkbest__k" in grid
    assert "logisticregression__C" in grid
    assert "selectkbest__k" in dists


def test_pipeline_fits_on_random_data():
    rng = np.random.default_rng(0)
    X = rng.normal(size=(60, 30))
    y = rng.integers(0, 2, 60)
    pipe = build_pipeline().set_params(selectkbest__k=5).fit(X, y)
    assert pipe.predict(X).shape == (60,)
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `scipy` and `pytest`.
2. Use a binary dataset with at least 500 rows. The breast cancer dataset from scikit-learn is a good choice. Keep a test part aside before any search.
3. Implement `build_pipeline` and `search_space`. Make the tests pass.
4. In `tune.py`, run `GridSearchCV` and `RandomizedSearchCV` on the training part with the same 5-fold stratified folds and roughly the same number of fits. Save the best parameters and scores to `results/search_comparison.md`.
5. Run nested cross-validation with an outer 5-fold and an inner 3-fold randomized search. Compare the outer mean with the best inner score from step 4, and write the difference to `results/nested_vs_naive.md`. The difference is the selection bias.
6. Choose the final configuration from the inner search on the full training part, score the held-out test part once, and record the number.
7. Write the README. Explain the difference between the naive score and the nested score, and why the nested score is the honest estimate.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python tune.py`
- [ ] The dataset source and license are stated
- [ ] The grid and random search results are in a table with their budgets
- [ ] The nested and naive scores are compared, with the selection bias stated
- [ ] The README states that the test set was used once, after the choice was made

**Done when:**

- `pytest` passes all three tests
- The naive best score is higher than the nested score, and the README explains why
- The grid and random search results are compared on the same budget, with a clear statement of which one you would use next time
- You can explain in one paragraph what leakage would have done to these numbers if the scaler had been fitted on the full data

**What to commit as evidence:** the source files, the tests, the search comparison, the nested comparison, and the README with the single test score.

---

## Check your understanding

1. A colleague says the regularization strength in ridge regression can be chosen by minimizing the training loss. Explain why this is wrong, and describe the correct method.
2. A grid search returns the best value of `max_depth` at the largest value in the grid. What does this suggest, and what would you do next?
3. Explain why random search can beat grid search when only one of several hyperparameters matters.
4. You tune a model on 200 configurations and report the best cross-validated score as the final performance. Explain the bias, and describe how to estimate the honest performance.
5. A scaler is fitted on the full dataset before splitting into folds. Describe the leak, and say whether the leak can make a model with random labels look better than chance.
6. A pipeline includes feature selection inside the search. Explain why this is safer than selecting features before the search, and name the one leak that a pipeline cannot prevent.
7. A team reports a test score for a model after running twenty tuning experiments with the test set in view. What is wrong with the report, and what should they do now?

### Answers

1. The training loss always falls as the penalty shrinks toward zero, so minimizing it would choose no penalty and remove the regularization. The correct method is to choose the penalty on held-out data, using cross-validation, and then train on all the training data with the chosen value.
2. The best value sits at the edge of the grid, so the true optimum may be larger. Extend the grid in that direction and search again, checking that the best value moves inside the new range.
3. Grid search spends most of its evaluations on combinations of the unimportant hyperparameters, so the important one is tested at only a few values. Random search tests a new value of every hyperparameter in each trial, so it explores the important one more thoroughly for the same budget.
4. The best of 200 configurations is selected because it scored highest on the same folds, so its score is biased upward by the selection. The honest estimate comes from nested cross-validation, where the outer folds are never used for the choice, or from a separate test set scored once.
5. The scaler learns the mean and the variance from the held-out rows, so the training features contain information about those rows. With random labels, this can still produce an apparent signal, which is why the score can look better than chance. The fix is to put the scaler inside the pipeline so that each fold refits it.
6. A pipeline refits the selection and the scaler inside each fold, so the held-out rows never influence the chosen features. The leak a pipeline cannot prevent is the human one: choosing the grid, the features or the metric after looking at the test results.
7. The test set was used to guide the tuning, so it is no longer an unbiased measure. The team should report the result honestly as a tuning result, state how many experiments were run, and if possible collect a new test set or wait for new data before reporting a final number.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Hyperparameter optimization](https://en.wikipedia.org/wiki/Hyperparameter_optimization)
- Wikipedia: [Cross-validation (statistics)](https://en.wikipedia.org/wiki/Cross-validation_(statistics))
- Wikipedia: [Leakage (machine learning)](https://en.wikipedia.org/wiki/Leakage_(machine_learning))
- Wikipedia: [Hyperparameter (machine learning)](https://en.wikipedia.org/wiki/Hyperparameter_(machine_learning))
- Bergstra and Bengio (2012), "Random Search for Hyper-Parameter Optimization", in the *Journal of Machine Learning Research*. Search the title to find the open-access PDF.
- scikit-learn user guide, "Tuning the hyper-parameters of an estimator" and "Common pitfalls and recommended practices", at scikit-learn.org under the stable documentation.

**Tutorials**

- scikit-learn documentation, `GridSearchCV`, `RandomizedSearchCV` and the example "Nested versus non-nested cross-validation".
- Optuna documentation, the tutorial on a first optimization study, at optuna.org, for a brief look at a Bayesian search tool.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Cross Validation, Clearly Explained!!!" and the related video on bias and variance.
- Google Machine Learning Crash Course, the sections on validation and on the training, validation and test split, at developers.google.com/machine-learning/crash-course.

**Practice**

- Kaggle Learn, the "Data Leakage" course, which includes exercises that show target leakage and train-test contamination on real data.

---

## Tomorrow

Tomorrow you ask why a model made the decision it did. Feature importance, model coefficients, partial dependence and SHAP each give a different answer, and each can mislead if you read it too literally. You will also look at error slices and failure analysis, which tell you whether the errors fall on a group that matters, and you will learn to separate dataset problems from model problems. Those tools turn a good score into a model you can defend.

**Next:** [Day 37: Interpretability and Error Analysis](Day-37-Interpretability-and-Error-Analysis.md)

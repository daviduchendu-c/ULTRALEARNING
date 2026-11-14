# Day 30: Ensemble Learning

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 29 (decision trees) and Day 24 (bias, variance and cross-validation). The boosting section uses gradients, so Day 25 helps.

**By the end of this day, you will have:**

- An explanation of why averaging weak models reduces variance, backed by a simulation you ran
- A bagged ensemble and a random forest, each compared with a single tree under the same cross-validation
- A gradient boosting model with a learning-rate and tree-count study
- A voting and stacking comparison that uses out-of-fold predictions, not the training rows

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, which includes random forests, gradient boosting and stacking
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the learning curves
- pytest 8.0 or newer (`pip install pytest`)
- Optional: XGBoost 2.1 or newer (`pip install xgboost`), Apache 2.0 license, if you want to compare with a dedicated boosting library
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 29: how a decision tree splits and how depth controls variance
- Day 24: bias, variance, overfitting and k-fold cross-validation
- Day 25: the gradient of a squared loss, used again in boosting
- Comfort with averages and the idea of a sample drawn with replacement

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Bagging](#bagging)
- [Random forests](#random-forests)
- [Boosting](#boosting)
- [Gradient boosting](#gradient-boosting)
- [Voting](#voting)
- [Stacking concepts](#stacking-concepts)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Ensemble comparison on a tabular dataset](#project-ensemble-comparison-on-a-tabular-dataset)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Ask one doctor for a diagnosis and you get one opinion, which may be biased or unlucky. Ask ten doctors who studied slightly different cases and average their judgments, and the result is usually more reliable than any one of them. Ensemble learning applies this idea to models. It trains several models and combines their predictions, and the combination often beats the best single model.

In practice, ensembles of trees are the strongest default for structured, tabular data. Gradient boosted trees are among the most common winning approaches in public machine learning competitions on tabular problems, and they are standard in credit scoring, insurance pricing, churn prediction and ranking systems. They are also harder to explain than a single tree, so a team has to decide how much accuracy is worth how much interpretability.

On the job, the ensemble question is practical. Which method fits the data size and the latency budget? How do you tune it without overfitting the validation set? When does stacking help, and when is it only a complication? Today you answer those questions with measurements.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1990 | Lars Hansen and Peter Salamon show that averaging neural networks trained from different starts can reduce error. | An early statement of the benefit of combining models. |
| 1996 | Leo Breiman publishes "Bagging Predictors", introducing bootstrap aggregating. | Bagging is the basis of random forests and of many variance-reduction methods. |
| 1995 to 1997 | Yoav Freund and Robert Schapire develop AdaBoost, which reweights hard examples and combines weak learners. | The first practical boosting algorithm, later recognized with the Gödel Prize in 2003. |
| 2001 | Leo Breiman publishes "Random Forests" in *Machine Learning*, adding random feature selection to bagging. | Random forests became one of the most reliable default tabular models. |
| 2001 | Jerome Friedman publishes "Greedy Function Approximation: A Gradient Boosting Machine". | Gradient boosting, which fits each new tree to the negative gradient of the loss, is the framework behind modern libraries. |
| 2014 onward | XGBoost, LightGBM and CatBoost implement fast, regularized gradient boosting for large tabular data. | These libraries are the production tools most teams use for boosting today. |

**Takeaway:** the ensemble idea is a few decades old, and its strength comes from two simple ways to combine models: average many independent-ish learners, or add learners that fix each other's mistakes.

---

## Glossary

- **Ensemble**: a set of models whose predictions are combined into one.
- **Base learner (weak learner)**: one model in the ensemble, often a shallow tree.
- **Bootstrap sample**: a sample of the training rows drawn with replacement, usually the same size as the original, so about 63 percent of unique rows appear in each one.
- **Bagging (bootstrap aggregating)**: training each base learner on its own bootstrap sample and averaging or voting their predictions.
- **Out-of-bag (OOB) error**: an error estimate from each row's predictions using only the trees that did not see that row.
- **Random forest**: bagging of decision trees with a random subset of features considered at each split.
- **Boosting**: training learners in sequence, where each one focuses on the examples the previous ones got wrong.
- **Gradient boosting**: boosting in which each new tree fits the negative gradient of the loss with respect to the current predictions.
- **Learning rate (shrinkage)**: a multiplier that scales each tree's contribution in boosting.
- **Hard voting**: combining classifiers by majority of their predicted labels.
- **Soft voting**: combining classifiers by averaging their predicted probabilities.
- **Stacking**: training a second-level model on the out-of-fold predictions of several base models.
- **Out-of-fold prediction**: a prediction for a row made by a model that did not see that row during training.

---

### Bagging

Imagine asking the same question to many people, each of whom has read a slightly different book. Each person's answer is noisy, but the errors go in different directions, so the average is steadier than any single answer. Bagging creates those different readers by resampling the data.

From first principles, draw `B` bootstrap samples from the training set, fit a model to each, and average their predictions for regression or take a majority vote for classification. If the base learners were independent, averaging `B` of them with variance `sigma^2` would give variance `sigma^2 / B`. They are not fully independent, because they share the same data, so the reduction is smaller. Bagging does not reduce bias much, which is why it works best with low-bias, high-variance learners such as deep trees. A bootstrap sample leaves out about 37 percent of the rows, and those left-out rows give a free validation estimate called the out-of-bag error.

```python
import numpy as np
from sklearn.tree import DecisionTreeRegressor

rng = np.random.default_rng(0)
x = np.linspace(-3, 3, 200).reshape(-1, 1)
y_true = np.sin(x).ravel()

def one_tree_prediction():
    idx = rng.integers(0, 200, 200)     # bootstrap sample with replacement
    y = y_true + rng.normal(0, 0.3, 200)
    return DecisionTreeRegressor(max_depth=6).fit(x[idx], y[idx]).predict(x)

single = one_tree_prediction()
bagged = np.mean([one_tree_prediction() for _ in range(50)], axis=0)
print("single tree error", np.mean((single - y_true) ** 2).round(4))
print("bagged error     ", np.mean((bagged - y_true) ** 2).round(4))
```

The mistake is expecting bagging to fix a model that is too simple. If the base learner has high bias, averaging only makes a stable wrong answer. The limitation is cost: `B` models take `B` times as long to train and use more memory at prediction time. In industry, bagging is often the cheapest way to make a noisy model stable, and it is built into many libraries as a wrapper around any learner.

### Random forests

Think of a committee in which each member is given a different subset of the evidence and a different subset of the questions. Their answers are more varied, so their combined vote is more robust. A random forest does this by adding one more source of randomness to bagging.

From first principles, a random forest is bagging of decision trees, with one change: at each split, the tree considers only a random subset of the features, often the square root of the total for classification. This decorrelates the trees. If one feature is very strong, plain bagging would pick it at the top of almost every tree, so the trees would look alike and their errors would not cancel. Random feature selection breaks that pattern. The out-of-bag error gives an honest estimate of performance without a separate validation set, and feature importances can be read from the impurity decrease summed across the forest.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import cross_val_score

X, y = load_breast_cancer(return_X_y=True)
rf = RandomForestClassifier(n_estimators=300, oob_score=True, random_state=0, n_jobs=-1)
print("cv accuracy", cross_val_score(rf, X, y, cv=5).mean().round(3))
rf.fit(X, y)
print("out-of-bag accuracy", round(rf.oob_score_, 3))
```

The mistake is treating feature importances as causal. Impurity-based importances favor features with many distinct values, and they can be misleading when features are correlated. Use permutation importance on held-out data for a more careful reading, which Day 37 covers. The limitation of random forests is that they are less accurate than boosting on many tabular tasks, and their predictions are slow to compute when the forest is large. In the market, a random forest is a common first serious model for insurance and retail data, because it needs little tuning and rarely fails badly.

### Boosting

Consider a student who takes a practice test, looks at the questions they got wrong, and studies those harder before the next test. Each round focuses effort where the errors are. Boosting does this with learners: each new model pays more attention to the examples the ensemble still gets wrong.

From first principles, AdaBoost trains a sequence of weak learners. After each round, it increases the weights of the misclassified rows and decreases the weights of the correct ones, so the next learner focuses on the hard cases. The final prediction is a weighted vote in which each learner's weight depends on its error. Boosting reduces bias, because each step adds capacity where the model is wrong, but it can also overfit if run too long on noisy labels. The sequential nature means it cannot be parallelized across trees the way bagging can.

```python
from sklearn.ensemble import AdaBoostClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import cross_val_score
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
ada = AdaBoostClassifier(
    estimator=DecisionTreeClassifier(max_depth=1), n_estimators=200, random_state=0
)
print("AdaBoost cv accuracy", cross_val_score(ada, X, y, cv=5).mean().round(3))
```

The mistake is running boosting on data with many mislabeled rows, because the algorithm keeps pushing weight onto the mislabeled examples. The limitation is that boosting is sensitive to noise and needs careful tuning of the number of rounds. In industry, boosting is used where the signal is subtle and the labels are clean, such as fraud signals in transaction data that have been audited.

### Gradient boosting

Picture a sculptor who has carved a rough shape. Each new cut removes the largest remaining error, which is the direction that reduces the shape's distance from the target. Gradient boosting works this way with a loss function: each tree is fitted to the negative gradient of the loss, which is the direction of steepest improvement for each row.

From first principles, start with a constant prediction, such as the mean of the targets. At each step, compute the residuals, which for squared error are the negative gradients, fit a shallow tree to them, and add that tree's output, multiplied by a learning rate, to the current prediction. For a different loss, such as log loss for classification, the negative gradient is different, but the procedure is the same. Smaller learning rates require more trees but usually generalize better. Early stopping, using a validation set, picks the number of trees that minimizes validation loss.

```python
import numpy as np
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.model_selection import train_test_split

rng = np.random.default_rng(0)
X = rng.uniform(-3, 3, (500, 1))
y = np.sin(X).ravel() + rng.normal(0, 0.2, 500)
X_tr, X_va, y_tr, y_va = train_test_split(X, y, random_state=0)

for lr in (0.3, 0.05):
    gb = GradientBoostingRegressor(
        n_estimators=1000, learning_rate=lr, max_depth=2, random_state=0
    ).fit(X_tr, y_tr)
    errors = [np.mean((p - y_va) ** 2) for p in gb.staged_predict(X_va)]
    best = int(np.argmin(errors)) + 1
    print(f"lr={lr} best trees={best} val MSE={min(errors):.4f}")
```

The mistake is setting the number of trees by default and never checking the validation loss, which can drift upward after the best point. The second mistake is using deep trees as base learners in boosting, which overfits quickly; shallow trees of depth 2 to 4 are a common starting point. The limitation is that gradient boosting is sensitive to hyperparameters and slower to train than a forest of the same size. In the market, XGBoost, LightGBM and CatBoost implement this procedure with extra regularization and speed, and they are used in credit, pricing and ranking systems.

### Voting

Imagine three friends each recommending a restaurant. If two say yes and one says no, you go. That is hard voting. If each friend gives a confidence score, and you average those scores, you are doing soft voting. Voting is the simplest way to combine different kinds of models.

From first principles, hard voting takes the most common predicted label among the base models. Soft voting averages their predicted probabilities and takes the class with the highest average, which usually works better when the models are reasonably calibrated. Voting helps most when the base models make different kinds of errors. Combining three versions of the same tree adds little, while combining a linear model, a tree and a nearest-neighbor model can help because each captures a different pattern.

```python
from sklearn.ensemble import VotingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.neighbors import KNeighborsClassifier
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import cross_val_score
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
vote = VotingClassifier(
    estimators=[
        ("lr", make_pipeline(StandardScaler(), LogisticRegression(max_iter=2000))),
        ("tree", DecisionTreeClassifier(max_depth=4, random_state=0)),
        ("knn", make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=15))),
    ],
    voting="soft",
)
print("soft voting cv accuracy", cross_val_score(vote, X, y, cv=5).mean().round(3))
```

The mistake is averaging probabilities from models that are poorly calibrated, such as a support vector machine without probability calibration, because the average then reflects the scale of each model rather than its confidence. The limitation of voting is that it does not learn how much to trust each model. In industry, a simple soft-voting ensemble is a cheap way to combine a team's existing models without a new training pipeline.

### Stacking concepts

Think of a manager who collects advice from several experts and learns, over time, which expert to trust on which kind of question. Stacking trains a second-level model that learns how to weight the base models' predictions, instead of averaging them.

From first principles, split the training data into folds. For each base model, produce out-of-fold predictions for every training row, using a model that did not see that row. These predictions form a new feature matrix, one column per base model. Train a simple meta-model, often a logistic regression, on that matrix. For new data, the base models produce their predictions, and the meta-model combines them. The out-of-fold step is essential: if the meta-model saw predictions made on the rows the base models trained on, it would learn to trust overfit predictions and would fail in production.

```python
from sklearn.ensemble import StackingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
stack = StackingClassifier(
    estimators=[
        ("tree", DecisionTreeClassifier(max_depth=4, random_state=0)),
        ("rf", RandomForestClassifier(n_estimators=200, random_state=0)),
    ],
    final_estimator=LogisticRegression(max_iter=2000),
    cv=5,
)
print("stacking cv accuracy", cross_val_score(stack, X, y, cv=5).mean().round(3))
```

The mistake is building the stack without the out-of-fold step, or reusing the same folds in the outer evaluation without care, which can leak information. Use the scikit-learn class, which handles the folds, and keep an outer test set for the final report. The limitation is complexity: a stack has many moving parts, is harder to debug and is harder to retrain, and its gain over a good single boosted model is often small. In the market, stacking shows up in competitions and in mature systems where a small percentage improvement is worth the maintenance cost.

---

## Hands-on exercises

1. Simulate bagging by hand for three bootstrap samples of the list `[1, 2, 3, 4, 5]`. Count how many distinct values each sample contains and compare with the 63 percent figure.
2. Run the bagging simulation from this lesson with 1, 10 and 50 trees. Plot the error against the number of trees and describe the shape of the curve.
3. Train a random forest on a dataset of your choice with `max_features` set to `sqrt` and to `None`. Compare the cross-validated accuracy and explain what changes when every split considers every feature.
4. Use the out-of-bag score to choose `n_estimators` from 50, 100, 200 and 400, then confirm the choice with cross-validation.
5. Run the gradient boosting learning-rate study on the sine data with learning rates 0.3, 0.1 and 0.01. Report the best number of trees and the validation error for each.
6. Compare hard and soft voting on the breast cancer dataset. State which is better and explain why in one sentence.
7. Build a stacking model with three base learners and a logistic regression meta-model. Compare its score with the best single base learner, and write one sentence on whether the extra complexity is justified.

---

## Project: Ensemble comparison on a tabular dataset

This is a medium-weight project. The code is short because scikit-learn does the heavy lifting. The value is in an honest comparison that uses one outer test set and a clear report.

**Repo name:** `100-days-ai-engineering-day-30-ensemble-learning`

**Goal:** compare a single decision tree, bagging, a random forest, gradient boosting, soft voting and stacking on the same tabular dataset, using cross-validation for model choice and a held-out test set for the final number.

**Repo layout:**

```
100-days-ai-engineering-day-30-ensemble-learning/
├── README.md
├── requirements.txt
├── models.py
├── compare.py
├── results/
│   ├── comparison.md
│   └── learning_curve.png
└── tests/
    └── test_models.py
```

**Starter code for `models.py`:**

```python
from sklearn.ensemble import (
    BaggingClassifier,
    GradientBoostingClassifier,
    RandomForestClassifier,
    StackingClassifier,
    VotingClassifier,
)
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.tree import DecisionTreeClassifier


def build_models(seed=0):
    """Return a dict mapping a model name to an unfitted estimator.
    Include: single tree, bagging of trees, random forest, gradient boosting,
    soft voting, and stacking. Use a fixed random_state everywhere.
    """
    raise NotImplementedError
```

**Starter code for `tests/test_models.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from models import build_models


def test_build_models_returns_six_named_estimators():
    models = build_models()
    assert len(models) == 6


def test_every_model_fits_and_predicts_binary_labels():
    rng = np.random.default_rng(0)
    X = rng.normal(size=(120, 4))
    y = (X[:, 0] + X[:, 1] > 0).astype(int)
    for name, model in build_models().items():
        pred = model.fit(X, y).predict(X)
        assert set(np.unique(pred)) <= {0, 1}, name
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib` and `pytest`.
2. Choose a tabular binary dataset with at least 1,000 rows, such as the breast cancer dataset from scikit-learn or the Adult Income dataset from UCI.
3. Implement `build_models`. Make the tests pass.
4. In `compare.py`, hold out a test set first. Use 5-fold cross-validation on the remaining data to score each model, and report the mean and standard deviation.
5. Choose the best model by cross-validation only. Fit it on the full training part and score the held-out test set once.
6. Plot a learning curve for the gradient boosting model with `sklearn.model_selection.learning_curve` and save it to `results/learning_curve.png`.
7. Write the README. Include the comparison table, the test score for the chosen model, and one paragraph on the accuracy-versus-interpretability trade-off.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python compare.py`
- [ ] The dataset source and license are named
- [ ] The comparison table reports cross-validation mean and standard deviation for all six models
- [ ] The README states that the test set was used once, for the model chosen by cross-validation
- [ ] The README names one reason a bank might still choose the single tree

**Done when:**

- `pytest` passes both tests
- The comparison table covers all six models with the same folds
- The test score is reported once, for the model chosen by cross-validation
- You can explain in one paragraph why bagging helps a deep tree but not a shallow one

**What to commit as evidence:** the source files, the tests, the comparison table, the learning curve, and the README with the single test score and the trade-off paragraph.

---

## Check your understanding

1. A bootstrap sample of 1,000 rows is drawn with replacement from 1,000 rows. About what fraction of the original rows appear at least once, and why does that matter for the out-of-bag estimate?
2. Explain why a random forest uses random feature selection on top of bagging, and describe what would happen to the trees if it did not.
3. Your boosted model's validation loss falls for 300 trees and then rises. Explain what is happening, and name two ways to choose the number of trees.
4. A colleague uses soft voting with a support vector machine whose scores were never calibrated. Explain why this can give misleading results.
5. Explain why stacking must use out-of-fold predictions to train the meta-model. What goes wrong if in-sample predictions are used?
6. You need a model that a regulator can read in full. Which ensemble method would you rule out, and what would you offer instead?
7. A single decision tree has 74 percent accuracy, and a random forest has 83 percent with a much larger training time. Describe the bias and variance changes that explain the gain, and state the cost you would report to the business.

### Answers

1. About 63 percent of the original rows appear at least once, since each draw has a probability of about 0.632 of being a unique row. The rows left out, about 37 percent, are the out-of-bag rows, which give a free validation estimate for each tree.
2. Plain bagging can produce trees that all split on the same strong feature, which makes their errors correlated and limits the variance reduction. Random feature selection decorrelates the trees so that their errors partly cancel when averaged.
3. The boosted model has started to fit noise in the training data, which is overfitting. Two ways to choose the number of trees are early stopping on a validation set, and cross-validation over a grid of tree counts.
4. Soft voting averages the probabilities, so the average depends on the scale of each model's scores. An uncalibrated support vector machine can produce scores on a very different scale from the other models, which dominates the average and makes the vote unfair.
5. In-sample predictions are too accurate, because the base models have already seen those rows. The meta-model would learn to trust them too much and would perform worse on new data. Out-of-fold predictions match the situation at deployment, where the base models have not seen the new rows.
6. I would rule out gradient boosting and random forests, since their combined trees are hard to read as a whole. I would offer a single shallow tree, or a logistic regression with clear coefficients, and explain the accuracy difference with the cross-validation table.
7. The forest reduces variance by averaging many decorrelated trees, which the single tree cannot do. The cost is training time, larger memory use, slower prediction, and a loss of the single readable rule. I would report that cost explicitly, along with a stability check and the accuracy gain.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Ensemble learning](https://en.wikipedia.org/wiki/Ensemble_learning)
- Wikipedia: [Bootstrap aggregating](https://en.wikipedia.org/wiki/Bootstrap_aggregating)
- Wikipedia: [Random forest](https://en.wikipedia.org/wiki/Random_forest)
- Wikipedia: [AdaBoost](https://en.wikipedia.org/wiki/AdaBoost)
- Wikipedia: [Gradient boosting](https://en.wikipedia.org/wiki/Gradient_boosting)
- Wikipedia: [Stacking (machine learning)](https://en.wikipedia.org/wiki/Stacking_(machine_learning))
- scikit-learn user guide, "Ensemble methods", at scikit-learn.org under the stable documentation, including the sections on bagging, forests, boosting, voting and stacking.

**Tutorials**

- scikit-learn documentation, the examples for `RandomForestClassifier`, `GradientBoostingClassifier` and `StackingClassifier`.
- XGBoost documentation, "Introduction to Boosted Trees", at xgboost.readthedocs.io, for the regularized version and its parameters.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Random Forests Part 1 - Building, Using and Evaluating" and "Gradient Boost Part 1 (of 4): Regression Main Ideas".
- StatQuest with Josh Starmer (YouTube), "AdaBoost, Clearly Explained" for the reweighting intuition.

**Practice**

- Kaggle Learn, the "Intermediate Machine Learning" course, which includes lessons on pipelines and gradient boosting.

---

## Tomorrow

Tomorrow you move to the margin. A support vector machine looks for the separating boundary that stays as far as possible from the closest examples on each side. You will meet hyperplanes, margins and support vectors, and you will see how the kernel trick lets a linear method separate data that is not linearly separable. The model is a useful contrast to the trees, because it depends on a few boundary points rather than on every row.

**Next:** [Day 31: Support Vector Machines](Day-31-Support-Vector-Machines.md)

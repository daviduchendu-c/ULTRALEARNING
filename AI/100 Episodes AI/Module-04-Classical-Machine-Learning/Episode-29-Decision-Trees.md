# Day 29: Decision Trees

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You should be comfortable with the train and test workflow from Day 24 and with probabilities from Day 28. The entropy calculations use logarithms, so keep Day 26 nearby.

**By the end of this day, you will have:**

- A hand-computed split, with Gini impurity, entropy and information gain for a small table
- A from-scratch decision stump and a recursive tree builder that you can read line by line
- A plot of training and validation error against tree depth, showing the overfitting curve
- A tuned scikit-learn `DecisionTreeClassifier` with a written explanation of each chosen setting

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the depth curve
- Graphviz (optional, free, from graphviz.org) to render the tree as an image
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 24: splits, overfitting and cross-validation
- Day 26: probabilities and log loss, since entropy uses the same logarithms
- Day 28: conditional probability, since impurity is computed from class proportions
- Recursion at the level of a function that calls itself on smaller inputs

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Splits](#splits)
- [Nodes and leaves](#nodes-and-leaves)
- [Impurity](#impurity)
- [Gini](#gini)
- [Entropy](#entropy)
- [Information gain](#information-gain)
- [Overfitting](#overfitting)
- [Tree depth](#tree-depth)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Decision tree from scratch with a depth study](#project-decision-tree-from-scratch-with-a-depth-study)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A decision tree is the closest thing machine learning has to a flowchart written from data. It asks a series of questions about the features, such as "is the income above 40,000", and each answer sends the row down a branch until it reaches a leaf that holds a prediction. People can read the result, check it against their own judgment and explain it to a regulator.

The tree also matters because it is the building block of the strongest methods for tabular data. Random forests and gradient boosted trees, covered on Day 30, combine many trees, and in many industry benchmarks on structured data they beat neural networks. A single tree is usually not the final model, but understanding how one is built, and how it fails, is necessary before you can tune an ensemble.

On the job, trees appear in credit approval rules, triage logic, medical screening aids and any setting where the team must show why a decision was made. The skill today is knowing how deep to let the tree grow, and recognizing the moment when a deeper tree is memorizing noise.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1963 | J. A. Morgan and J. N. Sonquist publish AID (Automatic Interaction Detection), an early recursive partitioning method for survey data. | The first widely used tree-growing procedure, built to find interactions in social data. |
| 1973 | Richard Breiman, Jerome Friedman and colleagues begin the CART (Classification and Regression Trees) work, published as a book in 1984. | CART introduced cost-complexity pruning and the Gini criterion, which many libraries still use. |
| 1986 | J. Ross Quinlan publishes ID3, which chooses splits by information gain based on entropy. | Entropy-based trees became the second major family, and later C4.5 extended them. |
| 1993 | Quinlan publishes C4.5, which handles continuous features and missing values. | C4.5 was a standard reference for tree learning for many years. |
| 2000s to today | Trees become the base learners of random forests and gradient boosting, which dominate many tabular tasks. | The single tree you learn today is the unit that those ensembles repeat. |

**Takeaway:** the tree idea is older than most of the methods that now depend on it, and its two main impurity measures, Gini and entropy, are still the defaults in libraries.

---

## Glossary

- **Split**: a rule of the form "feature j is less than or equal to threshold t" that sends rows left or right.
- **Node**: a point in the tree where a split happens.
- **Leaf**: a final node with no further split, which gives the prediction.
- **Root**: the top node that holds all the training rows.
- **Depth**: the number of splits on the path from the root to a leaf.
- **Impurity**: a measure of how mixed the classes are in a node. Pure nodes have impurity zero.
- **Gini impurity**: one minus the sum of squared class proportions.
- **Entropy**: the negative sum of each class proportion times its logarithm, measured in bits when the logarithm is base 2.
- **Information gain**: the reduction in impurity from a split, computed as the parent impurity minus the weighted average impurity of the children.
- **Pruning**: removing branches that do not improve generalization, to reduce variance.
- **Stump**: a tree with a single split.

---

### Splits

Think of sorting a box of mixed fruit into two bins. A good first question might be "is it round?", because it separates apples and oranges from bananas. A good split is a question that makes each bin more uniform than the box was.

From first principles, for one numeric feature, a tree considers thresholds placed between consecutive sorted values. For each threshold it divides the rows of the node into left and right groups, computes the impurity of each group, and scores the split by how much impurity it removes. For categorical features, the split tests membership in a set of categories. The tree picks the best feature and threshold, then repeats the process inside each child until a stopping rule applies, such as a maximum depth or a minimum number of rows in a node.

```python
import numpy as np

def candidate_thresholds(x):
    values = np.unique(x)
    return (values[:-1] + values[1:]) / 2.0   # midpoints between sorted unique values

print(candidate_thresholds(np.array([1.0, 3.0, 3.0, 7.0])))  # [2. 5.]
```

The mistake is treating the threshold as a meaningful number to report without checking its support. A split at 2.0 that separates only two rows is a fragile rule. The limitation of splits is that each one looks at a single feature, so diagonal boundaries need many splits to approximate. In the market, a loan officer's rule such as "income under 40,000 and debt above 15 percent" is a tree with two splits, and banks often start with a shallow tree to get a readable policy.

### Nodes and leaves

Imagine a flowchart taped to a wall. Each diamond is a question and each rectangle is an answer. The diamonds are nodes and the rectangles are leaves. Reading the path from the top to a rectangle gives the full reason for the decision.

From first principles, a tree is a recursive structure. A leaf stores a prediction: for classification, the majority class or the class proportions in that leaf, and for regression, the mean of the targets. An internal node stores a feature, a threshold and two children. Prediction for a new row is a loop that starts at the root and follows the branch that matches the row's values until it reaches a leaf.

```python
def predict_one(node, x):
    while "leaf" not in node:
        if x[node["feature"]] <= node["threshold"]:
            node = node["left"]
        else:
            node = node["right"]
    return node["leaf"]

tree = {
    "feature": 0, "threshold": 2.5,
    "left": {"leaf": 0},
    "right": {"feature": 1, "threshold": 0.5,
              "left": {"leaf": 1}, "right": {"leaf": 0}},
}
print(predict_one(tree, [3.0, 0.2]))  # 1
```

The mistake to avoid is assuming a leaf's prediction is certain. A leaf that holds three training rows, two of one class and one of another, predicts the majority but has a high uncertainty. The limitation of the tree structure is that its predictions are piecewise constant, so the output jumps at each threshold and is never smooth. In industry, the leaf sizes are often reported next to the predictions, because a rule supported by ten customers should be treated differently from one supported by ten thousand.

### Impurity

Consider two bags of marbles. One bag has only red marbles, which is a pure bag. The other has equal numbers of red and blue, which is maximally mixed. Impurity is a number that is zero for the pure bag and largest for the mixed one. A good split moves the marbles into bags that are purer than the original.

From first principles, if a node has class proportions `p_1` to `p_K`, then its impurity is a function of those proportions that is zero when one class has proportion 1 and is largest when the classes are equal. The two common choices are Gini and entropy, covered next. The quality of a split is the parent impurity minus the weighted average of the child impurities, where each child is weighted by its share of rows. The tree chooses the split that maximizes that quantity.

The mistake is to think impurity measures error. A node can be impure and still predict its majority class correctly most of the time, and a pure node can be wrong if its rows are mislabeled. The limitation is that impurity is local: a split that looks poor now may make a useful split possible later, which greedy tree building cannot anticipate. In industry, this greedy choice is one reason a single tree is less stable than an ensemble, and why teams compare trees with forests before deciding.

### Gini

Gini impurity answers a simple question: if you pick a row at random from a node and then guess its label by picking a label at random from the node's distribution, how often are you wrong? That is the Gini index.

From first principles, `G = 1 - sum_k p_k^2`. For a pure node, one `p_k` is 1, so `G = 0`. For a two-class node with proportions 0.5 and 0.5, `G = 1 - 0.25 - 0.25 = 0.5`, the maximum for two classes. Gini is cheap to compute and does not need logarithms, which is one reason libraries default to it.

```python
import numpy as np

def gini(labels):
    _, counts = np.unique(labels, return_counts=True)
    p = counts / counts.sum()
    return 1.0 - np.sum(p ** 2)

print(gini(np.array([0, 0, 0, 0])))  # 0.0, pure
print(gini(np.array([0, 0, 1, 1])))  # 0.5, maximally mixed for two classes
```

The mistake is comparing Gini values across different numbers of classes, because the maximum changes with the number of classes. The limitation is that Gini is less sensitive than entropy to small changes in rare classes, so it may choose different splits in highly imbalanced data. In practice, the two criteria often produce nearly identical trees, which is why the choice rarely changes results.

### Entropy

Entropy measures surprise. A coin that always lands heads is never surprising, so its entropy is zero. A fair coin is the most surprising, with one bit of entropy. For a node, entropy measures how many yes-or-no questions you would need on average to identify a row's class.

From first principles, `H = - sum_k p_k log2 p_k`, with the convention that `0 log 0 = 0`. For a pure node `H = 0`. For a two-class node with equal proportions, `H = 1` bit. Entropy is the criterion that ID3 and C4.5 used, and it connects directly to the information-theoretic cross-entropy from Day 26. The logarithm makes it slightly more expensive to compute, but on modern hardware the cost is negligible.

```python
import numpy as np

def entropy(labels):
    _, counts = np.unique(labels, return_counts=True)
    p = counts / counts.sum()
    return -np.sum(p * np.log2(p))

print(round(entropy(np.array([0, 0, 0, 1])), 3))  # 0.811 bits
```

The mistake is treating entropy as if it has a different meaning from Gini. Both measure mixing, and both are zero for pure nodes. The limitation is that entropy and Gini are heuristics. A split with a higher information gain does not necessarily produce a more accurate tree on new data, and the only reliable check is validation. In industry, the choice is a default setting to test.

### Information gain

Think of a detective with a suspect list. A good question cuts the list sharply. Information gain measures how much a question reduces the uncertainty about the answer, weighted by how many suspects fall on each side.

From first principles, for a parent node with entropy `H(parent)` and a split that sends `n_L` rows left and `n_R` right, information gain is `H(parent) - (n_L / n) H(left) - (n_R / n) H(right)`. A split that separates the classes cleanly gives a large gain, and a split that leaves both children as mixed as the parent gives zero. Working one example by hand is the best way to see it. Consider a node with 4 rows of class A and 4 of class B, which has entropy 1 bit. A split that sends 3 A and 1 B to the left, and 1 A and 3 B to the right, produces children with entropy 0.811 bits each. The gain is 1 minus 0.811, about 0.189 bits.

```python
import numpy as np

def entropy(labels):
    _, counts = np.unique(labels, return_counts=True)
    p = counts / counts.sum()
    return -np.sum(p * np.log2(p))

def info_gain(parent, left, right):
    n = len(parent)
    return entropy(parent) - (len(left) / n) * entropy(left) - (len(right) / n) * entropy(right)

parent = np.array(["A"] * 4 + ["B"] * 4)
left = np.array(["A"] * 3 + ["B"])
right = np.array(["A"] + ["B"] * 3)
print(round(info_gain(parent, left, right), 3))  # 0.189
```

The mistake is ignoring the weights. A split that produces one pure child and one large mixed child can have a small gain, even though one child is perfect. The limitation is that information gain favors features with many distinct values, because they can produce many small pure groups, which overfits. Scikit-learn avoids the worst cases by using thresholds and by letting you set minimum leaf sizes. In industry, a feature such as a customer ID should never be a split candidate, because it perfectly separates training rows and tells you nothing about new customers.

### Overfitting

A tree that keeps splitting until every leaf is pure has memorized the training data. It is a perfect answer key for last year's exam and an unhelpful guide for this year's. The deeper the tree, the closer it gets to the answer key, and the further it moves from the general pattern.

From first principles, a tree with enough leaves can isolate every training row, which gives zero training error. Its validation error falls at first, as the tree captures real structure, and then rises as it begins to fit noise. The gap between training and validation error is the measure of overfitting, the same idea from Day 24. Two strategies reduce it: stop growing the tree early with rules such as maximum depth and minimum leaf size, or grow a full tree and prune it back using a penalty on the number of leaves.

```python
import numpy as np
from sklearn.tree import DecisionTreeClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

X, y = make_classification(n_samples=800, n_features=10, flip_y=0.1, random_state=0)
X_tr, X_va, y_tr, y_va = train_test_split(X, y, test_size=0.3, random_state=0)
for depth in (1, 3, 6, None):
    m = DecisionTreeClassifier(max_depth=depth, random_state=0).fit(X_tr, y_tr)
    print(depth, round(m.score(X_tr, y_tr), 3), round(m.score(X_va, y_va), 3))
```

The mistake is judging a tree by training accuracy, which reaches 1.0 at unlimited depth on noisy data. The limitation of early stopping is that it can stop too soon, before a useful split that appears only after a weaker one. Pruning avoids this by growing first, but it costs more computation. In the market, credit and risk teams rarely deploy an unpruned tree, because the rules it produces are unstable across retraining.

### Tree depth

Imagine a flashlight in a dark room. A shallow tree lights a wide area coarsely. A deep tree lights small patches brightly and misses the layout of the room. Depth is the setting that controls how much detail the tree can represent.

From first principles, a tree of depth `d` can have at most `2^d` leaves, so depth controls capacity exponentially. Choosing depth is a bias and variance decision. Shallow trees have high bias and low variance, and deep trees have low bias and high variance. In practice, you pick the depth by cross-validation, and you may also tune `min_samples_leaf`, which often matters more than depth for noisy data because it prevents tiny leaves from making confident predictions.

```python
import numpy as np
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import cross_val_score
from sklearn.datasets import make_classification

X, y = make_classification(n_samples=800, n_features=10, flip_y=0.1, random_state=0)
for depth in range(1, 11):
    scores = cross_val_score(DecisionTreeClassifier(max_depth=depth, random_state=0), X, y, cv=5)
    print(depth, round(scores.mean(), 3))
```

The mistake is tuning depth alone and ignoring leaf size, or tuning on the test set. The limitation of a single tree is high variance: small changes in the training data can change the top split and everything below it. Ensembles, which you will meet on Day 30, reduce that variance. In industry, a single tree is often used when the rules must be shown to a regulator, and an ensemble is used when accuracy matters more than a single readable policy.

---

## Hands-on exercises

1. Compute the Gini impurity and the entropy by hand for a node with class counts 6 and 2, and check your results with the functions in this lesson.
2. For a parent with 5 rows of class A and 5 of class B, consider the split that sends 4 A and 1 B left, and 1 A and 4 B right. Compute the information gain by hand, then in code.
3. Repeat exercise 2 with a split that sends 5 A to the left and 5 B to the right. Explain why its gain is larger, and what the result implies for perfect separation.
4. Implement a decision stump that searches all thresholds on one feature and returns the threshold with the best information gain. Test it on a dataset of your choice.
5. Build a tree of depth 3 on the breast cancer dataset, draw it with `sklearn.tree.plot_tree`, and describe the rule at each leaf in plain words.
6. Plot training and validation accuracy against depth from 1 to 15. Mark the depth that gives the best validation score and explain why it is not the depth with the best training score.
7. Compare the `criterion="gini"` and `criterion="entropy"` versions with cross-validation. Report whether the best depth changes.

---

## Project: Decision tree from scratch with a depth study

This is a medium-weight project. The tree builder is short but needs care, and the depth study is the part that teaches the trade-off.

**Repo name:** `100-days-ai-engineering-day-29-decision-trees`

**Goal:** implement a binary decision tree classifier with Gini impurity, check it against scikit-learn on a small dataset, and measure how depth changes the bias and variance on a real dataset.

**Repo layout:**

```
100-days-ai-engineering-day-29-decision-trees/
├── README.md
├── requirements.txt
├── tree.py
├── depth_study.py
├── results/
│   └── depth_curve.png
└── tests/
    └── test_tree.py
```

**Starter code for `tree.py`:**

```python
import numpy as np


def gini(labels):
    """Gini impurity of an array of labels."""
    raise NotImplementedError


def best_split(X, y):
    """Return (feature, threshold, gain) for the split with the largest Gini gain.
    Return (None, None, 0.0) if no split improves impurity.
    """
    raise NotImplementedError


def build_tree(X, y, max_depth, depth=0):
    """Return a dict. Leaves hold {'leaf': majority_label}. Internal nodes hold
    'feature', 'threshold', 'left' and 'right'.
    """
    raise NotImplementedError


def predict_one(node, x):
    """Follow the tree from the root to a leaf and return its label."""
    raise NotImplementedError


def predict(root, X):
    return np.array([predict_one(root, row) for row in X])
```

**Starter code for `tests/test_tree.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from tree import gini, best_split, build_tree, predict


def test_gini_pure_and_mixed():
    assert gini(np.array([1, 1, 1])) == 0.0
    assert np.isclose(gini(np.array([0, 0, 1, 1])), 0.5)


def test_best_split_finds_the_obvious_threshold():
    X = np.array([[1.0], [2.0], [3.0], [10.0], [11.0], [12.0]])
    y = np.array([0, 0, 0, 1, 1, 1])
    feature, threshold, gain = best_split(X, y)
    assert feature == 0
    assert 3.0 < threshold < 10.0
    assert np.isclose(gain, 0.5)


def test_depth_zero_predicts_majority():
    X = np.array([[1.0], [2.0], [3.0]])
    y = np.array([1, 1, 0])
    root = build_tree(X, y, max_depth=0)
    assert (predict(root, X) == 1).all()


def test_deep_tree_memorizes_clean_data():
    X = np.array([[0.0, 0.0], [0.0, 1.0], [1.0, 0.0], [1.0, 1.0]])
    y = np.array([0, 1, 1, 0])
    root = build_tree(X, y, max_depth=3)
    assert (predict(root, X) == y).all()
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib` and `pytest`.
2. Implement `gini` and `best_split`. Test the first two tests, then iterate until they pass. In `best_split`, consider midpoints between sorted unique values, and skip splits that leave one side empty.
3. Implement `build_tree` recursively, stopping at `max_depth` or when a node is pure. Implement `predict_one` and `predict`. Make the remaining tests pass.
4. In `depth_study.py`, load a real dataset such as the breast cancer data, split it into train and validation parts, and compare your tree with `DecisionTreeClassifier(criterion="gini")` at depths from 1 to 10.
5. Save a plot of training and validation accuracy against depth to `results/depth_curve.png`.
6. Report the depth that gives the best validation accuracy, and report the training accuracy at that depth. Explain the gap.
7. Write the README. Include the depth table, the plot, and one paragraph on why a single tree is unstable.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python depth_study.py`
- [ ] The depth table with training and validation accuracy is included
- [ ] The depth curve is embedded with a caption that names the overfitting region
- [ ] The README names one reason a bank or clinic might prefer a shallow tree even when a deep one scores higher

**Done when:**

- `pytest` passes all four tests
- Your tree matches scikit-learn's validation accuracy within two percentage points at the same depth
- The depth curve clearly shows the point where validation accuracy stops improving
- You can explain in one paragraph why information gain can prefer features with many values, and how to avoid that trap

**What to commit as evidence:** the source files, the tests, the depth table and plot, and the README with the explanation of the trade-off.

---

## Check your understanding

1. A node has 8 rows of class A and 2 of class B. Compute its Gini impurity and entropy. Which one is larger, and does the ordering matter for choosing a split?
2. A split leaves both children with the same class proportions as the parent. What is its information gain, and why would a greedy tree never choose it?
3. Your tree has 100 percent training accuracy and 70 percent validation accuracy. Name the problem, and list two changes that reduce it.
4. Explain why information gain can favor a customer ID feature, and what check would prevent the tree from using it.
5. A single tree changes its top split when you remove 5 percent of the training rows. Explain why this happens, and name a method that reduces this instability.
6. Explain why a tree's predictions are piecewise constant, and describe a case where this is an advantage and a case where it is a disadvantage.
7. A depth-10 tree and a depth-3 tree have similar validation accuracy. Which would you ship in a regulated setting, and what evidence would you add to support the choice?

### Answers

1. Gini is `1 - (0.8^2 + 0.2^2) = 0.32`. Entropy is `-(0.8 log2 0.8 + 0.2 log2 0.2)`, about 0.722 bits. Entropy is larger on this scale, but the ordering matters only in comparison, since both criteria rank splits similarly in most cases.
2. The gain is zero, because the weighted average of the child entropies or Gini values equals the parent's. A greedy tree chooses the split with the largest gain, so it would only choose this split if every other option also had zero gain.
3. The tree has memorized the training data, which is overfitting. Two changes that reduce it are limiting the maximum depth and raising the minimum number of rows per leaf. Pruning or using an ensemble also helps.
4. A customer ID can split the training rows into many pure groups, which gives a large gain even though the feature does not generalize. The check is to remove identifiers from the features and to test whether validation accuracy holds up.
5. The top split is chosen greedily, and small changes in the data can change which feature or threshold wins, which then changes every branch below it. Random forests reduce this by averaging many trees trained on resampled data.
6. A leaf predicts one value across a whole region, so the output jumps at each threshold. The advantage is that the rule is easy to read and audit. The disadvantage is that small changes in an input near a threshold can change the prediction sharply, and the model cannot represent smooth trends without many splits.
7. I would ship the shallower tree, if its accuracy is comparable, because it is easier to audit and less likely to memorize noise. The evidence to add is a stability check across resampled training sets, a calibration check for the leaf proportions, and a review of the rules with domain experts.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Decision tree learning](https://en.wikipedia.org/wiki/Decision_tree_learning)
- Wikipedia: [Decision tree](https://en.wikipedia.org/wiki/Decision_tree)
- Wikipedia: [Entropy (information theory)](https://en.wikipedia.org/wiki/Entropy_(information_theory))
- Wikipedia: [Information gain in decision trees](https://en.wikipedia.org/wiki/Information_gain_in_decision_trees)
- Breiman, Friedman, Olshen and Stone, *Classification and Regression Trees* (1984), the original CART book. Search the title to find a library or publisher copy.
- scikit-learn user guide, "Decision Trees", at scikit-learn.org under the stable documentation, including the section on complexity and pruning.

**Tutorials**

- scikit-learn documentation, `DecisionTreeClassifier`, with the parameters `max_depth`, `min_samples_leaf` and `criterion`.
- GeeksforGeeks, search "decision tree classifier" for worked examples with the Gini and entropy steps.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Decision and Classification Trees, Clearly Explained!!!" and "Regression Trees, Clearly Explained!!!".
- StatQuest with Josh Starmer (YouTube), "Gini Impurity, Clearly Explained!!!" for a step-by-step calculation.

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course, which includes a decision tree notebook on overfitting and underfitting.

---

## Tomorrow

Tomorrow you combine trees. A random forest trains many trees on resampled data and averages them, which reduces the variance you saw today. Gradient boosting trains trees one after another, each correcting the errors of the last. You will meet bagging, boosting, voting and the idea of stacking, and you will see why the ensembles often beat a single tree by a wide margin on structured data.

**Next:** [Day 30: Ensemble Learning](Day-30-Ensemble-Learning.md)

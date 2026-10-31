# Day 27: k-Nearest Neighbors

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3 hours
**Level:** beginner to intermediate. You need Day 8 (vectors and norms) and Day 24 (splits and cross-validation). The model is easy to run, so the time goes to the choices around it.

**By the end of this day, you will have:**

- A working understanding of distance metrics and how the choice of metric changes which points are neighbors
- A from-scratch k-nearest neighbors classifier and regressor, checked against scikit-learn
- A plot of cross-validated accuracy against k, and a reasoned choice of k
- A demonstration that feature scaling changes the predictions, and a rule for when to scale

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the accuracy-versus-k plot
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 8: vectors, the dot product and the Euclidean norm
- Day 24: train, validation and test splits, and cross-validation
- Day 25 or Day 26: the idea of a model that maps features to a prediction
- Comfort with NumPy broadcasting, since the distance computation depends on it

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Distance metrics](#distance-metrics)
- [Neighborhoods](#neighborhoods)
- [Choosing k](#choosing-k)
- [Scaling sensitivity](#scaling-sensitivity)
- [Classification and regression](#classification-and-regression)
- [Hands-on exercises](#hands-on-exercises)
- [Project: k-nearest neighbors from scratch with scaling study](#project-k-nearest-neighbors-from-scratch-with-scaling-study)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Ask a friend for a restaurant recommendation in a city you do not know. They skip building a theory of your taste. They think of people similar to you and recommend what those people liked. That is the k-nearest neighbors method in everyday form, and it is one of the simplest models that still works well on many real problems.

The method matters for three reasons. It needs no training step, so it is a useful baseline and a quick check on whether a problem is learnable at all. It makes the role of distance explicit, which helps you understand every model that depends on similarity, including the search systems behind embeddings and retrieval-augmented generation. And it fails in instructive ways, especially when features have different units, which is one of the most common bugs in production data science.

On the job, nearest-neighbor search is used for recommendations, duplicate detection, anomaly scoring and as the retrieval step in systems that answer questions from documents. The classifier you build today is the simplest form of that idea. The approximate nearest-neighbor indexes used at scale, such as those in FAISS or HNSW-based libraries, are faster versions of the same question.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1951 | Evelyn Fix and Joseph Hodges write a technical report on discriminatory analysis, proposing the nearest neighbor rule as a nonparametric method. | The idea is one of the earliest nonparametric classifiers, and it predates most of modern statistics for machine learning. |
| 1967 | Thomas Cover and Peter Hart publish "Nearest neighbor pattern classification" with a proof that the one-nearest-neighbor error is at most twice the Bayes error in the limit. | The result gives a theoretical reason to trust the simple rule with enough data. |
| 1970s | Fukunaga and Narendra and others develop branch-and-bound search and tree structures such as the k-d tree to speed up neighbor queries. | The fast lookups used in libraries today descend from these data structures. |
| 2010s | Approximate nearest-neighbor methods such as locality-sensitive hashing and graph-based indexes handle millions of vectors. | Vector databases and embedding search rely on approximate versions of this method. |
| 2020s | Retrieval systems use nearest-neighbor search over embeddings to supply context to language models. | The distance you choose today decides which documents a retrieval system returns. |

**Takeaway:** the method is old and simple, and its main difficulty is not the algorithm but the notion of similarity that it depends on.

---

## Glossary

- **Instance-based learning**: a method that stores training examples and makes predictions by comparing new inputs with them, instead of learning a fixed formula.
- **Lazy learner**: a model with no training step, where the work happens at prediction time.
- **Distance metric**: a rule that gives the separation between two points. It must be nonnegative, zero only for identical points, symmetric and satisfy the triangle inequality.
- **Euclidean distance**: the straight-line distance, the square root of the sum of squared differences.
- **Manhattan distance**: the sum of absolute differences, also called the city-block or L1 distance.
- **Minkowski distance**: a family of distances that includes Euclidean (p = 2) and Manhattan (p = 1).
- **Cosine similarity**: the cosine of the angle between two vectors, which compares direction rather than length.
- **Neighborhood**: the set of k training points closest to a query.
- **Majority vote**: a classification rule that picks the most common label among the neighbors.
- **Feature scaling**: transforming features so they have comparable ranges, such as standardization to zero mean and unit variance.
- **Curse of dimensionality**: the effect in which distances become less informative as the number of features grows, covered in detail on Day 33.

---

### Distance metrics

Think of measuring how far apart two houses are. You could walk the streets of a grid city, which gives Manhattan distance, or fly in a straight line, which gives Euclidean distance. Both are valid, and they give different answers. Choosing a metric is choosing what "close" means.

From first principles, for two vectors `x` and `z` with `d` features, the Euclidean distance is `sqrt(sum_j (x_j - z_j)^2)`, the Manhattan distance is `sum_j |x_j - z_j|`, and the cosine distance is one minus the cosine similarity. Euclidean distance rewards points that are close on every feature. Manhattan distance is less sensitive to a single large difference. Cosine distance ignores vector length, which is useful for text, where a long document and a short one about the same topic should be near each other.

```python
import numpy as np

x = np.array([1.0, 0.0])
z = np.array([3.0, 4.0])
euclid = np.sqrt(np.sum((x - z) ** 2))
manhattan = np.sum(np.abs(x - z))
cos_sim = np.dot(x, z) / (np.linalg.norm(x) * np.linalg.norm(z))
print(euclid, manhattan)     # 4.47 6.0
print(round(cos_sim, 3))     # 0.6, the angle matters, not the length of z
print(np.dot(x, 10 * z) / (np.linalg.norm(x) * np.linalg.norm(10 * z)))  # still 0.6
```

The mistake to avoid is picking a metric without thinking about the features. Euclidean distance assumes the features are measured on comparable scales and that each contributes equally. Cosine distance fails on data where length carries information, such as the total spend of a customer. The limitation of every distance is that it is a fixed rule, so it cannot learn which features matter. In industry, embedding search typically uses cosine similarity or the dot product on normalized vectors, because the direction of an embedding carries most of its meaning.

### Neighborhoods

Imagine dropping a pebble in a pond and watching the ripples spread. The neighborhood of a query point is the set of training points inside the smallest ripple that contains k of them. The prediction is decided by who lives in that ripple.

From first principles, to predict for a query `q`, compute the distance from `q` to every training point, sort the distances, and take the k smallest. For classification, the predicted label is the majority label among those k points. For a tie, scikit-learn breaks it by using the nearest of the tied neighbors, and you should know that a tie is possible when k is even. The cost of one prediction is proportional to the number of training points times the number of features, which is why the method slows down on large datasets.

```python
import numpy as np

def knn_predict(X_train, y_train, q, k=3):
    dists = np.sqrt(((X_train - q) ** 2).sum(axis=1))
    nearest = np.argsort(dists)[:k]
    labels, counts = np.unique(y_train[nearest], return_counts=True)
    return labels[np.argmax(counts)]

X_train = np.array([[0.0, 0.0], [0.1, 0.2], [1.0, 1.0], [0.9, 1.1]])
y_train = np.array([0, 0, 1, 1])
print(knn_predict(X_train, y_train, np.array([0.8, 0.9]), k=3))  # 1
```

The mistake is ignoring the cost of prediction. A million training rows with fifty features means fifty million multiplications per query, and a web service with many queries per second needs an index. The limitation is that the neighborhood is only as good as the metric and the scale of the features. In the market, a retailer that recommends products from the purchases of similar customers computes neighborhoods over a sparse matrix, often with an approximate index, because exact search is too slow.

### Choosing k

A small k is like asking the one closest friend. The answer is detailed but noisy, because one friend may have an unusual taste. A large k is like polling the whole street. The answer is stable but may ignore local differences that matter. The best k is somewhere in between, and you find it by measurement.

From first principles, k controls the bias and variance trade-off from Day 24. At `k = 1`, the training error is zero, because each training point is its own nearest neighbor, which is a clear sign of overfitting. As k grows, the decision boundary becomes smoother and the variance falls, but the model may start to underfit when k approaches the size of a class. Choose k on a validation set or with cross-validation, and prefer an odd value for binary problems to avoid ties.

```python
import numpy as np
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
for k in (1, 5, 15, 31):
    model = make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=k))
    scores = cross_val_score(model, X, y, cv=5)
    print(k, round(scores.mean(), 3))
```

The mistake is choosing k by looking at training accuracy, which is always best at k = 1. Another is picking k from a single validation split when the data is small, which gives an unstable answer. The limitation is that a single global k ignores regions of different density, where a good k may differ. Adaptive methods exist, but they are less common in practice. In industry, a team that tunes k for a recommendation system usually does so on held-out interactions, not on the rows used to build the index.

### Scaling sensitivity

Suppose you measure people by height in centimeters and by income in dollars. Income differences of thousands dwarf height differences of tens of centimeters. A distance computed on the raw values is almost entirely about income, and the height information is ignored. The effect is large, and it can reverse the predictions.

From first principles, Euclidean distance adds squared differences across features, so a feature with a large numeric range dominates the sum. Standardizing each feature to zero mean and unit variance puts the features on a comparable scale, so each contributes according to its pattern rather than its units. Scaling must be fitted on the training part only and applied to the other parts, as you practiced on Day 24, because the mean and standard deviation are statistics of the data.

```python
import numpy as np
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler

rng = np.random.default_rng(0)
n = 300
height_cm = rng.normal(170, 10, n)
income_usd = rng.normal(50000, 15000, n)
X = np.column_stack([height_cm, income_usd])
y = (height_cm > 170).astype(int)   # the label depends only on height

raw = KNeighborsClassifier(n_neighbors=5).fit(X[:200], y[:200]).score(X[200:], y[200:])
scaled = KNeighborsClassifier(n_neighbors=5).fit(
    StandardScaler().fit(X[:200]).transform(X[:200]), y[:200]
).score(StandardScaler().fit(X[:200]).transform(X[200:]), y[200:])
print("raw accuracy", round(raw, 3), " scaled accuracy", round(scaled, 3))
```

The raw model performs close to chance, while the scaled model recovers the height signal. The mistake is forgetting to scale, or scaling with a statistic computed on the full dataset, which leaks information. Use a pipeline to avoid both. The limitation is that standardization assumes a rough symmetry in each feature. Heavily skewed features may need a log transform first, and features with a meaningful absolute scale, such as pixel intensities, may need a different treatment. In industry, a churn model built on mixed units such as tenure in months and revenue in currency will give misleading neighborhoods without scaling.

### Classification and regression

Classification and regression share the same neighborhood. The only difference is what the neighbors contribute. For classification, the neighbors vote on a label. For regression, they average a number. Both are simple, and both inherit the same weaknesses.

From first principles, the regression prediction for a query is the mean of the targets of its k nearest neighbors. A weighted version gives closer neighbors more influence, for example with weights equal to the inverse of the distance. The classifier can also return probabilities by taking the fraction of neighbors in each class, which is a crude but often useful estimate. Predictions are piecewise constant, because they only change when the neighborhood changes, so the output can look jagged.

```python
import numpy as np
from sklearn.neighbors import KNeighborsRegressor

rng = np.random.default_rng(0)
X = np.sort(rng.uniform(0, 10, 100)).reshape(-1, 1)
y = np.sin(X).ravel() + rng.normal(0, 0.2, 100)

reg = KNeighborsRegressor(n_neighbors=5, weights="distance").fit(X, y)
print(reg.predict(np.array([[2.0], [5.0], [8.0]])).round(2))
```

The mistake is treating the regressor as if it extrapolates. A neighbor method can only return values seen in the training data, so a query far outside the range of the data gets the boundary value, which is usually wrong. The limitation is that the method has no notion of a trend, so a linear relationship needs many more points to be learned than a linear model would require. In practice, kNN regression is used for smooth, dense problems, such as imputing a missing sensor reading from nearby sensors, and it is rarely the final model for forecasting.

---

## Hands-on exercises

1. For the points `(0, 0)`, `(3, 4)` and `(6, 8)`, compute the Euclidean, Manhattan and cosine distances between each pair by hand, then verify with NumPy.
2. Implement `knn_predict` from the section above and test it on a dataset of two Gaussian blobs. Plot the decision regions for `k = 1` and `k = 15`.
3. Show that at `k = 1` the training accuracy is 1.0 for any dataset with distinct points. Explain why in one sentence.
4. Run the scaling experiment from this lesson. Repeat it with `StandardScaler` replaced by `MinMaxScaler`, and report whether the accuracy changes.
5. Plot cross-validated accuracy against k for values from 1 to 51 on the breast cancer dataset. Identify the k where the curve flattens, and explain why very large k is not helpful here.
6. Compare your from-scratch classifier with `KNeighborsClassifier` on the same split. Report the maximum disagreement in predicted labels and explain any differences from tie-breaking.
7. Fit a kNN regressor with uniform weights and with distance weights on the sine data. Compare the two curves in a plot and describe where they differ.

---

## Project: k-nearest neighbors from scratch with scaling study

This is a medium-weight project. The algorithm is short, so the deliverable is a careful study of how scaling, metric and k change predictions, with tests that guard the implementation.

**Repo name:** `100-days-ai-engineering-day-27-k-nearest-neighbors`

**Goal:** implement a kNN classifier from scratch, check it against scikit-learn, and report how scaling and the choice of k change cross-validated accuracy on a real dataset.

**Repo layout:**

```
100-days-ai-engineering-day-27-k-nearest-neighbors/
├── README.md
├── requirements.txt
├── knn.py
├── study.py
├── results/
│   └── accuracy_vs_k.png
└── tests/
    └── test_knn.py
```

**Starter code for `knn.py`:**

```python
import numpy as np


def pairwise_euclidean(A, B):
    """Return the matrix of Euclidean distances between rows of A (n by d) and rows of B (m by d)."""
    raise NotImplementedError


class KNNClassifier:
    def __init__(self, k=5):
        self.k = k

    def fit(self, X, y):
        """Store the training data."""
        raise NotImplementedError

    def predict(self, X):
        """Majority vote among the k nearest stored rows, with ties broken by the nearest neighbor."""
        raise NotImplementedError
```

**Starter code for `tests/test_knn.py`:**

```python
import sys
from pathlib import Path

import numpy as np
from sklearn.neighbors import KNeighborsClassifier

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from knn import KNNClassifier, pairwise_euclidean


def test_pairwise_distance_matches_manual():
    A = np.array([[0.0, 0.0]])
    B = np.array([[3.0, 4.0], [0.0, 1.0]])
    assert np.allclose(pairwise_euclidean(A, B), [[5.0, 1.0]])


def test_k1_memorizes_training_data():
    rng = np.random.default_rng(0)
    X = rng.normal(size=(50, 3))
    y = rng.integers(0, 2, 50)
    model = KNNClassifier(k=1).fit(X, y)
    assert (model.predict(X) == y).all()


def test_matches_scikit_learn_on_blobs():
    rng = np.random.default_rng(1)
    X = np.vstack([rng.normal(0, 1, (80, 2)), rng.normal(3, 1, (80, 2))])
    y = np.array([0] * 80 + [1] * 80)
    Xq = rng.normal(1.5, 2, (40, 2))
    mine = KNNClassifier(k=5).fit(X, y).predict(Xq)
    ref = KNeighborsClassifier(n_neighbors=5).fit(X, y).predict(Xq)
    assert (mine == ref).mean() > 0.95
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib`, `pandas` and `pytest`.
2. Implement `pairwise_euclidean` with broadcasting, without a Python loop over rows. Make the first test pass.
3. Implement `KNNClassifier.fit` and `predict`. Make the remaining tests pass.
4. In `study.py`, load the breast cancer dataset, and run 5-fold stratified cross-validation for k from 1 to 51 with and without standardization. Use a pipeline for the scaled version.
5. Save a plot of accuracy against k with both curves to `results/accuracy_vs_k.png`.
6. Repeat the study with Manhattan distance in scikit-learn and note whether the best k changes.
7. Write the README. State the best k for each condition, the standard deviation across folds, and a rule for when to scale features.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python study.py`
- [ ] The best k and the accuracy for each condition are reported in a table
- [ ] The accuracy-versus-k plot is embedded with a caption
- [ ] The README names one case where kNN would be a poor choice, such as a large dataset that needs fast predictions

**Done when:**

- `pytest` passes all three tests
- Your classifier agrees with scikit-learn on at least 95 percent of the query points in the test
- The study shows a clear difference between the scaled and unscaled curves, and the README explains why
- You can explain in one paragraph why k = 1 gives perfect training accuracy and why that is a warning sign

**What to commit as evidence:** the source files, the tests, the study output in the README, the plot, and the written rule for scaling.

---

## Check your understanding

1. Two points differ by 1 in one feature measured in meters and by 1000 in another measured in millimeters. Explain which feature dominates the Euclidean distance and what to do about it.
2. Your kNN classifier has 100 percent training accuracy at k = 1 and 78 percent validation accuracy. Diagnose the situation and propose a change that is measurable.
3. Why is cosine similarity a natural choice for comparing text embeddings, and when would it be a poor choice for customer spend data?
4. A kNN regressor is asked to predict a value for an input far outside the training range. Predict what it returns, and explain why a linear model would behave differently.
5. Explain why the prediction cost of kNN grows with the size of the training set, and name one data structure or approximate method that reduces this cost.
6. You fit a scaler on the full dataset and then run cross-validation. Describe the leak, and state how a pipeline prevents it.
7. The curse of dimensionality makes distances less informative as the number of features grows. Explain in one paragraph why this hurts kNN more than a linear model, and what you would try first if you had 500 features.

### Answers

1. The millimeter feature dominates, because a difference of 1000 in it outweighs a difference of 1 in the meter feature. The fix is to standardize each feature so that the distances reflect pattern rather than units.
2. The model has memorized the training data and has high variance. Choose k by cross-validation over a range of values, and measure the validation accuracy for each. A larger k that improves the validation score is the measurable change.
3. Cosine similarity compares direction, so a long document and a short one on the same topic are close, which suits text embeddings where length is not the signal. For spend data, length is the signal, because a larger spend is a real difference, so cosine similarity would hide it.
4. The regressor returns a value close to the nearest training values, which is usually the boundary value of the training range. A linear model would extend the fitted line, which may or may not be right, but it is a principled guess rather than a copy of the boundary.
5. Each prediction computes the distance to every stored point, so the work grows with the number of rows. A k-d tree, a ball tree, or an approximate index such as HNSW reduces the number of distances computed per query.
6. The scaler learns the mean and variance from the rows that will later be used for testing, so the test statistics influence the training features. A pipeline refits the scaler inside each fold, so the held-out rows never contribute to the fitted statistics.
7. In high dimensions, most points become roughly equally far from each other, so the nearest and the farthest neighbors are not very different in distance. A linear model is less affected because it sums weighted features instead of relying on distances. With 500 features, I would first try dimensionality reduction such as PCA from Day 33, or a model that does not rely on distance, and compare them with kNN on validation data.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [K-nearest neighbors algorithm](https://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm)
- Wikipedia: [Nearest neighbor search](https://en.wikipedia.org/wiki/Nearest_neighbor_search)
- Wikipedia: [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance)
- Wikipedia: [Cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity)
- Wikipedia: [Curse of dimensionality](https://en.wikipedia.org/wiki/Curse_of_dimensionality)
- scikit-learn user guide, "Nearest Neighbors", at scikit-learn.org under the stable documentation.

**Tutorials**

- scikit-learn documentation, `KNeighborsClassifier` and `KNeighborsRegressor`, including the examples on choosing weights and the metric.
- GeeksforGeeks, search "K-Nearest Neighbours" for a worked example with a small table of distances.

**Watch**

- StatQuest with Josh Starmer (YouTube), "StatQuest: K-nearest neighbors, clearly explained".
- Stanford CS231n lecture notes on the k-nearest neighbor classifier, at cs231n.github.io. These notes include the distance formulas and the caveats on high dimensions.

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course, and a notebook that compares kNN with a linear model on the same dataset.

---

## Tomorrow

Tomorrow you bring in probability directly. Naive Bayes classifies a row by combining the evidence from each feature with Bayes' theorem, using an independence assumption that is almost never true and works surprisingly well anyway. You will compute posterior probabilities by hand, see why the assumption is wrong, and apply the model to text classification, where it is still a strong baseline.

**Next:** [Day 28: Naive Bayes](Day-28-Naive-Bayes.md)

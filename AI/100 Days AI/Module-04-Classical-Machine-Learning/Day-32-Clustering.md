# Day 32: Clustering

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 27 (distances and neighborhoods) and Day 31 (centroids and the geometry of boundaries). Clustering has no answer key, so you will judge results with reasoning and metrics rather than with accuracy.

**By the end of this day, you will have:**

- A from-scratch k-means with a clear centroid update, run on two datasets
- A comparison of initialization methods, with the inertia and the stability of each run
- A hierarchical clustering dendrogram and a DBSCAN run, with a note on when each fits the data
- A silhouette-based report that chooses the number of clusters and says honestly how much to trust it

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- SciPy 1.13 or newer (`pip install scipy`), BSD 3-Clause, for the hierarchical clustering and dendrogram
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the plots
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Day 27: distance metrics and why scaling matters
- Day 24: the habit of comparing methods and not trusting a single run
- Day 31: the idea of a centroid as a summary of a region
- Day 25: the mean as the value that minimizes squared error

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Unsupervised learning](#unsupervised-learning)
- [k-means](#k-means)
- [Centroids](#centroids)
- [Distance](#distance)
- [Initialization](#initialization)
- [Hierarchical clustering concept](#hierarchical-clustering-concept)
- [DBSCAN concept](#dbscan-concept)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Clustering a customer dataset with validation](#project-clustering-a-customer-dataset-with-validation)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every model so far needed a label. Clustering works without one. It takes a table of rows and proposes groups of rows that look alike, such as customers who buy similar products or sensor readings that come from the same operating mode. The groups are a hypothesis about the data, not a fact, and the work is in testing that hypothesis.

Clustering is valuable because most real data is unlabeled. A company has millions of transactions and almost none flagged as a particular kind of fraud. A research team has measurements but no categories. Clustering gives a first map. It can reveal segments for marketing, group documents for review, or find an unusual cluster that deserves investigation.

On the job, the common failure is to treat the output of a clustering algorithm as if it were ground truth. Clusters depend on the features, the scale, the distance and the number of groups you chose, and a different choice gives different clusters. The skill today is to make those choices visible and to check the result against something outside the algorithm, such as a business outcome or a domain expert's judgment.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1957 | Stuart Lloyd works out an iterative procedure for quantization at Bell Labs. The report was not published until 1982, and the loop is now called Lloyd's algorithm. | The core loop of k-means, assign and update, is a long-standing idea. |
| 1963 | Joe H. Ward Jr. publishes a hierarchical grouping method that minimizes the increase in within-group variance. | Ward's method is the default linkage in many libraries for hierarchical clustering. |
| 1965 | Edward Forgy publishes a similar iterative method for clustering. | The procedure entered the literature through several independent sources. |
| 1967 | James MacQueen names the method "k-means" in a paper on classification and analysis of multivariate observations. | The name that practitioners use today comes from this paper. |
| 1996 | Martin Ester and colleagues publish DBSCAN, a density-based method that finds clusters of arbitrary shape and marks noise points. | Density-based clustering is the standard answer when clusters are not round blobs. |

**Takeaway:** the core methods are decades old, and their assumptions about shape, scale and density matter more than their names.

---

## Glossary

- **Unsupervised learning**: learning structure from inputs alone, without target labels.
- **Cluster**: a group of rows that are closer to each other, under a chosen distance, than to rows in other groups.
- **Centroid**: the mean of the points in a cluster, which is the center that k-means uses.
- **Inertia (within-cluster sum of squares)**: the sum of squared distances from each point to its assigned centroid.
- **Initialization**: the starting positions of the centroids before the iterations begin.
- **k-means++**: an initialization method that chooses starting centroids far from one another, with probability proportional to the squared distance.
- **Linkage**: the rule for the distance between two clusters in hierarchical clustering, such as single, complete, average or Ward.
- **Dendrogram**: a tree diagram that shows the order and height at which clusters merge.
- **Core point**: in DBSCAN, a point with at least `min_samples` neighbors within radius `eps`.
- **Noise point**: in DBSCAN, a point that is not reachable from any core point, which is labeled -1 in scikit-learn.
- **Silhouette score**: a value between -1 and 1 that compares each point's distance to its own cluster with its distance to the nearest other cluster.

---

### Unsupervised learning

Imagine sorting a box of unlabeled buttons. You would start by grouping them by color, shape or size, without anyone telling you the categories. Unsupervised learning does the same thing with data: it looks for groups, directions or compressed summaries that the rows suggest.

From first principles, unsupervised methods define an objective that depends only on the inputs. In clustering, the objective is usually the within-group spread, the sum of distances from points to their group centers, or a density criterion. Because no label exists, you cannot compute accuracy. You measure the quality of the grouping with internal criteria, such as inertia and silhouette, and with external checks, such as whether the groups match a business outcome. The risk is that the algorithm always returns groups, even when the data has none.

```python
import numpy as np
from sklearn.cluster import KMeans

rng = np.random.default_rng(0)
uniform = rng.uniform(0, 1, (300, 2))   # no real groups in this data
labels = KMeans(n_clusters=3, n_init=10, random_state=0).fit_predict(uniform)
print(np.bincount(labels))              # k-means still returns three groups
```

The mistake is treating every output as meaningful. The limitation is that clustering is sensitive to the choice of features and scale, and a different preprocessing step can produce entirely different groups. The check is to ask whether the groups are stable under small changes and whether they mean something to someone who knows the domain. In the market, a marketing team may use clusters to design campaigns, and they test the clusters by measuring whether the campaigns perform differently across them.

### k-means

Picture placing a few flags on a map to mark the centers of towns. Each house goes to the nearest flag. Then you move each flag to the middle of the houses assigned to it. Repeat until nothing moves. That is k-means in plain words.

From first principles, k-means minimizes the inertia `sum_i ||x_i - mu_{c_i}||^2`, where `c_i` is the cluster of point `i` and `mu_c` is the centroid of cluster `c`. The algorithm alternates two steps. The assignment step gives each point to its nearest centroid. The update step sets each centroid to the mean of its points. Each step cannot increase the inertia, so the loop converges, but only to a local minimum. The number of clusters `k` is a parameter you choose, not one the algorithm discovers.

```python
import numpy as np

def kmeans(X, k, iters=100, seed=0):
    rng = np.random.default_rng(seed)
    centers = X[rng.choice(len(X), k, replace=False)]
    for _ in range(iters):
        dists = ((X[:, None, :] - centers[None, :, :]) ** 2).sum(axis=2)
        labels = dists.argmin(axis=1)
        new_centers = np.array([X[labels == j].mean(axis=0) for j in range(k)])
        if np.allclose(new_centers, centers):
            break
        centers = new_centers
    return labels, centers

rng = np.random.default_rng(1)
X = np.vstack([rng.normal([0, 0], 0.5, (50, 2)), rng.normal([4, 4], 0.5, (50, 2))])
labels, centers = kmeans(X, k=2)
print(centers.round(2))   # close to (0, 0) and (4, 4)
```

The mistake is ignoring empty clusters. If no point is assigned to a centroid, the mean is undefined, and the code above would produce NaN. Production code reseeds the empty centroid. The limitation of k-means is strong: it assumes round clusters of similar size, and it is sensitive to outliers because the mean is pulled by extreme values. In industry, k-means is the default for segmentation because it is fast and scales well, and teams accept its shape assumptions when the features have been engineered to be roughly spherical.

### Centroids

The centroid is the balance point of a cluster, the place where the points would sit if you hung them on a weightless bar. For squared distance, the mean is the best single representative of a set of points, which is why k-means uses it.

From first principles, for a set of points `S`, the value `c` that minimizes `sum_{x in S} ||x - c||^2` is the mean of `S`. This follows from setting the gradient to zero, the same kind of argument you used on Day 25. A consequence is that centroids summarize a cluster and move as the points change, which means they are not members of the data. A centroid can sit in an empty region, far from any actual row, and that is a sign that the cluster is not compact. Centroids are useful for interpretation: you can read them as a typical profile of each group.

```python
import numpy as np

points = np.array([[1.0, 2.0], [3.0, 2.0], [2.0, 5.0]])
centroid = points.mean(axis=0)
print(centroid.round(2))  # [2.   3.   ]

def sse(c):
    return np.sum((points - c) ** 2)

print(round(sse(centroid), 2), "<=", round(sse(points[0]), 2))  # the mean has the smallest sum
```

The mistake is treating a centroid as an example of a typical row. The centroid may not correspond to any row, especially in high dimensions or with categorical features encoded as numbers. The limitation is that the mean is sensitive to outliers, and in heavy-tailed data a median-based method such as k-medoids may describe clusters better. In industry, a centroid profile such as "average spend 40 dollars, mostly weekday visits" is how segments are described to business teams, so the choice of summary matters for communication.

### Distance

Consider two groups of stores in a city. Whether they look similar depends on whether you measure distance in kilometers, in travel time or in the number of shared customers. Each choice produces different clusters, and the algorithm has no opinion about which is right.

From first principles, k-means implicitly uses squared Euclidean distance, so features with large ranges dominate the assignment, as you saw in the kNN lesson on Day 27. Standardize features before clustering unless their units are comparable. For binary or count data, a different distance such as Jaccard or cosine distance may be more natural. Hierarchical clustering and DBSCAN accept other metrics. The choice of metric is therefore part of the model, and it should be reported.

```python
import numpy as np
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler

rng = np.random.default_rng(0)
income = rng.normal(50000, 15000, 300)
age = rng.normal(40, 10, 300)
X = np.column_stack([income, age])
raw = KMeans(n_clusters=2, n_init=10, random_state=0).fit(X)
scaled = KMeans(n_clusters=2, n_init=10, random_state=0).fit(StandardScaler().fit_transform(X))
print("raw cluster sizes   ", np.bincount(raw.labels_))
print("scaled cluster sizes", np.bincount(scaled.labels_))
```

The mistake is clustering raw features with mixed units and then describing the clusters as if age mattered. Income dominates the raw distance, so the clusters mostly split by income. The limitation of standardization is that it treats all features as equally important, which may not reflect the business. Weighting features is a deliberate decision, and it should be made in the open. In the market, teams doing segmentation usually document the feature scaling in the same table as the cluster profiles.

### Initialization

Imagine starting a search for three lost hikers by placing your searchers at random spots on the map. Some searchers may start in the same valley and waste time, while others never reach a region at all. A better start spreads the searchers out, so each covers new ground.

From first principles, k-means converges to a local minimum that depends on the starting centroids. Random initialization can place two centroids in the same true cluster, which leaves another true cluster without a centroid. The k-means++ method chooses the first centroid at random, then picks each next centroid with probability proportional to its squared distance from the nearest existing centroid. The authors show that this choice gives a solution that is, in expectation, within a logarithmic factor of the best possible inertia, and it is the default in scikit-learn. Running several starts, with `n_init`, and keeping the lowest inertia further reduces the risk of a poor local minimum.

```python
import numpy as np
from sklearn.cluster import KMeans

rng = np.random.default_rng(3)
X = np.vstack([rng.normal(c, 0.4, (100, 2)) for c in [(0, 0), (3, 0), (0, 3), (3, 3)]])

for init, n in [("random", 1), ("k-means++", 1), ("k-means++", 10)]:
    km = KMeans(n_clusters=4, init=init, n_init=n, random_state=0).fit(X)
    print(f"{init:10s} n_init={n:2d} inertia={km.inertia_:.2f}")
```

The mistake is trusting a single run. Two runs with different seeds can produce different inertia values, and the lower one is usually the better fit. The limitation is that lower inertia does not mean the clusters are more meaningful, because inertia always decreases as k grows. In industry, a team reports the run with the lowest inertia, the stability across seeds, and the chosen seed, so that others can reproduce the result.

### Hierarchical clustering concept

Think of a family tree. Individuals merge into couples, couples into families, and families into clans. You can read the tree at any height and get a different number of groups. Hierarchical clustering builds that tree from the data, from the bottom up.

From first principles, agglomerative clustering starts with every point in its own cluster. At each step it merges the two closest clusters, where the distance between clusters is defined by the linkage. Single linkage uses the closest pair of points, complete linkage uses the farthest pair, average linkage uses the mean of all pairs, and Ward linkage merges the pair that gives the smallest increase in within-cluster variance. The dendrogram records each merge and its height, and cutting it at a chosen height gives a flat clustering. The full procedure is `O(n^2)` in memory for the distance matrix, which limits it to moderate datasets.

```python
import numpy as np
from scipy.cluster.hierarchy import linkage, fcluster

rng = np.random.default_rng(0)
X = np.vstack([rng.normal([0, 0], 0.3, (20, 2)), rng.normal([3, 3], 0.3, (20, 2))])
Z = linkage(X, method="ward")
labels = fcluster(Z, t=2, criterion="maxclust")
print(np.bincount(labels))   # two groups of 20
```

The mistake is choosing the cut height by eye without checking stability, and the second is using single linkage on noisy data, where a chain of points can merge two real clusters. The limitation is cost and sensitivity to the linkage, and once a merge happens it cannot be undone. In practice, Ward linkage is a sensible default for compact groups, and the dendrogram is a helpful visual for explaining structure to a non-specialist.

### DBSCAN concept

Picture a crowd at night, with people gathered in clumps and a few walking alone. A clump is a cluster if enough people are close together. The people walking alone are noise. DBSCAN defines clusters the same way, by density rather than by distance to a center.

From first principles, DBSCAN has two parameters: `eps`, the radius of a neighborhood, and `min_samples`, the number of neighbors needed to call a point a core point. A core point has at least `min_samples` points within `eps`. Core points that are within `eps` of each other form a cluster, and points within `eps` of a core point join that cluster as border points. Everything else is noise. Because the method follows density, it finds clusters of any shape and does not require the number of clusters in advance. Its weakness is that a single `eps` must work for all clusters, and clusters with different densities are hard to capture together.

```python
import numpy as np
from sklearn.cluster import DBSCAN
from sklearn.datasets import make_moons

X, _ = make_moons(n_samples=300, noise=0.05, random_state=0)
labels = DBSCAN(eps=0.2, min_samples=5).fit_predict(X)
print("clusters:", sorted(set(labels) - {-1}), "noise points:", int((labels == -1).sum()))
```

The mistake is choosing `eps` without a plot of the k-distance, which shows how far each point is from its k-th neighbor. The knee in that plot suggests a reasonable `eps`. The limitation is high-dimensional data, where distances become less informative, and data with varying density. In the market, DBSCAN is used for anomaly detection in location and sensor data, where noise points are the signal worth investigating.

---

## Hands-on exercises

1. Run the k-means function from this lesson on the two-blob data with `k` set to 1, 2 and 3. Record the inertia for each and explain why it always falls as `k` grows.
2. Compute the centroid of three points by hand, then check that the sum of squared distances is smaller at the mean than at any of the three points.
3. Run `KMeans` on the same data 20 times with different seeds and `init="random"`, then with `init="k-means++"`. Report the spread of the final inertia in each case.
4. Build a dataset with two features on very different scales, cluster it before and after standardization, and describe in one sentence how the clusters change.
5. Build a dendrogram for 30 points with `scipy.cluster.hierarchy.dendrogram`, and compare the cluster labels from Ward linkage and single linkage at the same number of clusters.
6. Run DBSCAN on two moons with `eps` set to 0.1, 0.2 and 0.4. Describe how the number of clusters and the noise count change.
7. Compute the silhouette score for k from 2 to 6 on a dataset of your choice. Plot it against k and state whether the best silhouette matches the number of groups you would pick by looking at the data.

---

## Project: Clustering a customer dataset with validation

This is a medium-weight project. The algorithms are standard library calls, so the value is in the validation: stability across seeds, the choice of k, and a clear profile of each cluster.

**Repo name:** `100-days-ai-engineering-day-32-clustering`

**Goal:** segment a numeric customer dataset with k-means, check the choice of k and the stability of the clusters, compare with DBSCAN, and describe each segment in plain language.

**Repo layout:**

```
100-days-ai-engineering-day-32-clustering/
├── README.md
├── requirements.txt
├── segment.py
├── stability.py
├── results/
│   ├── silhouette.png
│   └── profiles.md
└── tests/
    └── test_segment.py
```

**Starter code for `segment.py`:**

```python
import numpy as np
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler


def prepare(X):
    """Standardize the numeric features. Return the scaled array and the fitted scaler."""
    raise NotImplementedError


def fit_kmeans(X_scaled, k, seed=0):
    """Fit k-means with k-means++ initialization and n_init=10. Return the fitted model."""
    raise NotImplementedError


def profile(X_raw, labels, names):
    """Return a list of dicts, one per cluster, with the size and the mean of each raw feature."""
    raise NotImplementedError
```

**Starter code for `tests/test_segment.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from segment import prepare, fit_kmeans, profile


def make_data(seed=0):
    rng = np.random.default_rng(seed)
    a = rng.normal([20, 100], [3, 10], (100, 2))
    b = rng.normal([50, 400], [5, 40], (100, 2))
    return np.vstack([a, b])


def test_prepare_gives_zero_mean_and_unit_variance():
    Xs, _ = prepare(make_data())
    assert np.allclose(Xs.mean(axis=0), 0.0, atol=1e-8)
    assert np.allclose(Xs.std(axis=0), 1.0, atol=1e-8)


def test_two_clusters_are_recovered():
    Xs, _ = prepare(make_data())
    labels = fit_kmeans(Xs, k=2).labels_
    assert len(set(labels)) == 2
    assert min(np.bincount(labels)) > 80


def test_profile_has_one_entry_per_cluster():
    X = make_data()
    Xs, _ = prepare(X)
    labels = fit_kmeans(Xs, k=2).labels_
    out = profile(X, labels, ["visits", "spend"])
    assert len(out) == 2
    assert out[0]["size"] + out[1]["size"] == 200
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `scipy`, `matplotlib` and `pytest`.
2. Find a numeric customer or transaction dataset. The UCI Online Retail dataset or a synthetic version generated with `numpy` both work. Check its license, and generate the data in code if the license does not allow redistribution.
3. Implement `prepare`, `fit_kmeans` and `profile`. Make the tests pass.
4. In `stability.py`, compute the silhouette score for k from 2 to 8, and save the plot to `results/silhouette.png`. Run k-means with ten different seeds for the chosen k and report the adjusted Rand index between each pair of runs, or the proportion of points that change cluster.
5. Choose k using both the silhouette score and the stability result, and write the reason in the README.
6. Compare the k-means result with DBSCAN on the same scaled data, and report the number of clusters and noise points.
7. Write `results/profiles.md` with the size and the average of each raw feature for every segment, and give each segment a short plain-language name.
8. Write the README, including the limitations: the features chosen, the scaling, and the fact that clusters are a hypothesis.
9. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python stability.py`
- [ ] The data source and license are stated
- [ ] The silhouette plot is embedded with a caption that gives the chosen k
- [ ] The stability result across seeds is reported
- [ ] The README names one feature choice that changes the segments, and one limitation of k-means

**Done when:**

- `pytest` passes all three tests
- The chosen k is supported by both the silhouette score and the stability check, or the README explains the disagreement
- Each segment has a profile table and a plain-language name
- You can explain in one paragraph why the clusters are a hypothesis and not a fact, and what check would confirm one of them

**What to commit as evidence:** the source files, the tests, the silhouette plot, the stability output, the profile table and the README with the limitations.

---

## Check your understanding

1. A colleague says the k-means output with `k = 5` proves there are five customer types. Explain why this claim is too strong, and name one check that would support or weaken it.
2. Two features are measured in dollars and in visits per year. Predict which feature dominates the k-means assignment before scaling, and explain the fix.
3. Explain why the k-means update step uses the mean and not the median, and say when the median would be a better choice.
4. A k-means run with random initialization gives a lower inertia than a second run. Is the first run necessarily the better clustering? Explain.
5. Describe the difference between single linkage and Ward linkage, and say which one is more likely to chain two real clusters together through noise.
6. A DBSCAN run labels 40 percent of the points as noise. Give two possible explanations and the next step you would take for each.
7. You are asked to segment customers into exactly four groups for a campaign. Explain how you would combine the silhouette score, the stability check and a business constraint to make the choice, and what you would report if the silhouette favors two groups.

### Answers

1. The algorithm always returns the requested number of groups, even when the data has no natural groups, so the output is a hypothesis about structure. A check is to measure whether the clusters are stable across seeds and sample splits, and whether they differ on an outcome that the business cares about, such as retention or spend.
2. Dollars will dominate, because the range of spend is usually much larger than the range of visits, so the squared distance is mostly about spend. Standardizing both features gives each one a comparable scale before clustering.
3. The mean minimizes the sum of squared distances, which is the objective that k-means minimizes, so the update is the step that reduces the objective most. The median minimizes the sum of absolute distances, and it is a better choice when outliers are common, which is the situation in which k-medoids or median-based variants are used.
4. No. Lower inertia means a better fit to the k-means objective, but it does not mean the clusters are more meaningful. Inertia also falls as k grows, and it does not reflect whether the groups match any real pattern.
5. Single linkage merges clusters by their closest pair of points, so a chain of noisy points can connect two distinct groups. Ward linkage merges by minimizing the increase in within-cluster variance, which favors compact groups and is less likely to chain them together.
6. One explanation is that `eps` is too small for the density of the data, so most points do not have enough neighbors. The next step is to plot the k-distance curve and choose `eps` near its knee. A second explanation is that the data is genuinely scattered, with many customers who do not belong to a dense group. In that case the noise points may be a real segment, and they should be profiled rather than discarded.
7. I would check the silhouette and the stability for k equal to 2, 3 and 4, then ask whether the four-group solution is stable and interpretable enough for the campaign. If the silhouette favors two groups, I would report that result honestly, show the four-group solution with its trade-off, and let the business decide whether the operational benefit of four segments justifies the weaker separation.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Cluster analysis](https://en.wikipedia.org/wiki/Cluster_analysis)
- Wikipedia: [K-means clustering](https://en.wikipedia.org/wiki/K-means_clustering)
- Wikipedia: [Hierarchical clustering](https://en.wikipedia.org/wiki/Hierarchical_clustering)
- Wikipedia: [DBSCAN](https://en.wikipedia.org/wiki/DBSCAN)
- Wikipedia: [Silhouette (clustering)](https://en.wikipedia.org/wiki/Silhouette_(clustering))
- scikit-learn user guide, "Clustering", at scikit-learn.org under the stable documentation, including the comparison of clustering algorithms on toy datasets.

**Tutorials**

- scikit-learn documentation, `KMeans`, `AgglomerativeClustering` and `DBSCAN`, with the parameter notes.
- GeeksforGeeks, search "K-means clustering introduction" and "hierarchical clustering in machine learning" for worked examples with small tables.

**Watch**

- StatQuest with Josh Starmer (YouTube), "K-means clustering" and "Hierarchical Clustering".

**Practice**

- Kaggle Learn, the "Intro to Machine Learning" course, and a notebook that clusters a public dataset and profiles the groups.

---

## Tomorrow

Tomorrow you deal with the curse of dimensionality directly. Many datasets have far more features than they need, and distances become less informative as the number of features grows. Principal component analysis from Day 10 returns as a practical tool, and you will also meet t-SNE and UMAP, which are used to draw high-dimensional data in two dimensions, along with their limitations for analysis.

**Next:** [Day 33: Dimensionality Reduction](Day-33-Dimensionality-Reduction.md)

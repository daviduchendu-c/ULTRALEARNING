# Day 33: Dimensionality Reduction

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You should have finished Module 2 Day 10 (linear algebra, eigenvalues and the SVD), since this day applies those ideas to real pipelines. The nonlinear methods are presented as tools with limits, not as magic.

**By the end of this day, you will have:**

- A simulation that shows distances losing contrast as dimensions grow
- A PCA pipeline with a variance-explained curve, used to choose a number of components
- A projection of high-dimensional data to two dimensions with PCA, t-SNE and UMAP, and a note on what each plot can and cannot show
- A downstream test showing whether the reduced data keeps the information a classifier needs

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- pandas 2.2 or newer (`pip install pandas`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause, for PCA and t-SNE
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the plots
- umap-learn 0.5 or newer (`pip install umap-learn`), BSD 3-Clause, for UMAP
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section

**Prerequisites:**

- Module 2 Day 10: eigenvalues, the singular value decomposition and PCA as a projection
- Day 24: train, validation and test splits, and the leakage rules
- Day 27: distances and why high-dimensional neighborhoods behave differently
- Day 32: clustering and the question of whether structure is real

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Curse of dimensionality](#curse-of-dimensionality)
- [PCA](#pca)
- [Variance](#variance)
- [Projection](#projection)
- [t-SNE and UMAP concepts and limitations](#t-sne-and-umap-concepts-and-limitations)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Dimensionality reduction with a downstream check](#project-dimensionality-reduction-with-a-downstream-check)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A dataset with 300 columns is not necessarily a dataset with 300 useful directions. Sensor readings move together, survey questions repeat each other, and word counts follow a few themes. Dimensionality reduction finds a smaller set of directions that keeps most of what matters, which makes models faster, more stable and easier to inspect.

The need is practical. Many models slow down or lose accuracy as the number of features grows, especially distance-based methods, as the curse of dimensionality shows. Visualization is another reason: a person cannot look at 300 axes, so a two-dimensional map is the only way to see the data. And embeddings from modern models are high-dimensional by design, so the same tools help you inspect them.

On the job, the mistakes are usually about what the reduction removed. A plot can look convincing and still hide the variation that the classifier needed. A reduced dataset can score well on a metric and fail on a rare but important group. The habit to build today is to test every reduction against the downstream task, and to read a visualization as a picture of one aspect of the data, not as the whole.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1901 | Karl Pearson describes lines and planes of closest fit to points, the earliest form of principal component analysis. | The geometry of PCA is more than a century old. |
| 1933 | Harold Hotelling develops principal components for correlated variables and gives them the name used today. | PCA became a standard tool in psychology, economics and quality control. |
| 1961 | Richard Bellman introduces the term "curse of dimensionality" in his work on adaptive control processes. | The phrase names the problem that dimensionality reduction tries to solve. |
| 2008 | Laurens van der Maaten and Geoffrey Hinton publish t-SNE in the Journal of Machine Learning Research, building on Stochastic Neighbor Embedding from 2002. The method preserves local neighborhoods in a two-dimensional map. | t-SNE became the standard tool for visualizing embeddings and single-cell data. |
| 2018 | Leland McInnes, John Healy and James Melville publish UMAP, which uses a fuzzy topological model and is faster than t-SNE on large data. | UMAP is now common in visualization pipelines and as a preprocessing step before clustering. |

**Takeaway:** the linear method is over a century old and remains the default. The nonlinear methods are newer, more powerful for pictures, and easier to misread.

---

## Glossary

- **Curse of dimensionality**: the set of problems that appear as the number of features grows, including sparse data, unreliable distances and the need for exponentially more samples.
- **Dimensionality reduction**: transforming data into fewer features while keeping as much useful structure as possible.
- **Feature extraction**: creating new features from old ones, such as principal components.
- **Feature selection**: keeping a subset of the original features.
- **Principal component**: a direction, a unit vector, along which the centered data has the largest remaining variance.
- **Explained variance ratio**: the fraction of total variance captured by one component.
- **Projection**: the coordinates of the data along a set of directions, computed as dot products.
- **Embedding**: a learned or computed vector representation of an item, usually with many dimensions.
- **Perplexity**: the main parameter of t-SNE, roughly the effective number of neighbors each point considers.
- **Manifold**: a lower-dimensional shape on which high-dimensional data lies approximately, such as a curved sheet in three dimensions.

---

### Curse of dimensionality

Imagine filling a cube with random points. In a line of length 1, a sample of 10 points covers the segment every 0.1 units. In a square, the same density needs 100 points. In a cube, it needs 1,000. Each added dimension multiplies the number of points needed to cover the space at the same density. Real data is never that dense, so models lose coverage.

From first principles, the volume of a cube grows exponentially in the number of dimensions, while the number of samples is fixed. As dimension grows, the typical distance between random points grows, and the gap between the nearest and farthest points shrinks relative to their size. That loss of contrast hurts distance-based methods such as kNN and clustering. The simulation below shows it directly.

```python
import numpy as np

rng = np.random.default_rng(0)
for d in (2, 10, 100, 1000):
    X = rng.uniform(size=(500, d))
    q = rng.uniform(size=d)
    dist = np.linalg.norm(X - q, axis=1)
    print(f"d={d:5d}  (max-min)/min = {(dist.max() - dist.min()) / dist.min():.2f}")
```

The contrast ratio falls as dimension grows, so the nearest and farthest points look more alike. The mistake is assuming that more features always give more information. Noise features add distance without adding signal, and the curse grows with them. The limitation of the curse is that it depends on the data: data with low intrinsic dimension, such as images that lie on a low-dimensional manifold, can be learned well even with many raw features. In industry, the first response to a slow or unstable distance model with hundreds of features is usually to remove redundant and noisy features, and then to test a reduction method.

### PCA

Think of photographing a long, thin cloud of dust from the best angle. You want the camera to show the long direction, because that is where the cloud spreads. Principal component analysis finds that direction in data and then the next one, at right angles to the first.

From first principles, after centering the data, the first principal component is the unit vector `w` that maximizes the variance of the projections `X w`. The solution is the eigenvector of the covariance matrix with the largest eigenvalue, as you saw in Module 2. Computing the singular value decomposition of the centered data gives the same components more stably. The projection onto the first `k` components keeps the most variance possible for any linear map to `k` dimensions, which is the Eckart-Young result from Module 2 applied to the data matrix.

```python
import numpy as np
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler

rng = np.random.default_rng(0)
base = rng.normal(size=(300, 3))
X = base @ rng.normal(size=(3, 20)) + 0.1 * rng.normal(size=(300, 20))  # 20 features from 3 signals
Xs = StandardScaler().fit_transform(X)

pca = PCA().fit(Xs)
cumulative = np.cumsum(pca.explained_variance_ratio_)
print("components for 95% variance:", int(np.argmax(cumulative >= 0.95)) + 1)
```

The mistake is running PCA without standardizing features that have different units. The first component will then follow the largest-scale feature, as you saw in Module 2. The limitation is that PCA is linear and unsupervised: the directions of greatest variance may carry none of the signal that the label needs. In industry, PCA is a standard step in sensor pipelines, image preprocessing and genomics, and it is often used to reduce noise before a classifier, which works when the signal is in the high-variance directions and fails when it is not.

### Variance

Think of a bag of coins with different sizes. Variance measures how much the coins differ from the average size. PCA ranks the directions by how much the data varies along them, and the explained variance ratio says what share of the total each direction holds.

From first principles, for centered data the total variance is the sum of the eigenvalues of the covariance matrix, which equals the sum of the squared singular values divided by `n - 1`. The explained variance ratio of component `j` is its eigenvalue divided by that total. A cumulative curve shows how many components are needed to reach a chosen threshold, such as 95 percent. This is a simple rule, but it is not a rule about information for a task. A component with tiny variance can still be the one a classifier needs, and a large-variance component can be pure noise, such as a sensor drift.

```python
import numpy as np
from sklearn.decomposition import PCA

rng = np.random.default_rng(1)
X = rng.normal(size=(200, 5)) * np.array([10.0, 5.0, 1.0, 0.5, 0.1])
pca = PCA().fit(X)
print(np.round(pca.explained_variance_ratio_, 3))
print("total variance equals sum of eigenvalues:",
      np.isclose(pca.explained_variance_.sum(), np.var(X, axis=0, ddof=1).sum()))
```

The mistake is equating variance with importance. The limitation is that the variance ratio measures spread in the features, not usefulness for the target. The correct test is to train the model on the reduced features and compare with the full features on validation data, which the project below does. In the market, teams often report "95 percent of variance retained" in a slide, and the claim is accurate but says nothing about accuracy, which is why the downstream check matters.

### Projection

Imagine shining a light on a three-dimensional object and looking at its shadow on a wall. The shadow loses the depth, but it keeps the shape along the wall. A projection is that shadow: it maps each point to coordinates along a chosen set of directions.

From first principles, if `W` is a matrix whose columns are orthonormal principal components, the projection of a centered row `x` is `z = W^T x`, a vector with one coordinate per component. The reconstruction back in the original space is `x_hat = W z`, which is the closest point to `x` that lies in the chosen subspace. The reconstruction error is the part of `x` that is orthogonal to the subspace, and its size tells you how much information the projection discarded.

```python
import numpy as np
from sklearn.decomposition import PCA

rng = np.random.default_rng(2)
X = rng.normal(size=(100, 6)) @ rng.normal(size=(6, 6))
X = X - X.mean(axis=0)

for k in (1, 3, 6):
    pca = PCA(n_components=k).fit(X)
    X_hat = pca.inverse_transform(pca.transform(X))
    print(k, "reconstruction RMSE", round(np.sqrt(np.mean((X - X_hat) ** 2)), 4))
```

The mistake is treating the projection as lossless. The reconstruction error is the price of the reduction, and it should be reported next to the number of components. The limitation is that a linear projection can only keep linear directions, so a curved structure is cut into pieces. In industry, reconstruction error is also used as an anomaly score: a point that reconstructs poorly is unusual with respect to the training data.

### t-SNE and UMAP concepts and limitations

Imagine a map of a city drawn by someone who only remembers which neighborhoods are next to each other. The map keeps local neighbors close, but it will distort the distances between far-apart districts. t-SNE and UMAP make maps of high-dimensional data in the same spirit.

From first principles, t-SNE converts the distances between points into probabilities that a point would pick each other point as a neighbor, using a Gaussian kernel whose width is set by the perplexity. It then places points in two dimensions so that similar neighbor probabilities are reproduced, using a heavy-tailed distribution in the low-dimensional space to separate clusters. UMAP builds a weighted nearest-neighbor graph, then optimizes a low-dimensional layout so that the graph structure is reproduced. Both preserve local neighborhoods well. Neither preserves global distances, cluster sizes or densities faithfully, and both depend on their parameters. The layout changes with the random seed.

```python
import numpy as np
from sklearn.manifold import TSNE
from sklearn.datasets import load_digits

X, y = load_digits(return_X_y=True)
emb = TSNE(n_components=2, perplexity=30, init="pca", random_state=0).fit_transform(X)
print(emb.shape)  # (1797, 2), ready to plot, colored by y
```

The mistake is reading distances between clusters in a t-SNE plot as real distances, or treating the size of a cluster as its real size. Both are artifacts of the method. Another mistake is running t-SNE once and drawing a conclusion. Run it with several perplexities and seeds before you describe a pattern. UMAP is often faster on large data, and it preserves more global structure, but it still should not be used as a measurement. In industry, t-SNE and UMAP are used to inspect embeddings and to check whether labeled groups separate, and the picture is a hypothesis to test with metrics.

---

## Hands-on exercises

1. Run the distance-contrast simulation from this lesson with 500 points and dimensions 2, 10, 50 and 200. Describe the trend in one sentence, and explain why it matters for kNN.
2. Generate 300 rows of 10 features where two features are nearly copies of each other. Compute the PCA explained variance ratio and explain what the second component is capturing.
3. Standardize a dataset with features in very different units, then run PCA with and without standardization. Report how the first component's loadings change.
4. Plot the cumulative explained variance for a real dataset, such as `load_digits`, and find the number of components needed for 90 and 95 percent.
5. Train a logistic regression on the full features and on the PCA-reduced features with the number of components from exercise 4, using a pipeline and a validation split. Report both scores.
6. Compute the reconstruction error for several values of k on a dataset of your choice. Plot it and explain where the curve bends.
7. Run t-SNE with perplexity 5, 30 and 50 on `load_digits`, and describe in a short paragraph which clusters stay together in each plot and which do not.

---

## Project: Dimensionality reduction with a downstream check

This is a medium-weight project. The code is short and uses library methods. The deliverable is a reduction that is tested against a downstream model, with honest plots and a clear recommendation.

**Repo name:** `100-days-ai-engineering-day-33-dimensionality-reduction`

**Goal:** reduce a high-dimensional dataset with PCA, check how much predictive information survives at each number of components, and produce a two-dimensional visualization that is labeled as a picture, not a measurement.

**Repo layout:**

```
100-days-ai-engineering-day-33-dimensionality-reduction/
├── README.md
├── requirements.txt
├── reduce.py
├── study.py
├── results/
│   ├── variance_curve.png
│   ├── downstream_table.md
│   └── embedding_maps.png
└── tests/
    └── test_reduce.py
```

**Starter code for `reduce.py`:**

```python
import numpy as np
from sklearn.decomposition import PCA
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler


def make_reducer(n_components, seed=0):
    """Return an unfitted pipeline: StandardScaler, then PCA with n_components and random_state=seed."""
    raise NotImplementedError


def reconstruction_rmse(pipeline, X):
    """Return the RMSE between X and its reconstruction through the pipeline.
    Hint: transform, then inverse-transform the PCA step, then undo the scaling.
    """
    raise NotImplementedError
```

**Starter code for `tests/test_reduce.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from reduce import make_reducer, reconstruction_rmse


def make_data(seed=0):
    rng = np.random.default_rng(seed)
    base = rng.normal(size=(200, 3))
    return base @ rng.normal(size=(3, 12)) + 0.05 * rng.normal(size=(200, 12))


def test_reducer_output_has_requested_dimension():
    X = make_data()
    Z = make_reducer(3).fit_transform(X)
    assert Z.shape == (200, 3)


def test_more_components_reduce_reconstruction_error():
    X = make_data()
    errors = []
    for k in (1, 3, 6):
        p = make_reducer(k).fit(X)
        errors.append(reconstruction_rmse(p, X))
    assert errors[0] > errors[1] > errors[2]


def test_three_components_capture_the_three_signals():
    X = make_data()
    p = make_reducer(3).fit(X)
    assert reconstruction_rmse(p, X) < 0.2
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib`, `umap-learn` and `pytest`.
2. Choose a high-dimensional labeled dataset. The `load_digits` dataset from scikit-learn, with 64 features, is a good start. A larger one such as Fashion-MNIST works if your hardware allows.
3. Implement `make_reducer` and `reconstruction_rmse`. Make the tests pass.
4. In `study.py`, split the data into train, validation and test parts. Fit the reducer on the training part only, sweep the number of components from 2 to 40, and record the variance captured, the reconstruction error and a logistic regression's validation accuracy at each value.
5. Save the variance curve and the downstream table to `results/`. Choose a number of components with a stated rule, such as the smallest that keeps validation accuracy within one percentage point of the full model.
6. Fit the chosen reducer on the training part, score the test part once, and record the number.
7. Make a two-dimensional map with PCA, t-SNE and UMAP, colored by label, and save it to `results/embedding_maps.png`. Add a caption that says the maps are pictures and do not measure distance.
8. Write the README. Include the rule for choosing components, the test score, and a paragraph on what the maps can and cannot show.
9. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python study.py`
- [ ] The dataset source and license are stated
- [ ] The variance curve and downstream table are embedded or linked
- [ ] The number of components is chosen by a written rule and reported with its test score
- [ ] The README states that the maps are pictures, not measurements, and names one misreading to avoid

**Done when:**

- `pytest` passes all three tests
- The chosen number of components keeps validation accuracy within the stated tolerance of the full-feature model
- The test score is reported once, after the choice was made
- You can explain in one paragraph why a high explained variance does not guarantee a good classifier

**What to commit as evidence:** the source files, the tests, the variance curve, the downstream table, the embedding maps and the README with the rule and the test score.

---

## Check your understanding

1. A feature set has 400 columns and a 95 percent explained variance at 30 components. A colleague says the reduced set keeps 95 percent of the information. Correct the claim, and name the test that would support a claim about the classifier.
2. Explain why the curse of dimensionality hurts a kNN classifier more than a linear model with the same features.
3. The first principal component of a dataset is dominated by one feature with a large unit. What did you forget to do, and what changes after you do it?
4. Explain why the projection onto the top `k` principal components is the best rank-`k` linear approximation in the squared-error sense, using the SVD from Module 2.
5. A t-SNE plot shows two clusters that look far apart and one that looks small. Explain why you cannot conclude that the distances or sizes are real, and name one check.
6. You use PCA to reduce data before a classifier. The classifier loses accuracy on a rare class. Explain how a high-variance projection could hide the signal for that class, and describe one alternative that uses the label.
7. You must present a two-dimensional map of customers to a non-technical audience. Describe what you would say the map shows, what it does not show, and how you would support the claim with a number.

### Answers

1. The 95 percent describes the variance of the features. It says nothing about the information a prediction needs. The test that supports a claim about the classifier is to train the same model on the full and reduced features and compare validation accuracy, with the reduction fitted only on the training data.
2. A kNN classifier relies on distances, and distances lose contrast in high dimensions, so the nearest neighbors become less informative. A linear model sums weighted features and does not depend on the ranking of distances, so it is less affected, though it still suffers from noise features and needs more data to estimate many coefficients.
3. The standardization step was missing. Without it, the feature with the largest unit has the largest variance and dominates the first component. After standardizing each feature, the components reflect shared variation rather than units.
4. The SVD of the centered data gives `X = U S V^T`, and keeping the top `k` singular values gives the best rank-`k` approximation in the Frobenius norm, which is the squared-error measure. Projecting onto the top `k` right singular vectors achieves that approximation, so PCA is the best linear projection to `k` dimensions.
5. t-SNE preserves local neighborhoods, and it does not preserve global distances or cluster sizes. The apparent distances and sizes depend on the perplexity, the seed and the optimization. A check is to rerun with different seeds and perplexities, and to see whether the clusters persist.
6. A class that differs only in a low-variance direction will be removed when the projection keeps only the high-variance components. An alternative that uses the label is a supervised reduction, such as linear discriminant analysis, or selecting components by how well they predict the target on validation data.
7. I would say that the map groups customers whose features are similar, and that customers close together behave alike on the measured features. I would say it does not show true distances between groups, cluster sizes or the reason customers belong to a group. To support the claim, I would report the cluster profiles and a measurable difference in an outcome, such as retention, between groups that look separate on the map.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Curse of dimensionality](https://en.wikipedia.org/wiki/Curse_of_dimensionality)
- Wikipedia: [Dimensionality reduction](https://en.wikipedia.org/wiki/Dimensionality_reduction)
- Wikipedia: [Principal component analysis](https://en.wikipedia.org/wiki/Principal_component_analysis)
- Wikipedia: [T-distributed stochastic neighbor embedding](https://en.wikipedia.org/wiki/T-distributed_stochastic_neighbor_embedding)
- Wikipedia: [Uniform manifold approximation and projection](https://en.wikipedia.org/wiki/Uniform_manifold_approximation_and_projection)
- scikit-learn user guide, "Decomposing signals in components (matrix factorization problems)" and "Manifold learning", at scikit-learn.org under the stable documentation.

**Tutorials**

- scikit-learn documentation, the example "Comparison of Manifold Learning methods", which shows t-SNE and other methods on the same data.
- UMAP documentation, "How UMAP Works" and the parameter guide, at umap-learn.readthedocs.io.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Principal Component Analysis (PCA), Step-by-Step".
- 3Blue1Brown (YouTube), "Eigenvectors and eigenvalues | Chapter 14, Essence of linear algebra", to connect the geometry with the variance directions.

**Practice**

- Google's "How to Use t-SNE Effectively" interactive article at distill.pub, which shows how perplexity changes the same data. Search the title to find it.

---

## Tomorrow

Tomorrow you measure classifiers properly. Accuracy is only one number, and on imbalanced data it can hide almost every mistake that matters. You will work with the confusion matrix, precision, recall, F1, specificity, ROC and precision-recall curves, and threshold selection. The thresholds from Day 26 and the probabilities from Day 28 return, and the reduced features from today become a live test of your metrics.

**Next:** [Day 34: Model Evaluation: Classification](Day-34-Model-Evaluation-Classification.md)

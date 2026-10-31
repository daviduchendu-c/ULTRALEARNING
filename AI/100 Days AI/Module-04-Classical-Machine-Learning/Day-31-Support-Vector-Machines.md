# Day 31: Support Vector Machines

*100 Days of AI Engineering, Module 4: Classical Machine Learning*

**Time:** about 3.5 hours
**Level:** intermediate. You need Day 25 (linear models and the gradient) and Day 27 (distances). The geometry is the main work, so draw each picture on paper before you run the code.

**By the end of this day, you will have:**

- A geometric account of hyperplanes, margins and support vectors, with a drawn example
- A linear support vector classifier trained with scikit-learn, with its margin and support vectors inspected
- A demonstration of the kernel idea on data that no straight line can separate, using an RBF kernel
- A tuned model with a grid over `C` and `gamma`, scaled inputs, and a written note on when an SVM is the wrong choice

**What you need:**

- Python 3.11 or newer
- NumPy 2.0 or newer (`pip install numpy`), BSD 3-Clause
- scikit-learn 1.5 or newer (`pip install scikit-learn`), BSD 3-Clause
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the margin plots
- pytest 8.0 or newer (`pip install pytest`)
- Git and a free GitHub account for the project section
- Paper and a pencil for the hyperplane sketches

**Prerequisites:**

- Day 8: dot products, norms and projections
- Day 24: scaling inside a pipeline, and cross-validation
- Day 25 or Day 26: a linear decision rule of the form `w^T x + b`
- Day 27: why feature scaling changes distance-based methods, since SVMs depend on distances too

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Hyperplanes](#hyperplanes)
- [Margins](#margins)
- [Support vectors](#support-vectors)
- [Kernels concept](#kernels-concept)
- [Linear vs nonlinear separation](#linear-vs-nonlinear-separation)
- [Hands-on exercises](#hands-on-exercises)
- [Project: Support vector classifier with a kernel study](#project-support-vector-classifier-with-a-kernel-study)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A support vector machine is a classifier with a clear geometric goal. Among all the straight lines that separate two groups of points, it picks the one that keeps the widest empty street between the groups. The points that touch the edges of that street determine the answer, and the rest can be removed without changing the boundary.

That idea made SVMs one of the most studied and widely used classifiers of the 1990s and 2000s. They were the standard tool for text classification, image recognition with hand-crafted features and bioinformatics before deep learning became dominant. They remain useful for small and medium datasets with many features, where they train quickly and resist overfitting when tuned well.

On the job, you are unlikely to ship a kernel SVM on millions of rows, because its training cost grows quickly with the number of samples. You will use one for a high-dimensional problem with a few thousand examples, such as a gene-expression panel or a set of engineered text features, and you will compare it with logistic regression as a sanity check. Knowing the geometry tells you when that comparison is worth running.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1963 | Vladimir Vapnik and Alexey Chervonenkis develop the generalized portrait method for finding optimal separating hyperplanes. | The geometric core of the SVM is older than the kernel extension. |
| 1964 | Aizerman, Braverman and Rozonoer publish work on the potential function method, which contains the idea behind kernels. | The kernel trick has roots in this early pattern recognition work. |
| 1992 | Bernhard Boser, Isabelle Guyon and Vapnik present a method for maximum-margin classifiers with kernels. | The modern kernel SVM is usually dated to this paper. |
| 1995 | Corinna Cortes and Vapnik publish "Support-Vector Networks", which introduces the soft margin for data that is not perfectly separable. | The soft-margin parameter `C` you tune today comes from this step. |
| 1998 to 2010s | SVMs become standard in text classification, bioinformatics and computer vision before deep learning took over. | The methods are still used for small, high-dimensional problems. |

**Takeaway:** the SVM combines a clean geometric idea with a practical kernel trick, and its key ideas, the margin and the soft penalty, shape how most modern linear models are trained.

---

## Glossary

- **Hyperplane**: in `d` dimensions, a flat surface of dimension `d - 1` that splits space into two sides. In two dimensions it is a line.
- **Separating hyperplane**: a hyperplane that places every training example of one class on one side and every example of the other class on the other side.
- **Margin**: the distance from the hyperplane to the nearest training point, measured on either side.
- **Maximum-margin hyperplane**: the separating hyperplane with the largest margin.
- **Support vector**: a training point that lies exactly on the edge of the margin, or on the wrong side of it in the soft-margin case. These points determine the hyperplane.
- **Soft margin**: a formulation that allows some points to violate the margin, at a cost set by the parameter `C`.
- **Hinge loss**: the loss `max(0, 1 - y f(x))`, which is zero for points beyond the margin and grows linearly for points that violate it.
- **Kernel**: a function `K(x, z)` that computes the inner product of two points after an implicit transformation to a higher-dimensional space.
- **Kernel trick**: using a kernel in place of an explicit transformation, so the model works in a large feature space without building it.
- **RBF (radial basis function) kernel**: `K(x, z) = exp(-gamma ||x - z||^2)`, which measures similarity by distance.
- **Gamma**: the parameter of the RBF kernel that sets how far the influence of one training point reaches.

---

### Hyperplanes

Think of a line drawn on a sheet of paper that separates red dots from blue dots. The line is the boundary, and every point lies on one side or the other. In three dimensions the boundary is a flat sheet. The general name for these flat boundaries is a hyperplane.

From first principles, a hyperplane in `d` dimensions is the set of points `x` where `w^T x + b = 0`. The vector `w` is perpendicular to the hyperplane, and `b` shifts it away from the origin. A point is classified by the sign of `w^T x + b`. The distance from a point `x` to the hyperplane is `|w^T x + b| / ||w||`. Scaling `w` and `b` by the same number gives the same hyperplane, so the SVM fixes the scale by requiring that the closest points satisfy `|w^T x + b| = 1`.

```python
import numpy as np

w = np.array([2.0, 1.0])
b = -4.0
points = np.array([[3.0, 0.0], [1.0, 1.0], [0.0, 4.0]])
scores = points @ w + b
distances = np.abs(scores) / np.linalg.norm(w)
print("signs:", np.sign(scores))           # [1 -1 1]
print("distances:", distances.round(3))
```

The mistake is treating `w` as a unit vector without checking. The sign of the score tells you the side, but the size of the score depends on the scale of `w`, so distances must divide by `||w||`. The limitation of a hyperplane is that it can only separate data in a linear way. When the classes are arranged in a curve or in concentric rings, no hyperplane in the original space works. In the market, a linear SVM is still a reasonable first model for sparse text features, because word counts are high-dimensional and often nearly separable.

### Margins

Picture two groups of people standing on a field, and a rope laid between them. The best rope position leaves the most room on both sides before anyone touches it. That room is the margin. A wide margin is a safer boundary, because a small shift in a point is less likely to cross it.

From first principles, for a separating hyperplane the margin is twice the distance from the hyperplane to the nearest point on either side, which is `2 / ||w||` once the scale is fixed. Maximizing this margin is the same as minimizing `||w||^2` subject to each point being on the correct side with a score of at least 1 in absolute value. This is a quadratic optimization problem with a unique solution for separable data. The wider the margin, the more the classifier is pushed away from individual points, which is the source of the SVM's resistance to overfitting in high dimensions.

```python
import numpy as np
from sklearn.svm import LinearSVC

rng = np.random.default_rng(0)
X = np.vstack([rng.normal([-2, -2], 0.5, (50, 2)), rng.normal([2, 2], 0.5, (50, 2))])
y = np.array([0] * 50 + [1] * 50)

svm = LinearSVC(C=1.0, max_iter=10000).fit(X, y)
w = svm.coef_.ravel()
print("margin width", round(2 / np.linalg.norm(w), 3))
```

The mistake is treating a large margin as proof of good generalization on its own. The margin is a useful guide, but the quality of the features and the choice of `C` matter just as much. The limitation of the hard margin is that it fails completely on any data that is not separable, which real data almost never is. The soft margin, covered in the next subsections, fixes that. In industry, teams that use SVMs for feature-rich problems often report the margin as a diagnostic, but they select the model using cross-validation.

### Support vectors

Imagine building a fence along a row of houses. Only the houses closest to the road determine where the fence goes. You could remove the houses in the middle of the street and the fence would stay in the same place. The closest houses are the support vectors.

From first principles, the solution of the SVM optimization depends only on the points whose constraints are active, that is, the points lying exactly on the margin, plus any points that violate it in the soft-margin case. These points have nonzero Lagrange multipliers in the dual problem. Every other training point has a multiplier of zero and does not affect the boundary. In scikit-learn, the fitted model exposes these points through `support_vectors_`, and the number of them is a useful diagnostic of how much of the data the model actually uses.

```python
import numpy as np
from sklearn.svm import SVC

rng = np.random.default_rng(0)
X = np.vstack([rng.normal([-2, -2], 0.7, (100, 2)), rng.normal([2, 2], 0.7, (100, 2))])
y = np.array([0] * 100 + [1] * 100)

clf = SVC(kernel="linear", C=1.0).fit(X, y)
print("support vectors:", clf.support_vectors_.shape[0], "of", X.shape[0])
```

The mistake is reading a small number of support vectors as a sign of a simple model, when it may also mean the boundary is pinned by a few noisy points. Look at the support vectors on a plot, and check whether they are mislabeled. The limitation is that the support vectors are only the points that matter for the boundary, so the model says nothing about the rest of the data distribution. In the market, support vectors are sometimes used to find the borderline cases in a dataset, which is a practical way to spend labeling effort.

### Kernels concept

Suppose you have points on a line, with the blue points in the middle and the red points on both sides. No single cut on the line separates them. Now lift the points into a plane by adding a second coordinate equal to the square of the first. In that plane, a straight line separates them. The kernel trick does this lift without computing the new coordinates explicitly.

From first principles, the SVM solution depends on the data only through inner products `x_i^T x_j`. Replacing each inner product with a kernel `K(x_i, x_j)` is equivalent to an inner product in a transformed space `phi(x)`, where `K(x, z) = phi(x)^T phi(z)`. The RBF kernel corresponds to an infinite-dimensional space, and it is computed in closed form from the distance between two points. The parameter `gamma` controls how quickly similarity decays with distance: a large gamma makes the boundary wiggle around single points, and a small gamma makes it smooth. You never compute `phi(x)`, which is why the method remains practical.

```python
import numpy as np

def rbf_kernel(x, z, gamma=0.5):
    return np.exp(-gamma * np.sum((x - z) ** 2))

x = np.array([1.0, 2.0])
z = np.array([1.5, 2.0])
print(round(rbf_kernel(x, x), 3))  # 1.0, a point is maximally similar to itself
print(round(rbf_kernel(x, z), 3))  # about 0.882
```

The mistake is choosing a kernel without scaling the features, because the RBF kernel depends on distances, which depend on units. The limitation of kernels is cost: kernel SVMs compute an `n` by `n` matrix of similarities, so memory and training time grow quickly with the number of rows. For a dataset with tens of thousands of rows, a kernel SVM is often too slow, and an approximate method or a different model is used instead.

### Linear vs nonlinear separation

Compare two problems. In one, two clusters sit on opposite sides of a plane, and a line separates them easily. In the other, one cluster is a disc and the other is a ring around it. No line separates them. A linear SVM fails on the second problem, and a kernel SVM can succeed by bending the boundary.

From first principles, a dataset is linearly separable when some hyperplane puts each class on its own side. Linear separability is a property of the data and the chosen features, not of the algorithm. When it fails, you have three options: add nonlinear features by hand, use a kernel that implicitly does this, or use a model that learns nonlinear boundaries directly, such as a tree or a neural network. The choice depends on the size of the data and on how much you need to explain the boundary.

```python
import numpy as np
from sklearn.svm import SVC
from sklearn.model_selection import cross_val_score
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

rng = np.random.default_rng(0)
r_inner = rng.uniform(0, 1, 150)
r_outer = rng.uniform(2, 3, 150)
angles = rng.uniform(0, 2 * np.pi, 300)
radii = np.concatenate([r_inner, r_outer])
X = np.column_stack([radii * np.cos(angles), radii * np.sin(angles)])
y = np.array([0] * 150 + [1] * 150)

for kernel in ("linear", "rbf"):
    model = make_pipeline(StandardScaler(), SVC(kernel=kernel, C=1.0, gamma="scale"))
    print(kernel, round(cross_val_score(model, X, y, cv=5).mean(), 3))
```

The linear kernel scores near chance on this ring problem, and the RBF kernel scores close to 1. The mistake is trying a kernel on every problem without first asking whether the data is nonlinear. The limitation is that the kernel makes the model less interpretable, and it may overfit if `gamma` is too large. In practice, a standard first step is to compare the linear and RBF kernels with cross-validation, the same comparison this lesson makes.

---

## Hands-on exercises

1. On graph paper, draw two clusters of five points each, then draw a separating line by eye and mark the margin. Estimate the margin and the support vectors, and explain why moving a non-support point does not change the boundary.
2. For `w = (3, 4)` and `b = -10`, compute the distance from the origin and from the point `(2, 2)` to the hyperplane. Verify your answers in NumPy.
3. Train `LinearSVC` on two Gaussian blobs with two values of `C`, 0.01 and 100. Plot both boundaries and describe what changes.
4. Implement the RBF kernel from scratch and compare a kernel matrix computed with your function to `sklearn.metrics.pairwise.rbf_kernel` on five random points.
5. Build the ring dataset from this lesson and plot the decision regions for the linear and RBF kernels.
6. Run a grid search over `C` in {0.1, 1, 10} and `gamma` in {0.01, 0.1, 1} for the RBF kernel with scaled features, using 5-fold cross-validation. Report the best pair and the spread of scores.
7. Remove all points except the support vectors, retrain the model, and confirm that the boundary is essentially unchanged. Explain the result using the dual formulation.

---

## Project: Support vector classifier with a kernel study

This is a medium-weight project. The model is a standard library call, so the deliverable is the study: how the margin, the kernel and the hyperparameters change the boundary and the score.

**Repo name:** `100-days-ai-engineering-day-31-support-vector-machines`

**Goal:** compare linear and RBF support vector classifiers on a dataset with a nonlinear boundary, tune them with cross-validation, and report the boundary and the support vector counts.

**Repo layout:**

```
100-days-ai-engineering-day-31-support-vector-machines/
├── README.md
├── requirements.txt
├── data.py
├── study.py
├── results/
│   ├── boundaries.png
│   └── grid_search.md
└── tests/
    └── test_data.py
```

**Starter code for `data.py`:**

```python
import numpy as np


def make_rings(n_per_class=200, seed=0):
    """Return (X, y) for two concentric rings with a little noise. X has shape (2 n_per_class, 2)."""
    raise NotImplementedError


def make_split(X, y, seed=0):
    """Return X_train, X_val, X_test, y_train, y_val, y_test using stratified 60/20/20 splits."""
    raise NotImplementedError
```

**Starter code for `tests/test_data.py`:**

```python
import sys
from pathlib import Path

import numpy as np

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from data import make_rings, make_split


def test_rings_have_two_classes_and_two_features():
    X, y = make_rings(n_per_class=100)
    assert X.shape == (200, 2)
    assert set(np.unique(y)) == {0, 1}


def test_split_sizes_are_60_20_20():
    X, y = make_rings(n_per_class=100)
    X_tr, X_va, X_te, y_tr, y_va, y_te = make_split(X, y)
    assert len(X_tr) == 120
    assert len(X_va) == 40
    assert len(X_te) == 40
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `scikit-learn`, `matplotlib` and `pytest`.
2. Implement `make_rings` using a radius and an angle for each point, so the two classes are separated by radius. Implement `make_split` with two calls to `train_test_split`. Make the tests pass.
3. In `study.py`, scale the features with a pipeline and fit a linear SVC and an RBF SVC on the training part.
4. Run a grid search over `C` and `gamma` for the RBF model using only the training part with cross-validation. Save the grid results to `results/grid_search.md`.
5. Select the best model on the validation part. Score the test part once, and record the number and the support vector count.
6. Plot the decision boundaries of the linear and RBF models side by side and save the figure to `results/boundaries.png`.
7. Write the README. Explain why the linear model fails on this data, what `gamma` controls, and what the support vector count tells you about the boundary.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is stated in one sentence
- [ ] Install and run commands are listed, including `pytest` and `python study.py`
- [ ] The grid search table is included and the best pair is named
- [ ] The boundary figure is embedded with a caption that compares the two models
- [ ] The README states that the test set was used once
- [ ] The README names one situation where an RBF SVM would be too slow

**Done when:**

- `pytest` passes both tests
- The RBF model clearly outperforms the linear model on the test part, and the README explains why in terms of the data shape
- The grid search table covers at least nine pairs of `C` and `gamma`
- You can explain in one paragraph what happens to the boundary as `gamma` grows large

**What to commit as evidence:** the source files, the tests, the grid search results, the boundary figure, and the README with the single test score and the support vector count.

---

## Check your understanding

1. Explain why maximizing the margin is the same as minimizing `||w||`, and why the scale of `w` must be fixed to make this statement meaningful.
2. Your SVM has 400 support vectors out of 1,000 training rows. What does that suggest about the boundary, and what would you check next?
3. Explain what the parameter `C` controls in a soft-margin SVM, and what happens to the margin when `C` is very large.
4. A colleague forgets to scale the features before using an RBF kernel. Predict what happens to the kernel values and to the boundary, and explain why.
5. Explain in one paragraph how the kernel trick lets an SVM work in a high-dimensional space without computing the features of that space.
6. Your RBF SVM with a large `gamma` scores 100 percent on training data and 65 percent on validation data. Diagnose the problem and name the parameter change that reduces it.
7. A team with 200,000 rows wants an RBF SVM for a churn model. Explain the main cost that makes this a poor choice, and suggest two alternatives that respect the same nonlinear structure.

### Answers

1. The distance from a separating hyperplane to its nearest point is `|w^T x + b| / ||w||`. If the closest points are scaled so that `|w^T x + b| = 1`, the distance is `1 / ||w||`, so the margin width is `2 / ||w||`. Maximizing the margin means minimizing `||w||`. The scale must be fixed, because multiplying `w` and `b` by 10 gives the same hyperplane but a different score, so the margin would be undefined without a fixed scale.
2. A large fraction of the rows lie on or inside the margin, so the boundary is determined by many points rather than a few clear ones. This may indicate overlapping classes or a too-small `C`. The next check is to plot the support vectors and to look at the validation score for different values of `C`.
3. `C` sets the penalty for margin violations. A small `C` allows many violations and gives a wider margin, while a large `C` penalizes violations heavily and gives a narrower margin that fits the training points more closely. With very large `C`, the model approaches a hard margin, which may overfit.
4. Without scaling, a feature with a large range dominates the squared distance, so the kernel value depends almost entirely on that feature. The boundary then ignores the other features, and the model performs poorly. Scaling puts the features on a common footing.
5. The solution depends on the data only through inner products. A kernel computes the inner product in a transformed space directly from the original points, so the transformed coordinates never need to be built. For the RBF kernel, that space is infinite-dimensional, and the kernel value is a simple function of the distance.
6. The large `gamma` makes each training point's influence very narrow, so the boundary wraps around individual points and memorizes noise. Reducing `gamma`, and tuning `C` by cross-validation, reduces the variance.
7. The main cost is the kernel matrix, which has `n` squared entries, so memory and training time grow very quickly with 200,000 rows. Two alternatives are gradient boosted trees, which capture nonlinear structure efficiently on tabular data, and a kernel approximation such as random Fourier features combined with a linear model.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Support vector machine](https://en.wikipedia.org/wiki/Support_vector_machine)
- Wikipedia: [Hyperplane](https://en.wikipedia.org/wiki/Hyperplane)
- Wikipedia: [Kernel method](https://en.wikipedia.org/wiki/Kernel_method)
- Wikipedia: [Hinge loss](https://en.wikipedia.org/wiki/Hinge_loss)
- Stanford CS229 lecture notes by Andrew Ng, the sections on the optimal margin classifier and kernels. Search "CS229 lecture notes Andrew Ng support vector machines".
- scikit-learn user guide, "Support Vector Machines", at scikit-learn.org under the stable documentation, including the section on kernel functions.

**Tutorials**

- scikit-learn documentation, `SVC`, `LinearSVC` and the example "Non-linear SVM with RBF kernel" from the gallery.
- GeeksforGeeks, search "support vector machine in machine learning" for a worked example with a plotted margin.

**Watch**

- StatQuest with Josh Starmer (YouTube), "Support Vector Machines Part 1: Main Ideas" and "Support Vector Machines Part 2: The Radial (RBF) Kernel".
- MIT OpenCourseWare, 6.034 "Artificial Intelligence" by Patrick Winston, the lecture on support vector machines.

**Practice**

- Kaggle Learn, the "Intermediate Machine Learning" course, and a notebook that compares an SVM with a tree on the same dataset.

---

## Tomorrow

Tomorrow you stop predicting labels and start looking for structure without them. Clustering groups rows that are similar, with no answer key. You will use k-means, compute centroids and distances, test different initializations, and meet hierarchical clustering and DBSCAN. The distances from Day 27 and the centroid idea from today's geometry come back in full.

**Next:** [Day 32: Clustering](Day-32-Clustering.md)

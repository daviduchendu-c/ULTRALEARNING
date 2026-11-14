# Day 10: Linear Algebra for ML

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3.5 hours
**Level:** intermediate. You need Days 8 and 9 without hesitation. Today's ideas are more abstract, so take the pictures slowly.

**By the end of this day, you will have:**

- A working understanding of linear transformations, bases, rank, orthogonality, eigenvalues, the singular value decomposition and dimensionality reduction
- A Python implementation of principal component analysis (PCA) built on NumPy's singular value decomposition, tested against known properties
- A short written analysis of how many components a real dataset needs and what the discarded directions cost

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 2.0 or newer (`pip install numpy`), free and open source under the BSD license
- pytest 8.0 or newer (`pip install pytest`)
- Matplotlib 3.8 or newer (`pip install matplotlib`), for the optional 2D plot in the project
- A code editor such as VS Code (free) or any editor you already use
- Git and a free GitHub account for the project section
- Paper and graph paper for the sketches in the exercises

**Prerequisites:**

- Day 8: vectors, the dot product, magnitude and projection
- Day 9: matrix multiplication, shapes, transpose, identity and inverses
- Day 7: summation notation and the idea of a function
- Basic statistics: the mean and what it means for values to vary

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Linear transformations](#linear-transformations)
- [Basis](#basis)
- [Rank](#rank)
- [Orthogonality](#orthogonality)
- [Eigenvalues/eigenvectors](#eigenvalueseigenvectors)
- [Singular Value Decomposition concept](#singular-value-decomposition-concept)
- [Dimensionality reduction intuition](#dimensionality-reduction-intuition)
- [Hands-on exercises](#hands-on-exercises)
- [Project: PCA from scratch with NumPy](#project-pca-from-scratch-with-numpy)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Yesterday you treated a matrix as a table and a set of operations on that table. Today you treat it as a machine that moves points. Feed a vector into a matrix and out comes a new vector, which may be rotated, stretched, flipped or squashed onto a line. That change of view is the core of linear algebra, and it is what explains almost every technique in classical machine learning and the layers of deep networks.

The practical payoff is large. A dataset with 500 features may really live close to a space of 20 dimensions, because many features move together. Finding those directions lets you compress data, speed up training, remove redundancy and draw pictures of data that has too many columns to plot. The tools that do this, principal component analysis and the singular value decomposition, are built from the ideas in this lesson: rank, orthogonal bases and eigenvalues.

On the job, you will use these ideas in feature engineering before a clustering model, in choosing how many dimensions to keep for an embedding index, in diagnosing collinearity in a regression, and in understanding why a parameter-efficient fine-tuning method can change a large model by adding small low-rank matrices. None of these require you to compute an eigenvalue by hand. They do require knowing what the number means when a library hands it to you.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1873 to 1874 | Eugenio Beltrami and Camille Jordan independently develop the singular value decomposition for real bilinear forms. | The factorization you use today dates from the same decade as the first matrix theory. |
| 1901 | Karl Pearson publishes a method for finding the lines and planes of closest fit to a set of points. | The earliest form of principal component analysis. |
| 1933 | Harold Hotelling develops principal components as a way to explain correlated variables, and the name catches on. | The method that statisticians and data scientists use under the name PCA. |
| 1936 | Carl Eckart and Gale Young prove that the best low-rank approximation of a matrix comes from its SVD. | The theorem that justifies compressing a matrix by keeping its largest singular values. |
| 1998 | Lawrence Page, Sergey Brin and colleagues describe PageRank, which ranks web pages using the dominant eigenvector of a link matrix. | An eigenvector problem that powered a search engine at scale. |

**Takeaway:** the tools you learn today are older than the computers that run them. Their age is an advantage, because the libraries are mature and well tested.

---

## Glossary

- **Linear transformation**: a function on vectors that preserves addition and scalar multiplication. Every linear transformation of finite-dimensional vectors is a matrix.
- **Linear combination**: a sum of vectors, each multiplied by a scalar, such as `2u - 3v`.
- **Span**: the set of all linear combinations of a collection of vectors.
- **Linear independence**: a set of vectors is independent when no vector in it can be written as a combination of the others.
- **Basis**: a set of linearly independent vectors that spans the whole space.
- **Rank**: the number of linearly independent columns of a matrix, which equals the dimension of the space its outputs can reach.
- **Orthogonal**: perpendicular. Two vectors are orthogonal when their dot product is zero.
- **Orthonormal**: orthogonal and each of length 1.
- **Eigenvector**: a nonzero vector that a matrix only stretches or flips, without changing its direction.
- **Eigenvalue**: the factor by which the matrix stretches an eigenvector.
- **Singular value**: a nonnegative number that measures how much a matrix stretches space along one of its principal directions.
- **Singular value decomposition (SVD)**: the factorization of any matrix into an orthogonal matrix, a diagonal matrix of singular values and another orthogonal matrix.
- **Variance**: the average squared distance of values from their mean. It measures spread.
- **Covariance matrix**: a square matrix whose entries are the covariances between pairs of features.
- **Principal component**: a direction in feature space along which the data varies the most, subject to being orthogonal to the earlier directions.
- **Dimensionality reduction**: representing data with fewer features while keeping as much of its structure as possible.
- **Low-rank approximation**: a matrix that has fewer independent directions than the original and is close to it in a chosen measure.

---

### Linear transformations

Imagine a sheet of graph paper printed on rubber. You can stretch it, rotate it, flip it, or pull it so the grid lines lean. Whatever you do, the grid lines stay straight and the origin stays where it is. A linear transformation is any operation that behaves this way on space.

A function `T` is linear when `T(u + v) = T(u) + T(v)` and `T(c v) = c T(v)` for all vectors and scalars. Every such function on a finite-dimensional space is multiplication by a matrix `A`. The columns of `A` tell you where the standard basis vectors land. A rotation by 90 degrees counterclockwise, for example, sends `(1, 0)` to `(0, 1)` and `(0, 1)` to `(-1, 0)`, so its matrix has those images as its columns.

```python
import numpy as np

R = np.array([[0.0, -1.0],
              [1.0,  0.0]])   # rotate 90 degrees counterclockwise
S = np.diag([2.0, 1.0])       # stretch x by 2, leave y alone

print(R @ np.array([1.0, 0.0]))  # [0. 1.]
print(S @ np.array([3.0, 3.0]))  # [6. 3.]
```

One limitation is that many useful functions are not linear. A bias term makes the map affine, not linear, because `T(0)` is no longer zero. ReLU, which sets negative values to zero, is not linear either, since `ReLU(-1 + 2)` is not `ReLU(-1) + ReLU(2)`. A neural network is therefore a sequence of linear transformations interleaved with nonlinear functions. Without the nonlinearities, a deep network would collapse to a single matrix, which is why the activation function matters.

Treating an affine layer as linear is a common mistake when you reason about it, for example when you assume that scaling the input scales the output. The fix is to check whether the transformation sends zero to zero. In industry, a dense layer in PyTorch computes a linear transformation plus a bias, followed by an activation, and the matrix is the part that carries the learned geometry.

### Basis

A basis is a set of building blocks for a space. In a city, north and east are a basis for position: every location is some number of blocks north plus some number of blocks east. The building blocks must be independent, since a third direction that is just a mix of north and east adds nothing.

From first principles, a set of vectors is a basis for a space when it is linearly independent and spans the space. The standard basis for `R^n` consists of the unit vectors along each axis, such as `(1, 0)` and `(0, 1)` in the plane. Every vector in the plane has a unique expression in any basis. The coordinates of a vector in a new basis are the solution to a linear system, which you solved on Day 9. Changing basis is a change of description, not a change of the vector itself.

```python
import numpy as np

B = np.array([[1.0, 1.0],
              [0.0, 1.0]])           # columns (1, 0) and (1, 1) form a basis
v = np.array([3.0, 1.0])
coords = np.linalg.solve(B, v)       # coordinates of v in the new basis
print(coords)                        # [2. 1.]
print(B @ coords)                    # [3. 1.], back to the original vector
```

A common mistake is assuming that the standard axes are the most meaningful basis for your data. The axes are the ones you were given, which may be arbitrary. For a dataset of house sizes and prices, the directions in which prices vary together may be far more informative than the raw axes. Finding a better basis is what PCA does. Another mistake is treating a set of vectors as a basis when it is only a spanning set, containing redundant vectors. Redundancy is measured by rank, which is the next concept. In the real market, a feature set with redundant columns is a spanning set, and a feature engineer removes the redundancy before a model is trained.

### Rank

Suppose you have three ingredients, but one of them is always a mix of the other two. You have only two independent ingredients, however many you list. Rank counts the independent directions a matrix actually uses.

The rank of a matrix is the maximum number of linearly independent columns, which equals the maximum number of linearly independent rows. It is the dimension of the column space, the set of all possible outputs `A x`. A matrix with `m` rows and `n` columns has rank at most `min(m, n)`. A matrix is full rank when its rank equals that maximum. For a square matrix, full rank is the same as being invertible, which connects this concept to Day 9.

```python
import numpy as np

A = np.array([[1.0, 2.0],
              [2.0, 4.0]])      # the second row is twice the first
print(np.linalg.matrix_rank(A))  # 1

I = np.eye(3)
print(np.linalg.matrix_rank(I))  # 3
```

The matrix `A` maps the whole plane onto a single line, because its columns point in the same direction. Every output lies along `(1, 2)`, so a whole dimension of information is lost. The mistake that follows is treating rank as an exact property you can read off. In floating-point arithmetic, a matrix can be almost rank deficient, with a tiny singular value that is not exactly zero. `np.linalg.matrix_rank` uses a tolerance to decide, and the tolerance is a judgment call. In industry, the rank of a feature matrix is how you detect collinearity in a regression, and low-rank structure is the reason LoRA, the low-rank adaptation method for fine-tuning large models, can learn useful changes with far fewer parameters than the full weight matrix.

### Orthogonality

A carpenter's square is a useful picture. Its two edges meet at a right angle, and each edge is perpendicular to the other. In a space, orthogonal vectors are the directions that do not overlap at all. Moving along one gives no progress along the other.

From first principles, two vectors are orthogonal when their dot product is zero. A set of orthogonal vectors, each scaled to length 1, is orthonormal. A matrix `Q` whose columns are orthonormal satisfies `Q^T Q = I`, which means `Q^T` is its inverse. Multiplying by an orthogonal matrix preserves lengths and angles. Rotations and reflections are orthogonal transformations. That is why an orthogonal basis is so convenient: the coordinates of a vector in it are just dot products with the basis vectors, with no linear system to solve.

```python
import numpy as np

theta = np.pi / 4
Q = np.array([[np.cos(theta), -np.sin(theta)],
              [np.sin(theta),  np.cos(theta)]])
print(np.allclose(Q.T @ Q, np.eye(2)))   # True
v = np.array([3.0, 4.0])
print(np.linalg.norm(v), np.linalg.norm(Q @ v))  # both 5.0, lengths preserved
```

A common mistake is equating orthogonal with independent in the statistical sense. Two features are uncorrelated when their covariance is zero, which is the statistical analogue of orthogonality after the data is centered. Uncorrelated does not mean independent. Two variables can be uncorrelated and still depend on each other in a nonlinear way, for example `y = x^2` with `x` symmetric around zero. The second mistake is assuming that an orthogonal basis is always the best one. It is convenient, but it is only useful if the directions matter for your problem. In industry, the QR decomposition uses orthogonal matrices to solve least-squares problems stably, and PCA produces orthogonal components.

### Eigenvalues/eigenvectors

Most vectors change direction when a matrix acts on them. A few special vectors do not. They are only stretched, or flipped, and they keep their line through the origin. These special vectors are the eigenvectors, and the stretch factors are the eigenvalues.

An eigenvector of a square matrix `A` is a nonzero vector `v` with `A v = lambda v` for some scalar `lambda`, the eigenvalue. Rewriting the equation as `(A - lambda I) v = 0` shows that `lambda` must make the matrix `A - lambda I` singular, so its determinant is zero. That condition gives the eigenvalues. For a symmetric matrix, the eigenvalues are real and the eigenvectors can be chosen orthonormal. That fact, called the spectral theorem, is the reason covariance matrices behave so nicely in PCA.

Consider the symmetric matrix `[[2, 1], [1, 2]]`. The vector `(1, 1)` is an eigenvector with eigenvalue 3, and `(1, -1)` is an eigenvector with eigenvalue 1.

```python
import numpy as np

A = np.array([[2.0, 1.0],
              [1.0, 2.0]])
vals, vecs = np.linalg.eigh(A)    # eigh is for symmetric matrices, values ascending
print(vals)                        # [1. 3.]
print(vecs)                        # columns are unit eigenvectors
print(np.allclose(A @ vecs[:, 1], vals[1] * vecs[:, 1]))  # True
```

A common mistake is assuming eigenvectors are unique. An eigenvector can be scaled by any nonzero number and still be an eigenvector, and its sign is arbitrary. Libraries return unit vectors, but the sign may change between versions and platforms. Compare subspaces or results, not signs. Another mistake is using `np.linalg.eig` on a non-symmetric matrix and expecting real results. A 90-degree rotation matrix has complex eigenvalues, because no real direction stays on its line. The limitation is that eigen-decomposition needs a square matrix, and for a non-square dataset you need the singular value decomposition instead. In the real market, PageRank computes the dominant eigenvector of a matrix of links between web pages, and PCA computes the eigenvectors of a covariance matrix, which is how you find the directions of greatest variance.

### Singular Value Decomposition concept

Any photograph can be described by a grid of pixel values. A rectangular grid has no eigenvectors in the usual sense, because the matrix is not square. The singular value decomposition solves that problem for every matrix. It breaks any matrix into three pieces: a rotation, a stretch along the axes, and another rotation.

Every real matrix `A` with shape `(m, n)` can be written as `A = U S V^T`. The matrix `U` has orthonormal columns with shape `(m, k)`, `V` has orthonormal columns with shape `(n, k)`, and `S` is a diagonal matrix of nonnegative singular values, sorted from largest to smallest, where `k` is the smaller of `m` and `n`. The columns of `V` are the input directions that the matrix stretches the most, and the singular values say how much. The singular values are the square roots of the eigenvalues of `A^T A`, which connects the SVD to the eigen-decomposition you just learned.

```python
import numpy as np

A = np.array([[2.0, 0.0],
              [0.0, 1.0],
              [-2.0, 0.0],
              [0.0, -1.0]])
U, s, Vt = np.linalg.svd(A, full_matrices=False)
print(np.round(s, 3))                      # [2.828 1.414]
print(np.allclose(U @ np.diag(s) @ Vt, A))  # True
```

The most useful consequence is the low-rank approximation. If you keep only the largest `r` singular values and their vectors, you get the best rank-`r` approximation of `A` in the least-squares sense. That is the Eckart-Young theorem from the history table. It is why image compression works and why recommender systems can represent a huge user-by-item table with a few hundred factors.

Treating the singular values as eigenvalues of `A` is a frequent mistake. They are not, unless `A` is symmetric and positive semidefinite. Another mistake is reading the signs of the singular vectors as meaningful. The pair `U` and `V` can be flipped together without changing the product. In the market, latent semantic analysis, a classic text technique, applies the SVD to a term-by-document matrix, and LoRA factorizes weight updates into products of low-rank matrices, which is an SVD-shaped idea applied to training.

### Dimensionality reduction intuition

Imagine a set of points on a long, thin cloud, tilted at 45 degrees. Two axes describe each point, but one axis captures almost all of the spread. You can describe most of the cloud with one number, the position along the long direction, and lose only the small wiggle across it. Dimensionality reduction makes that trade on purpose.

PCA looks for the direction along which the centered data has the largest variance. It then looks for the next direction with the largest remaining variance, subject to being orthogonal to the first, and so on. The variance along a unit direction `w` is `w^T C w`, where `C` is the covariance matrix. The maximizing directions are the eigenvectors of `C`, and their eigenvalues are the variances. The SVD of the centered data gives the same directions without forming `C`, which is faster and more stable. Keeping the first `k` components gives a `k`-dimensional description, and the fraction of total variance kept is the sum of the top `k` eigenvalues divided by the total.

The limitations are real. PCA is linear, so it cannot unfold a curved structure such as a spiral. It assumes that high variance means important, which is often but not always true. Features measured on different scales dominate the result unless you standardize them first. And discarded directions discard information, which may matter for a downstream task. Compressing a dataset to 2 dimensions for a plot is useful, but a classifier trained on that compression may lose accuracy in ways the plot does not show.

Running PCA on data that was not centered is a frequent mistake. The first component then points toward the mean rather than toward the spread. Center the data first, and standardize when the units differ. In industry, PCA is used before clustering to reduce noise, to speed up nearest-neighbor search, to visualize embeddings and to diagnose which features carry most of the variation in a sensor or survey dataset.

---

## Hands-on exercises

Do the first four by hand or on graph paper, then check each one in Python.

1. Sketch the rotation by 45 degrees and the stretch `diag(3, 1)` on graph paper, and apply each to the unit square's corner `(1, 1)`. Write the output of each by hand, then verify with NumPy.
2. Find the rank of `[[1, 2, 3], [2, 4, 6], [1, 0, 1]]` by hand. Explain how you can tell the answer from the first two rows, then verify with `np.linalg.matrix_rank`.
3. Check that `(1, 1)` and `(1, -1)` are orthogonal, normalize them to unit length, and verify that the matrix with these as columns satisfies `Q^T Q = I`.
4. For `A = [[3, 1], [1, 3]]`, find the eigenvalues by solving `det(A - lambda I) = 0` by hand, then find the eigenvectors. Verify with `np.linalg.eigh`, and explain why the eigenvalues are 4 and 2.
5. Compute the SVD of `[[3, 0], [0, 1]]` by hand. Then compute the best rank-1 approximation of `[[3, 1], [1, 3]]` using only the largest singular value and its vectors, and measure the error with `np.linalg.norm`.
6. Generate 200 points with `rng = np.random.default_rng(0)`, `X = rng.standard_normal((200, 2)) @ np.array([[2.0, 0.0], [1.0, 0.5]])`. Center the data, compute the covariance matrix with `np.cov(X.T)`, and find its eigenvectors. Compare the eigenvector with the largest eigenvalue (the last column returned by `eigh`) with the first row of `Vt` from `np.linalg.svd` on the centered data, and explain any sign difference.
7. Use the same data to compute the fraction of variance captured by the first component. Explain in one sentence what the number means for a team that wants to keep only one feature.

Record each place where your hand calculation and the code disagreed, along with the reason. Sign differences and rounding are normal, and knowing when they are harmless is part of the skill.

---

## Project: PCA from scratch with NumPy

This is a medium-weight project. Principal component analysis is short to write once you understand it, and the value is in testing the properties and writing a clear analysis of what it does to a dataset. The project is lighter than the Day 8 and Day 9 projects because the code is a single class, and the main deliverable is the written analysis.

**Repo name:** `100-days-ai-engineering-day-10-linear-algebra-for-ml`

**Goal:** implement PCA with NumPy's SVD, check that its components are orthonormal, measure how much reconstruction error each extra component removes, and explain the result in the README.

**Repo layout:**

```
100-days-ai-engineering-day-10-linear-algebra-for-ml/
├── README.md
├── requirements.txt
├── pca.py
├── analysis.py
├── results/
│   └── pca_scatter.png
└── tests/
    └── test_pca.py
```

**Starter code for `pca.py`:**

```python
import numpy as np


class PCA:
    """Principal component analysis computed with the singular value decomposition."""

    def fit(self, X):
        """Center X, compute the SVD of the centered data, and store:
        mean_ (shape d), components_ (shape d by d, rows are unit directions),
        and explained_variance_ratio_ (shape d, sums to 1).
        """
        raise NotImplementedError

    def transform(self, X, k):
        """Project X (shape n by d) onto the first k components. Return shape (n, k)."""
        raise NotImplementedError

    def inverse_transform(self, Z):
        """Map Z (shape n by k) back to the original feature space. Return shape (n, d)."""
        raise NotImplementedError
```

**Starter code for `tests/test_pca.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from pca import PCA


def make_data():
    rng = np.random.default_rng(0)
    base = rng.standard_normal((200, 2))
    mixing = rng.standard_normal((2, 5))
    return base @ mixing + 0.05 * rng.standard_normal((200, 5))


def test_components_are_orthonormal():
    model = PCA().fit(make_data())
    W = model.components_[:3]
    assert np.allclose(W @ W.T, np.eye(3))


def test_explained_variance_sums_to_one():
    model = PCA().fit(make_data())
    assert model.explained_variance_ratio_.sum() == pytest.approx(1.0)


def test_two_components_capture_most_variance():
    model = PCA().fit(make_data())
    assert model.explained_variance_ratio_[:2].sum() > 0.95


def test_transform_shape():
    X = make_data()
    assert PCA().fit(X).transform(X, k=2).shape == (200, 2)


def test_reconstruction_error_shrinks_with_more_components():
    X = make_data()
    model = PCA().fit(X)
    errors = []
    for k in (1, 2, 3):
        X_hat = model.inverse_transform(model.transform(X, k))
        errors.append(np.linalg.norm(X - X_hat))
    assert errors[0] > errors[1] > errors[2]
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy`, `pytest` and `matplotlib`.
2. Implement `fit`. Compute the mean along axis 0, subtract it, and call `np.linalg.svd` with `full_matrices=False`. Store the right singular vectors as `components_`, and convert the squared singular values into variance ratios.
3. Implement `transform` by centering with `mean_` and multiplying by the transpose of the first `k` rows of `components_`.
4. Implement `inverse_transform` by multiplying `Z` by the first `Z.shape[1]` rows of `components_` and adding `mean_` back.
5. Run `pytest` until all five tests pass.
6. Write `analysis.py`. It should load the same synthetic data, print the explained variance ratio for each component and the reconstruction error for `k` from 1 to 5, and save a scatter plot of the first two projected coordinates to `results/pca_scatter.png`.
7. In the README, write a short analysis of the output. State how many components you would keep and why, and say what the discarded components contain.
8. Commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is written in one sentence
- [ ] Install and run commands are listed (`pip install -r requirements.txt`, `pytest`, `python analysis.py`)
- [ ] The explained variance and reconstruction error results are pasted in a table
- [ ] The scatter plot is embedded with a caption
- [ ] The README names two limitations of PCA that apply to this dataset, such as linearity and sensitivity to scale

**Done when:**

- `pytest` passes all five tests
- The explained variance of the first two components is above 95 percent on the test data
- `analysis.py` prints a table and saves `results/pca_scatter.png`
- The README recommends a number of components with a clear reason
- You can explain in one paragraph why the components are orthogonal

**What to commit as evidence:** the source files, the tests, the analysis output pasted into the README, the scatter plot, and the written analysis.

---

## Check your understanding

1. A layer computes `ReLU(W x + b)`. Explain which parts of this are linear transformations, which part is not, and what would happen to the stacked layers if you removed the ReLU.
2. A feature matrix has 12 columns, but its rank is 9. Explain what this tells you about the features, and give one practical step to take before training a linear model.
3. Two features have a covariance of zero. Can you conclude that they are independent? Give an example where two variables are uncorrelated and dependent.
4. A symmetric matrix has eigenvalues 5, 2 and -1. Explain what the negative eigenvalue tells you about the direction of its eigenvector, and whether the matrix is invertible.
5. Explain why the singular values of a matrix are always nonnegative, while eigenvalues can be negative, and say which of the two the SVD guarantees to be sorted.
6. You apply PCA to data with two features measured in very different units, such as income in dollars and age in years, without standardizing. Predict which component the first principal component will mostly follow, and explain the fix.
7. A teammate says PCA with 2 components keeps 95 percent of the information in a dataset. What is the precise claim the variance ratio supports, and what might it miss for a classification task?

### Answers

1. The matrix multiplication `W x` and the addition of a bias are part of an affine map, where the multiplication is the linear part and the bias makes it affine. The ReLU is the nonlinear part. Without the ReLU, stacking layers would still produce one linear map, so the depth would add no expressive power.
2. The 12 features span only a 9-dimensional space, so at least three features are linear combinations of others and carry no new information. A practical step is to detect the redundancy with the correlation matrix or the rank tests, then remove or combine the redundant columns before fitting a regression with unstable coefficients.
3. No. Zero covariance means the variables are uncorrelated, which is weaker than independence. For example, if `x` is symmetric around zero and `y = x^2`, the covariance is zero, yet `y` depends on `x` completely.
4. The eigenvector for `-1` points in a direction that the matrix reverses, which is a legitimate stretch by a negative factor. The matrix is invertible, because all its eigenvalues are nonzero, and its determinant is `5 * 2 * (-1) = -10`.
5. Singular values are defined as the square roots of the eigenvalues of `A^T A`, which are nonnegative, so they cannot be negative. Eigenvalues of a general matrix can be negative. The SVD guarantees that the singular values are sorted from largest to smallest.
6. The first component will mostly follow income, because its numbers are much larger, so its variance dominates the covariance matrix. The fix is to standardize each feature to zero mean and unit variance before PCA, so that each feature contributes according to its variation rather than its unit.
7. The claim is that 95 percent of the total variance in the features lies in the first two components. Variance is not information about a label. A classifier may depend on a low-variance direction that PCA discards, so the 95 percent figure can hide a large loss in predictive power.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Linear map](https://en.wikipedia.org/wiki/Linear_map)
- Wikipedia: [Basis (linear algebra)](https://en.wikipedia.org/wiki/Basis_(linear_algebra))
- Wikipedia: [Rank (linear algebra)](https://en.wikipedia.org/wiki/Rank_(linear_algebra))
- Wikipedia: [Eigenvalues and eigenvectors](https://en.wikipedia.org/wiki/Eigenvalues_and_eigenvectors)
- Wikipedia: [Singular value decomposition](https://en.wikipedia.org/wiki/Singular_value_decomposition)
- Wikipedia: [Principal component analysis](https://en.wikipedia.org/wiki/Principal_component_analysis)
- Wikipedia: [Dimensionality reduction](https://en.wikipedia.org/wiki/Dimensionality_reduction)
- Stanford CS229, "Linear Algebra Review and Reference" by Zico Kolter, the sections on eigenvalues and the SVD. Search the title with "Stanford CS229" to find the current PDF.
- NumPy documentation, the reference pages for `numpy.linalg.svd`, `numpy.linalg.eigh` and `numpy.cov` on numpy.org.

**Tutorials**

- scikit-learn documentation, "Principal component analysis (PCA)" in the decomposition user guide. Read it after the project to compare your implementation with a production one.
- GeeksforGeeks, search "eigenvalues and eigenvectors" and "singular value decomposition" for worked examples with numbers.

**Watch**

- 3Blue1Brown (YouTube), "Linear transformations and matrices | Chapter 3, Essence of linear algebra". It gives the picture of a matrix as a machine that moves space.
- 3Blue1Brown (YouTube), "Eigenvectors and eigenvalues | Chapter 14, Essence of linear algebra". It shows why some vectors keep their direction under a transformation.
- StatQuest with Josh Starmer (YouTube), "Principal Component Analysis (PCA), Step-by-Step". It walks through the variance calculation with numbers.
- MIT OpenCourseWare, "18.06 Linear Algebra" by Prof. Gilbert Strang, the lectures on eigenvalues and on the singular value decomposition.

**Practice**

- Khan Academy, "Linear algebra" course, the units on eigenvalues and bases, with interactive exercises.

---

## Tomorrow

Tomorrow you move from the shape of data to the change in a function. Derivatives measure how fast an output changes when an input moves a little, and the gradient collects those rates for many inputs at once. The gradient is the direction in which a loss function rises fastest, and training a model means moving the weights the other way. You will compute derivatives by hand, check them numerically and apply the chain rule to a small network, which is the calculus that backpropagation uses.

**Next:** [Day 11: Calculus I](Day-11-Calculus-I.md)

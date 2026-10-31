# Day 17: NumPy

*100 Days of AI Engineering, Module 3: Data Engineering for AI*

**Time:** about 3.5 hours
**Level:** intermediate. You can write Python functions and loops and you have seen vectors and matrices in Days 8 and 9. Today you turn that math into code that runs on whole arrays at once.

**By the end of this day, you will have:**

- A working grasp of ndarrays, shapes, dimensions, indexing, slicing, broadcasting, vectorization, matrix operations and seeded random generation
- A small Python package, `numpy_toolkit`, with standardization, pairwise distances, covariance and a reproducible dataset generator, all tested with pytest
- A timing comparison that shows, with your own numbers, how much faster a vectorized expression runs than a Python loop

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 2.0 or newer (`pip install "numpy>=2.0"`). NumPy is free and open source under the BSD 3-Clause license.
- pytest 8.0 or newer (`pip install "pytest>=8.0"`)
- A code editor such as VS Code (free) or any editor you already use
- Git and a free GitHub account for the project section
- Paper for the shape diagrams in the exercises

**Prerequisites:**

- Day 8: vectors, the dot product and magnitude
- Day 9: matrices, matrix multiplication, transpose and shapes
- Day 16: summary statistics such as the mean and standard deviation
- Basic Python: functions, lists, `for` loops and importing modules

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Ndarrays](#ndarrays)
- [Shapes](#shapes)
- [Dimensions](#dimensions)
- [Indexing and slicing](#indexing-and-slicing)
- [Broadcasting](#broadcasting)
- [Vectorization](#vectorization)
- [Matrix operations](#matrix-operations)
- [Random generation](#random-generation)
- [Hands-on exercises](#hands-on-exercises)
- [Project: numpy_toolkit, tested array utilities](#project-numpy_toolkit-tested-array-utilities)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every dataset you will touch in this series, from a table of house prices to a batch of 512 token embeddings, eventually becomes an array of numbers. Python lists can hold numbers, but they hold them as separate objects scattered through memory, and every arithmetic operation runs through the interpreter one element at a time. NumPy solves both problems. It stores numbers in a contiguous block of memory and runs operations on the whole block in compiled code. The library is the foundation that pandas, scikit-learn, PyTorch and most of the Python machine learning ecosystem sits on.

Today is about thinking in arrays. When you read a line like `X = (X - X.mean(axis=0)) / X.std(axis=0)`, you should see one operation applied to every entry of a table at once, not a loop. That habit is what separates code that trains a model in seconds from code that takes an afternoon.

On the job, you will use NumPy to preprocess features before they reach a model, to build the matrices that a similarity search or a recommender needs, to generate synthetic test data with a fixed seed, and to debug the shape errors that appear whenever a tensor from one stage meets a tensor from another. Most real bugs in array code are shape bugs, so the shape vocabulary in this lesson will save you hours.

---

## Short history

| Period | What happened | Why it matters today |
|---|---|---|
| 1995 | Jim Hugunin creates Numeric, an early array package for Python. | The first widely used numerical array library for Python. |
| 2001 to 2005 | Travis Oliphant and others build Numarray and then merge the ideas into NumPy, released in 2006. | NumPy becomes the single array standard, which is why it still uses the name today. |
| 2006 to 2010 | The scientific Python community adds SciPy, matplotlib and pandas on top of NumPy arrays. | Data science in Python is built on one shared array type. |
| 2017 to 2019 | NumPy adds the `__array_ufunc__` (NumPy 1.13) and `__array_function__` (NumPy 1.17) protocols, so other libraries can take over NumPy calls. | Libraries such as Dask and CuPy can accept NumPy-style calls on distributed or GPU arrays. |
| 2024 | NumPy 2.0 is released, with a cleaner namespace and new type promotion and printing rules. | Code written for NumPy 1.x may need small updates, so check the version you install. |

**Takeaway:** NumPy is old, stable and everywhere. Learn its array model once and you can read code from almost any Python data project.

---

## Glossary

- **Array**: a grid of values that all share one data type and are stored in one block of memory.
- **ndarray**: NumPy's name for its array type, short for n-dimensional array.
- **Shape**: a tuple that gives the length of the array along each axis, such as `(3, 4)`.
- **Dimension (ndim)**: the number of axes an array has. A vector has one, a matrix has two.
- **Axis**: a direction along which an array is indexed. In a `(rows, columns)` matrix, axis 0 runs down the rows and axis 1 runs across the columns.
- **dtype**: the data type of every element, such as `float64` or `int32`.
- **Slice**: a range of indices written as `start:stop:step`, where `stop` is not included.
- **View**: an array that shares memory with another array, so changing one changes the other.
- **Broadcasting**: the rule that lets NumPy combine arrays of different shapes by stretching the smaller one without copying it.
- **Vectorization**: writing an operation on whole arrays so that NumPy's compiled code does the looping.
- **Ufunc**: a universal function, which is NumPy's name for an element-wise operation such as `np.add` or `np.exp`.
- **Seed**: a number that sets the starting state of a random number generator so that the same random numbers come out again.
- **Pseudorandom number generator (PRNG)**: an algorithm that produces numbers that look random but are fully determined by the seed.

---

### Ndarrays

Think of a shipping container. Each container holds boxes of the same size, packed in a fixed grid. You can count the boxes along each wall, find any box by its coordinates, and move the whole load at once. An ndarray is that container. Every element has the same data type, the elements sit in one contiguous block of memory, and the array knows its own dimensions.

Python lists are more like a bag of mixed items, where each item is a separate object with its own type and location. That flexibility costs speed. A list of one million floats stores one million pointers to one million float objects. The same data in an ndarray is one block of about eight megabytes for `float64`, with no per-element objects.

From first principles, an ndarray is four pieces of information: a pointer to the memory block, a shape, a dtype, and a set of strides that say how many bytes to skip to move one step along each axis. Because strides are arithmetic, many operations, such as transposing or taking a slice, can return a view that reuses the same block without copying a single number.

```python
import numpy as np

from_list = np.array([1, 2, 3])             # built from a Python list
zeros = np.zeros((2, 3))                    # 2 rows, 3 columns, all 0.0
ramp = np.arange(0, 10, 2)                  # [0 2 4 6 8]
grid = np.linspace(0.0, 1.0, num=5)         # 5 evenly spaced values from 0 to 1

print(from_list.dtype, zeros.shape, ramp, grid)
```

A common mistake is mixing types in the constructor and not noticing. `np.array([1, 2.5, 3])` silently becomes a float array, and `np.array([1, "a"])` becomes a string array with every number converted to text. Always print `dtype` after you build an array from mixed data. An ndarray is the object that sits underneath a pandas DataFrame, a scikit-learn feature matrix and a PyTorch tensor, and the conversion between them is usually a view, not a copy.

---

### Shapes

The shape of an array is its size along each axis, written as a tuple. A list of 100 numbers has shape `(100,)`. A table of 100 rows and 3 columns has shape `(100, 3)`. A batch of 32 color images of size 224 by 224 with 3 channels, in the usual channels-last layout, has shape `(32, 224, 224, 3)`.

Shapes are the contract between stages of a pipeline. When a model expects `(batch, features)` and receives `(features, batch)`, the arithmetic may still run without error on some inputs, which makes the bug harder to see. Printing `.shape` before and after every step is cheap insurance.

```python
import numpy as np

X = np.arange(12).reshape(4, 3)   # 4 rows, 3 columns
print(X.shape)                    # (4, 3)

flat = X.reshape(-1)              # -1 means "work out this length"
print(flat.shape)                 # (12,)

col = X[:, 0:1]                   # keep a 2D column instead of a 1D vector
print(col.shape)                  # (4, 1)
```

`reshape` changes the shape without changing the data, and it returns a view when it can. The `-1` placeholder is convenient but dangerous when the total size does not divide evenly, because NumPy will raise an error that is easy to misread. The mistake to watch for is the difference between `(n,)` and `(n, 1)`. A one-dimensional vector and a single-column matrix hold the same numbers, but they broadcast differently and matrix multiplication treats them differently. The real-world use is in every model input pipeline, where the batch dimension is usually the first axis and a reshape that moves it is a silent source of wrong predictions.

---

### Dimensions

Dimensions count the axes, and each axis is a level of nesting. A scalar has zero dimensions. A vector has one. A matrix has two. A tensor has three or more. The `ndim` attribute reports the count, and the shape tells you the length of each level.

It helps to picture a stack of pages. One page is a matrix. A stack of pages is a three-dimensional array. A set of stacks, one per day of a week, is four dimensions. Each added dimension multiplies the number of entries, so a `(1000, 1000, 1000)` array of `float64` already needs about 8 gigabytes of memory.

```python
import numpy as np

scalar = np.array(7.0)
vector = np.array([1.0, 2.0, 3.0])
matrix = np.ones((2, 3))
tensor = np.zeros((2, 3, 4))

for name, arr in [("scalar", scalar), ("vector", vector),
                  ("matrix", matrix), ("tensor", tensor)]:
    print(name, arr.ndim, arr.shape, arr.size)
```

Two mistakes come up often. The first is confusing `size` with `shape`: `size` is the total number of elements, and `shape` is the per-axis breakdown. The second is assuming that the first axis is always the sample axis. In pandas and scikit-learn it usually is, but in some image libraries and time series code it is not. Read the documentation for the function you call. Dimension bookkeeping is the core skill of anyone who debugs a tensor shape error in a deep learning job.

---

### Indexing and slicing

Indexing selects single elements. Slicing selects ranges. Think of a seat plan for a theater: a row and a seat number pick one person, and a range of rows picks a block of people. NumPy follows the same logic, counting from zero, with one index or slice per axis separated by commas.

The key idea is that a basic slice returns a view that shares memory with the original array. Changing the view changes the source. When you need an independent copy, you call `.copy()`. NumPy also supports boolean indexing, where a mask of True and False values keeps only the matching elements, and integer array indexing, where you pass a list of positions.

```python
import numpy as np

X = np.array([[10, 11, 12],
              [20, 21, 22],
              [30, 31, 32]])

print(X[1, 2])          # 22, row 1, column 2
print(X[:2, 1:])        # rows 0 and 1, columns 1 and 2
print(X[X[:, 0] > 15])  # rows where the first column is above 15

view = X[0]             # shares memory with X
copy = X[0].copy()      # independent
view[0] = -1
print(X[0, 0], copy[0]) # -1 10
```

The mistake to avoid is editing a slice and assuming the original is safe, or the reverse. Fancy indexing, which uses arrays of positions or boolean masks, returns a copy, so the same code can behave differently depending on which kind of index you used. Boolean masks are how analysts drop bad rows, filter out outliers and select a class before computing a metric.

---

### Broadcasting

Broadcasting is the rule that lets NumPy combine arrays of different shapes. Picture a single row of cookie cutters pressed into a sheet of dough. The cutter is smaller than the sheet, but it is applied everywhere without you making copies of it. Broadcasting does the same with arrays.

From first principles, NumPy compares shapes from the last axis backward. Two sizes are compatible when they are equal or when one of them is 1. A missing leading axis counts as 1. The smaller array is then virtually stretched along the axes where it has length 1. Nothing is copied into memory in the stretched form, which is why broadcasting is fast.

```python
import numpy as np

X = np.array([[1.0, 2.0],
              [3.0, 4.0],
              [5.0, 6.0]])      # shape (3, 2)

col_means = X.mean(axis=0)      # shape (2,): [3. 4.]
centered = X - col_means        # shape (3, 2) minus shape (2,) -> (3, 2)
print(centered)
# [[-2. -2.]
#  [ 0.  0.]
#  [ 2.  2.]]

A = np.array([[0.0, 0.0],
              [3.0, 4.0]])      # 2 points
B = np.array([[1.0, 0.0],
              [0.0, 1.0],
              [3.0, 4.0]])      # 3 points

diff = A[:, None, :] - B[None, :, :]   # shape (2, 3, 2)
dist = np.sqrt((diff ** 2).sum(axis=-1))
print(dist.round(3))
# [[1.    1.    5.   ]
#  [4.472 4.243 0.   ]]
```

By inserting axes with `None`, you turn two lists of points into a full table of distances between every point in A and every point in B, with no Python loop. The pattern `A[:, None, :] - B[None, :, :]` appears in nearest-neighbor search, kernel methods and attention score computations.

The common mistake is broadcasting silently in the wrong direction. If `X` has shape `(3, 2)` and you subtract a vector of length 3 instead of length 2, NumPy raises an error, which is good. But if you subtract a `(3, 1)` column from a `(3, 2)` matrix, the code runs and gives a valid-looking result that is wrong for your intent. Check the shapes of both operands before you trust the output. Broadcasting is central to every normalization layer in a neural network and every feature scaler in scikit-learn.

---

### Vectorization

Vectorization means expressing a computation on whole arrays so that the loop runs inside NumPy's compiled code. Think of a cashier who scans a full cart in one motion, compared with one who picks up each item and reads its price one at a time. The work is the same, but the overhead of the interpreter disappears.

The speedup can be large. A Python loop pays for a type check, a function lookup and an object creation on every iteration. A vectorized operation pays for them once. The cost of the actual arithmetic is the same either way, so the gap is biggest for simple operations on large arrays.

```python
import time
import numpy as np

values = np.random.default_rng(0).random(5_000_000)

start = time.perf_counter()
total_loop = 0.0
for v in values:               # Python loop, one element at a time
    total_loop += v * v
loop_time = time.perf_counter() - start

start = time.perf_counter()
total_vec = np.dot(values, values)   # vectorized
vec_time = time.perf_counter() - start

print(f"loop {loop_time:.3f} s, vectorized {vec_time:.5f} s")
print(np.isclose(total_loop, total_vec))  # True
```

On a typical laptop the loop is often one hundred times slower or more. Run the snippet yourself and record both times, because the ratio depends on your machine.

Vectorization has limits. Not every algorithm is an element-wise operation. A recurrence where each step depends on the previous result, such as a running state in a time series filter, does not vectorize cleanly, and you may need a compiled extension or a library such as Numba. Another limit is memory: a vectorized expression creates temporary arrays, so `a * b + c * d` allocates intermediate results. In a memory-tight job, you can reduce that cost with in-place operators such as `+=` or with `np.einsum`. The real-world consequence is that a feature pipeline written with loops over a few hundred thousand rows can take minutes, while the same logic written as array expressions finishes in well under a second.

---

### Matrix operations

Matrix operations are the arithmetic of linear algebra. You met them as ideas on Days 8 and 9. In NumPy, the `@` operator performs matrix multiplication, `.T` gives the transpose, and `np.linalg` provides solvers, inverses and decompositions. The distinction that trips up beginners is that `*` multiplies element by element, while `@` multiplies rows by columns.

Think of a spreadsheet. Element-wise multiplication multiplies each cell by the cell in the same position in another sheet. Matrix multiplication computes each cell of the result as a sum of products along a row of one sheet and a column of the other. The shapes must agree: an `(m, k)` matrix times a `(k, n)` matrix gives an `(m, n)` result.

```python
import numpy as np

X = np.array([[1.0, 2.0],
              [3.0, 4.0],
              [5.0, 6.0]])      # shape (3, 2), three samples with two features

gram = X.T @ X                  # shape (2, 2), feature by feature
print(gram)                     # [[35. 44.] [44. 56.]]

y = np.array([1.0, 0.0, 1.0])
w = np.linalg.lstsq(X, y, rcond=None)[0]   # least squares solution
print(w.shape)                  # (2,)

A = np.array([[3.0, 1.0],
              [1.0, 2.0]])
b = np.array([9.0, 8.0])
print(np.linalg.solve(A, b))    # [2. 3.]
```

`np.linalg.solve` is preferred over computing `np.linalg.inv(A) @ b`. Forming an inverse is slower and less numerically stable, and solving the system directly is what you almost always want. The mistake is to write a formula from a textbook directly into code without checking whether a library routine does it more accurately. The Gram matrix `X.T @ X` and least squares solves sit inside linear regression, ridge regression and the covariance calculations used in PCA.

---

### Random generation

Random numbers power data splits, weight initialization, dropout, data augmentation and synthetic test data. The important property for engineering is reproducibility. A random generator with a fixed seed produces the same sequence every time, so a bug you find today can be reproduced tomorrow.

Think of a shuffled deck with a known shuffle. If you write down the seed, anyone can deal the same hand. NumPy's modern interface is `np.random.default_rng`, which returns a `Generator` object you pass around explicitly. The older `np.random.seed` function sets a hidden global state, which makes code harder to reason about when several parts of a program use random numbers.

```python
import numpy as np

rng = np.random.default_rng(seed=42)

samples = rng.normal(loc=0.0, scale=1.0, size=(4, 3))  # standard normal
ints = rng.integers(low=0, high=10, size=5)            # integers in [0, 10)
perm = rng.permutation(8)                              # shuffled 0..7

rng_again = np.random.default_rng(seed=42)
same = rng_again.normal(loc=0.0, scale=1.0, size=(4, 3))
print(np.array_equal(samples, same))                   # True
```

A common mistake is creating a new generator inside a loop with the same seed. Every iteration then draws the same numbers, and your "random" augmentations are identical. Create the generator once, at the top of the program or in a function argument, and pass it down. Another mistake is forgetting that a seed fixes the sequence only for the same library version and the same order of calls. Seeded generators make experiments comparable and let a test suite check that a data generator gives identical output on every run.

---

## Hands-on exercises

Work through these in a notebook or a single script. Print the output of every step and write one sentence next to each result explaining why the number is what it is.

1. **Shape tour.** Create an array with `np.arange(24)`. Reshape it to `(2, 3, 4)`, then to `(4, 6)`, then to `(2, 12)`. Print `ndim`, `shape` and `size` after each reshape. Predict each shape before you run the line.
2. **View or copy.** Build a `(3, 3)` array of zeros. Take a row slice and set it to 5. Print the original. Repeat with `.copy()` and confirm the original is unchanged. Write down which operations gave views.
3. **Boolean filter.** Generate 20 random integers between 0 and 100 with a seeded generator. Keep only the values above 50 and print how many remain. Then count them with `np.sum(values > 50)` and confirm the two numbers match.
4. **Broadcasting check.** Given `X` of shape `(5, 3)`, write one line that subtracts the per-column minimum and divides by the per-column range. Then predict the shape of the intermediate results. Verify with `.shape`.
5. **Pairwise distance.** Using the broadcasting pattern from this lesson, compute the distance between 4 points in 2D and 3 points in 2D. Compare your result with a double loop for the same points. Check that every entry matches.
6. **Timing.** Repeat the loop-versus-vectorized timing with 1 million and then 10 million values. Record both times in a table. Describe in one paragraph how the ratio changes.
7. **Least squares.** Create a design matrix with a column of ones and one feature, plus a target that is roughly linear in the feature. Fit the line with `np.linalg.lstsq`. Print the slope and intercept and check that they are close to the values you used to generate the data.
8. **Seed discipline.** Write a function `make_noise(n, rng)` that returns `n` samples from a normal distribution. Call it twice with the same generator and confirm the outputs differ. Then call it twice with two new generators seeded identically and confirm the outputs match.

---

## Project: numpy_toolkit, tested array utilities

### Goal

Build a small, tested Python package of array utilities that you would be willing to reuse in a feature pipeline. The package provides standardization with safe handling of constant columns, a broadcasting-based pairwise distance function, a covariance function that you check against NumPy's own implementation, and a reproducible synthetic dataset generator.

Suggested repository name: `100-days-ai-day-17-numpy`

### Repo layout

```text
100-days-ai-day-17-numpy/
├── README.md
├── numpy_toolkit/
│   ├── __init__.py
│   └── features.py
├── tests/
│   └── test_features.py
├── benchmarks/
│   └── timing.py
├── requirements.txt
└── .gitignore
```

### Starter code

`requirements.txt`:

```text
numpy>=2.0
pytest>=8.0
```

`numpy_toolkit/__init__.py`:

```python
from .features import standardize, pairwise_distances, covariance, make_dataset

__all__ = ["standardize", "pairwise_distances", "covariance", "make_dataset"]
```

`numpy_toolkit/features.py`:

```python
import numpy as np


def standardize(X: np.ndarray) -> np.ndarray:
    """Center each column to mean 0 and scale it to standard deviation 1.

    Columns with zero standard deviation are centered but not scaled,
    so they do not produce NaN or infinity.
    """
    X = np.asarray(X, dtype=np.float64)
    mean = X.mean(axis=0)
    std = X.std(axis=0)
    std = np.where(std == 0.0, 1.0, std)
    return (X - mean) / std


def pairwise_distances(A: np.ndarray, B: np.ndarray) -> np.ndarray:
    """Return the Euclidean distance between every row of A and every row of B.

    Output shape is (len(A), len(B)).
    """
    A = np.asarray(A, dtype=np.float64)
    B = np.asarray(B, dtype=np.float64)
    diff = A[:, None, :] - B[None, :, :]
    return np.sqrt((diff ** 2).sum(axis=-1))


def covariance(X: np.ndarray) -> np.ndarray:
    """Sample covariance matrix of the columns of X, with denominator n - 1."""
    X = np.asarray(X, dtype=np.float64)
    n = X.shape[0]
    Xc = X - X.mean(axis=0)
    return (Xc.T @ Xc) / (n - 1)


def make_dataset(n_samples: int, n_features: int, seed: int) -> np.ndarray:
    """Create a reproducible matrix of correlated Gaussian features."""
    rng = np.random.default_rng(seed)
    latent = rng.normal(size=(n_samples, 2))
    mixing = rng.normal(size=(2, n_features))
    noise = 0.1 * rng.normal(size=(n_samples, n_features))
    return latent @ mixing + noise
```

`tests/test_features.py`:

```python
import numpy as np
from numpy_toolkit import standardize, pairwise_distances, covariance, make_dataset


def test_standardize_zero_mean_unit_std():
    X = make_dataset(200, 5, seed=1)
    Z = standardize(X)
    assert np.allclose(Z.mean(axis=0), 0.0, atol=1e-12)
    assert np.allclose(Z.std(axis=0), 1.0, atol=1e-12)


def test_standardize_constant_column_is_safe():
    X = np.array([[1.0, 5.0], [2.0, 5.0], [3.0, 5.0]])
    Z = standardize(X)
    assert np.all(np.isfinite(Z))
    assert np.allclose(Z[:, 1], 0.0)


def test_pairwise_matches_brute_force():
    rng = np.random.default_rng(3)
    A = rng.normal(size=(4, 2))
    B = rng.normal(size=(3, 2))
    expected = np.zeros((4, 3))
    for i in range(4):
        for j in range(3):
            expected[i, j] = np.linalg.norm(A[i] - B[j])
    assert np.allclose(pairwise_distances(A, B), expected)


def test_covariance_matches_numpy():
    X = make_dataset(100, 4, seed=7)
    assert np.allclose(covariance(X), np.cov(X, rowvar=False))


def test_make_dataset_is_reproducible():
    a = make_dataset(50, 3, seed=11)
    b = make_dataset(50, 3, seed=11)
    c = make_dataset(50, 3, seed=12)
    assert np.array_equal(a, b)
    assert not np.array_equal(a, c)
```

`benchmarks/timing.py`:

```python
import time
import numpy as np


def loop_sum_squares(x):
    total = 0.0
    for v in x:
        total += v * v
    return total


if __name__ == "__main__":
    x = np.random.default_rng(0).random(2_000_000)
    t0 = time.perf_counter()
    loop_sum_squares(x)
    t1 = time.perf_counter()
    np.dot(x, x)
    t2 = time.perf_counter()
    print(f"loop: {t1 - t0:.3f} s")
    print(f"vectorized: {t2 - t1:.5f} s")
```

### Step-by-step tasks

1. Create the repository folder, initialize Git with `git init`, and create a virtual environment with `python -m venv .venv`. Activate it and run `pip install -r requirements.txt`.
2. Create the files exactly as shown above. Run `pytest -q`. All five tests should pass.
3. Break one thing on purpose. Change `std = np.where(std == 0.0, 1.0, std)` to a plain division and rerun the tests. Confirm that the constant-column test fails with NaN values. Restore the line.
4. Add a test that `pairwise_distances(A, A)` has zeros on its diagonal.
5. Run `python benchmarks/timing.py` and paste the two timings into your README.
6. Add a short function `top_k_neighbors(A, B, k)` that returns the indices of the k nearest rows of B for each row of A, using `np.argsort` on the output of `pairwise_distances`. Write one test for a case where you know the answer by hand.
7. Write the README with four sections: what the package does, how to run the tests, the benchmark numbers from your machine, and one limitation (for example, the pairwise function builds a full `(n, m, d)` array and uses a lot of memory for large inputs).

### README checklist

- [ ] Install instructions with the NumPy version you tested
- [ ] One example call for each public function
- [ ] Output of `pytest -q`, copied as text
- [ ] Your benchmark numbers and the machine you ran them on
- [ ] A note on the memory cost of the broadcasting distance function
- [ ] The seed used in `make_dataset` and why it matters

### Done when

- `pytest -q` passes with no warnings about deprecated NumPy calls
- `standardize` returns finite values for a matrix that contains a constant column
- `pairwise_distances` agrees with a brute-force double loop on random points
- `covariance` agrees with `np.cov(X, rowvar=False)` to within floating-point tolerance
- `make_dataset` returns identical arrays for the same seed and different arrays for a different seed
- The README reports your own benchmark numbers
- The repository has at least three commits with clear messages

### Evidence to commit to GitHub

Commit the package, the tests, the benchmark script, the README and the `requirements.txt`. The reader should be able to show a green `pytest -q` run, a README with real benchmark numbers, and a commit history that records the broken-then-fixed standardization step. Do not commit the `.venv` folder.

---

## Check your understanding

1. You write `scores = X @ w` where `X` has shape `(100, 5)` and `w` has shape `(5,)`, then later `scores = X * w`, which has the same shape and no error. Why does the second line silently compute something different, and which one would a linear model want?
2. A colleague says that a `float64` array of one million rows and ten columns is fine, but that a list of lists with the same numbers is "about the same" in memory. Explain what is stored in each case and estimate the size of the array.
3. Your pairwise distance function works for 500 points but crashes with a memory error at 20,000 points. Using what you know about broadcasting, explain why, and propose a change that keeps the same output with less memory.
4. Two runs of an experiment give different results even though you set a seed. List three things in the code, besides the seed value itself, that could cause this, and say how you would check each one.
5. A teammate reports that changing a value in a slice of a feature matrix also changed a value in the raw data that was kept for auditing. What kind of indexing created the slice, and what would you change so the audit copy is safe?
6. When is `np.linalg.solve(A, b)` the right call, and when might you still need an explicit inverse? Give the reason for your answer in terms of numerical stability and the number of times the matrix is reused.

Work each one out in writing before you read the answers below.

**Answers**

1. `X @ w` is matrix-vector multiplication. It takes each row of `X`, dots it with `w`, and returns 100 scores, which is what a linear model wants. `X * w` broadcasts `w` across the rows and multiplies each column by one weight, which gives a 100 by 5 matrix of products. It runs without error because `(5,)` broadcasts against `(100, 5)`. Print the shape of the result and compare it with what you expected.
2. The NumPy array stores 8 bytes per `float64` value in one contiguous block, so one million rows by ten columns is about 80 megabytes, plus a small header. A list of lists stores a pointer to each row list and a separate Python float object for each value, and each float object takes about 24 bytes on top of the 8-byte pointer in the list. The list version is several times larger and slower to process.
3. `A[:, None, :] - B[None, :, :]` creates an intermediate array of shape `(n, m, d)`. At 20,000 points on each side with 2 features, that is 800 million float64 values, about 6.4 gigabytes, before the square root. A fix is to use the identity `|a - b|^2 = |a|^2 + |b|^2 - 2 a.b`, which builds the result from an `(n, m)` matrix product and needs only `(n, m)` memory. Check that the two versions agree on small inputs before you switch.
4. Three common causes: the generator is created inside a loop or re-seeded in a helper function; a function uses the global `np.random` state, and another library consumes random numbers in a different order; or the library version changed between runs. To check, print the first few random values at the start of each stage, log the library versions, and make sure one generator object is passed explicitly through the code.
5. A basic slice such as `X[:, 0:5]` returns a view that shares memory with `X`, so writing to the slice writes to the original. Use `.copy()` on the slice, or store the audit data in a separate array before any transformation, so that the two never share memory.
6. `np.linalg.solve` is right when you need the solution of one system, because it avoids forming the inverse and is more accurate. An explicit inverse is only worth considering when the same matrix is applied to many different right-hand sides in a way that cannot be restructured, and even then a factorization that is reused, such as an LU or Cholesky factorization, is usually the better choice.

---

## Sources to read, watch and practice

**Read**

- Wikipedia, article "NumPy", for the project history and its role in scientific Python.
- Wikipedia, article "Array programming", for the general idea behind vectorized operations.
- Wikipedia, article "Pseudorandom number generator", for why seeded generators behave the way they do.
- NumPy documentation, User Guide, including the sections on the ndarray, indexing, broadcasting and random sampling. Search "NumPy User Guide" on numpy.org.
- Jake VanderPlas, *Python Data Science Handbook*, the chapter on NumPy. The book is free to read online and the author hosts it on GitHub under his name.

**Tutorials**

- GeeksforGeeks, "NumPy Tutorial" and the articles on NumPy broadcasting and NumPy indexing.
- W3Schools, "NumPy Tutorial" in the Python section of w3schools.com.

**Watch**

- Search YouTube for "NumPy broadcasting explained" and pick a talk with a clear whiteboard or screen-recorded shape diagram.
- Search YouTube for "SciPy conference NumPy talk" to find the recorded conference talks by NumPy maintainers.
- 3Blue1Brown, "Essence of linear algebra", on YouTube. It gives the picture behind the matrix operations you run here.

**Practice**

- The `numpy-100` exercise set by Nicolas Rougier, a collection of 100 NumPy problems hosted on GitHub. Solve at least twenty of them, checking your answers against the shape rules in this lesson.

---

## Tomorrow

Tomorrow is pandas. NumPy gives you fast arrays, but a real dataset has named columns, mixed types and missing entries, and it needs to be sliced by label as well as by position. pandas wraps NumPy arrays in a table called a DataFrame and adds the operations you need to filter, group, merge and reshape real data. Everything you built today, the shape reasoning, broadcasting and seeded generation, will show up again when you call `.to_numpy()` on a DataFrame to feed a model.

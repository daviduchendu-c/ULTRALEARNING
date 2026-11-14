# Day 9: Matrices

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** beginner. You should be comfortable with Day 8's vectors and dot products. If you have used a spreadsheet or a pandas DataFrame, you already have the right picture.

**By the end of this day, you will have:**

- A matrix toolkit in Python that multiplies matrices with an explicit loop and checks the result against NumPy's `@` operator
- A forward pass for one dense neural-network layer, written as a matrix product with a bias and a ReLU, with shape checks at every step
- A solver for a small system of linear equations that reports the residual, so you can judge whether the answer is trustworthy

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 2.0 or newer (`pip install numpy`), a free and open-source library released under the BSD license
- pytest 8.0 or newer (`pip install pytest`)
- A code editor such as VS Code (free) or any editor you already use
- Git and a free GitHub account for the project section
- Paper and a pencil for the multiplication exercises

**Prerequisites:**

- Day 7: functions, sums and exponents
- Day 8: vectors, the dot product and cosine similarity
- Basic Python: lists, nested loops, functions, and running a script from the terminal

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Matrices](#matrices)
- [Shapes](#shapes)
- [Matrix addition](#matrix-addition)
- [Multiplication](#multiplication)
- [Transpose](#transpose)
- [Identity](#identity)
- [Inverse concept](#inverse-concept)
- [Systems of equations](#systems-of-equations)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a matrix toolkit and a dense layer](#project-a-matrix-toolkit-and-a-dense-layer)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A matrix is a rectangle of numbers arranged in rows and columns. It looks like a spreadsheet, and that is a good place to start. But the reason matrices matter in AI is that a whole collection of data can be handled with one symbol and one operation. A table of 10,000 customers with 20 features each is a matrix with shape 10,000 by 20. A neural-network layer that turns those 20 features into 5 hidden values is a matrix with shape 20 by 5. Applying the layer to all 10,000 customers at once is a single matrix multiplication.

Matrix multiplication is also the most time-consuming operation in modern machine learning. Training a large language model is mostly matrix multiplication running on GPUs, and the reason GPUs are used is that they are built to do many dot products in parallel. Understanding the shape rules and the row-by-column pattern of the product is what lets you read a model's architecture, predict its memory use and find the bug when two tensors do not line up.

On the job, matrices show up as the dataset that a pandas DataFrame converts into a NumPy array before training, as the weights of a linear layer in PyTorch, as the design matrix of a regression, and as the table of transformations that a computer vision pipeline applies to every image. Today you learn the operations on those tables and build a small piece of a neural network from them.

---

## Glossary

- **Matrix**: a rectangular grid of numbers with a fixed number of rows and columns. Its entries are indexed by row and column.
- **Entry (element)**: one number inside a matrix, identified by its row and column position.
- **Shape**: the pair (number of rows, number of columns). In NumPy it is stored as `A.shape`.
- **Square matrix**: a matrix with the same number of rows and columns.
- **Row vector and column vector**: matrices with one row or one column. A vector from Day 8 is one of these, depending on how you write it.
- **Matrix addition**: adding two matrices of the same shape entry by entry.
- **Matrix multiplication**: a row-by-column operation in which each entry of the result is a dot product of a row of the first matrix and a column of the second.
- **Transpose**: the matrix you get by swapping rows and columns, written `A^T`.
- **Identity matrix**: a square matrix with ones on the main diagonal and zeros elsewhere, written `I`. Multiplying by it changes nothing.
- **Inverse**: for a square matrix `A`, the matrix `A^-1` that undoes `A`, so that `A^-1 A = I`.
- **Singular matrix**: a square matrix with no inverse. Its columns depend on each other in a way that loses information.
- **System of linear equations**: several linear equations solved together. In matrix form it is written `Ax = b`.
- **Residual**: the difference `A x - b` between the two sides of a system after you plug in a candidate solution. A residual near zero means the solution works.
- **Dense layer**: a neural-network layer in which every input connects to every output, computed as `X W + b`.

---

### Matrices

Think of a spreadsheet. Each row is a record, such as a customer, and each column is a field, such as age or income. A matrix is the same kind of grid, but with no labels attached. The labels live in your code and your documentation, and the matrix holds only numbers.

A matrix with `m` rows and `n` columns has `m * n` entries. The entry in row `i` and column `j` is written `A[i, j]` in code or `a_ij` in mathematics. Rows are counted from the top and columns from the left, starting at zero in Python and one in mathematical notation. Two matrices are equal only when they have the same shape and the same entries.

```python
import numpy as np

A = np.array([
    [1.0, 2.0, 3.0],
    [4.0, 5.0, 6.0],
])
print(A[0, 2])   # 3.0, row 0 and column 2
print(A[1])      # [4. 5. 6.], the whole second row
print(A[:, 0])   # [1. 4.], the whole first column
```

Mixing up the order of indices is a frequent error. In mathematics, the entry `a_23` is in row 2 and column 3, counting from one. In NumPy, the same entry is `A[1, 2]`, because indexing starts at zero. Writing `A[2, 3]` by mistake either fails with an index error or silently reads a different entry, so the zero-based rule has to be checked every time. Write the meaning of each axis in a comment when you are unsure. In industry, the convention for a dataset is rows as samples and columns as features. Most libraries follow it, and the mistake of transposing the data by accident is one of the most common reasons a model trains on nonsense.

### Shapes

A shape is the size of a box that holds the numbers. Before you multiply two boxes together, you check that their sizes fit. Most errors in machine learning code are shape errors that the computer catches only at runtime, so checking sizes is a habit worth building early.

From first principles, a matrix of shape `(m, n)` has `m` rows and `n` columns. Two matrices can be multiplied only when the number of columns of the first equals the number of rows of the second. If `A` has shape `(m, n)` and `B` has shape `(n, p)`, then `A B` has shape `(m, p)`. The inner dimensions `n` must match, and the outer dimensions `m` and `p` give the shape of the result. A vector has one dimension, so its shape is `(n,)` in NumPy, which is neither a row nor a column until you add an axis.

```python
import numpy as np

A = np.zeros((2, 3))
B = np.zeros((3, 4))
print((A @ B).shape)  # (2, 4)

v = np.zeros(3)
print(v.shape)                 # (3,)
print(v.reshape(3, 1).shape)   # (3, 1), an explicit column
```

A common mistake is multiplying in the wrong order. `A` with shape `(2, 3)` times `B` with shape `(3, 4)` works, but `B` times `A` does not, because the inner dimensions are 4 and 2. NumPy raises a `ValueError` that names the mismatch, and reading that message is a skill you should practice. Another mistake is confusing the shape of a batch with the shape of a single example. A batch of 32 images of size 28 by 28 has shape `(32, 28, 28)`, and a layer that expects flattened inputs needs `(32, 784)`. In the real market, debugging a model usually starts with printing shapes at each stage of the forward pass, and the shape a teammate writes in a docstring is often the fastest way to find the bug.

### Matrix addition

Imagine two spreadsheets with the same layout. One holds the sales forecast for each product and region, and the other holds the adjustments from the regional managers. Adding them cell by cell gives the adjusted forecast. Matrix addition works the same way.

If `A` and `B` both have shape `(m, n)`, then `(A + B)[i, j] = A[i, j] + B[i, j]` for every position. The shapes must match exactly. Scalar multiplication works entry by entry too, so `c A` multiplies every entry by `c`. Together these operations make matrices a vector space, which is the reason the rules you learned for vectors on Day 8 carry over.

NumPy adds a useful extra rule called broadcasting. When you add a vector of shape `(n,)` to a matrix of shape `(m, n)`, NumPy repeats the vector for each row. Neural networks use this to add a bias to every example in a batch.

```python
import numpy as np

A = np.array([[1.0, 2.0], [3.0, 4.0]])
B = np.array([[10.0, 20.0], [30.0, 40.0]])
print(A + B)              # [[11. 22.] [33. 44.]]

bias = np.array([0.5, -1.0])
print(A + bias)           # the bias is added to each row
```

A common mistake is relying on broadcasting without checking the result. The expression `A + bias` works when `bias` has length `n`, but if you pass a column of shape `(m, 1)` by mistake, NumPy broadcasts it across the columns instead, and the answer has the right shape with the wrong values. Print a small example and check one entry by hand. Another limitation is that matrix addition has no meaning when the rows and columns do not correspond, even if the shapes happen to match. Adding a sales table to a table of temperatures produces a number that means nothing. In industry, residual connections in deep networks add the input of a block to its output, and that is matrix addition on tensors of identical shape.

### Multiplication

Matrix multiplication looks strange the first time you see it, and the reason is that it is designed for composing transformations and combining tables, not for multiplying entries. The rule is simple to state: the entry in row `i` and column `j` of `A B` is the dot product of row `i` of `A` and column `j` of `B`.

From first principles, if `A` has shape `(m, n)` and `B` has shape `(n, p)`, then `C = A B` has entries `C[i, j] = sum over k of A[i, k] * B[k, j]`. That sum is the dot product you learned on Day 8, and the summation notation from Day 7 is the formula. Each entry of the result is one dot product, so the product is `m * p` dot products, each of length `n`. Matrix multiplication is associative, so `(A B) C = A (B C)`, and it is distributive over addition. It is not commutative: in general `A B` is different from `B A`, and when the shapes differ one of the products may not exist at all.

Here is the product written out with an explicit loop, so you can see the dot products:

```python
import numpy as np

def matmul(A, B):
    n, k = A.shape
    k2, m = B.shape
    if k != k2:
        raise ValueError(f"inner dimensions differ: {k} vs {k2}")
    C = np.zeros((n, m))
    for i in range(n):
        for j in range(m):
            C[i, j] = sum(A[i, t] * B[t, j] for t in range(k))
    return C

A = np.array([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]])
B = np.array([[1.0, 0.0], [0.0, 1.0], [2.0, 2.0]])
print(matmul(A, B))   # [[ 7.  8.] [16. 17.]]
print(np.allclose(matmul(A, B), A @ B))  # True
```

Check the first entry by hand. Row 0 of `A` is `(1, 2, 3)` and column 0 of `B` is `(1, 0, 2)`. Their dot product is `1*1 + 2*0 + 3*2 = 7`, which matches the result. The `@` operator in NumPy computes the same thing much faster, and it should be your default in real code.

A common mistake is writing `*` where you mean matrix multiplication. In NumPy, `A * B` multiplies entries and requires the same shape, while `A @ B` performs the row-by-column product. Confusing them produces a result with a valid shape and wrong values. The second mistake is assuming commutativity. `A @ B` and `B @ A` have different shapes in general, and even when both exist they are usually different. In the market, a neural layer is `X @ W`, where `X` is a batch of inputs and `W` is the weight matrix. Reversing the order would fail in most cases, and the code that does work with the reversed order usually computes something a model was never trained to compute.

### Transpose

Hold a notebook with a table written across the top row and flip it over its diagonal. The rows become columns and the columns become rows. That flip is the transpose, and it is one of the most used operations in linear algebra and in machine learning code.

The transpose of `A` with shape `(m, n)` is `A^T` with shape `(n, m)`, where `(A^T)[j, i] = A[i, j]`. The transpose of a column vector is a row vector, and the transpose of a transpose returns the original matrix. Transposing a product reverses the order: `(A B)^T = B^T A^T`. That rule is useful when you derive gradients, and it explains why many formulas flip the order of their factors.

```python
import numpy as np

A = np.array([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]])
print(A.shape)     # (2, 3)
print(A.T.shape)   # (3, 2)
print(A.T)

B = np.array([[1.0, 0.0], [0.0, 1.0], [2.0, 2.0]])
print(np.allclose((A @ B).T, B.T @ A.T))  # True
```

One frequent error is forgetting that a transpose is needed to make shapes match. A weight matrix for a layer that maps 3 inputs to 2 outputs can be stored as shape `(3, 2)` or `(2, 3)`, and the code must agree with the convention used by its library. PyTorch's `torch.nn.Linear` stores its weight with shape `(out_features, in_features)`, so the forward pass multiplies by the transpose of that stored matrix. Read the documentation of the library you use before you write a formula by hand. In industry, almost every bug that involves a layer and its input comes down to this convention, and the transposed weight is the first thing to check when a model's output has the wrong shape.

### Identity

The number 1 has a special property: multiplying anything by 1 leaves it unchanged. The identity matrix plays the same role for matrices. It is a square matrix with ones on the main diagonal, from the top left to the bottom right, and zeros everywhere else.

From first principles, the identity matrix `I` of size `n` satisfies `I A = A` and `A I = A` for every matrix `A` of compatible shape. The size is important: for a matrix `A` with shape `(m, n)`, you need `I_n` on the right and `I_m` on the left. Each column of `I` is a standard basis vector, which is the same idea as the unit vectors of Day 8 stacked side by side.

```python
import numpy as np

I = np.eye(3)
print(I)

A = np.array([[2.0, 5.0, 1.0], [0.0, 3.0, 4.0], [7.0, 1.0, 2.0]])
print(np.allclose(I @ A, A))   # True
print(np.allclose(A @ I, A))   # True
```

Confusing the identity with a matrix of ones is an easy error. `np.ones((3, 3))` is a full grid of ones, and multiplying by it sums each row, which is not the identity. Another mistake is building an identity of the wrong size for a non-square product. In practice, the identity matters because it is the starting point for the inverse and for regularized solutions, where you add a small multiple of the identity to a matrix before solving. In industry, ridge regression adds `lambda I` to the matrix `X^T X` before inverting it, and that single term is what makes the solution stable when features are correlated.

### Inverse concept

Undoing an operation is a common idea. If you multiply a number by 4, you undo it by dividing by 4. The inverse of a matrix plays the same role: if `A` transforms a vector, `A^-1` transforms it back. For a square matrix `A`, the inverse is the matrix that satisfies `A^-1 A = I` and `A A^-1 = I`.

From first principles, not every square matrix has an inverse. For a `2` by `2` matrix `[[a, b], [c, d]]`, the inverse exists when the determinant `a d - b c` is not zero, and then `A^-1 = (1 / (a d - b c)) [[d, -b], [-c, a]]`. If the determinant is zero, the matrix is singular. Its columns point in a way that squashes some direction to zero, and information is lost. You cannot recover the lost information by any matrix, so no inverse exists. Rank, which you meet on Day 10, is the general version of this test.

```python
import numpy as np

M = np.array([[4.0, 7.0], [2.0, 6.0]])
print(np.linalg.det(M))           # about 10.0
M_inv = np.linalg.inv(M)
print(np.round(M_inv, 2))         # [[ 0.6 -0.7] [-0.2  0.4]]
print(np.allclose(M_inv @ M, np.eye(2)))  # True

S = np.array([[1.0, 2.0], [2.0, 4.0]])
try:
    np.linalg.inv(S)
except np.linalg.LinAlgError as err:
    print("error:", err)   # Singular matrix
```

Computing an inverse when you only need to solve a system is a common error. Computing `A^-1 b` explicitly is slower and less accurate than calling `np.linalg.solve(A, b)`, which uses a factorization that avoids forming the inverse. A second mistake is believing that a determinant near zero is harmless. A matrix can be invertible in theory and still be so close to singular that rounding errors dominate the answer. Such matrices are called ill-conditioned, and `np.linalg.cond` measures how badly. In industry, linear regression solved with the normal equations involves `X^T X`, and when two features are nearly the same, that matrix becomes ill-conditioned, which is why production code uses regularization or a least-squares solver.

### Systems of equations

Consider two equations: `4x + 7y = 18` and `2x + 6y = 14`. Each equation describes a line on a plane. The solution is the point where the lines cross. A system of linear equations is a set of such constraints, and solving it means finding the values that satisfy all of them at once.

From first principles, write the coefficients as a matrix `A`, the unknowns as a vector `x` and the right-hand sides as a vector `b`. The system becomes `A x = b`. For a square system with an invertible `A`, the unique solution is `x = A^-1 b`. The geometry is the same in any dimension: each equation is a flat surface, and the solution is where they all meet. When `A` is singular, the surfaces may be parallel and never meet, or they may coincide and meet everywhere. Either way there is no single answer.

```python
import numpy as np

A = np.array([[4.0, 7.0], [2.0, 6.0]])
b = np.array([18.0, 14.0])
x = np.linalg.solve(A, b)
print(x)                  # [1. 2.]
print(A @ x - b)          # residual, close to [0. 0.]
```

The residual `A x - b` is the check you should always compute. A solution with a large residual is wrong, and a solution with a tiny residual is right only if the matrix is well-conditioned. When there are more equations than unknowns, as in a regression with many data points, the system usually has no exact solution. Then you look for the `x` that makes the residual as small as possible, which is the least-squares problem. NumPy's `np.linalg.lstsq` solves it.

A common mistake is treating an overdetermined system as if it had an exact answer, or trusting a solution without checking its residual. Another limitation is that the equations must be linear. Real problems with curves or interactions need a different model, and forcing them into a linear system gives a poor fit. In the market, nearly every regression model in an industrial forecasting pipeline is solved as a least-squares system, whether the code calls `lstsq`, a closed-form normal equation or an iterative solver.

---

## Hands-on exercises

Do the first four by hand, then check each one in Python.

1. Multiply `A = [[1, 2], [3, 4]]` by `B = [[0, 1], [1, 0]]` by hand. Then compute `B A`. Confirm the two products differ and explain what `B` does to the rows of `A`.
2. Compute the product of a `(2, 3)` matrix and a `(3, 1)` matrix by hand, and write its shape. Then try the product in the other order, predict the error and check your prediction in Python.
3. Compute the transpose of `[[1, 2, 3], [4, 5, 6]]` by hand. Verify that `(A B)^T = B^T A^T` for `A = [[1, 2], [3, 4]]` and `B = [[5, 6], [7, 8]]`.
4. Find the inverse of `[[2, 1], [5, 3]]` using the `2` by `2` formula, verify that `A^-1 A = I`, and compute the determinant. Then find a `2` by `2` matrix with determinant zero and show that `np.linalg.inv` raises an error for it.
5. Write `matmul` from the lesson and time it against `A @ B` for two random `200` by `200` matrices. Report the ratio of the two times and explain why the loop is slower.
6. Build a `(4, 3)` data matrix with one row per sample and three features, multiply it by a `(3, 2)` weight matrix, add a bias of length 2, and print the output shape. Change the bias to length 3 and read the error message.
7. Solve `x + 2y = 5` and `3x + 4y = 6` with `np.linalg.solve`. Compute the residual, then change the second equation to `6x + 12y = 6` and explain what happens to the solver.

Write down each step that surprised you. The places where your hand calculation disagreed with the code are worth keeping as notes.

---

## Project: a matrix toolkit and a dense layer

This is a practical mini-project. You will implement matrix multiplication from scratch, verify it against NumPy, use it to compute the forward pass of a dense layer, and solve a small system with a residual check. The layer code is the same structure that every linear layer in a deep learning framework uses, so the shape checks you write here are the checks you will need in a real model.

**Repo name:** `100-days-ai-engineering-day-09-matrices`

**Goal:** implement matrix multiplication and a dense-layer forward pass with explicit shape checks, test them and document the shape rules in the README.

**Repo layout:**

```
100-days-ai-engineering-day-09-matrices/
├── README.md
├── requirements.txt
├── matrix_ops.py
├── dense_layer.py
├── solve_system.py
└── tests/
    └── test_matrix_ops.py
```

**Starter code for `matrix_ops.py`:**

```python
import numpy as np


def matmul(A, B):
    """Multiply A (m, n) by B (n, p) with an explicit loop. Raise ValueError on a shape mismatch."""
    raise NotImplementedError
```

**Starter code for `dense_layer.py`:**

```python
import numpy as np
from matrix_ops import matmul


def relu(Z):
    """Return the elementwise maximum of Z and 0."""
    raise NotImplementedError


def dense_forward(X, W, b):
    """Return relu(X W + b) for X (batch, in), W (in, out) and b (out,).

    Raise ValueError if the shapes do not fit together.
    Use matmul from matrix_ops for the product.
    """
    raise NotImplementedError
```

**Starter code for `solve_system.py`:**

```python
import numpy as np


def solve_with_residual(A, b):
    """Solve A x = b and return x and the maximum absolute residual of A x - b."""
    raise NotImplementedError


if __name__ == "__main__":
    A = np.array([[4.0, 7.0], [2.0, 6.0]])
    b = np.array([18.0, 14.0])
    x, res = solve_with_residual(A, b)
    print("x =", x)
    print("max residual =", res)
```

**Starter code for `tests/test_matrix_ops.py`:**

```python
import sys
from pathlib import Path

import numpy as np
import pytest

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

from matrix_ops import matmul
from dense_layer import relu, dense_forward
from solve_system import solve_with_residual


def test_matmul_matches_numpy_on_random_input():
    rng = np.random.default_rng(0)
    A = rng.standard_normal((4, 3))
    B = rng.standard_normal((3, 5))
    assert np.allclose(matmul(A, B), A @ B)


def test_matmul_rejects_bad_shapes():
    with pytest.raises(ValueError):
        matmul(np.zeros((2, 3)), np.zeros((2, 3)))


def test_relu_clips_negatives():
    assert np.array_equal(relu(np.array([-1.0, 0.0, 2.0])), [0.0, 0.0, 2.0])


def test_dense_output_shape():
    X = np.zeros((5, 3))
    W = np.zeros((3, 4))
    b = np.zeros(4)
    assert dense_forward(X, W, b).shape == (5, 4)


def test_dense_rejects_wrong_bias_length():
    with pytest.raises(ValueError):
        dense_forward(np.zeros((5, 3)), np.zeros((3, 4)), np.zeros(3))


def test_solve_known_system():
    A = np.array([[4.0, 7.0], [2.0, 6.0]])
    b = np.array([18.0, 14.0])
    x, res = solve_with_residual(A, b)
    assert np.allclose(x, [1.0, 2.0])
    assert res < 1e-9
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy` and `pytest`.
2. Implement `matmul` with the loop from the lesson. Check that `n` and `k` are compared before the loop runs.
3. Run `pytest` and make the matmul tests pass.
4. Implement `relu` and `dense_forward`. In `dense_forward`, check that `X.shape[1] == W.shape[0]` and that `b.shape == (W.shape[1],)`, and raise `ValueError` with a clear message if either check fails.
5. Implement `solve_with_residual` using `np.linalg.solve`, and return the maximum absolute value of `A x - b`.
6. Run `pytest` until all six tests pass. Then run `python solve_system.py` and check that the printed solution matches the one you computed on paper.
7. Add a short script at the bottom of `dense_layer.py` that builds a batch of four random inputs with three features, applies a layer with two outputs and prints the shapes at each step.
8. Write the README with a shape table for every function, commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is written in one sentence
- [ ] Install and run commands are listed (`pip install -r requirements.txt`, `pytest`, `python solve_system.py`)
- [ ] A shape table lists the input and output shapes of `matmul`, `dense_forward` and `solve_with_residual`
- [ ] The README explains why `*` and `@` give different results in NumPy
- [ ] One limitation is named, such as the loop being slow or the residual test not catching an ill-conditioned matrix

**Done when:**

- `pytest` passes all six tests
- `dense_forward` raises a clear `ValueError` when the bias length does not match the output size
- `python solve_system.py` prints the solution `[1, 2]` and a residual near zero
- The shape table in the README matches what the code prints
- The commit history shows the work in small steps, with one commit per function

**What to commit as evidence:** the source files, the test file, the `pytest` output pasted into the README, and the printed shapes from the layer script.

---

## Check your understanding

1. A batch of 64 examples with 20 features goes through a layer with 8 outputs. Write the shape of the input, the weight matrix and the output, and say why the weight matrix must have shape `(20, 8)` for the product to work.
2. A teammate computes `B @ A` where `A` has shape `(2, 3)` and `B` has shape `(3, 4)`, and the code raises an error. Explain why, and state the product that does work.
3. You find that `A @ B` and `B @ A` give different results for two square matrices. Explain which property of matrix multiplication this shows, and why it matters when you compose two neural layers.
4. An engineer computes `np.linalg.inv(X.T @ X) @ X.T @ y` for a regression and gets huge coefficients. Suggest two causes and one fix that uses the identity matrix.
5. A system of three equations has two unknowns. Explain why it usually has no exact solution, and name the method that finds the best approximate answer.
6. A matrix has determinant 0.0000001. Is it singular? Explain why a tiny determinant can still give unreliable results, and name the NumPy function that measures this.
7. A colleague replaces `np.linalg.solve(A, b)` with `np.linalg.inv(A) @ b` to make the code clearer. Describe one risk this introduces, and explain why the solver call is preferred.

### Answers

1. The input is `(64, 20)`, the weights are `(20, 8)` and the output is `(64, 8)`. The inner dimensions 20 and 20 must match, and the outer dimensions give the output shape.
2. `B` has shape `(3, 4)` and `A` has shape `(2, 3)`, so the number of columns of `B` (4) does not equal the number of rows of `A` (2). The product that works is `A @ B`, with shape `(2, 4)`.
3. It shows that matrix multiplication is not commutative. When you compose two layers, the order of the transformations matters, so swapping them changes what the network computes.
4. The matrix `X^T X` may be singular or ill-conditioned, for example when two features are nearly identical, which makes its inverse amplify rounding errors. A fix is to add a small multiple of the identity, `lambda I`, before inverting, which is ridge regression, or to use `np.linalg.lstsq` directly.
5. Each equation is a constraint, and three constraints on two unknowns usually cannot all be met exactly. The least-squares method finds the values that make the residual as small as possible.
6. A determinant of 0.0000001 is not exactly zero, so the matrix is technically invertible, but it is nearly singular and small rounding errors can change the answer a great deal. The NumPy function that measures this is `np.linalg.cond`, which reports the condition number, and a large value signals an unreliable solution.
7. The inverse is computed explicitly, which is slower and loses more precision than the factorization inside `solve`. For a singular or near-singular matrix, the inverse call can produce huge or meaningless values with no error, while `solve` reports the problem. Prefer `np.linalg.solve` for solving a system.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Matrix (mathematics)](https://en.wikipedia.org/wiki/Matrix_(mathematics))
- Wikipedia: [Matrix multiplication](https://en.wikipedia.org/wiki/Matrix_multiplication)
- Wikipedia: [Transpose](https://en.wikipedia.org/wiki/Transpose)
- Wikipedia: [Identity matrix](https://en.wikipedia.org/wiki/Identity_matrix)
- Wikipedia: [Invertible matrix](https://en.wikipedia.org/wiki/Invertible_matrix)
- Wikipedia: [System of linear equations](https://en.wikipedia.org/wiki/System_of_linear_equations)
- Stanford CS229, "Linear Algebra Review and Reference" by Zico Kolter, the course notes on matrices and their operations. Search the title with "Stanford CS229" to find the current PDF.
- NumPy documentation, "NumPy quickstart" and the reference for `numpy.linalg` on numpy.org.

**Tutorials**

- GeeksforGeeks, search "matrix multiplication" and "transpose of a matrix" for worked examples with shape rules.
- PyTorch documentation, "torch.nn.Linear", which shows how a framework stores the weight matrix of a dense layer.

**Watch**

- 3Blue1Brown (YouTube), "Matrix multiplication as composition | Chapter 4, Essence of linear algebra". It explains why the product is defined the way it is.
- 3Blue1Brown (YouTube), "Inverse matrices, column space and null space | Chapter 7, Essence of linear algebra". It explains the inverse and singular matrices visually.
- MIT OpenCourseWare, "18.06 Linear Algebra" by Prof. Gilbert Strang. Search the course on ocw.mit.edu and watch the early lectures on matrix multiplication and elimination.

**Practice**

- Khan Academy, "Linear algebra" course, the units on matrices and systems of equations, with interactive exercises.

---

## Tomorrow

Tomorrow you ask what a matrix does to space. Every matrix you met today is a linear transformation: it moves vectors, stretches them, flips them or squashes them. You will learn what a basis is, how the rank of a matrix tells you how many independent directions a dataset really uses, and why orthogonal directions are so valuable. Those ideas lead to eigenvalues, the singular value decomposition and dimensionality reduction, which are the tools that compress large feature sets and reveal their structure.

**Next:** [Day 10: Linear Algebra for ML](Day-10-Linear-Algebra-for-ML.md)

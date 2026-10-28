# Day 8: Vectors

*100 Days of AI Engineering, Module 2: Mathematics for AI*

**Time:** about 3 hours
**Level:** beginner. You should be comfortable with Day 7's functions and sums. No geometry beyond the Pythagorean theorem is needed.

**By the end of this day, you will have:**

- A small Python module that implements vector addition, scalar multiplication, magnitude, unit vectors, the dot product, angles and projections, all with NumPy
- A test file that checks each operation against known answers and against NumPy's built-in functions
- A short written explanation of why cosine similarity compares direction and ignores length, which you will reuse when you build a search system later in the series

**What you need:**

- A computer with Python 3.10 or newer
- NumPy 2.0 or newer (`pip install numpy`)
- pytest 8.0 or newer (`pip install pytest`), used to run the tests
- A code editor such as VS Code (free) or any editor you already use
- Git and a free GitHub account for the project section
- Paper and a pencil for the sketches in the exercises

**Prerequisites:**

- Day 7: functions, sums and the square root
- Basic Python: lists, functions, `import`, and running a script from the terminal
- The Pythagorean theorem: for a right triangle, the square of the hypotenuse equals the sum of the squares of the other two sides

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Scalars](#scalars)
- [Vectors](#vectors)
- [Vector addition](#vector-addition)
- [Scalar multiplication](#scalar-multiplication)
- [Magnitude](#magnitude)
- [Unit vectors](#unit-vectors)
- [Dot products](#dot-products)
- [Geometric interpretation](#geometric-interpretation)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a vector toolkit with tests](#project-a-vector-toolkit-with-tests)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A single number can describe a temperature or a price. Most real things need several numbers at once. A house is described by its area, the number of bedrooms and its age. A customer is described by how often they buy, how much they spend and how long they have been subscribed. A sentence is described by a list of numbers called an embedding, which may have hundreds or thousands of entries. A vector is the mathematical object for an ordered list of numbers, and it is the basic data structure of machine learning.

Vectors also have a geometric meaning. A list of two numbers is a point on a plane, or an arrow from the origin to that point. A list of three numbers is a point in space. Once you can picture the arrow, operations such as adding two lists, measuring their length or asking how similar they are become visual ideas instead of arithmetic tricks.

On the job, vectors show up as feature rows in a training table, as embeddings returned by a model API, as gradients that an optimizer uses to update weights, and as the query and document representations that a retrieval system compares. When a teammate says "the cosine similarity between the query and the top result was 0.42," they are doing vector math. Today you learn the operations behind that sentence and implement them yourself.

---

## Glossary

- **Scalar**: a single number with no direction, such as a temperature or a learning rate.
- **Vector**: an ordered list of numbers, often written as a column, that can represent a point, an arrow or a feature set.
- **Dimension**: the number of entries in a vector. A vector with 3 entries is 3-dimensional.
- **Component**: one entry of a vector, identified by its position.
- **Vector addition**: the entry-by-entry sum of two vectors with the same dimension.
- **Scalar multiplication**: multiplying every entry of a vector by one number.
- **Magnitude**: the length of a vector, also called its norm. For Euclidean length, it is the square root of the sum of squared components.
- **Unit vector**: a vector with magnitude 1, used to represent a direction.
- **Dot product**: the sum of the products of matching components. It returns one number.
- **Orthogonal**: perpendicular. Two nonzero vectors are orthogonal when their dot product is 0.
- **Projection**: the shadow of one vector on the direction of another.
- **Cosine similarity**: the dot product divided by the product of the two magnitudes. It measures the angle between vectors, independent of their lengths.
- **Broadcasting**: NumPy's rule for combining arrays of different shapes, such as adding a scalar to every entry of a vector.

---

### Scalars

Picture a thermometer. It shows a single number, 21 degrees, and that number has no direction. Nothing about it points anywhere. A scalar is that kind of quantity: one number that can be added, subtracted and multiplied like any other number. The learning rate of a model is a scalar. So is the batch size, the loss value at one step, and the probability that an email is spam.

A scalar is an element of the real numbers, the set of values you can place on a number line. Scalars are called that because they scale the vectors they multiply. Writing a scalar in lowercase italic, such as `c`, separates it from a vector, which is usually written in bold or with an arrow.

In Python, the difference between a scalar and a one-element array is easy to see:

```python
import numpy as np

c = 2.5
v = np.array([2.5])

print(np.ndim(c))  # 0, a scalar has no dimensions
print(np.ndim(v))  # 1, a vector with one entry
print(c * np.array([1.0, 2.0]))  # [2.5 5. ], scalars broadcast
```

A common mistake is confusing a scalar with a vector that has one entry, or with a NumPy array of shape `()`. The results often look the same when printed, but the shapes differ, and shape mismatches are the most common bug in array code. Check shapes with `.shape` or `np.ndim` when a result looks wrong. In industry, a hyperparameter such as the learning rate is a scalar, and tuning libraries search over scalars. A mistake in its type can silently change the run.

### Vectors

A grocery receipt is a good analogy. It lists items in a fixed order: milk, bread, eggs. Each item has a quantity, and the order matters because the first number always refers to milk. A vector is an ordered list of numbers in the same spirit. The position is what gives each number its meaning.

Formally, a vector with `n` entries lives in an `n`-dimensional space and is written as `v = (v1, v2, ..., vn)`. The entries are called components. A house described by area in square meters, bedrooms and age in years is the vector `(120, 3, 15)`. The first component always means area, the second always means bedrooms, and the third always means age. Changing the order changes the meaning, even though the numbers are the same.

Geometrically, the vector `(3, 4)` is an arrow from the origin `(0, 0)` to the point with x-coordinate 3 and y-coordinate 4. The same arrow can be drawn anywhere on the plane as long as its length and direction are unchanged. In NumPy, a vector is a one-dimensional array:

```python
import numpy as np

house = np.array([120.0, 3.0, 15.0])  # area, bedrooms, age
print(house.shape)  # (3,)
print(house[0])     # 120.0
```

A common mistake is mixing features on very different scales. If area is measured in square meters and age in years, the area numbers dominate any calculation that uses distance. Scaling the features before comparing them is a standard fix, and you will see it in the Day 9 dataset exercises. Another mistake is assuming a vector has a column shape by default. In NumPy, `np.array([1, 2, 3])` has shape `(3,)`, which is neither a row nor a column, while `np.array([[1], [2], [3]])` has shape `(3, 1)`, which is a column. In production, a sentence embedding returned by a model API is a vector with a fixed dimension. BERT-base produces 768-dimensional vectors for each token, and every downstream system that stores or compares embeddings depends on that dimension staying the same.

### Vector addition

Think of two walks in a city. The first walk moves you 3 blocks east and 4 blocks north. The second moves you 1 block east and 2 blocks north. Walking the first and then the second brings you 4 blocks east and 6 blocks north. Adding the two walks is adding their components.

From first principles, vector addition adds matching components: `(u1, u2) + (v1, v2) = (u1 + v1, u2 + v2)`. The vectors must have the same dimension, since a component with no partner has no meaning. Addition is commutative, `u + v = v + u`, and associative. Geometrically, placing the tail of `v` at the tip of `u` gives the sum as the arrow from the origin to the new tip. Placing the two vectors tail to tail gives a parallelogram whose diagonal is the sum.

```python
import numpy as np

u = np.array([3.0, 4.0])
v = np.array([1.0, 2.0])
print(u + v)  # [4. 6.]
```

A common mistake is adding vectors of different lengths and expecting NumPy to pad with zeros. NumPy does not do that for vectors of shape `(2,)` and `(3,)`. It raises an error, which is the correct behavior. The more dangerous mistake is adding a vector to a matrix or to another vector with broadcasting in an unintended way. Broadcasting can silently add a vector to every row of a matrix, and the result may have the right shape but the wrong meaning. Always check the output shape. In industry, gradient accumulation adds gradient vectors from several mini-batches before an update, and the code depends on every batch's gradient having the same shape as the weight vector.

### Scalar multiplication

Imagine a map with a scale bar. If you double the scale, every distance on the map doubles, and the directions stay the same. Multiplying a vector by a scalar does the same thing: every component is multiplied by that number.

From first principles, `c * v = (c * v1, c * v2, ..., c * vn)`. For a positive `c`, the arrow gets longer or shorter but keeps its direction. For `c = 2`, the arrow doubles. For `c = 0.5`, it halves. For a negative `c`, the arrow flips to point the other way. For `c = -1`, you get the reversed vector, which is the basis of subtraction: `u - v` is `u + (-1) * v`. Multiplying by zero gives the zero vector, the only vector with magnitude zero.

```python
import numpy as np

v = np.array([3.0, 4.0])
print(2 * v)     # [6. 8.]
print(0.5 * v)   # [1.5 2. ]
print(-1 * v)    # [-3. -4.]
```

A common mistake is treating scalar multiplication as a change in direction. It changes length and possibly sign, but never the direction of a non-zero vector when `c` is positive. Another mistake is applying the scalar to only one component by accident, which happens in code when a loop updates some entries and forgets the rest. The gradient descent update `w = w - learning_rate * gradient` is scalar multiplication followed by vector subtraction. Almost every optimizer in deep learning is a variation of that single line applied to large vectors of weights.

### Magnitude

Put a ruler from the origin to the tip of a vector. Its length is the magnitude. For the vector `(3, 4)`, the ruler is the hypotenuse of a right triangle with legs 3 and 4, so its length is 5. The Pythagorean theorem gives the answer directly.

From first principles, the Euclidean magnitude, also called the L2 norm, of `v` is the square root of the sum of the squared components: `||v|| = sqrt(v1^2 + v2^2 + ... + vn^2)`. This is a sum from Day 7 wrapped in a square root. Magnitude is always zero or positive. It is zero only for the zero vector. Magnitude also has the property that scaling a vector by `c` scales its magnitude by the absolute value of `c`, which you can verify with the code below.

```python
import numpy as np

v = np.array([3.0, 4.0])
print(np.linalg.norm(v))  # 5.0
print(np.sqrt(np.sum(v ** 2)))  # 5.0, the same formula written out
print(np.linalg.norm(-2 * v))   # 10.0, twice the length
```

A common mistake is using the sum of absolute values, which is the L1 norm, when the formula calls for the Euclidean norm, or the reverse. Both are used in machine learning. L2 regularization penalizes large weights using squared values, and L1 regularization uses absolute values and tends to produce sparse weights. The second mistake is comparing magnitudes of vectors with different dimensions without thinking about scale. A vector with 768 entries will often have a larger magnitude than one with 3 entries, even if every entry is similar. In practice, magnitude appears in gradient clipping, where a training library rescales the gradient vector whenever its norm exceeds a threshold, and the threshold is a norm, not a single entry.

### Unit vectors

Suppose you give someone directions, but you do not care how far to walk, only which way to face. You want a unit vector. It is a vector scaled so that its length is exactly 1, which keeps the direction and removes the length.

The unit vector in the direction of `v` is `v / ||v||`. Dividing by the magnitude is the same as multiplying by `1 / ||v||`, a scalar, which is why the operation is legitimate. The result has magnitude 1, because `||v / ||v|| || = ||v|| / ||v|| = 1`. The vector `(3, 4)` has magnitude 5, so its unit vector is `(0.6, 0.8)`. The standard basis vectors `(1, 0)` and `(0, 1)` are unit vectors too, and they point along the axes.

```python
import numpy as np

v = np.array([3.0, 4.0])
u_hat = v / np.linalg.norm(v)
print(u_hat)                    # [0.6 0.8]
print(np.linalg.norm(u_hat))    # 1.0
```

A common mistake is normalizing the zero vector, which has no direction and gives a division by zero. NumPy returns `nan` with a warning, which can spread through a model unnoticed. Guard against it explicitly. Another mistake is normalizing when the length carries information. A house with a larger area vector is not just pointing in a different direction, it is bigger, and normalizing throws that away. In industry, normalizing embeddings before storing them in a vector database is common, because it lets a system use fast inner-product search as a cosine similarity search.

### Dot products

Suppose you are shopping, and each item has a price and a quantity. The total cost is the sum of price times quantity for each item. That total is a dot product. It pairs up matching entries, multiplies each pair and adds the results.

Formally, `u . v = u1*v1 + u2*v2 + ... + un*vn`. The dot product takes two vectors of the same dimension and returns a single scalar. It is symmetric, so `u . v = v . u`, and it is linear, so `(c*u) . v = c*(u . v)`. The geometric meaning, covered in the next section, is `u . v = ||u|| ||v|| cos(theta)`, where theta is the angle between the two vectors. That single equation is the reason the dot product is so widely used. It measures how much two vectors point in the same direction.

```python
import numpy as np

u = np.array([3.0, 4.0])
v = np.array([1.0, 2.0])
print(np.dot(u, v))  # 3*1 + 4*2 = 11.0
print(u @ v)         # the same operation, written with @
```

From the dot product, cosine similarity follows directly: divide by both magnitudes. For `u` and `v` above, the cosine is `11 / (5 * sqrt(5))`, which is about 0.984. The vectors point in nearly the same direction, even though `v` is shorter than `u` in one component and longer in another.

```python
cos = np.dot(u, v) / (np.linalg.norm(u) * np.linalg.norm(v))
print(round(float(cos), 4))  # 0.9839
```

A common mistake is using the dot product as a similarity score without normalizing. The dot product grows with the length of both vectors, so a long document with ordinary word counts scores higher against everything. Use cosine similarity when only direction matters. Another mistake is confusing `*` with `@` in NumPy. For two one-dimensional arrays, `*` multiplies entry by entry and returns a vector, not a scalar. The dot product needs `np.dot` or `@`. In production, search and recommendation systems compare embedding vectors with the dot product or cosine similarity, and a retrieval-augmented generation pipeline uses that comparison to decide which passages to send to a language model.

### Geometric interpretation

A shadow is a good picture of what the dot product measures. At noon, the sun is straight overhead, and your shadow is as long as you are tall. In the evening, the sun is low and the shadow stretches or disappears. The shadow of one arrow on another arrow's direction is the projection, and its length depends on the angle between them.

From first principles, the projection of `v` onto `u` is the length `(u . v) / ||u||` along the direction of `u`. Multiply that length by the unit vector `u / ||u||` and you get the projection vector: `proj_u(v) = ((u . v) / (u . u)) * u`. When the vectors are orthogonal, the projection is the zero vector and the dot product is zero. When they point in the same direction, the dot product is positive and the projection is `v` itself, shrunk or stretched to lie along `u`. When they point in opposite directions, the dot product is negative.

Here is the projection of `v = (1, 2)` onto `u = (3, 4)`:

```python
import numpy as np

u = np.array([3.0, 4.0])
v = np.array([1.0, 2.0])
proj = (np.dot(u, v) / np.dot(u, u)) * u
print(proj)  # [1.32 1.76]
```

The angle itself comes from the cosine formula, and NumPy's `arccos` returns it in radians. For these two vectors the angle is about 10.3 degrees. Sketch the two arrows on graph paper and measure the angle with a protractor to confirm it.

Assuming that geometric intuition from two or three dimensions carries over unchanged to hundreds of dimensions is a common mistake. In high dimensions, random vectors are almost always nearly orthogonal, and the distances between random points become similar. This is one reason nearest-neighbor search in embedding spaces needs careful indexing and why the choice of similarity measure matters. The limitation is real: a picture helps you reason about two dimensions, and the algebra is what you trust for hundreds. In industry, the geometry of projection is behind principal component analysis, which you will meet on Day 10, and behind the least-squares fits that statisticians and engineers use every day.

---

## Hands-on exercises (about 60 minutes)

Do the first four on paper, then check each one in Python.

1. Draw the vectors `u = (2, 1)` and `v = (1, 3)` on graph paper, tail at the origin. Draw `u + v` by placing the tail of `v` at the tip of `u`. Compute `u + v` numerically and confirm the tip matches your drawing.
2. Compute `3u`, `-v` and `u - v` by hand. Verify each with NumPy and explain in one sentence what `-v` does geometrically.
3. Find the magnitude of `(1, 2, 2)` by hand. Verify with `np.linalg.norm`. Then find the unit vector in the same direction and confirm its magnitude is 1.
4. Compute `u . v` for `u = (2, 1)` and `v = (1, 3)`, then compute `u . v` for `u = (1, 0)` and `v = (0, 5)`. Explain why the second result is zero using the cosine formula.
5. Write a function `cos_sim(u, v)` that returns the cosine similarity of two vectors. Test it on `(1, 0)` and `(0, 1)` (expected 0), on `(1, 1)` and `(2, 2)` (expected 1), and on `(1, 0)` and `(-1, 0)` (expected -1). Check that your function does not change when you multiply one input by 100.
6. Scale the features of a small dataset. Create three house vectors `(120, 3, 15)`, `(80, 2, 40)` and `(200, 4, 5)`. Compute the magnitude of each raw vector, then divide each feature column by its maximum value and recompute. Explain which feature dominated the raw magnitudes and why.
7. Explain in two or three sentences why normalizing a vector to unit length before the dot product gives cosine similarity.

Keep a short note of every step that surprised you. The surprises are the parts you will remember.

---

## Project: a vector toolkit with tests

This is a practical mini-project. You will build a small module of vector functions, test it against known answers and against NumPy, and commit the result. The module is the same set of operations that a retrieval or recommendation system runs millions of times per day, so writing it yourself is the fastest way to trust the library calls later.

**Repo name:** `100-days-ai-engineering-day-08-vectors`

**Goal:** implement seven vector operations from scratch with NumPy, test them, and document where each one appears in a machine learning system.

**Repo layout:**

```
100-days-ai-engineering-day-08-vectors/
├── README.md
├── requirements.txt
├── vectors.py
├── demo.py
└── tests/
    └── test_vectors.py
```

**Starter code for `vectors.py`:**

```python
import numpy as np


def add(u, v):
    """Return the vector sum u + v. Raise ValueError if the shapes differ."""
    raise NotImplementedError


def scale(c, v):
    """Return the vector c * v."""
    raise NotImplementedError


def magnitude(v):
    """Return the Euclidean length of v."""
    raise NotImplementedError


def unit(v):
    """Return v divided by its magnitude. Raise ValueError for the zero vector."""
    raise NotImplementedError


def dot(u, v):
    """Return the dot product of u and v as a float. Raise ValueError if the shapes differ."""
    raise NotImplementedError


def cosine_similarity(u, v):
    """Return the cosine of the angle between u and v. Raise ValueError if either is zero."""
    raise NotImplementedError


def project(v, onto):
    """Return the projection of v onto the direction of `onto`."""
    raise NotImplementedError
```

**Starter code for `tests/test_vectors.py`:**

```python
import numpy as np
import pytest

from vectors import add, scale, magnitude, unit, dot, cosine_similarity, project


def test_add_matches_known_answer():
    assert np.allclose(add(np.array([3.0, 4.0]), np.array([1.0, 2.0])), [4.0, 6.0])


def test_add_rejects_different_shapes():
    with pytest.raises(ValueError):
        add(np.array([1.0, 2.0]), np.array([1.0, 2.0, 3.0]))


def test_scale_by_negative_one_reverses_vector():
    assert np.allclose(scale(-1, np.array([3.0, 4.0])), [-3.0, -4.0])


def test_magnitude_of_3_4_is_5():
    assert magnitude(np.array([3.0, 4.0])) == pytest.approx(5.0)


def test_unit_has_length_one():
    assert magnitude(unit(np.array([3.0, 4.0]))) == pytest.approx(1.0)


def test_unit_rejects_zero_vector():
    with pytest.raises(ValueError):
        unit(np.zeros(3))


def test_dot_matches_numpy():
    u = np.array([3.0, 4.0])
    v = np.array([1.0, 2.0])
    assert dot(u, v) == pytest.approx(np.dot(u, v))


def test_orthogonal_vectors_have_zero_dot():
    assert dot(np.array([1.0, 0.0]), np.array([0.0, 1.0])) == pytest.approx(0.0)


def test_cosine_is_scale_invariant():
    u = np.array([1.0, 2.0, 3.0])
    v = np.array([3.0, 1.0, 2.0])
    assert cosine_similarity(u, v) == pytest.approx(cosine_similarity(100 * u, v))


def test_cosine_of_opposite_vectors_is_minus_one():
    assert cosine_similarity(np.array([1.0, 0.0]), np.array([-1.0, 0.0])) == pytest.approx(-1.0)


def test_projection_matches_known_answer():
    assert np.allclose(project(np.array([1.0, 2.0]), np.array([3.0, 4.0])), [1.32, 1.76])
```

**Step-by-step tasks:**

1. Create the repo on GitHub, clone it and add the layout above. Add `requirements.txt` with `numpy` and `pytest`.
2. Implement `add`, `scale`, `magnitude`, `unit` and `dot` in `vectors.py`. Each function should check shapes or zero vectors and raise `ValueError` where the docstring says so.
3. Run `pytest` and make the first five tests pass.
4. Implement `cosine_similarity` by dividing `dot` by the product of the magnitudes. Clip the result to the range from -1 to 1 before returning it, since rounding can push a value slightly outside that range.
5. Implement `project` using the formula `((u . v) / (u . u)) * u`, where `u` is `onto` and `v` is the vector being projected.
6. Run `pytest` until all tests pass.
7. Write `demo.py` that builds three embedding-like vectors, prints their pairwise cosine similarities and shows which pair is most similar.
8. Write the README, commit the files and push to GitHub.

**README checklist:**

- [ ] The goal is written in one sentence
- [ ] Install and run commands are listed (`pip install -r requirements.txt`, `pytest`, `python demo.py`)
- [ ] Each function has one line that says what it computes
- [ ] The README explains why cosine similarity ignores length and when that is the wrong choice
- [ ] One honest limitation is named, such as the zero vector failure or the high-dimensional geometry issue

**Done when:**

- `pytest` passes all eleven tests with no failures
- `cosine_similarity` returns the same value after one input is multiplied by 100
- `unit` and `cosine_similarity` raise `ValueError` for the zero vector
- `demo.py` prints the three pairwise similarities and names the most similar pair
- The README explains the difference between dot product and cosine similarity in plain words

**What to commit as evidence:** the source files, the test file, the output of `pytest` pasted into the README, and the output of `demo.py`.

---

## Check your understanding

1. A search system ranks documents by raw dot product with the query embedding. A long document scores higher than a short, very relevant one. Explain why, and state which change makes the ranking depend on direction alone.
2. Two customers have purchase vectors `(2, 0, 1)` and `(20, 0, 10)`. Compute their cosine similarity by hand and explain what it tells you that the Euclidean distance between them would not.
3. A teammate writes `np.array([1, 2, 3]) * np.array([4, 5, 6])` and expects the dot product. Explain what the expression returns, and show the one-line change that gives the dot product.
4. You normalize every embedding to unit length and then store them in a database that sorts by dot product. Explain why this ranking matches a cosine similarity ranking.
5. The projection of `v` onto `u` is zero. What does this tell you about the vectors, and what is the dot product equal to in that case?
6. A gradient clipping routine rescales the gradient vector whenever its norm exceeds 1.0. Explain why the rule uses the magnitude of the whole vector rather than each component separately, and what would change in the model if the rule clipped each component to a fixed range instead.
7. A colleague says two embeddings have a cosine similarity of 0.97, so they must mean the same thing. Give two reasons this conclusion can be wrong, one about the geometry and one about the model.

### Answers

1. The dot product grows with the length of both vectors, so a long document with large components has a larger dot product even when its direction is less relevant. Normalizing both vectors to unit length, which turns the dot product into cosine similarity, makes the ranking depend on direction only.
2. The dot product is `2*20 + 0*0 + 1*10 = 50`, the norms are `sqrt(5)` and `sqrt(500)`, so the cosine is `50 / (sqrt(5) * sqrt(500))`, which equals 1 because the second vector is ten times the first. The two customers buy in exactly the same proportions, and cosine similarity reports that. Euclidean distance would report them as far apart because the second customer buys more in total.
3. The expression multiplies entries pairwise and returns a vector, `[4, 10, 18]`, not a scalar. The fix is `np.dot(a, b)` or `a @ b`, which returns `32`.
4. For unit vectors, the dot product equals the cosine, because both magnitudes are 1. Sorting by the dot product therefore sorts by cosine similarity, and the database can use its fast inner-product index.
5. The vectors are orthogonal, so their dot product is zero. The projection is the zero vector because the shadow of `v` on the direction of `u` has no length.
6. Clipping by the whole norm preserves the direction of the gradient and reduces only its size, so the update still points where the loss decreases fastest. Clipping each component separately changes the direction, which can push the model toward a different and possibly worse update.
7. Geometrically, a cosine of 0.97 means the vectors point nearly the same way, but in high dimensions, most random vectors are also nearly orthogonal, so high similarity can be rare and meaningful, or the embedding space may compress many unrelated concepts into a narrow cone. From the model's side, the two embeddings may be close because the model was trained on a narrow task, or because the text shares surface features such as length or formatting, so the similarity says little about meaning in the reader's sense.

---

## Sources to read, watch and practice

**Read**

- Wikipedia: [Euclidean vector](https://en.wikipedia.org/wiki/Euclidean_vector)
- Wikipedia: [Dot product](https://en.wikipedia.org/wiki/Dot_product)
- Wikipedia: [Cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity)
- Wikipedia: [Unit vector](https://en.wikipedia.org/wiki/Unit_vector)
- Wikipedia: [Norm (mathematics)](https://en.wikipedia.org/wiki/Norm_(mathematics))
- Wikipedia: [Scalar (mathematics)](https://en.wikipedia.org/wiki/Scalar_(mathematics))
- NumPy documentation: "NumPy quickstart" and the reference page for `numpy.linalg.norm` on numpy.org

**Tutorials**

- GeeksforGeeks, search "vector addition and subtraction" and "dot product of two vectors" for worked examples with diagrams.
- NumPy documentation, "Broadcasting" in the user guide, which explains the shape rules used in this day's code.

**Watch**

- 3Blue1Brown (YouTube), "Vectors | Chapter 1, Essence of linear algebra". It gives the arrow and coordinate pictures used in this lesson.
- 3Blue1Brown (YouTube), "Dot products | Chapter 9, Essence of linear algebra". It gives the projection picture for the dot product.
- StatQuest with Josh Starmer (YouTube), search for "Cosine similarity" if you want a second explanation with worked numbers.

**Practice**

- Khan Academy, "Linear algebra" course, the section on vectors and the dot product, with interactive exercises.

---

## Tomorrow

Tomorrow you stack vectors into matrices. A matrix is a grid of numbers, and a dataset with one row per example and one column per feature is exactly that. You will learn how matrices add, multiply and transpose, how their shapes must line up, and how a single matrix multiplication computes the outputs of a neural network layer for a whole batch at once. The dot product you implemented today is the building block of every entry in that product.

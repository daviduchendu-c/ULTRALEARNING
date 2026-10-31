# Day 41: Complexity Analysis

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** beginner to intermediate. You should be comfortable writing Python functions, loops and lists. This day is for anyone who has heard "that's O(n squared)" in a code review and nodded without being sure what it meant.

**By the end of this day, you will have:**

- The ability to count the work in a loop and state its time complexity in Big-O notation
- A clear view of why best, average and worst cases differ for the same function
- An explanation, backed by a counting test, of why appending to a dynamic array is amortized O(1)
- A small benchmark that shows growth rates on your own machine
- A GitHub repository whose GitHub Actions workflow runs the test suite on every push

**What you need:**

- Python 3.12 or newer. The code in this day was checked with Python 3.14 (`python --version` should print at least 3.12)
- Git 2.23 or newer (`git --version`), with the habits from Day 6 and Day 7
- A free GitHub account
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- Optional: VS Code with the official Python extension

**Prerequisites:**

- Module 3: Python basics, including functions, lists, sets, `for` loops and `unittest`
- Day 6: Git Mental Model (`git add`, `git commit`, `git status`)
- Day 8: GitHub Workflow, at least to create a repository and push to it

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Big-O notation](#big-o-notation)
  - [Time and space](#time-and-space)
  - [Best, average and worst cases](#best-average-and-worst-cases)
  - [Amortized analysis](#amortized-analysis)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a growth lab with a CI pipeline](#project-a-growth-lab-with-a-ci-pipeline)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Your program works on ten items. It also has to work on ten million. A function that takes a fraction of a second on the first input can take hours on the second, and the code looks almost the same. Complexity analysis is the set of tools for predicting that difference before you run anything. It asks how the amount of work grows as the input grows, and it ignores the details of your laptop, the version of Python and the mood of the cache.

This matters on the job because almost every performance bug is a growth problem. A report page that loads in 200 milliseconds for a test account and 40 seconds for a real customer usually has a nested loop, or a lookup that scans a list instead of using a set. Interviewers ask about complexity because it is the fastest way to see whether you can reason about code you have not run yet. Teammates ask about it because it decides which fix is worth writing.

The outline for this day has four parts: Big-O, time and space, best and worst cases, and amortized cost. Each one gets a concept section below, with an example you can run. You will finish with a repository that counts operations in code, measures growth with a benchmark, and runs its tests in GitHub Actions. Every later day in this module uses the same repository layout, so the habits you build here will carry forward.

---

## Short history

Big-O notation is older than computers. The table shows how the idea spread. The dates come from the Wikipedia article on Big O notation, which is a good place to check them.

| When | Who or what | What changed |
|------|-------------|--------------|
| 1894 | Paul Bachmann | Introduced the O symbol in a number theory book to describe how a function grows |
| 1909 | Edmund Landau | Popularized the notation in mathematics, which is why it is often called Landau notation |
| 1970s | Donald Knuth and others | Brought the family of notations (Big-O, Big-Omega, Big-Theta) into computer science to describe algorithms |
| Today | Every algorithms course | Used as the common language for comparing an algorithm before and after a change |

The history matters for one reason. Big-O was built to describe the limit of a function as its input grows, and it says nothing about seconds on a particular machine. Keep that in mind when a benchmark disagrees with a formula.

---

## Glossary

- **Algorithm:** a finite, step-by-step procedure that turns inputs into an output.
- **Input size (n):** the count that the running time depends on, such as the number of items in a list or characters in a string.
- **Operation:** a unit of work you count, such as a comparison, an assignment or a loop iteration.
- **Growth rate:** how the work changes when the input size doubles or grows by a fixed step.
- **Big-O:** an upper bound on growth. O(n squared) means the work never grows faster than a constant times n squared, for large enough n.
- **Big-Omega:** a lower bound on growth.
- **Big-Theta:** a bound from both sides. The work grows at the same rate as the function named.
- **Constant factor:** the multiplier that Big-O drops. Two loops of n steps are still O(n).
- **Best, average and worst case:** the least, typical and most work the same function does across all inputs of size n.
- **Amortized cost:** the average cost per operation over a sequence of operations, where a rare expensive step is spread across cheap ones.
- **Dynamic array:** a resizable array that grows its storage when full. Python's `list` is one.
- **Benchmark:** a repeatable timing run that measures real time on a specific machine.

---

## Concept deep dives

This section explains the four ideas in the order you will need them. Each concept has its own short flow with an analogy, the first principles, a worked example, a common mistake and a sentence about where it appears in practice.

### Big-O notation

Imagine a grocery checkout. If the cashier scans one item per second, a basket of 10 items takes 10 seconds and a basket of 100 takes 100 seconds. If a second cashier opens and both scan at once, the time halves, but it is still a straight line that rises with the basket. Big-O describes the shape of that line. It does not care that one cashier is faster than another, because a faster cashier only changes the slope, and Big-O drops constant factors on purpose.

Start from first principles. Count the operations a function performs as a formula in n. Then keep only the term that grows fastest, and drop its constant. A function that does 3n plus 7 operations is O(n). A function that does n squared plus 50n operations is O(n squared), because for large n the square term dominates. The definition says that a function f(n) is O(g(n)) if there are constants c and n0 such that f(n) is at most c times g(n) for every n at least n0. You do not need to write the proof to use the notation. You need to find the dominant term and know the common classes: O(1), O(log n), O(n), O(n log n), O(n squared) and O(2 to the n).

Here is a function whose work you can count by reading the loops:

```python
def count_pairs(items: list[int]) -> int:
    total = 0
    for a in items:          # runs n times
        for b in items:      # runs n times for each a
            total += 1       # constant work
    return total             # n * n = n squared steps in total
```

The outer loop runs n times. For each pass, the inner loop runs n times. The body does constant work, so the total is n times n, which is O(n squared). If you double the list, the work quadruples. You can check that claim with the benchmark in this day's project.

The common mistake is to multiply loops that are not nested. Two separate loops of length n are O(n) plus O(n), which is O(n). A loop inside a loop over the same list is O(n squared). To spot the difference, follow the indentation: each nested `for` that depends on the outer variable multiplies the work. Another mistake is to forget the cost of the operations you call. A `x in some_list` test looks like one step, but Python scans the list, so it costs O(n) inside a loop that already runs n times.

Big-O shows up everywhere that code scales with data: database query planners choose index scans or full table scans based on it, and code reviewers at most product teams flag nested loops over user data for exactly this reason.

### Time and space

Think about packing for a trip. Time is how long it takes you to pack. Space is how much luggage you carry at the peak. Some strategies are quick but bulky, such as taking one bag for each item, while others take longer but fit in one suitcase. Algorithms make the same trade. Time complexity counts operations as a function of input size. Space complexity counts the extra memory the algorithm needs as a function of input size, not counting the input itself.

The first principles are simple. A function that stores one number regardless of input size uses O(1) extra space. A function that builds a new list the same length as the input uses O(n) extra space. A recursive function uses space for each active call on the call stack, so a recursion that goes n levels deep uses O(n) space even if it stores nothing else. Day 50 returns to that cost.

```python
def running_totals(items: list[int]) -> list[int]:
    result = []                  # new list, grows to length n
    total = 0
    for x in items:              # one pass: O(n) time
        total += x
        result.append(total)
    return result                # O(n) extra space, O(n) time

def running_total_max(items: list[int]) -> int:
    best = float("-inf")         # a fixed number of variables
    total = 0
    for x in items:
        total += x
        best = max(best, total)
    return best                  # O(n) time, O(1) extra space
```

Both functions take O(n) time, but only the first keeps a growing list. The second computes the same maximum running total with a few variables. When memory is tight, that difference decides which version can run on a phone or inside a function with a small memory limit.

The common mistake is to report space as zero because nothing was "allocated" on purpose. Python allocates when you create a list, a dictionary, a string slice or a new object in a loop. A slice such as `items[1:]` copies the data, which costs O(n) space and O(n) time even though it looks like a single expression. To spot this, look for constructs that create new containers, and ask how large those containers can get.

Space limits show up in production as out-of-memory crashes in data pipelines and as lag in mobile apps, where a copy of a large list can make a screen freeze. Game engines and embedded firmware teams track space as carefully as time for the same reason.

### Best, average and worst cases

A library catalog is a fair model. If you walk to the shelf and the book is the first one you see, you are done in one step. If it is not on the shelf at all, you check every book. The average is somewhere in between. One function can have all three answers, and an honest report names which one it is quoting.

Start with a plain linear search. The function checks each item in order and stops at the match. Write it so that it counts its comparisons, which makes the cases visible in numbers:

```python
def linear_search_comparisons(items: list[int], target: int) -> int:
    """Return how many comparisons a linear search makes, found or not."""
    comparisons = 0
    for value in items:
        comparisons += 1
        if value == target:
            return comparisons
    return comparisons
```

The best case happens when the target is first. The function makes one comparison, so the best case is O(1). The worst case is a target that is missing or last. The function makes n comparisons, so the worst case is O(n). The average depends on which targets you expect. If every position is equally likely, the average over the found targets is (n plus 1) divided by 2 comparisons, which is still O(n). You can verify that claim exactly: for a list of 10 distinct values, summing the comparisons over every possible target gives 1 plus 2 plus up to 10, which is 55. Dividing by 10 gives an average of 5.5, which equals (10 plus 1) divided by 2.

The mistake to watch for is quoting only one case. When someone says "binary search is O(log n)", they usually mean the worst case. When someone says "hash tables are O(1)", they usually mean the average case with a good hash function, and the worst case can be O(n) when many keys collide. Day 45 shows how that happens. Name the case in every claim you write, or your reader will assume the most flattering one.

The best, average and worst cases matter in production because user traffic is rarely uniform. A cache, a search box and a login check all see skewed inputs, and an algorithm whose worst case is rare may be fine in practice while one whose average is bad may not be.

### Amortized analysis

Consider a suitcase that doubles in size whenever it fills. Most items slide in without any effort. Once in a while, you have to move everything into a bigger case. That move is expensive, but it happens rarely, so the cost per item over the whole trip is small. Amortized analysis averages a cost over a sequence of operations so that rare, expensive steps are shared across many cheap ones.

Python's `list` is a dynamic array. Behind the scenes it keeps a block of memory with a capacity that is larger than its length. Appending into free space is cheap. When the block is full, Python allocates a bigger block and copies every element, which costs O(n) for that single append. If the capacity grows by a fixed amount each time, the copies add up to O(n squared) over n appends. If the capacity doubles, the copies form a geometric series that totals less than 2n for n appends, so each append costs O(1) amortized.

The code in this day's project counts those copies so you do not have to take the argument on trust:

```python
def dynamic_array_copies(n: int) -> int:
    """Count the element copies made while appending n items, doubling when full."""
    capacity = 1
    size = 0
    copies = 0
    for _ in range(n):
        if size == capacity:
            copies += size        # move every existing element to the new block
            capacity *= 2
        size += 1
    return copies
```

Run it for a few values. For n equal to 3 the function returns 3, because the second append copies 1 element and the third copies 2. For n equal to 5 it returns 7, because the fifth append copies 4 more. The total for n appends is always less than 2n, so the average copy cost per append is below 2, which is O(1) amortized. CPython does not double. Its growth rule in `Objects/listobject.c` adds about one eighth of the size plus a small constant each time. That factor is smaller, so the copies per append are larger, but the rule is still geometric, and the amortized bound still holds. The exact constant is an implementation detail that can change between versions, so do not build a design on it.

The common mistake is to read amortized O(1) as "every append is O(1)". The rare resize costs O(n) on its own. That is why a real-time system that cannot tolerate a 50 millisecond pause can still see a spike, even when the average is cheap. The way to spot this is to ask whether a single operation can be expensive and whether the report is about the average or the worst single step.

Amortized analysis shows up in the standard library of most languages. Java's `ArrayList`, C++'s `std::vector` and Rust's `Vec` all grow geometrically, usually by a factor between 1.5 and 2 depending on the library, and game engines that cannot afford a pause often reserve capacity up front to avoid the spike.

---

## Hands-on exercises

Work in a new folder called `growth-lab` that you will turn into the project repository. Use `git init -b main` so the branch name matches Day 7.

Exercise 1 reads complexity from code. Write five small functions in `docs/growth-notes.md`: one with a single loop, one with two sequential loops, one with nested loops, one that halves its input each step, and one that slices a list inside a loop. For each function, state the time complexity, the best and worst case, and the extra space. Check your answers by counting the operations for n equal to 10 and n equal to 20.

Done when `docs/growth-notes.md` has five entries and every entry names a case and a space bound.

Exercise 2 measures growth. Create `benchmark.py` as shown in the project and run it. Record the time for `pairwise` at each size in the notes. Doubling n should roughly quadruple the pairwise time. Hashed lookup should grow more slowly. Write one sentence explaining any result that does not match the formula.

Done when your notes contain both timing tables and a sentence about the growth ratio between sizes.

Exercise 3 counts comparisons. In `tests/test_complexity.py`, add a test that checks the best case (target first), the worst case (target missing) and the position-by-position sum that proves the average is (n plus 1) divided by 2 for n equal to 20.

Done when `python -m unittest` passes and the new test fails if you change `linear_search_comparisons` to return one more than it should.

Exercise 4 checks amortized cost. Write a loop that calls `dynamic_array_copies(n)` for n from 1 to 64 and prints the ratio of copies to n. Confirm the ratio stays below 2 and explain why it dips after each power of two.

Done when your notes show the ratio at n equal to 1, 2, 4, 8, 16, 32 and 64, with one sentence explaining the pattern.

Exercise 5 closes the loop with CI. Commit the work, push it to a new GitHub repository, and open the Actions tab. The workflow should run and pass.

Done when the GitHub Actions run for your latest commit shows a green check.

---

## Project: a growth lab with a CI pipeline

The project is a small library of complexity examples, a benchmark that measures them, and a GitHub Actions workflow that runs the tests. The code is short on purpose. The skill being tested is the analysis, and the workflow is what makes the analysis repeatable for the rest of the module.

Suggested repo name: `100-days-software-day-41-complexity-analysis`.

The repo layout:

```
100-days-software-day-41-complexity-analysis/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── dsa/
│   ├── __init__.py
│   ├── bench.py
│   └── complexity.py
├── docs/
│   └── growth-notes.md
├── tests/
│   ├── __init__.py
│   ├── test_bench.py
│   └── test_complexity.py
└── README.md
```

The empty `__init__.py` files matter. `tests/__init__.py` makes `python -m unittest` discover the test folder. Without it, the command finds zero tests and still prints a short OK, which is a quiet failure. `dsa/__init__.py` marks the folder as a package so that `from dsa.complexity import ...` works from any script in the repo root.

The timing helper, which the benchmark uses. Starter code for `dsa/bench.py`:

```python
import time
from collections.abc import Callable


def best_time(func: Callable[[], object], repeat: int = 5) -> float:
    """Return the fastest of several runs, in seconds."""
    if repeat < 1:
        raise ValueError("repeat must be at least 1")
    best = float("inf")
    for _ in range(repeat):
        start = time.perf_counter()
        func()
        best = min(best, time.perf_counter() - start)
    return best


def growth_table(
    make_call: Callable[[int], Callable[[], object]],
    sizes: list[int],
    repeat: int = 5,
) -> list[tuple[int, float]]:
    """For each size n, build a zero-argument call and time it."""
    rows = []
    for n in sizes:
        rows.append((n, best_time(make_call(n), repeat=repeat)))
    return rows
```

Taking the minimum over several runs is a deliberate choice. Other programs on your machine add noise, and noise only makes a run slower, so the fastest run is the closest to the true cost of the code.

Starter code for `dsa/complexity.py`:

```python
def has_duplicate_pairwise(items: list[int]) -> bool:
    """O(n squared): compare every pair of positions."""
    n = len(items)
    for i in range(n):
        for j in range(i + 1, n):
            if items[i] == items[j]:
                return True
    return False


def has_duplicate_hashed(items: list[int]) -> bool:
    """O(n) expected: remember each value in a set."""
    seen: set[int] = set()
    for x in items:
        if x in seen:
            return True
        seen.add(x)
    return False


def linear_search_comparisons(items: list[int], target: int) -> int:
    """Return how many comparisons a linear search makes, found or not."""
    comparisons = 0
    for value in items:
        comparisons += 1
        if value == target:
            return comparisons
    return comparisons


def dynamic_array_copies(n: int) -> int:
    """Count the element copies made while appending n items, doubling when full."""
    capacity = 1
    size = 0
    copies = 0
    for _ in range(n):
        if size == capacity:
            copies += size
            capacity *= 2
        size += 1
    return copies
```

Starter code for `benchmark.py`:

```python
import random

from dsa.bench import growth_table
from dsa.complexity import has_duplicate_hashed, has_duplicate_pairwise


def make_pairwise(n: int):
    data = random.sample(range(10 * n), n)  # all distinct: the worst case
    return lambda: has_duplicate_pairwise(data)


def make_hashed(n: int):
    data = random.sample(range(10 * n), n)
    return lambda: has_duplicate_hashed(data)


if __name__ == "__main__":
    sizes = [500, 1000, 2000, 4000]
    for name, make in [("pairwise", make_pairwise), ("hashed", make_hashed)]:
        print(name)
        for n, seconds in growth_table(make, sizes, repeat=3):
            print(f"  n={n:>5}  {seconds * 1000:8.2f} ms")
```

Starter code for `tests/test_complexity.py`:

```python
import unittest

from dsa.complexity import (
    dynamic_array_copies,
    has_duplicate_hashed,
    has_duplicate_pairwise,
    linear_search_comparisons,
)


class DuplicateTests(unittest.TestCase):
    def test_both_versions_agree(self):
        cases = [[], [1], [1, 2, 3], [1, 2, 1], [5, 5], [3, 1, 4, 1, 5]]
        for items in cases:
            with self.subTest(items=items):
                self.assertEqual(
                    has_duplicate_pairwise(items), has_duplicate_hashed(items)
                )


class LinearSearchTests(unittest.TestCase):
    def test_best_case_first_item(self):
        self.assertEqual(linear_search_comparisons([7, 8, 9], 7), 1)

    def test_worst_case_missing_target(self):
        self.assertEqual(linear_search_comparisons([7, 8, 9], 100), 3)

    def test_average_over_all_targets(self):
        items = list(range(10))
        total = sum(linear_search_comparisons(items, t) for t in items)
        self.assertEqual(total, 55)  # 1 + 2 + ... + 10


class DynamicArrayTests(unittest.TestCase):
    def test_small_known_values(self):
        self.assertEqual(dynamic_array_copies(0), 0)
        self.assertEqual(dynamic_array_copies(1), 0)
        self.assertEqual(dynamic_array_copies(3), 3)
        self.assertEqual(dynamic_array_copies(5), 7)

    def test_copies_stay_below_two_per_append(self):
        for n in range(1, 300):
            with self.subTest(n=n):
                self.assertLess(dynamic_array_copies(n), 2 * n)


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_bench.py`:

```python
import unittest

from dsa.bench import best_time, growth_table


class BenchTests(unittest.TestCase):
    def test_best_time_is_non_negative_float(self):
        seconds = best_time(lambda: sum(range(1000)), repeat=3)
        self.assertIsInstance(seconds, float)
        self.assertGreaterEqual(seconds, 0.0)

    def test_repeat_must_be_positive(self):
        with self.assertRaises(ValueError):
            best_time(lambda: None, repeat=0)

    def test_growth_table_keeps_sizes_in_order(self):
        rows = growth_table(lambda n: (lambda: sum(range(n))), [10, 20], repeat=2)
        self.assertEqual([n for n, _ in rows], [10, 20])


if __name__ == "__main__":
    unittest.main()
```

Starter code for the workflow in `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Run tests
        run: python -m unittest -v
```

The workflow pins Python 3.12 so the result does not change when GitHub updates its default runner image. GitHub's "Building and testing Python" guide in its documentation describes the same setup.

Step-by-step tasks:

1. Create the folder with `git init -b main`, add the files above, create the two empty `__init__.py` files, and run `python -m unittest` locally. All tests should pass. If the output says "Ran 0 tests", the `tests/__init__.py` file is missing.
2. Run `python benchmark.py`. Copy the output into `docs/growth-notes.md` with a sentence on the ratio between the pairwise times at n equal to 1000 and 2000.
3. Add a test to `tests/test_complexity.py` that checks `has_duplicate_hashed` returns False for an empty list and True when the same value appears twice at the end of a long list.
4. Write `README.md` with three lines: what the project does, how to run the tests, and how to run the benchmark.
5. Commit with the message "Add complexity examples, benchmark and CI", push to GitHub, and check the Actions tab.

Done when:

- `python -m unittest` passes on your machine and in GitHub Actions
- `python benchmark.py` prints two tables, and your notes explain the growth ratio
- `docs/growth-notes.md` names the best, average and worst case for `linear_search_comparisons`
- The latest commit on `main` has a green check

Evidence to commit: `dsa/`, `tests/`, `benchmark.py`, `.github/workflows/ci.yml`, `docs/growth-notes.md` and `README.md`. The GitHub Actions run is the proof that the tests pass on a clean machine as well as yours.

This project is light on new ideas and heavy on repetition. The repository and the workflow are the parts you will reuse for every day in this module.

---

## Check your understanding

1. A function loops over a list once to find the maximum, then loops over it again to count how many items equal that maximum. What is its time complexity, and would your answer change if the second loop ran only on the first half of the list?
2. Two functions both return the same answer. One is O(n log n) and the other is O(n squared). For n equal to 10, the second one might be faster in practice. Explain why, and say at what kind of input size the complexity class should decide your choice.
3. A teammate says a hash table lookup is O(1) and so it is always the right choice. Name the case they are quoting, describe one input pattern that breaks the claim, and say what your report should say instead.
4. Why does `dynamic_array_copies(n)` stay below 2n for every n, even though one single append can copy almost n elements?
5. A slice such as `items[1:]` inside a loop looks cheap. Compute its total time complexity for a list of n items, and explain the extra space it uses.
6. Your benchmark shows that doubling n makes a function take 2.1 times longer instead of 4 times. Name two possible causes that are not in the algorithm itself, and say how you would check each one.

### Answers

1. The first loop is O(n) and the second is O(n), so the total is O(n) plus O(n), which is O(n). Running the second loop on only half the list still gives O(n), because the constant factor of one half is dropped. Only a nested loop over the same data would make it O(n squared).
2. For n equal to 10, the constant factors and overhead can outweigh the growth rate, so the O(n squared) function can win. The complexity class decides the choice once n is large enough that the dominant term controls the total. Measure at the sizes your product actually sees before you decide.
3. They are quoting the average case with a good hash function. An input where many keys share one bucket, such as keys chosen to collide, makes lookups scan a long chain and reach O(n). The report should name the average case, name the worst case, and mention that the bound depends on the hash function and the input distribution.
4. Each resize doubles the capacity, so the sizes of the expensive copies form a geometric series, 1 plus 2 plus 4 and so on. A geometric series with ratio 2 sums to less than twice its largest term, and the largest term is less than n, so the total is less than 2n. A single append can cost O(n), but the average over all appends stays below 2.
5. Each slice creates a new list with about n minus 1 elements, so one slice costs O(n) time. Inside a loop of n iterations, the total time becomes O(n squared). Each slice also allocates O(n) extra space, and the old slice is freed only when the reference is dropped.
6. Two common causes are CPU frequency scaling or background processes that add noise, and memory effects, such as the data no longer fitting in the cache at larger sizes. To check, repeat the run several times, close other programs, and look at the minimum time as the report says. To check memory effects, measure at more sizes and look for a jump in the ratio where the working set crosses a cache size.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Big O notation](https://en.wikipedia.org/wiki/Big_O_notation)
- [Wikipedia: Time complexity](https://en.wikipedia.org/wiki/Time_complexity) and [Wikipedia: Space complexity](https://en.wikipedia.org/wiki/Space_complexity)
- [Wikipedia: Amortized analysis](https://en.wikipedia.org/wiki/Amortized_analysis)
- The Python Wiki page "TimeComplexity", on wiki.python.org, which lists the cost of common operations on `list`, `dict` and `set`
- The Python documentation for the `time` module, specifically `time.perf_counter`, at docs.python.org
- The Python documentation for the `unittest` module, at docs.python.org
- Thomas H. Cormen and others, *Introduction to Algorithms*, the chapter on asymptotic notation, from the MIT Press

*Tutorials*

- GeeksforGeeks, "Analysis of Algorithms" and "Big-O Analysis of Algorithms". Search the site for each title.
- W3Schools has no full course on this topic. Use the Python tutorial on lists at docs.python.org/3/tutorial/datastructures.html for the operations you will count.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the early lectures on asymptotic notation and the lectures on hashing. Search the course site for "asymptotic notation".
- Abdul Bari's YouTube playlist on algorithm analysis. Search YouTube for "Abdul Bari asymptotic notation".

*Practice*

- The Big-O Cheat Sheet at bigocheatsheet.com. Use it to check the complexity of common operations after you have counted them yourself.
- LeetCode, the "Explore" track on time complexity, at leetcode.com. Look for the free problems that ask you to state the complexity of your solution.

---

## Tomorrow

Day 42 moves from counting operations to using arrays and strings directly. You will write prefix sums, two-pointer scans and a sliding window, and you will use this day's counting habit to show that each one does O(n) work where a naive version does O(n squared). The `dsa` package and the CI workflow you built today stay in place, and Day 42 adds `dsa/arrays.py` and its tests next to them.

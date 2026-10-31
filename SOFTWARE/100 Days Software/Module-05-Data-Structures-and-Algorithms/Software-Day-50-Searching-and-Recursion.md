# Day 50: Searching and Recursion

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** intermediate. You should have finished Day 49, understand sorted order and tree recursion, and be comfortable reading a traceback. This day is for anyone who has written a recursive function and not been sure why it sometimes crashes with `RecursionError`.

**By the end of this day, you will have:**

- An iterative and a recursive binary search, each with a clear invariant, and a `lower_bound` and `upper_bound` that answer "first at or after" and "first after" questions
- A recursive thinking method: base case, smaller subproblem, combine, and a check that the recursion always gets smaller
- An understanding of the call stack, with a demonstration of `RecursionError` and the explicit-stack fix
- A divide-and-conquer fast power function that uses O(log n) multiplications
- A release log that finds the version in effect on any date, tested against a linear scan, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 49
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 46: Trees and BSTs (recursive functions over a structure)
- Day 49: Sorting (sorted order, and merge sort's divide step)
- Day 42: Arrays and Strings (two-pointer ranges and the off-by-one rules)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Binary search](#binary-search)
  - [Recursive thinking](#recursive-thinking)
  - [Call stack](#call-stack)
  - [Divide-and-conquer](#divide-and-conquer)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a release log with date lookup](#project-a-release-log-with-date-lookup)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Finding a value in a list is the most common question a program asks, and the answer depends on what you know about the list. If nothing is known, you must check every item, which costs O(n). If the list is sorted, you can throw away half of it with each comparison, which costs O(log n). For a million items, that is the difference between about a million checks and about twenty. Binary search is the algorithm that makes the sorted case work, and it shows up in the standard library, in database indexes, in version lookups and in debugging tools that find the first bad commit.

Recursion is the other half of this day. Many of the best algorithms, including binary search in its cleanest form, merge sort, tree traversal and fast exponentiation, are written as a function that calls itself on a smaller problem. Recursion is a natural way to think about these problems, but it has a cost: each call uses memory on the call stack, and Python stops at a fixed depth. Understanding both the power and the limit is the reason this day exists.

The outline asks for binary search, recursive thinking, the call stack and divide-and-conquer. The project applies them to a release log, which answers "which version was live on this date" with a binary search over sorted release dates.

---

## Glossary

- **Binary search:** a search that repeatedly halves a sorted range by comparing the middle element with the target.
- **Invariant:** a statement that is true before and after each step of a loop, which shows the algorithm is correct.
- **Lower bound:** the first position where a value could be inserted without breaking sorted order, which is the first element at or above the target.
- **Upper bound:** the first position after all elements equal to the target.
- **Recursion:** a function that calls itself on a smaller input.
- **Base case:** the input for which a recursive function returns directly without calling itself.
- **Recursive case:** the part of a recursive function that calls itself on a smaller input.
- **Call stack:** the record of active function calls, with one frame for each call that has not yet returned.
- **RecursionError:** the exception Python raises when the call stack exceeds the recursion limit.
- **Divide and conquer:** splitting a problem into smaller independent parts, solving each one, and combining the answers.
- **Fast exponentiation:** computing a power with about log2 of the exponent multiplications, by squaring.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Binary search

Think of a phone book. You do not start at page one. You open near the middle, see whether your name comes before or after that page, and tear out the half that cannot contain it. Repeat on what is left. Each look roughly halves the range, which is why a phone book with a million names takes about 20 looks.

From first principles, keep a range of positions, `lo` and `hi`, that is guaranteed to contain the target if it is present. The invariant is that the target, if it exists, lies between `lo` and `hi`. Check the middle. If the middle is smaller than the target, the target must be to the right, so `lo` moves past the middle. If the middle is larger, `hi` moves before it. When the range is empty, the target is absent. Each step halves the range, so there are at most about log2 n steps.

```python
def binary_search(items: list, target) -> int:
    lo, hi = 0, len(items) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if items[mid] == target:
            return mid
        if items[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1

print(binary_search([1, 3, 5, 7, 9], 7))   # 3
print(binary_search([1, 3, 5, 7, 9], 4))   # -1
```

A related function is more useful in practice. The lower bound finds where a value would go in the sorted list, which answers "first element at or above this value". Using `hi = len(items)` and `hi = mid` keeps the answer inside the range even when every element is smaller than the target.

```python
def lower_bound(items: list, target) -> int:
    lo, hi = 0, len(items)
    while lo < hi:
        mid = (lo + hi) // 2
        if items[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo

print(lower_bound([1, 3, 3, 5], 3))   # 1, the first 3
print(lower_bound([1, 3, 3, 5], 6))   # 4, past the end
```

The limitation is the sorted requirement. If the list is not sorted, binary search returns wrong answers without any error, which is one of the most dangerous bugs in this area. Check the precondition when the cost is low, or document it in the function name.

The common mistake is the off-by-one error in the range update. Writing `lo = mid` instead of `mid + 1` can loop forever when `lo` and `hi` are neighbors. To spot it, test a two-element list and a target that is the larger element. Those cases hit the boundary.

Binary search is used in `bisect` in the Python standard library, in `git bisect` to find the commit that broke a test, and in database indexes that use B-trees, which are a wide form of the same idea.

### Recursive thinking

Imagine you are asked to count the people in a long line. You could walk the line and count, or you could ask the person at the front to count everyone behind them and add one for yourself. The second method is recursive. It works when the question for the front person is the same question, on a smaller line, and when someone knows the answer for an empty line.

From first principles, every recursive function needs a base case that does not recurse, and a recursive case that moves toward the base case. Write the base case first. Then ask: if I already had the answer for a smaller input, how would I combine it with the current one to get the answer for this input? If you can answer that, the recursive case is complete. The check to run before you trust the function is that every call makes the input strictly smaller, so the recursion must end.

```python
def sum_to(n: int) -> int:
    if n == 0:                       # base case: an empty sum is 0
        return 0
    return n + sum_to(n - 1)         # recursive case: one term plus a smaller sum

print(sum_to(10))   # 55
```

The limitation is that recursion trades loops for calls. Each call uses memory, so deep recursion can fail. A loop that does the same work uses constant memory. Some recursive functions, such as those with tail calls, can be rewritten as loops, but Python does not remove tail calls, so you must do that rewrite yourself.

The common mistake is a recursive case that does not get smaller, such as calling `sum_to(n)` instead of `sum_to(n - 1)`, or a base case that is never reached for some input, such as a negative number passed to `sum_to`. To spot both, call the function with the boundary values, including zero and negative numbers, and watch for a crash.

Recursive thinking is how parsers read nested expressions, how file system tools walk folders and how divide-and-conquer algorithms are described in textbooks.

### Call stack

Picture a stack of trays in a cafeteria. Each time a function calls another function, a new tray goes on top, holding the function's variables and the place it will return to. When a function returns, its tray comes off. The stack grows as deep as the calls go, and it shrinks as they return.

From first principles, recursion uses one frame per active call. A recursive call to depth d holds d frames at once. Python's default limit is 1000 frames, which you can read with `sys.getrecursionlimit()`. When the limit is reached, Python raises `RecursionError` instead of crashing. That is a safety feature, but it means a recursive function on an input that is 5000 items deep will fail, even if the algorithm is correct.

```python
import sys

print(sys.getrecursionlimit())   # 1000 by default

try:
    sum_to(5000)
except RecursionError:
    print("the call stack ran out")
```

The fix is an explicit stack or a loop. Converting `sum_to` to a loop uses one variable and no extra frames, and it works for any n.

```python
def sum_to_iterative(n: int) -> int:
    total = 0
    for k in range(n + 1):
        total += k
    return total

print(sum_to_iterative(5000))   # 12502500
```

The limitation of the loop version is that it can be less clear for problems that naturally split into parts, such as tree traversal or merge sort. For those, the usual choice is recursion with a note about the maximum depth, or an explicit stack, as the tree traversal from Day 46 showed.

The common mistake is to assume that recursion depth equals the number of items in the list. For a balanced tree or merge sort it is about log n, but for a chain it is n. The depth depends on the shape of the recursion, not on the size of the input alone. To spot the risk, test the function on the worst input shape you can imagine.

The call stack is the reason a stack overflow happens in every language with recursion, and it is why production code that handles user-supplied nesting, such as deeply nested JSON, often limits the depth explicitly.

### Divide-and-conquer

Think of a long book that must be indexed. You split the book into two halves, ask two readers to index each half, and merge the two indexes. Each reader does the same job on a smaller book. The split is the divide, the solving of each half is the conquer, and the merge is the combine. Divide and conquer is the general pattern behind merge sort, binary search and many fast algorithms.

From first principles, a divide-and-conquer algorithm works when the halves are independent and the combine step is cheap. Binary search uses only one half, because the target can be on one side only, so each step does constant work and the total is O(log n). Fast exponentiation uses the same idea on the exponent: computing `base ** exp` requires only the result for `exp // 2`, squared, and one extra multiplication if the exponent is odd.

```python
def fast_power(base: int, exp: int) -> int:
    if exp < 0:
        raise ValueError("exp must be non-negative")
    if exp == 0:
        return 1
    half = fast_power(base, exp // 2)
    result = half * half
    if exp % 2 == 1:
        result *= base
    return result

print(fast_power(3, 13))    # 1594323
print(3 ** 13)              # 1594323, the built-in agrees
```

The limitation is that divide and conquer only helps when the subproblems do not overlap in a costly way. When they do, as in the naive Fibonacci recursion, the same subproblem is solved many times, and the total cost explodes. Day 52 handles that case with dynamic programming.

The common mistake is to write the halves so that they overlap or miss elements, such as using `mid` in both halves of a split, or dropping the last element. To spot it, count the elements in the two halves on paper for a list of five items. They must add to five and not share an index.

Divide and conquer appears in merge sort, in binary search in sorted data, in fast Fourier transforms used by audio and signal processing, and in the closest-pair algorithms used in geographic and graphics software.

---

## Hands-on exercises

Work in the repository from Day 49. Keep the search functions in `dsa/searching.py` and the release log in `dsa/release_lookup.py`.

Exercise 1 traces binary search by hand. For the list `[2, 4, 6, 8, 10, 12, 14]` and target 10, write `lo`, `hi` and `mid` for each step until the target is found. Then do the same for target 5, which is absent.

Done when your traces match the code's results, and each step shows the range halving.

Exercise 2 counts steps. Write a function that counts the loop iterations of `binary_search` on a sorted list of one million integers for a set of 1000 random targets, using `random.Random(15)`. Record the largest count in `docs/searching-notes.md` and compare it with log2 of one million.

Done when the largest count is at most about 20, and your notes explain the comparison.

Exercise 3 checks the boundary. Test `binary_search` and `lower_bound` on an empty list, a one-element list, a target smaller than every element, a target larger than every element, and a list with duplicates. Confirm `lower_bound` returns the first index of a duplicate.

Done when each case passes in a subtest.

Exercise 4 finds the recursion limit. Call the recursive sum from the concept section (`sum_to`, which the project calls `sum_to_recursive`) for n equal to 500, 900, 1000 and 1100, and record which values succeed. Then call `sum_to_iterative` for n equal to one million and time it.

Done when your notes name the first n that fails, and the iterative version succeeds for one million.

Exercise 5 checks fast power. Compare `fast_power(b, e)` with `b ** e` for all bases from 1 to 20 and exponents from 0 to 40. Count the multiplications made by `fast_power` for exponent 1024, by adding a counter in a copy, and record the count.

Done when all comparisons pass and your count for exponent 1024 is about 11 squarings plus at most as many extra multiplications.

---

## Project: a release log with date lookup

The project is a release log that answers one question: which version was in effect on a given date. Releases are stored as sorted ISO dates, which sort correctly as strings. A binary search finds the last release on or before the date, using an upper bound. The tests compare the answers with a linear scan, and they check the boundary dates, where the off-by-one errors live.

Suggested repo name: `100-days-software-day-50-searching-and-recursion`. Copy the Day 49 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-50-searching-and-recursion/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── build_demo.py
├── count_words.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── bst.py
│   ├── browser_history.py
│   ├── build_order.py
│   ├── complexity.py
│   ├── graph.py
│   ├── hash_table.py
│   ├── heap.py
│   ├── leaderboard.py
│   ├── linked_list.py
│   ├── release_lookup.py
│   ├── scheduler.py
│   ├── searching.py
│   ├── sorting.py
│   ├── stacks_queues.py
│   └── word_count.py
├── docs/
│   ├── graph-notes.md
│   ├── searching-notes.md
│   └── sorting-notes.md
├── height_demo.py
├── leaderboard_demo.py
├── print_queue.py
├── release_demo.py
├── schedule_demo.py
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_bst.py
│   ├── test_browser_history.py
│   ├── test_build_order.py
│   ├── test_complexity.py
│   ├── test_graph.py
│   ├── test_hash_table.py
│   ├── test_heap.py
│   ├── test_leaderboard.py
│   ├── test_linked_list.py
│   ├── test_print_queue.py
│   ├── test_release_lookup.py
│   ├── test_scheduler.py
│   ├── test_searching.py
│   ├── test_sorting.py
│   ├── test_stacks_queues.py
│   └── test_word_count.py
└── README.md
```

Starter code for `dsa/searching.py`:

```python
from typing import Any


def binary_search(items: list, target: Any) -> int:
    """Index of target in the sorted list items, or -1 if absent."""
    lo, hi = 0, len(items) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if items[mid] == target:
            return mid
        if items[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1


def binary_search_recursive(items: list, target: Any, lo: int = 0, hi: int | None = None) -> int:
    if hi is None:
        hi = len(items) - 1
    if lo > hi:
        return -1
    mid = (lo + hi) // 2
    if items[mid] == target:
        return mid
    if items[mid] < target:
        return binary_search_recursive(items, target, mid + 1, hi)
    return binary_search_recursive(items, target, lo, mid - 1)


def lower_bound(items: list, target: Any) -> int:
    """First index whose value is at least target, or len(items) if none."""
    lo, hi = 0, len(items)
    while lo < hi:
        mid = (lo + hi) // 2
        if items[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo


def upper_bound(items: list, target: Any) -> int:
    """First index whose value is greater than target, or len(items) if none."""
    lo, hi = 0, len(items)
    while lo < hi:
        mid = (lo + hi) // 2
        if items[mid] <= target:
            lo = mid + 1
        else:
            hi = mid
    return lo


def fast_power(base: int, exp: int) -> int:
    """base ** exp in O(log exp) multiplications, using divide and conquer."""
    if exp < 0:
        raise ValueError("exp must be non-negative")
    if exp == 0:
        return 1
    half = fast_power(base, exp // 2)
    result = half * half
    if exp % 2 == 1:
        result *= base
    return result


def sum_to_recursive(n: int) -> int:
    if n == 0:
        return 0
    return n + sum_to_recursive(n - 1)


def sum_to_iterative(n: int) -> int:
    total = 0
    for k in range(n + 1):
        total += k
    return total
```

Starter code for `dsa/release_lookup.py`:

```python
from dsa.searching import upper_bound


class ReleaseLog:
    """Releases as (ISO date, version) pairs. ISO dates such as 2026-03-02 sort correctly as text."""

    def __init__(self, releases: list[tuple[str, str]]) -> None:
        self._releases = sorted(releases)
        self._dates = [day for day, _ in self._releases]

    def version_on(self, day: str) -> str | None:
        """The latest version released on or before day, or None if day is before the first release."""
        i = upper_bound(self._dates, day)
        if i == 0:
            return None
        return self._releases[i - 1][1]
```

Starter code for `release_demo.py`:

```python
from dsa.release_lookup import ReleaseLog

if __name__ == "__main__":
    log = ReleaseLog(
        [("2026-01-10", "1.0.0"), ("2026-03-02", "1.1.0"), ("2026-06-18", "2.0.0")]
    )
    for day in ["2025-12-31", "2026-01-10", "2026-02-14", "2026-07-01"]:
        print(day, "->", log.version_on(day) or "no release yet")
```

Running `python release_demo.py` prints no release yet for 2025-12-31, 1.0.0 for 2026-01-10 and 2026-02-14, and 2.0.0 for 2026-07-01.

Starter code for `tests/test_searching.py`:

```python
import random
import unittest

from dsa.searching import (
    binary_search,
    binary_search_recursive,
    fast_power,
    lower_bound,
    sum_to_iterative,
    sum_to_recursive,
    upper_bound,
)


class BinarySearchTests(unittest.TestCase):
    def test_finds_present_values(self):
        items = [1, 3, 5, 7, 9]
        for i, value in enumerate(items):
            self.assertEqual(binary_search(items, value), i)
            self.assertEqual(binary_search_recursive(items, value), i)

    def test_absent_values(self):
        items = [1, 3, 5, 7, 9]
        self.assertEqual(binary_search(items, 4), -1)
        self.assertEqual(binary_search_recursive(items, 4), -1)

    def test_edge_cases(self):
        self.assertEqual(binary_search([], 1), -1)
        self.assertEqual(binary_search([5], 5), 0)
        self.assertEqual(binary_search([5], 6), -1)

    def test_bounds_with_duplicates(self):
        items = [1, 3, 3, 5]
        self.assertEqual(lower_bound(items, 3), 1)
        self.assertEqual(upper_bound(items, 3), 3)
        self.assertEqual(lower_bound(items, 0), 0)
        self.assertEqual(lower_bound(items, 6), 4)

    def test_matches_linear_scan_on_random_data(self):
        rng = random.Random(15)
        items = sorted(rng.sample(range(1000), 200))
        for _ in range(300):
            target = rng.randint(0, 999)
            expected = items.index(target) if target in items else -1
            with self.subTest(target=target):
                self.assertEqual(binary_search(items, target), expected)


class PowerAndRecursionTests(unittest.TestCase):
    def test_fast_power_matches_builtin(self):
        for base in range(1, 21):
            for exp in range(0, 41):
                with self.subTest(base=base, exp=exp):
                    self.assertEqual(fast_power(base, exp), base**exp)

    def test_negative_exponent_raises(self):
        with self.assertRaises(ValueError):
            fast_power(2, -1)

    def test_recursive_and_iterative_sums_agree(self):
        for n in range(0, 200):
            self.assertEqual(sum_to_recursive(n), sum_to_iterative(n))

    def test_deep_recursion_hits_the_limit(self):
        with self.assertRaises(RecursionError):
            sum_to_recursive(5000)
        self.assertEqual(sum_to_iterative(5000), 12502500)


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_release_lookup.py`:

```python
import random
import unittest

from dsa.release_lookup import ReleaseLog


def linear_version_on(releases, day):
    best = None
    for release_day, version in sorted(releases):
        if release_day <= day:
            best = version
    return best


class ReleaseLogTests(unittest.TestCase):
    def setUp(self):
        self.releases = [
            ("2026-01-10", "1.0.0"),
            ("2026-03-02", "1.1.0"),
            ("2026-06-18", "2.0.0"),
        ]
        self.log = ReleaseLog(self.releases)

    def test_before_first_release(self):
        self.assertIsNone(self.log.version_on("2025-12-31"))

    def test_exact_release_day_is_included(self):
        self.assertEqual(self.log.version_on("2026-03-02"), "1.1.0")

    def test_between_releases(self):
        self.assertEqual(self.log.version_on("2026-02-14"), "1.0.0")

    def test_after_last_release(self):
        self.assertEqual(self.log.version_on("2030-01-01"), "2.0.0")

    def test_matches_linear_scan(self):
        rng = random.Random(21)
        days = [f"2026-{m:02d}-{d:02d}" for m in range(1, 13) for d in range(1, 29)]
        releases = [(rng.choice(days), f"v{i}") for i in range(15)]
        log = ReleaseLog(releases)
        for day in days:
            with self.subTest(day=day):
                self.assertEqual(log.version_on(day), linear_version_on(releases, day))


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 49 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python release_demo.py` and confirm the four lines match the expected output in the notes above.
3. Add a `versions_between(self, start, end)` method to `ReleaseLog` that returns every version released from `start` to `end`, inclusive, using `lower_bound` and `upper_bound`. Test it with a range that contains no releases and a range that contains all of them.
4. Write `docs/searching-notes.md` with the step counts from exercise 2, one paragraph on the recursion limit from exercise 4, and one paragraph on why `upper_bound` is the right tool for "on or before a date".
5. Add a `git bisect`-style function `first_bad(commits, is_bad)` to `dsa/searching.py` that takes a list of commit IDs in order and a function that says whether a commit is bad. It returns the first bad commit, using binary search, and test it against a linear scan. Assume the bad commits form a suffix of the list.
6. Commit with the message "Add searching, recursion and release lookup", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python release_demo.py` prints the four expected lines
- The linear scan comparison passes for every day in the test
- `docs/searching-notes.md` names the recursion depth at which `sum_to` fails
- The latest commit on `main` has a green check

Evidence to commit: `dsa/searching.py`, `dsa/release_lookup.py`, `release_demo.py`, both test files, `docs/searching-notes.md` and the Actions run. The linear scan comparison is the proof that the binary search boundary is correct.

This project is light on new code and heavy on boundaries. The lookup is short, but the same off-by-one reasoning decides correctness in every sorted search you will write.

---

## Check your understanding

1. A sorted list has 1024 items. How many comparisons does binary search need in the worst case, and why does halving the range give a logarithmic bound?
2. A teammate's binary search loops forever on `[1, 2]` looking for 2. Find the likely bug and write the fix.
3. Why does `lower_bound` on a list with duplicates return the first matching index, while `binary_search` may return any matching index?
4. A recursive function on a linked list with 5000 nodes fails with `RecursionError`, but the same logic on a balanced tree with 5000 nodes works. Explain why the depth differs.
5. Explain why `fast_power` makes about log2 of the exponent recursive calls, and why it is faster than multiplying `base` by itself `exp` times.
6. `first_bad` in the exercise assumes the bad commits form a suffix. Name a case where that assumption fails, and say what the function would report.

### Answers

1. About 11 comparisons, since 2 to the 10th is 1024 and each comparison halves the range, so at most 10 halvings plus one final check are needed. The range halves each step, so after k steps the range has about n divided by 2 to the k items, and the search ends when that reaches one.
2. The likely bug is `lo = mid` instead of `lo = mid + 1`, or a similar range update that does not move past the middle. With `lo = 0` and `hi = 1`, the middle is 0, so the search keeps the same range. The fix is to move `lo` to `mid + 1` when the middle is smaller than the target.
3. `lower_bound` returns the leftmost position where the value could be inserted, which is well-defined for duplicates. `binary_search` stops at the first middle it finds equal to the target, which may be any of the duplicates, depending on where the middle falls.
4. A linked list is a chain, so the recursion depth is the length of the chain, 5000. A balanced tree has a depth of about log2 of its node count, which is about 13 for 5000 nodes. The depth depends on the shape of the recursion, not only the number of items.
5. Each recursive call handles half the exponent, so the number of calls is about log2 of exp. Repeated multiplication does exp multiplications, which is linear. For exp equal to 1024, fast power uses about 10 squarings and at most 10 extra multiplications, while the naive method uses 1023.
6. If the bad commits are not a suffix, for example if a commit is bad, then good, then bad again, the binary search can land on a good commit and report a commit that is not the first bad one, or it can report a commit from the wrong side. The function would return a commit, but it would be wrong.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Binary search algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm)
- [Wikipedia: Recursion (computer science)](https://en.wikipedia.org/wiki/Recursion_(computer_science))
- [Wikipedia: Call stack](https://en.wikipedia.org/wiki/Call_stack)
- [Wikipedia: Divide-and-conquer algorithm](https://en.wikipedia.org/wiki/Divide-and-conquer_algorithm)
- The Python documentation for the `bisect` module, at docs.python.org/3/library/bisect.html, which provides the `bisect_left` and `bisect_right` functions used for bounds in the standard library
- The Python documentation for `sys.setrecursionlimit`, at docs.python.org/3/library/sys.html, with its warning about raising the limit

*Tutorials*

- GeeksforGeeks, "Binary Search" and "Recursion in Python" tutorials. Search the site for each title.
- W3Schools, "Python Recursion" tutorial. Search the W3Schools Python section for the title and follow the factorial example.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lecture on binary search and the lectures on divide and conquer. Search the course site for "binary search".
- Abdul Bari on YouTube, the playlist on binary search and recursion. Search YouTube for "Abdul Bari binary search".

*Practice*

- Python Tutor at pythontutor.com. Paste `binary_search` and step through a search on the list `[1, 2]` looking for 2, watching how `lo` and `hi` change.
- LeetCode, the "Binary Search" study plan at leetcode.com. Start with "Binary Search" and "Search Insert Position", which are the two lower-bound problems from this day.

---

## Tomorrow

Day 51 covers backtracking and greedy algorithms. Backtracking is a systematic search through choices, pruned when a partial answer cannot work, and it uses the recursion and call stack from today. Greedy algorithms make the best local choice at each step, which is fast when it is right and misleading when it is not. You will solve N-Queens and subset problems with pruning, and you will find a coin set where the greedy choice fails, which is a counterexample.

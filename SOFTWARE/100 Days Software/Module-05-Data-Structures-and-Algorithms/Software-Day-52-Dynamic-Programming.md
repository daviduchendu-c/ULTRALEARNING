# Day 52: Dynamic Programming

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 4 hours
**Level:** advanced. You should have finished Day 51, be comfortable with recursion and with two-dimensional lists, and be ready to write a small table by hand. This day is for anyone who has seen a recursive solution take minutes on an input that should take milliseconds, and wants to know why and how to fix it.

**By the end of this day, you will have:**

- A clear definition of a DP state, the transitions between states, and the base cases, written down before any code
- Memoized and tabulated versions of Fibonacci, showing the same answer with a reduction from exponential to linear work
- Working solutions for the fewest-coins problem, the longest common subsequence (LCS) and the 0/1 knapsack, each checked against brute force
- A line diff tool that uses the LCS table to report added, removed and unchanged lines
- The diff tool committed with tests and a CI run

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 51
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)
- Paper for the tables, which is the most useful tool for this day

**Prerequisites:**

- Day 41: Complexity Analysis (counting the work of recursive calls)
- Day 50: Searching and Recursion (base cases and recursion depth)
- Day 51: Backtracking and Greedy (the brute-force searches used as references)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [State](#state)
  - [Transitions](#transitions)
  - [Memoization](#memoization)
  - [Tabulation](#tabulation)
  - [Common patterns](#common-patterns)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a line diff built on an LCS table](#project-a-line-diff-built-on-an-lcs-table)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Dynamic programming is the technique for problems where a naive recursion does the same work over and over. A recursive Fibonacci function computes `fib(3)` many times while computing `fib(30)`, and the number of calls grows like the Fibonacci numbers themselves, which is exponential. Dynamic programming notices that the subproblems repeat, computes each one once, stores the answer, and reuses it. The result is often the difference between a program that finishes and one that never does.

The technique appears in many places you use every day. Diff tools that show what changed between two versions of a file use the longest common subsequence. Spell checkers and DNA alignment tools measure how far two strings are from each other with edit distance. Packing problems in logistics and finance are knapsack problems. Route planners and network protocols use shortest-path tables built the same way. Each of them is a small set of rules applied to a table of answers.

The outline asks for state, transitions, memoization, tabulation and common patterns. This day builds those five ideas in order, using a small set of problems that each show one idea clearly. The project turns the LCS table into a diff tool, which is the same algorithm behind line-by-line change views in version control.

---

## Glossary

- **Dynamic programming (DP):** solving a problem by solving its overlapping subproblems once each, and reusing the answers.
- **State:** the smallest set of values that determines the answer to a subproblem, such as an index and a remaining amount.
- **Transition:** the rule that computes a state's answer from the answers to smaller states.
- **Base case:** a state whose answer is known directly, such as `fib(0)` equal to 0.
- **Memoization:** top-down DP. A recursive function stores each answer the first time it computes it, and returns the stored value later.
- **Tabulation:** bottom-up DP. The states are computed in an order where each answer's dependencies are already in the table.
- **Subsequence:** a sequence obtained from another by deleting zero or more items without changing the order of the rest.
- **Longest common subsequence (LCS):** the longest subsequence that appears in two sequences.
- **Knapsack:** the problem of choosing items with weights and values to maximize total value within a weight limit. The 0/1 version allows each item once.
- **Edit distance:** the minimum number of insertions, deletions and substitutions needed to turn one string into another.
- **Overlapping subproblems:** the property that a recursive solution calls the same subproblem more than once. DP needs this property to help.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### State

Think of a travel log for a long road trip. Each entry records where you are and how much fuel is left. Knowing those two things, and nothing else about the trip so far, tells you everything you need to plan the rest. The fuel level and the position are the state. A DP state is the same: the smallest collection of values that fully describes a subproblem.

From first principles, choosing the state is the most important decision in DP. It must contain enough information that the answer depends only on the state, however you reached it. For fewest coins, the state is "the amount still to make", because the number of coins still needed does not depend on which coins were used before. For LCS, the state is a pair of positions, one in each string, because the answer for the rest of the two strings depends only on where each one is. A state that is too small gives wrong answers. A state that is too large wastes memory and time.

```python
# Fewest coins: state = amount remaining.
# fewest[a] answers "how many coins make exactly a?"
# fewest[0] = 0 is the base case, because zero coins make zero.

# LCS: state = (i, j) where i and j are how much of each string is used.
# table[i][j] answers "how long is the LCS of a[:i] and b[:j]?"
```

The limitation is that a bad state can hide the problem. If the state forgets a constraint, such as which items have been used in a knapsack, the answer will depend on history and the table will be wrong. Writing the state in words before writing code is the check.

The common mistake is to use the input directly as the state, such as the whole remaining list of coins, which makes the number of states explode. To spot it, count the states on paper. If the count is a product of several independent sizes, you probably included too much.

Choosing a state is the design step of every real DP. Compilers use states that track positions in a parse, and version control systems use states that track positions in two files.

### Transitions

Imagine a chain of stepping stones across a stream. To reach stone k, you must have come from stone k minus 1, or from stone k minus 2. Once you know the best way to reach each of those, the best way to reach k is one step past the better of them. That rule is a transition. It turns answers for smaller states into the answer for the current state.

From first principles, a transition is a formula that refers only to states that are already solved. For fewest coins, the transition is: the best answer for amount a is one more than the best answer for a minus some coin, minimized over the coins that fit. For LCS, if the characters at the two positions match, the answer is one more than the answer for both positions moved back, and otherwise it is the larger of the two answers with one position moved back. For 0/1 knapsack, each item either fits and improves the total, or it is skipped.

```python
# Fewest coins transition:
# fewest[a] = min(fewest[a - c] + 1 for each coin c with c <= a)

# LCS transition:
# if a[i-1] == b[j-1]: table[i][j] = table[i-1][j-1] + 1
# else:                table[i][j] = max(table[i-1][j], table[i][j-1])
```

The limitation is that the transition must be correct for every state, including the edge states. A transition that reads `fewest[a - c]` for a coin larger than a produces a negative index, which Python silently reads from the end of the list. That is a serious bug, and the guard `c <= a` prevents it.

The common mistake is to write a transition that depends on a state not yet computed, which gives a value of zero or infinity by accident. To spot it, trace the table on a small input by hand, and check that every cell reads only cells already filled.

Transitions are how a spell checker scores a candidate correction and how a scheduling service decides whether a job fits in a time window.

### Memoization

Picture a cook writing each recipe result on a sticky note and checking the notes before starting a new dish. If the note is there, the dish is ready. Memoization is that habit in code: a recursive function checks a cache before computing, and stores the result after. The recursion stays exactly as the transition describes it, and the cache removes the repeated work.

From first principles, a memoized function calls itself on smaller states, just as a naive recursion does, but each state is computed at most once. The total work is the number of states times the work per state. For Fibonacci, there are n states and constant work per state, so the total is O(n), down from the exponential naive version. Python's `functools.cache` decorator provides this behavior in one line, and the explicit dictionary below shows what it does.

```python
def fib_memo(n: int) -> int:
    memo: dict[int, int] = {}

    def go(k: int) -> int:
        if k < 2:
            return k
        if k not in memo:
            memo[k] = go(k - 1) + go(k - 2)
        return memo[k]

    return go(n)

print(fib_memo(80))   # 23416728348467685, computed instantly
```

The limitation of memoization is recursion depth, which is the same limit you saw on Day 50. Each new level of the recursion is a frame on the call stack, so `fib_memo(2000)` raises `RecursionError` even though the work is small. The tabulated version avoids that.

The common mistake is to memoize with a key that does not capture the whole state. If the cache key for fewest coins is the amount but the function also depends on the coin set, the cache returns wrong answers when the coins change. To spot it, call the function with two different coin sets and check the results.

Memoization is used in autocompletion engines, in parsers that cache sub-expressions, and in the `functools.cache` decorator that many Python projects use for expensive pure functions.

### Tabulation

Think of filling in a spreadsheet from the top row down, where each cell uses only cells above it. You never need to recurse, because you always compute the cells that others depend on first. Tabulation is that approach: order the states so every dependency comes before the state that needs it, and fill a table in that order.

From first principles, tabulation removes recursion and its depth limit. It usually runs faster than memoization because it avoids function call overhead, and it is easier to reason about memory, because the table has a known size. The cost is that you must find an order that respects the dependencies, which is sometimes harder than writing the recursion. For Fibonacci, the order is simply increasing n. The version below keeps only the last two values, which reduces the memory from n values to constant.

```python
def fib_table(n: int) -> int:
    if n < 2:
        return n
    prev, cur = 0, 1
    for _ in range(n - 1):
        prev, cur = cur, prev + cur
    return cur

print(fib_table(80))           # 23416728348467685
print(fib_table(2000) > 0)     # True, with no recursion limit issue
```

The limitation of tabulation is that it computes every state, even ones that the final answer does not need. Memoization computes only the states reached from the answer. When most states are unneeded, memoization can be faster. Tabulation is safer for predictable performance and for avoiding stack overflow.

The common mistake is to fill the table in the wrong order, so a cell is read before it is written and holds a zero or a placeholder. To spot it, initialize the table with a clear sentinel, such as infinity or -1, and check that no answer contains it.

Tabulation is how many numerical libraries compute recurrences, and how a video encoder computes the best block sizes for each frame region.

### Common patterns

Think of a set of recipes that all use the same few techniques, such as sautéing and reducing. Once you know the techniques, you can cook many dishes you have never seen. DP problems are similar. Many of them reduce to a handful of patterns, and recognizing the pattern is most of the work.

From first principles, the patterns in this day are four. The first is a one-dimensional state over a number, such as Fibonacci and fewest coins. The second is a two-dimensional state over two sequences, such as LCS and edit distance. The third is a state over items and capacity, such as 0/1 knapsack. The fourth is the fill-in-once trick, where a single pass with a rolling array replaces a full table. The knapsack code below uses a one-dimensional array over capacity, with the inner loop running backward so each item is used once.

```python
def knapsack(items: list[tuple[int, int]], capacity: int) -> int:
    """items are (weight, value). Best total value with total weight at most capacity."""
    best = [0] * (capacity + 1)
    for weight, value in items:
        for cap in range(capacity, weight - 1, -1):
            best[cap] = max(best[cap], best[cap - weight] + value)
    return best[capacity]

print(knapsack([(1, 1), (3, 4), (4, 5), (5, 7)], 7))   # 9, from items of weight 3 and 4
```

The limitation of pattern matching is that a new problem can look like a known one and still need a different state. The knapsack loop above goes backward because forward would let an item be used twice, which is the difference between 0/1 knapsack and unbounded knapsack. Small changes in the loop direction change the problem.

The common mistake is to go forward in the knapsack loop, which reuses an item in the same pass. To spot it, test an input with one item that fits twice, such as weight 1 and value 5 with capacity 2. The answer should be 5, not 10.

Common DP patterns show up in version control, in genome analysis, in logistics planning and in finance, where a portfolio must fit a budget.

---

## Hands-on exercises

Work in the repository from Day 51. Keep the DP functions in `dsa/dynamic.py` and the diff tool in `dsa/diff.py`.

Exercise 1 writes states on paper. For the fewest-coins problem with coins 1, 3 and 4, draw a table with amounts 0 to 8 and fill in the answer for each by hand, using the transition. Check your row against `fewest_coins`.

Done when your table matches the function for every amount, and each cell cites the cell it came from.

Exercise 2 counts the naive work. Write a naive recursive Fibonacci and count its calls for n equal to 10, 20 and 25. Record the counts in `docs/dp-notes.md`, then compare with `fib_memo`, using a counter in a copy.

Done when your notes show the exponential growth of the naive count and the linear count for the memoized version.

Exercise 3 finds the recursion limit. Call `fib_memo` for n equal to 900 and then 1200. Record which one fails and read the traceback. Then call `fib_table` for n equal to 1200.

Done when your notes name the failing input and the table version succeeds.

Exercise 4 checks LCS by brute force. Write `brute_lcs(a, b)` in the test file that tries every subsequence of `a` and checks whether it is a subsequence of `b`. Compare it with `lcs_length` on 100 random pairs of strings of length 0 to 6 over the letters "abc", using `random.Random(52)`.

Done when every pair agrees, and your notes name one pair where the LCS is not unique.

Exercise 5 checks knapsack by brute force. Use `subsets` from the backtracking module to find the best value by trying every subset under the capacity, and compare with `knapsack` on 100 random item lists of at most 8 items.

Done when every case agrees, and your notes explain why the backward loop is required.

---

## Project: a line diff built on an LCS table

The project is a line diff tool. Given two versions of a text as lists of lines, it reports which lines were kept, removed or added, in order. It is the same idea that powers the changed-lines view in version control. The core is the LCS table: lines in the longest common subsequence are kept, and the rest are removed or added. The tests check that the kept and removed lines rebuild the old text, and that the kept and added lines rebuild the new text.

Suggested repo name: `100-days-software-day-52-dynamic-programming`. Copy the Day 51 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-52-dynamic-programming/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── build_demo.py
├── count_words.py
├── diff_demo.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── backtracking.py
│   ├── bench.py
│   ├── bst.py
│   ├── browser_history.py
│   ├── build_order.py
│   ├── complexity.py
│   ├── diff.py
│   ├── dynamic.py
│   ├── graph.py
│   ├── greedy.py
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
│   ├── dp-notes.md
│   ├── graph-notes.md
│   ├── greedy-notes.md
│   ├── searching-notes.md
│   └── sorting-notes.md
├── height_demo.py
├── leaderboard_demo.py
├── meetings_demo.py
├── print_queue.py
├── release_demo.py
├── schedule_demo.py
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_backtracking.py
│   ├── test_bench.py
│   ├── test_bst.py
│   ├── test_browser_history.py
│   ├── test_build_order.py
│   ├── test_complexity.py
│   ├── test_diff.py
│   ├── test_dynamic.py
│   ├── test_graph.py
│   ├── test_greedy.py
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

Starter code for `dsa/dynamic.py`:

```python
from collections.abc import Sequence


def fib_memo(n: int) -> int:
    memo: dict[int, int] = {}

    def go(k: int) -> int:
        if k < 2:
            return k
        if k not in memo:
            memo[k] = go(k - 1) + go(k - 2)
        return memo[k]

    return go(n)


def fib_table(n: int) -> int:
    if n < 2:
        return n
    prev, cur = 0, 1
    for _ in range(n - 1):
        prev, cur = cur, prev + cur
    return cur


def fewest_coins(coins: list[int], amount: int) -> int | None:
    """Fewest coins that sum to amount, or None if no combination works."""
    inf = float("inf")
    best: list[float] = [0] + [inf] * amount
    for a in range(1, amount + 1):
        for coin in coins:
            if coin <= a and best[a - coin] + 1 < best[a]:
                best[a] = best[a - coin] + 1
    return None if best[amount] == inf else int(best[amount])


def lcs_table(a: Sequence, b: Sequence) -> list[list[int]]:
    """table[i][j] is the LCS length of a[:i] and b[:j]."""
    m, n = len(a), len(b)
    table = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i - 1] == b[j - 1]:
                table[i][j] = table[i - 1][j - 1] + 1
            else:
                table[i][j] = max(table[i - 1][j], table[i][j - 1])
    return table


def lcs_length(a: Sequence, b: Sequence) -> int:
    return lcs_table(a, b)[len(a)][len(b)]


def knapsack(items: list[tuple[int, int]], capacity: int) -> int:
    """items are (weight, value). Best total value with total weight at most capacity."""
    best = [0] * (capacity + 1)
    for weight, value in items:
        for cap in range(capacity, weight - 1, -1):
            best[cap] = max(best[cap], best[cap - weight] + value)
    return best[capacity]
```

Starter code for `dsa/diff.py`:

```python
from dsa.dynamic import lcs_table


def line_diff(old: list[str], new: list[str]) -> list[tuple[str, str]]:
    """Return (op, line) pairs in order. op is '=' for kept, '-' for removed, '+' for added."""
    table = lcs_table(old, new)
    i, j = len(old), len(new)
    ops: list[tuple[str, str]] = []
    while i > 0 or j > 0:
        if i > 0 and j > 0 and old[i - 1] == new[j - 1]:
            ops.append(("=", old[i - 1]))
            i -= 1
            j -= 1
        elif j > 0 and (i == 0 or table[i][j - 1] >= table[i - 1][j]):
            ops.append(("+", new[j - 1]))
            j -= 1
        else:
            ops.append(("-", old[i - 1]))
            i -= 1
    ops.reverse()
    return ops
```

Starter code for `diff_demo.py`:

```python
from dsa.diff import line_diff

if __name__ == "__main__":
    old = ["def total(items):", "    sum = 0", "    for x in items:", "        sum += x", "    return sum"]
    new = ["def total(items):", "    result = 0", "    for x in items:", "        result += x", "    return result"]
    marks = {"=": "  ", "-": "- ", "+": "+ "}
    for op, line in line_diff(old, new):
        print(marks[op] + line)
```

Running `python diff_demo.py` prints the new file with added and removed lines marked, and the unchanged lines with two leading spaces.

Starter code for `tests/test_dynamic.py`:

```python
import itertools
import random
import unittest

from dsa.backtracking import fewest_coins_bruteforce, subsets
from dsa.dynamic import fib_memo, fib_table, fewest_coins, knapsack, lcs_length


def brute_lcs(a: str, b: str) -> int:
    best = 0
    for k in range(len(a) + 1):
        for idx in itertools.combinations(range(len(a)), k):
            sub = [a[i] for i in idx]
            it = iter(b)
            if all(ch in it for ch in sub):
                best = max(best, k)
    return best


class FibonacciTests(unittest.TestCase):
    def test_known_values(self):
        self.assertEqual(fib_memo(10), 55)
        self.assertEqual(fib_table(10), 55)
        self.assertEqual(fib_memo(80), fib_table(80))

    def test_memo_and_table_agree(self):
        for n in range(0, 60):
            self.assertEqual(fib_memo(n), fib_table(n))


class CoinTests(unittest.TestCase):
    def test_matches_brute_force(self):
        for amount in range(0, 16):
            with self.subTest(amount=amount):
                self.assertEqual(
                    fewest_coins([1, 3, 4], amount),
                    fewest_coins_bruteforce([1, 3, 4], amount),
                )

    def test_unreachable_returns_none(self):
        self.assertIsNone(fewest_coins([2], 3))


class LcsTests(unittest.TestCase):
    def test_classic_example(self):
        self.assertEqual(lcs_length("ABCBDAB", "BDCABA"), 4)

    def test_matches_brute_force(self):
        rng = random.Random(52)
        for _ in range(100):
            a = "".join(rng.choice("abc") for _ in range(rng.randint(0, 6)))
            b = "".join(rng.choice("abc") for _ in range(rng.randint(0, 6)))
            with self.subTest(a=a, b=b):
                self.assertEqual(lcs_length(a, b), brute_lcs(a, b))


class KnapsackTests(unittest.TestCase):
    def test_classic_example(self):
        self.assertEqual(knapsack([(1, 1), (3, 4), (4, 5), (5, 7)], 7), 9)

    def test_each_item_used_once(self):
        self.assertEqual(knapsack([(1, 5)], 2), 5)

    def test_matches_brute_force(self):
        rng = random.Random(53)
        for _ in range(100):
            items = [(rng.randint(1, 6), rng.randint(1, 9)) for _ in range(rng.randint(0, 6))]
            capacity = rng.randint(0, 12)
            best = 0
            for group in subsets(items):
                if sum(w for w, _ in group) <= capacity:
                    best = max(best, sum(v for _, v in group))
            with self.subTest(items=items, capacity=capacity):
                self.assertEqual(knapsack(items, capacity), best)


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_diff.py`:

```python
import random
import unittest

from dsa.diff import line_diff


class LineDiffTests(unittest.TestCase):
    def test_simple_change(self):
        ops = line_diff(["a", "b", "c"], ["a", "c", "d"])
        self.assertEqual(ops, [("=", "a"), ("-", "b"), ("=", "c"), ("+", "d")])

    def test_identical_files_have_only_kept_lines(self):
        lines = ["x", "y"]
        self.assertEqual(line_diff(lines, lines), [("=", "x"), ("=", "y")])

    def test_empty_inputs(self):
        self.assertEqual(line_diff([], []), [])
        self.assertEqual(line_diff([], ["a"]), [("+", "a")])
        self.assertEqual(line_diff(["a"], []), [("-", "a")])

    def test_ops_rebuild_both_versions(self):
        rng = random.Random(7)
        for _ in range(200):
            old = [rng.choice("abc") for _ in range(rng.randint(0, 8))]
            new = [rng.choice("abc") for _ in range(rng.randint(0, 8))]
            ops = line_diff(old, new)
            with self.subTest(old=old, new=new):
                self.assertEqual([line for op, line in ops if op in "=-"], old)
                self.assertEqual([line for op, line in ops if op in "=+"], new)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 51 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python diff_demo.py` on the sample. Check that the lines that changed are exactly the ones you expect from reading the two versions.
3. Add a `edit_distance(a: str, b: str) -> int` function to `dsa/dynamic.py` using a table of size (len(a) + 1) by (len(b) + 1), with insert, delete and substitute costs of 1. Test it on "kitten" and "sitting", which should give 3, and against a brute force on short strings.
4. Write `docs/dp-notes.md` with the counts from exercise 2, one paragraph on why `fib_table` survives n equal to 1200 and `fib_memo` does not, and one paragraph on the transition for LCS.
5. Add a `--stats` option to `diff_demo.py` that prints the number of kept, removed and added lines. Test the function that computes the counts, not the command line.
6. Commit with the message "Add dynamic programming and line diff", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python diff_demo.py` marks the changed lines correctly
- The brute-force comparisons for LCS and knapsack pass for every random case
- `docs/dp-notes.md` names the input at which `fib_memo` fails
- The latest commit on `main` has a green check

Evidence to commit: `dsa/dynamic.py`, `dsa/diff.py`, `diff_demo.py`, both test files, `docs/dp-notes.md` and the Actions run. The brute-force comparisons are the evidence that the DP tables are correct, and the diff property test is the evidence that the tool never drops or invents a line.

This project is light on new theory and heavy on the table. The skill is writing the state, the transition and the base case before writing code, and checking each table against a brute force.

---

## Check your understanding

1. For fewest coins, why is "amount remaining" enough as the state, when the coins used so far are not part of it?
2. A memoized Fibonacci and a naive recursive Fibonacci both return the same answer. Explain why the memoized one does far less work, and what the work is proportional to.
3. Explain why the knapsack loop over capacity runs backward, and what changes if it runs forward.
4. In `line_diff`, why does the backtrack prefer an insertion when `table[i][j - 1]` is at least `table[i - 1][j]`?
5. A teammate claims LCS with the table takes O(m times n) time and O(m times n) space. Confirm the time, and explain how the space could be reduced to O(min(m, n)) for the length alone, and why the diff tool still needs the full table.
6. Give one reason tabulation is preferred over memoization in a long-running service, and one reason memoization can be faster.

### Answers

1. The number of coins still needed to make the remaining amount does not depend on which coins were used earlier. Any way of making the remaining amount with the fewest coins works equally well, so the history is not needed. That independence is what makes the state sufficient.
2. The naive version recomputes the same subproblems, so its number of calls grows like the Fibonacci numbers, which is exponential in n. The memoized version computes each of the n+1 subproblems once, so its work is proportional to n, with constant work per subproblem.
3. The loop goes backward so that `best[cap - weight]` still holds the value from before this item was considered. Going forward would use an updated value from the same item, which allows the item to be counted twice. Forward iteration solves the unbounded knapsack instead.
4. When no lines match at this position, the table tells the backtrack which way still reaches the best common subsequence. If the cell to the left is at least as large as the cell above, dropping the new line keeps the LCS length, so the code records an insertion. The rule is a fixed tie-break, which makes the output deterministic. Either direction gives a correct diff, because both keep the full LCS.
5. Filling the table takes one pass over each of the m times n cells, so O(m times n) time is right. For length alone, only two rows are needed at a time, so the space drops to O(min(m, n)) after swapping so that the shorter string is the column dimension. The diff tool needs the full table, because it walks back from the end to recover which lines are in the common subsequence.
6. Tabulation avoids recursion, so it cannot hit the recursion limit, and its memory use is predictable, which suits a long-running service. Memoization can be faster when many states are never needed, because it computes only the states reached from the answer.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Dynamic programming](https://en.wikipedia.org/wiki/Dynamic_programming)
- [Wikipedia: Longest common subsequence](https://en.wikipedia.org/wiki/Longest_common_subsequence)
- [Wikipedia: Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem)
- [Wikipedia: Edit distance](https://en.wikipedia.org/wiki/Edit_distance)
- [Wikipedia: Diff](https://en.wikipedia.org/wiki/Diff), which describes how line diffs are used in version control
- The Python documentation for `functools.cache`, at docs.python.org/3/library/functools.html

*Tutorials*

- GeeksforGeeks, "Dynamic Programming" and "Longest Common Subsequence Problem" tutorials. Search the site for each title.
- W3Schools, the Python section on recursion, which covers the recursive pattern that memoization speeds up. Search W3Schools for "Python recursion".

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on dynamic programming, including the sections on subproblems and the LCS. Search the course site for "dynamic programming".
- William Fiset on YouTube, the playlist on dynamic programming, including the knapsack and LCS videos. Search YouTube for "William Fiset dynamic programming".

*Practice*

- LeetCode, the "Dynamic Programming" study plan at leetcode.com. Start with "Climbing Stairs" and "Coin Change", then solve "Longest Common Subsequence".
- Python Tutor at pythontutor.com. Paste `lcs_table` and step through the loop on the strings "ab" and "ba" to watch each cell fill in.

---

## Tomorrow

Day 53 is the DSA assessment, the checkpoint for the module. You will work mixed problems from the five days on arrays, linked structures, hashing, trees, heaps, graphs, sorting, search, greedy and dynamic programming. For each problem you will write the complexity, explain why your approach works, compare it with an alternative, and solve it both untimed and under a clock. The repository from today already contains the test habits you will use, and the assessment harness you build tomorrow runs the same kind of checks.

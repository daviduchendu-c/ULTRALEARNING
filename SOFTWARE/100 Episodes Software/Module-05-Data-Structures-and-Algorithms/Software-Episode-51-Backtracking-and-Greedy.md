# Day 51: Backtracking and Greedy

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** intermediate. You should have finished Day 50, be comfortable with recursion, and be willing to test an idea against a brute-force check before you trust it. This day is for anyone who has been asked to "find the best schedule" or "make change" and has written a loop that looked right but was wrong.

**By the end of this day, you will have:**

- A backtracking search that explores choices as a tree, with pruning that cuts off dead ends early
- N-Queens solution counts, permutations and subsets, each checked against the standard library
- A greedy scheduler that picks the largest set of non-overlapping meetings, proven by an exchange argument and checked by brute force
- A concrete coin set where the greedy choice fails, and a brute-force search that finds the correct answer
- A meeting picker project with tests, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 50
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 46: Trees and BSTs (the idea of a tree of choices, where each node is a decision)
- Day 49: Sorting (sorting by a key, which the greedy algorithms rely on)
- Day 50: Searching and Recursion (base cases, the call stack and recursion depth)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Search trees](#search-trees)
  - [Pruning](#pruning)
  - [Greedy-choice intuition](#greedy-choice-intuition)
  - [Counterexamples](#counterexamples)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a meeting picker with brute-force checks](#project-a-meeting-picker-with-brute-force-checks)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Many real problems ask for the best combination out of a huge number of options. Which subset of tasks fits in a day? Where can eight queens sit on a board without attacking each other? Which coins make a total with the fewest pieces? Trying every combination works for small inputs, and it stops working quickly as the input grows. Two techniques handle these problems in practice, and they make opposite bets.

Backtracking is a careful search. It builds a partial answer one choice at a time, and when a partial answer can no longer lead to a valid result, it undoes the last choice and tries another. It finds every valid answer, or proves none exists. Its speed comes from pruning, which means cutting off whole branches of the search early. Greedy algorithms make a different bet. They choose the locally best option at each step and never reconsider. When the bet is right, the result is fast and simple, and when it is wrong, the result is a valid answer that is not the best.

Both techniques show up in real software. Backtracking drives constraint solvers, regular expression engines, puzzle solvers and dependency resolvers. Greedy choices drive scheduling, data compression with Huffman codes, network routing trees and interval planning in calendars. The skill this day builds is knowing when a greedy choice is safe, and proving it, and recognizing the counterexample when it is not.

---

## Glossary

- **Backtracking:** a search that extends a partial solution one choice at a time and undoes choices that lead to a dead end.
- **State space tree:** the tree of all partial choices, where each path from the root is one partial solution.
- **Pruning:** skipping a branch of the search tree because no solution in it can be valid or better than the best found.
- **Permutation:** an ordering of all the items in a collection.
- **Subset:** any selection of items from a collection, including the empty selection and the full set.
- **Greedy algorithm:** an algorithm that makes the locally best choice at each step and does not undo it.
- **Greedy-choice property:** the property that a locally best choice can always be part of some globally best solution.
- **Exchange argument:** a proof technique that shows an optimal solution can be changed to use the greedy choice without getting worse.
- **Counterexample:** a specific input that shows a claimed algorithm or property is false.
- **Brute force:** checking every possibility. It is slow, but it is simple enough to be trusted as a reference.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Search trees

Imagine planning a route through a maze with forks. At each fork you pick a corridor, and you keep a note of which corridors you have tried. When you reach a dead end, you walk back to the last fork and try the next corridor. The whole set of possible routes is a tree, with the start at the root and each fork as a node. Backtracking is a walk through that tree that visits each route once.

From first principles, the search tree has one level for each decision. For permutations of n items, the first level has n choices, the next has n minus 1, and so on, so there are n factorial leaves. Backtracking is a depth-first walk of that tree. It keeps one partial answer in memory, adds a choice, recurses, then removes the choice before trying the next one. That undo step is the backtrack, and it is what keeps memory small. Without it, the algorithm would need a copy of the partial answer at every node.

```python
def permutations(items: list) -> list[list]:
    result: list[list] = []
    used = [False] * len(items)
    current: list = []

    def extend() -> None:
        if len(current) == len(items):
            result.append(list(current))
            return
        for i, item in enumerate(items):
            if used[i]:
                continue
            used[i] = True
            current.append(item)
            extend()
            current.pop()
            used[i] = False

    extend()
    return result

print(permutations([1, 2, 3]))   # six orderings, from [1, 2, 3] to [3, 2, 1]
```

The limitation is that the tree has n factorial leaves. Backtracking cannot shrink that count. It only keeps the work per leaf small and avoids storing every partial answer. For n equal to 12, there are about 479 million permutations, so exhaustive listing is not practical, and you need pruning or a different method.

The common mistake is to forget the undo step, so `current` keeps items from a previous branch and the output contains wrong lists. To spot it, print the partial answer at each step on a three-item example. If a list grows past its limit, the undo is missing.

Search trees with backtracking are used in SAT solvers that check logic puzzles, in sudoku and crossword generators, and in the parsers of some regular expression engines.

### Pruning

Think of a chess player who sees that a move loses a queen and does not even consider the moves after it. That is pruning. The player does not reject the move by playing it out. They reject it because no continuation can help. In backtracking, pruning means checking whether a partial answer can still become valid before going deeper.

From first principles, pruning is the reason backtracking is practical. A check that costs a little at each node can cut off a whole subtree. For N-Queens, the rule is simple. A queen placed in a row cannot share a column or a diagonal with any queen already placed. Checking that rule before placing the queen removes every branch that would break it. The function below uses three sets to make each check constant time.

```python
def n_queens_count(n: int) -> int:
    cols: set[int] = set()
    diag_down: set[int] = set()   # row minus column is constant along a diagonal
    diag_up: set[int] = set()     # row plus column is constant along the other diagonal

    def place(row: int) -> int:
        if row == n:
            return 1
        total = 0
        for col in range(n):
            if col in cols or (row - col) in diag_down or (row + col) in diag_up:
                continue                      # prune: this square is under attack
            cols.add(col)
            diag_down.add(row - col)
            diag_up.add(row + col)
            total += place(row + 1)
            cols.remove(col)
            diag_down.remove(row - col)
            diag_up.remove(row + col)
        return total

    return place(0)

print([n_queens_count(n) for n in range(1, 9)])   # [1, 0, 0, 2, 10, 4, 40, 92]
```

The limitation of pruning is that it depends on the problem. A check that is cheap and strong, such as the attack rule, cuts many branches. A check that is expensive or weak cuts few, and can cost more than it saves. The art is to find a test that is both cheap and early.

The common mistake is to prune too late, after the recursive call instead of before it, which keeps the search tree the same size. To spot it, count the calls to `place` for n equal to 6 with and without the prune. The counts should be very different.

Pruning is used in constraint solvers for scheduling and configuration, in game engines for move search with alpha-beta pruning, and in compilers that check type constraints during overload resolution.

### Greedy-choice intuition

Think of a cashier who gives change with the largest coin first, or a traveler who always takes the nearest unvisited town next. Both are reasonable habits. Greedy algorithms formalize the habit: at each step take the option that looks best now, and never reconsider. The question is whether the local best always leads to the global best.

From first principles, a greedy algorithm is correct for a problem when two things hold. The first is the greedy-choice property: some optimal solution agrees with the greedy choice at the first step. The second is optimal substructure: after that choice, the rest of the problem is smaller and has the same form. For interval scheduling, the greedy choice is the meeting that ends first, and an exchange argument shows that swapping it into any optimal schedule does not make the schedule worse. The method below sorts by end time and keeps every meeting that starts after the last one kept.

```python
def max_non_overlapping(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    chosen: list[tuple[int, int]] = []
    last_end = float("-inf")
    for start, end in sorted(intervals, key=lambda iv: iv[1]):
        if start >= last_end:
            chosen.append((start, end))
            last_end = end
    return chosen

meetings = [(9, 10), (9, 11), (10, 12), (11, 13), (12, 13), (13, 14)]
print(max_non_overlapping(meetings))   # [(9, 10), (10, 12), (12, 13), (13, 14)]
```

The limitation is that the proof is specific. Choosing the meeting that starts first, or the shortest meeting, does not work for every input. A short meeting can sit in the middle of two long ones and block both. Earliest finish is the rule that is safe here, and you should test each greedy rule against a brute-force check before you rely on it.

The common mistake is to pick the rule by intuition and skip the check. To spot the problem, write the brute-force version and compare on random inputs. A mismatch means the rule is wrong, even if the code looks right.

Greedy scheduling is used in operating system schedulers that pick the job with the earliest deadline, in network packet scheduling and in calendar apps that show the largest number of conflict-free events.

### Counterexamples

Think of a claim in a meeting: "the biggest coin is always the right first choice." A counterexample is a single case that breaks the claim. One case is enough to refute a claim, and no number of passing cases proves it. That makes counterexamples the most useful tool for checking a greedy idea.

From first principles, coin change with a greedy rule takes the largest coin that fits, repeatedly. For a coin system such as 1, 5, 10 and 25 cents, that rule gives the fewest coins, which is why it works for everyday money. For the coin set 1, 3 and 4, it fails. To make 6, the greedy rule takes 4, then 1, then 1, which is three coins. The optimal answer is 3 and 3, which is two coins. The brute-force search below finds the optimum by trying every choice.

```python
def coin_change_greedy(coins: list[int], amount: int) -> list[int]:
    chosen: list[int] = []
    for coin in sorted(coins, reverse=True):
        while amount >= coin:
            chosen.append(coin)
            amount -= coin
    if amount != 0:
        raise ValueError("the greedy rule could not make the exact amount")
    return chosen

def fewest_coins_bruteforce(coins: list[int], amount: int) -> int | None:
    best: int | None = None

    def search(remaining: int, used: int) -> None:
        nonlocal best
        if remaining == 0:
            if best is None or used < best:
                best = used
            return
        for coin in coins:
            if coin <= remaining:
                search(remaining - coin, used + 1)

    search(amount, 0)
    return best

print(coin_change_greedy([1, 3, 4], 6))          # [4, 1, 1], three coins
print(fewest_coins_bruteforce([1, 3, 4], 6))     # 2, from 3 and 3
```

The limitation of brute force is cost. The search above tries every ordered sequence of coins, which grows fast with the amount. It is a reference for small inputs and is not meant for production. The right production tool for the general problem is dynamic programming, which Day 52 covers.

The common mistake is to think a counterexample means the whole approach is useless. It means the greedy rule is not valid for that coin set. Greedy still works for canonical coin systems and for many scheduling problems, but the proof must match the problem.

Counterexamples are how engineers check cost rules in pricing engines, how security reviewers find inputs that break a validation rule, and how researchers show that a plausible algorithm is not correct.

---

## Hands-on exercises

Work in the repository from Day 50. Keep the search code in `dsa/backtracking.py`, the greedy code in `dsa/greedy.py`, and the meeting demo at the root.

Exercise 1 traces permutations. On paper, write the state of `current` and `used` for the first six steps of `permutations([1, 2, 3])`. Mark each undo. Then compare your sequence of completed lists with the output.

Done when your list of completed permutations matches the code in the same order.

Exercise 2 counts pruning. Add a counter to `n_queens_count` and record the number of calls to `place` for n equal to 6. Then remove the prune checks, count again, and record both numbers in `docs/greedy-notes.md`.

Done when your notes show the two counts and the reduction, and you can state which prune check removes the most branches.

Exercise 3 checks subsets. Test `subsets([1, 2, 3])` against `itertools.combinations` for every size, and confirm the total count is 2 to the n for n from 0 to 6.

Done when the test passes and the count matches for every n.

Exercise 4 tests the greedy rule. Write a brute-force function that finds the largest set of non-overlapping meetings by trying every subset, using `subsets` from the backtracking module. Compare it with `max_non_overlapping` on 200 random lists of at most eight meetings, generated with `random.Random(51)`, with starts and ends between 0 and 20 and start less than end.

Done when the sizes match in every case. Note in your notes one rule you considered and rejected, such as earliest start, with a counterexample.

Exercise 5 finds a counterexample. Write a test that shows the greedy coin rule fails for coins 1, 3 and 4 with amount 6, and that greedy succeeds for the coins 1, 5, 10 and 25 for amounts 1 to 30. Use the brute force for the first case and for amounts up to 12 in the second, and explain in your notes why the brute force cannot run for amount 30.

Done when both claims are tested, and your notes state the counterexample in one sentence.

---

## Project: a meeting picker with brute-force checks

The project is a meeting picker. Given a list of meetings, each with a start and an end hour, it chooses the largest set that does not overlap, so one person can attend as many as possible. It prints the chosen meetings in order. The tests compare the answer with a brute force over all subsets, so the greedy rule is checked against the true answer.

Suggested repo name: `100-days-software-day-51-backtracking-and-greedy`. Copy the Day 50 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-51-backtracking-and-greedy/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── build_demo.py
├── count_words.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── backtracking.py
│   ├── bench.py
│   ├── bst.py
│   ├── browser_history.py
│   ├── build_order.py
│   ├── complexity.py
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

Starter code for `dsa/backtracking.py`:

```python
from typing import Any


def n_queens_count(n: int) -> int:
    """Number of ways to place n non-attacking queens on an n by n board."""
    cols: set[int] = set()
    diag_down: set[int] = set()
    diag_up: set[int] = set()

    def place(row: int) -> int:
        if row == n:
            return 1
        total = 0
        for col in range(n):
            if col in cols or (row - col) in diag_down or (row + col) in diag_up:
                continue
            cols.add(col)
            diag_down.add(row - col)
            diag_up.add(row + col)
            total += place(row + 1)
            cols.remove(col)
            diag_down.remove(row - col)
            diag_up.remove(row + col)
        return total

    return place(0)


def permutations(items: list[Any]) -> list[list[Any]]:
    result: list[list[Any]] = []
    used = [False] * len(items)
    current: list[Any] = []

    def extend() -> None:
        if len(current) == len(items):
            result.append(list(current))
            return
        for i, item in enumerate(items):
            if used[i]:
                continue
            used[i] = True
            current.append(item)
            extend()
            current.pop()
            used[i] = False

    extend()
    return result


def subsets(items: list[Any]) -> list[list[Any]]:
    result: list[list[Any]] = []
    current: list[Any] = []

    def choose(start: int) -> None:
        result.append(list(current))
        for i in range(start, len(items)):
            current.append(items[i])
            choose(i + 1)
            current.pop()

    choose(0)
    return result


def fewest_coins_bruteforce(coins: list[int], amount: int) -> int | None:
    """Reference answer for small amounts: tries every sequence of coins."""
    best: int | None = None

    def search(remaining: int, used: int) -> None:
        nonlocal best
        if remaining == 0:
            if best is None or used < best:
                best = used
            return
        for coin in coins:
            if coin <= remaining:
                search(remaining - coin, used + 1)

    search(amount, 0)
    return best
```

Starter code for `dsa/greedy.py`:

```python
def max_non_overlapping(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    """Largest set of intervals that do not overlap. Two intervals may touch:
    one may start at the moment the previous one ends."""
    chosen: list[tuple[int, int]] = []
    last_end = float("-inf")
    for start, end in sorted(intervals, key=lambda iv: iv[1]):
        if start >= last_end:
            chosen.append((start, end))
            last_end = end
    return chosen


def coin_change_greedy(coins: list[int], amount: int) -> list[int]:
    """Take the largest coin that fits, repeatedly. Optimal only for some coin sets."""
    chosen: list[int] = []
    for coin in sorted(coins, reverse=True):
        while amount >= coin:
            chosen.append(coin)
            amount -= coin
    if amount != 0:
        raise ValueError("the greedy rule could not make the exact amount")
    return chosen
```

Starter code for `meetings_demo.py`:

```python
from dsa.greedy import max_non_overlapping

if __name__ == "__main__":
    meetings = [(9, 10), (9, 11), (10, 12), (11, 13), (12, 13), (13, 14)]
    chosen = max_non_overlapping(meetings)
    print(f"You can attend {len(chosen)} meetings:")
    for start, end in chosen:
        print(f"  {start:02d}:00 to {end:02d}:00")
```

Running `python meetings_demo.py` prints four meetings: 09:00 to 10:00, 10:00 to 12:00, 12:00 to 13:00 and 13:00 to 14:00. The 10:00 to 12:00 meeting and the 12:00 to 13:00 meeting touch at 12:00, which the picker allows.

Starter code for `tests/test_backtracking.py`:

```python
import itertools
import unittest

from dsa.backtracking import fewest_coins_bruteforce, n_queens_count, permutations, subsets


class BacktrackingTests(unittest.TestCase):
    def test_n_queens_known_counts(self):
        expected = [1, 0, 0, 2, 10, 4, 40, 92]
        for n, count in enumerate(expected, start=1):
            with self.subTest(n=n):
                self.assertEqual(n_queens_count(n), count)

    def test_permutations_match_itertools(self):
        items = [1, 2, 3, 4]
        self.assertEqual(
            permutations(items),
            [list(p) for p in itertools.permutations(items)],
        )

    def test_subsets_match_combinations_and_count(self):
        items = [1, 2, 3, 4, 5]
        for n in range(len(items) + 1):
            with self.subTest(size=n):
                expected = [list(c) for c in itertools.combinations(items, n)]
                got = [s for s in subsets(items) if len(s) == n]
                self.assertEqual(got, expected)
        self.assertEqual(len(subsets(items)), 2 ** len(items))

    def test_fewest_coins_reference(self):
        self.assertEqual(fewest_coins_bruteforce([1, 3, 4], 6), 2)
        self.assertIsNone(fewest_coins_bruteforce([2], 3))


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_greedy.py`:

```python
import random
import unittest

from dsa.backtracking import subsets
from dsa.greedy import coin_change_greedy, max_non_overlapping


def is_valid_schedule(meetings):
    ordered = sorted(meetings)
    return all(ordered[i][1] <= ordered[i + 1][0] for i in range(len(ordered) - 1))


class IntervalTests(unittest.TestCase):
    def test_example_schedule(self):
        meetings = [(9, 10), (9, 11), (10, 12), (11, 13), (12, 13), (13, 14)]
        self.assertEqual(max_non_overlapping(meetings), [(9, 10), (10, 12), (12, 13), (13, 14)])

    def test_touching_meetings_are_allowed(self):
        self.assertEqual(max_non_overlapping([(1, 2), (2, 3)]), [(1, 2), (2, 3)])

    def test_matches_brute_force_on_random_meetings(self):
        rng = random.Random(51)
        for _ in range(200):
            meetings = []
            for _ in range(rng.randint(0, 8)):
                start = rng.randint(0, 19)
                end = rng.randint(start + 1, 20)
                meetings.append((start, end))
            best = max(
                len(group)
                for group in subsets(meetings)
                if is_valid_schedule(group)
            )
            with self.subTest(meetings=meetings):
                chosen = max_non_overlapping(meetings)
                self.assertTrue(is_valid_schedule(chosen))
                self.assertEqual(len(chosen), best)


class CoinTests(unittest.TestCase):
    def test_greedy_fails_on_1_3_4(self):
        chosen = coin_change_greedy([1, 3, 4], 6)
        self.assertEqual(sum(chosen), 6)
        self.assertEqual(len(chosen), 3)

    def test_greedy_is_optimal_for_us_coins_up_to_12(self):
        from dsa.backtracking import fewest_coins_bruteforce

        for amount in range(1, 13):
            with self.subTest(amount=amount):
                greedy = coin_change_greedy([1, 5, 10, 25], amount)
                self.assertEqual(len(greedy), fewest_coins_bruteforce([1, 5, 10, 25], amount))

    def test_impossible_amount_raises(self):
        with self.assertRaises(ValueError):
            coin_change_greedy([2], 3)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 50 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python meetings_demo.py` and confirm the four meetings are the ones listed above. Work out by hand which meeting the greedy rule takes first, and why (12, 13) is taken after (10, 12).
3. Add a `count_schedules(meetings)` function to `dsa/greedy.py` that returns how many subsets of meetings are valid, using brute force over `subsets`. Test it on the example schedule, where the answer is the number of valid groups, and state in a test why it is slower than the greedy rule.
4. Write `docs/greedy-notes.md` with the prune counts from exercise 2, one paragraph on the greedy-choice property for interval scheduling, and one paragraph naming the coin counterexample and what it shows.
5. Add a second greedy rule, earliest start, and write a test that shows it fails on a specific input. Put the failing input in the test and explain it in the notes.
6. Commit with the message "Add backtracking, greedy algorithms and meeting picker", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python meetings_demo.py` prints four meetings
- The random brute-force comparison passes for 200 cases
- `docs/greedy-notes.md` names the counterexample with the coins and the amount
- The latest commit on `main` has a green check

Evidence to commit: `dsa/backtracking.py`, `dsa/greedy.py`, `meetings_demo.py`, both test files, `docs/greedy-notes.md` and the Actions run. The brute-force comparison is the evidence that the greedy rule is right for these inputs, and the failing earliest-start case is the evidence that you tested a rule instead of trusting it.

This project is light on new code and heavy on checking. The habit to take away is that a greedy rule is a claim, and a brute-force check or a counterexample is how you test it.

---

## Check your understanding

1. A backtracking search over all subsets of 20 items visits about how many nodes in the full tree, and what does pruning do to that number?
2. Why does the N-Queens prune check use `row minus col` and `row plus col` as keys, and not the square's coordinates?
3. The earliest-finish greedy rule keeps the first meeting that ends soonest. Explain in one sentence why this rule cannot be beaten by a schedule that starts with a later-ending meeting, using an exchange argument.
4. A teammate proposes choosing the shortest meeting first. Give a small input where that rule chooses too few meetings.
5. Why does the greedy coin rule work for 1, 5, 10 and 25 cents but fail for 1, 3 and 4?
6. A greedy algorithm returns a valid answer for every input. Does that mean it is optimal? Explain using the coin example.

### Answers

1. The full tree for subsets has one node for each subset, which is 2 to the 20th, about one million, plus the nodes for partial subsets along the way. Pruning cuts branches whose partial choice already breaks a rule, so the count depends on the rule. Without a rule that prunes, the search visits all of them.
2. Two queens on the same diagonal have the same row minus column, or the same row plus column, so those two values act as diagonal labels. Storing them in sets makes each attack check a constant-time lookup, which is faster than scanning the placed queens.
3. Take any optimal schedule and let its first meeting be X. The earliest-finishing meeting E in the whole list ends no later than X does. Swap X for E. Every other meeting in the schedule starts after X ends, so it also starts after E ends, and the swap is still valid with the same count. An optimal schedule that starts with E therefore exists, which is the greedy choice.
4. Take the meetings (0, 4), (3, 6) and (5, 9). The shortest is (3, 6), with length 3, and it overlaps both of the others. Choosing it gives one meeting. The best choice is (0, 4) and (5, 9), which gives two, because those two do not overlap. A short meeting in the middle blocks two longer ones that do not block each other.
5. For 1, 5, 10 and 25, every coin is a multiple of the coin before it, and that structure makes the largest-first choice safe, which is a known property of canonical coin systems. For 1, 3 and 4, the coin 3 can be combined to make 6 with two coins, but the largest-first rule takes 4 first and needs three coins.
6. No. A valid answer only means the amount was made exactly. The coin example returns three coins for 6, which is valid, but two coins is better, so the greedy answer is not optimal. Optimality needs a proof for the problem, or a brute-force check that finds a counterexample.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Backtracking](https://en.wikipedia.org/wiki/Backtracking)
- [Wikipedia: Greedy algorithm](https://en.wikipedia.org/wiki/Greedy_algorithm)
- [Wikipedia: Eight queens puzzle](https://en.wikipedia.org/wiki/Eight_queens_puzzle)
- [Wikipedia: Interval scheduling](https://en.wikipedia.org/wiki/Interval_scheduling)
- [Wikipedia: Coin problem](https://en.wikipedia.org/wiki/Coin_problem), which covers when greedy coin change is optimal
- The Python documentation for `itertools`, at docs.python.org/3/library/itertools.html, which provides the reference permutations and combinations used in the tests

*Tutorials*

- GeeksforGeeks, "Backtracking Introduction" and "Greedy Algorithms" tutorials. Search the site for each title.
- W3Schools, the Python section on recursion, which covers the call pattern that backtracking uses. Search W3Schools for "Python recursion".

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on greedy algorithms and on exchange arguments. Search the course site for "greedy".
- Abdul Bari on YouTube, the videos on backtracking and the N-Queens problem. Search YouTube for "Abdul Bari N queens".

*Practice*

- LeetCode, the "Backtracking" and "Greedy" study plans at leetcode.com. Start with "Permutations" and "Jump Game".
- Python Tutor at pythontutor.com. Paste `n_queens_count` with n equal to 4, and step through the first few placements to see the prune checks take effect.

---

## Tomorrow

Day 52 covers dynamic programming. The coin counterexample from today shows that greedy fails for some coin sets, and the brute force search wastes work by solving the same subproblem many times. Dynamic programming solves each subproblem once, stores the answer, and reuses it. You will write memoized and tabulated versions of Fibonacci, the fewest-coins problem and the longest common subsequence, and compare each with today's brute force.

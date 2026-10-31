# Day 53: DSA Assessment

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 4 hours, split into an untimed session of about 2 hours and a timed session of about 1 hour, with time for the write-up
**Level:** intermediate to advanced. You should have finished Days 41 to 52. This day is for checking what you can do without notes, and for finding the topics that need another pass before the next module on web foundations.

**By the end of this day, you will have:**

- A solved set of mixed problems drawn from arrays, linked structures, hashing, trees, heaps, graphs, sorting, searching, greedy and dynamic programming
- For each problem, a stated complexity, a reason the approach works and a comparison with at least one other approach
- An untimed and a timed result for each problem, recorded in a log, with the difference between them
- A small assessment harness that scores candidate solutions automatically, with its own tests
- A gap list that names the two topics to review first, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 52
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)
- A timer or clock for the timed session, and a blank sheet of paper for the untimed one

**Prerequisites:**

- Days 41 to 52, with the repository and tests from each day working
- The habit of checking a solution against a brute force before you trust it

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Mixed problems](#mixed-problems)
  - [Explaining complexity](#explaining-complexity)
  - [Comparing approaches](#comparing-approaches)
  - [Timed and untimed practice](#timed-and-untimed-practice)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an assessment harness with a gap list](#project-an-assessment-harness-with-a-gap-list)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Module 5 covered a lot of ground in thirteen days: complexity, arrays, linked structures, stacks and queues, hashing, trees, heaps, graphs, sorting, searching, backtracking, greedy and dynamic programming. Knowing each topic is one thing. Recognizing which technique a new problem needs, writing it under pressure, and explaining why it is fast enough is another. That second skill is what technical interviews test, and it is what teammates need when they review your code.

An assessment is a checkpoint. Its point is to find the gaps. If you can solve a sliding window problem untimed but freeze under a clock, the gap is speed and pattern recognition, not understanding. If you solve a graph problem but cannot say why BFS gives shortest paths, the gap is explanation. Each kind of gap has a different fix, and this day helps you name which one you have.

The outline asks for mixed problems, explaining complexity, comparing approaches, and timed and untimed practice. The project builds a small harness that runs candidate solutions against automatic checks, so the scoring is objective. The gap list you write at the end tells you where to spend the next week.

---

## Glossary

- **Mixed problem:** a problem that does not announce its topic, so you must recognize which technique applies.
- **Explanation of complexity:** a statement of the time and space cost, with the reason, written in terms of the input size.
- **Approach comparison:** a short argument for why one method beats another on a given input, with the trade-off named.
- **Untimed practice:** solving a problem with no clock, focusing on correctness and clear reasoning.
- **Timed practice:** solving under a fixed limit, which tests recall and pattern recognition under pressure.
- **Harness:** the code that runs a candidate solution against checks and records the result.
- **Gap list:** a short, ranked list of topics or skills where your results were weakest.
- **Reference solution:** a trusted, usually slow solution used to check a faster one.
- **Regression:** a case that used to work and now fails after a change. Tests guard against it.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Mixed problems

Think of a doctor who sees a patient with a set of symptoms and must decide which of several diagnoses fits. The doctor works out the pattern before picking a treatment. The skill is matching the pattern. Mixed problems in an assessment work the same way. The problem gives you a few words, such as "longest", "contiguous" or "shortest path", and you must map them to a technique.

From first principles, every technique in this module answers a family of questions. "Contiguous segment with a property" points to a sliding window. "Pairs that sum to a target" points to two pointers if the input is sorted, and to a hash table if it is not. "Shortest route in edges" points to BFS. "Choices with a constraint" points to backtracking. "Best total with a limit" points to dynamic programming or a greedy rule, and the counterexample habit from Day 51 tells you which. Build a list of these signal words for yourself, and check it against the day you learned each technique.

```python
# Signal words and the technique they usually point to
# "contiguous", "window", "substring" -> sliding window (Day 42)
# "pair that sums to", sorted input   -> two pointers (Day 42)
# "pair that sums to", any input      -> hash table (Day 45)
# "shortest path", unweighted         -> BFS (Day 48)
# "all arrangements", "all subsets"   -> backtracking (Day 51)
# "best with a limit", overlap        -> DP or greedy, then test (Day 51, Day 52)
```

The limitation of signal words is that they mislead. "Subsequence" can mean LCS or a sliding window, and "minimum" can mean a heap, a greedy rule or a DP table. The signal narrows the options, and the constraints of the problem decide among them. Read the input size and any ordering guarantee before you commit.

The common mistake is to jump to the first technique that comes to mind, such as a nested loop, and not check whether a known pattern would remove a level of work. To spot it, ask what the input size is and whether any structure is given, such as sorting or a graph, before you write code.

Mixed problems are how real engineering work arrives. A bug report or a feature request rarely says "use a heap". It describes a symptom, and the engineer has to name the structure.

### Explaining complexity

Imagine a mechanic explaining a repair to a customer. "The engine needs a new part" gives the customer nothing to act on. "The part is inside the engine, and we must take the engine apart to reach it, which is most of the work" is useful, because it says where the cost is. A good complexity explanation names the expensive step and why it happens that many times.

From first principles, state the time complexity in terms of the input size n, name the loop or recursion that dominates, and explain why it runs that many times. For a nested loop, say that the inner loop runs n times for each of the n outer iterations. For a recursive function, say how many calls there are, and how much work each call does. State the space as well: whether the algorithm copies its input, whether it uses a table, and how deep the recursion goes. If a bound is amortized or average, say so, and say what the worst case is.

```python
def has_duplicate(items: list[int]) -> bool:
    seen: set[int] = set()
    for x in items:              # runs n times at most
        if x in seen:            # average O(1) set lookup
            return True
        seen.add(x)              # average O(1) insertion
    return False

# Time: O(n) average, because each of the n items costs O(1) on average.
# Worst case: O(n^2) only if every set operation degraded to a scan, which
# does not happen with Python's built-in hashing of ints.
# Space: O(n), because the set can hold every item.
```

The limitation of explanation is that it can hide assumptions. A claim of O(1) for a lookup assumes a good hash function, and a claim of O(log n) for a search assumes the data is sorted. Always state the assumption next to the bound.

The common mistake is to report only the time, or to report the best case when the question asks for the worst. To spot it, reread the question and check whether it asks for worst, average or amortized cost.

Complexity explanations are the language of design reviews. A reviewer who asks "what is the worst case here?" is asking for exactly this sentence.

### Comparing approaches

Think of two routes on a map. One is shorter but has traffic, and the other is longer but reliable. Neither is always better. The right choice depends on the time of day and what you value. Comparing approaches in an assessment means naming the two routes, their costs under each condition, and the condition that decides between them.

From first principles, a comparison has three parts. Name the alternative, state its time and space, and give an input where the choice matters. For duplicate detection, the sort-then-scan approach costs O(n log n) time and O(1) extra space if you sort in place, while the set approach costs O(n) time and O(n) space. The set wins when time matters more than memory, and the sort wins when memory is tight or the data is already sorted. A good comparison ends with the condition that decides it.

```python
def has_duplicate_sorted(items: list[int]) -> bool:
    ordered = sorted(items)                  # O(n log n) time, O(n) with a copy
    return any(a == b for a, b in zip(ordered, ordered[1:]))

def has_duplicate_set(items: list[int]) -> bool:
    return len(set(items)) < len(items)      # O(n) average time, O(n) space
```

The limitation of comparison is that real costs include constants and memory access patterns. A set of ten items is faster than sorting them, even though sorting is O(n log n) and set building is O(n). Measure when the choice is close, and say so in your answer.

The common mistake is to compare only the big-O bounds and ignore the input size. For small n, the simpler approach often wins. To spot it, ask what n is in the problem and whether the difference matters at that size.

Comparisons decide architecture. Choosing a database index, a cache or a sort step is a comparison of this kind, and the reasons written in a design document are usually the same reasons you give in an interview.

### Timed and untimed practice

Think of two kinds of practice for a musician. In the first, you slow down, play each passage correctly, and understand the phrasing. In the second, you play at tempo with a metronome, to build the speed the piece needs. Both are useful, and each one exposes a different weakness. Untimed practice finds gaps in understanding. Timed practice finds gaps in recall and pattern speed.

From first principles, run untimed practice first. Solve each problem on paper, check it with tests, and write the complexity and one comparison. Record how long it took you, and what you looked up. Then run timed practice on a new set of problems, with a limit that is hard but fair, such as 15 minutes per problem. Record the result as pass or fail, and whether you recognized the technique within the first few minutes. The difference between the two sessions is the signal. A large gap means recall is the problem, and the fix is repetition. A small gap means understanding is the problem, and the fix is to go back to the day's concept section.

```python
import time

start = time.perf_counter()
# ... solve the problem here ...
elapsed = time.perf_counter() - start
print(f"{elapsed:.1f} seconds")   # the timed session should record this number
```

The limitation of timing is that it measures the wrong thing if you use it carelessly. A clock on a problem you have seen before measures recall, not reasoning. Use new problems in timed sessions, and use familiar problems in untimed sessions.

The common mistake is to run timed practice on problems you have already solved, which shows speed on memory and hides the real gap. To spot it, check whether you recognized the problem before the timer started.

Timed practice matches the pressure of an incident, where the fix must be found quickly, and untimed practice matches the slower work of design and review, where the reasoning must be sound.

---

## Hands-on exercises

Work in the repository from Day 52. Keep the harness in `dsa/assessment.py`, the demo in `assess_demo.py`, and your results in `docs/assessment-log.md`.

Exercise 1 is the untimed session. Without notes, solve these three problems on paper: find two numbers in an unsorted list that sum to a target, find the k-th largest value in a list, and decide whether one node can reach another in an undirected graph given as an edge list. For each, write the complexity and one comparison in `docs/assessment-log.md`.

Done when each of the three problems has a written solution, a complexity with its reason, and a comparison with a named alternative.

Exercise 2 is the check. Write the three solutions as Python functions in `tests/test_assessment_solutions.py`, using the function names from the bank in `dsa/assessment.py`. Run them through `run_assessment` and record the result of each.

Done when all three pass, or your log names the failing case and the reason you think it failed.

Exercise 3 is the timed session. Pick three problems you have not solved before, from a source in the Sources section, such as LeetCode. Set a 15-minute limit for each, start a timer, and solve them in Python. Record pass or fail, the time, and whether you recognized the technique in the first five minutes.

Done when your log has three timed rows with those three fields filled in.

Exercise 4 compares the two sessions. Write a short paragraph in the log that compares the untimed and timed results, and names the gap that the difference shows, recall, understanding or explanation.

Done when the paragraph names one gap and one specific topic from Days 41 to 52 that you will review.

Exercise 5 writes the gap list. Rank the topics where your results were weakest, from most to least important, and write one action for each. Put the list at the top of `docs/assessment-log.md`.

Done when the gap list has at least two topics, each with an action you can finish in under an hour.

---

## Project: an assessment harness with a gap list

The project is an assessment harness. It holds a bank of problems, each with an automatic check that compares a candidate solution with a trusted reference on random inputs. It runs candidates with an optional time limit, records a pass or fail for each, and scores the set. The tests use reference solutions to confirm that the harness accepts correct code, rejects wrong code and flags a timeout. You then add problems to the bank, and write the gap list from your own results.

Suggested repo name: `100-days-software-day-53-dsa-assessment`. Copy the Day 52 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-53-dsa-assessment/
├── .github/
│   └── workflows/
│       └── ci.yml
├── assess_demo.py
├── benchmark.py
├── build_demo.py
├── count_words.py
├── diff_demo.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── assessment.py
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
│   ├── assessment-log.md
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
│   ├── test_assessment.py
│   ├── test_assessment_solutions.py
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

Starter code for `dsa/assessment.py`:

```python
from __future__ import annotations

import random
import time
from collections.abc import Callable
from dataclasses import dataclass

from dsa.graph import Graph


@dataclass(frozen=True)
class Problem:
    name: str
    prompt: str
    expected_time: str
    check: Callable[[Callable], bool]


@dataclass(frozen=True)
class Result:
    name: str
    passed: bool
    seconds: float
    within_limit: bool


def check_two_sum(solve: Callable) -> bool:
    """solve(nums, target) returns (i, j) with i < j and nums[i] + nums[j] == target, or None."""
    rng = random.Random(1)
    for _ in range(200):
        nums = [rng.randint(-10, 10) for _ in range(rng.randint(0, 12))]
        target = rng.randint(-20, 20)
        exists = any(
            nums[i] + nums[j] == target
            for i in range(len(nums))
            for j in range(i + 1, len(nums))
        )
        answer = solve(nums, target)
        if answer is None:
            if exists:
                return False
        else:
            i, j = answer
            if not (0 <= i < j < len(nums)) or nums[i] + nums[j] != target:
                return False
    return True


def check_kth_largest(solve: Callable) -> bool:
    """solve(nums, k) returns the k-th largest value, with k counted from 1."""
    rng = random.Random(2)
    for _ in range(200):
        nums = [rng.randint(0, 20) for _ in range(rng.randint(1, 15))]
        k = rng.randint(1, len(nums))
        if solve(nums, k) != sorted(nums, reverse=True)[k - 1]:
            return False
    return True


def check_reachable(solve: Callable) -> bool:
    """solve(edges, start, goal) returns True if goal can be reached from start."""
    rng = random.Random(3)
    for _ in range(200):
        n = rng.randint(1, 10)
        edges = [(rng.randrange(n), rng.randrange(n)) for _ in range(rng.randint(0, 12))]
        start, goal = rng.randrange(n), rng.randrange(n)
        g = Graph()
        g.add_vertex(start)
        g.add_vertex(goal)
        for u, v in edges:
            g.add_edge(u, v)
        if solve(edges, start, goal) != (goal in g.bfs(start)):
            return False
    return True


BANK: list[Problem] = [
    Problem("two_sum", "Find two positions whose values sum to a target.", "O(n) with a hash table", check_two_sum),
    Problem("kth_largest", "Return the k-th largest value.", "O(n log k) with a heap", check_kth_largest),
    Problem("reachable", "Can goal be reached from start in an undirected graph?", "O(V + E) with BFS or DFS", check_reachable),
]


def run_problem(problem: Problem, candidate: Callable, time_limit: float | None = None) -> Result:
    start = time.perf_counter()
    try:
        passed = problem.check(candidate)
    except Exception:
        passed = False
    seconds = time.perf_counter() - start
    within = time_limit is None or seconds <= time_limit
    return Result(problem.name, passed, seconds, within)


def run_assessment(candidates: dict[str, Callable], time_limit: float | None = None) -> list[Result]:
    """A problem with no candidate counts as a failure."""
    results: list[Result] = []
    for problem in BANK:
        candidate = candidates.get(problem.name)
        if candidate is None:
            results.append(Result(problem.name, False, 0.0, True))
        else:
            results.append(run_problem(problem, candidate, time_limit))
    return results


def score(results: list[Result]) -> float:
    """Fraction of problems that passed and finished within the limit."""
    if not results:
        return 0.0
    return sum(1 for r in results if r.passed and r.within_limit) / len(results)


def summarize(results: list[Result]) -> list[str]:
    lines = []
    for r in results:
        status = "PASS" if r.passed and r.within_limit else "FAIL"
        lines.append(f"{r.name:<12} {status}  {r.seconds:.3f} s")
    lines.append(f"score: {score(results):.2f}")
    return lines
```

Starter code for `assess_demo.py`:

```python
from dsa.assessment import run_assessment, summarize


def bad_two_sum(nums, target):
    return None   # deliberately wrong: it never finds a pair


if __name__ == "__main__":
    candidates = {"two_sum": bad_two_sum}
    for line in summarize(run_assessment(candidates, time_limit=5.0)):
        print(line)
```

Running `python assess_demo.py` prints FAIL for `two_sum`, which is wrong, and FAIL for the two problems with no candidate, then a score of 0.00.

Starter code for `tests/test_assessment.py`:

```python
import unittest

from dsa.assessment import BANK, run_assessment, run_problem, score

from collections import deque


def two_sum_ref(nums, target):
    seen: dict[int, int] = {}
    for j, x in enumerate(nums):
        if target - x in seen:
            return seen[target - x], j
        seen.setdefault(x, j)
    return None


def kth_largest_ref(nums, k):
    return sorted(nums, reverse=True)[k - 1]


def reachable_ref(edges, start, goal):
    adj: dict[int, list[int]] = {}
    for u, v in edges:
        adj.setdefault(u, []).append(v)
        adj.setdefault(v, []).append(u)
    seen = {start}
    queue = deque([start])
    while queue:
        u = queue.popleft()
        for v in adj.get(u, []):
            if v not in seen:
                seen.add(v)
                queue.append(v)
    return goal in seen


class HarnessTests(unittest.TestCase):
    def test_reference_solutions_pass(self):
        candidates = {"two_sum": two_sum_ref, "kth_largest": kth_largest_ref, "reachable": reachable_ref}
        results = run_assessment(candidates)
        self.assertTrue(all(r.passed for r in results))
        self.assertEqual(score(results), 1.0)

    def test_wrong_solution_fails(self):
        results = run_assessment({"two_sum": lambda nums, target: None})
        two_sum = next(r for r in results if r.name == "two_sum")
        self.assertFalse(two_sum.passed)

    def test_exception_counts_as_failure(self):
        def crashes(nums, k):
            raise RuntimeError("boom")

        problem = next(p for p in BANK if p.name == "kth_largest")
        self.assertFalse(run_problem(problem, crashes).passed)

    def test_missing_candidate_scores_zero(self):
        self.assertEqual(score(run_assessment({})), 0.0)

    def test_time_limit_is_enforced(self):
        problem = next(p for p in BANK if p.name == "two_sum")
        result = run_problem(problem, two_sum_ref, time_limit=0.0)
        self.assertTrue(result.passed)
        self.assertFalse(result.within_limit)


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_assessment_solutions.py`:

```python
import unittest

from dsa.assessment import BANK, run_assessment, score


class SolutionTests(unittest.TestCase):
    def test_your_solutions_pass_the_bank(self):
        # Replace these placeholders with your solutions from Exercise 2.
        # A missing candidate fails the problem, so this test shows your progress.
        candidates: dict = {}
        results = run_assessment(candidates, time_limit=5.0)
        self.assertEqual(len(results), len(BANK))
        self.assertGreaterEqual(score(results), 0.0)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 52 repository, add the files above, and run `python -m unittest`. All tests should pass, including the harness tests that use the reference solutions.
2. Run `python assess_demo.py` and read the FAIL lines. Explain in one sentence why each one fails.
3. Write your solutions for the three problems from the untimed session in `tests/test_assessment_solutions.py`, in a dictionary keyed by problem name. Run the test and record the score.
4. Add two more problems to `BANK`, each with a check function that compares with a trusted slow solution on random inputs. Pick two from the timed session, such as "longest substring without repeats" and "shortest path in edges". Add a harness test for each.
5. Fill in `docs/assessment-log.md` with the untimed and timed results, the complexity and comparison for each problem, and the gap list from exercise 5.
6. Commit with the message "Add DSA assessment harness and gap list", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python assess_demo.py` shows FAIL for the wrong candidate and for the missing ones
- Your solutions for the three starter problems are in the solution test, and the score is recorded in the log
- `docs/assessment-log.md` has the untimed and timed rows, the comparisons and a gap list with at least two topics
- The latest commit on `main` has a green check

Evidence to commit: `dsa/assessment.py`, `assess_demo.py`, the test files, `docs/assessment-log.md` and the Actions run. The log is the main artifact, because it is your own record of what you can do, and the harness is the proof that the scoring is objective.

This project is light on new algorithms and heavy on honest measurement. The most useful output is the gap list, because it tells you what to study next, and it is only useful if you wrote it from your results as they actually were.

---

## Check your understanding

1. A problem asks for "the longest run of positive numbers in a list". Name the technique you would try first, and the signal word that points to it.
2. Write the time and space complexity of the set-based duplicate check, and explain why the time is O(n) on average but not in every case.
3. You have two solutions: one O(n log n) with sorting and one O(n) with a hash table. For inputs of ten items, which would you choose, and for ten million? Name the condition that decides.
4. A teammate says their solution is O(n) because it has one loop. Explain why that reasoning is incomplete, using a loop that calls `list.pop(0)` inside it.
5. Your timed session scores lower than your untimed session. Name two different causes that would give that result, and say which one the untimed session could not have revealed.
6. Why should the harness use a trusted slow reference to check a candidate, instead of a fixed list of expected answers?

### Answers

1. The technique is a sliding window or a single pass with a running count, because "longest run" describes a contiguous segment with a property. The signal word is "run", which means contiguous, and the single pass keeps the count of the current run and the best seen so far, which is O(n).
2. The set-based check takes O(n) time on average, because each of the n insertions and lookups costs O(1) on average, and O(n) extra space to hold the set. The worst case is worse if many keys collide in the hash table, which can make each lookup scan a long chain. Python's hashing of ints makes that unlikely, but the claim must name the average.
3. For ten items, the constant factors dominate, so either is fine and the simpler code wins. For ten million items, the O(n) hash table wins on time, but it uses O(n) extra memory. If memory is tight and the data can be sorted in place, the sort-based approach may be the right choice. The deciding condition is the memory limit as much as the time.
4. The loop calls `list.pop(0)`, which is O(n) on a list, so the total work is O(n squared), not O(n). The loop count is only the number of iterations. Each operation inside the loop must also be counted.
5. One cause is recall: the timed session tests whether you remember the technique under pressure, and the untimed session let you look at the material. Another cause is speed of implementation, such as slow typing or debugging. The untimed session could not reveal either, because it had no clock and no pressure.
6. A fixed list of expected answers covers only the cases you thought of, so a wrong solution can pass by luck on those cases. A trusted reference on random inputs covers many more cases, and it finds the boundary cases that a fixed list misses. The reference must be simple enough to trust, which is why the brute force is slow but clear.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Big O notation](https://en.wikipedia.org/wiki/Big_O_notation), to check the definitions used in your explanations
- The Python Wiki page "TimeComplexity", on wiki.python.org, to check the cost of each built-in operation you use
- Thomas H. Cormen and others, *Introduction to Algorithms*, the table of contents, to find the chapter for any topic that the log shows as weak
- Steven S. Skiena, *The Algorithm Design Manual*, the chapter on problem solving, which gives a method for recognizing problem types

*Tutorials*

- GeeksforGeeks, "Top 100 Data Structure and Algorithms Interview Questions" and the topic tutorials for each day of this module. Search the site for each title.
- W3Schools, the Python tutorials on lists, dictionaries and recursion, to refresh the syntax you will use in the harness.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the course overview and the final lectures, which tie the topics of this module together. Search the course site for "course overview".
- NeetCode on YouTube, the roadmap videos for each topic, which present problems by pattern. Search YouTube for "NeetCode roadmap".

*Practice*

- LeetCode, the "Top Interview Questions" list at leetcode.com. Pick three problems you have not seen, one from arrays, one from graphs and one from dynamic programming, and use them for the timed session.
- VisuAlgo at visualgo.net, to replay any algorithm from this module when the log shows a gap in understanding.

---

## Tomorrow

Day 54 begins the Web Foundations module, and it changes the subject from algorithms to the systems you use every day. You will learn how the web works: the client and server model, URLs, DNS, TCP and IP at the level of intuition, and the request and response cycle of HTTP and HTTPS. The data structures from this module return inside the browser, which uses trees for the page and maps for headers and cookies, so the habits of complexity and of explaining a design carry over directly.

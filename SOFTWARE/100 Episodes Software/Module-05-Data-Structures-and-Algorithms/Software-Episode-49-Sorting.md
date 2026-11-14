# Day 49: Sorting

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** intermediate. You should have finished Day 48, be comfortable with recursion and with the heap from Day 47, and have used `sorted` in Python. This day is for anyone who has called `sort` on a list of records and not known whether the result would keep the old order for equal items.

**By the end of this day, you will have:**

- Working implementations of insertion sort, merge sort, quicksort and heap sort, each checked against Python's `sorted`
- A clear view of stability, with a test that shows which algorithms keep equal items in their original order
- The comparison-sort lower bound, stated in plain terms, and the reason O(n log n) is the best a comparison sort can do in the worst case
- A leaderboard that sorts by score and then by name, using stability to combine the two keys
- The leaderboard project committed with tests and a CI run

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 48
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 41: Complexity Analysis (Big-O, and the average and worst case distinction)
- Day 46: Trees and BSTs (recursion on halves of a structure)
- Day 47: Heaps and Priority Queues (`MinHeap` from the project)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Comparison sorting](#comparison-sorting)
  - [Merge sort](#merge-sort)
  - [Quicksort](#quicksort)
  - [Heap sort](#heap-sort)
  - [Stability](#stability)
  - [Complexity](#complexity)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a leaderboard with a stable multi-key sort](#project-a-leaderboard-with-a-stable-multi-key-sort)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Sorting is the most common operation in software that you rarely write yourself. It runs when a search results page orders items by relevance, when a report lists customers by revenue, and when a database answers an `ORDER BY` clause. Python's built-in `sorted` and `list.sort` are fast and reliable, so you will almost always use them. This day exists for the cases where you need to know what they do: why the output is stable, why a sort can be slow on some inputs, and why no comparison sort can beat a certain speed.

Sorting algorithms also teach a set of techniques that reappear across this module. Merge sort is divide and conquer, the same method as the recursive searches in Day 50. Heap sort uses the heap from Day 47. Quicksort uses partitioning, which is a two-pointer scan from Day 42. The question to ask of each algorithm is the same one you asked on Day 41: how much work does it do as n grows, and what does it do in the worst case?

The outline asks for comparison sorting, merge, quick and heap concepts, stability and complexity. The project uses a stable sort in a practical job: ranking players by score, with ties broken by name. That task is a common interview question and a common product requirement.

---

## Glossary

- **Sorting:** rearranging items so that each one is no greater than the one after it, according to a key.
- **Comparison sort:** a sort that learns the order only by comparing two items at a time.
- **In-place sort:** a sort that uses only a small, fixed amount of extra memory beyond the input.
- **Stable sort:** a sort that keeps equal items in their original relative order.
- **Divide and conquer:** splitting a problem into smaller parts, solving each part, and combining the results.
- **Partition:** rearranging a list so that elements less than a pivot come before it and elements greater come after.
- **Pivot:** the element chosen to split the list during partitioning.
- **Key function:** a function that gives the value to compare for each item, such as `lambda p: p[1]`.
- **Lower bound:** a limit on how much work any algorithm in a class must do in the worst case.
- **Insertion sort:** a simple sort that builds a sorted prefix one element at a time.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Comparison sorting

Think of sorting a hand of playing cards. You pick up one card at a time and slide it into place among the cards you already hold. Each slide compares the new card with the ones in your hand. That is insertion sort, and it is the simplest comparison sort.

From first principles, a comparison sort is an algorithm that learns the order only by asking "is a less than b" about pairs of items. Insertion sort does that by moving each element left past larger elements. For a list of n items, the worst case, which is a reversed list, needs about n squared over 2 comparisons. The best case, which is an already sorted list, needs n minus 1 comparisons. Insertion sort is fast on small or nearly sorted inputs, and that is why real libraries use it for short runs inside larger algorithms.

```python
def insertion_sort(items: list) -> list:
    result = list(items)
    for i in range(1, len(result)):
        current = result[i]
        j = i - 1
        while j >= 0 and result[j] > current:
            result[j + 1] = result[j]
            j -= 1
        result[j + 1] = current
    return result

print(insertion_sort([5, 2, 4, 1]))   # [1, 2, 4, 5]
```

The limitation is that insertion sort is O(n squared) in the worst case, so it becomes slow for large inputs. There is also a deeper limit. Any comparison sort must make at least about n log2 n comparisons in the worst case, because there are n factorial possible orders and each comparison can only split that set in two. Algorithms that beat that bound must use extra information, such as the values being small integers, which counting sort uses.

The common mistake is to think a sort that works on small examples is good enough. A test with five items cannot show the difference between n squared and n log n. To spot the problem, time the sort at two sizes that differ by a factor of ten, and see how the time changes.

Insertion sort appears in the standard libraries of several languages as the base case for small arrays, because its low overhead wins on short inputs.

### Merge sort

Imagine two stacks of sorted papers on a desk. To combine them, you look at the top of each stack, take the smaller page, and repeat until both stacks are empty. Merging two sorted lists takes time proportional to their total length. Merge sort is built from that step: split the list in half, sort each half, then merge the two sorted halves.

From first principles, the recursion has log2 n levels, because the list halves each time. At each level, the merges together touch every element once, so the work per level is O(n). The total is O(n log n) in every case: best, average and worst. The price is memory. The merge step needs a new output list of size n, so merge sort uses O(n) extra space.

```python
def merge_sort(items: list, key=lambda x: x) -> list:
    if len(items) <= 1:
        return list(items)
    mid = len(items) // 2
    left = merge_sort(items[:mid], key)
    right = merge_sort(items[mid:], key)
    return _merge(left, right, key)

def _merge(left: list, right: list, key) -> list:
    out = []
    i = j = 0
    while i < len(left) and j < len(right):
        if key(right[j]) < key(left[i]):
            out.append(right[j])
            j += 1
        else:
            out.append(left[i])
            i += 1
    out.extend(left[i:])
    out.extend(right[j:])
    return out

print(merge_sort([5, 2, 4, 1, 3]))   # [1, 2, 3, 4, 5]
```

The limitation is the O(n) extra memory and the copying of slices, which makes merge sort slower in practice than quicksort on arrays that fit in cache. It is the choice when stability matters, when the data is a linked list, or when you need guaranteed O(n log n) time.

The common mistake is to take from the right side on ties, which breaks stability. The code above takes from the left on ties, because `right[j]` must be strictly smaller to move first. To spot the problem, sort records with equal keys and check the order of the tags.

Merge sort is used in Java's object sorting, in Python's own `sorted` through Timsort, which is a merge sort variant, and in external sorting of files too large for memory.

### Quicksort

Picture a group of children lined up by height. You choose one child at random, ask everyone shorter to stand on the left and everyone taller to stand on the right, then sort each side the same way. That is quicksort. The partition step does most of the work, and the recursion handles the rest.

From first principles, partitioning moves the pivot into its final position. If the pivot splits the list evenly, the recursion depth is log2 n and the total work is O(n log n). If the pivot is always the smallest or largest item, one side is empty every time, the depth becomes n, and the work is O(n squared). Choosing a random pivot makes the bad case extremely unlikely for any input, which is why it is the common choice. The code below uses a random pivot and recurses on the smaller side first, which keeps the recursion depth to O(log n) even in bad cases.

```python
import random

def quick_sort(items: list) -> list:
    result = list(items)
    _quick(result, 0, len(result) - 1)
    return result

def _quick(a: list, lo: int, hi: int) -> None:
    while lo < hi:
        pick = random.randint(lo, hi)
        a[pick], a[hi] = a[hi], a[pick]
        pivot = a[hi]
        i = lo
        for j in range(lo, hi):
            if a[j] < pivot:
                a[i], a[j] = a[j], a[i]
                i += 1
        a[i], a[hi] = a[hi], a[i]
        if i - lo < hi - i:
            _quick(a, lo, i - 1)
            lo = i + 1
        else:
            _quick(a, i + 1, hi)
            hi = i - 1

print(quick_sort([3, 6, 1, 8, 2, 9, 2]))   # [1, 2, 2, 3, 6, 8, 9]
```

The limitation is that quicksort is not stable, because the partition swaps items across the pivot. It also has the worst-case risk above, and it is recursive. The standard defenses are the random pivot and the smaller-side recursion used here.

The common mistake is to pick a fixed pivot, such as the first element, on data that arrives already sorted. That is the worst case, and the sort becomes quadratic on exactly the input that is common in practice, such as a log that is already in time order. To spot it, time quicksort on a sorted list of 5000 items with a fixed pivot.

Quicksort and its variants power the default sort in many C libraries and the introsort used by C++ standard libraries, which switches to heap sort when recursion gets too deep.

### Heap sort

Think of a line of people who must be sorted by height. Instead of arranging the whole line, you repeatedly pull out the shortest person who remains and place them next. A heap makes that pull cheap, because the minimum sits at the root. Heap sort is that idea, with the heap from Day 47.

From first principles, building the heap from n items costs O(n) with the bottom-up method. Then n pops each cost O(log n), so the total is O(n log n) in every case. Heap sort uses no extra memory beyond the heap if you implement it in place, which is its main advantage over merge sort. The version in this day builds a separate heap, so it uses O(n) extra space, but the time is the same.

```python
from dsa.heap import MinHeap

def heap_sort(items: list) -> list:
    heap = MinHeap.from_list(items)
    return [heap.pop() for _ in range(len(heap))]

print(heap_sort([4, 1, 3, 2]))   # [1, 2, 3, 4]
```

The limitation is that heap sort is not stable, because the heap moves equal items around as it sifts. It also jumps around the array in memory, which is slower on modern hardware than the sequential access of merge sort or quicksort.

The common mistake is to call `MinHeap.from_list` and then pop without checking the length first, which is safe here because `range(len(heap))` is evaluated once. The danger is a heap that is mutated during the loop, so keep the work inside a single list comprehension.

Heap sort is used in embedded systems and in situations where worst-case memory and time must both be bounded, because it is O(n log n) in every case and needs no recursion.

### Stability

Imagine a list of runners sorted by finish time. Two runners finished at the same time. If the sort is stable, they still appear in the order they registered, which is the order the race committee cares about. If the sort is not stable, their order could change, even though the times are equal. Stability is the rule that keeps equal items in the order they came in.

From first principles, stability matters whenever you sort in more than one pass. If you sort by name first, then sort by score with a stable sort, names stay in order within each score. That is the basis of multi-key sorting. Merge sort and insertion sort are stable in the versions above. Quicksort and heap sort in this day are not. Python's `sorted` and `list.sort` are stable, which is documented and reliable.

```python
records = [("ann", 2), ("ben", 1), ("cy", 2), ("dee", 1)]
by_score = merge_sort(records, key=lambda r: r[1])
print(by_score)   # [('ben', 1), ('dee', 1), ('ann', 2), ('cy', 2)]
```

The limitation is that stability is a property of the algorithm. If you use an unstable sort on a list where equal items are indistinguishable, the result is the same, so stability does not matter. It matters only when equal keys carry different data.

The common mistake is to assume that every sort in a library is stable, or that a sort on a single key is enough when you need two. Check the documentation of the function you use, and test the case with equal keys. The leaderboard project depends on this.

Stability matters in database engines that sort by multiple columns, in spreadsheet programs that sort by column in steps, and in build tools that must preserve the order of files with equal timestamps.

### Complexity

Think of a competition with a fixed format. Each match eliminates some options, and you need enough matches to find the answer. A comparison sort is the same. Each comparison gives one bit of information about the order. There are n factorial orders, and log2 of n factorial is about n log2 n, so at least that many comparisons are needed in the worst case.

From first principles, the table below lists the complexity of the sorts in this day. Each entry gives the best, average and worst time, and the extra space. Choose by the constraints of the job, not by the table alone.

| Sort | Best | Average | Worst | Extra space | Stable |
|------|------|---------|-------|-------------|--------|
| Insertion | O(n) | O(n squared) | O(n squared) | O(1) | yes |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | yes |
| Quicksort with random pivot | O(n log n) | O(n log n) | O(n squared), very unlikely | O(log n) | no |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) in place | no |

The limitation of this table is that it ignores constants and memory access patterns. Quicksort is often faster than merge sort in practice, even though merge sort has a better worst case, because it moves less data and uses cache well. Measure on your own data before you choose for performance.

The common mistake is to read the table and conclude that heap sort is always the best choice because its worst case is good and it uses little memory. Its unstable order and poor cache behavior make it slower in many real programs. To spot the difference, benchmark it against merge sort on a list of a million integers.

Sorting complexity decides how a database plans a query, how a search engine ranks results at scale and how a streaming system keeps a window of events in order.

---

## Hands-on exercises

Work in the repository from Day 48. Keep the sorting functions in `dsa/sorting.py` and the leaderboard in `dsa/leaderboard.py`.

Exercise 1 counts comparisons. Add a counter to a copy of insertion sort. Run it on a sorted list, a reversed list and a random list of 100 items. Record the counts in `docs/sorting-notes.md`.

Done when your notes show about 99 comparisons for the sorted list, about 4950 for the reversed list, and a value between them for the random list.

Exercise 2 traces merge. On paper, sort the list `[6, 3, 5, 1]` with merge sort, writing each split and each merge. Compare with `merge_sort`.

Done when your trace produces the sorted list and each merge step is a comparison that you can point to.

Exercise 3 tests stability. Create 20 records with keys from 0 to 2 and a tag that records the original position. Sort them with `merge_sort` and with `quick_sort` by key. Check that the merge result keeps tags in order within each key, and report whether the quicksort result does.

Done when the merge sort test passes and your notes state what happened with quicksort on this input.

Exercise 4 shows the fixed-pivot problem. Write a version of quicksort that always uses the last element as the pivot, without the random swap. Time it on a sorted list of 3000 items and a random list of 3000 items. Record the times and explain the difference.

Done when your notes show the sorted list is much slower, and you can state the recursion depth of the fixed-pivot version on sorted input.

Exercise 5 compares with the library. For a list of 100000 random integers, time `merge_sort`, `quick_sort`, `heap_sort` and `sorted`. Record the times, note which is fastest, and explain why `sorted` is usually the fastest in Python.

Done when your notes show all four times and a sentence on the reason.

---

## Project: a leaderboard with a stable multi-key sort

The project is a leaderboard. Players have a name and a score, and the board must list them by score from highest to lowest, with ties broken by name in alphabetical order. The trick is to use stability. Sort by name first, then sort by score with a stable sort, and the names stay in alphabetical order within each score. The tests check the result against Python's `sorted` with a two-part key, so you can see that both methods agree.

Suggested repo name: `100-days-software-day-49-sorting`. Copy the Day 48 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-49-sorting/
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
│   ├── scheduler.py
│   ├── sorting.py
│   ├── stacks_queues.py
│   └── word_count.py
├── docs/
│   ├── graph-notes.md
│   └── sorting-notes.md
├── height_demo.py
├── leaderboard_demo.py
├── print_queue.py
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
│   ├── test_scheduler.py
│   ├── test_sorting.py
│   ├── test_stacks_queues.py
│   └── test_word_count.py
└── README.md
```

Starter code for `dsa/sorting.py`:

```python
import random
from collections.abc import Callable
from typing import Any

from dsa.heap import MinHeap


def _identity(x: Any) -> Any:
    return x


def insertion_sort(items: list) -> list:
    result = list(items)
    for i in range(1, len(result)):
        current = result[i]
        j = i - 1
        while j >= 0 and result[j] > current:
            result[j + 1] = result[j]
            j -= 1
        result[j + 1] = current
    return result


def merge_sort(items: list, key: Callable[[Any], Any] = _identity) -> list:
    """Stable O(n log n) sort. Equal keys keep their original order."""
    if len(items) <= 1:
        return list(items)
    mid = len(items) // 2
    left = merge_sort(items[:mid], key)
    right = merge_sort(items[mid:], key)
    return _merge(left, right, key)


def _merge(left: list, right: list, key: Callable[[Any], Any]) -> list:
    out: list = []
    i = j = 0
    while i < len(left) and j < len(right):
        if key(right[j]) < key(left[i]):
            out.append(right[j])
            j += 1
        else:
            out.append(left[i])
            i += 1
    out.extend(left[i:])
    out.extend(right[j:])
    return out


def quick_sort(items: list) -> list:
    """O(n log n) expected. Not stable."""
    result = list(items)
    _quick(result, 0, len(result) - 1)
    return result


def _quick(a: list, lo: int, hi: int) -> None:
    while lo < hi:
        pick = random.randint(lo, hi)
        a[pick], a[hi] = a[hi], a[pick]
        pivot = a[hi]
        i = lo
        for j in range(lo, hi):
            if a[j] < pivot:
                a[i], a[j] = a[j], a[i]
                i += 1
        a[i], a[hi] = a[hi], a[i]
        if i - lo < hi - i:
            _quick(a, lo, i - 1)
            lo = i + 1
        else:
            _quick(a, i + 1, hi)
            hi = i - 1


def heap_sort(items: list) -> list:
    """O(n log n) in every case. Not stable."""
    heap = MinHeap.from_list(items)
    return [heap.pop() for _ in range(len(heap))]


def is_sorted(items: list, key: Callable[[Any], Any] = _identity) -> bool:
    return all(key(items[i]) <= key(items[i + 1]) for i in range(len(items) - 1))
```

Starter code for `dsa/leaderboard.py`:

```python
from dsa.sorting import merge_sort


def rank(players: list[tuple[str, int]]) -> list[tuple[str, int]]:
    """Sort by score, highest first, then by name, A to Z.
    Sorting by name first, then by score with a stable sort, gives both keys."""
    by_name = merge_sort(players, key=lambda p: p[0])
    return merge_sort(by_name, key=lambda p: -p[1])
```

Starter code for `leaderboard_demo.py`:

```python
from dsa.leaderboard import rank

if __name__ == "__main__":
    players = [("maya", 80), ("ali", 95), ("zoe", 80), ("ben", 95), ("kim", 60)]
    for place, (name, score) in enumerate(rank(players), start=1):
        print(f"{place}. {name:<6} {score}")
```

Running `python leaderboard_demo.py` prints ali and ben at 95, then maya and zoe at 80, then kim at 60.

Starter code for `tests/test_sorting.py`:

```python
import random
import unittest

from dsa.sorting import heap_sort, insertion_sort, is_sorted, merge_sort, quick_sort


class SortCorrectnessTests(unittest.TestCase):
    def test_all_algorithms_match_sorted_on_random_lists(self):
        rng = random.Random(14)
        algorithms = [insertion_sort, merge_sort, quick_sort, heap_sort]
        for _ in range(60):
            items = [rng.randint(-20, 20) for _ in range(rng.randint(0, 40))]
            expected = sorted(items)
            for algorithm in algorithms:
                with self.subTest(algorithm=algorithm.__name__, items=items):
                    self.assertEqual(algorithm(items), expected)

    def test_empty_and_single(self):
        for algorithm in [insertion_sort, merge_sort, quick_sort, heap_sort]:
            self.assertEqual(algorithm([]), [])
            self.assertEqual(algorithm([7]), [7])

    def test_input_is_not_modified(self):
        items = [3, 1, 2]
        quick_sort(items)
        merge_sort(items)
        self.assertEqual(items, [3, 1, 2])


class StabilityTests(unittest.TestCase):
    def test_merge_sort_keeps_equal_keys_in_order(self):
        records = [("ann", 2), ("ben", 1), ("cy", 2), ("dee", 1), ("eve", 2)]
        result = merge_sort(records, key=lambda r: r[1])
        self.assertEqual(
            result,
            [("ben", 1), ("dee", 1), ("ann", 2), ("cy", 2), ("eve", 2)],
        )

    def test_quick_sort_result_is_a_permutation(self):
        records = [("a", 1), ("b", 1), ("c", 0)]
        result = quick_sort([score for _, score in records])
        self.assertEqual(sorted(result), sorted(score for _, score in records))
        self.assertTrue(is_sorted(result))


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_leaderboard.py`:

```python
import random
import unittest

from dsa.leaderboard import rank


class LeaderboardTests(unittest.TestCase):
    def test_score_then_name(self):
        players = [("maya", 80), ("ali", 95), ("zoe", 80), ("ben", 95), ("kim", 60)]
        self.assertEqual(
            rank(players),
            [("ali", 95), ("ben", 95), ("maya", 80), ("zoe", 80), ("kim", 60)],
        )

    def test_matches_two_key_sorted_on_random_data(self):
        rng = random.Random(3)
        names = [f"p{i:02d}" for i in range(30)]
        for _ in range(20):
            players = [(rng.choice(names), rng.randint(0, 5)) for _ in range(25)]
            expected = sorted(players, key=lambda p: (-p[1], p[0]))
            with self.subTest(players=players):
                self.assertEqual(rank(players), expected)

    def test_empty_board(self):
        self.assertEqual(rank([]), [])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 48 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python leaderboard_demo.py` and confirm the order against the expected output in the notes above.
3. Add a `top_n(players, n)` function to `dsa/leaderboard.py` that returns the first n places from `rank`, and test it with n larger than the list.
4. Write `docs/sorting-notes.md` with the times from exercise 5, one paragraph on why `rank` needs stability, and one paragraph on the quicksort fixed-pivot result from exercise 4.
5. Add a timing check to `benchmark.py` that compares `merge_sort` and `sorted` on 50000 integers, printing both times. Do not assert on the times, because they change between machines.
6. Commit with the message "Add sorting algorithms and leaderboard", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python leaderboard_demo.py` lists ali and ben first, and kim last
- The stability test checks that equal scores keep their input order
- `docs/sorting-notes.md` explains the stability requirement in `rank` with one example
- The latest commit on `main` has a green check

Evidence to commit: `dsa/sorting.py`, `dsa/leaderboard.py`, `leaderboard_demo.py`, both test files, `docs/sorting-notes.md` and the Actions run. The two-key comparison with `sorted` and the stability test are the evidence that the board is correct.

This project is light on new code and heavy on one idea: stability lets you build a multi-key sort from single-key sorts. That idea shows up in every table view that lets users sort by more than one column.

---

## Check your understanding

1. An insertion sort takes 20 milliseconds on 1000 items. Estimate its time on 10000 items in the worst case, and explain the estimate.
2. Why does merge sort use O(n) extra space, and why does quicksort use only O(log n) for its recursion stack in the version above?
3. A quicksort implementation picks the first element as the pivot. Describe the input that makes it slow, and say what the random pivot changes.
4. Explain why `rank` in the leaderboard sorts by name first and then by score, and what would go wrong if the order were reversed.
5. A teammate says heap sort is always better than merge sort because it uses less memory. Give one reason the statement is incomplete, using the stability column of the table.
6. Why is `sorted` in Python usually the right choice, and when would you still write your own sort?

### Answers

1. Insertion sort is O(n squared) in the worst case, so going from 1000 to 10000 items multiplies the work by 100. The estimate is about 2 seconds, which is 20 milliseconds times 100, assuming the worst-case input and that the per-operation cost stays the same.
2. Merge sort needs a new list to hold each merge, so the extra space is proportional to n. Quicksort partitions in place and recurses on the smaller side first, so the recursion depth is at most log2 n, and each level uses constant extra space.
3. A first-element pivot on an already sorted input splits the list into an empty side and a side with n minus 1 items each time, so the work is O(n squared). A random pivot makes an even split likely for any input, so the expected work is O(n log n).
4. Sorting by name first gives an alphabetical order. The score sort then keeps that order within each score, because it is stable. If the order were reversed, the second sort by name would undo the score order, and the list would be alphabetical with scores in no order.
5. Heap sort uses less memory than merge sort, but it is not stable. If two players have equal scores and their input order matters, heap sort can change it, so the statement is incomplete. Stability is a requirement the table shows separately from time and space.
6. `sorted` is stable, well tested and fast, and it works on any comparable items with a key function. Write your own sort when you need a property the library does not give, such as a specific memory bound, sorting data stored on disk in chunks, or a teaching exercise like this one.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Sorting algorithm](https://en.wikipedia.org/wiki/Sorting_algorithm), with the comparison table
- [Wikipedia: Merge sort](https://en.wikipedia.org/wiki/Merge_sort)
- [Wikipedia: Quicksort](https://en.wikipedia.org/wiki/Quicksort)
- [Wikipedia: Heapsort](https://en.wikipedia.org/wiki/Heapsort)
- [Wikipedia: Stable sort](https://en.wikipedia.org/wiki/Sorting_algorithm#Stability)
- The Python documentation on sorting, the "Sorting HOW TO" at docs.python.org/3/howto/sorting.html, which covers key functions and stability

*Tutorials*

- GeeksforGeeks, "Sorting Algorithms" and "Stability in Sorting Algorithms". Search the site for each title.
- W3Schools, "Python Sorting" examples under the list tutorial. Search the W3Schools Python section for "Python sort list".

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on sorting, including merge sort, quicksort and the lower bound for comparison sorts. Search the course site for "comparison sort lower bound".
- Abdul Bari on YouTube, the videos on merge sort and quicksort. Search YouTube for "Abdul Bari quick sort".

*Practice*

- VisuAlgo at visualgo.net, the sorting visualization, which runs each algorithm step by step.
- LeetCode, the "Sorting" problems, such as "Sort an Array" and "Sort Colors". Solve one with your own merge sort and compare the timing with `sorted`.

---

## Tomorrow

Day 50 covers searching and recursion. Binary search uses the sorted order from today to find a value in O(log n), and it is the first algorithm in this module that depends on the input being sorted. You will write iterative and recursive binary search, see how the call stack grows with recursion, and use divide and conquer to compute powers in O(log n) multiplications.

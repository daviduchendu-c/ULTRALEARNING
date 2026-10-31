# Day 47: Heaps and Priority Queues

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** intermediate. You should have finished Day 46, understand tree height, and be comfortable with tuples and comparisons. This day is for anyone who has needed "the next most urgent item" from a changing set and used a sort in a loop to get it.

**By the end of this day, you will have:**

- A binary min-heap stored in a Python list, with push, pop, peek and a sift-based invariant you can check
- A bottom-up `heapify` that builds a heap from a list in O(n)
- A top-k function that keeps only k items in memory and runs in O(n log k)
- A priority scheduler whose ties resolve in arrival order, with the reasons spelled out
- A job scheduler project with tests that compare the heap with `heapq`, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 46
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 44: Stacks and Queues (the difference between FIFO and priority order)
- Day 46: Trees and BSTs (tree height and the idea of a complete tree)
- Module 3: Python tuples, comparisons and the `heapq` module in the standard library

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Heap property](#heap-property)
  - [Operations](#operations)
  - [Top-k](#top-k)
  - [Scheduling](#scheduling)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a priority job scheduler](#project-a-priority-job-scheduler)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every system with more work than workers needs to answer one question quickly: what should run next? A web server needs the request that has waited longest, a payment system needs the most overdue invoice, and an operating system needs the thread with the highest priority. A sorted list answers that question, but every new arrival means another sort. A plain queue answers it in order of arrival, which ignores urgency. A priority queue answers the question directly. It gives you the most important item in O(1), and it adds or removes items in O(log n).

The structure that makes this efficient is the heap. A heap is a complete binary tree with one rule: each parent is no larger than its children, so the root holds the minimum. Because the tree is complete, it fits in a flat array with no node objects, which is why heaps are compact and fast. Heaps are the engine behind Python's `heapq`, the scheduler in most operating systems, the Dijkstra shortest-path algorithm you will meet on Day 48, and the top-k queries in search and analytics.

This day builds the heap from scratch, so you see the two operations that keep the rule true, sift up and sift down. It then uses the heap for top-k selection and for a scheduler, and it compares the result with the standard library at every step.

---

## Short history

The heap idea has a clear history, and the table is a short guide. The Wikipedia article on heaps and the article on heapsort are good places to confirm the details.

| When | Who or what | What changed |
|------|-------------|--------------|
| 1964 | J. W. J. Williams | Introduced heapsort, which uses a heap to sort in place |
| 1964 | Robert W. Floyd | Proposed the bottom-up build for heaps, which runs in linear time |
| 1970s | Priority queues in operating systems | Used for scheduling and timers in early multitasking systems |
| Today | Standard libraries | `heapq` in Python, `PriorityQueue` in Java and `std::priority_queue` in C++ |

---

## Glossary

- **Heap:** a complete binary tree that satisfies the heap property.
- **Complete binary tree:** a tree where every level is full except possibly the last, which fills from left to right. Such a tree fits in an array with no gaps.
- **Min-heap:** a heap where each parent is less than or equal to its children, so the minimum is at the root.
- **Max-heap:** the mirror image, where each parent is greater than or equal to its children.
- **Priority queue:** an abstract data type that returns the highest-priority item on each removal. A heap is the usual implementation.
- **Sift up:** moving a newly added item up the array until its parent is no larger.
- **Sift down:** moving an item down the array until both children are no smaller.
- **Heapify:** turning an arbitrary array into a heap.
- **Top-k:** the k largest or smallest values in a collection.
- **Tie-breaking:** the rule that decides the order of items with equal priority.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Heap property

Picture a hospital triage board. The most critical patient is at the top, and every patient is no more critical than the people they are placed above. The board only guarantees the top rule and leaves the rest of the line unsorted. That is the heap property: a parent is no larger than its children, but the left and right children have no defined order between them.

From first principles, the complete tree shape gives you the array layout for free. For an item at index i, its children are at 2i plus 1 and 2i plus 2, and its parent is at (i minus 1) divided by 2, rounded down. The root is at index 0. Because the tree is complete, the array has no holes, and the height of a heap with n items is about log2 n. Those two facts are the reason heaps are fast and compact.

```python
data = [1, 3, 2, 7, 4, 5, 6]
#        1
#      /   \
#     3     2
#    / \   / \
#   7   4 5   6
for i in range(1, len(data)):
    parent = (i - 1) // 2
    assert data[parent] <= data[i]    # the heap property holds
```

The limitation is that the heap only orders the top. Finding the second smallest item is not O(1), and searching for an arbitrary value is O(n). If you need sorted iteration or range queries, use a balanced BST from Day 46 instead.

The common mistake is to think a heap is sorted because the array looks ordered. A heap array like `[1, 3, 2, 7, 4, 5, 6]` is not sorted, and `[1, 2, 3]` is sorted but not the only valid heap shape. To spot a broken heap, check the parent-child relation for every index, which the test in this day does for you.

Heaps are the core of job schedulers, of the frontier in shortest-path search and of the streaming top-k that analytics dashboards use to report the largest values in real time.

### Operations

Think of a line of people where you add someone at the back and they slowly move forward past anyone who is less important. Or you remove the person at the front, then move the last person into that spot and let them sink back to their proper place. Those two motions are sift up and sift down, and together they keep the heap property after every change.

From first principles, a push appends the item to the end of the array, which keeps the tree complete. Then the item may be smaller than its parent, so it swaps upward until the rule holds. The number of swaps is at most the height, so push is O(log n). A pop takes the root, moves the last item to the root, then swaps it downward with its smaller child until the rule holds. That is also O(log n). Peek reads index 0, which is O(1).

```python
def _sift_up(data: list[int], i: int) -> None:
    while i > 0:
        parent = (i - 1) // 2
        if data[i] >= data[parent]:
            break
        data[i], data[parent] = data[parent], data[i]
        i = parent

def _sift_down(data: list[int], i: int) -> None:
    n = len(data)
    while True:
        left, right = 2 * i + 1, 2 * i + 2
        smallest = i
        if left < n and data[left] < data[smallest]:
            smallest = left
        if right < n and data[right] < data[smallest]:
            smallest = right
        if smallest == i:
            break
        data[i], data[smallest] = data[smallest], data[i]
        i = smallest
```

A bottom-up build is an important refinement. Pushing n items one at a time costs O(n log n). Starting from the last parent and sifting down each one, back to the root, builds the heap in O(n), because most nodes are near the bottom and sift only a short distance. The project implements this as `from_list`.

The limitation of the array heap is that removing an arbitrary item, or changing its priority, needs its index, which the heap does not track. Production heaps add an index map for that case. For this day, a heap supports push, pop and peek, and nothing else.

The common mistake is to forget the last-element move in pop, or to move it but skip the sift down. The heap then looks valid at a glance and returns wrong minimums later. To spot it, pop everything from a random heap and check that the output is sorted. The randomized test in the project does exactly that.

Heap operations run inside the event loops of network servers, where timers are kept in a heap so the next expired timer is at the root.

### Top-k

Suppose you need the ten best-selling products out of ten million sales records. Sorting all of them costs O(n log n) and needs all the data in memory at once. A better method keeps a heap of the ten best so far. When a new value arrives, it competes with the smallest of the ten. If it is larger, it replaces that smallest value. The heap never grows beyond k items.

From first principles, use a min-heap of size k, because its root is the smallest of the current best k, which is the value a new item must beat. Each of the n values costs at most one push and one pop on a heap of size k, so the total is O(n log k). For a small k, that is close to linear, and the memory is O(k) rather than O(n).

```python
def top_k_largest(nums: list[int], k: int) -> list[int]:
    if k <= 0:
        return []
    heap = MinHeap()
    for x in nums:
        if len(heap) < k:
            heap.push(x)
        elif x > heap.peek():
            heap.pop()
            heap.push(x)
    result = []
    while len(heap):
        result.append(heap.pop())
    return result[::-1]

print(top_k_largest([3, 1, 5, 2, 8], 2))   # [8, 5]
```

The limitation is that the answer is only as good as the comparison. Top-k with ties may return any of the tied values, and the code above uses strict greater-than, so an equal value does not replace the existing one. Decide what ties mean for your application and test that case.

The common mistake is to use a max-heap for top-k. A max-heap's root is the largest value, so it does not tell you which of the current best k to drop. The min-heap is the one with the right root. To spot the error, test with k equal to 1 and an input where the largest value comes first.

Top-k is used in search engines for the best results, in monitoring tools for the slowest requests, and in leaderboards that update with every game event.

### Scheduling

Think of an emergency room. Patients are seen by urgency, not by arrival time. Among patients with the same urgency, the earlier arrival goes first. A heap alone does not give you that second rule, because a heap has no memory of arrival order for equal keys. The fix is to add a counter to every item, so that equal priorities compare by arrival.

From first principles, a scheduler stores tuples of (priority, sequence number, job name). Python compares tuples element by element, so the priority decides first, and the sequence number breaks ties. Because the sequence number is unique, the comparison never reaches the name. This matters because two jobs with the same name and priority would otherwise compare as equal, and their order would depend on the heap's internal movements.

```python
import itertools

class Scheduler:
    def __init__(self) -> None:
        self._heap = MinHeap()
        self._counter = itertools.count()

    def add(self, name: str, priority: int) -> None:
        self._heap.push((priority, next(self._counter), name))

    def next_job(self) -> str:
        _, _, name = self._heap.pop()
        return name
```

The limitation is that a plain priority scheduler can starve low-priority jobs forever if high-priority jobs keep arriving. Real schedulers add aging, which slowly raises the priority of waiting jobs, so that nothing waits indefinitely.

The common mistake is to use the priority alone as the heap key for jobs that can share a priority, and then to assume first-come, first-served order. The heap will not give it to you. To spot it, add three jobs with equal priority and check the output order. The sequence counter is the fix.

Priority scheduling runs in operating system kernels, in job queues such as Celery and Sidekiq, and in cloud function platforms that prioritize user-facing requests over batch work.

---

## Hands-on exercises

Work in the repository from Day 46. Keep the heap in `dsa/heap.py` and the scheduler in `dsa/scheduler.py`.

Exercise 1 draws a heap. Insert the values 9, 4, 7, 1, 8, 2 into an empty min-heap by hand, drawing the tree and the array after each push and each sift. Then pop once and draw the result. Compare with the code's `as_list()` output after the same operations.

Done when your final array matches the code's array after the pop, and each sift in your drawing is a swap with the parent.

Exercise 2 times heapify. Build a list of 100000 random integers. Time `MinHeap(items)`, which pushes one by one, against `MinHeap.from_list(items)`, which uses the bottom-up build. Record both times in `docs/heap-notes.md`.

Done when your notes show both times and a sentence about the difference between O(n log n) and O(n).

Exercise 3 compares with `heapq`. Using `random.Random(4)`, push 200 random integers into your heap and into `heapq`, then pop everything from both. Assert the two sequences match.

Done when the comparison passes and you can explain why it compares values only. Plain integers that are equal cannot be told apart, but tuples with equal priorities could come out in a different order from the standard library, so a real test must use the full items.

Exercise 4 checks top-k at the edges. Write tests for `top_k_largest` with k equal to 0, k larger than the list, a list with duplicates such as `[5, 5, 5, 1]` with k equal to 2, and a list where the largest value comes first.

Done when each case passes and your notes explain what the duplicate case returns.

Exercise 5 proves the tie rule. Add three jobs with the same priority to the scheduler in the order "first", "second", "third", and check that `next_job` returns them in that order. Then add a job with a lower priority number after them and check that it runs first.

Done when the tests pass and the GitHub Actions run for your latest commit is green.

---

## Project: a priority job scheduler

The project is a job scheduler built on the heap you wrote. Jobs have a name and a priority, where a lower number means more urgent. Jobs with equal priority run in arrival order. The project includes a demo script and tests that compare the heap with the standard library, so you know the implementation is correct before you trust it with work.

Suggested repo name: `100-days-software-day-47-heaps-and-priority-queues`. Copy the Day 46 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-47-heaps-and-priority-queues/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── count_words.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── bst.py
│   ├── browser_history.py
│   ├── complexity.py
│   ├── hash_table.py
│   ├── heap.py
│   ├── linked_list.py
│   ├── scheduler.py
│   ├── stacks_queues.py
│   └── word_count.py
├── docs/
│   ├── bst-notes.md
│   ├── hash-notes.md
│   └── heap-notes.md
├── height_demo.py
├── print_queue.py
├── schedule_demo.py
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_bst.py
│   ├── test_browser_history.py
│   ├── test_complexity.py
│   ├── test_hash_table.py
│   ├── test_heap.py
│   ├── test_linked_list.py
│   ├── test_print_queue.py
│   ├── test_scheduler.py
│   ├── test_stacks_queues.py
│   └── test_word_count.py
└── README.md
```

Starter code for `dsa/heap.py`:

```python
from __future__ import annotations

from collections.abc import Iterable
from typing import Any


class MinHeap:
    """A binary min-heap stored in a Python list."""

    def __init__(self, items: Iterable[Any] = ()) -> None:
        self._data: list[Any] = []
        for item in items:
            self.push(item)

    @classmethod
    def from_list(cls, items: Iterable[Any]) -> MinHeap:
        """Build a heap in O(n) by sifting down from the last parent to the root."""
        heap = cls()
        heap._data = list(items)
        for i in range(len(heap._data) // 2 - 1, -1, -1):
            heap._sift_down(i)
        return heap

    def __len__(self) -> int:
        return len(self._data)

    def as_list(self) -> list[Any]:
        return list(self._data)

    def peek(self) -> Any:
        if not self._data:
            raise IndexError("peek from empty heap")
        return self._data[0]

    def push(self, item: Any) -> None:
        self._data.append(item)
        self._sift_up(len(self._data) - 1)

    def pop(self) -> Any:
        if not self._data:
            raise IndexError("pop from empty heap")
        top = self._data[0]
        last = self._data.pop()
        if self._data:
            self._data[0] = last
            self._sift_down(0)
        return top

    def _sift_up(self, i: int) -> None:
        data = self._data
        while i > 0:
            parent = (i - 1) // 2
            if data[i] >= data[parent]:
                break
            data[i], data[parent] = data[parent], data[i]
            i = parent

    def _sift_down(self, i: int) -> None:
        data = self._data
        n = len(data)
        while True:
            left, right = 2 * i + 1, 2 * i + 2
            smallest = i
            if left < n and data[left] < data[smallest]:
                smallest = left
            if right < n and data[right] < data[smallest]:
                smallest = right
            if smallest == i:
                break
            data[i], data[smallest] = data[smallest], data[i]
            i = smallest


def top_k_largest(nums: list[int], k: int) -> list[int]:
    """The k largest values, largest first, using a min-heap of size k."""
    if k <= 0:
        return []
    heap = MinHeap()
    for x in nums:
        if len(heap) < k:
            heap.push(x)
        elif x > heap.peek():
            heap.pop()
            heap.push(x)
    result = []
    while len(heap):
        result.append(heap.pop())
    return result[::-1]
```

Starter code for `dsa/scheduler.py`:

```python
import itertools

from dsa.heap import MinHeap


class Scheduler:
    """Run jobs by priority, where a lower number is more urgent.
    Jobs with equal priority run in the order they were added."""

    def __init__(self) -> None:
        self._heap = MinHeap()
        self._counter = itertools.count()

    def __len__(self) -> int:
        return len(self._heap)

    def add(self, name: str, priority: int) -> None:
        self._heap.push((priority, next(self._counter), name))

    def next_job(self) -> str:
        if not self._heap:
            raise IndexError("no jobs to run")
        _, _, name = self._heap.pop()
        return name
```

Starter code for `schedule_demo.py`:

```python
from dsa.scheduler import Scheduler

if __name__ == "__main__":
    scheduler = Scheduler()
    jobs = [("backup", 3), ("send-email", 1), ("resize-image", 2), ("send-receipt", 1)]
    for name, priority in jobs:
        scheduler.add(name, priority)
    while len(scheduler):
        print(scheduler.next_job())
```

Running `python schedule_demo.py` prints send-email, send-receipt, resize-image and backup, in that order.

Starter code for `tests/test_heap.py`:

```python
import heapq
import random
import unittest

from dsa.heap import MinHeap, top_k_largest


def is_min_heap(data: list[int]) -> bool:
    return all(data[(i - 1) // 2] <= data[i] for i in range(1, len(data)))


class MinHeapTests(unittest.TestCase):
    def test_push_keeps_heap_property(self):
        heap = MinHeap()
        for value in [9, 4, 7, 1, 8, 2]:
            heap.push(value)
            self.assertTrue(is_min_heap(heap.as_list()))
        self.assertEqual(heap.peek(), 1)

    def test_pop_returns_sorted_order(self):
        heap = MinHeap([5, 1, 4, 2, 3])
        self.assertEqual([heap.pop() for _ in range(5)], [1, 2, 3, 4, 5])

    def test_empty_heap_raises(self):
        with self.assertRaises(IndexError):
            MinHeap().pop()
        with self.assertRaises(IndexError):
            MinHeap().peek()

    def test_from_list_builds_valid_heap(self):
        heap = MinHeap.from_list([9, 8, 7, 6, 5, 4, 3, 2, 1])
        self.assertTrue(is_min_heap(heap.as_list()))
        self.assertEqual(heap.pop(), 1)

    def test_matches_heapq_on_random_values(self):
        rng = random.Random(4)
        values = [rng.randint(0, 1000) for _ in range(200)]
        mine = MinHeap(values)
        reference = []
        for value in values:
            heapq.heappush(reference, value)
        for _ in range(len(values)):
            self.assertEqual(mine.pop(), heapq.heappop(reference))

    def test_random_push_pop_mix_stays_valid(self):
        rng = random.Random(8)
        heap = MinHeap()
        for _ in range(300):
            if len(heap) == 0 or rng.random() < 0.6:
                heap.push(rng.randint(0, 99))
            else:
                heap.pop()
            self.assertTrue(is_min_heap(heap.as_list()))


class TopKTests(unittest.TestCase):
    def test_top_k_example(self):
        self.assertEqual(top_k_largest([3, 1, 5, 2, 8], 2), [8, 5])

    def test_k_zero_and_k_larger_than_input(self):
        self.assertEqual(top_k_largest([3, 1], 0), [])
        self.assertEqual(top_k_largest([3, 1], 5), [3, 1])

    def test_matches_sorted_on_random_input(self):
        rng = random.Random(2)
        for _ in range(50):
            nums = [rng.randint(0, 50) for _ in range(rng.randint(1, 40))]
            k = rng.randint(1, len(nums))
            with self.subTest(nums=nums, k=k):
                self.assertEqual(top_k_largest(nums, k), sorted(nums, reverse=True)[:k])


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_scheduler.py`:

```python
import unittest

from dsa.scheduler import Scheduler


class SchedulerTests(unittest.TestCase):
    def test_lower_priority_number_runs_first(self):
        s = Scheduler()
        s.add("low", 5)
        s.add("urgent", 1)
        self.assertEqual(s.next_job(), "urgent")
        self.assertEqual(s.next_job(), "low")

    def test_equal_priority_runs_in_arrival_order(self):
        s = Scheduler()
        for name in ["first", "second", "third"]:
            s.add(name, 2)
        self.assertEqual([s.next_job() for _ in range(3)], ["first", "second", "third"])

    def test_new_urgent_job_jumps_the_queue(self):
        s = Scheduler()
        s.add("a", 2)
        s.add("b", 2)
        s.add("c", 1)
        self.assertEqual(s.next_job(), "c")
        self.assertEqual(len(s), 2)

    def test_empty_scheduler_raises(self):
        with self.assertRaises(IndexError):
            Scheduler().next_job()


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 46 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python schedule_demo.py` and confirm the order is send-email, send-receipt, resize-image, backup.
3. Add a `peek_job` method to `Scheduler` that returns the next name without removing it. Add a test that it does not change `len`.
4. Add a `top_k_by_priority(self, k)` method that returns the names of the k most urgent jobs without removing them. Hint: pop k items, collect the names, then push the same tuples back. Test it against a Scheduler with five jobs.
5. Write `docs/heap-notes.md` with the timings from exercise 2, one paragraph on why ties need a sequence counter, and one paragraph on starvation with a sentence on aging.
6. Commit with the message "Add heap, top-k and priority scheduler", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- The random push and pop mix keeps the heap property on every step
- `python schedule_demo.py` prints the four names in the expected order
- `docs/heap-notes.md` contains the heapify timing comparison with numbers
- The latest commit on `main` has a green check

Evidence to commit: `dsa/heap.py`, `dsa/scheduler.py`, `schedule_demo.py`, both test files, `docs/heap-notes.md` and the Actions run. The comparison with `heapq` and the heap-property check after each operation are the strongest evidence that the heap is correct.

This project is light on new behavior and heavy on invariants. The skill is keeping a rule true through every operation, and checking it, which applies to every structure in this module.

---

## Check your understanding

1. An array of seven values is `[2, 5, 3, 9, 6, 4, 8]`. Is it a valid min-heap? Explain using the parent of index 6, and name the index of the parent of index 5.
2. Why does `from_list` sift down from index `n // 2 - 1` to 0, and not from the last index? What does that choice buy you in total work?
3. A teammate says top-k should use a max-heap of all values and pop k times. Compare that approach with the min-heap of size k in memory and time, for n equal to ten million and k equal to ten.
4. A scheduler uses `(priority, name)` tuples instead of `(priority, counter, name)`. Describe a situation where the order is wrong, and say what the counter adds.
5. Why does `pop` move the last item to the root, and not the first item of the second level? What would break if it removed the root and shifted everything left?
6. A priority scheduler serves a steady stream of urgent jobs. The batch jobs never run. Name the problem, and describe a fix that keeps priorities but prevents the problem.

### Answers

1. Yes, it is a valid min-heap. The root holds 2, and each parent is no larger than its children: index 1 (5) has children 9 and 6, and index 2 (3) has children 4 and 8. The parent of index 5 is index 2, because (5 minus 1) divided by 2 is 2, and index 6 has the same parent. Both children of index 2 are larger than 3, so the rule holds.
2. Nodes in the second half of the array are leaves, which need no sifting, so sifting only the internal nodes is enough. Most nodes are near the bottom, and each one sifts only a short distance, so the total work sums to O(n) rather than O(n log n).
3. The max-heap holds all ten million values in memory, which is O(n) space, and popping ten times costs O(n) to build plus O(k log n). The min-heap of size k holds only ten values, which is O(k) space, and does O(n log k) work, which is much faster when k is small.
4. Two jobs with the same priority compare by name, so the job whose name comes first alphabetically runs first, even if it was added later. The counter makes the comparison depend on arrival order, so ties resolve first come, first served.
5. Removing the root from the middle of a heap would leave a gap in a complete tree, and shifting every item would break the parent-child arithmetic for the whole array. Moving the last item to the root keeps the tree complete and changes only one position before sift down.
6. The problem is starvation: the batch jobs always have a larger priority number than the urgent ones, so they never reach the root. A fix is aging, where a job's effective priority improves the longer it waits, so that after enough time it outranks new urgent work.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Heap (data structure)](https://en.wikipedia.org/wiki/Heap_(data_structure))
- [Wikipedia: Priority queue](https://en.wikipedia.org/wiki/Priority_queue)
- [Wikipedia: Heapsort](https://en.wikipedia.org/wiki/Heapsort)
- The Python documentation for the `heapq` module, at docs.python.org/3/library/heapq.html, including the note on heap invariants and the `nlargest` function
- The Python documentation for `itertools.count`, at docs.python.org/3/library/itertools.html

*Tutorials*

- GeeksforGeeks, "Heap Data Structure" and "Priority Queue" tutorials. Search the site for each title.
- W3Schools, the Python tutorial on lists, which covers the list operations used in the array-based heap. Search W3Schools for "Python lists".

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on heaps and heapsort. Search the course site for "heap".
- William Fiset on YouTube, the playlist on heaps and priority queues. Search YouTube for "William Fiset heap".

*Practice*

- VisuAlgo at visualgo.net, the heap visualization. Insert and delete values and watch the sift operations.
- LeetCode, the "Heap (Priority Queue)" study plan at leetcode.com. Start with "Kth Largest Element in a Stream", which is top-k in a live setting.

---

## Tomorrow

Day 48 covers graphs. A graph is a set of nodes with edges between them, and it generalizes the tree from Day 46 by allowing cycles and shared children. You will store graphs as adjacency lists, search them with breadth-first and depth-first search, count connected components, and detect cycles. The heap from today returns in Dijkstra's shortest-path algorithm, which you will meet at the end of the module.

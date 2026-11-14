# Day 44: Stacks and Queues

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** beginner to intermediate. You should have finished Day 43 and be comfortable with linked lists and `collections`. This day is for anyone who has used a call stack, a print spooler or a message broker without knowing the rules that make them behave.

**By the end of this day, you will have:**

- A `Stack` and a `Queue` with clear error behavior, both built with the right Python containers
- A bracket checker that uses a stack to validate nested structure in O(n)
- A monotonic stack that answers "next greater element" for every position in O(n) total
- A monotonic deque that returns the maximum of every sliding window in O(n) total
- A print-queue simulator, committed with tests and a CI run

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 43
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 42: Arrays and Strings (indexing, sliding windows)
- Day 43: Linked Lists (head and tail references, deletion from both ends)
- Module 3: Python `list`, `dict` and `collections`

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [LIFO and FIFO](#lifo-and-fifo)
  - [Deque](#deque)
  - [Monotonic structures](#monotonic-structures)
  - [Applications](#applications)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a print queue simulator](#project-a-print-queue-simulator)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every time your program calls a function, the runtime pushes a frame onto a stack. Every time a web server accepts more requests than it can handle at once, it puts them in a queue. Every time an editor undoes your last change, it pops from a stack of edits. These two shapes, last in first out and first in first out, appear in nearly every system you will work on. They are simple enough to implement in an afternoon and strict enough that a wrong order is an obvious bug.

The reason to study them carefully is that the rules are the point. A stack gives you access only to the top, and that restriction is what makes it useful for nested work such as parsing brackets or tracking function calls. A queue gives you access only to the front and the back, and that restriction makes it fair. Choosing the wrong shape is a common design mistake. Using a Python list as a queue and calling `pop(0)` on it looks harmless, but it costs O(n) per removal, and a busy service can spend most of its time shifting memory.

Today also introduces the monotonic idea. A monotonic stack or deque keeps its contents in sorted order by discarding anything that can never matter again. That single trick turns some problems that look like O(n squared) searches into one pass. You will see it used for "next greater element" and for sliding window maximum, and you will see why each element enters and leaves the structure at most once.

---

## Glossary

- **Stack:** a collection where the last item added is the first one removed. The operations are push, pop and peek.
- **LIFO:** last in, first out. The rule that defines a stack.
- **Queue:** a collection where the first item added is the first one removed. The operations are enqueue and dequeue.
- **FIFO:** first in, first out. The rule that defines a queue.
- **Deque:** a double-ended queue, which supports adding and removing at both ends in O(1). Python's `collections.deque` is one.
- **Call stack:** the stack of active function calls that a runtime maintains during execution.
- **Monotonic:** always increasing or always decreasing, with no reversals.
- **Monotonic stack:** a stack whose values stay in increasing or decreasing order, maintained by popping values that break the order.
- **Monotonic deque:** a deque with the same rule, used to read the front value as the current maximum or minimum.
- **Amortized O(1):** an average cost per operation over a sequence, where some operations are expensive but rare. Day 41 introduced it.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### LIFO and FIFO

Picture a stack of plates in a cafeteria. You add a plate to the top and take the top plate off. You never reach into the middle. That is last in, first out, and it is the rule for a stack. Now picture a line at a ticket counter. The person who arrived first is served first, and new people join at the back. That is first in, first out, the rule for a queue.

From first principles, a stack needs only one end. Push and pop both touch the top, so each is O(1) when the top is the end of a Python list. The `append` and `pop` methods of a list work at the end, so a list is a good stack. A queue needs two ends. Enqueue adds at the back and dequeue removes at the front. Removing from the front of a Python list with `pop(0)` shifts every remaining element, so it is O(n). `collections.deque` removes from the left in O(1), which is why it is the right choice for a queue.

```python
from collections import deque

stack: list[str] = []
stack.append("a")
stack.append("b")
print(stack.pop())            # b, the last one in

queue: deque[str] = deque()
queue.append("a")
queue.append("b")
print(queue.popleft())        # a, the first one in
```

The limitation of a stack is that you cannot reach the middle without popping everything above it. That is a feature when you need the most recent item, and a problem when you need an item in the middle. A queue has the same limit: you cannot jump the line without a different design, such as a priority queue, which Day 47 covers.

The common mistake is using `list.pop(0)` as a queue dequeue. To spot it, search your code for `pop(0)` or `insert(0, ...)`. Both are O(n) on a list, and in a loop they become O(n squared). The fix is to use `deque`.

LIFO shows up in the call stack that your language runtime uses, and FIFO shows up in message queues such as those between a web server and a background worker.

### Deque

Think of a hallway with doors at both ends. People can enter or leave from either end, and the hallway does not care which door they used. A deque is that hallway. It supports appending and popping at both ends, and in Python it does all four operations in O(1) amortized time.

The first principle here is that a deque is a stack and a queue at once. If you use only one end for adding and the same end for removing, it behaves as a stack. If you add at one end and remove at the other, it behaves as a queue. The third style, adding and removing at both ends, is the one that makes a deque useful for the monotonic algorithms later in this day.

```python
from collections import deque

d = deque([2, 3])
d.appendleft(1)     # deque([1, 2, 3])
d.append(4)         # deque([1, 2, 3, 4])
print(d.popleft())  # 1
print(d.pop())      # 4
print(d)            # deque([2, 3])
```

The limitation of `deque` is that indexing in the middle is slower than in a list. `d[i]` is O(n) in the worst case because the deque is a linked sequence of blocks, not one contiguous array. Use a list when you need fast random access and a deque when you need fast removal at the ends.

The common mistake is to pass a `deque` where code expects a list and then slice it, such as `d[1:3]`. Slicing a deque is not supported, so you get a `TypeError`. Convert with `list(d)` first if you need a slice, and remember that the conversion is O(n).

Deques appear in task schedulers that move work between two ends of a queue, in sliding window algorithms, and in breadth-first search, which you will meet on Day 48.

### Monotonic structures

Consider a row of hikers, each with a height, and you ask for each hiker the first person to their right who is taller. A simple approach checks everyone to the right for every hiker, which is O(n squared). A better approach keeps a pile of hikers who are still waiting for a taller person. When a new hiker arrives, any waiting hiker shorter than them is answered, and removed from the pile. The pile stays in order, which is what makes the method monotonic.

From first principles, the monotonic stack works because of one rule: a value that is smaller than a newer value can never be the answer for anything the newer value does not already cover. So you can throw it away. Each element is pushed once and popped at most once, so the total work over the whole array is O(n), even though one step inside the loop may pop several items.

```python
def next_greater_element(nums: list[int]) -> list[int]:
    result = [-1] * len(nums)
    stack: list[int] = []        # indexes still waiting for a larger value
    for i, value in enumerate(nums):
        while stack and nums[stack[-1]] < value:
            result[stack.pop()] = value
        stack.append(i)
    return result

print(next_greater_element([2, 1, 2, 4, 3]))   # [4, 2, 4, -1, -1]
```

The sliding window maximum uses the same idea with a deque. The deque holds indexes, with values decreasing from front to back. The front is always the maximum of the current window. When the window moves, the front is removed if it has fallen out of the window, and the back is trimmed whenever a new value is larger than what is stored there.

```python
def sliding_window_max(nums: list[int], k: int) -> list[int]:
    window: deque[int] = deque()   # indexes, values decreasing front to back
    result: list[int] = []
    for i, value in enumerate(nums):
        while window and nums[window[-1]] <= value:
            window.pop()
        window.append(i)
        if window[0] <= i - k:
            window.popleft()
        if i >= k - 1:
            result.append(nums[window[0]])
    return result

print(sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3))   # [3, 3, 5, 5, 6, 7]
```

The limitation of monotonic structures is that they answer one question at a time. A monotonic stack gives you the next greater element for a fixed definition, and the deque gives you the window maximum for a fixed window size. If the question changes, such as the second greater element or a variable window, you need to adapt the rule and re-check it.

The common mistake is to use `<` where `<=` is needed, or the reverse, and get duplicates wrong. In the window maximum, `<=` removes equal values from the back, so the newest equal value stays at the front. Test with a list of equal numbers, such as `[5, 5, 5]`, to see the difference.

Monotonic stacks appear in stock analysis for "days until a higher price", in histogram problems, and in compilers that resolve operator precedence. Streaming systems use the monotonic deque to track maximums over sliding windows of events.

### Applications

Think of a checklist with nested sections. Each opened section must be closed before its parent. A stack keeps track of the open sections. When you close one, it must match the most recent open section. That is the bracket checker, and it is the same logic a compiler uses to check parentheses, braces and tags in source code.

From first principles, every opening bracket is pushed, and every closing bracket must match the top of the stack and then pop it. If the stack is empty when a closing bracket arrives, or the top does not match, the input is invalid. If anything is still on the stack at the end, something was never closed.

```python
PAIRS = {")": "(", "]": "[", "}": "{"}

def balanced_brackets(text: str) -> bool:
    stack: list[str] = []
    for ch in text:
        if ch in "([{":
            stack.append(ch)
        elif ch in PAIRS:
            if not stack or stack.pop() != PAIRS[ch]:
                return False
    return not stack

print(balanced_brackets("([]{})"))   # True
print(balanced_brackets("([)]"))     # False
```

A print queue is the other classic application. Jobs arrive at the back and the printer takes the job at the front. The project in this day builds that simulator, and it shows how waiting time depends on the order of arrival.

The limitation of these tools is that they assume the work can be processed in arrival order. A print queue that gives urgent jobs priority needs a priority queue instead, which Day 47 covers.

The common mistake in bracket checking is to forget the final stack check. Input such as `"((("` passes every closing-bracket test and returns True if you return early. Always check that the stack is empty at the end. To spot it, test a string that only contains openers.

Bracket checking appears in every language parser, in JSON and XML validators, and in code formatters that need to know nesting depth before they indent a file.

---

## Hands-on exercises

Work in the repository from Day 43. Keep the new module in `dsa/stacks_queues.py` and the simulator in the repository root.

Exercise 1 traces a stack. On paper, write the stack contents after each step of this sequence: push 1, push 2, push 3, pop, push 4, pop, pop. Write the value returned by each pop. Then run the same sequence on a list in the Python shell and compare.

Done when your paper trace and the shell output match exactly.

Exercise 2 finds the cost of a mistake. Write a function that builds a list of 100000 integers, then removes them one at a time with `pop(0)`. Time it with `time.perf_counter`. Repeat with `popleft` on a `deque` of the same size. Record both times in `docs/stack-queue-notes.md`.

Done when your notes show both times and one sentence explaining the difference.

Exercise 3 traces a monotonic stack. For `nums = [4, 5, 2, 10, 8]`, write the stack of indexes after each value is processed in `next_greater_element`. Check your final answer, which should be `[5, 10, 10, -1, -1]`, against the function.

Done when your trace has five rows and the function output matches.

Exercise 4 tests for brackets. Add cases to `tests/test_stacks_queues.py` for an empty string, a string of openers only, a string with letters between brackets and a string with the wrong closer last. Each case must pass.

Done when `python -m unittest` passes and each of the four cases is in its own `subTest`.

Exercise 5 compares with a brute force. Write `max_window_naive(nums, k)` in the test file that uses `max(nums[i:i+k])` for each window. Compare it with `sliding_window_max` on 200 random lists generated with `random.Random(21)`.

Done when the randomized comparison passes, and the GitHub Actions run for your latest commit is green.

---

## Project: a print queue simulator

The project is a print queue simulator. Jobs arrive in a queue with a page count, and the printer handles them in the order they arrived. The simulator reports how many pages were printed before each job starts, which is a simple model of waiting time. The stack and queue classes come from `dsa/stacks_queues.py`, and the monotonic functions go in the same module so the tests cover all of them.

Suggested repo name: `100-days-software-day-44-stacks-and-queues`. Copy the Day 43 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-44-stacks-and-queues/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── browser_history.py
│   ├── complexity.py
│   ├── linked_list.py
│   └── stacks_queues.py
├── docs/
│   ├── linked-list-notes.md
│   ├── stack-queue-notes.md
│   └── window-notes.md
├── print_queue.py
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_browser_history.py
│   ├── test_complexity.py
│   ├── test_linked_list.py
│   ├── test_print_queue.py
│   └── test_stacks_queues.py
└── README.md
```

Starter code for `dsa/stacks_queues.py`:

```python
from collections import deque


class Stack:
    def __init__(self) -> None:
        self._items: list = []

    def push(self, item) -> None:
        self._items.append(item)

    def pop(self):
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()

    def peek(self):
        if not self._items:
            raise IndexError("peek at empty stack")
        return self._items[-1]

    def is_empty(self) -> bool:
        return not self._items

    def __len__(self) -> int:
        return len(self._items)


class Queue:
    def __init__(self) -> None:
        self._items: deque = deque()

    def enqueue(self, item) -> None:
        self._items.append(item)

    def dequeue(self):
        if not self._items:
            raise IndexError("dequeue from empty queue")
        return self._items.popleft()

    def peek(self):
        if not self._items:
            raise IndexError("peek at empty queue")
        return self._items[0]

    def is_empty(self) -> bool:
        return not self._items

    def __len__(self) -> int:
        return len(self._items)


PAIRS = {")": "(", "]": "[", "}": "{"}


def balanced_brackets(text: str) -> bool:
    stack = Stack()
    for ch in text:
        if ch in "([{":
            stack.push(ch)
        elif ch in PAIRS:
            if stack.is_empty() or stack.pop() != PAIRS[ch]:
                return False
    return stack.is_empty()


def next_greater_element(nums: list[int]) -> list[int]:
    """For each index, the next larger value to its right, or -1 if none exists."""
    result = [-1] * len(nums)
    stack: list[int] = []
    for i, value in enumerate(nums):
        while stack and nums[stack[-1]] < value:
            result[stack.pop()] = value
        stack.append(i)
    return result


def sliding_window_max(nums: list[int], k: int) -> list[int]:
    """Maximum of every window of k consecutive values."""
    if k <= 0 or k > len(nums):
        raise ValueError("k must be between 1 and len(nums)")
    window: deque[int] = deque()
    result: list[int] = []
    for i, value in enumerate(nums):
        while window and nums[window[-1]] <= value:
            window.pop()
        window.append(i)
        if window[0] <= i - k:
            window.popleft()
        if i >= k - 1:
            result.append(nums[window[0]])
    return result
```

Starter code for `print_queue.py`:

```python
from dsa.stacks_queues import Queue


def simulate(jobs: list[tuple[str, int]]) -> list[tuple[str, int]]:
    """Process jobs in arrival order. Return (name, pages printed before it starts)."""
    queue = Queue()
    for job in jobs:
        queue.enqueue(job)
    pages_done = 0
    report: list[tuple[str, int]] = []
    while not queue.is_empty():
        name, pages = queue.dequeue()
        report.append((name, pages_done))
        pages_done += pages
    return report


if __name__ == "__main__":
    jobs = [("report.pdf", 12), ("invoice.pdf", 2), ("slides.pdf", 30)]
    for name, start in simulate(jobs):
        print(f"{name:<14} starts after {start} pages")
```

Running `python print_queue.py` prints the three jobs with starts after 0, 12 and 14 pages. Work out those numbers yourself before you run the script, because they are the check that the running total is correct.

Starter code for `tests/test_stacks_queues.py`:

```python
import random
import unittest

from dsa.stacks_queues import (
    Queue,
    Stack,
    balanced_brackets,
    next_greater_element,
    sliding_window_max,
)


class StackQueueTests(unittest.TestCase):
    def test_stack_is_lifo(self):
        s = Stack()
        s.push(1)
        s.push(2)
        self.assertEqual(s.pop(), 2)
        self.assertEqual(s.peek(), 1)
        self.assertEqual(len(s), 1)

    def test_queue_is_fifo(self):
        q = Queue()
        q.enqueue("a")
        q.enqueue("b")
        self.assertEqual(q.dequeue(), "a")
        self.assertEqual(q.dequeue(), "b")
        self.assertTrue(q.is_empty())

    def test_empty_operations_raise(self):
        with self.assertRaises(IndexError):
            Stack().pop()
        with self.assertRaises(IndexError):
            Queue().dequeue()


class BracketTests(unittest.TestCase):
    def test_balanced(self):
        for text in ["", "()", "([]{})", "a(b[c]{d})e"]:
            with self.subTest(text=text):
                self.assertTrue(balanced_brackets(text))

    def test_unbalanced(self):
        for text in ["(", "(]", "())", "([)]"]:
            with self.subTest(text=text):
                self.assertFalse(balanced_brackets(text))


class MonotonicTests(unittest.TestCase):
    def test_next_greater_element(self):
        self.assertEqual(next_greater_element([2, 1, 2, 4, 3]), [4, 2, 4, -1, -1])

    def test_sliding_window_max_example(self):
        self.assertEqual(
            sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3), [3, 3, 5, 5, 6, 7]
        )

    def test_sliding_window_matches_brute_force(self):
        rng = random.Random(11)
        for _ in range(100):
            nums = [rng.randint(-5, 5) for _ in range(rng.randint(1, 25))]
            k = rng.randint(1, len(nums))
            expected = [max(nums[i : i + k]) for i in range(len(nums) - k + 1)]
            with self.subTest(nums=nums, k=k):
                self.assertEqual(sliding_window_max(nums, k), expected)

    def test_next_greater_matches_brute_force(self):
        rng = random.Random(12)
        for _ in range(100):
            nums = [rng.randint(0, 9) for _ in range(rng.randint(0, 20))]
            expected = []
            for i, value in enumerate(nums):
                later = [x for x in nums[i + 1 :] if x > value]
                expected.append(later[0] if later else -1)
            with self.subTest(nums=nums):
                self.assertEqual(next_greater_element(nums), expected)


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_print_queue.py`:

```python
import unittest

from print_queue import simulate


class PrintQueueTests(unittest.TestCase):
    def test_jobs_run_in_arrival_order(self):
        jobs = [("a", 3), ("b", 5), ("c", 1)]
        names = [name for name, _ in simulate(jobs)]
        self.assertEqual(names, ["a", "b", "c"])

    def test_start_pages_add_up(self):
        report = simulate([("report.pdf", 12), ("invoice.pdf", 2), ("slides.pdf", 30)])
        self.assertEqual(report, [("report.pdf", 0), ("invoice.pdf", 12), ("slides.pdf", 14)])

    def test_empty_queue_has_empty_report(self):
        self.assertEqual(simulate([]), [])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 43 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python print_queue.py` and check the output against the starts you worked out by hand: 0, 12 and 14 pages.
3. Add a `Stack`-based function `reverse_words(text: str) -> str` to `dsa/stacks_queues.py` that pushes each word and pops them to rebuild the sentence in reverse order. Add a test for an empty string and for a sentence with two words.
4. Extend `balanced_brackets` tests with a long string of 10000 openers followed by 10000 closers, and check that it returns True without a recursion error.
5. Write `docs/stack-queue-notes.md` that explains, in your own words, why `deque` is the right choice for the print queue and why the monotonic deque is O(n) overall.
6. Commit with the message "Add stacks, queues and monotonic algorithms", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- `python print_queue.py` prints three lines, with starts at 0, 12 and 14 pages
- The randomized tests for both monotonic functions pass
- `docs/stack-queue-notes.md` names the cost of `pop(0)` on a list and why `deque` avoids it
- The latest commit on `main` has a green check

Evidence to commit: `dsa/stacks_queues.py`, `print_queue.py`, both test files, `docs/stack-queue-notes.md` and the Actions run. The randomized comparisons are the strongest evidence that the monotonic code is right.

This project is light on new theory and heavy on careful testing. The print queue is a small model, but the discipline of checking each order and each edge is the skill that transfers.

---

## Check your understanding

1. A function uses a Python list as a queue and calls `pop(0)` on every dequeue, with a million items. Estimate how its total work compares with a `deque`, and explain why the difference grows with the queue length.
2. Why does the bracket checker return False on `"())"` even though every opening bracket has a matching closer somewhere in the string?
3. In `next_greater_element`, why can the algorithm pop several values for one new element, and still run in O(n) overall?
4. In `sliding_window_max`, why does the code check `window[0] <= i - k` before reading the front value?
5. A teammate uses `<` instead of `<=` in the window maximum pop condition. Describe a list where the output is still correct, and one where the deque grows larger than needed but the answer stays the same.
6. You are building a support ticket system where urgent tickets must be handled first. Explain why a plain queue is the wrong choice, and name the structure from Day 47 that fits.

### Answers

1. A list `pop(0)` shifts every remaining element, so each dequeue is O(n), and draining a million items costs roughly half a trillion element moves in total. A `deque` removes from the left in O(1), so the total is O(n). The gap grows with the queue length because the shift cost grows with it.
2. The bracket checker reads left to right, and the first closing bracket at the end must match an opener that is still on the stack. In `"())"`, the first closer pops the only opener, and the second closer finds an empty stack, so it returns False. Order matters as much as the count.
3. Each value is pushed once and popped at most once, so the total number of pops across the whole run is at most n. One expensive step can pop many values, but those pops are paid for by the earlier pushes.
4. The front of the deque can hold an index that has already left the window. Removing it first makes sure the front is always inside the current window, so reading the front gives the correct maximum.
5. With `<`, equal values stay in the deque, so `[5, 5, 5]` with k equal to 2 keeps both indexes. The front is still the correct maximum, so the answer is correct, but the deque holds more entries than needed. With `<=`, the older equal value is removed, which keeps the deque smaller.
6. A plain queue returns tickets strictly in arrival order, so an urgent ticket waits behind everything that arrived earlier. A priority queue, built on a heap, returns the highest-priority item first, which Day 47 covers.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Stack (abstract data type)](https://en.wikipedia.org/wiki/Stack_(abstract_data_type))
- [Wikipedia: Queue (abstract data type)](https://en.wikipedia.org/wiki/Queue_(abstract_data_type))
- [Wikipedia: Double-ended queue](https://en.wikipedia.org/wiki/Double-ended_queue)
- [Wikipedia: Call stack](https://en.wikipedia.org/wiki/Call_stack)
- The Python documentation for `collections.deque`, at docs.python.org/3/library/collections.html

*Tutorials*

- GeeksforGeeks, "Stack Data Structure" and "Queue Data Structure" tutorials. Search the site for each title.
- GeeksforGeeks, "Next Greater Element" and "Sliding Window Maximum". Search the site for each title and compare the approach with the one in this day.
- W3Schools, "Python Lists" tutorial, for the list methods you will compare against the deque. Search the W3Schools Python section for the title.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures that cover stacks, queues and amortized analysis. Search the course site for "stack queue".
- William Fiset on YouTube, the videos on the deque and monotonic queue. Search YouTube for "William Fiset monotonic queue".

*Practice*

- LeetCode, the "Stack" study plan at leetcode.com. Start with "Valid Parentheses" and "Daily Temperatures", which uses the monotonic stack from today.
- Python Tutor at pythontutor.com. Paste `next_greater_element` and step through it on `[2, 1, 2, 4, 3]` to see the stack change at each step.

---

## Tomorrow

Day 45 covers hash tables. A hash table answers "is this key present" in O(1) average time by mapping keys to buckets, and it is the structure behind Python's `dict` and `set`. You will build a chained hash table, watch collisions happen on purpose, and measure how the load factor decides speed. Today's queues and stacks reappear there in one place: each bucket is a small list, and the table's resize rule uses the same amortized argument from Day 41.

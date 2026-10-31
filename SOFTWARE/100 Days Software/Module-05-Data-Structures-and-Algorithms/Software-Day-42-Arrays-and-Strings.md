# Day 42: Arrays and Strings

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** beginner to intermediate. You should have finished Day 41 and be comfortable with Python lists, strings, `while` loops and `dict`. This day is for anyone who writes nested loops over a list and wants a faster way that is still easy to read.

**By the end of this day, you will have:**

- The ability to index and traverse arrays and strings without off-by-one errors
- A prefix sum helper that answers any range sum in constant time after one pass
- A two-pointer search on sorted data and a palindrome check that uses two pointers
- A fixed-size and a variable-size sliding window, each proven correct by tests against a brute-force version
- The extended repository from Day 41, with the new module and its tests running in GitHub Actions

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 41
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)
- Optional: VS Code with the Python extension, for the debugger

**Prerequisites:**

- Day 41: Complexity Analysis (Big-O, the counting habit, `unittest`)
- Module 3: Python lists, slicing, `dict` and string methods
- Day 8: GitHub Workflow, for pushing commits

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Indexing and traversal](#indexing-and-traversal)
  - [Prefix sums](#prefix-sums)
  - [Two pointers](#two-pointers)
  - [Sliding window](#sliding-window)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an arrays and strings toolkit](#project-an-arrays-and-strings-toolkit)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Arrays and strings are the most common data in software. A list of prices, a row of pixels, a log line, a username and a search query are all sequences. Most of the code you will read at work loops over one of them. The difference between a fast loop and a slow one is usually whether the loop does a fresh scan for every answer or reuses the work it has already done.

Day 41 showed that two nested loops over n items do about n squared steps. This day shows the standard ways to remove that nested loop. Prefix sums turn many range queries into subtraction. Two pointers move through sorted data from both ends, so each element is visited once. A sliding window keeps a running summary of a segment and updates it as the segment moves. Each technique has a short proof, and each one has a brute-force version you can test against. The habit to build is simple: write the slow version first, then write the fast version and check that the two agree on many random inputs.

These patterns appear in log analysis, in time series dashboards, in spell checkers, in text search and in any interview question that begins with "given an array". They are also the basis for the more advanced structures in the rest of this module. Hash tables in Day 45 and heaps in Day 47 both come up in the same problems.

---

## Glossary

- **Array:** a sequence of values stored side by side, accessed by integer index. Python's `list` is a dynamic array.
- **String:** an immutable sequence of characters. Python strings support indexing and slicing like lists, but you cannot change a character in place.
- **Index:** the position of an element, starting at 0 in Python.
- **Traversal:** visiting every element of a sequence once, in order.
- **Off-by-one error:** a bug where a loop or slice starts or ends one position too early or too late.
- **Prefix sum:** the running total of the first i elements, stored so that any segment sum becomes a subtraction.
- **Two pointers:** a technique that uses two indices that move through the sequence, usually from both ends or in the same direction at different speeds.
- **Sliding window:** a contiguous segment of the sequence that moves forward, with a summary (sum, count or set) updated as elements enter and leave.
- **Brute force:** the simplest correct method, usually a nested loop, that serves as a reference for checking faster code.
- **Palindrome:** a string that reads the same forwards and backwards after you ignore case and punctuation.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Indexing and traversal

Think of a row of mailboxes numbered from 0. You can open box 3 directly without checking boxes 0, 1 and 2. That is indexing, and it takes the same time no matter where the box is. Traversal is walking down the row and opening each box once. Python lists and strings give you both, and the cost of each is clear from the rules: reading `items[i]` is O(1), and a full pass with `for` is O(n).

The first principles are that a valid index runs from 0 to len minus 1, and that a slice `items[start:stop]` includes `start` and excludes `stop`. Negative indices count from the end, so `items[-1]` is the last element. Those three rules explain most of the bugs in array code.

```python
word = "stack"
print(word[0])       # s
print(word[-1])      # k
print(word[1:4])     # tac, which is indexes 1, 2 and 3
print(len(word))     # 5, so the last valid index is 4

for i, ch in enumerate(word):   # traversal: each character once, O(n)
    print(i, ch)
```

A reliable pattern is to write loop bounds as `range(len(items))` or `for x in items`, and avoid manual index arithmetic unless the algorithm needs it. When you do need it, write a short example with three elements before you trust the bounds. Many off-by-one bugs appear only at the edges, which are the first and last positions.

The common mistake is to write `range(len(items) + 1)` and then read `items[i]` inside the loop, which crashes on the last pass. To spot it, check the loop on an empty list and on a single element. Both should run without an exception. Strings have one more trap: `s[i] = "x"` raises a `TypeError` because strings are immutable. Build a new string with `"".join(chars)` instead.

Traversal is the first step in nearly every data pipeline, from parsing CSV rows to iterating over frames in a video. The cost is always linear, so the question to ask is whether the single pass is enough or whether the job needs a smarter structure.

### Prefix sums

Picture a bank statement with a running balance on every line. To find how much you spent between two dates, you do not add every transaction again. You subtract the balance on the earlier line from the balance on the later line. A prefix sum array is that running balance, computed once.

From first principles, define `prefix[i]` as the sum of the first i elements, so `prefix[0]` is 0 and `prefix` has one more entry than the input. The sum of elements from `left` to `right`, inclusive, is `prefix[right + 1] - prefix[left]`. Building the array takes one pass, O(n), and then each range query takes O(1). If you have q queries and do not precompute, the total is O(n times q). With precomputation it is O(n plus q), which is the difference between a page that answers in milliseconds and one that answers in minutes.

```python
def prefix_sums(nums: list[int]) -> list[int]:
    prefix = [0]
    for x in nums:
        prefix.append(prefix[-1] + x)
    return prefix

nums = [3, 1, 4, 1, 5]
prefix = prefix_sums(nums)       # [0, 3, 4, 8, 9, 14]
print(prefix[3 + 1] - prefix[1]) # sum of nums[1..3] = 1 + 4 + 1 = 6
```

The limitation is that a prefix array is a snapshot. If the input changes, you rebuild the array or use a structure built for updates, such as a Fenwick tree, which is outside this day's scope. Prefix sums also work for sums and counts that can be subtracted. They do not directly give you a range maximum, because you cannot subtract two maxima.

The common mistake is to use `prefix[right] - prefix[left]` and forget the offset. That formula excludes `nums[right]`. The fix is to define the prefix array clearly, document the inclusive or exclusive convention in the function name and in the docstring, and test a range of length one. A quick test is to query `left` equal to `right`, which must return the single element.

Prefix sums show up in analytics dashboards, where a chart asks for the total between two timestamps, and in image processing, where summed-area tables answer box filters in constant time.

### Two pointers

Imagine two people looking for a pair of socks in a sorted drawer, one starting at the left and one at the right. If the pair they hold is too heavy, the person on the right steps in. If it is too light, the person on the left steps forward. Each step rules out one end, so the search never needs to check every pair.

The first principle that makes this work is order. If the array is sorted and you want two numbers that add to a target, moving the left pointer right increases the sum, and moving the right pointer left decreases it. You can decide which pointer to move from one comparison. Each pointer moves at most n times, so the total is O(n), not O(n squared).

```python
def two_sum_sorted(nums: list[int], target: int) -> tuple[int, int] | None:
    left, right = 0, len(nums) - 1
    while left < right:
        total = nums[left] + nums[right]
        if total == target:
            return left, right
        if total < target:
            left += 1
        else:
            right -= 1
    return None

print(two_sum_sorted([1, 3, 4, 6], 7))   # (0, 3), because 1 + 6 = 7
```

The limitation is the sorted requirement. If the input is unsorted, sorting first costs O(n log n), and you lose the original indices unless you carry them along. For unsorted input, a hash table gives O(n) expected time, which Day 45 covers.

The common mistake is to move both pointers on every iteration, which skips valid pairs. Each step should move exactly one pointer, chosen by the comparison. Test with a pair near the middle, such as `[1, 2, 3, 4]` with target 5, to make sure the pointers meet correctly. Two pointers also appears in palindrome checks, where you compare characters from both ends:

```python
def is_palindrome(s: str) -> bool:
    left, right = 0, len(s) - 1
    while left < right:
        if not s[left].isalnum():
            left += 1
        elif not s[right].isalnum():
            right -= 1
        else:
            if s[left].lower() != s[right].lower():
                return False
            left += 1
            right -= 1
    return True
```

Two pointers is the core of merge operations in sorting, where two sorted lists are combined in one pass, and of deduplication in data cleaning, where a sorted list is compacted in place.

### Sliding window

Consider a train with a fixed number of seats. When a new passenger boards at the front, the passenger at the back leaves. You do not count everyone again each time. You update the count with one arrival and one departure. A sliding window does the same with array elements.

From first principles, a window is a contiguous range `[left, right]`. You grow it by moving `right` forward and add the new element to your summary. You shrink it by moving `left` forward and remove the element that leaves. For a window of fixed size k, each step adds one element and removes one, so the total is O(n). For a variable window, such as the longest substring without repeats, each index moves forward at most once, so the total is still O(n).

```python
def max_window_sum(nums: list[int], k: int) -> int:
    if k <= 0 or k > len(nums):
        raise ValueError("k must be between 1 and len(nums)")
    window = sum(nums[:k])
    best = window
    for i in range(k, len(nums)):
        window += nums[i] - nums[i - k]   # add the new element, drop the old one
        best = max(best, window)
    return best

def longest_unique_substring(s: str) -> int:
    last_seen: dict[str, int] = {}
    start = 0
    best = 0
    for end, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= start:
            start = last_seen[ch] + 1          # jump the left edge past the repeat
        last_seen[ch] = end
        best = max(best, end - start + 1)
    return best

print(max_window_sum([1, 2, 3, 4], 2))       # 7, from 3 + 4
print(longest_unique_substring("abcabcbb"))  # 3, from "abc"
```

The limitation is that the summary must support both adding and removing. A sum, a count and a dictionary of counts all do. A maximum does not, because removing the current maximum leaves you needing the next one. That case needs a monotonic deque, which Day 44 covers.

The common mistake is to forget to remove the element that leaves the window, so the sum keeps growing. To catch it, write the window on paper for a three-element array and check that the sum after each step equals the sum of the three elements currently inside. Another mistake in the variable window is to move `start` backward. The check `last_seen[ch] >= start` exists so that an old repeat outside the window is ignored.

Sliding windows appear in network rate limiting, where a service counts requests in the last minute, in streaming analytics and in text search for the shortest span containing every query word.

---

## Hands-on exercises

Work in the repository from Day 41. Create the new module and tests as described in the project, and use the same CI workflow.

Exercise 1 traces indices by hand. On paper, list the indices and values for `nums = [5, 2, 8, 1]`. Write down `nums[-2]`, `nums[1:3]` and `len(nums) - 1` before you run anything. Then run them in the Python shell and check each answer.

Done when your answers match the shell and you can explain why `nums[1:3]` has two elements.

Exercise 2 builds prefix sums by hand. For `nums = [2, 4, 6]`, write the prefix array on paper. Then compute the sum of the range index 1 to index 2 using only the prefix array. Compare with `sum(nums[1:3])`.

Done when your hand-built prefix array is `[0, 2, 6, 12]` and both sums are 10.

Exercise 3 traces two pointers. For `[1, 2, 4, 7, 11, 15]` and target 15, write the left and right values and the sum at each step until you find a pair. Then call `two_sum_sorted` on the same input and check that the returned pair matches your trace. The answer is `(2, 4)`, because 4 plus 11 equals 15.

Done when your trace and the function agree, and you can state the rule that moved each pointer.

Exercise 4 proves a sliding window by testing. Write a brute-force version of `longest_unique_substring` that tries every substring with a set. Use `random.Random(7)` to generate 200 random strings of lowercase letters with lengths from 0 to 20, and assert that the two versions agree.

Done when the randomized comparison passes and you can name the case the brute force handles that the window handles in one pass.

Exercise 5 pushes the work. Add a test that calls `max_window_sum` with `k` equal to the length of the array, and a second test with `k` equal to 0. Confirm the first returns the total sum and the second raises `ValueError`.

Done when `python -m unittest` passes and the GitHub Actions run for your latest commit is green.

---

## Project: an arrays and strings toolkit

The project is a small module of array and string utilities, with tests that compare each fast function to a slow reference. The repository from Day 41 is the starting point, so the workflow, the `dsa` package and the test layout stay the same. The project adds one module, one test file and a short notes file.

Suggested repo name: `100-days-software-day-42-arrays-and-strings`. Start by copying the Day 41 repository into a new folder, deleting the old `docs/growth-notes.md` if you do not need it, and renaming the folder.

The repo layout:

```
100-days-software-day-42-arrays-and-strings/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   └── complexity.py
├── docs/
│   └── window-notes.md
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   └── test_complexity.py
└── README.md
```

The module `dsa/arrays.py` contains the functions from this day. Starter code:

```python
def prefix_sums(nums: list[int]) -> list[int]:
    """prefix[i] is the sum of nums[0:i]. The list has len(nums) + 1 entries."""
    prefix = [0]
    for x in nums:
        prefix.append(prefix[-1] + x)
    return prefix


def range_sum(prefix: list[int], left: int, right: int) -> int:
    """Sum of nums[left..right], inclusive, computed from a prefix list."""
    if not 0 <= left <= right < len(prefix) - 1:
        raise IndexError("range out of bounds")
    return prefix[right + 1] - prefix[left]


def two_sum_sorted(nums: list[int], target: int) -> tuple[int, int] | None:
    """Return indices (i, j), with i < j, where nums[i] + nums[j] == target."""
    left, right = 0, len(nums) - 1
    while left < right:
        total = nums[left] + nums[right]
        if total == target:
            return left, right
        if total < target:
            left += 1
        else:
            right -= 1
    return None


def max_window_sum(nums: list[int], k: int) -> int:
    """Largest sum of any k consecutive numbers."""
    if k <= 0 or k > len(nums):
        raise ValueError("k must be between 1 and len(nums)")
    window = sum(nums[:k])
    best = window
    for i in range(k, len(nums)):
        window += nums[i] - nums[i - k]
        best = max(best, window)
    return best


def longest_unique_substring(s: str) -> int:
    """Length of the longest substring with no repeated characters."""
    last_seen: dict[str, int] = {}
    start = 0
    best = 0
    for end, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= start:
            start = last_seen[ch] + 1
        last_seen[ch] = end
        best = max(best, end - start + 1)
    return best


def is_palindrome(s: str) -> bool:
    """True if the letters and digits in s read the same both ways."""
    left, right = 0, len(s) - 1
    while left < right:
        if not s[left].isalnum():
            left += 1
        elif not s[right].isalnum():
            right -= 1
        else:
            if s[left].lower() != s[right].lower():
                return False
            left += 1
            right -= 1
    return True
```

Starter code for `tests/test_arrays.py`:

```python
import random
import unittest

from dsa.arrays import (
    is_palindrome,
    longest_unique_substring,
    max_window_sum,
    prefix_sums,
    range_sum,
    two_sum_sorted,
)


class PrefixSumTests(unittest.TestCase):
    def test_prefix_sums_small(self):
        self.assertEqual(prefix_sums([3, 1, 4]), [0, 3, 4, 8])

    def test_range_sum_matches_naive(self):
        rng = random.Random(42)
        for _ in range(50):
            nums = [rng.randint(-10, 10) for _ in range(rng.randint(1, 30))]
            prefix = prefix_sums(nums)
            left = rng.randrange(len(nums))
            right = rng.randrange(left, len(nums))
            with self.subTest(nums=nums, left=left, right=right):
                self.assertEqual(
                    range_sum(prefix, left, right), sum(nums[left : right + 1])
                )

    def test_range_out_of_bounds_raises(self):
        with self.assertRaises(IndexError):
            range_sum(prefix_sums([1, 2]), 0, 2)


class TwoPointerTests(unittest.TestCase):
    def test_finds_pair(self):
        self.assertEqual(two_sum_sorted([1, 3, 4, 6], 7), (0, 3))

    def test_no_pair_returns_none(self):
        self.assertIsNone(two_sum_sorted([1, 3, 4, 6], 100))

    def test_empty_input(self):
        self.assertIsNone(two_sum_sorted([], 5))

    def test_palindromes(self):
        self.assertTrue(is_palindrome("A man, a plan, a canal: Panama"))
        self.assertFalse(is_palindrome("race a car"))
        self.assertTrue(is_palindrome(""))
        self.assertFalse(is_palindrome("0P"))


class WindowTests(unittest.TestCase):
    def test_max_window_sum(self):
        self.assertEqual(max_window_sum([1, 2, 3, 4], 2), 7)
        self.assertEqual(max_window_sum([5, -1, 5], 1), 5)

    def test_window_size_must_be_valid(self):
        with self.assertRaises(ValueError):
            max_window_sum([1, 2], 0)

    def test_longest_unique_substring(self):
        self.assertEqual(longest_unique_substring("abcabcbb"), 3)
        self.assertEqual(longest_unique_substring("bbbbb"), 1)
        self.assertEqual(longest_unique_substring("pwwkew"), 3)
        self.assertEqual(longest_unique_substring(""), 0)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 41 repository into the new folder and create `dsa/arrays.py` with the starter code. Run `python -m unittest` and confirm the new tests pass alongside the old ones.
2. Create `tests/test_arrays.py` from the starter. Run it on its own with `python -m unittest tests.test_arrays -v` and read the list of test names.
3. Add a brute-force `max_window_sum_naive` in the test file. It should compute every window with `sum` inside a loop. Add a randomized test that compares it with `max_window_sum` for 100 random lists.
4. Add a brute-force `is_palindrome_naive` that builds a cleaned lowercase string and compares it with its reverse. Compare it with `is_palindrome` on 100 random strings drawn from the characters `"aA1 ,."`.
5. Write `docs/window-notes.md` with one paragraph for each function: what the window or pointers hold, what moves each step, and the total time complexity with a sentence on why.
6. Commit with the message "Add prefix sums, two pointers and sliding window", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- Each fast function has at least one randomized test against a brute-force version
- `docs/window-notes.md` states the time complexity of each function with its reason
- The latest commit on `main` has a green check

Evidence to commit: `dsa/arrays.py`, `tests/test_arrays.py`, `docs/window-notes.md`, the updated README and the workflow run on GitHub. The randomized tests are the proof that the fast versions agree with the slow ones.

This project is light on new ideas, because the techniques are short. The point is the habit of pairing every fast function with a slow reference that you can run.

---

## Check your understanding

1. An array has n elements and you answer q range-sum queries. Compare the total time of recomputing each sum with a loop against building a prefix array first. At what value of q does the prefix array become worth the setup cost?
2. Why does `two_sum_sorted` return None on an empty list without raising an exception, and what would happen if you changed the loop condition to `left <= right`?
3. A sliding window keeps a running sum. A teammate says it also works for finding the largest element in each window. Explain why the sum technique does not transfer directly, and name the structure you would reach for instead.
4. Why does `longest_unique_substring` move `start` to `last_seen[ch] + 1` only when the repeat is inside the current window?
5. Two pointers on a sorted array has O(n) time. Sorting first costs O(n log n). Explain when the total cost of sorting and then using two pointers is still better than a brute-force O(n squared) search, and when it is not.
6. A string-building loop uses `s += ch` for 100000 characters and runs slower than expected in an older Python version. Explain why strings make this harder, and write the fix.

### Answers

1. Recomputing costs about n times q, and the prefix array costs n once plus q times a constant. The prefix array wins once q is more than a small number, because the setup is one pass. For q equal to 1 the loop is as fast. For q equal to 10 with n equal to 1000 the difference is already large.
2. The loop never runs when the list is empty, because `left` is 0 and `right` is negative, so the function falls through to `return None`. Changing the condition to `left <= right` would allow `left` and `right` to be the same element, which pairs an element with itself. That is wrong when the target is twice that element.
3. The sum can be updated by adding and removing an element, but a maximum cannot be updated after removal without knowing the next largest value. A monotonic deque keeps the candidates in order and gives O(n) total time, which Day 44 covers.
4. A repeat before `start` belongs to an older window that has already been left behind, so it must not move `start` backward. Only a repeat inside the current window forces the left edge to move.
5. Sorting plus two pointers is O(n log n) and beats O(n squared) when n is large enough that the difference matters. For small n the brute force can be just as fast. If the input is already sorted, the sort step costs little, and the two-pointer scan alone is O(n).
6. Strings are immutable, so each `+=` can create a new string and copy the old contents, which makes the loop quadratic in the worst case. CPython has an optimization for some cases, but you cannot rely on it. The fix is to collect the characters in a list and call `"".join(parts)` once at the end.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Prefix sum](https://en.wikipedia.org/wiki/Prefix_sum)
- [Wikipedia: Sliding window protocol](https://en.wikipedia.org/wiki/Sliding_window_protocol). It uses a different setting, but the idea of a moving window of contents is the same.
- [Wikipedia: Array data structure](https://en.wikipedia.org/wiki/Array_data_structure)
- [Wikipedia: String (computer science)](https://en.wikipedia.org/wiki/String_(computer_science))
- The Python tutorial, "Strings" and "More on Lists", at docs.python.org/3/tutorial

*Tutorials*

- GeeksforGeeks, "Prefix Sum Array" and "Two Pointers Technique" tutorials. Search the site for each title.
- GeeksforGeeks, "Window Sliding Technique". Search the site for the title and follow the examples in Python.
- W3Schools, "Python Strings" and "Python Lists" tutorials. Use them to check the indexing and slicing rules.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lecture on hashing, which compares two-pointer and hash table approaches to pair finding. Search the course site for "two sum".
- NeetCode on YouTube, the videos on "Two Pointers" and "Sliding Window" in the NeetCode roadmap. Search YouTube for "NeetCode sliding window".

*Practice*

- LeetCode, the "Two Pointers" and "Sliding Window" study plans at leetcode.com. Solve the easy problems first, then compare your solution with the brute force from this day.
- Python Tutor at pythontutor.com. Paste the sliding window function and step through it one line at a time to see the window move.

---

## Tomorrow

Day 43 moves from arrays, which store values side by side, to linked lists, which store each value in a node with a pointer to the next one. You will see why inserting into the middle of an array costs O(n) while a linked list can do it in O(1) once you hold the node, and why that tradeoff matters. The two-pointer habit from today, where one pointer moves faster than the other, returns in the fast and slow pointer technique for finding the middle of a list.

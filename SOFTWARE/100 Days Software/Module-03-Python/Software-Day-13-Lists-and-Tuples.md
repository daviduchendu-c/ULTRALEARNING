# Day 13: Lists and Tuples

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner to intermediate. You know objects, mutability and slicing from Day 10 and Day 12. This is for anyone who has copied a list and then watched both copies change, or who has written a loop to build a list that a single line could build.

**By the end of this day, you will have:**

- A working model of `list` and `tuple` as ordered sequences, and the operations each supports
- The ability to mutate a list safely, and to know which methods change it and which return a new value
- A clear distinction between assignment, shallow copy and deep copy, and the habit of checking which one you need
- Confident use of tuple and list unpacking, including starred targets
- Fluency with list comprehensions and their filtering form, and the judgment to know when a loop reads better
- A playlist repo with tests that prove each copying and mutation rule

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 10: Objects, Variables and Types (references, mutability, `is`)
- Day 12: Strings and Unicode (indexing and slicing, immutable sequences)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Sequence operations](#sequence-operations)
  - [Mutation](#mutation)
  - [Copying](#copying)
  - [Unpacking](#unpacking)
  - [Comprehensions](#comprehensions)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a playlist manager](#project-a-playlist-manager)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most real data is a collection. A cart holds line items, a log holds events, a report holds rows, and an API returns a page of records. Lists and tuples are the two ordered collections that Python gives you for this, and almost every program that processes data uses them constantly. Three things cause most of the trouble: which operations change the collection, what exactly is copied when you pass a list to a function, and why changing one row changes a row in a different table.

The answers come from the object model you learned on Day 10. A list is a mutable object that holds references to other objects. A tuple is an immutable object that also holds references. Copying a list copies the references, not the objects they point to, and that one fact explains most of the surprises. This day builds the full picture, from basic operations through copying, unpacking and comprehensions.

The official reference is the section "Sequence Types (list, tuple, range)" in the Python standard library documentation at docs.python.org/3/library/stdtypes.html. The Python tutorial's section "Data Structures" at docs.python.org/3/tutorial/datastructures.html is short and covers most of this day's material with examples.

---

## Glossary

- **Sequence:** an ordered collection whose items can be reached by position, such as `list`, `tuple`, `str` and `range`.
- **List:** a mutable sequence written with square brackets, such as `[1, 2, 3]`.
- **Tuple:** an immutable sequence written with parentheses or commas, such as `(1, 2, 3)` or `1, 2, 3`.
- **Mutation:** changing an object in place, such as adding an item to a list.
- **Shallow copy:** a new container that holds the same references as the original.
- **Deep copy:** a new container with recursively copied contents, so no inner object is shared.
- **Unpacking:** assigning the items of a sequence to several names in one statement.
- **Starred target:** a name prefixed with `*` in an unpacking, which collects the remaining items into a list.
- **Comprehension:** an expression that builds a list, set or dict from an iterable in one statement.
- **Iterable:** any object that can produce its items one at a time, such as a list, a file or a range.
- **Amortized cost:** the average cost of an operation over a long sequence of operations, which can be low even when a single call is occasionally expensive.

---

## Concepts

The five concepts follow the way you use collections in code. You read and change them, you copy them, you take them apart, and you build new ones from old ones.

### Sequence operations

Think of a train. Each carriage has a position counted from the front, and you can count carriages, ask whether a particular one is in the train, join two trains into a longer one, or repeat a short train several times. A list and a tuple support the same set of operations because both are sequences.

From first principles, `len(seq)` returns the number of items. `x in seq` checks for membership by comparing each item with `==` until it finds a match, which takes time that grows with the length. `seq[i]` reads by position, and `seq[i:j]` returns a new sequence of the same type with the items in that range. `+` joins two sequences of the same type into a new one, and `*` repeats a sequence. Indexing and slicing rules are the same as for strings on Day 12.

```python
tracks = ["Intro", "Main Theme", "Outro"]
print(len(tracks))            # 3
print("Outro" in tracks)      # True
print(tracks[-1])             # Outro
print(tracks[:2])             # ['Intro', 'Main Theme']
print(tracks + ["Bonus"])     # a new list; tracks is unchanged
print([0] * 4)                # [0, 0, 0, 0]
```

The `*` operator has one trap. `[[0] * 3] * 3` creates three references to the same inner list, not three separate lists. Use a comprehension for that case, as shown below.

The common mistake is using `in` on a large list inside a loop, which turns a simple loop into a quadratic one. A set or dictionary gives constant-time membership checks on average. Day 14 covers those, and Day 41 explains the cost model in depth.

### Mutation

Mutation changes a list in place. Think of a shopping list on a whiteboard. You can add an item at the end, cross one out, or rewrite a line, and the whiteboard stays the same whiteboard. A tuple is more like a printed receipt, which you cannot edit.

From first principles, list methods such as `append`, `insert`, `pop`, `remove`, `extend`, `sort` and `reverse` change the list and return `None` or a removed item. `append` adds to the end in amortized constant time, because CPython over-allocates space and grows the array occasionally. `insert(0, x)` shifts every item to the right, so it takes time proportional to the length. `pop()` removes and returns the last item quickly, and `pop(0)` shifts everything left.

```python
queue = [1, 2, 3]
queue.append(4)        # [1, 2, 3, 4]
queue.insert(0, 0)     # [0, 1, 2, 3, 4]
first = queue.pop(0)   # first == 0, queue == [1, 2, 3, 4]
queue.sort(reverse=True)
print(queue)           # [4, 3, 2, 1]

point = (3, 4)
try:
    point[0] = 9
except TypeError as err:
    print("error:", err)   # 'tuple' object does not support item assignment
```

The tuple example confirms immutability. A tuple can contain a list, and then the list can change even though the tuple cannot. That is a fact to remember when you choose a tuple for safety.

The common mistake is calling `sort` or `reverse` and assigning the result. `ordered = tracks.sort()` makes `ordered` equal `None`, and `tracks` is sorted in place. Use `sorted(tracks)` when you want a new list and keep the original. A second mistake is removing items from a list while looping over it with a `for` loop, which skips items. Build a new list with a comprehension or iterate over a copy instead.

### Copying

Copying is where most list bugs live. Think of a photocopy of a folder that contains loose photographs. The copy of the folder is new, but the photographs inside are the same prints. A shallow copy duplicates the folder and keeps the same prints. A deep copy makes new prints as well.

From first principles, `b = a` creates one more reference to the same list. `b = a.copy()`, `b = list(a)` and `b = a[:]` each create a new outer list that holds the same references as `a`. If the items are immutable, such as numbers and strings, the shallow copy is fully independent. If an item is a mutable object, such as another list, the copy and the original share it. `copy.deepcopy(a)` recursively copies every mutable object it reaches, so the copy shares nothing with the original.

```python
import copy

original = [["rock"], ["jazz"]]
shallow = original.copy()
deep = copy.deepcopy(original)

shallow[0].append("pop")
print(original)   # [['rock', 'pop'], ['jazz']], changed through the shared inner list
print(deep)       # [['rock'], ['jazz']], unaffected
```

The shallow copy changed the original because the inner list is one object referenced twice. The deep copy is independent.

The common mistake is calling `copy` and believing that nothing is shared. For a flat list of numbers or strings, a shallow copy is enough. For nested structures, use `deepcopy`, but know that it is slower and can copy large objects you did not intend to copy. In the job market, bugs of this kind show up in configuration handling, where a default settings dictionary gets mutated by one request and affects the next. Clear copying rules prevent that.

### Unpacking

Unpacking is a way of taking a sequence apart into named parts. Think of opening a parcel and laying out its contents on a table, each item in its own place. Python does this with a single assignment statement, and the number of names must match the number of items unless a starred name collects the rest.

From first principles, `a, b = seq` iterates over `seq` and assigns each item in order. If the counts differ, Python raises `ValueError`. A starred target such as `*rest` takes all the items that the other names do not claim, and always produces a list, even if it is empty. You can put the starred name at the start, the middle or the end. Unpacking also works on function arguments with `*` and on dictionaries with `**`, which Day 17 covers. The swap idiom `a, b = b, a` is unpacking on a tuple built from the right side.

```python
x, y = (3, 4)
first, *rest = [1, 2, 3, 4]
*head, last = "abcd"
a, b = 1, 2
a, b = b, a

print(x, y)         # 3 4
print(first, rest)  # 1 [2, 3, 4]
print(head, last)   # ['a', 'b', 'c'] d
print(a, b)         # 2 1
```

The starred name always becomes a list, which is why `head` is `['a', 'b', 'c']` and not a string. The difference shows up when the input is a string.

The common mistake is unpacking a sequence of unknown length into a fixed set of names. A file with one more column than expected raises `ValueError: too many values to unpack`, and that error is often the first sign of a changed input format. Validate the length before unpacking when the source is outside your control, or use a starred target to accept extra columns deliberately.

### Comprehensions

A comprehension is a sentence that describes a list. Think of a recipe card that says "take every ripe apple from the basket and slice it". The sentence names the source, the condition and the transformation in one place. A list comprehension does the same job as a `for` loop that appends items.

From first principles, `[expr for item in iterable if condition]` evaluates `expr` for each `item` where `condition` holds, and collects the results in a new list. The `if` part is optional. You can nest comprehensions, but a second `for` clause reads left to right, in the same order as nested loops. Comprehensions create a new list, so they are not a way to mutate an existing one.

```python
durations = [95, 241, 60, 180]
minutes = [round(s / 60, 1) for s in durations]
long_ones = [s for s in durations if s > 120]
grid = [[0] * 3 for _ in range(3)]    # three separate inner lists
pairs = [(i, j) for i in range(2) for j in range(2)]

print(minutes)     # [1.6, 4.0, 1.0, 3.0]
print(long_ones)   # [241, 180]
print(grid)        # [[0, 0, 0], [0, 0, 0], [0, 0, 0]]
print(pairs)       # [(0, 0), (0, 1), (1, 0), (1, 1)]
```

The `grid` line shows the fix for the `[[0] * 3] * 3` trap. The comprehension builds a new inner list on each iteration.

The common mistake is writing a comprehension with side effects, such as one that calls `print` or appends to another list. The comprehension then reads like data, but it behaves like a loop, and the reader misses the effect. Use a `for` loop when the purpose is an action. Keep comprehensions for transformations and filters. Nested comprehensions with three or more clauses are a sign that a function with a loop will read better. In the job market, code reviewers ask for this simplification often, because the shorter version is not always the clearer one.

---

## Hands-on exercises

Work in a folder called `lists-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices sequence operations.

1. Create `sequence.py`. Make a list of five track titles. Print its length, check membership with `in`, print the last item, and print the first three items as a slice.
2. Print the result of `titles + ["Bonus"]` and confirm that `titles` is unchanged.
3. Print `[0] * 4`. Then write one sentence explaining why `[[0] * 3] * 3` is risky.

Done when every printed value matches your prediction and your sentence names the shared-reference problem.

Exercise 2 practices mutation.

1. Create `mutation.py`. Start with `[1, 2, 3]`. Append 4, insert 0 at the front, pop the first item into a variable, and sort in reverse.
2. Print the list after each step, and print the return value of `sort`.
3. Write a tuple version of the same starting data and try to assign to an index. Record the error message.

Done when you can explain why `sort` returns `None` and why the tuple refuses assignment.

Exercise 3 demonstrates copying.

1. Create `copying.py`. Make a list of lists that holds two genres, each with a list of artists.
2. Create a shallow copy with `.copy()` and a deep copy with `copy.deepcopy`. Append a new artist to the first genre in the shallow copy.
3. Print the original and the deep copy after the change. Write one sentence explaining the result for each.

Done when the original changed and the deep copy did not.

Exercise 4 practices unpacking and comprehensions.

1. Create `records.py`. Take a list of `("title", seconds)` tuples and unpack each into `title` and `seconds` in a loop.
2. Use a starred unpacking to split a list of ten numbers into the first, the last and the middle.
3. Write a comprehension that returns the titles longer than four minutes, and a second comprehension that returns the durations in minutes rounded to one decimal place.

Done when both comprehensions produce the expected values and you can rewrite each one as a loop.

Exercise 5 runs the type checker.

1. Write `typed_lists.py` with a function `average(values: list[float]) -> float` that returns the mean. Handle the empty list by raising `ValueError`.
2. Call `average` once with a list of integers and once with a list of strings.
3. Run `python -m mypy typed_lists.py`. Read the error for the string call, fix the call, and rerun until mypy reports `Success: no issues found`.

Done when the final run is clean and the function raises a clear error on an empty list.

---

## Project: a playlist manager

The project is a small playlist library. Each track is a title and a duration. Functions add and remove tracks without changing the caller's list, compute totals, find the longest track, and show when a copy is shallow or deep. The code is short and the rules are precise. The evidence is a test suite that demonstrates each rule and a findings file that explains the shallow-copy result in your own words.

Suggested repo name: `100-days-software-day-13-lists-and-tuples`.

Repo layout:

```
100-days-software-day-13-lists-and-tuples/
├── .gitignore
├── README.md
├── playlist.py
├── sequence_demo.py
├── tests/
│   ├── __init__.py
│   └── test_playlist.py
└── docs/
    └── findings.md
```

Starter code for `playlist.py`:

```python
from copy import deepcopy

type Track = tuple[str, int]


def total_seconds(tracks: list[Track]) -> int:
    return sum(seconds for _, seconds in tracks)


def add_track(tracks: list[Track], track: Track) -> list[Track]:
    return [*tracks, track]


def remove_title(tracks: list[Track], title: str) -> list[Track]:
    return [track for track in tracks if track[0] != title]


def longest(tracks: list[Track]) -> Track | None:
    if not tracks:
        return None
    return max(tracks, key=lambda track: track[1])


def formatted(track: Track) -> str:
    title, seconds = track
    minutes, rest = divmod(seconds, 60)
    return f"{title} ({minutes}:{rest:02d})"


def shallow_snapshot(playlists: list[list[str]]) -> list[list[str]]:
    return playlists.copy()


def deep_snapshot(playlists: list[list[str]]) -> list[list[str]]:
    return deepcopy(playlists)


if __name__ == "__main__":
    library: list[Track] = [("Intro", 95), ("Main Theme", 241), ("Outro", 60)]
    library = add_track(library, ("Bonus", 180))
    print([formatted(track) for track in library])
    print("total seconds:", total_seconds(library))
    top = longest(library)
    if top is not None:
        print("longest:", formatted(top))
```

Starter code for `sequence_demo.py`:

```python
if __name__ == "__main__":
    first, *middle, last = [1, 2, 3, 4, 5]
    print(first, middle, last)

    left, right = "ab"
    print(left, right)

    a, b = 1, 2
    a, b = b, a
    print(a, b)
```

Starter code for `tests/test_playlist.py`:

```python
import unittest

from playlist import (
    add_track,
    deep_snapshot,
    formatted,
    longest,
    remove_title,
    shallow_snapshot,
    total_seconds,
)


class PlaylistTests(unittest.TestCase):
    def setUp(self):
        self.library = [("Intro", 95), ("Main Theme", 241), ("Outro", 60)]

    def test_add_returns_new_list(self):
        updated = add_track(self.library, ("Bonus", 180))
        self.assertEqual(len(self.library), 3)
        self.assertEqual(len(updated), 4)

    def test_remove_title(self):
        remaining = remove_title(self.library, "Outro")
        self.assertEqual([title for title, _ in remaining], ["Intro", "Main Theme"])

    def test_total_seconds(self):
        self.assertEqual(total_seconds(self.library), 396)

    def test_formatted(self):
        self.assertEqual(formatted(("Intro", 95)), "Intro (1:35)")

    def test_longest_of_empty_is_none(self):
        self.assertIsNone(longest([]))

    def test_longest(self):
        self.assertEqual(longest(self.library), ("Main Theme", 241))

    def test_tuples_are_immutable(self):
        with self.assertRaises(TypeError):
            self.library[0][1] = 10

    def test_shallow_copy_shares_inner_lists(self):
        playlists = [["rock"]]
        snapshot = shallow_snapshot(playlists)
        snapshot[0].append("pop")
        self.assertEqual(playlists, [["rock", "pop"]])

    def test_deep_copy_is_independent(self):
        playlists = [["rock"]]
        snapshot = deep_snapshot(playlists)
        snapshot[0].append("pop")
        self.assertEqual(playlists, [["rock"]])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python sequence_demo.py`. Paste the output into `docs/findings.md`, and write one sentence explaining why `middle` is a list even when it has one item.
3. Run `python playlist.py`. Paste the output into `docs/findings.md`.
4. Run `python -m unittest` and make every test pass.
5. Add a function `sort_by_duration(tracks: list[Track]) -> list[Track]` that returns a new sorted list, using `sorted` with a key. Add a test showing that the original list is unchanged.
6. Add a comprehension-based function `titles_over(tracks: list[Track], seconds: int) -> list[str]`. Add a test with at least two tracks above and one below the threshold.
7. Run `python -m mypy playlist.py sequence_demo.py tests/test_playlist.py` and fix every reported issue.
8. Write a short section in `docs/findings.md` called "Shallow and deep copies", which explains the output of the two snapshot tests in your own words, and name one case where a shallow copy is correct.
9. Commit with the message "Add playlist manager with copy and unpacking tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least eleven tests and passes
- `python -m mypy` reports no issues on the modules
- `docs/findings.md` contains the outputs, the copy explanation, and the one-line case where a shallow copy is correct
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with the run commands), and `.gitignore`.

The project is light on features and heavy on rules about copying. A reviewer who reads the two snapshot tests should be able to see which one shares state and why that matters for a playlist that several users edit.

---

## Check your understanding

1. You write `playlist2 = playlist` and then `playlist2.append(track)`. Does `playlist` change? Explain using the reference model, and name the operation that would avoid the change.
2. A list of lists is copied with `.copy()`. One inner list is then changed through the copy. Explain why the original changed, and name the function that would prevent it.
3. A developer writes `for track in tracks: if track.title == "Old": tracks.remove(track)`. Predict what goes wrong and give a correct approach.
4. A student expects `first, *rest = ()` to succeed and assign `first` to `None`. Explain what actually happens, and say what the failure looks like.
5. Why is `[[0] * 3] * 3` a bug, and why is `[[0] * 3 for _ in range(3)]` correct? Explain in terms of references.
6. A codebase uses `in` on a list of ten thousand IDs inside a loop over ten thousand records. Explain why this is slow, and name the data structure that makes the membership check cheap.

### Answers

1. Yes, `playlist` changes, because both names refer to the same list object, so the append is visible through either name. `playlist2 = playlist.copy()` avoids the shared reference, and `[*playlist, track]` builds a new list without touching the original.
2. The copy holds the same references to the inner lists, so the inner list is one object reachable from both outer lists. `copy.deepcopy` creates new inner lists as well, so the copy shares nothing with the original.
3. Removing items from a list while iterating over it skips the item after each removal, so some matching items may remain. A correct approach is to build a new list with a comprehension, `tracks = [t for t in tracks if t.title != "Old"]`, or to iterate over a copy.
4. Unpacking an empty tuple into `first` and a starred name raises `ValueError: not enough values to unpack`, because `first` needs at least one item. Nothing is assigned, and no `None` appears.
5. The first form creates three references to one inner list, so changing one row changes all of them. The comprehension evaluates `[0] * 3` on each pass, which creates a new inner list every time, so each row is independent.
6. A list membership check scans the list one item at a time, so ten thousand checks over ten thousand items do about one hundred million comparisons. A `set` of IDs gives an average constant-time check, which reduces the work to about ten thousand operations.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Sequence Types (list, tuple, range)", at docs.python.org/3/library/stdtypes.html
- Python documentation, "Data Structures", at docs.python.org/3/tutorial/datastructures.html
- Python documentation, module `copy`, at docs.python.org/3/library/copy.html
- [Wikipedia: Array data structure](https://en.wikipedia.org/wiki/Array_data_structure)
- [Wikipedia: Tuple](https://en.wikipedia.org/wiki/Tuple)
- [Wikipedia: Object copying](https://en.wikipedia.org/wiki/Object_copying)

*Tutorials*

- GeeksforGeeks, "Python List" and "Python Tuples" tutorials, at geeksforgeeks.org
- W3Schools, "Python Lists" and "Python Tuples" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on lists and loops.

*Practice*

- Python Tutor at pythontutor.com. Step through the shallow copy example and the list comprehension example, and watch how the references in the diagram change.

---

## Tomorrow

Day 14 covers the two collections that answer "is this key present" in constant time on average. Dictionaries map keys to values, and sets hold unique items. You will learn what hashing means, why only hashable objects can be keys, how mapping operations work, and how dictionary and set comprehensions fit into the patterns from today. The membership cost you measured at the end of this day is the problem those structures solve.

**Next:** [Day 14: Dictionaries and Sets](Software-Day-14-Dictionaries-and-Sets.md)

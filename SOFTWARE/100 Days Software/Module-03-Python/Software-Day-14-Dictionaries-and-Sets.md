# Day 14: Dictionaries and Sets

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner to intermediate. You know lists, tuples, comprehensions and mutability from Day 13. This is for anyone who has searched a list inside a loop and watched the program slow down as the data grew.

**By the end of this day, you will have:**

- A working model of hashing: how a dictionary or set turns a key into a place to look, and why that makes lookups fast
- The ability to tell which objects are hashable, and to choose a valid key for a dictionary or set
- Fluency with mapping operations, including `get`, `setdefault`, `pop`, `update`, merging with `|`, and iteration over items
- Confident use of set operations (union, intersection, difference, symmetric difference) and subset tests
- Dictionary and set comprehensions, and the judgment to choose between them and a loop
- An inventory repo with tests that show each rule, and a timing comparison that shows the cost of lists against sets

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 10: Objects, Variables and Types (identity, equality, mutability)
- Day 13: Lists and Tuples (membership cost, copying, comprehensions)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Hashing concept](#hashing-concept)
  - [Hashability](#hashability)
  - [Mapping operations](#mapping-operations)
  - [Sets](#sets)
  - [Comprehensions for dicts and sets](#comprehensions-for-dicts-and-sets)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an inventory counter](#project-an-inventory-counter)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A product catalog has ten thousand items. A customer types a product code, and the system must find the price immediately. A list could be searched one item at a time, which means up to ten thousand comparisons per lookup. A dictionary finds the item in about the same time whether the catalog has ten items or ten million. That difference is why dictionaries and sets appear in almost every serious program: caches, indexes, deduplication, feature flags, permission checks, graph traversal and counting all depend on them.

The idea behind both structures is hashing. A hash function turns a key into a number, and the number tells the structure where to look. Once you understand that mechanism, three things become clear: why only certain objects can be keys, why iteration order follows insertion, and why a set of values cannot contain duplicates. This day builds that understanding and then uses it in the operations you will write every week.

The official reference is the section "Mapping Types (dict)" and "Set Types (set, frozenset)" in the Python standard library documentation at docs.python.org/3/library/stdtypes.html. The Python tutorial section "Data Structures" at docs.python.org/3/tutorial/datastructures.html covers the same ground with short examples.

---

## Glossary

- **Hash:** an integer that a hash function computes from an object's value. Equal objects must have equal hashes.
- **Hash function:** a function that maps an object to an integer, such as the built-in `hash`.
- **Hash table:** a structure that uses hashes to choose a slot for each key, so lookups jump directly to the slot.
- **Hashable:** an object that has a hash value that never changes during its lifetime. Immutable built-in types are hashable.
- **Key:** the name a dictionary uses to find a value, such as `"A1"` in `{"A1": 3}`.
- **Mapping:** a collection that associates keys with values, such as a `dict`.
- **Set:** an unordered collection of unique, hashable items.
- **frozenset:** an immutable set. It can be a key or an item of another set.
- **Collision:** two different keys that produce the same slot in a hash table. The structure resolves it and keeps both keys.
- **Insertion order:** the order in which keys were first added to a dictionary, which Python preserves when iterating.
- **Comprehension:** an expression that builds a collection in one statement, covered on Day 13 for lists.

---

## Concepts

The concepts move from the mechanism to its consequences. Hashing explains the speed, hashability explains which keys are allowed, mapping and set operations are the everyday tools, and comprehensions are the concise way to build both.

### Hashing concept

Think of a coat check at a theater. You hand over your coat and receive a ticket with a number. The attendant does not search every rack. The number tells them the exact hook. A hash table works the same way. A hash function turns a key into a number, the number points to a slot, and the value goes into that slot. Finding the value later means computing the number again and going straight to the slot.

From first principles, a dictionary keeps an array of slots. To insert `key: value`, Python computes `hash(key)`, reduces it to a slot index, and stores the entry there. To look up `key`, it computes the same hash, goes to the slot, and checks that the stored key is equal to the one it searched for. If two different keys land in the same slot, which is a collision, CPython probes other slots until it finds a free one or the matching key. With a reasonable hash function and a table that is kept sparse by resizing, the average cost of lookup, insertion and deletion stays roughly constant as the table grows.

```python
prices = {"tea": 2.5, "cake": 4.25}
print(prices["cake"])     # 4.25, found by hash, not by scanning
print(hash("cake") == hash("cake"))  # True in the same process
```

Hash values for strings are randomized per process by default, which is a security feature against crafted collisions. Running `python -c "print(hash('cake'))"` twice gives different numbers. Never store hash values to disk and expect them to match later, because the value is not stable across runs.

The common mistake is believing that a dictionary lookup is always constant time. The average case is constant, and the worst case degrades when many keys collide, which happens with bad hash functions and with mutable keys that change after insertion. The second mistake is confusing the speed of a lookup with the speed of a full scan. Iterating over every item in a dictionary is still linear. In the job market, the cost model of hash tables appears in system design interviews and in performance reviews, and knowing it lets you explain why a cache helps.

### Hashability

Hashability is a promise. An object is hashable when it has a `__hash__` method that returns the same number for its whole life, and when equal objects produce equal hashes. Think of a locker key that never changes its cut. If the key changed after you put it in the locker, you could not find the locker again.

From first principles, Python sets `__hash__` for immutable built-in types, such as `int`, `float`, `str`, `bytes`, `tuple` (when its contents are hashable) and `frozenset`. Mutable containers such as `list`, `dict` and `set` set their `__hash__` to `None`, so `hash([1, 2])` raises `TypeError`. A class you write has a default hash based on identity, unless you define `__eq__`, in which case Python sets `__hash__` to `None` unless you define it too. Day 21 explains the data model methods in full.

```python
print(hash((1, 2)) == hash((1, 2)))   # True
try:
    lookup = {[1, 2]: "list key"}
except TypeError as err:
    print("error:", err)              # cannot use 'list' as a dict key (unhashable type: 'list')

pairs = {(1, 2): "seat A1"}
print(pairs[(1, 2)])                  # seat A1
```

The tuple works as a key because its contents are immutable. A tuple that contains a list is not hashable, even though the tuple itself is immutable, because the list inside has no stable hash.

The common mistake is using a mutable object as a key, changing it, and then losing the entry. The dictionary still holds the entry in the slot chosen by the old hash, and a lookup with the new value looks in the wrong slot. The dictionary may appear to lose data. A safe rule is to use strings, numbers and tuples of those as keys, and to convert a list into a tuple when you need it as a key. In the job market, cache keys built from request parameters are a common place where this rule matters.

### Mapping operations

A dictionary is a set of labeled drawers. Each drawer has a label, which is the key, and holds one item, which is the value. You can open a drawer, check whether a label exists, add a drawer, remove one, or list all the labels and contents. The operations are the verbs of the structure.

From first principles, `d[key]` returns the value or raises `KeyError`. `d.get(key, default)` returns the value or the default, and never raises for a missing key. `d.setdefault(key, default)` inserts the default if the key is missing, then returns the value. `d.pop(key, default)` removes and returns a value. `d.update(other)` merges another mapping into `d`, and `d | other` returns a new merged dictionary, which Python added in version 3.9. `keys()`, `values()` and `items()` return views that reflect later changes to the dictionary. Since Python 3.7, iteration follows insertion order.

```python
stock = {"A1": 3, "B2": 1}
print(stock.get("Z9", 0))           # 0, no error
stock.setdefault("C3", 0)           # inserts C3 with 0
print(stock.pop("B2"))              # 1, removed
print(stock | {"A1": 5})            # merged copy, A1 becomes 5
for sku, qty in stock.items():
    print(sku, qty)
```

The `|` operator on dictionaries returns a new dictionary, and when keys overlap the right side wins. The original `stock` is unchanged, which is the same non-mutating style you used for lists on Day 13.

The common mistake is using `d[key]` for a value that might be missing, then catching `KeyError` in a wide `except` block. Use `get` when a default makes sense, and `in` when you need to branch on presence. The second mistake is changing a dictionary while iterating over it, which raises `RuntimeError` when the size changes. Iterate over `list(d)` when you need to modify the dictionary during the loop. Code reviewers catch these patterns in data pipelines, where a dictionary often holds the state of a long-running process.

### Sets

A set is a bag of unique items with no labels and no order guarantees. Think of the guest list for a party, where each person appears once and you care only about who is on it. Set operations answer questions about two lists of people: who is on both, who is on either, who is on one and not the other.

From first principles, a set stores items in a hash table with no values. Adding an item that is already present does nothing. Union `a | b` contains items in either set. Intersection `a & b` contains items in both. Difference `a - b` contains items in `a` but not in `b`. Symmetric difference `a ^ b` contains items in exactly one of the two. Subset `a <= b` checks that every item of `a` is in `b`. Intersection runs in time proportional to the smaller set on average, and union and difference run in time proportional to the sizes of the sets they read, not to the product of the two sizes.

```python
scanned = {"A1", "B2", "C3"}
expected = {"B2", "C3", "D4"}

print(scanned & expected)   # {'B2', 'C3'}
print(expected - scanned)   # {'D4'}, missing from the scan
print(scanned | expected)   # all four codes
print({"B2"} <= scanned)    # True
print(scanned ^ expected)   # {'A1', 'D4'}
```

Sets print in an arbitrary order, so sort them before showing them to a user. `frozenset` provides the same operations on an immutable set, and you can use it as a dictionary key.

The common mistake is using a set when order matters, such as a queue of tasks or a list of steps. A set has no order to rely on, and the output may change between runs for strings because of hash randomization. Use a list when order is part of the meaning. The second mistake is writing `{}` and expecting an empty set. It creates an empty dictionary. Write `set()` for an empty set.

### Comprehensions for dicts and sets

Comprehensions build dictionaries and sets with the same shape as list comprehensions from Day 13. The difference is the brackets and the output. Think of a sorting table with two outputs: one tray gets the labeled drawers, and the other gets the unique tags. The source is the same, and the destination changes.

From first principles, `{key_expr: value_expr for item in iterable if condition}` builds a dictionary, and `{expr for item in iterable if condition}` builds a set. Later items with the same key overwrite earlier ones in a dictionary comprehension, and duplicates vanish in a set comprehension. The `if` clause filters, and it works in both forms.

```python
orders = ["A1", "B2", "A1", "C3", "A1"]
counts = {sku: orders.count(sku) for sku in set(orders)}
unique = {sku.lower() for sku in orders}
prices = {"A1": 2.5, "B2": 4.25, "C3": 0.0}
paid = {sku: price for sku, price in prices.items() if price > 0}

print(sorted(counts.items()))   # [('A1', 3), ('B2', 1), ('C3', 1)]
print(sorted(unique))           # ['a1', 'b2', 'c3']
print(paid)                     # {'A1': 2.5, 'B2': 4.25}
```

The first comprehension calls `orders.count` for each unique SKU, which is quadratic. The next section's project uses a single pass instead, and the difference becomes visible on large inputs.

The common mistake is using `orders.count(sku)` inside a comprehension and not noticing that it scans the whole list each time. That is correct output with poor performance. Count in one pass with a loop, `collections.Counter`, or a dictionary updated with `get`. Day 23 covers `collections`. In the job market, interviewers look for exactly this kind of hidden quadratic work.

---

## Hands-on exercises

Work in a folder called `dicts-sets-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 demonstrates hashing and hashability.

1. Create `hashing.py`. Print `hash("tea") == hash("tea")`. Then try to use a list as a dictionary key inside a `try` block, and print the error message.
2. Use a tuple `(3, 4)` as a key and look up its value.
3. Run `python -c "print(hash('tea'))"` twice in separate processes. Write one sentence on what you observe and why.

Done when the list key produces a `TypeError` and your sentence explains the hash randomization.

Exercise 2 practices mapping operations.

1. Create `mapping.py`. Start with `stock = {"A1": 3, "B2": 1}`.
2. Use `get` to read `"Z9"` with a default of 0. Use `setdefault` to add `"C3"` with 0. Use `pop` to remove `"B2"`.
3. Merge `{"A1": 5, "D4": 2}` into a new dictionary with `|`. Print the original and the merged result.

Done when the original dictionary is unchanged after the merge, and you can explain why `setdefault` and `get` differ.

Exercise 3 practices set operations.

1. Create `sets.py`. Make a set of scanned codes and a set of expected codes. Each should have at least four items and some overlap.
2. Print the intersection, the difference in both directions, the union, and the symmetric difference.
3. Check whether the scanned codes are a subset of the expected codes, and print the result.

Done when each printed set matches the result you worked out on paper first.

Exercise 4 practices comprehensions.

1. Create `comprehensions.py`. Given a list of order codes with repeats, build a dictionary of counts in a single pass with a `for` loop and a `get` call.
2. Build the same dictionary with a comprehension over `set(orders)` and `count`. Compare the results.
3. Time both versions on a list of one hundred thousand codes using `time.perf_counter`. Record both times and explain which is slower and why.

Done when the two results are equal and your timing note names the quadratic step.

Exercise 5 runs the type checker.

1. Write `typed_inventory.py` with a function `restock(stock: dict[str, int], sku: str, qty: int) -> dict[str, int]` that returns a new dictionary with the quantity increased.
2. Call it once with a key that exists and once with a key that does not.
3. Run `python -m mypy typed_inventory.py`. Fix any reported issue, and rerun until mypy reports `Success: no issues found`.

Done when the function never mutates the input dictionary and mypy is clean.

---

## Project: an inventory counter

The project is a small inventory library that counts scanned codes, merges stock counts, finds the unique codes, looks up prices with defaults, and groups codes by category. Each function uses one of the structures from this day, and each test checks the rule it depends on. The evidence is the passing test suite and a findings document that records the timing comparison from the exercise.

Suggested repo name: `100-days-software-day-14-dictionaries-and-sets`.

Repo layout:

```
100-days-software-day-14-dictionaries-and-sets/
├── .gitignore
├── README.md
├── inventory.py
├── hashing_demo.py
├── tests/
│   ├── __init__.py
│   └── test_inventory.py
└── docs/
    ├── findings.md
    └── timing.py
```

Starter code for `inventory.py`:

```python
from collections import defaultdict
from collections.abc import Iterable


def count_stock(skus: Iterable[str]) -> dict[str, int]:
    counts: dict[str, int] = {}
    for sku in skus:
        counts[sku] = counts.get(sku, 0) + 1
    return counts


def merge_stock(a: dict[str, int], b: dict[str, int]) -> dict[str, int]:
    merged = dict(a)
    for sku, qty in b.items():
        merged[sku] = merged.get(sku, 0) + qty
    return merged


def unique_skus(skus: Iterable[str]) -> set[str]:
    return set(skus)


def price_of(prices: dict[str, float], sku: str, default: float = 0.0) -> float:
    return prices.get(sku, default)


def group_by_category(categories: dict[str, str]) -> dict[str, list[str]]:
    groups: defaultdict[str, list[str]] = defaultdict(list)
    for sku, category in categories.items():
        groups[category].append(sku)
    return dict(groups)


if __name__ == "__main__":
    scans = ["A1", "B2", "A1", "C3", "A1"]
    counts = count_stock(scans)
    print(counts)
    print(merge_stock(counts, {"B2": 2, "D4": 1}))
    print(sorted(unique_skus(scans)))
    print(group_by_category({"A1": "tea", "B2": "cake", "C3": "tea"}))
```

Starter code for `hashing_demo.py`:

```python
if __name__ == "__main__":
    print(hash((1, 2)) == hash((1, 2)))
    try:
        lookup = {[1, 2]: "list key"}
    except TypeError as err:
        print("error:", err)
    seats = {(1, 2): "seat A1"}
    print(seats[(1, 2)])
    print({1, 1.0, True})
```

Starter code for `tests/test_inventory.py`:

```python
import unittest

from inventory import count_stock, group_by_category, merge_stock, price_of, unique_skus


class InventoryTests(unittest.TestCase):
    def test_count_stock(self):
        self.assertEqual(count_stock(["A1", "B2", "A1"]), {"A1": 2, "B2": 1})

    def test_count_empty(self):
        self.assertEqual(count_stock([]), {})

    def test_merge_does_not_mutate_inputs(self):
        a = {"A1": 3}
        b = {"A1": 1, "D4": 2}
        merged = merge_stock(a, b)
        self.assertEqual(merged, {"A1": 4, "D4": 2})
        self.assertEqual(a, {"A1": 3})
        self.assertEqual(b, {"A1": 1, "D4": 2})

    def test_unique_skus(self):
        self.assertEqual(unique_skus(["A1", "A1", "B2"]), {"A1", "B2"})

    def test_price_default(self):
        self.assertEqual(price_of({"A1": 2.5}, "Z9"), 0.0)
        self.assertEqual(price_of({"A1": 2.5}, "A1"), 2.5)

    def test_group_by_category(self):
        groups = group_by_category({"A1": "tea", "B2": "cake", "C3": "tea"})
        self.assertEqual(groups, {"tea": ["A1", "C3"], "cake": ["B2"]})

    def test_unhashable_key_raises(self):
        with self.assertRaises(TypeError):
            {[1, 2]: "bad"}

    def test_set_difference(self):
        scanned = {"A1", "B2", "C3"}
        expected = {"B2", "C3", "D4"}
        self.assertEqual(expected - scanned, {"D4"})


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python inventory.py` and paste the output into `docs/findings.md`. Explain in one sentence why the group for `"tea"` lists `A1` before `C3`.
3. Run `python hashing_demo.py` and paste the output into `docs/findings.md`. Explain why `{1, 1.0, True}` has one element.
4. Run `python -m unittest` and make every test pass.
5. Add a function `low_stock(stock: dict[str, int], threshold: int) -> set[str]` that returns the codes with quantity below the threshold. Add a test.
6. Write a benchmark in `docs/timing.py` that compares a list membership check with a set membership check on 20,000 codes, in a loop of 20,000 lookups. Use `time.perf_counter`, record both times in `docs/findings.md`, and explain the difference in one paragraph.
7. Run `python -m mypy inventory.py hashing_demo.py tests/test_inventory.py` and fix every issue.
8. Commit with the message "Add inventory counter with dict and set tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least nine tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs of both scripts, the timing results, and the paragraph explaining the difference
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `docs/timing.py`, `README.md` (with run commands), and `.gitignore`.

The project is small because the lesson is about choosing the right structure. A reviewer should be able to see that membership checks use sets, that merges return new dictionaries, and that the timing claim is backed by a measurement you ran.

---

## Check your understanding

1. A developer uses a dictionary with a list as a key. Explain the error, and describe one change that fixes it without losing the data's meaning.
2. Why does `d[key]` in a loop over a large dictionary stay fast even when the dictionary has a million entries, while `key in some_list` does not? Answer in terms of how each structure finds the item.
3. A cache dictionary uses a mutable object as a key. The object is changed after insertion, and later lookups return `KeyError`. Explain what happened in terms of the hash.
4. A teammate writes `seen = {}` and then calls `seen.add(item)`. Predict the error and explain how to fix it.
5. Given two sets of user IDs, which operation would you use to find users who are in the first set but not the second, and why is the order of the items irrelevant?
6. Why is a dictionary comprehension that calls `list.count` on every key quadratic, and what is a single-pass alternative?

### Answers

1. A list is mutable, so its `__hash__` is `None`, and Python raises `TypeError: unhashable type: 'list'`. Convert the list to a tuple, which keeps the same contents and is hashable as long as its items are hashable.
2. The dictionary computes the hash of the key and jumps to the slot, so the cost does not depend on the number of entries on average. A list must compare the key with each item in turn, so the cost grows with the length.
3. The entry was stored in the slot chosen by the hash at insertion time. After the change, the hash is different, so the lookup checks a different slot and does not find the entry. The rule is that keys must be immutable, or at least their hash must not change while they are in the dictionary.
4. The `seen = {}` line creates a dictionary, and dictionaries have no `add` method, so the call raises `AttributeError`. Use `seen = set()` for a set, or use `seen[item] = True` if the structure is a dictionary.
5. The set difference `first - second` returns the items in the first set that are not in the second. Order is irrelevant because sets have no order, and the operation depends only on membership.
6. For each of the `n` keys, `list.count` scans the whole list of `n` items, which makes the total about `n` squared. A single pass with `counts[sku] = counts.get(sku, 0) + 1` in a loop, or with `collections.Counter`, does the same work in linear time.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Mapping Types (dict)", and "Set Types (set, frozenset)", at docs.python.org/3/library/stdtypes.html
- Python documentation, "Data Structures", at docs.python.org/3/tutorial/datastructures.html
- Python documentation, "Hashing" and "Hash randomization" notes in the reference for `hash`, at docs.python.org/3/library/functions.html#hash
- [Wikipedia: Hash table](https://en.wikipedia.org/wiki/Hash_table)
- [Wikipedia: Set (abstract data type)](https://en.wikipedia.org/wiki/Set_(abstract_data_type))
- [Wikipedia: Associative array](https://en.wikipedia.org/wiki/Associative_array)

*Tutorials*

- GeeksforGeeks, "Python Dictionary" and "Python Sets" tutorials, at geeksforgeeks.org
- W3Schools, "Python Dictionaries" and "Python Sets" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on dictionaries and sets.

*Practice*

- Python Tutor at pythontutor.com. Run the dictionary merge example and the set operation example, and watch how each structure changes at every step.

---

## Tomorrow

Day 15 turns to control flow: the `if`, `elif` and `else` branches you have been reading in every example, the `match` statement for structured patterns, `for` and `while` loops, `range`, `break` and `continue`, the `else` clause on loops, and comprehensions with conditions. You will use the dictionaries and sets from today as the data that loops walk through.

**Next:** [Day 15: Control Flow](Software-Day-15-Control-Flow.md)

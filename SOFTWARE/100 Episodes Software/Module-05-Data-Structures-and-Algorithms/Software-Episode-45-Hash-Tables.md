# Day 45: Hash Tables

*100 Days of Software Engineering, Module 5: Data Structures and Algorithms*

**Time:** about 3 hours
**Level:** intermediate. You should have finished Day 44 and be comfortable with classes, lists of tuples and `unittest`. This day is for anyone who uses `dict` and `set` every day and wants to know what happens underneath, and why a lookup is sometimes slow.

**By the end of this day, you will have:**

- A working chained hash table with `put`, `get`, `remove`, membership tests and resizing
- A demonstration of collisions that you caused on purpose, and a count of how they change lookup cost
- The ability to explain the load factor and why resizing keeps lookups near O(1)
- A clear view of which hash functions are safe to use, and which Python behaviors can surprise you
- A word counter built on your hash table, tested against `collections.Counter`, committed with CI

**What you need:**

- Python 3.12 or newer (the code was checked with Python 3.14)
- Git 2.23 or newer, and the repository from Day 44
- A terminal (Git Bash, PowerShell or Ubuntu on WSL 2)

**Prerequisites:**

- Day 41: Complexity Analysis, especially average and worst case
- Day 43: Linked Lists, because each bucket is a small list
- Module 3: Python `dict`, `set` and tuples

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Short history](#short-history)
- [Glossary](#glossary)
- [Concept deep dives](#concept-deep-dives)
  - [Hash functions](#hash-functions)
  - [Collisions](#collisions)
  - [Maps and sets](#maps-and-sets)
  - [Complexity tradeoffs](#complexity-tradeoffs)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a word counter on a hand-built hash table](#project-a-word-counter-on-a-hand-built-hash-table)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Suppose you have a million user records and you need to find one by its ID. A list means checking records one by one, which is O(n) per search. Sorting the list and using binary search would make each search O(log n), but a new user means re-sorting or inserting in the right place. A hash table offers something faster on average: the key itself tells you where to look. The lookup computes a number from the key, uses that number to choose a bucket, and checks only the few items in that bucket.

That idea sits under Python's `dict` and `set`, under the symbol tables in compilers, under the caches in web servers, and under the indexes that databases build for joins. When it works, it makes lookups, inserts and deletes O(1) on average. When it fails, a table with many keys in one bucket behaves like a list and every lookup slows down. Understanding the failure is as important as using the success, which is why this day builds a table by hand and forces collisions on purpose.

The module outline lists four items: hash functions, collisions, maps and sets, and complexity tradeoffs. The project turns them into a word counter, a simple and practical application that every text tool needs.

---

## Short history

The hash table idea grew out of practical needs in the 1950s and 1960s. The table below is a rough guide. The Wikipedia article on hash tables gives fuller detail, and the dates are approximate.

| When | What happened | Why it mattered |
|------|---------------|-----------------|
| 1953 | Hans Peter Luhn at IBM proposed hashing with chaining for information retrieval | Chaining, the method in this day, is one of the earliest collision strategies |
| 1950s to 1960s | Open addressing methods, such as linear probing, were developed | Stored all entries in one array and avoided pointers |
| 1990s and 2000s | Hash-based maps and sets became standard library types in mainstream languages, including Python's `dict` and `set` | Programmers stopped writing their own tables for common work |
| Today | Used in databases, compilers, caches and the `dict` and `set` types | The basis of most key-based lookup |

---

## Glossary

- **Hash function:** a function that turns a key into an integer, which is then reduced to a bucket index.
- **Bucket:** a slot in the table's array. In a chained table, each bucket holds a small list of entries.
- **Collision:** two different keys that map to the same bucket.
- **Chaining:** a collision strategy where each bucket holds a list of all entries that landed there.
- **Open addressing:** a collision strategy where all entries live in the array, and a collision searches for another free slot.
- **Load factor:** the number of entries divided by the number of buckets. A chained table with load factor 2 has about two entries per bucket on average.
- **Resize:** creating a bigger array and moving all entries into it, usually when the load factor passes a threshold.
- **Map:** a collection of key-value pairs, such as Python's `dict`.
- **Set:** a collection of unique keys, such as Python's `set`.
- **Hashable:** a type whose hash value stays the same for its whole lifetime. Lists are not hashable, because they can change.

---

## Concept deep dives

Each concept below follows the same flow: an everyday analogy, the first principles, a worked example, a common mistake and a real-world use.

### Hash functions

Imagine a coat check at a theater. You hand over your coat and get a numbered ticket. The number is computed from something stable about the coat, so the attendant can find it again. A hash function plays the same role. It takes a key, such as a string, and produces a number that is always the same for the same key.

From first principles, a hash table needs three things from its hash function. It must be deterministic, so the same key always gives the same number within one run. It should spread keys evenly across buckets, so no bucket fills up. And it must be fast, because it runs on every operation. Python's built-in `hash` gives you all three for common types. The bucket index is `hash(key) % number_of_buckets`, which is always between 0 and one less than the number of buckets, because Python's `%` returns a non-negative result for a positive divisor.

```python
print(hash(42) % 8)        # 2, because hash(42) is 42 and 42 % 8 is 2
print(hash("cat") % 8)     # some value from 0 to 7, which can change between runs
```

The integer example is predictable, because Python's hash of a small integer is the integer. The string example is not. Since Python 3.3, string hashes are randomized for each process unless `PYTHONHASHSEED` is set. That protects servers against attacks where the attacker crafts keys that collide, but it means you cannot save hash values to disk and expect them to match in the next run. Use `hashlib` if you need a stable hash for storage or for sharding across machines.

The limitation is that a poor hash function produces collisions even when the keys are different. A function that returns the length of a string sends every three-letter word to the same bucket. The fix is a good hash function, which for built-in types you do not write yourself.

The common mistake is to use a mutable object as a key. A list is not hashable, so `{[1, 2]: "a"}` raises `TypeError`. Tuples work if their contents are hashable. To spot the problem, look at what you are using as a key. If it can change after you insert it, the table cannot find it again.

Hash functions decide the performance of caches in web frameworks, where request keys are hashed to pick a cache shard, and of distributed storage systems that assign keys to servers.

### Collisions

Think of a parking lot with 8 spaces and a rule that each car parks in the space matching the last digit of its plate. Two cars ending in 3 must share space 3. If the lot is nearly empty, that sharing is rare. If most cars end in 3, every visit to space 3 means checking a long line of cars. Collisions work the same way: they are unavoidable in a finite table, and they only hurt when they pile up.

From first principles, a table with B buckets and N keys must have collisions when N is larger than B, by the pigeonhole principle. Even with fewer keys, collisions happen by chance. Chaining handles them by storing each colliding entry in the bucket's list. A lookup computes the bucket, then searches that list. If the average list length is the load factor alpha, the expected cost of an unsuccessful lookup is about 1 plus alpha, which is O(1) when alpha stays bounded.

The code below forces collisions by using integer keys that are multiples of the table size. Each key lands in bucket 0:

```python
from dsa.hash_table import HashTable

table = HashTable(capacity=8)
for key in [0, 8, 16, 24]:
    table.put(key, str(key))

print(table.bucket_lengths())   # [4, 0, 0, 0, 0, 0, 0, 0]
print(table.get(16))            # "16", found by searching a list of 4
```

Every key still works, because the bucket keeps a list of all of them. The cost is that a lookup in bucket 0 checks four entries instead of one. With n keys all in one bucket, the lookup is O(n), which is the same as a plain list. That is the worst case of a hash table, and it is what a hash flooding attack tries to produce.

The limitation of chaining is memory and pointer chasing. Each bucket holds a separate list object, which costs memory, and walking the lists jumps around memory. Open addressing avoids that by storing entries in one array, but it needs more care when deleting.

The common mistake is to think collisions are rare because the table is "big enough". The real test is the load factor and the distribution of keys. To spot a problem, check the longest bucket. If it is much longer than the average, your hash function or your key pattern is at fault. This is the reason the project tests a deliberately bad hash function.

Collision handling appears in network routers, where packet flow tables map addresses to queues, and in compilers, where symbol tables use chaining to keep track of identifiers with the same hash.

### Maps and sets

Picture a filing cabinet with labeled drawers. A map is a drawer that holds a card with a name and a value on it. A set is a drawer that only holds names, with no value attached. In both cases, finding the drawer is the hash function's job, and the drawer's contents are the table's job.

From first principles, a map is a hash table where each entry stores a key and a value, and a set is a hash table where each entry stores only the key. Python's `dict` and `set` both use open addressing with a compact layout, which is faster in practice than the chaining in this day, but they have the same interface and the same average complexity. Their methods, such as `get`, `in` and `add`, map directly to the table operations you will write.

```python
ages = {"Ada": 36, "Linus": 54}
print("Ada" in ages)        # True, average O(1)
print(ages.get("Grace"))    # None, because the key is missing

seen: set[int] = set()
for n in [3, 1, 3, 2]:
    seen.add(n)
print(sorted(seen))         # [1, 2, 3], duplicates removed
```

The limitation of maps and sets is that they are unordered by key, in the sense that you cannot ask for the "next" key efficiently. Python's `dict` keeps insertion order, which is a different thing. If you need sorted keys, you need a tree, which Day 46 covers.

The common mistake is to use a list for membership tests in a loop, such as `if item in my_list` inside a loop over another list. That makes the whole program O(n squared). Convert the list to a set once before the loop. To spot it, look for `in` checks on a list inside a `for` loop.

Maps and sets appear in almost every program. Deduplication in data pipelines, joins in query engines and lookups of session tokens in web servers all depend on them.

### Complexity tradeoffs

Think of a bookshelf. A small shelf is cheap and easy to search, but it fills up quickly. A large shelf takes more room and more time to build, but it leaves space for new books. A hash table makes the same trade between memory and speed. A low load factor means short bucket lists and fast lookups, but unused space. A high load factor saves memory and slows lookups.

The first principle is the load factor. If the table has B buckets and N entries, the load factor alpha is N divided by B. Average lookup cost is roughly 1 plus alpha, assuming good hashing. To keep alpha bounded, the table resizes when alpha passes a threshold, such as 0.75. Each resize doubles the number of buckets and re-inserts every entry, which costs O(N) for that one operation. The resizes form a geometric series, so the amortized cost of an insert stays O(1), as Day 41 showed for dynamic arrays.

```python
def put(self, key, value) -> None:
    bucket = self._buckets[self._index(key)]
    for i, (k, _) in enumerate(bucket):
        if k == key:
            bucket[i] = (key, value)
            return
    bucket.append((key, value))
    self._size += 1
    if self._size / len(self._buckets) > self.MAX_LOAD:
        self._resize(len(self._buckets) * 2)
```

The limitation is that resizing is an expensive single step. In a latency-sensitive system, a resize can cause a visible pause, and some systems use incremental resizing, which moves a few entries on each operation, to avoid it. Another limitation is that a hash table uses more memory than a sorted array with binary search, which uses almost none beyond the data.

The common mistake is to read O(1) as "instant" and ignore the constant factor. A hash lookup in Python calls a hash function, computes an index and compares keys, which costs more than an array index. For small collections, a list can be faster than a set. Measure before you choose. The project asks you to measure the cost of a lookup at two sizes.

The tradeoffs matter in production systems where memory is the constraint, such as embedded devices and in-memory caches. Engineers choose a load factor and a hash function based on how the table will be used.

---

## Hands-on exercises

Work in the repository from Day 44. Keep the hash table in `dsa/hash_table.py` and the word counter in `dsa/word_count.py`.

Exercise 1 traces a bucket index. For a table with 8 buckets, write down the bucket index for the integer keys 3, 11, 19 and 27 on paper, using `hash(key) % 8`. Then run the same expressions in the shell and compare. Explain why all four land in the same bucket.

Done when your paper answers match the shell output and your explanation mentions the remainder after dividing by 8.

Exercise 2 forces and measures collisions. Write a script that inserts the 16 keys 0, 8, 16, ... up to 120 into a `HashTable(capacity=8)`, one at a time. Print `bucket_lengths()` after every four inserts, and print `capacity` at the same points. Record the results in `docs/hash-notes.md`.

Done when your notes show the longest bucket and the capacity after each group of four, and a sentence explaining why the table resizes but the longest bucket still holds several keys. Work out the answer first: once the capacity is 32, every key is a multiple of 8, so only the buckets 0, 8, 16 and 24 are ever used.

Exercise 3 compares load factors. For a table that starts with 8 buckets and receives 1000 distinct string keys, print the capacity and load factor after the insert. Confirm the load factor stays at or below 0.75.

Done when the printed load factor is at most 0.75 and the capacity is a power of two.

Exercise 4 tests against `dict`. Add a randomized test in `tests/test_hash_table.py` that performs 500 random `put`, `get`, `remove` and `in` operations on a `HashTable` and a Python `dict` in parallel, using `random.Random(5)`. Assert that results match after each operation.

Done when the test passes and you can name one bug the test would catch if `remove` forgot to decrement the size.

Exercise 5 measures membership. Time `key in my_list` and `key in my_set` for lists of 1000 and 10000 items, with a key that is not present. Record the ratio in your notes. Explain why the list time grows with size.

Done when your notes show both timings at both sizes and the ratio, with one sentence on the growth.

---

## Project: a word counter on a hand-built hash table

The project is a word counter. It reads text, splits it into lowercase words, and counts each word in a `HashTable` you built. The tests compare the result with `collections.Counter`, which is the standard library's version, so you know your table is correct. The project also includes a deliberately bad hash function, used only in tests, to show what a poor distribution does to the bucket lengths.

Suggested repo name: `100-days-software-day-45-hash-tables`. Copy the Day 44 repository into a new folder and rename it.

The repo layout:

```
100-days-software-day-45-hash-tables/
├── .github/
│   └── workflows/
│       └── ci.yml
├── benchmark.py
├── count_words.py
├── dsa/
│   ├── __init__.py
│   ├── arrays.py
│   ├── bench.py
│   ├── browser_history.py
│   ├── complexity.py
│   ├── hash_table.py
│   ├── linked_list.py
│   ├── stacks_queues.py
│   └── word_count.py
├── docs/
│   ├── hash-notes.md
│   ├── linked-list-notes.md
│   ├── stack-queue-notes.md
│   └── window-notes.md
├── tests/
│   ├── __init__.py
│   ├── test_arrays.py
│   ├── test_bench.py
│   ├── test_browser_history.py
│   ├── test_complexity.py
│   ├── test_hash_table.py
│   ├── test_linked_list.py
│   ├── test_print_queue.py
│   ├── test_stacks_queues.py
│   └── test_word_count.py
└── README.md
```

Starter code for `dsa/hash_table.py`:

```python
from collections.abc import Callable, Iterator
from typing import Any

_MISSING = object()


class HashTable:
    """A chained hash table. Each bucket is a list of (key, value) pairs."""

    MAX_LOAD = 0.75

    def __init__(self, capacity: int = 8, hash_func: Callable[[Any], int] = hash) -> None:
        if capacity < 1:
            raise ValueError("capacity must be at least 1")
        self._hash = hash_func
        self._buckets: list[list[tuple[Any, Any]]] = [[] for _ in range(capacity)]
        self._size = 0

    def __len__(self) -> int:
        return self._size

    def __contains__(self, key) -> bool:
        return self.get(key, _MISSING) is not _MISSING

    @property
    def capacity(self) -> int:
        return len(self._buckets)

    @property
    def load_factor(self) -> float:
        return self._size / len(self._buckets)

    def bucket_lengths(self) -> list[int]:
        return [len(bucket) for bucket in self._buckets]

    def _index(self, key) -> int:
        return self._hash(key) % len(self._buckets)

    def put(self, key, value) -> None:
        bucket = self._buckets[self._index(key)]
        for i, (k, _) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)
                return
        bucket.append((key, value))
        self._size += 1
        if self.load_factor > self.MAX_LOAD:
            self._resize(len(self._buckets) * 2)

    def get(self, key, default=None):
        for k, v in self._buckets[self._index(key)]:
            if k == key:
                return v
        return default

    def remove(self, key) -> bool:
        bucket = self._buckets[self._index(key)]
        for i, (k, _) in enumerate(bucket):
            if k == key:
                del bucket[i]
                self._size -= 1
                return True
        return False

    def items(self) -> Iterator[tuple[Any, Any]]:
        for bucket in self._buckets:
            yield from bucket

    def _resize(self, new_capacity: int) -> None:
        old_items = list(self.items())
        self._buckets = [[] for _ in range(new_capacity)]
        self._size = 0
        for key, value in old_items:
            self._buckets[self._index(key)].append((key, value))
            self._size += 1
```

Starter code for `dsa/word_count.py`:

```python
import re

from dsa.hash_table import HashTable

WORD = re.compile(r"[a-z']+")


def count_words(text: str) -> HashTable:
    """Count lowercase words. Apostrophes stay inside a word, so don't is one word."""
    table = HashTable()
    for word in WORD.findall(text.lower()):
        table.put(word, table.get(word, 0) + 1)
    return table
```

Starter code for `count_words.py`:

```python
import sys
from pathlib import Path

from dsa.word_count import count_words

if __name__ == "__main__":
    text = Path(sys.argv[1]).read_text(encoding="utf-8")
    counts = count_words(text)
    ranked = sorted(counts.items(), key=lambda pair: (-pair[1], pair[0]))
    for word, n in ranked[:10]:
        print(f"{n:>5}  {word}")
```

Running `python count_words.py some-file.txt` prints the ten most common words. Sorting here is O(n log n) and is fine for a report. Day 47 shows a faster way to get only the top k.

Starter code for `tests/test_hash_table.py`:

```python
import random
import unittest

from dsa.hash_table import HashTable


class HashTableTests(unittest.TestCase):
    def test_put_get_and_overwrite(self):
        table = HashTable()
        table.put("a", 1)
        table.put("a", 2)
        self.assertEqual(table.get("a"), 2)
        self.assertEqual(len(table), 1)

    def test_missing_key_returns_default(self):
        table = HashTable()
        self.assertIsNone(table.get("nope"))
        self.assertEqual(table.get("nope", 0), 0)
        self.assertNotIn("nope", table)

    def test_remove(self):
        table = HashTable()
        table.put("x", 10)
        self.assertTrue(table.remove("x"))
        self.assertFalse(table.remove("x"))
        self.assertEqual(len(table), 0)

    def test_forced_collisions_still_work(self):
        table = HashTable(capacity=8)
        for key in [0, 8, 16, 24]:
            table.put(key, str(key))
        self.assertEqual(table.bucket_lengths()[0], 4)
        self.assertEqual(table.get(16), "16")

    def test_bad_hash_puts_everything_in_one_bucket(self):
        table = HashTable(capacity=8, hash_func=lambda key: 0)
        for i in range(5):
            table.put(i, i * i)
        self.assertEqual(max(table.bucket_lengths()), 5)
        self.assertEqual([table.get(i) for i in range(5)], [0, 1, 4, 9, 16])

    def test_resize_keeps_load_under_threshold(self):
        table = HashTable(capacity=8)
        for i in range(100):
            table.put(f"key{i}", i)
            self.assertLessEqual(table.load_factor, HashTable.MAX_LOAD)
        self.assertEqual(len(table), 100)
        self.assertEqual(table.get("key42"), 42)

    def test_matches_dict_on_random_operations(self):
        rng = random.Random(5)
        table = HashTable()
        mirror: dict[int, int] = {}
        for _ in range(500):
            op = rng.choice(["put", "get", "remove", "in"])
            key = rng.randint(0, 30)
            if op == "put":
                value = rng.randint(0, 99)
                table.put(key, value)
                mirror[key] = value
            elif op == "get":
                self.assertEqual(table.get(key), mirror.get(key))
            elif op == "remove":
                self.assertEqual(table.remove(key), key in mirror)
                mirror.pop(key, None)
            else:
                self.assertEqual(key in table, key in mirror)
            self.assertEqual(len(table), len(mirror))


if __name__ == "__main__":
    unittest.main()
```

Starter code for `tests/test_word_count.py`:

```python
import unittest
from collections import Counter

from dsa.word_count import WORD, count_words


class WordCountTests(unittest.TestCase):
    def test_matches_counter(self):
        text = "The cat and the hat. The cat's hat, isn't it? the end"
        expected = Counter(WORD.findall(text.lower()))
        self.assertEqual(dict(count_words(text).items()), dict(expected))

    def test_empty_text(self):
        self.assertEqual(len(count_words("")), 0)

    def test_apostrophes_stay_inside_words(self):
        counts = dict(count_words("don't DON'T dont").items())
        self.assertEqual(counts, {"don't": 2, "dont": 1})


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Copy the Day 44 repository, add the files above, and run `python -m unittest`. All tests should pass.
2. Run `python count_words.py` on a text file of your choice, such as a README or a public-domain book from Project Gutenberg. Record the top ten words in `docs/hash-notes.md`.
3. Add a `set`-based function `unique_words(text: str) -> int` to `dsa/word_count.py` that counts distinct words. Add a test that compares it with `len(Counter(...))` on a sentence you write.
4. In `docs/hash-notes.md`, record the load factor after inserting 1000 keys, and the longest bucket after inserting the keys 0 to 999 into a table with a hash function that returns the key modulo 4. Explain the second number.
5. Commit with the message "Add hash table and word counter", push to GitHub, and check that the Actions run passes.

Done when:

- `python -m unittest` passes locally and in GitHub Actions
- The randomized test matches `dict` on 500 operations
- `python count_words.py` prints ten lines for a file with at least ten distinct words
- `docs/hash-notes.md` explains one collision and one load factor measurement with numbers
- The latest commit on `main` has a green check

Evidence to commit: `dsa/hash_table.py`, `dsa/word_count.py`, `count_words.py`, the test files, `docs/hash-notes.md` and the Actions run. The randomized comparison with `dict` is the main proof that the table works.

This project is light on new code and heavy on measurement. A hash table you can explain, measure and test is more useful in a job than one you only know how to call.

---

## Check your understanding

1. Why does `HashTable.put` replace the value when the key already exists, instead of adding a second pair with the same key?
2. A table has 16 buckets and 12 entries. What is its load factor, and what average bucket length should you expect with a good hash function?
3. Explain why resizing a table is O(N) in one operation, but inserts are still O(1) amortized. Use the doubling argument from Day 41.
4. A teammate writes `if word in word_list:` inside a loop over a long text. Describe the cost and the fix.
5. Why does Python randomize string hashes between runs, and what breaks if you save string hash values to a file and read them back in the next run?
6. A hash table with a bad hash function still returns correct results. So why does the bad function matter in production?

### Answers

1. A key is unique in a map. Adding a second pair with the same key would make `get` ambiguous and would make `len` count the key twice. Replacing the value keeps the invariant that each key appears once.
2. The load factor is 12 divided by 16, which is 0.75. With a good hash function the average bucket holds about 0.75 entries, so most lookups check at most one or two items.
3. A resize copies all N entries once, which is O(N) for that operation. The resizes happen when the table doubles, so the total copy work across n inserts is a geometric series that sums to less than 2n. Dividing the total by n gives an amortized O(1) per insert.
4. Each `in` check on a list scans the list, which is O(m) for a list of size m. Inside a loop over n words, the total is O(n times m). Converting `word_list` to a set once before the loop makes each check average O(1), which brings the total down to about O(n plus m).
5. The randomization protects against attackers who craft keys that all collide, which could slow down a web server to a crawl. A saved hash value from one run will not match the hash of the same string in another run. Use a stable hash such as one from `hashlib` if the values must persist.
6. A bad hash function turns a hash table into a list in disguise. Results stay correct, but each lookup can cost O(n), which turns a fast endpoint into a slow one under load. An attacker who controls the keys can exploit the same weakness to cause a denial of service.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Hash table](https://en.wikipedia.org/wiki/Hash_table)
- [Wikipedia: Hash function](https://en.wikipedia.org/wiki/Hash_function)
- [Wikipedia: Associative array](https://en.wikipedia.org/wiki/Associative_array), which covers maps and their interfaces
- The Python documentation on `dict` and `set` in the tutorial and the library reference, at docs.python.org/3/library/stdtypes.html
- The Python documentation on `hash()` and `PYTHONHASHSEED`, at docs.python.org/3/reference/datamodel.html and docs.python.org/3/using/cmdline.html

*Tutorials*

- GeeksforGeeks, "Hashing Data Structure" and "Hash Table" tutorials. Search the site for each title.
- W3Schools, "Python Dictionaries" tutorial. Search the W3Schools Python section for the title and follow the examples on membership.

*Watch*

- MIT OpenCourseWare, 6.006 Introduction to Algorithms, the lectures on hashing, including chaining and universal hashing. Search the course site for "hashing".
- William Fiset on YouTube, the playlist on hash tables. Search YouTube for "William Fiset hash table".

*Practice*

- Python Tutor at pythontutor.com. Paste the `HashTable` class and step through `put` on the forced-collision example, watching the bucket list grow.
- LeetCode, the "Hash Table" study plan at leetcode.com. Start with "Two Sum" and "Contains Duplicate", then compare your solution with a sorted-array version.

---

## Tomorrow

Day 46 covers trees and binary search trees. A hash table gives fast lookups but no order, so you cannot ask for the next key or a range of keys. A binary search tree keeps keys sorted, which makes search, insert and delete O(log n) when the tree is balanced, and it introduces recursion, which Day 50 examines in detail. You will build a BST, traverse it in three orders and see what happens to its height when the input is already sorted.

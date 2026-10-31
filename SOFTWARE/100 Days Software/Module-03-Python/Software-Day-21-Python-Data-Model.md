# Day 21: Python Data Model

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate to advanced. You can write classes with methods and validate their input from Day 19 and Day 20. This is for anyone who has wondered how `len(x)`, `for item in x` and `with open(...)` work on their own objects, and who wants their classes to fit into the language rather than fight it.

**By the end of this day, you will have:**

- A working model of the data model: Python calls special methods, written with double underscores, when you use an operator or a built-in on your object
- The ability to write `__repr__` and `__str__` with the right audience in mind, and to know which one `print` and the REPL use
- The ability to write `__eq__` and `__hash__` together, and to explain why defining `__eq__` alone can make a class unhashable
- The ability to make a class iterable with `__iter__`, and to separate an iterable from its iterator
- Fluency with generator functions that use `yield`, and an understanding of why they produce values lazily
- The ability to write context managers with `__enter__` and `__exit__`, and with the `contextlib` decorator
- A money-and-iteration repo with tests that check each protocol

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 10: Objects, Variables and Types (identity, `is`, `==`, hashing)
- Day 14: Dictionaries and Sets (hashability)
- Day 19 and Day 20: OOP I and II (classes, methods, inheritance, dataclasses)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Dunder methods](#dunder-methods)
  - [Repr and str](#repr-and-str)
  - [Iterables](#iterables)
  - [Iterators](#iterators)
  - [Generators](#generators)
  - [Context managers](#context-managers)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a money and iteration lab](#project-a-money-and-iteration-lab)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Python's built-in types feel natural to use because they follow a shared set of rules. `len` works on lists, strings and dictionaries. `for` works on anything that can produce items. `with` works on files, locks and database connections. These rules are ordinary methods with names that begin and end with double underscores, and any class can define them. The collection of these rules is called the Python data model, and learning it lets you make your own objects behave the way Python programmers expect.

This matters on the job because the code you read in real projects is full of these protocols. A class that defines `__eq__` and `__hash__` correctly can be used as a dictionary key. A generator that yields rows keeps memory use flat when a file is large. A context manager guarantees that a lock is released even when an exception is raised. Each of these is a small contract, and each one is easy to get subtly wrong.

The official reference is the Python language reference chapter "Data model" at docs.python.org/3/reference/datamodel.html. It is long, and it is the authority for every method named here. Read the sections on emulating container types, iterators, generators and context managers, and keep it open while you work through the exercises.

---

## Glossary

- **Data model:** the set of protocols, expressed as special methods, that define how Python objects behave with operators, built-ins and syntax.
- **Dunder method:** a special method whose name begins and ends with double underscores, such as `__len__` or `__iter__`. "Dunder" is short for double underscore.
- **Protocol:** a set of methods that an object must provide to take part in a language feature, such as the iteration protocol.
- **`__repr__`:** the method that returns a developer-facing representation of an object, ideally one that could recreate it.
- **`__str__`:** the method that returns a user-facing string. `print` uses it when it exists.
- **Iterable:** an object with `__iter__`, which can produce an iterator.
- **Iterator:** an object with `__next__`, which returns the next item or raises `StopIteration`. Iterators also have `__iter__`, which returns themselves.
- **Generator:** an iterator created by a function that contains `yield`, or by a generator expression.
- **yield:** a statement that pauses a generator, returns a value to the caller, and resumes on the next request.
- **Lazy evaluation:** computing values only when they are requested.
- **Context manager:** an object with `__enter__` and `__exit__` methods, used with `with`.
- **contextlib:** the standard library module that provides `contextmanager`, which turns a generator into a context manager.

---

## Concepts

The six concepts follow the protocols in the order most programs use them: the naming rule, the two text forms of an object, the iteration pair, generators as a way to write iterators, and the context manager.

### Dunder methods

A dunder method is a hook. Think of a restaurant kitchen where the waiter writes an order in a standard way, and the kitchen knows how to act on each standard order. When a customer says "add a side", the kitchen runs its side-dish routine. When you write `a + b`, Python looks for a routine named `__add__` on `a`, and when you write `len(a)`, it looks for `__len__`.

From first principles, the interpreter does not call methods named `__add__` or `__len__` directly by name in the way your code does. It looks them up on the type of the object, not on the instance, and calls them with the instance as `self`. This is why `len(obj)` works even if the instance has no `__len__` attribute set in its dictionary. If the method is missing, Python raises `TypeError` with a clear message. Dunder methods are the mechanism behind operators, built-in functions and syntax such as `for`, `with` and subscripting.

```python
class Box:
    def __init__(self, items: list[str]) -> None:
        self.items = items

    def __len__(self) -> int:
        return len(self.items)

    def __getitem__(self, index: int) -> str:
        return self.items[index]


box = Box(["pen", "ink"])
print(len(box))      # 2, Python calls __len__
print(box[1])        # ink, Python calls __getitem__
print("pen" in box)  # True, membership falls back to iteration through __getitem__
```

The `in` operator worked without defining `__contains__`, because Python falls back to iterating with `__getitem__` when the other protocol is missing. That fallback is a design feature of the model.

The common mistake is calling dunder methods directly, such as `box.__len__()`, in application code. It works, but it bypasses the rules that the built-in functions follow, and it reads as a sign of confusion. Use `len(box)`. The second mistake is inventing a dunder name that the language does not know, such as `__size__`, which Python never calls. In the job market, a reviewer asks whether a class defines the protocol it claims, because a half-implemented protocol causes surprising errors in library code.

### Repr and str

An object has two text forms. Think of a passport and a name badge. The passport is the full record, written for officials, with every detail. The badge is a friendly short label, written for people in the room. `__repr__` is the passport. `__str__` is the badge.

From first principles, `repr(obj)` calls `__repr__` and is used by the REPL, by the debugger and inside containers such as lists. `str(obj)` calls `__str__` and is used by `print` and by f-strings with the default conversion. If a class defines only `__repr__`, `str` falls back to it. The convention for `__repr__` is that it should look like valid Python that recreates the object, when that is practical. For a money class, that means showing the amount and currency as arguments.

```python
from decimal import Decimal


class Money:
    def __init__(self, amount: str, currency: str = "USD") -> None:
        self.amount = Decimal(amount)
        self.currency = currency

    def __repr__(self) -> str:
        return f"Money({str(self.amount)!r}, {self.currency!r})"

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"


price = Money("2.5")
print(price)            # 2.50 USD, uses __str__
print(repr(price))      # Money('2.5', 'USD'), uses __repr__
print([price])          # [Money('2.5', 'USD')], containers use __repr__
```

The list output shows the repr, which is why a good `__repr__` matters when debugging a collection of objects.

The common mistake is writing a `__repr__` that hides information, such as returning only the class name. Then a list of objects prints as a list of identical strings, and the bug is invisible. The second mistake is putting user-facing formatting in `__repr__`, which makes logs and error messages hard to read. In the job market, the `dataclass` decorator from Day 20 generates a `__repr__` for this reason, and reviewers check that custom representations keep the important fields.

### Iterables

An iterable is anything you can loop over. Think of a deck of cards on a table. You can walk along the deck, and the deck itself is not the walker. If you want to deal the cards, you hire a dealer who holds a position in the deck. Each time you ask the deck for a dealer, you get a new dealer starting at the top.

From first principles, `for item in obj` calls `iter(obj)`, which calls `obj.__iter__()`. That method must return an iterator. A list is an iterable, and so is a string, a dictionary, a file and a range. Your class becomes iterable by defining `__iter__`. The class itself does not need to be an iterator, and keeping them separate lets several loops run over the same object independently.

```python
class Playlist:
    def __init__(self, tracks: list[str]) -> None:
        self.tracks = tracks

    def __iter__(self):
        return iter(self.tracks)


playlist = Playlist(["intro", "theme", "outro"])
for track in playlist:
    print(track)
print(list(playlist))   # ['intro', 'theme', 'outro'], a fresh iterator each time
```

The playlist can be iterated many times because each `for` loop asks for a new iterator. The iterator belongs to the loop that created it.

The common mistake is returning `self` from `__iter__` when the object also has `__next__` and mutable state. The first loop consumes the state, and the second loop sees nothing. The fix is to return a new iterator each time, as the next section shows. A second mistake is changing the underlying list while a loop is running over it, which gives unpredictable results. In the job market, iterables that are safe to reuse are a common review point for data-loading code.

### Iterators

An iterator is a one-way cursor. Think of a conveyor belt at an airport. Each bag comes off once, in order, and when the belt is empty, the belt stops. You cannot rewind it. An iterator keeps a position, returns the next item when asked, and raises `StopIteration` at the end.

From first principles, an iterator implements `__next__`, which returns the next value or raises `StopIteration`. The `for` loop catches that exception and ends the loop. By convention, an iterator also implements `__iter__` and returns itself, so that it can be used wherever an iterable is expected. Once exhausted, an iterator stays exhausted, which is a rule that surprises people who expect to restart it.

```python
class CountdownIterator:
    def __init__(self, start: int) -> None:
        self.current = start

    def __iter__(self):
        return self

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value


class Countdown:
    def __init__(self, start: int) -> None:
        self.start = start

    def __iter__(self):
        return CountdownIterator(self.start)


print(list(Countdown(3)))   # [3, 2, 1]
print(list(Countdown(3)))   # [3, 2, 1], a new iterator each time
```

The `Countdown` is the iterable and `CountdownIterator` is the iterator. Splitting them means the countdown can be read many times. A single iterator stored on the countdown would be exhausted after the first call.

The common mistake is storing the iterator's position on the iterable, so the second loop finds it empty. Another mistake is calling `next` on an iterator without a default and catching `StopIteration` by hand. Use `next(iterator, default)` when a default is correct, or use a `for` loop. In the job market, reading a large file through an iterator is a standard pattern, and understanding the one-way rule prevents bugs where a function consumes a file that another function still needs.

### Generators

A generator is an iterator written as a function. Think of a tap that fills a glass only when you open it. It produces water on demand, and when you close it, it stops. A function with `yield` works the same way: each `yield` produces one value and pauses, and the next request resumes the function where it stopped.

From first principles, calling a generator function returns a generator object without running the body. The body runs during `next()`, until the next `yield`. The local variables survive between calls because the generator keeps its frame. When the body returns, the generator raises `StopIteration`. A generator expression, written like a list comprehension with parentheses, creates a generator the same way. Generators use constant memory for one item at a time, which is why they suit large files and streams.

```python
from collections.abc import Iterator


def chunks(items: list[int], size: int) -> Iterator[list[int]]:
    if size <= 0:
        raise ValueError("size must be positive")
    for start in range(0, len(items), size):
        yield items[start:start + size]


batches = chunks([1, 2, 3, 4, 5], 2)
print(next(batches))    # [1, 2]
print(list(batches))    # [[3, 4], [5]], the rest of the same generator
```

The first `next` ran the function up to the first `yield`. The `list` call consumed the rest. The same laziness applies to errors: the `size` check in `chunks` runs on the first `next`, not on the call, so `chunks([1], 0)` by itself raises nothing until it is consumed. A generator can be used only once, so the second use of `batches` finds it empty.

The common mistake is expecting a generator to be reusable. Once consumed, it stays empty, and the next `for` loop does nothing and raises no error. Create a new generator for each pass, or store a list if you need several passes and the data fits in memory. The second mistake is putting a generator inside a function that is called many times without a clear reason. In the job market, data pipelines are built from generators, and the one-pass rule is a frequent source of bugs when a function consumes an iterator that another part still expects.

### Context managers

A context manager is a set-up and clean-up pair. Think of renting a meeting room. When you enter, the room is prepared, and when you leave, it is cleared, even if the meeting ends in an argument. The `with` statement guarantees that the leave step runs.

From first principles, `with expr as name:` calls `expr.__enter__()` and binds its return value to `name`. When the block ends, normally or through an exception, Python calls `expr.__exit__(exc_type, exc, traceback)`. If `__exit__` returns a true value, the exception is suppressed. If it returns `None` or a false value, the exception continues to propagate. The `contextlib.contextmanager` decorator lets you write the same thing as a generator with one `yield`, where the code before `yield` is the set-up and the code after is the clean-up, wrapped in `try` and `finally`.

```python
from collections.abc import Iterator
from contextlib import contextmanager
from time import perf_counter


class Timer:
    def __enter__(self) -> "Timer":
        self._start = perf_counter()
        self.elapsed = 0.0
        return self

    def __exit__(self, exc_type, exc, traceback) -> None:
        self.elapsed = perf_counter() - self._start


@contextmanager
def labelled(label: str) -> Iterator[None]:
    print(f"start {label}")
    try:
        yield
    finally:
        print(f"end {label}")


with labelled("report"):
    print("working")
```

The output is `start report`, `working`, then `end report`. The `finally` guarantees the last line even if the block raises an exception. The `Timer` class records elapsed time on exit, which is the same pattern for files and locks.

The common mistake is writing `__exit__` that catches an exception and returns `True` without a reason, which hides the error. Return `None` unless the context manager is designed to swallow a specific exception. The second mistake is doing clean-up outside the `with` block, which runs only when the code reaches that line, not when the block fails. In the job market, context managers guard locks, transactions and file handles, and reviewers check that any resource acquired inside a function is released through `with`.

---

## Hands-on exercises

Work in a folder called `data-model-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices dunder methods.

1. Create `dunders.py`. Define a `Box` class with `__len__` and `__getitem__`, and a list of three items inside it.
2. Call `len(box)`, `box[0]` and `"x" in box` and record the results.
3. Remove `__getitem__` and try `box[0]` in a `try` block. Record the error and write one sentence on what it means.

Done when you can explain which method each built-in called.

Exercise 2 practices repr and str.

1. Create `text_forms.py` with a `Money` class that has `__repr__` and `__str__`.
2. Print one money value, print its `repr`, and print a list that contains it.
3. Write one sentence on which text form the list uses and why.

Done when the three outputs match what the protocol predicts.

Exercise 3 practices iterables and iterators.

1. Create `countdown.py` with a `CountdownIterator` and a `Countdown` iterable, as in the concepts section.
2. Iterate over one `Countdown` twice in separate `list` calls and record both results.
3. Call `next()` on a `CountdownIterator` until it stops, then call `next()` once more inside a `try` block and record the exception.

Done when both lists are equal and the exhausted iterator raises `StopIteration`.

Exercise 4 practices generators.

1. Create `gens.py` with the `chunks` generator from the concepts section.
2. Call `chunks` with a list of seven items and size three. Print the result with `list`.
3. Write a generator expression that yields squares of the numbers from 1 to 1,000,000 and use `sum` on the first ten thousand items with `itertools.islice`. Record the result and explain why the list of a million squares is not created.

Done when `chunks` returns the expected batches and the generator approach produces the sum without building the whole list.

Exercise 5 practices context managers.

1. Create `contexts.py` with the `labelled` context manager and the `Timer` class.
2. Use `labelled` around a block that raises `ZeroDivisionError`, catch the error outside the block, and record the printed output.
3. Use `Timer` around a loop that sums a range, and print `elapsed` as a number greater than or equal to zero.

Done when the `end` line appears even when the block fails, and the timer reports a non-negative time.

---

## Project: a money and iteration lab

The project is a small library that brings together the protocols from this day. `Money` has text forms, equality and hashing rules, and addition. `Countdown` is an iterable that returns fresh iterators. `chunks` is a generator. `Timer` and `labelled` are context managers. The tests check each protocol, and they include the cases that break when a protocol is wrong. The evidence is a passing test suite and a findings file that explains which protocol fixed which bug.

Suggested repo name: `100-days-software-day-21-python-data-model`.

Repo layout:

```
100-days-software-day-21-python-data-model/
├── .gitignore
├── README.md
├── datamodel_lab.py
├── tests/
│   ├── __init__.py
│   └── test_datamodel_lab.py
└── docs/
    └── findings.md
```

Starter code for `datamodel_lab.py`:

```python
from collections.abc import Iterator
from contextlib import contextmanager
from decimal import Decimal
from time import perf_counter
from types import TracebackType


class Money:
    def __init__(self, amount: str, currency: str = "USD") -> None:
        self.amount = Decimal(amount)
        self.currency = currency

    def __repr__(self) -> str:
        return f"Money({str(self.amount)!r}, {self.currency!r})"

    def __str__(self) -> str:
        return f"{self.amount:.2f} {self.currency}"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __hash__(self) -> int:
        return hash((self.amount, self.currency))

    def __lt__(self, other: "Money") -> bool:
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return self.amount < other.amount

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(str(self.amount + other.amount), self.currency)


class CountdownIterator:
    def __init__(self, start: int) -> None:
        self.current = start

    def __iter__(self) -> "CountdownIterator":
        return self

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value


class Countdown:
    def __init__(self, start: int) -> None:
        self.start = start

    def __iter__(self) -> CountdownIterator:
        return CountdownIterator(self.start)


def chunks(items: list[int], size: int) -> Iterator[list[int]]:
    if size <= 0:
        raise ValueError("size must be positive")
    for start in range(0, len(items), size):
        yield items[start:start + size]


class Timer:
    def __enter__(self) -> "Timer":
        self._start = perf_counter()
        self.elapsed = 0.0
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        traceback: TracebackType | None,
    ) -> None:
        self.elapsed = perf_counter() - self._start


@contextmanager
def labelled(label: str) -> Iterator[None]:
    print(f"start {label}")
    try:
        yield
    finally:
        print(f"end {label}")


if __name__ == "__main__":
    total = Money("2.50") + Money("1.25")
    print(total, repr(total))
    print(list(Countdown(3)))
    print(list(chunks([1, 2, 3, 4, 5], 2)))
    with labelled("demo"):
        print("working")
```

Starter code for `tests/test_datamodel_lab.py`:

```python
import unittest
from decimal import Decimal

from datamodel_lab import Countdown, CountdownIterator, Money, Timer, chunks, labelled


class MoneyTests(unittest.TestCase):
    def test_str_and_repr(self):
        price = Money("2.5")
        self.assertEqual(str(price), "2.50 USD")
        self.assertEqual(repr(price), "Money('2.5', 'USD')")

    def test_equality_and_hash_agree(self):
        first = Money("2.50")
        second = Money("2.5")
        self.assertEqual(first, second)
        self.assertEqual(hash(first), hash(second))
        self.assertEqual({first: "tea"}[second], "tea")

    def test_equality_with_other_type(self):
        self.assertNotEqual(Money("1"), 1)

    def test_addition_and_currency_mismatch(self):
        self.assertEqual(Money("2.50") + Money("1.25"), Money("3.75"))
        with self.assertRaises(ValueError):
            Money("1", "USD") + Money("1", "EUR")

    def test_sorting_uses_lt(self):
        prices = [Money("3"), Money("1"), Money("2")]
        self.assertEqual([str(p) for p in sorted(prices)], ["1.00 USD", "2.00 USD", "3.00 USD"])


class IterationTests(unittest.TestCase):
    def test_countdown_can_be_read_twice(self):
        countdown = Countdown(3)
        self.assertEqual(list(countdown), [3, 2, 1])
        self.assertEqual(list(countdown), [3, 2, 1])

    def test_iterator_is_exhausted(self):
        iterator = CountdownIterator(1)
        self.assertEqual(next(iterator), 1)
        with self.assertRaises(StopIteration):
            next(iterator)

    def test_chunks_returns_batches(self):
        self.assertEqual(list(chunks([1, 2, 3, 4, 5], 2)), [[1, 2], [3, 4], [5]])

    def test_chunks_rejects_bad_size(self):
        with self.assertRaises(ValueError):
            list(chunks([1], 0))

    def test_chunks_is_a_one_pass_generator(self):
        batches = chunks([1, 2, 3], 2)
        self.assertEqual(list(batches), [[1, 2], [3]])
        self.assertEqual(list(batches), [])


class ContextTests(unittest.TestCase):
    def test_timer_records_non_negative_time(self):
        with Timer() as timer:
            sum(range(1000))
        self.assertGreaterEqual(timer.elapsed, 0.0)

    def test_timer_runs_exit_on_error(self):
        with self.assertRaises(ZeroDivisionError):
            with Timer() as timer:
                1 / 0
        self.assertGreaterEqual(timer.elapsed, 0.0)

    def test_labelled_prints_end_on_error(self):
        import contextlib
        import io

        buffer = io.StringIO()
        with contextlib.redirect_stdout(buffer):
            with self.assertRaises(ZeroDivisionError):
                with labelled("job"):
                    1 / 0
        self.assertEqual(buffer.getvalue(), "start job\nend job\n")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python datamodel_lab.py` and paste the output into `docs/findings.md`. Explain in one sentence which method `print` called on the total.
3. Run `python -m unittest` and make every test pass.
4. Add `__contains__` to `Countdown` so that `2 in Countdown(3)` is decided without iterating through the whole countdown. Add a test. Explain in `docs/findings.md` why the protocol fallback would have worked without it.
5. Remove `__hash__` from `Money` temporarily and run the tests. Record which test fails and what Python says about the class. Restore `__hash__` and write one sentence on why defining `__eq__` changes hashing behavior.
6. Write a generator `numbered(lines: list[str]) -> Iterator[str]` that yields `"1. first"` and so on. Add a test.
7. Run `python -m mypy datamodel_lab.py tests/test_datamodel_lab.py` and fix every issue.
8. Commit with the message "Add data model lab with dunder, iterator and context tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least fourteen tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs, the `__hash__` experiment result, and the `__contains__` explanation
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with run commands), and `.gitignore`.

The project is compact because each protocol is short. A reviewer should be able to match each class to the protocol it implements, and to see the tests that would fail if the protocol were wrong.

---

## Check your understanding

1. A class defines `__eq__` so that two objects with equal fields compare equal, and does not define `__hash__`. Predict what happens when an instance is used as a dictionary key, and explain the rule that causes it.
2. A developer writes `for item in iterator_obj` twice on the same object and gets an empty result the second time. Explain why, and name the change that fixes it.
3. `print(obj)` shows a useful message, but `[obj]` in a list shows a cryptic string. Explain which method controls each output, and which method the developer should add.
4. A generator is passed to two functions, and the second one sees no items. Explain the mechanism and give a fix that keeps memory use low.
5. A context manager's `__exit__` returns `True` on every exception. Explain what this does to errors raised in the block, and say when returning `True` is justified.
6. Why does `in` work on a class that defines only `__getitem__` and `__len__`, and when would defining `__contains__` be better?

### Answers

1. The instance becomes unhashable, so using it as a key raises `TypeError`. Python sets `__hash__` to `None` when a class defines `__eq__` without `__hash__`, because equal objects must have equal hashes, and the class has not promised a hash that matches its equality. The fix is to define `__hash__` from the same fields that `__eq__` compares, as in the project.
2. The first loop uses up the iterator, and the object returns the same exhausted iterator on the second call because `__iter__` returns `self`. The fix is to make `__iter__` return a new iterator each time, as the `Countdown` class does.
3. `print` uses `__str__`, and a list uses `__repr__` for its items. The developer should add `__repr__`, which controls the container view and the REPL, and may also add `__str__` when a friendlier form is wanted.
4. A generator produces its items once, so the first function consumes them, and the second finds the generator exhausted. The fix is to create the generator inside each function, or to pass a function that produces a fresh generator, or to store a list when the data fits in memory and several passes are needed.
5. Returning `True` suppresses the exception, so the error disappears from the program's flow, and the code after the `with` block continues as if nothing had happened. Returning `True` is justified only for a context manager that exists to ignore a specific, expected exception, such as a helper that suppresses `FileNotFoundError` on purpose. Otherwise return `None`.
6. Python falls back to iterating over the object with `__getitem__` until it raises `IndexError`, and checks each item for equality. Defining `__contains__` is better when the check can be done directly, such as a range check, because it avoids a full scan and makes the intent clear.

---

## Sources to read, watch and practice

*Read*

- Python language reference, "Data model", at docs.python.org/3/reference/datamodel.html
- Python documentation, "Iterator Types" and "Generator Types" in "Built-in Types", at docs.python.org/3/library/stdtypes.html
- Python documentation, module `contextlib`, at docs.python.org/3/library/contextlib.html
- PEP 343, "The 'with' Statement", at peps.python.org/pep-0343/
- PEP 255, "Simple Generators", at peps.python.org/pep-0255/
- [Wikipedia: Iterator](https://en.wikipedia.org/wiki/Iterator)
- [Wikipedia: Generator (computer programming)](https://en.wikipedia.org/wiki/Generator_(computer_programming))

*Tutorials*

- GeeksforGeeks, "Python Iterators", "Python Generators" and "Python Context Managers" tutorials, at geeksforgeeks.org
- W3Schools, "Python Iterators" and "Python Generators" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on iteration and on working with files.

*Practice*

- Python Tutor at pythontutor.com. Step through the generator example and the iterable-versus-iterator example, and watch when each object is created and when its state changes.

---

## Tomorrow

Day 22 covers what happens when things go wrong. You will read tracebacks from the bottom up, use `try`, `except`, `else` and `finally` with precision, raise and chain exceptions, define custom exception classes, and use assertions and logging. You will also step through a program with the built-in debugger. The context managers you wrote today are the tool that guarantees clean-up when an error occurs.

**Next:** [Day 22: Exceptions and Debugging](Software-Day-22-Exceptions-and-Debugging.md)

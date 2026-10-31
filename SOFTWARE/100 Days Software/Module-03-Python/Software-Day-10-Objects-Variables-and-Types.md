# Day 10: Objects, Variables and Types

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner. You finished Day 9 and can run a script. This is for anyone who has been surprised that changing one list changed another, or who has written `if x:` without knowing what it checks.

**By the end of this day, you will have:**

- A working model of Python objects, in which every value has an identity, a type and a value
- The ability to tell identity (`is`) from equality (`==`) and to predict when two names share one object
- A clear rule for which types can change in place and which cannot, and what that means for function arguments
- A correct habit for checking `None` and for relying on truthiness, without the bugs that come from guessing
- A small repo with tests that demonstrate each of these ideas

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer, with a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2

**Prerequisites:**

- Day 9: Python Runtime and Syntax (running scripts, `python -m`, naming rules, indentation)
- Day 6 to Day 8: Git basics and pushing a repo to GitHub

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [The object model](#the-object-model)
  - [Identity](#identity)
  - [References](#references)
  - [Dynamic and strong typing](#dynamic-and-strong-typing)
  - [Mutability](#mutability)
  - [None](#none)
  - [Truthiness](#truthiness)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an object model explorer](#project-an-object-model-explorer)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most bugs in Python that look mysterious come from one of three ideas. A name is not the same as the value it refers to. Two names can refer to the same value. And some values can be changed in place while others cannot. A function that appends to a list you passed in, a check that treats an empty list as missing, and a comparison that says two lists are "the same" but fails a stricter test are all symptoms of these ideas. Once you can see the objects, the symptoms are predictable.

This day builds that picture from the ground up. You will see what a variable is in Python, why assignment copies nothing, how `is` differs from `==`, and why some operations change the object they act on while others return a new one. You will also learn the two checks that new programmers get wrong most often: comparing with `None`, and testing whether a value is "empty" through truthiness.

The official reference for these ideas is the Python Data Model chapter at docs.python.org/3/reference/datamodel.html. It is long, so read the sections on objects, values and types now, and return to the rest when you meet special methods on Day 21.

---

## Glossary

- **Object:** a value stored in memory, with an identity, a type and a value. Every piece of data in Python is an object.
- **Identity:** a number that uniquely identifies an object while it exists. You read it with `id(obj)`.
- **Type:** the kind of an object, such as `int`, `str` or `list`. Types are objects too.
- **Name (variable):** a label bound to an object. Assignment binds a name to an object.
- **Reference:** a link from a name, or from another object, to an object. Many references can point to one object.
- **Alias:** two names bound to the same object.
- **Mutable:** able to change its value in place, such as a `list` or a `dict`.
- **Immutable:** unable to change once created, such as an `int`, a `str` or a `tuple`.
- **Dynamic typing:** names have no fixed type. The object they point to has the type.
- **Strong typing:** operations between incompatible types raise an error instead of converting silently.
- **Singleton:** an object of which only one exists in a program. `None` is one.
- **Truthiness:** the true or false value Python assigns to any object when it is used in a condition.

---

## Concepts

The seven concepts below form a chain. The object model is the base, identity and references describe how objects are shared, typing and mutability explain what can change, and `None` and truthiness are the two everyday checks built on top.

### The object model

Think of a parking garage. Each car is a physical object. Each car has a plate that identifies it, a make that tells you what it is, and a state, such as full or empty. A parking ticket is a label that points to one car. You can hand a second ticket to the same car, and you can throw a ticket away without moving the car. In Python, the car is the object, the plate is the identity, the make is the type, and the ticket is the name.

From first principles, every value in a running Python program is an object, including integers, functions, classes and modules. Each object stores three things: an identity, which the interpreter uses to track it, a type, which decides what operations it supports, and a value, which is its contents. A type is itself an object, so `type(3)` returns the `int` object, and `type(int)` returns `type`. That loop is part of why Python feels uniform.

The definition to keep is short. An assignment such as `x = 3` creates or finds an object for the value `3`, then binds the name `x` to it. The name does not hold the object. It refers to it.

```python
x = 3
print(type(x))      # <class 'int'>
print(id(x))        # a number, which will differ on your machine
print(x + 1)        # 4
```

The common mistake is assuming that a name is a box that holds a value and that assignment copies the value into the box. In Python, assignment copies nothing. It creates one more reference. The job market rewards engineers who can explain this, because code reviews on data pipelines and web services often hinge on whether two parts of a program share one object. A linter cannot tell you that, so you learn to look for aliasing when reading code.

### Identity

Two objects can look identical and still be different objects. Imagine two copies of the same book from the same printing. Their titles match, so they are equal in content, but they sit on two different shelves. Identity asks whether it is the same physical book. Equality asks whether the contents match.

From first principles, the `is` operator compares identity, and `==` compares values through the object's `__eq__` method. The two are separate questions. `a is b` is true only when both names refer to the same object in memory. `a == b` is true when the contents are considered equal by the type's rules.

```python
a = [1, 2]
b = a            # same object
c = [1, 2]       # equal contents, different object

print(a is b)    # True
print(a is c)    # False
print(a == c)    # True
```

Small integers and some strings are cached by CPython, so `x = 5` and `y = 5` often give `x is y` as `True`. Do not rely on that. CPython documents it as an implementation detail, not a language rule. Use `is` only for singletons such as `None`, and use `==` for values.

The common mistake is writing `if value is "red":` and expecting it to compare text. This can pass in one test and fail in another, because string interning is an optimization that depends on how the string was created. Python 3.8 and later warn about `is` with a literal. In the job market, reviewers flag this immediately, and a linter such as Ruff with its comparison rules catches it. The fix is `==` for values and `is` for `None`.

### References

A reference is a link. When you bind a name to an object, the name holds a reference to that object, and the object keeps a count of how many references point to it. When the count drops to zero, CPython frees the memory. Mutable objects can also hold references to other objects, which is how a list of lists works.

From first principles, passing an object to a function passes a reference to the same object, not a copy. Python calls this "call by object reference" or "pass by assignment". The function's parameter becomes another name for the same object. If the object is mutable and the function changes it, the caller sees the change. If the function rebinds the parameter to a new object, the caller does not.

```python
def add_flag(items: list[str]) -> None:
    items.append("done")      # changes the caller's object


def replace(items: list[str]) -> None:
    items = ["new"]           # rebinds only the local name


data = ["todo"]
add_flag(data)
print(data)                   # ['todo', 'done']
replace(data)
print(data)                   # still ['todo', 'done']
```

The first function mutates the object both names refer to. The second function changes only what its local name refers to, and the caller's name is untouched. This is the mechanism behind a lot of surprises.

The common mistake is using a mutable default argument, such as `def add(item, items=[])`. The default list is created once, when the function is defined, so every call that uses the default shares it. Use `None` as the default and create the list inside the function. Day 16 returns to this. In the job market, this bug appears in production code often enough that interviewers ask about it directly.

### Dynamic and strong typing

Think of a label maker that can print any label you like, and a glue gun that refuses to join two incompatible parts. Python's labels are names, and any name can point to any type of object at any time. That is dynamic typing. Python's rule that stops you from joining incompatible parts is strong typing.

From first principles, dynamic typing means the type check happens at runtime, on the object, not on the name. You can write `x = 1`, then `x = "one"`, and both are legal. Strong typing means the interpreter does not silently convert between unrelated types when an operator is used. Python refuses `"1" + 1` with a `TypeError`, where some languages would turn the number into text or the text into a number.

```python
x = 1
x = "one"              # legal: the name now refers to a str
print(type(x).__name__)  # str

try:
    total = "1" + 1
except TypeError as err:
    print("error:", err)
```

The output of the second part is `error: can only concatenate str (not "int") to str`. The message names the problem directly. Strong typing is why a missing conversion usually fails loudly, which is the behavior you want during development.

The common mistake is treating the lack of declared types as the lack of types. Python has types, and a function that expects a number will fail when it gets a string. Type hints, such as `def price(qty: int) -> float`, document the intent, and the tool `mypy` checks them without running the code. In the job market, typed Python is standard in serious codebases, and type checking in CI is a common requirement.

### Mutability

Mutability describes whether an object can change in place. A mutable object is like a whiteboard, where you can erase and rewrite the contents while it stays the same board. An immutable object is like a printed page. To change its text, you must print a new page.

From first principles, `list`, `dict` and `set` are mutable. Methods such as `append`, `update` and `add` change the object and return `None`. `int`, `float`, `str` and `tuple` are immutable. Operations on them produce new objects. `s = s + "!"` does not change the string `s` refers to at first. It creates a new string and rebinds `s`.

```python
names = ["Ada"]
names.append("Grace")      # mutates the list in place
print(names)               # ['Ada', 'Grace']

label = "task"
label.upper()              # returns a new string; label is unchanged
print(label)               # task
label = label.upper()      # rebind to the new string
print(label)               # TASK
```

The common mistake is calling a method that returns a new value and ignoring the return value. `text.strip()` does nothing unless you write `text = text.strip()`. The reverse mistake is assuming `list.sort()` returns a sorted list. It sorts in place and returns `None`, so `ordered = items.sort()` makes `ordered` equal `None`. Use `sorted(items)` when you want a new list. Teams that process data in pipelines often rely on this distinction to avoid accidental mutation, and writing code that is explicit about it is a hiring signal.

### None

`None` is Python's way of saying "no value here". It is a singleton object of type `NoneType`. Every function that finishes without a `return` statement returns `None`, which is why an accidental `None` often appears where a value was expected.

From first principles, there is exactly one `None` in a running program. That is why the correct check is `value is None`, which compares identity with the single object. Using `==` works in most cases, but a class can define its own `__eq__`, and then `==` may not behave as you expect. The `is` check is faster and unambiguous.

```python
def find_price(prices: dict[str, float], item: str) -> float | None:
    return prices.get(item)


price = find_price({"tea": 2.5}, "coffee")
if price is None:
    print("not stocked")
else:
    print(f"price: {price:.2f}")
```

The `float | None` annotation says the function may return a float or nothing. Day 16 covers annotations in full, and Day 14 covers `dict.get`.

The common mistake is forgetting that `None` is a value that can propagate. A function that returns `None` by accident is called, its result stored, and an error appears three functions later with `'NoneType' object has no attribute`. Read the traceback from the bottom up to find where the `None` first appeared. In the job market, `None` errors are among the most common runtime exceptions in Python services, and type checkers catch them when annotations are present.

### Truthiness

Every object in Python has a truth value. When you use an object in an `if`, `while` or `not`, Python asks for its truth value. Think of a sieve: most objects pass through as true, but a small set of empty or zero values are caught and treated as false.

From first principles, the rule is defined in the language reference. Python treats these values as false: `None`, `False`, zero of any numeric type (`0`, `0.0`, `0j`), and empty sequences and collections (`""`, `[]`, `()`, `{}`, `set()`, `range(0)`). Everything else is true. A class controls its own truth value through `__bool__`, or, if that is missing, through `__len__`.

```python
for value in [0, 1, "", "a", [], [0], None]:
    print(repr(value), "is", "true" if value else "false")
```

The output shows `[0]` as true, because it has one element, even though that element is false. Truthiness asks about the container's emptiness, not the emptiness of its contents.

The common mistake is using truthiness where `None` is the real question. `if count:` treats `0` the same as "no count". If zero is a valid answer, write `if count is not None:`. This bug shows up in form validation, pagination and inventory code, and the fix is always to name the case you mean. Reviewers in the job market look for this specific precision, because a single missed zero can ship wrong totals.

---

## Hands-on exercises

Work in a folder called `object-model-lab`, with Python 3.14 or 3.12 or newer.

Exercise 1 explores identity with `id` and `is`.

1. Create `identity.py`. Bind `a = [1, 2]`, `b = a`, and `c = [1, 2]`. Print `a is b`, `a is c`, `a == c`, and the `id` of each.
2. Run it. Confirm that `a` and `b` have the same `id` and that `c` differs.
3. Add `x = 5` and `y = 5` and print `x is y`. Then add `s = "hello world"` and `t = "hello world"` and print `s is t`. Write one sentence on why these results should not be relied on.

Done when you can predict each printed boolean before running the file.

Exercise 2 demonstrates pass-by-object-reference.

1. Write a function `add_tag(tags: list[str]) -> None` that appends `"new"` to the list.
2. Write a second function `replace_tags(tags: list[str]) -> None` that assigns `tags = ["replaced"]`.
3. Call each on a list that starts as `["old"]` and print the list after each call.

Done when the first call changes the list and the second does not.

Exercise 3 shows mutable and immutable methods.

1. Create `mutable_demo.py`. Make a list and call `append`, `sort` and `reverse`. After each call, print the value and the return value of the call.
2. Make a string and call `upper`, `strip` and `replace`. Print the original string after each call.
3. Write a sentence explaining why the return value of `list.sort()` is `None`.

Done when you can explain which methods change the object and which return a new one.

Exercise 4 checks `None` and truthiness.

1. Write a function `safe_div(a: float, b: float) -> float | None` that returns `None` when `b` is zero.
2. Write a function `report(value: float | None) -> str` that returns `"missing"` when the value is `None`, and `"value: X"` otherwise, using `is None`.
3. Call `report` with `0`, `0.0`, `None` and `safe_div(1, 0)`. Confirm that `0` and `0.0` are reported as values, not as missing.

Done when `0` and `0.0` produce the value branch and `None` produces the missing branch.

Exercise 5 runs the type checker.

1. Install `mypy` with `python -m pip install mypy`. Confirm it prints its version with `python -m mypy --version`.
2. Create `typed.py` with a function `price(qty: int) -> float` that returns `qty * 2.5`. Call it once with the string `"3"`.
3. Run `python -m mypy typed.py` and read the error. Fix the call and rerun.

Done when mypy reports `Success: no issues found` on the fixed file.

---

## Project: an object model explorer

The project is a small repo whose code demonstrates the object model in four short modules and tests the claims with `unittest`. It is light on new features. It is heavy on precision: each function encodes a rule from this day, and each test checks that rule. The evidence you commit is a set of passing tests plus a short write-up in which you explain each rule in your own words.

Suggested repo name: `100-days-software-day-10-objects-variables-and-types`.

Repo layout:

```
100-days-software-day-10-objects-variables-and-types/
├── .gitignore
├── README.md
├── objects_lab.py
├── mutation_demo.py
├── optional_lookup.py
├── tests/
│   ├── __init__.py
│   └── test_objects_lab.py
└── docs/
    └── findings.md
```

Starter code for `objects_lab.py`:

```python
def same_object(a: object, b: object) -> bool:
    return a is b


def describe(value: object) -> str:
    return f"{type(value).__name__}: {value!r}"


TRUTHY_SAMPLES: list[object] = [0, 1, "", "a", [], [0], {}, None, 0.0, 0.1]


def truth_table() -> list[tuple[str, bool]]:
    return [(repr(v), bool(v)) for v in TRUTHY_SAMPLES]


if __name__ == "__main__":
    a = [1, 2]
    b = a
    c = [1, 2]
    print(describe(a))
    print("a is b:", same_object(a, b))
    print("a is c:", same_object(a, c), "| a == c:", a == c)
    for text, flag in truth_table():
        print(f"{text:>8} -> {flag}")
```

Starter code for `mutation_demo.py`:

```python
def add_item_in_place(items: list[str], item: str) -> None:
    items.append(item)


def add_item_copy(items: list[str], item: str) -> list[str]:
    return [*items, item]


if __name__ == "__main__":
    shared = ["apple"]
    alias = shared
    add_item_in_place(alias, "pear")
    print("shared after in-place append:", shared)

    original = ["apple"]
    new = add_item_copy(original, "pear")
    print("original after copy:", original)
    print("new:", new)
```

Starter code for `optional_lookup.py`:

```python
def find_user(users: dict[int, str], user_id: int) -> str | None:
    return users.get(user_id)


def greeting(users: dict[int, str], user_id: int) -> str:
    name = find_user(users, user_id)
    if name is None:
        return "Hello, guest"
    return f"Hello, {name}"


if __name__ == "__main__":
    people = {1: "Ada", 2: "Grace"}
    print(greeting(people, 1))
    print(greeting(people, 99))
```

Starter code for `tests/test_objects_lab.py`:

```python
import unittest

from mutation_demo import add_item_copy, add_item_in_place
from objects_lab import describe, same_object, truth_table
from optional_lookup import greeting


class ObjectTests(unittest.TestCase):
    def test_identity_versus_equality(self):
        a = [1, 2]
        b = a
        c = [1, 2]
        self.assertTrue(same_object(a, b))
        self.assertFalse(same_object(a, c))
        self.assertEqual(a, c)

    def test_describe(self):
        self.assertEqual(describe(3), "int: 3")

    def test_truth_table(self):
        table = dict(truth_table())
        self.assertFalse(table["0"])
        self.assertFalse(table["''"])
        self.assertTrue(table["'a'"])
        self.assertFalse(table["None"])
        self.assertTrue(table["[0]"])

    def test_mutation_versus_copy(self):
        shared = ["apple"]
        add_item_in_place(shared, "pear")
        self.assertEqual(shared, ["apple", "pear"])

        original = ["apple"]
        new = add_item_copy(original, "pear")
        self.assertEqual(original, ["apple"])
        self.assertEqual(new, ["apple", "pear"])

    def test_greeting_handles_none(self):
        people = {1: "Ada"}
        self.assertEqual(greeting(people, 1), "Hello, Ada")
        self.assertEqual(greeting(people, 2), "Hello, guest")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the four code files and the test package, and add `__pycache__/` to `.gitignore`.
2. Run `python objects_lab.py`. In `docs/findings.md`, paste the output and write two sentences explaining why `a is c` is false while `a == c` is true.
3. Run `python mutation_demo.py`. In `docs/findings.md`, explain in two sentences why the first list changed and the second did not.
4. Add a function `first_item(items: list[str]) -> str | None` to `mutation_demo.py` that returns `None` for an empty list and the first item otherwise. Use `is None` or truthiness deliberately, and write one sentence in `docs/findings.md` explaining which you chose and why.
5. Run `python -m unittest`. Add one test for `first_item` on an empty list and one on a non-empty list. Rerun until every test passes.
6. Run `python -m mypy objects_lab.py mutation_demo.py optional_lookup.py` and fix any reported issue. Record the final mypy line in `docs/findings.md`.
7. Commit the code with the message "Add object model explorer with tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least six tests and passes
- `python -m mypy` reports no issues on the four modules
- `docs/findings.md` contains the outputs from steps 2 and 3, the sentences from steps 2, 3 and 4, and the mypy line
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (listing the run commands for the scripts, tests and mypy), and `.gitignore`.

The project is light on code because the concepts are about precision, not volume. A reviewer who reads `docs/findings.md` next to the tests should be able to see that each rule was both tested and described.

---

## Check your understanding

1. You write `a = [1]`, `b = a`, then `b.append(2)`. What does `a` contain, and why does `b.append` affect `a`? Name the concept that explains it.
2. A teammate says that `x is y` and `x == y` mean the same thing for integers. Give a case where they could disagree in a way that matters, and say what the correct check should be.
3. A function `def add(item, items=[])` appends to `items` and returns it. Call it twice without passing `items`. What is the second return value, and what change fixes it?
4. Given `s = "report"`, a student writes `s.strip()` and then `print(s)`. Explain what prints and why.
5. A function returns `0` when a customer has no points. Another code path checks `if points:` and skips the reward. Explain the bug and write the corrected check.
6. Why is `None` checked with `is None` rather than `== None` in the code you commit?

### Answers

1. `a` contains `[1, 2]`. `a` and `b` are two names bound to the same list object, so the append through `b` mutates the object that `a` also refers to. The concept is aliasing, which follows from assignment creating references rather than copies.
2. For small integers, CPython may cache objects, so `is` can return `True` in some cases, but that is an implementation detail. For large integers created separately, `is` can return `False` while `==` returns `True`. The correct check for numeric values is `==`.
3. The second call returns a list containing both values, for example `[1, 2]` when the calls were `add(1)` and `add(2)`. The default list is created once, when the function is defined, and every call without `items` shares it. Use `items: list | None = None` and create the list inside the function when `items is None`.
4. Nothing new prints, so `s` still contains `"report"`. `strip` returns a new string and `s` is immutable, so the result is discarded. The fix is `s = s.strip()`.
5. A points value of `0` is falsy, so `if points:` treats a customer with zero points the same as a customer with no record at all. If the rule is "award a reward for any points", zero correctly gets nothing, but if the rule is "award a reward only when a record exists", the check silently skips the customer who has a record with zero points. The corrected check names the case: `if points is not None:` when `None` means no record, and `if points > 0:` when the rule is about positive points.
6. `None` is a singleton, so identity is the precise question. A class can define `__eq__` to return true for `None`, and `is` cannot be fooled that way. It is also the convention in PEP 8.

---

## Sources to read, watch and practice

*Read*

- Python Software Foundation, "The Python Language Reference", chapter "Data model", at docs.python.org/3/reference/datamodel.html
- Python documentation, "Built-in Types", at docs.python.org/3/library/stdtypes.html, sections on truth value testing and sequence types
- [Wikipedia: Object (computer science)](https://en.wikipedia.org/wiki/Object_(computer_science))
- [Wikipedia: Mutable object](https://en.wikipedia.org/wiki/Mutable_object)
- [Wikipedia: Type system](https://en.wikipedia.org/wiki/Type_system)

*Tutorials*

- GeeksforGeeks, "Mutable and Immutable Objects in Python", and "Python Data Types" tutorials, at geeksforgeeks.org
- W3Schools, "Python Data Types" and "Python Booleans" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segment on variables and data types.

*Practice*

- Python Tutor at pythontutor.com. Paste the aliasing example and the mutable default example, step through them, and watch the reference arrows change.

---

## Tomorrow

Day 11 moves from objects to the numeric types that most programs use. You will work with integers and floats, see why `0.1 + 0.2` is not exactly `0.3`, learn how booleans are a subtype of integers, and practice conversions and operator precedence. The idea of an object with a type, which you studied today, is what makes those conversions meaningful.

**Next:** [Day 11: Numbers and Operators](Software-Day-11-Numbers-and-Operators.md)

# Day 16: Functions I

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner to intermediate. You can write branches and loops from Day 15 and you understand references from Day 10. This is for anyone who has pasted the same block of code three times, or who has passed the wrong value into a function and got an answer that looked plausible.

**By the end of this day, you will have:**

- A working model of a function as a named, reusable block that takes inputs through parameters and returns a result
- The ability to tell parameters from arguments, to bind arguments by position and by keyword, and to use defaults safely
- A correct understanding of `return`, including the implicit `None` and returning several values as a tuple
- A clear model of scope: which names a function can read, which it can assign, and in what order Python looks them up
- The ability to write docstrings that `help()` displays, and type annotations that a checker such as mypy verifies
- A small calculations repo with tests that check parameter binding, scope rules, and documented behavior

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 9: Python Runtime and Syntax (scripts, `__name__`, indentation)
- Day 10: Objects, Variables and Types (names, references, mutable defaults)
- Day 15: Control Flow (branches and loops inside functions)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Parameters](#parameters)
  - [Arguments](#arguments)
  - [Return](#return)
  - [Scope](#scope)
  - [LEGB](#legb)
  - [Docstrings](#docstrings)
  - [Annotations](#annotations)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a calculations module](#project-a-calculations-module)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A function is the first tool that turns a script into a program. Without functions, every calculation is a copy of the last one, and every fix must be made in several places. With functions, a calculation has one name, one set of inputs, one result, and one place to test. Functions are also the unit that the rest of the course tests, documents, types and reuses. A codebase is mostly functions and the classes that group them.

This day covers the mechanics of a Python function: how its parameters receive values, how `return` sends a result back, how Python decides which name a function sees, and how docstrings and annotations describe what a function does. The next day builds on this with defaults, variable-length arguments, closures and lambdas. Today's goal is to write functions whose inputs, outputs and surrounding names are all predictable.

The official reference is the Python tutorial section "Defining Functions" at docs.python.org/3/tutorial/controlflow.html#defining-functions, and the language reference section "Function definitions" at docs.python.org/3/reference/compound_stmts.html#function-definitions. The scope rules are documented under "Naming and binding" at docs.python.org/3/reference/executionmodel.html.

---

## Glossary

- **Function:** a named block of code that can be called with inputs and returns a result.
- **Parameter:** a name in a function definition that receives a value when the function is called. Also called a formal parameter.
- **Argument:** the actual value passed to a parameter in a call. Also called an actual argument.
- **Positional argument:** an argument matched to a parameter by its place in the call.
- **Keyword argument:** an argument matched to a parameter by name, written as `name=value`.
- **Default value:** a value a parameter takes when the caller does not pass one.
- **Return value:** the value a function sends back to its caller with `return`. A function with no `return` returns `None`.
- **Scope:** the region of code where a name is visible.
- **Local scope:** names assigned inside the current function.
- **Enclosing scope:** names in any function that contains the current one.
- **Global scope:** names defined at the top level of a module.
- **Built-in scope:** names that Python provides, such as `len` and `print`.
- **LEGB:** the lookup order Local, Enclosing, Global, Built-in.
- **Docstring:** a string literal that is the first statement of a function, module or class, and that documents it.
- **Annotation:** a type hint written after a parameter or after `->`, such as `width: float` or `-> float`.

---

## Concepts

The concepts follow a function's life. Values come in through parameters, names are looked up through scope, a result goes out through return, and the function is described through docstrings and annotations.

### Parameters

Think of a vending machine's input slot. The machine accepts coins of a particular type in a particular slot, and it does not care what coin you hold in your head. A parameter is a slot. It has a name, and it receives whatever value the caller passes in that position or under that name.

From first principles, `def area(width, height):` creates a function object and binds the name `area` to it. The names `width` and `height` are parameters. They exist only while the function runs. When the function is called, Python creates a fresh local namespace, binds each parameter to the argument it receives, runs the body, and then discards the namespace. Two calls therefore never share parameter names, even when the names are the same.

```python
def area(width: float, height: float) -> float:
    return width * height


print(area(3, 4))      # 12
print(area(4, 3))      # 12, each call has its own width and height
```

The parameters are names, and the arguments are the values that the calls provide. Because the function body only sees the names, you can change the function without changing its callers, as long as the parameter names and their meanings stay the same.

The common mistake is changing a parameter name in a function that callers pass by keyword. The keyword call breaks with `TypeError: area() got an unexpected keyword argument`. Treat parameter names as part of the public interface. In the job market, renaming a parameter in a shared library is a breaking change, and maintainers document it in release notes for that reason.

### Arguments

Arguments are the values you hand to a function at the call site. Think of ordering a meal with a set menu. You can say "the first course is soup and the second is fish", which relies on order, or you can say "the main is fish", which names the course. Python lets you use both styles.

From first principles, Python matches positional arguments to parameters from left to right, and then matches keyword arguments by name. A positional argument cannot follow a keyword argument in a call. Each parameter receives exactly one value. If a parameter has a default value, the caller may omit it. Defaults are evaluated once, when the `def` statement runs, not each time the function is called.

```python
def make_label(name: str, prefix: str = "item", sep: str = ": ") -> str:
    return f"{prefix}{sep}{name}"


print(make_label("tea"))                        # item: tea
print(make_label("tea", "drink"))               # drink: tea
print(make_label("tea", sep=" - "))             # item - tea
print(make_label(prefix="drink", name="tea"))   # drink: tea
```

The second call uses positional matching for `prefix`. The third skips `prefix` and names `sep`, which is clearer when a function has several defaults.

The common mistake is a default that holds a mutable object, such as `def add(item, items=[])`. The default list is created once, when the function is defined. Every call that omits `items` shares it, so items accumulate across calls. Use `None` as the default and create the list inside the function. Day 10 covered this trap, and it returns here because it is one of the most frequent interview questions about functions. Another common mistake is forgetting that `print(make_label(prefix="drink", name="tea"))` works only because the keyword names match the parameters exactly.

### Return

`return` sends a value back to the caller and ends the function. Think of a service counter, where the clerk takes your form, processes it, and hands you a receipt. The receipt is the return value, and the counter closes for that transaction when the receipt is handed over.

From first principles, when Python reaches `return value`, it evaluates the expression, leaves the function, and gives the value to the caller. If the function reaches its end without a `return`, the result is `None`. A bare `return` also gives `None`. A function can return several values by separating them with commas. Python packs them into a tuple, and the caller can unpack the tuple on Day 13's pattern.

```python
def split_total(total: float, parts: int) -> tuple[float, float]:
    share = round(total / parts, 2)
    remainder = round(total - share * parts, 2)
    return share, remainder


share, remainder = split_total(10.0, 3)
print(share, remainder)   # 3.33 0.01

def log(message: str) -> None:
    print(message)


result = log("saved")     # prints saved, result is None
print(result)             # None
```

The first function returns a tuple, and the caller unpacks it. The second function has no meaningful return value, and its annotation says so with `-> None`.

The common mistake is writing a function that prints a value when it should return it. The caller then receives `None`, and an error appears later, when the `None` is used in arithmetic. Return the value, and let the caller decide whether to print it. A second mistake is returning early in one branch and forgetting the other branch, so some paths return `None` without intending to. Type checkers catch this when the return annotation is `float` and a branch returns nothing.

### Scope

Scope answers the question "which name does this line mean?" Think of a building with several floors. A name on the top floor is visible only on that floor unless it is announced to the floors below. Python's scopes are nested in the same way, and each function call creates its own top floor.

From first principles, Python decides at compile time which names in a function body are local. A name becomes local when the function assigns to it anywhere in its body, even after the line where the name is first read. A name that the function only reads is looked up through the enclosing scopes. Assigning to a name inside a function does not change a global of the same name, unless you declare it with `global`. Each function call gets fresh local names, which is why two calls never leak values to each other through their locals.

```python
rate = 0.2                  # global name


def show_rate() -> float:
    return rate             # reads the global name


def change_rate() -> None:
    global rate
    rate = 0.25             # changes the global name


show_rate()                 # 0.2
change_rate()
print(rate)                 # 0.25
```

The `global` declaration is needed only for assignment. Reading a global needs no declaration, and the function sees the current value at the time of the call.

The common mistake is reading a name before assigning it in the same function, which raises `UnboundLocalError`. The function treats the name as local for the whole body, so the earlier read fails. Another common mistake is using `global` to pass data between functions. That couples the functions through hidden state, which makes testing hard. Pass values as parameters and return results instead. In the job market, global state is a frequent cause of flaky tests, and code reviewers ask for its removal.

### LEGB

LEGB is the order in which Python looks up a name. Think of checking a list of possible owners for a lost umbrella. First you check your own bag, then the coat you are wearing, then the cupboard at home, and last the public lost-and-found desk. You stop at the first place that has the name. The letters stand for Local, Enclosing, Global and Built-in.

From first principles, when code reads a name, Python searches the local scope first, then the enclosing function scopes from the nearest outward, then the module's global scope, and finally the built-in names. The search stops at the first match. If none matches, Python raises `NameError`. The rule also explains shadowing: a local name hides a global or a built-in of the same name inside that function.

```python
name = "global"                     # G


def outer() -> str:
    name = "enclosing"              # E
    def inner() -> str:
        return name                 # reads E, the nearest enclosing name
    return inner()


def shadow_len() -> int:
    len = 3                         # L, shadows the built-in len in here only
    return len


print(outer())            # enclosing
print(shadow_len())       # 3
print(len("abc"))         # 3, the built-in is unchanged at module level
```

The `inner` function does not assign to `name`, so it looks outward and finds the enclosing `name`. The `shadow_len` function assigns to `len`, so `len` is local there. Module-level code still sees the built-in.

The common mistake is shadowing a built-in such as `list`, `str` or `sum`, which breaks later calls in the same scope. The second mistake is assuming that a name assigned inside a loop or an `if` becomes local only in that block. Python function scope is not block-scoped, so the name exists for the whole function. In the job market, reviewers catch shadowing with linters such as Ruff, which has a rule for built-in shadowing, and the fix is a clear, specific name.

### Docstrings

A docstring is the first statement in a function body, written as a string literal. It explains what the function does, its inputs and its result. Think of the instruction card that comes with a piece of furniture. You do not need to read it to use the chair, but the card tells you how to assemble and use it correctly.

From first principles, Python stores the docstring in the function's `__doc__` attribute, and tools read it. The built-in `help()` displays it, IDEs show it on hover, and tools such as `pydoc` and documentation generators collect it. The convention from PEP 257 is a one-line summary in the first line, a blank line, and then a longer description if the function needs one. Use triple double quotes, which allow multiple lines.

```python
def apply_discount(price: float, percent: float, minimum: float = 0.0) -> float:
    """Return the price after a percentage discount.

    The result is never below minimum. The percent is a number from 0 to 100.
    """
    discounted = price * (1 - percent / 100)
    return max(discounted, minimum)


print(apply_discount.__doc__.splitlines()[0])   # Return the price after a percentage discount.
help(apply_discount)                            # shows the signature and the docstring
```

The docstring states a unit (percent from 0 to 100), which the signature alone does not show. That is the kind of fact a docstring should carry.

The common mistake is writing a docstring that repeats the function name and adds nothing, such as "apply_discount applies a discount". Say what the function promises: the inputs it accepts, the result it returns, and any error it raises. The second mistake is letting the docstring drift from the code after a change. In the job market, a stale docstring is a common review comment, and tests that check documented behavior are one way to keep them honest.

### Annotations

An annotation is a type hint. It says what type a parameter expects and what type the function returns. Think of a label on a jar that says "flour, 500 grams". The label does not stop someone from putting sugar in the jar, but it tells everyone what the jar is supposed to hold, and an inspector can check it.

From first principles, Python does not enforce annotations at runtime. `def area(width: float) -> float:` runs the same way as the unannotated version, and passing a string does not raise an error when the function is called. The annotations are stored as metadata, and a static type checker such as mypy reads them and reports mismatches before the code runs. Python 3.14 evaluates annotations lazily, which means a forward reference to a name defined later is no longer an error at definition time.

```python
from typing import get_type_hints


def area(width: float, height: float) -> float:
    return width * height


print(get_type_hints(area))   # {'width': <class 'float'>, 'height': <class 'float'>, 'return': <class 'float'>}
print(area("3", 4))           # prints 3333: str * int repeats the string
```

The second call shows the limit. Python runs the function without complaint and returns the string `'3333'`, where the annotation promised a float, because annotations are not checked at runtime. Mypy would flag the call before the program runs.

The common mistake is believing that annotations make the code safe. They make intent visible and let a checker find problems, but they do not validate input from users or files. Validate external data explicitly, as you did with `ValueError` in Day 11. A second mistake is annotating with `float` when `int` is also allowed. Python's numeric tower treats `int` as acceptable where `float` is expected, so `float` is the correct annotation for a value that may be an integer. In the job market, typed codebases use mypy or pyright in continuous integration, and a function without annotations is harder to review.

---

## Hands-on exercises

Work in a folder called `functions-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices parameters and arguments.

1. Create `binding.py`. Write `make_label(name, prefix="item", sep=": ")` that returns a label.
2. Call it four ways: with positional arguments, with a keyword for `sep`, with keywords in reverse order, and with only the name.
3. Predict the output of each call before running the file.

Done when every output matches your prediction and you can explain why the reversed keywords work.

Exercise 2 practices return values.

1. Create `returns.py`. Write `split_total(total, parts)` that returns a tuple of the share and the remainder, both rounded to two decimal places.
2. Write `log(message)` that prints the message and returns `None`. Assign its result to a variable and print the variable.
3. Write one sentence explaining why `log` should not return the message instead of printing it, if the function is also used by other code.

Done when `split_total(10.0, 3)` returns `(3.33, 0.01)` and the log result prints `None`.

Exercise 3 practices scope and `global`.

1. Create `scope.py`. Define a global `rate = 0.2`, a function that reads it, and a function that changes it with `global`.
2. Call the reader, then the writer, then the reader again, and record the three values.
3. Write a version of the writer that takes the rate as a parameter and returns the new rate, and compare the two designs in one sentence.

Done when the reader returns 0.2, then 0.25 after the writer, and the parameter version needs no global.

Exercise 4 practices LEGB.

1. Create `legb.py`. Define a global string, an outer function with an enclosing variable, and an inner function that reads it.
2. Define a function that assigns to a name that shadows a built-in, and call it. Then call the built-in at module level.
3. Write a function that reads a variable before assigning it, and observe the `UnboundLocalError`.

Done when you can name the scope each name came from and explain the error.

Exercise 5 uses docstrings and annotations.

1. Create `documented.py` with `apply_discount(price, percent, minimum=0.0)`. Give it a docstring that states the unit of `percent`.
2. Run `help(apply_discount)` in the REPL and copy the output's first lines.
3. Run `python -m mypy documented.py` on a file that calls the function with a string argument for `price`, read the error, and fix the call.

Done when `help` shows your docstring and mypy reports the error on the bad call and is clean after the fix.

---

## Project: a calculations module

The project is a small calculations module with documented functions, type annotations, and tests that check parameter binding, returns, defaults, and scope. The functions are simple. Their contracts are precise, and the tests make the contracts visible. The evidence is the passing test suite and a findings document that shows the `help` output and the mypy result.

Suggested repo name: `100-days-software-day-16-functions-i`.

Repo layout:

```
100-days-software-day-16-functions-i/
├── .gitignore
├── README.md
├── calculations.py
├── scope_demo.py
├── tests/
│   ├── __init__.py
│   └── test_calculations.py
└── docs/
    └── findings.md
```

Starter code for `calculations.py`:

```python
def area(width: float, height: float) -> float:
    """Return the area of a rectangle.

    Both sides must be zero or greater. A negative side raises ValueError.
    """
    if width < 0 or height < 0:
        raise ValueError("sides must be non-negative")
    return width * height


def make_label(name: str, prefix: str = "item", sep: str = ": ") -> str:
    """Build a label such as 'item: tea'."""
    return f"{prefix}{sep}{name}"


def total_with_tax(prices: list[float], rate: float = 0.2) -> float:
    """Return the sum of prices plus tax, rounded to two decimal places.

    The rate is a fraction, so 0.2 means 20 percent.
    """
    subtotal = sum(prices)
    return round(subtotal * (1 + rate), 2)


def apply_discount(price: float, percent: float, minimum: float = 0.0) -> float:
    """Return the price after a percentage discount.

    The percent is a number from 0 to 100. The result is never below minimum.
    """
    discounted = price * (1 - percent / 100)
    return max(discounted, minimum)


def split_total(total: float, parts: int) -> tuple[float, float]:
    """Split a total into equal shares and a remainder, both rounded to cents."""
    share = round(total / parts, 2)
    remainder = round(total - share * parts, 2)
    return share, remainder


if __name__ == "__main__":
    print(area(3, 4))
    print(make_label("tea"))
    print(total_with_tax([10.0, 5.0], 0.1))
    print(apply_discount(100, 10))
    print(split_total(10.0, 3))
```

Starter code for `scope_demo.py`:

```python
rate = 0.2


def show_rate() -> float:
    return rate


def change_rate() -> None:
    global rate
    rate = 0.25


def shadow_len() -> int:
    len = 3
    return len


if __name__ == "__main__":
    print(show_rate())
    change_rate()
    print(show_rate())
    print(shadow_len(), len("abc"))
```

Starter code for `tests/test_calculations.py`:

```python
import unittest

import scope_demo
from calculations import apply_discount, area, make_label, split_total, total_with_tax


class CalculationTests(unittest.TestCase):
    def test_area(self):
        self.assertEqual(area(3, 4), 12)

    def test_area_rejects_negative_side(self):
        with self.assertRaises(ValueError):
            area(-1, 2)

    def test_label_defaults_and_keywords(self):
        self.assertEqual(make_label("tea"), "item: tea")
        self.assertEqual(make_label("tea", "drink"), "drink: tea")
        self.assertEqual(make_label("tea", sep=" - "), "item - tea")
        self.assertEqual(make_label(prefix="drink", name="tea"), "drink: tea")

    def test_total_with_tax(self):
        self.assertEqual(total_with_tax([10.0, 5.0], 0.1), 16.5)

    def test_discount_with_floor(self):
        self.assertEqual(apply_discount(100, 10), 90.0)
        self.assertEqual(apply_discount(100, 95, minimum=20), 20.0)

    def test_split_total(self):
        self.assertEqual(split_total(10.0, 3), (3.33, 0.01))

    def test_global_changes_only_through_function(self):
        scope_demo.rate = 0.2
        self.assertEqual(scope_demo.show_rate(), 0.2)
        scope_demo.change_rate()
        self.assertEqual(scope_demo.show_rate(), 0.25)
        scope_demo.rate = 0.2

    def test_local_shadow_does_not_change_builtin(self):
        self.assertEqual(scope_demo.shadow_len(), 3)
        self.assertEqual(len("abc"), 3)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python calculations.py` and paste the output into `docs/findings.md`. Explain in one sentence why `split_total(10.0, 3)` returns two values.
3. Run `python scope_demo.py` and paste the output into `docs/findings.md`. Explain why the second `show_rate` call changed.
4. Run `python -m unittest` and make every test pass.
5. Add a function `with_default_list(item: str, items: list[str] | None = None) -> list[str]` that appends the item to a list, creating a new list when `items` is `None`. Add two tests that call it twice without passing `items`, and check that the second call does not include the first item.
6. Run `python -c "import calculations; help(calculations.apply_discount)"` and copy the first ten lines of the output into `docs/findings.md`.
7. Run `python -m mypy calculations.py scope_demo.py tests/test_calculations.py` and fix every issue.
8. In `docs/findings.md`, write one paragraph explaining why the `global` test resets `scope_demo.rate` at its end, and what would happen if it did not.
9. Commit with the message "Add calculations module with scope and docstring tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least ten tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs of both scripts, the `help` excerpt, and the global-state paragraph
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with the run commands), and `.gitignore`.

The module is light on new syntax and heavy on contracts. A reviewer should be able to read each docstring, check the matching test, and see that the function does what it promises without relying on hidden state.

---

## Check your understanding

1. A function `def add(item, items=[]):` is called twice with one argument each time. Predict the length of the list after the second call and explain how the default is stored. Then write the corrected signature.
2. A developer writes `def save(data):` that prints the file path and returns nothing, and a caller then does `path = save(data)`. Explain what `path` contains and what change makes the function useful to callers.
3. A function reads `total` on its first line and assigns `total = 0` on its second line. Explain the error, and why Python cannot treat the first read as the global `total`.
4. Two functions both use the name `limit`. One assigns it, and the other only reads it. Explain which one changes a global, and what the other one reads.
5. A student says that annotations stop a function from receiving a string. Explain why that is incorrect, and name the tool that would catch the mistake before the program runs.
6. Name the four scopes of LEGB in the order Python searches them, and give one example of a name that is found in the enclosing scope and not in the local one.

### Answers

1. The second call returns a list with two items, because the default list is created once when the `def` runs and shared by every call that omits `items`. The corrected signature is `def add(item, items=None):` with `items = [] if items is None else items` inside the function.
2. `path` is `None`, because the function has no `return`. Make the function return the path it wrote, for example `return path`, so the caller can use the value.
3. Assigning `total` anywhere in the function makes it local for the whole body, so the first read looks in the local scope before the assignment and raises `UnboundLocalError`. Python decides locality at compile time, not by the order of lines, so it cannot fall back to the global.
4. The function that assigns `limit` without a `global` declaration creates a local name and does not change the global. The function that declares `global limit` and assigns changes the global. The reader that only reads `limit` looks through the enclosing and global scopes and sees the current global value.
5. Annotations are not enforced at runtime, so a string is passed through and the function runs. A static type checker such as mypy reads the annotations and reports the mismatch before the code is executed, which is the appropriate place to catch it.
6. The order is Local, Enclosing, Global, Built-in. An example is a variable from an outer function that an inner function reads without assigning, such as `name` in `inner` in the LEGB example, which is not in the local scope of `inner`.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Defining Functions", at docs.python.org/3/tutorial/controlflow.html
- Python language reference, "Function definitions", at docs.python.org/3/reference/compound_stmts.html
- Python language reference, "Naming and binding", at docs.python.org/3/reference/executionmodel.html
- PEP 257, "Docstring Conventions", at peps.python.org/pep-0257/
- PEP 484, "Type Hints", at peps.python.org/pep-0484/
- [Wikipedia: Subroutine](https://en.wikipedia.org/wiki/Subroutine)
- [Wikipedia: Scope (computer science)](https://en.wikipedia.org/wiki/Scope_(computer_science))

*Tutorials*

- GeeksforGeeks, "Python Functions" and "Python Scope of Variables" tutorials, at geeksforgeeks.org
- W3Schools, "Python Functions" and "Python Scope" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the lecture on functions and variables.

*Practice*

- Python Tutor at pythontutor.com. Step through the scope example and the default argument example, and watch the frames change.

---

## Tomorrow

Day 17 extends functions in three directions. You will use default values more carefully, collect variable numbers of arguments with `*args` and `**kwargs`, require some parameters to be keyword-only or positional-only, and see how functions become values that can be passed around. You will meet closures, which keep a value alive after the function that created it returns, and lambda expressions, which are small anonymous functions. Day 16's scope rules are what make closures work.

**Next:** [Day 17: Functions II](Software-Day-17-Functions-II.md)

# Day 17: Functions II

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You wrote functions with parameters, returns, scope and annotations on Day 16. This is for anyone who has seen `*args` in a library signature and wondered what it does, or who needs a function that remembers a setting without a global variable.

**By the end of this day, you will have:**

- A working understanding of default values, including the rule that defaults are evaluated once
- The ability to accept any number of positional and keyword arguments with `*args` and `**kwargs`, and to pass them on correctly
- The ability to require parameters to be positional-only or keyword-only, and to explain why an API would do that
- A correct recursive function with a base case, and the judgment to replace recursion with a loop when depth matters
- Fluency with higher-order functions such as `map`, `filter` and `sorted(key=...)`
- A working closure that keeps state, a lambda used where it helps, and an awareness of the late-binding trap
- A functions toolkit repo with tests that check each of these behaviors

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 10: Objects, Variables and Types (references, the mutable default trap)
- Day 13: Lists and Tuples (unpacking, `sorted`)
- Day 16: Functions I (parameters, arguments, return, scope, LEGB, annotations)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Defaults](#defaults)
  - [Args and kwargs](#args-and-kwargs)
  - [Keyword-only and positional-only](#keyword-only-and-positional-only)
  - [Recursion](#recursion)
  - [Higher-order functions](#higher-order-functions)
  - [Closures](#closures)
  - [Lambda](#lambda)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a functions toolkit](#project-a-functions-toolkit)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Day 16 gave you functions with fixed parameters. Real code needs more flexibility. A logging function must accept any number of values. A formatting function must accept options by name and refuse to accept some of them by position, so that future changes do not break callers. A sorting call must accept a rule for ordering. A rate limiter must remember a count between calls. A tree walker must call itself. Python handles all of these with a small set of features, and each one is a tool you will meet in almost every library you read.

This day covers those features in the order you are likely to need them. Defaults and variable arguments come first, because they shape how a function is called. Keyword-only and positional-only parameters come next, because they let an author protect an API. Recursion, higher-order functions, closures and lambdas then show how a function can be data: passed to others, returned from one, and used to keep state.

The official reference is the Python tutorial section "More on Defining Functions" at docs.python.org/3/tutorial/controlflow.html#more-on-defining-functions, and the language reference section "Function definitions" at docs.python.org/3/reference/compound_stmts.html#function-definitions. The PEP that introduced positional-only parameters is PEP 570, and it is short and worth reading.

---

## Glossary

- **Default value:** a value a parameter takes when the caller omits it. Defaults are evaluated once, at definition time.
- **Variadic parameter:** a parameter that collects any number of extra arguments. `*args` collects positional arguments into a tuple, and `**kwargs` collects keyword arguments into a dictionary.
- **Keyword-only parameter:** a parameter that must be passed by name. It appears after `*` or after `*args` in the signature.
- **Positional-only parameter:** a parameter that must be passed by position. It appears before `/` in the signature.
- **Recursion:** a function that calls itself, with a base case that stops the calls.
- **Base case:** the input for which a recursive function returns without calling itself.
- **Higher-order function:** a function that takes another function as an argument, returns one, or both.
- **Closure:** a function that remembers names from the scope where it was created, even after that scope has finished.
- **Free variable:** a name a function reads from an enclosing scope without assigning it.
- **Lambda:** a small anonymous function written as one expression, such as `lambda x: x * 2`.
- **Late binding:** the behavior in which a closure reads the value of a free variable when it is called, not when it is created.

---

## Concepts

The seven concepts move from how callers pass values, to how an author controls that passing, to how functions become values in their own right.

### Defaults

A default is a fallback answer. Think of a coffee order at a counter. If you say nothing about the size, the barista uses the standard size. The fallback is written on the menu once, and every customer who does not specify gets the same thing. A default parameter works the same way.

From first principles, Python evaluates each default expression when the `def` statement runs, and stores the result on the function object. A later call that omits the argument receives that stored object. For immutable defaults, such as numbers, strings and `None`, this is harmless. For mutable defaults, such as a list or a dictionary, every call shares one object, which makes the default a hidden piece of state.

```python
def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
    if tags is None:
        tags = []
    tags.append(tag)
    return tags


print(add_tag("new"))     # ['new']
print(add_tag("hot"))     # ['hot'], a fresh list each time
```

The `None` sentinel is the standard fix. The function checks for `None` and creates the list inside the call, so no state carries over.

The common mistake is writing `tags: list[str] = []` in the signature, which reads as a type and a default in one line, and then wondering why tags accumulate. The check that finds this bug is simple: any default that is not `None`, a number, a string or a tuple deserves a second look. Linters such as Ruff flag mutable default arguments with a specific rule. In the job market, this bug is among the first things interviewers test, because it reveals whether a candidate understands when Python runs code.

### Args and kwargs

`*args` and `**kwargs` let a function accept a variable number of inputs. Think of a shared dinner bill. Some people order two dishes, some order one, and the waiter adds everything up without knowing the number in advance. `*args` is the list of dishes, and `**kwargs` is the list of named extras, such as "tip" or "discount".

From first principles, inside a function, `args` is a tuple of the extra positional arguments, and `kwargs` is a dictionary of the extra keyword arguments that did not match a named parameter. The names `args` and `kwargs` are conventions; the `*` and `**` are what matter. On the caller's side, `f(*values)` unpacks a sequence into positional arguments, and `f(**options)` unpacks a dictionary into keyword arguments. That is the same unpacking you met on Day 13, applied to calls.

```python
def total(*numbers: float) -> float:
    return sum(numbers)


def describe(name: str, **details: str) -> str:
    parts = [f"{key}={value}" for key, value in sorted(details.items())]
    return name if not parts else f"{name} " + ", ".join(parts)


values = [1, 2, 3]
print(total(*values))                       # 6
print(describe("order", id="A1", status="paid"))  # order id=A1, status=paid
print(describe("order"))                    # order
```

The caller unpacks `values` with `*`, so the function receives three arguments. The `describe` function collects the named details into a dictionary.

The common mistake is forwarding arguments incorrectly in a wrapper, for example by dropping `**kwargs` and so silently losing options. Always pass `*args` and `**kwargs` through when you write a wrapper, unless you have a reason to filter them. The second mistake is using `**kwargs` as a substitute for a clear signature. A function that accepts any keyword and does not document them is hard to call correctly. In the job market, wrapper functions such as decorators and logging helpers use `*args` and `**kwargs` heavily, and reviewers check that they forward everything.

### Keyword-only and positional-only

Some parameters should be passed in one specific way. Think of a form at a government office. Some fields must be written in a fixed box, where the position tells the clerk what the value means. Other fields must be labeled, because a bare number on a line would be unclear. Python lets an author mark either kind.

From first principles, a `/` in the signature means every parameter before it is positional-only, so callers cannot pass it by keyword. A `*` in the signature means every parameter after it is keyword-only, so callers must name it. Positional-only parameters let an author rename a parameter later without breaking callers who used the position. Keyword-only parameters make the meaning of a flag visible at the call site, and they let the author add optional parameters later without changing the order of existing ones.

```python
def format_price(amount: float, /, *, currency: str = "USD") -> str:
    return f"{amount:.2f} {currency}"


print(format_price(4.5, currency="EUR"))   # 4.50 EUR
print(format_price(4.5))                   # 4.50 USD

try:
    format_price(amount=4.5)
except TypeError as err:
    print("error:", err)
```

The `amount` parameter cannot be passed by keyword because of the `/`. The `currency` parameter must be passed by keyword because of the `*`. Both rules show up in the error messages.

The common mistake is trying to use `/` or `*` in a function that is called by keyword in the existing code. The call fails with `TypeError`. Check the call sites before adding these markers to a public function. The second mistake is using positional-only parameters to hide an argument that you still want readers to see. The markers are a contract, and they belong on parameters whose names carry no meaning for callers. In the job market, library authors use these markers on public APIs, and you will see them in standard library signatures such as `len`.

### Recursion

Recursion is a function that calls itself. Think of a set of nested boxes, where each box contains a smaller box, until you reach an empty box that stops the search. The function describes the work for one box, and it calls itself to open the next box. The empty box is the base case.

From first principles, each call to a recursive function creates a new frame on the call stack, and the frame holds its parameters and local names. Python stops the chain of frames at a limit, set by `sys.getrecursionlimit()`, which is 1000 by default. A recursive function must reach a base case for every input, and it must move toward that base case on each call. Otherwise it raises `RecursionError`. Recursion is a clear way to express tree-shaped work, such as walking a folder tree, but a loop is the better choice for a simple counting task with large inputs.

```python
def factorial(n: int) -> int:
    if n < 0:
        raise ValueError("n must be zero or greater")
    if n <= 1:
        return 1
    return n * factorial(n - 1)


def sum_digits(n: int) -> int:
    if n < 0:
        raise ValueError("n must be zero or greater")
    if n < 10:
        return n
    return n % 10 + sum_digits(n // 10)


print(factorial(5))        # 120
print(sum_digits(1234))    # 10
```

Each call moves toward the base case: `n - 1` for the factorial, and `n // 10` for the digits. The guard for negative input stops the function from calling itself forever with values that never reach the base.

The common mistake is a recursive call that does not move toward the base case, such as calling `factorial(n)` instead of `factorial(n - 1)`. The stack grows until `RecursionError` appears. The second mistake is recursing over a long input, such as a list of a hundred thousand items, where a loop would be safer. Python does not optimize tail calls, so each level of recursion uses stack space. In the job market, recursion appears in tree algorithms, parsers and divide-and-conquer code, and interviewers ask candidates to convert a recursive solution to a loop to test whether they understand the stack.

### Higher-order functions

A higher-order function is a function that takes another function as an input, returns one as an output, or does both. Think of a factory that accepts a machine design and produces a machine. The factory takes the design as input and builds the machine it describes, without needing to know what the machine does. In Python, functions are objects, so you can pass them around like any other value.

From first principles, `map(func, iterable)` applies `func` to each item and returns an iterator of results. `filter(func, iterable)` keeps items for which `func` returns a true value. `sorted(iterable, key=func)` uses `func` to compute a sort key for each item. The built-in functions are all higher-order, and so are many of the decorators you will meet in larger projects. A list comprehension often does the same job as `map` and `filter`, and Python programmers choose whichever reads better.

```python
def apply_all(funcs, value):
    return [f(value) for f in funcs]


def double(x: float) -> float:
    return x * 2


def halve(x: float) -> float:
    return x / 2


print(apply_all([double, halve], 10))      # [20, 5.0]
print(list(map(double, [1, 2, 3])))        # [2, 4, 6]
print(list(filter(lambda n: n > 1, [0, 1, 2, 3])))  # [2, 3]
```

The `apply_all` function takes a list of functions and applies each one to the same value. That pattern appears in validation pipelines, where each check is a function.

The common mistake is calling `map` or `filter` and printing the result without converting it, which shows an iterator object instead of the values. Use `list(...)` or a comprehension when you need the values more than once. Another common mistake is treating `filter` and a comprehension as the same in performance, when the difference is negligible for most work. Choose by readability. In the job market, data processing code uses `map` and `filter` in generator pipelines, and understanding that they are lazy is important for memory use.

### Closures

A closure is a function that remembers the variables of the scope where it was created. Think of a vending machine with a built-in price setting. The machine is built once with a price, and every sale uses that price, long after the technician who set it has gone home. The inner function carries the setting with it.

From first principles, when an inner function refers to a name from the enclosing function, Python stores the name in a cell object. The inner function keeps a reference to that cell, so the value survives after the enclosing function returns. You can see the stored names in `func.__closure__`. The closure reads the current value of the cell when it is called, which means later changes to the enclosing variable are visible to it. That behavior is late binding.

```python
def make_multiplier(factor: float):
    def multiply(value: float) -> float:
        return value * factor
    return multiply


triple = make_multiplier(3)
double = make_multiplier(2)
print(triple(10), double(10))     # 30 20
print(triple.__closure__[0].cell_contents)  # 3
```

Each call to `make_multiplier` creates a new cell, so `triple` and `double` keep separate factors. The state lives in the closure rather than in a global variable, which is the design the Day 16 scope lesson recommended.

The common mistake is the late-binding trap in a loop. A list of lambdas built with `[lambda x: x + i for i in range(3)]` all refer to the same variable `i`, which ends at 2, so every lambda returns `x + 2`. The fix is to bind the current value with a default argument, as in `lambda x, i=i: x + i`, or to create the function inside a helper that takes `i` as a parameter. Test closures created in loops with more than one input, because the bug only shows when the loop variable has changed. In the job market, callbacks registered in loops are a common source of this bug in event-driven code.

### Lambda

A lambda is a function written as an expression. Think of a sticky note with a one-line instruction, such as "sort by length". The note is useful for a short job, and it is thrown away when the job is done. A lambda is a function without a name, which holds one expression, and which Python returns as a value.

From first principles, `lambda parameters: expression` creates a function object. The body is a single expression, so a lambda cannot contain statements, such as assignments, loops or `return`. Its result is the value of the expression. A lambda can be assigned to a name, but the style guide PEP 8 recommends a `def` in that case, because a named `def` shows its name in tracebacks and can carry a docstring.

```python
def sort_by_length(words: list[str]) -> list[str]:
    return sorted(words, key=lambda word: (len(word), word))


print(sort_by_length(["pear", "fig", "banana"]))   # ['fig', 'pear', 'banana']
```

The lambda is the key function for `sorted`. It returns a tuple, so words of the same length are ordered alphabetically, which is a common trick for stable, predictable sorting.

The common mistake is writing a lambda that is long, has a condition chain, or needs a docstring. Then it is harder to read than a `def`. Use a named function for anything that needs more than one line of logic. The second mistake is assigning a lambda to a name and using it as a regular function, which hides the name in tracebacks. In the job market, lambdas appear in key functions and small callbacks, and reviewers expect them to stay short and simple.

---

## Hands-on exercises

Work in a folder called `functions-ii-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices defaults and the `None` sentinel.

1. Create `defaults.py`. Write `add_tag(tag, tags=None)` that appends to a list, creating a new list when `tags` is `None`.
2. Write the broken version with `tags=[]` in a second function, call it twice, and record the output.
3. Explain in one sentence why the two versions behave differently.

Done when the fixed version returns a single-item list on each call and you can describe where the shared default lives.

Exercise 2 practices `*args` and `**kwargs`.

1. Create `varargs.py`. Write `total(*numbers)` that returns the sum.
2. Write `describe(name, **details)` that returns the name followed by the sorted details.
3. Call `total` with a list unpacked using `*`, and call `describe` with a dictionary unpacked using `**`.

Done when the unpacking calls work and your output matches your prediction.

Exercise 3 practices keyword-only and positional-only parameters.

1. Create `markers.py`. Write `format_price(amount, /, *, currency="USD")`.
2. Call it with and without the `currency` keyword. Then try `format_price(amount=4.5)` inside a `try` block and record the error.
3. Write one sentence explaining what the `/` and `*` each prevent.

Done when the error message confirms the positional-only rule.

Exercise 4 practices recursion and closures.

1. Create `recursion.py`. Write `factorial(n)` and `sum_digits(n)` recursively with base cases and a guard for negative inputs.
2. Call `factorial(5000)` in a `try` block, catch `RecursionError`, and record what happens.
3. Write `make_counter()` that returns a function which increments and returns its own count each time it is called. Call it three times and record the results.

Done when the recursion error appears for the large input, and the counter returns 1, 2 and 3.

Exercise 5 practices closures in loops.

1. Create `late_binding.py`. Build a list of three functions with `[lambda x: x + i for i in range(3)]`, and call each one with the value 10.
2. Build the same list with a default argument `lambda x, i=i: x + i`, and call each one with the value 10.
3. Run `python -m mypy late_binding.py` and read whether mypy reports any issue. Write one sentence on why mypy does not catch this bug.

Done when the first list returns 12 three times and the second returns 10, 11 and 12.

---

## Project: a functions toolkit

The project is a small toolkit of function patterns: a total that accepts any number of values, a describe function with keyword details, a formatter with positional-only and keyword-only parameters, recursive math with a base case, a counter closure, and a sort helper with a lambda key. Each function is a pattern you will meet in real code. The evidence is a test suite that covers each pattern and a findings file that records the recursion limit result.

Suggested repo name: `100-days-software-day-17-functions-ii`.

Repo layout:

```
100-days-software-day-17-functions-ii/
├── .gitignore
├── README.md
├── toolkit.py
├── late_binding.py
├── tests/
│   ├── __init__.py
│   └── test_toolkit.py
└── docs/
    └── findings.md
```

Starter code for `toolkit.py`:

```python
from collections.abc import Callable, Iterable


def total(*numbers: float) -> float:
    return sum(numbers)


def describe(name: str, **details: str) -> str:
    parts = [f"{key}={value}" for key, value in sorted(details.items())]
    return name if not parts else f"{name} " + ", ".join(parts)


def format_price(amount: float, /, *, currency: str = "USD") -> str:
    return f"{amount:.2f} {currency}"


def make_multiplier(factor: float) -> Callable[[float], float]:
    def multiply(value: float) -> float:
        return value * factor

    return multiply


def apply_all(funcs: Iterable[Callable[[float], float]], value: float) -> list[float]:
    return [f(value) for f in funcs]


def factorial(n: int) -> int:
    if n < 0:
        raise ValueError("n must be zero or greater")
    if n <= 1:
        return 1
    return n * factorial(n - 1)


def sum_digits(n: int) -> int:
    if n < 0:
        raise ValueError("n must be zero or greater")
    if n < 10:
        return n
    return n % 10 + sum_digits(n // 10)


def make_counter() -> Callable[[], int]:
    count = 0

    def increment() -> int:
        nonlocal count
        count += 1
        return count

    return increment


def sort_by_length(words: list[str]) -> list[str]:
    return sorted(words, key=lambda word: (len(word), word))


if __name__ == "__main__":
    print(total(1, 2, 3))
    print(describe("order", id="A1", status="paid"))
    print(format_price(4.5, currency="EUR"))
    double = make_multiplier(2)
    print(apply_all([double, make_multiplier(0.5)], 10))
    print(factorial(5), sum_digits(1234))
    counter = make_counter()
    print(counter(), counter())
    print(sort_by_length(["pear", "fig", "banana"]))
```

Starter code for `late_binding.py`:

```python
def broken_adders():
    return [lambda x: x + i for i in range(3)]


def fixed_adders():
    return [lambda x, i=i: x + i for i in range(3)]


if __name__ == "__main__":
    print([f(10) for f in broken_adders()])
    print([f(10) for f in fixed_adders()])
```

Starter code for `tests/test_toolkit.py`:

```python
import unittest

from late_binding import broken_adders, fixed_adders
from toolkit import (
    describe,
    factorial,
    format_price,
    make_counter,
    make_multiplier,
    sort_by_length,
    sum_digits,
    total,
)


class ToolkitTests(unittest.TestCase):
    def test_total_accepts_any_count(self):
        self.assertEqual(total(), 0)
        self.assertEqual(total(1, 2, 3), 6)

    def test_describe_with_and_without_details(self):
        self.assertEqual(describe("order"), "order")
        self.assertEqual(describe("order", status="paid", id="A1"), "order id=A1, status=paid")

    def test_positional_only_and_keyword_only(self):
        self.assertEqual(format_price(4.5), "4.50 USD")
        self.assertEqual(format_price(4.5, currency="EUR"), "4.50 EUR")
        with self.assertRaises(TypeError):
            format_price(amount=4.5)

    def test_closure_keeps_its_own_factor(self):
        triple = make_multiplier(3)
        double = make_multiplier(2)
        self.assertEqual(triple(10), 30)
        self.assertEqual(double(10), 20)

    def test_factorial_and_digits(self):
        self.assertEqual(factorial(0), 1)
        self.assertEqual(factorial(5), 120)
        self.assertEqual(sum_digits(1234), 10)

    def test_negative_input_rejected(self):
        with self.assertRaises(ValueError):
            factorial(-1)

    def test_recursion_limit_on_large_input(self):
        with self.assertRaises(RecursionError):
            factorial(5000)

    def test_counter_increments(self):
        counter = make_counter()
        self.assertEqual([counter(), counter(), counter()], [1, 2, 3])

    def test_sort_by_length_then_alpha(self):
        self.assertEqual(sort_by_length(["pear", "fig", "banana"]), ["fig", "pear", "banana"])

    def test_late_binding_trap_and_fix(self):
        self.assertEqual([f(10) for f in broken_adders()], [12, 12, 12])
        self.assertEqual([f(10) for f in fixed_adders()], [10, 11, 12])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python toolkit.py` and paste the output into `docs/findings.md`. Explain in one sentence why the two counter calls print `1` and `2`.
3. Run `python late_binding.py` and paste the output into `docs/findings.md`. Explain why the broken version returns the same value three times.
4. Run `python -m unittest` and make every test pass.
5. Add a function `pipeline(value: float, *steps: Callable[[float], float]) -> float` that applies each step in order. Add a test with two steps.
6. Run `python -c "import sys; print(sys.getrecursionlimit())"` and record the value in `docs/findings.md`. Then change `factorial` to an iterative version named `factorial_loop` and add a test that it handles `factorial_loop(5000)` without error. Explain in `docs/findings.md` why the loop version does not hit the limit.
7. Run `python -m mypy toolkit.py late_binding.py tests/test_toolkit.py` and fix every issue.
8. Write one paragraph in `docs/findings.md` on the difference between a default argument used for a mutable object and the `nonlocal` state in `make_counter`.
9. Commit with the message "Add functions toolkit with closures and argument tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least eleven tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs, the recursion limit, the late-binding explanation, and the paragraph on state
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with run commands), and `.gitignore`.

The toolkit is small on purpose. A reviewer should be able to read each function's signature and see which callers are allowed, which arguments are required, and where the state lives.

---

## Check your understanding

1. A function `def log(event, history=[]):` is called three times with different events. Predict what `history` contains after the third call and explain the mechanism. Write the fixed signature.
2. A wrapper function is defined as `def wrapper(*args):` and it calls the wrapped function with `func(args)`. Explain what goes wrong, and write the forwarding call that fixes it.
3. A library function is declared `def connect(host, port, timeout):`. A maintainer wants to rename `timeout` to `timeout_seconds` without breaking callers who pass it by name. Explain why the rename breaks those callers, and name the signature change that would let the maintainer rename a parameter safely if callers used positions.
4. A recursive function `count_down(n)` has the base case `if n == 0: return` and calls `count_down(n + 1)`. Explain why it fails, and rewrite the recursive call.
5. Explain why a closure created in a loop, such as `[lambda x: x + i for i in range(3)]`, returns the same result for each function, and name two ways to fix it.
6. A developer writes `sorted(words, key=lambda w: w.lower())` and then changes it to a named helper with a docstring. Explain why the second version can be better even though both work, in terms of what a reader and a traceback see.

### Answers

1. After three calls, `history` contains all three events, because the default list is created once and shared. The mechanism is the same as the shared mutable default in Day 10. The fixed signature is `def log(event, history=None):` with `history = [] if history is None else history` inside the function.
2. `func(args)` passes the tuple as one argument, so the wrapped function receives a tuple where it expected separate values. The forwarding call is `func(*args)`, and the wrapper should also accept and forward `**kwargs` when the wrapped function accepts keyword arguments.
3. Renaming a parameter removes the old name from the signature, so a caller that passes `timeout=5` gets a `TypeError`. The safe change, if callers should use positions, is to mark the parameters before `/` as positional-only, which lets the name change without breaking those callers. For keyword callers, the maintainer would add the new name and keep the old one as a deprecated keyword for a release.
4. The base case is `n == 0`, and the calls move `n` away from zero, so the recursion never stops, and Python raises `RecursionError`. The recursive call should move toward the base case, which for counting down is `count_down(n - 1)` with a base case that returns for `n <= 0`.
5. Each lambda refers to the same variable `i` in the enclosing scope, and by the time they are called, `i` is 2. Two fixes are a default argument such as `lambda x, i=i: x + i`, which binds the current value at creation, and a helper function that takes `i` as a parameter and returns the lambda.
6. The named helper has a name that appears in tracebacks, so a failure shows `lower_key` rather than `<lambda>`, and it can carry a docstring that explains the rule. The lambda is fine for one short expression, but a named function documents intent and is easier to test directly.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "More on Defining Functions", at docs.python.org/3/tutorial/controlflow.html#more-on-defining-functions
- Python language reference, "Function definitions", at docs.python.org/3/reference/compound_stmts.html#function-definitions
- PEP 570, "Python Positional-Only Parameters", at peps.python.org/pep-0570/
- PEP 3102, "Keyword-Only Arguments", at peps.python.org/pep-3102/
- Python documentation, module `sys`, for `getrecursionlimit`, at docs.python.org/3/library/sys.html
- [Wikipedia: Closure (computer programming)](https://en.wikipedia.org/wiki/Closure_(computer_programming))
- [Wikipedia: Higher-order function](https://en.wikipedia.org/wiki/Higher-order_function)
- [Wikipedia: Recursion (computer science)](https://en.wikipedia.org/wiki/Recursion_(computer_science))

*Tutorials*

- GeeksforGeeks, "Args and Kwargs in Python", "Python Lambda Functions", and "Python Closures" tutorials, at geeksforgeeks.org
- W3Schools, "Python Lambda" and "Python Recursion" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on functions and recursion.

*Practice*

- Python Tutor at pythontutor.com. Step through the closure example and the late-binding example, and watch which variable each function reads.

---

## Tomorrow

Day 18 moves from functions to the organization of code across files. You will learn how an `import` statement finds a module, how a module's namespace works, what `__name__` means when a file runs or is imported, and how packages group modules into folders. You will also meet relative and absolute imports. The functions you wrote today will live in modules, and the tests you wrote will import them.

**Next:** [Day 18: Modules and Packages](Software-Day-18-Modules-and-Packages.md)

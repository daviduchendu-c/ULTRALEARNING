# Day 11: Numbers and Operators

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner. You are comfortable with variables and functions from Day 9 and Day 10. This is for anyone who has written a price calculator and watched a total come out as `10.499999999`, or who has wondered why `-7 // 2` is not `-3`.

**By the end of this day, you will have:**

- A clear model of Python integers, which have arbitrary size and exact arithmetic
- An understanding of floats, their limits, and the tools that compare them correctly
- The ability to choose `Decimal` or `Fraction` when money or exact ratios matter
- Confident use of arithmetic, comparison, logical and bitwise operators, and their precedence
- A small money-and-math repo with tests that catch rounding bugs before they reach a user

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- An editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed in the project with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 9: Python Runtime and Syntax (running scripts, expressions and statements)
- Day 10: Objects, Variables and Types (identity, types, truthiness)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Integers](#integers)
  - [Floats](#floats)
  - [Precision](#precision)
  - [Booleans](#booleans)
  - [Conversions](#conversions)
  - [Operators](#operators)
  - [Precedence](#precedence)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a money and number lab](#project-a-money-and-number-lab)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Numbers look like the easy part of programming, and they are until a checkout total is one cent off, or a sensor reading jumps between two values that should be equal. Computers store numbers in finite binary form, so some decimal values cannot be represented exactly. Integers in Python do not have that limit, but floats do. Operators follow rules that differ from the ones you learned in school, such as `-2 ** 2` being `-4`. Each of these facts is small. Together they decide whether the numbers in your program are the numbers you meant.

This day covers the numeric types that Python gives you, how they convert between each other, the operators that act on them, and the precedence rules that decide what an expression means. It ends with a practical choice: when `float` is fine, and when money needs `Decimal`. That choice comes up in almost every paid software job that touches finance, pricing, billing or analytics.

The official reference is the Python documentation section "Numeric Types (int, float, complex)" at docs.python.org/3/library/stdtypes.html, together with the reference chapter on expressions at docs.python.org/3/reference/expressions.html. Read the operator precedence table in the second source once. You will return to it.

---

## Glossary

- **int:** an integer type with arbitrary precision. It can grow as large as memory allows.
- **float:** a binary floating-point number, usually 64 bits, following the IEEE 754 standard.
- **Precision:** how many significant digits a number type can represent exactly.
- **Rounding error:** the difference between the exact decimal value you wanted and the nearest value the type can store.
- **bool:** a boolean type. In Python, `bool` is a subclass of `int`, with values `True` (1) and `False` (0).
- **Conversion:** turning a value of one type into another, such as `int("42")` or `float(3)`.
- **Operator:** a symbol that combines values, such as `+`, `//`, `and` or `<<`.
- **Precedence:** the rule that decides which operator groups first when several appear in one expression.
- **Associativity:** the rule that decides grouping when operators have equal precedence. Most are left-to-right, and `**` is right-to-left.
- **Decimal:** a type in the standard library module `decimal` that stores base-10 numbers with a set precision.
- **Fraction:** a type in the module `fractions` that stores an exact ratio of two integers.
- **Bitwise operator:** an operator that works on the binary digits of integers, such as `&`, `|`, `^` and `<<`.

---

## Concepts

The concepts build from the simplest type to the decisions that matter in production. Integers come first, then floats and their precision, then booleans, conversions and the operators that act on all of them.

### Integers

Think of a set of mechanical counters on a bike. Each wheel has digits from 0 to 9, and when one wheel passes 9, the next one advances. A counter in a computer works the same way, except it counts in base 2, where each wheel holds 0 or 1. Python's `int` is a counter that adds more wheels when it needs them, so it never overflows the way a fixed-size counter does.

From first principles, Python stores each `int` as a sequence of digits in a base close to 2^30, plus a size and a sign. The arithmetic follows the usual laws exactly, with no rounding. You can write numbers with underscores for readability, such as `1_000_000`, and you can write them in binary, octal and hexadecimal with the prefixes `0b`, `0o` and `0x`.

```python
big = 2 ** 200
print(big)                # a 61-digit exact number
print(big % 97)           # exact modular arithmetic
print(1_000_000 == 1000000, 0b1010, 0o17, 0xFF)  # True 10 15 255
```

The output shows the exact value of 2 to the power 200, followed by its remainder modulo 97. Python computes both exactly, because the type has no fixed size.

The common mistake is assuming that integers in Python behave like the fixed-size `int` in C or Java. They do not overflow, which is a feature, but it means memory use grows with the size of the number, and very large loops can be slow. In data work, the same thinking applies to counts that grow beyond what a database column can store, so engineers pick explicit widths when they write to storage. Knowing that Python integers are unbounded helps you read those decisions.

### Floats

A float is a number written in scientific notation with a fixed number of digits. Picture a measuring tape marked only at every millimeter. It can tell you a length to the nearest millimeter, and anything in between gets rounded to the nearest mark. A float is a tape with marks at powers of two, and most decimal numbers fall between the marks.

From first principles, a Python `float` is a 64-bit IEEE 754 double. It stores a sign, an exponent and a 53-bit significand. Numbers such as `0.5` and `0.25` are exact because they are sums of powers of two. Numbers such as `0.1` and `0.2` are not, so the stored value is the nearest representable number. The float also has special values, `inf`, `-inf` and `nan`, which come from overflow, division by zero in some libraries, and invalid operations.

```python
import sys

print(sys.float_info.max)     # the largest finite float
print(sys.float_info.epsilon) # the gap between 1.0 and the next float
print(float("inf") > 10**308) # True
print(float("nan") == float("nan"))  # False, NaN is not equal to itself
```

The last line is the surprise most people meet first. `nan` is defined to be unequal to everything, including itself, so the check for `nan` must use `math.isnan`.

The common mistake is treating a float as exact because it prints neatly. Python shows the shortest decimal string that round-trips to the same float, so `print(0.1)` looks exact, but the stored value is still an approximation. Floats are a good fit for measurements, scientific computation and graphics, where the error is far below what matters. They are the wrong tool for money, where a one-cent error is a real bug.

### Precision

Precision is the number of significant digits a type keeps. A float keeps about 15 to 17 significant decimal digits. An `int` keeps all of its digits. A `Decimal` keeps as many as you tell it to, which is useful when you need a fixed decimal result. Precision is not accuracy. A value can be stored with many digits and still be wrong, if the calculation that produced it lost information.

From first principles, floating-point addition is not associative. Adding a very large number and a very small one in a different order gives a different result, because the small value is rounded away at the first step. Summing a long list of floats with `sum` can therefore give a result that depends on the order of the items. `math.fsum` uses a more careful algorithm and returns the correctly rounded sum.

```python
import math
from decimal import Decimal, getcontext

values = [1e16, 1.0, -1e16]
print(sum(values))         # 0.0, the 1.0 was absorbed
print(math.fsum(values))   # 1.0

getcontext().prec = 28
print(Decimal("0.1") + Decimal("0.2"))  # 0.3
```

The first two lines show the loss of precision, and the last shows that `Decimal` keeps the decimal digits you wrote. The `getcontext().prec` setting is the default precision for `Decimal` arithmetic, and 28 is the default.

The common mistake is comparing floats for exact equality after a calculation. `0.1 + 0.2 == 0.3` is `False`. Use `math.isclose` with a tolerance that fits the problem. In the job market, a surprising number of pricing and tax bugs come from this exact line, and the fix is an explicit decision about the number type.

### Booleans

A boolean is a switch with two positions. It is useful for decisions, and in Python it has a second life as a number. `True` behaves as 1 and `False` as 0 in arithmetic, because `bool` is a subclass of `int`. That design is convenient, and it is also a source of confusion.

From first principles, `True` and `False` are the only two instances of `bool`, and they are singletons, like `None`. `isinstance(True, int)` is `True`, which means that a function that checks for `int` will accept a boolean unless it rules that out. Comparisons such as `==`, `<` and `in` return `bool` values, and the logical operators `and`, `or` and `not` also work on any objects, returning one of the operands rather than necessarily a `bool`.

```python
print(True + True)            # 2
print(isinstance(True, int))  # True
print(3 > 2 and "yes")        # yes, not True
print(0 or "fallback")        # fallback
```

The `and` and `or` operators return the operand that decided the result, which is the basis of short-circuit evaluation. Day 15 covers that in control flow.

The common mistake is letting a boolean slip into a numeric total. `sum([True, False, True])` returns `2`, which is correct for counting, but a value that was meant to be a rating or a price becomes a switch value silently. Type checkers flag this when annotations are present, and `mypy` treats `bool` as a subtype of `int`, so the check is about intent, not a hard rule. Reviewers in the job market look for explicit conversions at the boundary of a system.

### Conversions

A conversion turns a value into another type. Think of an adapter plug. It lets a device fit a socket it was not built for, and it works only when the two sides share a standard. `int("42")` works because the text is a valid integer literal. `int("4.2")` fails because the text is not, even though the number is real.

From first principles, the built-in constructors `int`, `float`, `str` and `bool` each implement a conversion. `int(x)` truncates a float toward zero, so `int(3.9)` is `3` and `int(-3.9)` is `-3`. `float(x)` converts an integer to a float, which may round if the integer is large. `round(x)` rounds to the nearest integer, using banker's rounding (round half to even), so `round(2.5)` is `2`, and `round(3.5)` is `4`.

```python
print(int("42") + 1)        # 43
print(int(-3.9))            # -3, truncates toward zero
print(round(2.5), round(3.5))  # 2 4
print(float(10) / 4)        # 2.5
print(int("4.2"))           # ValueError
```

The last line raises `ValueError: invalid literal for int() with base 10: '4.2'`. To convert text such as `"4.2"` to an integer, first convert it to a float and then decide how to round it.

The common mistake is using `int()` for rounding. It truncates. A value of `9.99` becomes `9`, which can cut a customer's refund or a sensor's reading by a whole unit. Use `round()`, `math.floor()` or `math.ceil()` with a clear intent. Conversion errors in parsing user input are common in job interviews and in real forms, and the fix is always to catch the `ValueError` at the boundary and give the user a clear message.

### Operators

Operators are the symbols that combine values. Python has arithmetic operators (`+`, `-`, `*`, `/`, `//`, `%`, `**`), comparison operators (`==`, `!=`, `<`, `<=`, `>`, `>=`, `is`, `in`), logical operators (`and`, `or`, `not`), and bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`). Think of them as the verbs of an expression, and the operands as the nouns.

From first principles, `/` always returns a float, even for two integers, so `6 / 3` is `2.0`. `//` is floor division, which rounds toward negative infinity, so `-7 // 2` is `-4`. `%` is the remainder that matches floor division, so `a == (a // b) * b + a % b` holds for all integers. `divmod(a, b)` returns both results at once. `**` raises to a power, and a negative exponent gives a float.

```python
print(7 / 2, 7 // 2, 7 % 2)   # 3.5 3 1
print(-7 // 2, -7 % 2)        # -4 1
print(divmod(-7, 2))          # (-4, 1)
print(5 & 3, 5 | 3, 5 ^ 3)    # 1 7 6
print(1 << 4, 256 >> 2)       # 16 64
```

The bitwise line uses binary digits. `5` is `101`, `3` is `011`, so `&` keeps the bits set in both (`001`), `|` keeps bits set in either (`111`), and `^` keeps bits set in exactly one (`110`).

The common mistake is expecting `//` and `%` to behave the way they do in C or Java for negative numbers. In Python they round toward negative infinity, so `-7 // 2` is `-4`, not `-3`. This matters for time arithmetic, array indexing and pagination. Write a test with a negative value whenever you use these operators on data that can be negative.

### Precedence

Precedence decides which operator is applied first when several are written without parentheses. Think of a recipe with several steps written in one sentence. Without punctuation, the reader cannot tell which step comes first. Parentheses are the punctuation of expressions.

From first principles, Python's precedence from highest to lowest runs roughly as follows: parentheses, then `**`, then unary `+`, `-` and `~`, then `*`, `/`, `//` and `%`, then `+` and `-`, then the bitwise shifts and `&`, `^`, `|`, then comparisons and membership, then `not`, then `and`, then `or`. The exact table is in the Python reference, and reading it once is worth the time. Two details surprise people: `**` binds tighter than a unary minus on its left, so `-2 ** 2` is `-(2 ** 2)`, and `**` is right-associative, so `2 ** 3 ** 2` is `2 ** 9`.

```python
print(-2 ** 2)          # -4
print((-2) ** 2)        # 4
print(2 ** 3 ** 2)      # 512
print(1 + 2 * 3 == 7)   # True
print(not 1 == 2)       # True, because not binds looser than ==
```

The common mistake is writing a long expression that depends on precedence, and then having a reviewer misread it. The fix is to add parentheses for every mixed group, even when Python does not need them. Explicit grouping costs nothing at runtime, and it makes intent visible. Code review comments asking for parentheses are routine, and clear grouping is part of the professional habit.

---

## Hands-on exercises

Work in a folder called `numbers-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 explores integers.

1. Create `ints.py`. Print `10 ** 30`, `10 ** 30 // 7` and `10 ** 30 % 7`.
2. Print `0b1111`, `0o777` and `0xDEAD`.
3. Write one sentence on what would happen in a language with 64-bit fixed-size integers when you compute `10 ** 30`.

Done when the three results match the exact values you expect and your sentence explains the fixed-size case.

Exercise 2 shows the float surprise.

1. Create `floats.py`. Print `0.1 + 0.2`, `0.1 + 0.2 == 0.3` and `math.isclose(0.1 + 0.2, 0.3)`.
2. Print `sum([0.1] * 10)` and `math.fsum([0.1] * 10)`. Compare them with `1.0`.
3. Write two sentences describing when a tolerance comparison is right and when it is a sign that the wrong type is in use.

Done when you can explain each printed line in one sentence.

Exercise 3 checks conversion behavior.

1. Create `convert.py`. Write a function `parse_quantity(text: str) -> int` that converts text to an integer, raising a `ValueError` with a clear message for input such as `"4.2"` or `"abc"`.
2. Write a function `round_price(value: float) -> int` that uses `round` and returns the nearest whole number.
3. Test both functions with `"12"`, `"4.2"`, `"abc"`, `2.5` and `3.5`. Record the results.

Done when `round_price(2.5)` returns `2` and `round_price(3.5)` returns `4`, and invalid text produces a readable error.

Exercise 4 checks operator precedence.

1. Create `precedence.py`. Write five expressions that mix arithmetic, comparison and logical operators, and print each one.
2. For each expression, write the result you predicted before running the file.
3. Rewrite each expression with explicit parentheses and confirm the output is unchanged.

Done when every prediction matches the output or you have found and explained the difference.

Exercise 5 runs the type checker on conversions.

1. Install `mypy` with `python -m pip install mypy`.
2. Write `total.py` with a function `add_tax(price: float, rate: float) -> float`. Call it once with a string argument.
3. Run `python -m mypy total.py`, read the error, fix the call, and rerun until mypy reports `Success: no issues found`.

Done when the final run is clean and you can explain what mypy caught.

---

## Project: a money and number lab

The project is a small money-and-math library with tests. It parses amounts with `Decimal`, splits a total into cents without losing a penny, and demonstrates the float problems side by side with the correct tools. The code is short because the discipline is what matters. The evidence is a passing test suite and a findings document in which you record each rounding decision.

Suggested repo name: `100-days-software-day-11-numbers-and-operators`.

Repo layout:

```
100-days-software-day-11-numbers-and-operators/
├── .gitignore
├── README.md
├── money.py
├── float_checks.py
├── operators_demo.py
├── tests/
│   ├── __init__.py
│   └── test_money.py
└── docs/
    └── findings.md
```

Starter code for `money.py`:

```python
from decimal import ROUND_DOWN, ROUND_HALF_UP, Decimal, InvalidOperation

CENT = Decimal("0.01")


def parse_amount(text: str) -> Decimal:
    try:
        value = Decimal(text)
    except InvalidOperation as exc:
        raise ValueError(f"not a decimal amount: {text!r}") from exc
    if not value.is_finite():
        raise ValueError(f"amount must be finite: {text!r}")
    return value.quantize(CENT, rounding=ROUND_HALF_UP)


def add_amounts(*amounts: Decimal) -> Decimal:
    return sum(amounts, Decimal("0")).quantize(CENT)


def format_money(amount: Decimal) -> str:
    return f"${amount:,.2f}"


def split_evenly(total: Decimal, parts: int) -> list[Decimal]:
    if parts <= 0:
        raise ValueError("parts must be positive")
    share = (total / parts).quantize(CENT, rounding=ROUND_DOWN)
    shares = [share] * parts
    remainder = total - share * parts
    step = CENT if remainder >= 0 else -CENT
    index = 0
    while remainder != 0:
        shares[index] += step
        remainder -= step
        index += 1
    return shares


if __name__ == "__main__":
    total = parse_amount("10.00")
    pieces = split_evenly(total, 3)
    print([format_money(p) for p in pieces], format_money(add_amounts(*pieces)))
```

Starter code for `float_checks.py`:

```python
import math
from fractions import Fraction


def close_enough(a: float, b: float) -> bool:
    return math.isclose(a, b, rel_tol=1e-9, abs_tol=1e-12)


if __name__ == "__main__":
    print(0.1 + 0.2)
    print(0.1 + 0.2 == 0.3)
    print(close_enough(0.1 + 0.2, 0.3))
    print(Fraction(1, 10) + Fraction(2, 10) == Fraction(3, 10))
    print(2**53 + 1 == 2**53)
    print(float(2**53 + 1))
```

Starter code for `operators_demo.py`:

```python
if __name__ == "__main__":
    print(7 // 2, -7 // 2)
    print(7 % 3, -7 % 3)
    print(divmod(7, 3))
    print(2 ** 3 ** 2)
    print(-2 ** 2, (-2) ** 2)
```

Starter code for `tests/test_money.py`:

```python
import unittest
from decimal import Decimal

from money import add_amounts, format_money, parse_amount, split_evenly


class MoneyTests(unittest.TestCase):
    def test_parse_rounds_half_up(self):
        self.assertEqual(parse_amount("10.005"), Decimal("10.01"))

    def test_parse_rejects_text(self):
        with self.assertRaises(ValueError):
            parse_amount("abc")

    def test_parse_rejects_nan(self):
        with self.assertRaises(ValueError):
            parse_amount("NaN")

    def test_split_keeps_every_cent(self):
        pieces = split_evenly(Decimal("10.00"), 3)
        self.assertEqual(pieces, [Decimal("3.34"), Decimal("3.33"), Decimal("3.33")])
        self.assertEqual(add_amounts(*pieces), Decimal("10.00"))

    def test_split_negative_total(self):
        pieces = split_evenly(Decimal("-10.00"), 3)
        self.assertEqual(add_amounts(*pieces), Decimal("-10.00"))

    def test_split_rejects_zero_parts(self):
        with self.assertRaises(ValueError):
            split_evenly(Decimal("1.00"), 0)

    def test_format_money(self):
        self.assertEqual(format_money(Decimal("1234.5")), "$1,234.50")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python money.py`. Paste the output into `docs/findings.md`. Explain in one sentence why the three pieces add up to the total.
3. Run `python float_checks.py`. In `docs/findings.md`, write one sentence for each line explaining why it printed what it did.
4. Run `python operators_demo.py`. Add two more expressions of your own that depend on precedence, and record the predicted and actual results.
5. Run `python -m unittest` and make every test pass. Add one test for `parse_amount("0")` that returns `Decimal("0.00")`.
6. Add a short section to `docs/findings.md` titled "Rounding decisions". Name the rounding mode used by `parse_amount` and by `split_evenly`, and say why they differ.
7. Run `python -m mypy money.py tests/test_money.py` and record the result line in `docs/findings.md`.
8. Commit with the message "Add money and number lab with tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least eight tests and passes
- `python -m mypy money.py` reports no issues
- `docs/findings.md` contains the outputs of the three scripts, the explanations, the rounding decisions section and the mypy line
- The GitHub repo shows the committed files and `git status` is clean

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with run commands for each script, tests and mypy), and `.gitignore`.

The project is light on new features and heavy on decisions about types. A reviewer who reads `docs/findings.md` should be able to see why `Decimal` was used for money and `float` was used nowhere in the money path.

---

## Check your understanding

1. A colleague computes an invoice total with floats and gets `10.499999999999998`. Explain the source of the error and name the type you would use instead, with the reason.
2. Why does `-7 // 2` return `-4` in Python, and what would a C programmer expect? Give one place where this difference would change the result of a real program.
3. A function returns `sum([True, True, False])` as a rating. Explain what is happening, and name the one-line change that makes the intent clear.
4. You need the average of a million floating-point measurements in a particular order. Why might `sum(values) / len(values)` differ from `math.fsum(values) / len(values)`, and which would you choose for a scientific result?
5. Predict the output of `print(-2 ** 2 + 3 * 2 ** 2 // 4)` before running it. Then explain the steps Python takes, in order.
6. A user types `"4.2"` into a form that asks for a quantity. Explain why `int("4.2")` fails and write a conversion that rejects non-integers with a clear message.

### Answers

1. The decimal values `0.1`, `0.2` and similar amounts are not exact in binary floating point, so each stored value carries a tiny error that accumulates through the sums. The correct type for money is `Decimal`, which stores base-10 digits exactly within its precision, so amounts such as `0.10` are represented as written.
2. Python's `//` is floor division, which rounds toward negative infinity, while C truncates toward zero and gives `-3`. A reporting script that splits a negative balance into whole units would show a different value, which changes totals and reconciliation.
3. `bool` is a subclass of `int`, so `sum` adds `True` as `1` and `False` as `0`. The fix is to convert explicitly at the boundary, for example `sum(int(x) for x in items)`, or to name the rating scale in a docstring so the intent is written down.
4. Floating-point addition is not associative, so a plain loop can lose low-order bits at each step, and the result depends on the order of the values. `math.fsum` tracks the lost bits and returns a correctly rounded sum. For a scientific result, `math.fsum` is the better choice, because the answer is not hostage to input order.
5. The output is `-1`. Python evaluates `2 ** 2` first, giving 4, and the unary minus applies to that power, so `-2 ** 2` is `-4`. Next `3 * 2 ** 2` is `3 * 4`, which is 12. Then `12 // 4` is 3, because `//` and `*` have equal precedence and group left to right, and the last step is `-4 + 3`.
6. `int()` accepts only text that is a valid integer literal, and `"4.2"` contains a decimal point, so it raises `ValueError`. A safe conversion uses `int(text)` inside a `try` block, raises a clear message on failure, and does not silently truncate a decimal.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Numeric Types (int, float, complex)", at docs.python.org/3/library/stdtypes.html
- Python documentation, "Expressions", chapter "Operator precedence", at docs.python.org/3/reference/expressions.html
- Python documentation, module `decimal`, at docs.python.org/3/library/decimal.html, and the FAQ section on floating-point arithmetic at docs.python.org/3/tutorial/floatingpoint.html
- [Wikipedia: Floating-point arithmetic](https://en.wikipedia.org/wiki/Floating-point_arithmetic)
- [Wikipedia: IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)
- [Wikipedia: Arbitrary-precision arithmetic](https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic)

*Tutorials*

- GeeksforGeeks, "Python Operators" and "Python Type Conversion" tutorials, at geeksforgeeks.org
- W3Schools, "Python Operators" and "Python Numbers" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the lecture segments on numbers and operators.

*Practice*

- Exercism, the Python track at exercism.org/tracks/python, where the "Leap" and "Bob" exercises practice operators and conditions with automated tests.

---

## Tomorrow

Day 12 moves from numbers to text. Strings are immutable sequences of characters, and they look simple until you meet Unicode, encodings and bytes. You will index and slice strings, format them with f-strings, and learn why the length of a string and the number of bytes it occupies can differ. The immutability rule from Day 10 applies throughout.

**Next:** [Day 12: Strings and Unicode](Software-Day-12-Strings-and-Unicode.md)

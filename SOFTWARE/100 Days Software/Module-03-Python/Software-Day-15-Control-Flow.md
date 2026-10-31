# Day 15: Control Flow

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner to intermediate. You can write functions that return values, and you know truthiness and comprehensions from Day 10 and Day 13. This is for anyone whose program takes the wrong branch, or who writes a loop that runs forever because the stop condition never becomes true.

**By the end of this day, you will have:**

- A working model of how Python chooses a branch with `if`, `elif` and `else`, and how it stops at the first true condition
- The ability to write structural `match` statements with literal, sequence, OR and guard patterns, and to know when `match` is the wrong tool
- Fluency with `for` loops over any iterable, `while` loops with a clear stop condition, and `range` with a start, stop and step
- Correct use of `break`, `continue` and the `else` clause on loops, including the search-loop pattern
- Confident use of comprehensions with conditions, and the judgment to choose one over a loop
- A small command-and-grades repo with tests that check each branch and each loop exit path

**What you need:**

- Python 3.14 (any 3.12 or newer works). The `match` statement needs 3.10 or newer, so check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 9: Python Runtime and Syntax (indentation, statements)
- Day 10: Objects, Variables and Types (truthiness)
- Day 13: Lists and Tuples (iteration, unpacking)
- Day 14: Dictionaries and Sets (iterating over mappings)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [If, elif and else](#if-elif-and-else)
  - [Match](#match)
  - [Loops](#loops)
  - [Range](#range)
  - [Break and continue](#break-and-continue)
  - [Loop else](#loop-else)
  - [Comprehensions with conditions](#comprehensions-with-conditions)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a command and grade lab](#project-a-command-and-grade-lab)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A program is mostly decisions and repetition. A login system decides whether a password is correct, a checkout decides which discount applies, and a data import repeats the same steps for every row. Control flow is the set of rules that decides which code runs and how many times. If the rules are wrong, the program is wrong, even if every line of each branch is correct. The bugs that come from control flow are common and expensive: a discount applied twice, a loop that never ends, a retry that retries forever, an error case that falls through to the success path.

This day covers the full set of Python control structures, from the basic `if` to the `match` statement that handles structured data, the `for` and `while` loops, the `range` object, the `break` and `continue` keywords, and the `else` clause on loops that many people have never used. Each one answers a different question about what should happen next. The goal is to pick the right one on purpose and to test the paths it creates.

The official reference is the Python tutorial section "More Control Flow Tools" at docs.python.org/3/tutorial/controlflow.html, and the language reference section "The match statement" at docs.python.org/3/reference/compound_stmts.html#the-match-statement. Read the tutorial section first, then the reference when you need exact rules.

---

## Glossary

- **Branch:** a block of code that runs only when a condition is true.
- **Condition:** an expression whose truth value decides a branch, such as `score >= 90`.
- **Pattern:** a shape that a `match` statement compares with a value, such as `["add", *items]`.
- **Guard:** an extra `if` condition attached to a `case`, which must also be true for that case to match.
- **Loop:** a statement that runs a block repeatedly, either for each item of an iterable or while a condition holds.
- **Iterable:** any object that can produce items one at a time, such as a list, a string, a dictionary or a range.
- **Iterator:** an object that produces the next item on request, and raises `StopIteration` when no items remain.
- **Sentinel:** a value that signals the end of a sequence, such as an empty input line.
- **break:** a statement that leaves the nearest loop immediately.
- **continue:** a statement that skips the rest of the current loop iteration and moves to the next one.
- **Fall-through:** continuing from one case into the next. Python `match` does not do it.
- **Wildcard:** the pattern `_`, which matches anything and binds nothing.

---

## Concepts

The concepts follow two lines of thought: choosing a branch, and repeating a block. Each one has a clear exit rule, and each one has a failure mode.

### If, elif and else

Think of a sorting station on a conveyor belt. Each parcel passes a check, and the first check it satisfies decides where it goes. Parcels that match no check go to the last bin. The `if`, `elif` and `else` keywords build exactly that sorting station.

From first principles, Python evaluates the `if` condition, and if its truth value is true, it runs the block under it and skips the rest of the chain. Otherwise it evaluates each `elif` in order and runs the first block whose condition is true. The `else` block runs only when none of the earlier conditions was true. Only one block in the chain runs, and the order of the conditions is part of the logic.

```python
def letter_grade(score: int) -> str:
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    elif score >= 70:
        return "C"
    else:
        return "below C"


print(letter_grade(95), letter_grade(85), letter_grade(10))  # A B below C
```

Because the checks run from the top, the order matters. If the `score >= 70` check came first, a score of 95 would return `C` and never reach the `A` check. Put the most specific or most common cases first, and keep the conditions mutually clear.

The common mistake is writing separate `if` statements where `elif` was meant. Each `if` is checked on its own, so if the branches assign a value instead of returning, a score of 95 sets the grade to `A`, then to `B`, then to `C`, and the last assignment wins. Returning from each branch hides this bug, which is why the early `return` pattern is common, but the independent-`if` version is still wrong when the branches assign values. The fix is to write `elif` or to return immediately. A second mistake is checking `if x == True`, which is a style problem because `if x:` already tests truth. In the job market, code reviewers often ask for the chain to be rewritten as a table or a dictionary lookup when it grows past four or five branches.

### Match

The `match` statement, added in Python 3.10, is a structured version of the branch chain. Think of a postal sorter who looks at the shape of a parcel, not only its weight. The sorter sees that a parcel is a box, an envelope or a tube, and then checks the size. `match` checks the shape of a value and binds names from its parts.

From first principles, `match subject:` evaluates the subject once. Each `case` pattern is tried in order. A literal pattern compares by equality. A capture pattern is a plain name that binds the value. A sequence pattern matches lists and tuples by length and items, and `*rest` captures the remaining items. The `|` operator combines alternatives. A guard after `if` adds a condition. The wildcard `_` matches anything and binds nothing. The first case that matches runs, and there is no fall-through to the next case.

```python
def handle(command: str) -> str:
    match command.split():
        case ["add", *items] if items:
            return f"adding {len(items)} item(s)"
        case ["remove", item]:
            return f"removing {item}"
        case ["quit" | "exit"]:
            return "bye"
        case []:
            return "empty command"
        case _:
            return f"unknown: {command}"


print(handle("add milk eggs"))   # adding 2 item(s)
print(handle("exit"))            # bye
print(handle("add"))             # unknown: add, the guard fails
```

The `add` case has a guard. When the command is only `add`, the sequence pattern matches but the guard `if items` is false, so Python moves on to the next case. That is the behavior that makes guards useful.

The common mistake is using `match` for a simple comparison of a single value against a few constants, where `if` and `elif` read better. Use `match` when you are destructuring a shape, such as a command, a message or a parsed tree. A second mistake is writing a capture pattern by accident. In `case quit:`, the word `quit` is a capture that matches anything, not the constant `"quit"`. Constants must be written as literals or as dotted names such as `Color.RED`. Linters and type checkers catch this in many cases, and reading the code with that rule in mind avoids it.

### Loops

A loop is a set of instructions that repeats. Think of a librarian who checks each book on a cart, one at a time. The `for` loop walks through a sequence of items, and it stops when the sequence ends. The `while` loop repeats as long as a condition is true, and it stops when the condition becomes false.

From first principles, `for item in iterable:` asks the iterable for an iterator, then repeatedly asks for the next item, runs the block with that item bound to the loop name, and stops when the iterator signals that it is exhausted. `while condition:` evaluates the condition before each pass, and it runs the block only when the condition is true. A `while` loop is the right tool when you do not know the number of passes in advance, such as reading input until a sentinel value appears.

```python
total = 0
for price in [2.5, 4.25, 3.0]:
    total += price
print(total)            # 9.75

countdown = 3
while countdown > 0:
    print(countdown)
    countdown -= 1
print("liftoff")
```

The `while` loop changes `countdown` on each pass. If that line were missing, the loop would never end, and the program would hang until you interrupt it with Ctrl+C.

The common mistake is a `while` loop whose condition never changes in a way that makes it false. The loop runs forever, and the program appears frozen. Before writing a `while`, write the line that changes the condition, and check that it moves toward the stop. A second mistake is modifying the list you are iterating over. Iterate over a copy or build a new list. Day 13 covered this. In the job market, runaway loops in retry logic cause production incidents, so a bounded number of attempts is a common requirement.

### Range

`range` is a lightweight sequence of integers. Think of a ruler with marks at regular steps. You do not need a physical ruler with every mark drawn in advance. The ruler describes the marks, and you read them when you need them.

From first principles, `range(stop)` produces 0 up to but not including `stop`. `range(start, stop)` starts at `start`. `range(start, stop, step)` adds the step each time, and a negative step counts down. A `range` object stores only the start, the stop and the step, so `range(10**9)` takes the same small amount of memory as `range(10)`. It supports `len`, membership tests and indexing, and it produces its items lazily.

```python
print(list(range(3)))          # [0, 1, 2]
print(list(range(2, 10, 3)))   # [2, 5, 8]
print(list(range(5, 0, -2)))   # [5, 3, 1]
print(len(range(0, 100, 7)))   # 15
print(50 in range(0, 100, 10)) # True
```

The common mistake is expecting `range(1, 5)` to include 5. It does not, and the stop value is always excluded. The second mistake is calling `list(range(...))` when you only need to loop. The loop `for i in range(n)` is more efficient, because it does not build a list. In the job market, off-by-one errors around ranges are among the most frequent bugs found in code review, which is why the stop-excluded rule is worth memorizing.

### Break and continue

`break` and `continue` change the path through a loop. Think of a security guard on a patrol route. `continue` means "skip this room and go to the next one". `break` means "stop the patrol now and go to the exit". Both affect only the nearest enclosing loop.

From first principles, `continue` jumps to the end of the loop body and then moves to the next item or re-evaluates the `while` condition. `break` exits the loop entirely, and execution continues after the loop. Neither keyword works outside a loop, and `break` inside a nested loop only exits the inner loop. A `return` inside a loop also leaves the loop, because it leaves the whole function.

```python
def odd_countdown(start: int) -> list[int]:
    result = []
    n = start
    while n > 0:
        n -= 1
        if n % 2 == 0:
            continue
        result.append(n)
    return result


print(odd_countdown(6))   # [5, 3, 1]
```

The `continue` skips the even values, and the loop still ends when `n` reaches zero. The loop condition handles termination, so `continue` cannot cause an infinite loop unless the update line is skipped as well. Put the update before any `continue`, as the example does.

The common mistake is placing `continue` before the line that updates the loop variable in a `while` loop. The variable never changes, and the loop repeats the same skipped branch forever. The second mistake is using `break` where a `return` or a condition would make the intent clearer. A loop with several `break` statements is often a sign that the function should be split. Code reviewers suggest extracting a helper that returns the answer directly, and the result is usually easier to test.

### Loop else

The `else` clause on a loop is one of the least known parts of Python. Think of a search of a filing cabinet. If you find the file, you stop and report it. If you reach the end of the cabinet without finding it, you report that it is missing. The `else` block is the "missing" report, and it runs only when the loop finished without a `break`.

From first principles, a `for` or `while` loop may have an `else` block after its body. Python runs the `else` block when the loop ends normally, meaning the iterator was exhausted or the condition became false. If the loop exits through `break`, the `else` block is skipped. A `return` also skips it, because the function has already ended.

```python
def find_user(users: list[str], target: str) -> str:
    for user in users:
        if user == target:
            return f"found {target}"
    else:
        return f"{target} not found"


print(find_user(["ada", "grace"], "grace"))  # found grace
print(find_user(["ada", "grace"], "linus"))  # linus not found
```

The `else` here is clearer than a flag variable. The function returns from inside the loop when it finds the item, and the `else` covers the other case.

The common mistake is reading the `else` of a loop as the `else` of the `if` inside it. It belongs to the loop. Another mistake is using a loop `else` in a loop with a `break` that the reader does not notice, which makes the logic hard to follow. Name the case in a comment or restructure the function so the meaning is obvious. In the job market, the loop `else` appears in search and validation code, and interviewers ask about it because it tests whether a candidate reads Python's grammar carefully.

### Comprehensions with conditions

A comprehension is a loop written as one expression. You met it on Day 13 and Day 14. Think of a filter on a conveyor belt. Items pass through the filter, and only the ones that satisfy the rule come out the other side. The `if` clause is the filter.

From first principles, `[expr for item in iterable if condition]` runs the loop, evaluates the condition, and includes `expr` only when the condition is true. The same pattern works for sets and dictionaries. A comprehension is an expression, so it can be used as an argument to a function, but it cannot contain `break` or `continue`. If you need to stop early, use a loop.

```python
durations = [95, 241, 60, 180]
long_tracks = [d for d in durations if d > 120]
labels = ["long" if d > 120 else "short" for d in durations]
print(long_tracks)  # [241, 180]
print(labels)       # ['short', 'long', 'short', 'long']
```

The second comprehension uses a conditional expression, `"long" if d > 120 else "short"`, which chooses a value. That is different from the `if` clause at the end, which filters items. Both are useful, and they are easy to confuse.

The common mistake is trying to use `break` inside a comprehension, which is a syntax error, or writing a long comprehension that does several things, which is hard to read. Keep comprehensions short, with one condition and one expression. In the job market, reviewers prefer a loop with a name for each step over a dense comprehension with nested conditions, because the loop can be debugged line by line.

---

## Hands-on exercises

Work in a folder called `control-flow-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices `if`, `elif` and `else`.

1. Create `grades.py` with the `letter_grade` function from the concepts section. Print the grade for 95, 85, 72 and 10.
2. Rewrite the function with separate `if` statements, with a `return` in each, and observe that the result is the same for these four inputs. Then add the input 101 and predict what happens.
3. Add a check for a score outside 0 to 100 that raises `ValueError`. Test it with 101 and with -1.

Done when you can explain why the order of the checks matters, and the out-of-range inputs raise the error.

Exercise 2 practices `match`.

1. Create `commands.py` with the `handle` function from the concepts section.
2. Call it with `"add milk eggs"`, `"remove milk"`, `"exit"`, `"add"`, the empty string and `"dance"`. Record each result.
3. Add a case for `["help"]` that returns a help message, and place it before the `case _`.

Done when every result matches your prediction and the help case works.

Exercise 3 practices loops and `range`.

1. Create `loops.py`. Use a `for` loop to sum the numbers from 1 to 10 using `range`.
2. Use a `while` loop to print the powers of 2 that are less than 100.
3. Use `range` with a negative step to print the even numbers from 10 down to 0.

Done when the sum is 55, the powers are 1, 2, 4, 8, 16, 32 and 64, and the even numbers are printed in descending order.

Exercise 4 practices `break`, `continue` and loop `else`.

1. Create `search.py` with `find_user` from the concepts section.
2. Write a function `first_negative_index(numbers: list[int]) -> int | None` that uses a loop with `break`, and a second function that does the same job with `enumerate` and `return`.
3. Write one sentence on which version you would choose and why.

Done when both functions return the same results for `[3, -1, 2]` and `[1, 2]`, and your sentence names a reason.

Exercise 5 runs the type checker.

1. Write `typed_control.py` with a function `count_passing(scores: list[int], pass_mark: int = 50) -> int` that uses a comprehension with a condition.
2. Call it with a list that includes a string by mistake.
3. Run `python -m mypy typed_control.py`, read the error, fix the call, and rerun until mypy reports `Success: no issues found`.

Done when the function returns the correct count and mypy is clean.

---

## Project: a command and grade lab

The project is a small command handler and a grade reporter. It uses `match` to parse commands, `if` and `elif` for grades, a `while` loop with a sentinel for input, `break` and loop `else` for search, and a comprehension with a condition for filtering. Each branch and each loop exit path has a test. The evidence is a test suite that covers the paths and a findings document that records one case where the loop `else` made the logic clearer.

Suggested repo name: `100-days-software-day-15-control-flow`.

Repo layout:

```
100-days-software-day-15-control-flow/
├── .gitignore
├── README.md
├── control_lab.py
├── commands.py
├── tests/
│   ├── __init__.py
│   └── test_control_lab.py
└── docs/
    └── findings.md
```

Starter code for `control_lab.py`:

```python
def letter_grade(score: int) -> str:
    if not 0 <= score <= 100:
        raise ValueError(f"score out of range: {score}")
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    elif score >= 70:
        return "C"
    else:
        return "below C"


def odd_countdown(start: int) -> list[int]:
    result = []
    n = start
    while n > 0:
        n -= 1
        if n % 2 == 0:
            continue
        result.append(n)
    return result


def find_user(users: list[str], target: str) -> str:
    for user in users:
        if user == target:
            return f"found {target}"
    else:
        return f"{target} not found"


def count_passing(scores: list[int], pass_mark: int = 50) -> int:
    return len([s for s in scores if s >= pass_mark])


if __name__ == "__main__":
    for score in [95, 85, 72, 10]:
        print(score, letter_grade(score))
    print(odd_countdown(6))
    print(find_user(["ada", "grace"], "grace"))
```

Starter code for `commands.py`:

```python
def handle(command: str) -> str:
    match command.split():
        case ["add", *items] if items:
            return f"adding {len(items)} item(s)"
        case ["remove", item]:
            return f"removing {item}"
        case ["quit" | "exit"]:
            return "bye"
        case []:
            return "empty command"
        case _:
            return f"unknown: {command}"


def run(lines: list[str]) -> list[str]:
    responses = []
    for line in lines:
        response = handle(line)
        responses.append(response)
        if response == "bye":
            break
    return responses


if __name__ == "__main__":
    print(run(["add milk eggs", "remove milk", "exit", "add more"]))
```

Starter code for `tests/test_control_lab.py`:

```python
import unittest

from commands import handle, run
from control_lab import count_passing, find_user, letter_grade, odd_countdown


class ControlFlowTests(unittest.TestCase):
    def test_letter_grades(self):
        self.assertEqual(letter_grade(95), "A")
        self.assertEqual(letter_grade(85), "B")
        self.assertEqual(letter_grade(72), "C")
        self.assertEqual(letter_grade(10), "below C")

    def test_grade_out_of_range(self):
        with self.assertRaises(ValueError):
            letter_grade(101)

    def test_odd_countdown(self):
        self.assertEqual(odd_countdown(6), [5, 3, 1])
        self.assertEqual(odd_countdown(0), [])

    def test_find_user_found_and_missing(self):
        self.assertEqual(find_user(["ada", "grace"], "grace"), "found grace")
        self.assertEqual(find_user(["ada", "grace"], "linus"), "linus not found")

    def test_count_passing(self):
        self.assertEqual(count_passing([40, 50, 90]), 2)
        self.assertEqual(count_passing([]), 0)

    def test_match_add_with_items(self):
        self.assertEqual(handle("add milk eggs"), "adding 2 item(s)")

    def test_match_add_without_items_falls_through(self):
        self.assertEqual(handle("add"), "unknown: add")

    def test_match_quit_alternatives(self):
        self.assertEqual(handle("quit"), "bye")
        self.assertEqual(handle("exit"), "bye")

    def test_empty_command(self):
        self.assertEqual(handle("   "), "empty command")

    def test_run_stops_at_exit(self):
        responses = run(["add milk", "exit", "add more"])
        self.assertEqual(responses, ["adding 1 item(s)", "bye"])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python control_lab.py` and paste the output into `docs/findings.md`. Explain in one sentence which branch produced the grade `"B"`.
3. Run `python commands.py` and paste the output into `docs/findings.md`. Explain why the last command is not in the output.
4. Run `python -m unittest` and make every test pass.
5. Add a `while` loop function `read_until_blank(lines: list[str]) -> list[str]` that collects lines until it reaches an empty string, using an index variable that you increment. Add a test that includes a blank line in the middle. Explain in `docs/findings.md` what would happen if you forgot to increment the index.
6. Add one case to `handle` for `["help"]` and one test for it.
7. Run `python -m mypy control_lab.py commands.py tests/test_control_lab.py` and fix every issue.
8. Write a paragraph in `docs/findings.md` naming the one place where a loop `else` made the code clearer, and say why a flag variable would have been worse.
9. Commit with the message "Add command and grade lab with control flow tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least twelve tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs of both scripts, the explanations, the infinite loop note, and the loop `else` paragraph
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with run commands), and `.gitignore`.

The project covers every control structure from this day, and each test checks one exit path. A reviewer should be able to see that the `match` falls through when the guard fails, that the `run` function stops at `exit`, and that the search function answers both cases.

---

## Check your understanding

1. A discount function uses three separate `if` statements, each assigning a `discount` value, with the smallest threshold checked first. A customer with a large order gets the smallest discount. Explain why, and rewrite the function so that the correct branch is the only one that runs.
2. A `match` statement has a case `case status:` meant to match the string `"active"`. Explain what the pattern actually does and how to write it correctly.
3. A `while` loop reads commands until the user types `quit`, and the program hangs after the user types `quit` the second time. Give two possible causes and the change that prevents each.
4. Why does `range(1, 5)` produce four items, and how do you write a range that includes 5?
5. Predict the output of a function that searches a list with a `for` loop and a `break`, and has an `else` block that prints "not found". The item is in the list, and the loop breaks. Explain whether the `else` runs.
6. A comprehension `[x for x in items if x > 0 and break_here()]` is proposed as a way to stop at the first zero. Explain why it cannot work as intended, and write a loop that does the job.

### Answers

1. Each independent `if` is tested, so a large order matches the smallest threshold, sets `discount`, then matches the larger thresholds and overwrites it. The last assignment wins, which is the smallest discount. Write the chain with the largest threshold first and `elif` for the rest: `if total >= 500: discount = 0.2` then `elif total >= 100: discount = 0.1`, and so on, with a default for the smallest orders.
2. The bare name `status` is a capture pattern, which matches any value and binds it to `status`, so the case always matches. Write the literal `case "active":` to compare with the string, or use a dotted name such as `case Status.ACTIVE:` for an enum.
3. Possible causes include a condition that checks the wrong variable, so `quit` is never recognized, and a `continue` placed before the line that reads the next command, so the loop stays on the old value. The fix is to check the same variable that holds the input, and to read the next input before any `continue`, or to use a `break` when the sentinel appears.
4. A range stops before its stop value, so `range(1, 5)` gives 1, 2, 3 and 4, which is four items. To include 5, use `range(1, 6)`.
5. The `else` does not run. A `break` exits the loop without triggering the loop's `else` block, so the "not found" message is printed only when the loop finishes without finding the item.
6. A comprehension cannot contain `break`, because it is an expression and not a statement, so the syntax is invalid. A loop does the job: `for x in items:` with `if x == 0: break` at the top, and the processing of each `x` placed after that check.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "More Control Flow Tools", at docs.python.org/3/tutorial/controlflow.html
- Python language reference, "The match statement" under "Compound statements", at docs.python.org/3/reference/compound_stmts.html
- PEP 636, "Structural Pattern Matching: Tutorial", at peps.python.org/pep-0636/
- [Wikipedia: Control flow](https://en.wikipedia.org/wiki/Control_flow)
- [Wikipedia: Pattern matching](https://en.wikipedia.org/wiki/Pattern_matching)

*Tutorials*

- GeeksforGeeks, "Python if else" and "Python Loops" tutorials, at geeksforgeeks.org
- W3Schools, "Python If...Else" and "Python For Loops" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on conditionals and loops.

*Practice*

- Exercism, the Python track at exercism.org/tracks/python, where the exercises on conditionals and loops check each branch with automated tests.

---

## Tomorrow

Day 16 starts the function half of the course. You will define functions with parameters and return values, understand how arguments are bound, learn the LEGB rule that decides which name a function sees, and write docstrings and type annotations. The branches and loops from today are the body of most functions, and the mutable default trap from Day 10 returns in a new form.

**Next:** [Day 16: Functions I](Software-Day-16-Functions-I.md)

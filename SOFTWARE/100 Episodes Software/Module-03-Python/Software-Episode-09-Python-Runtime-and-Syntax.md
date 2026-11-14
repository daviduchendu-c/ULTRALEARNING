# Day 9: Python Runtime and Syntax

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner. You have written a few lines of code in any language, or you finished Day 3 and have Python installed. This is for anyone who types `python` and hopes for the best, and wants to know what actually happens next.

**By the end of this day, you will have:**

- A working Python 3.12 or newer install that you can name, locate and check from the command line
- The ability to explain what CPython does between your text file and the running program
- Fluency with the REPL, script mode and `python -m`, and the habit of choosing the right one
- A small repo that reports its own runtime, checks variable names against the language rules, and shows the bytecode of a function
- A clear model of indentation, expressions and statements, which removes most first-week syntax errors

**What you need:**

- Python 3.14 (any version from 3.12 upward works for this day). Check with `python --version`. On Windows, the `py` launcher is also fine: `py -3.14 --version`.
- Git 2.40 or newer, from Day 6 and Day 7
- A GitHub account from Day 8
- A code editor that understands Python, such as Visual Studio Code with the Python extension from Microsoft (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2

**Prerequisites:**

- Day 2: How Computers Execute Programs (CPU, memory, the idea of machine code)
- Day 3: Developer Environment (terminal, editor, PATH)
- Day 4 and Day 5: Linux CLI (`cd`, `ls`, running commands)
- Day 6 to Day 8: Git basics, branches and GitHub

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [CPython and the interpreter](#cpython-and-the-interpreter)
  - [The REPL](#the-repl)
  - [Scripts and the command line](#scripts-and-the-command-line)
  - [Bytecode](#bytecode)
  - [Indentation](#indentation)
  - [Expressions and statements](#expressions-and-statements)
  - [Naming](#naming)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a runtime and syntax lab](#project-a-runtime-and-syntax-lab)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most people learn Python by typing lines into a prompt and seeing results. That works until a script fails with an error that mentions a file you did not write, or until a colleague says "that runs on 3.9 but not on 3.12" and you have no idea why. Those moments are the reason to understand the runtime. Python is a language, but when you run it you are running a specific program, the interpreter, which reads your text, turns it into instructions, and executes them. Knowing which program is running, where its files live, and how it reads your indentation lets you diagnose a failure instead of guessing at it.

This day covers the foundation that the next seventeen days stand on. Every later topic, from data types to testing, assumes you can start the right Python, read its errors and predict how it parses a line. You will finish with a repo you can show a reviewer, and with a command line habit you will use for the rest of the course.

The official reference for the language is the Python Language Reference at docs.python.org/3/reference/. Read its "Lexical analysis" chapter once; it is short and it is the source of the indentation rules covered below.

---

## Glossary

- **Interpreter:** a program that reads source code and runs it. For Python, the usual interpreter is CPython.
- **CPython:** the reference implementation of Python, written in C, maintained by the Python Software Foundation.
- **Source code:** the text you write, usually in files ending in `.py`.
- **REPL:** Read, Evaluate, Print, Loop. An interactive prompt where each line you type is run and its result printed.
- **Script:** a file of Python code that you run from start to finish.
- **Bytecode:** a compact list of low-level instructions that CPython runs on its virtual machine.
- **Virtual machine:** a software engine that executes bytecode. It is not a hardware machine.
- **Indentation:** leading spaces at the start of a line. In Python, it marks blocks of code.
- **Expression:** code that produces a value, such as `2 + 3`.
- **Statement:** code that performs an action, such as an assignment, a loop or an import.
- **Identifier:** a name you choose for a variable, function, class or module.
- **Keyword:** a word the language reserves, such as `if`, `for` or `return`. You cannot use it as a name.
- **PATH:** a list of folders the shell searches when you type a command name.

---

## Concepts

Each concept below builds on the one before it. Start with what the interpreter is, then how you talk to it, then what it does with your text.

### CPython and the interpreter

Think of a restaurant kitchen. The menu is written in a language the customer understands. The cooks read a recipe card in their own working language and carry out the steps. Python source code is the menu. CPython is the kitchen that takes your recipe and does the work, and it is written in C, a different language, which is why people call it "CPython" to distinguish it from the language itself.

From first principles, running `python my_script.py` does four things. The operating system finds the `python` executable on your PATH and starts it as a process. CPython reads the file from disk as text. It parses that text into a tree that represents the structure of your code, and then it compiles the tree into bytecode. Finally it runs the bytecode on its virtual machine, one instruction at a time. If a name is missing or a line breaks the grammar, the error happens in the parse or compile stage, before any of your code runs.

The definition that matters for this course is short. CPython is the interpreter that most people mean when they say "Python". Other implementations exist, such as PyPy, which uses a just-in-time compiler, and MicroPython, built for microcontrollers. This course uses CPython because it is the default everywhere, and because every library you will install assumes it.

You can ask the running interpreter about itself. Create a file called `where.py` with these two lines and run it:

```python
import sys

print(sys.implementation.name, sys.version.split()[0], sys.executable)
```

On a typical Windows install with Python 3.14, the output looks like `cpython 3.14.6 C:\Users\you\AppData\Local\Programs\Python\Python314\python.exe`. Your path will differ. The first word tells you the implementation, the second the version, and the third the exact program that ran. When two Pythons are installed and the wrong one answers, this line tells you which one it is.

The common mistake is assuming that `python` and `python3` and `py` are the same program. They often are not. On Windows, `python` may point to the Microsoft Store stub, and `py` is the launcher that picks an installed version. Run `where python` and `py --list` in PowerShell, or `which -a python3` in Bash, to see every candidate. In the job market, every team that ships a Python service pins an interpreter version in a file such as `.python-version` or in the `requires-python` field of `pyproject.toml`, and CI systems such as GitHub Actions select that exact interpreter before running tests.

### The REPL

A REPL is a notebook you can talk to. You type one line, the interpreter reads it, evaluates it, prints the result, and waits for the next line. The name stands for Read, Evaluate, Print, Loop, and the Wikipedia article "Read–eval–print loop" describes the idea across many languages.

Start it by typing `python` with no file name. The prompt is `>>>`, and a continuation prompt `...` appears when a statement needs more lines.

```python
>>> 2 + 3
5
>>> name = "Ada"
>>> name.upper()
'ADA'
>>> for n in range(3):
...     print(n)
...
0
1
2
```

Notice that the REPL shows the value of an expression automatically, but it shows nothing for an assignment. The line `name = "Ada"` is a statement, and statements do not produce a value to print. The variable `name` remains in the session until you exit, so the REPL is good for poking at objects one at a time.

The REPL has limits. Nothing you type is saved, and multi-line blocks are awkward to edit. Use it for quick checks, such as testing how `str.split` behaves, or confirming what `len("héllo")` returns. Put anything you want to keep in a file. Python 3.13 and newer ship a richer default interactive shell with multi-line editing and colored prompts, so the experience is better than it was in older releases.

The common mistake is pasting a file into the REPL, or the reverse, typing a long function into it and losing the work when you exit. Save to a file as soon as a snippet is worth keeping. Experienced engineers keep an interactive session open next to the editor for quick checks before they write the real code.

### Scripts and the command line

A script is a file that the interpreter reads from top to bottom and runs. You start it with `python path/to/file.py`. Python runs the code in the file, and when the file ends, the process exits. Any top-level code in the file runs when the file is executed.

There are two ways to start a module, and the difference matters for the rest of the course. `python hello.py` runs the file as the main program. `python -m hello` finds the module named `hello` on the module search path and runs it as the main program. The `-m` form is preferred for packages and tests, because it puts the current directory on `sys.path` and makes imports behave predictably. You will see `python -m unittest` and `python -m pip` throughout this course for the same reason.

Create `hello.py` with the following content and run both forms:

```python
def main() -> None:
    print("hello from", __name__)


if __name__ == "__main__":
    main()
```

Running `python hello.py` prints `hello from __main__`. Running `python -m hello` from the same folder prints `hello from __main__` as well. Importing the file from another script with `import hello` prints nothing, because the `if __name__ == "__main__":` guard is false when the file is imported. Day 18 explains that guard in full. For now, remember that a script which runs code on import will surprise you later.

On Windows, `py -3.14 hello.py` picks a specific version, which is useful when several versions are installed. On Linux and macOS, `python3.14 hello.py` does the same. Pin the version in the command when a project depends on it.

The common mistake is running a script from the wrong folder, then getting a `FileNotFoundError` for a file that exists. Relative paths resolve against the current working directory, not against the script's folder. Use `pathlib.Path(__file__).parent` when a script must find files next to itself. Day 23 covers `pathlib` properly.

### Bytecode

When CPython compiles your file, it does not run the text directly and it does not produce a machine-code program for your CPU. It produces bytecode. Think of bytecode as a set of small, numbered instructions, such as "load this constant", "add the top two values", "call this function". The virtual machine reads them in order and keeps a stack of values as it goes.

From first principles, bytecode exists so that Python can be portable and fast to start. The same bytecode runs on any platform that has a matching CPython build, and the compiled form is cached. When you import a module, CPython stores its bytecode in a `__pycache__` folder next to the source, using a file name that includes the implementation and version, such as `hello.cpython-314.pyc`. The next import reuses it if the source has not changed.

You can look at bytecode with the standard library module `dis`. Create `bytecode_peek.py`:

```python
import dis


def add_tax(price: float, rate: float) -> float:
    return price * (1 + rate)


dis.dis(add_tax)
```

Running it prints a table of instructions that include names such as `LOAD_FAST`, `BINARY_OP` and `RETURN_VALUE`. The exact names change between Python versions; Python 3.14, for example, prints `LOAD_FAST_BORROW` where older releases print `LOAD_FAST`. For example, older versions printed `BINARY_MULTIPLY` where 3.11 and newer print a general `BINARY_OP`. Do not memorize the opcodes. Learn to read the shape: the function loads two arguments, multiplies, adds, and returns. That is the same arithmetic you wrote, expressed as a stack machine.

The common mistake is believing that Python is "interpreted line by line" and therefore slow in every case. In practice, the whole file is compiled first, and the speed is set by what the functions do with the bytecode. A tight loop in pure Python is slow because each operation goes through the interpreter, which is why libraries such as NumPy push the loop into compiled code. In the job market, performance work starts with profiling, and profilers such as `cProfile` report time per function, which is a view of your code's cost that bytecode makes possible.

### Indentation

Most languages use braces to mark a block. Python uses indentation. A line that ends with a colon, such as `if`, `for`, `def` or `class`, opens a block, and every following line indented at the same level belongs to that block until the indentation drops back. Think of an outline in a document. Each level of bullet sits under the one above it, and the shape of the outline is the structure of the content.

From first principles, the Python tokenizer converts leading spaces into `INDENT` and `DEDENT` tokens, and the parser uses those tokens to build the tree. The language reference specifies that the indentation must be consistent. Mixing tabs and spaces in one file is an error in Python 3, and a block must be indented more than the line that opened it.

Here is a small example that shows the rule:

```python
def grade(score: int) -> str:
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    return "C or below"


print(grade(85))
```

The body of `grade` is indented four spaces. The `if` body is indented eight. The `print` line is back at column zero, so it runs at module level, after the function is defined. Running the file prints `B`.

The common mistake is an "unindent does not match any outer indentation level" error. It means one line is indented by a different amount from the blocks around it, often because an editor converted tabs to spaces partway through. Turn on "show whitespace" in your editor, and set the editor to insert four spaces. The PEP 8 style guide, at peps.python.org/pep-0008/, recommends four spaces per level. In the job market, every Python codebase you will read uses this rule, and most code formatters, such as Ruff, fix indentation automatically.

### Expressions and statements

An expression evaluates to a value. `2 + 3`, `len(items)` and `"a" in text` are expressions. You can put an expression anywhere a value is expected, such as on the right side of an assignment or as an argument to a function.

A statement performs an action. Assignment (`x = 2 + 3`), `if`, `for`, `return` and `import` are statements. A statement does not return a value that you can use in another expression, which is why `x = 5` on its own line does nothing that the REPL shows, while `2 + 3` does.

From first principles, Python 3.8 introduced the assignment expression `:=`, sometimes called the walrus operator, which lets an expression also store a value. It is useful inside a condition when you want the result and the test in one line. Use it sparingly, because a plain assignment is easier to read in most code.

```python
values = [4, 9, 16]
total = sum(values)          # statement: assignment, the value of sum() is stored
average = total / len(values)  # expression on the right side
print(f"total={total}, average={average:.2f}")
```

The output is `total=29, average=9.67`. Each line is one statement, and the right side of each assignment is an expression. The f-string is also an expression, and `print` is called with it as its argument.

The common mistake is trying to assign inside an expression that does not allow a statement, such as a lambda body, and getting a `SyntaxError`. Read the error caret: it points at the token where the parser gave up. In the job market, code review comments often ask a developer to split a long expression into named statements, because the reviewer must read it to understand the logic, and named steps make that possible.

### Naming

A name is a label that points to a value. The label is not the value, and the value is not stored in the label. This is the idea that Day 10 develops in full. For now, focus on the rules for what makes a valid name, and on the conventions that make names easy to read.

From first principles, a Python identifier must start with a letter or underscore, and may then contain letters, digits and underscores. Letters beyond ASCII are allowed, so `café` is a valid name. A keyword cannot be an identifier. The `keyword` module lists the reserved words for the running interpreter, and `str.isidentifier()` tests a string against the grammar.

The conventions come from PEP 8. Use `lower_case_with_underscores` for functions and variables, `CapWords` for classes, and `UPPER_CASE` for constants that a module treats as fixed. A single leading underscore marks a name as internal to a module. A double leading and trailing underscore, such as `__init__`, is reserved for the language and for its special methods, so do not invent new ones.

```python
import keyword

print(keyword.iskeyword("class"))    # True
print("total_price".isidentifier())   # True
print("2fast".isidentifier())         # False, starts with a digit
```

The common mistake is naming a variable after a built-in, such as `list`, `str` or `sum`. The name is then shadowed for the rest of that scope, and the next call to the built-in fails in a confusing way. Use `items`, `text` or `total` instead. A linter such as Ruff, with its `A` rules for builtins, flags this for you. In the job market, naming reviews are routine, and clear names are one of the cheapest ways to make your code look professional.

---

## Hands-on exercises

Work in a folder called `python-runtime-lab` on your machine. Use Python 3.14 or any 3.12 or newer version, and check the version before each exercise.

Exercise 1 confirms your interpreter.

1. Run `python --version` and, on Windows, `py --list`. Note which versions are installed.
2. Create `where.py` with the three-line script from the CPython section and run it.
3. Write down the three values it prints. If `sys.version` is older than 3.12, install a newer Python and rerun with the full path.

Done when the script prints a version of 3.12 or newer and the path of the executable you expected.

Exercise 2 compares the REPL and a script.

1. Start the REPL and type `"héllo".upper()`, then `len("héllo")`, then `"héllo".encode("utf-8")`. Record the three results.
2. Exit with `exit()` and write the same three calls into `text_checks.py`, each wrapped in `print`.
3. Run the file with `python text_checks.py`. Confirm the output matches the REPL.

Done when both runs give the same three results and you can explain why the script needed `print` but the REPL did not.

Exercise 3 uses `python -m` and the `__main__` guard.

1. Create `greeter.py` with a `main` function that prints a greeting, and the `if __name__ == "__main__":` guard.
2. Run it with `python greeter.py` and with `python -m greeter`.
3. Create `importer.py` that contains `import greeter` and prints `"imported"`. Run it. The greeting should not appear.

Done when you can explain why the greeting appears in the first two runs and not in the third.

Exercise 4 reads bytecode.

1. Create `bytecode_peek.py` from the Bytecode section and run it.
2. Add a second function that takes a list and returns its length. Run `dis.dis` on it.
3. Write two sentences describing the stack operations you see for the first function.

Done when you can name the operation that corresponds to the `*` symbol in the source.

Exercise 5 triggers and reads an indentation error.

1. Create `indent_error.py` with a function whose body is indented by three spaces and another line indented by four.
2. Run it and read the error message. Note the line number it reports.
3. Fix the file, enable whitespace display in your editor, and run `python -m py_compile indent_error.py` to confirm the fix.

Done when the compile step exits without printing an error.

---

## Project: a runtime and syntax lab

The project is a small, honest repo: it reports the interpreter it runs under, checks names against the language rules, and prints the bytecode of a function. None of it is large. Its job is to show that you can run Python deliberately and read what it tells you. This is a light project in terms of code, and that is intentional. The skill here is precision about the runtime, and the evidence is the output you commit.

Suggested repo name: `100-days-software-day-09-python-runtime-and-syntax`.

Repo layout:

```
100-days-software-day-09-python-runtime-and-syntax/
├── .gitignore
├── .python-version
├── README.md
├── runtime_report.py
├── naming_check.py
├── bytecode_peek.py
├── tests/
│   ├── __init__.py
│   └── test_naming.py
└── docs/
    └── evidence.md
```

The `.python-version` file records the version the project was built with. Tools such as `pyenv` read it. Write `3.14` into it, or the version you installed.

Starter code for `runtime_report.py`:

```python
import platform
import sys


def report() -> dict[str, str]:
    return {
        "python": sys.version.split()[0],
        "implementation": sys.implementation.name,
        "cache_tag": sys.implementation.cache_tag,
        "executable": sys.executable,
        "platform": platform.platform(),
    }


if __name__ == "__main__":
    if sys.version_info < (3, 12):
        raise SystemExit("This project needs Python 3.12 or newer.")
    for key, value in report().items():
        print(f"{key:>14}: {value}")
```

Starter code for `naming_check.py`:

```python
import keyword


def check_name(name: str) -> str:
    if keyword.iskeyword(name):
        return "reserved keyword"
    if not name.isidentifier():
        return "not a valid identifier"
    if name.startswith("__") and name.endswith("__"):
        return "dunder name, reserved for the language"
    return "ok"


if __name__ == "__main__":
    for candidate in ["total_price", "class", "2fast", "__init__", "Total"]:
        print(f"{candidate:>12} -> {check_name(candidate)}")
```

Starter code for `bytecode_peek.py`:

```python
import dis


def add_tax(price: float, rate: float) -> float:
    return price * (1 + rate)


if __name__ == "__main__":
    dis.dis(add_tax)
```

Starter code for `tests/test_naming.py`:

```python
import unittest

from naming_check import check_name


class CheckNameTests(unittest.TestCase):
    def test_valid_name(self):
        self.assertEqual(check_name("total_price"), "ok")

    def test_keyword(self):
        self.assertEqual(check_name("class"), "reserved keyword")

    def test_digit_start(self):
        self.assertEqual(check_name("2fast"), "not a valid identifier")

    def test_dunder(self):
        self.assertEqual(check_name("__init__"), "dunder name, reserved for the language")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Create the folder, run `git init -b main`, and add the `.gitignore` with lines for `__pycache__/` and `*.pyc`. Create the files above, using the starter code.
2. Run `python runtime_report.py`. Copy the output into `docs/evidence.md` under a heading called "Runtime".
3. Run `python naming_check.py`. Check that each result matches the rules in the Naming section. Add `"lambda"` and `"my-name"` to the list and rerun. Record both results.
4. Run `python -m unittest` from the project root. It should report four tests and `OK`. The `tests/__init__.py` file must exist, or unittest finds zero tests and still prints a misleading result.
5. Run `python bytecode_peek.py`. Copy the first five lines of the output into `docs/evidence.md` and write one sentence explaining what they do.
6. Commit with the message "Add runtime report, naming check and bytecode view". Push the repo to GitHub as you learned on Day 8.

Done when:

- `python runtime_report.py` prints a version of 3.12 or newer and an executable path
- `python -m unittest` runs four tests and passes
- `docs/evidence.md` contains the runtime output, the naming results, and the bytecode excerpt with one explanatory sentence
- `.gitignore` excludes `__pycache__`, and `git status` is clean after the commit
- The GitHub repo shows the commit with those files

Evidence to commit: all the code files, `tests/`, `.python-version`, `.gitignore`, `README.md` (with the run commands) and `docs/evidence.md`.

The README should list the three run commands: `python runtime_report.py`, `python naming_check.py` and `python -m unittest`. A reviewer who clones the repo should be able to run each one without guessing.

---

## Check your understanding

1. You run `python script.py` and get a `SyntaxError` on line 14, but line 14 looks correct. Where in the pipeline did the error occur, and what would you check first?
2. Two developers on the same team get different results from the same script. One is on 3.9 and one on 3.14. Name two runtime facts that could explain the difference, and say how `sys.version` would confirm it.
3. A colleague says that because Python is interpreted, the compiler does not matter. Explain why that is only partly true, referring to what the `__pycache__` folder contains.
4. Why does `python -m mypackage` behave differently from `python mypackage.py` when the package uses relative imports? Name the mechanism that changes.
5. You write `total = 0` and then `total` on its own line in a script. Nothing is printed. Explain why, and say what the REPL would show for the same two lines and why.
6. A student names a variable `list` and then gets a `TypeError` when calling `list(range(3))` later in the file. Explain the mechanism and name the fix.

### Answers

1. The error occurred in the parse stage, before any code ran, because the grammar was violated. Check the line before line 14 first, since a missing closing bracket or a colon on a previous line often surfaces on the next line. The caret in the traceback marks the token where the parser gave up.
2. Two candidates are different language features (for example, the `match` statement exists from 3.10, so code using it fails on 3.9), and different installed library versions. `sys.version` and `sys.executable` confirm which interpreter ran, which is the first step before comparing anything else.
3. The compiler does matter, because source is always compiled to bytecode before it runs. The `__pycache__` folder stores the compiled bytecode per version, using a tag such as `cpython-314`, so the compile step is cached rather than repeated. The part that is "interpreted" is the virtual machine running that bytecode.
4. With `-m`, Python sets the module's package context and places the current directory on `sys.path`, so relative imports resolve against the package. Running the file directly makes it a plain script with no package, so the relative import fails.
5. The assignment `total = 0` is a statement, and a bare name as a statement is an expression that evaluates to a value but is not printed in a script. The REPL prints the value of the expression `total` because it echoes results of expression statements. The script prints nothing unless you call `print`.
6. Assigning `list` to a name shadows the built-in in that module's global scope. Calling `list(range(3))` then calls the integer or list object you assigned, not the type, which raises a `TypeError`. Rename the variable to `items` or similar, and enable a linter rule that flags builtin shadowing.

---

## Sources to read, watch and practice

*Read*

- Python Software Foundation, "The Python Language Reference", chapter "Lexical analysis", at docs.python.org/3/reference/lexical_analysis.html
- Python documentation, "The Python Tutorial", section "An Informal Introduction to Python", at docs.python.org/3/tutorial/
- Python documentation, module `dis`, at docs.python.org/3/library/dis.html
- PEP 8, "Style Guide for Python Code", at peps.python.org/pep-0008/
- [Wikipedia: Python (programming language)](https://en.wikipedia.org/wiki/Python_(programming_language))
- [Wikipedia: CPython](https://en.wikipedia.org/wiki/CPython)
- [Wikipedia: Bytecode](https://en.wikipedia.org/wiki/Bytecode)

*Tutorials*

- GeeksforGeeks, "Python Interpreter", and "Python Keywords" tutorials, at geeksforgeeks.org. Search each title on the site.
- W3Schools, "Python Syntax" and "Python Variables" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the first lectures on Python syntax and functions.

*Practice*

- Python Tutor at pythontutor.com. Paste the bytecode example and the indentation example, and step through each line to see how names and values change. Its visualizations match the mental model in this day.

---

## Tomorrow

Day 10 keeps the runtime in view and asks what a name actually refers to. You will learn how Python objects carry an identity, a type and a value, why two names can point to one object, why assignment does not copy, and how mutability decides what that means in practice. The name rules you learned today are the surface. Tomorrow is the object model underneath.

**Next:** [Day 10: Objects, Variables and Types](Software-Day-10-Objects-Variables-and-Types.md)

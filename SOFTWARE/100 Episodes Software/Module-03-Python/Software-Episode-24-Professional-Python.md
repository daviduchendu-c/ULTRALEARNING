# Day 24: Professional Python

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You have written modules, classes and standard library code from Day 18 to Day 23. This is for anyone whose code works on their own machine and nowhere else, and who wants a project another person can install, format, check and build without guessing.

**By the end of this day, you will have:**

- A virtual environment for each project, created and activated on your system, with dependencies kept out of the global Python
- The ability to install a project in editable mode with `pip`, and to explain runtime and development dependencies
- A project laid out with a `src` directory and a `pyproject.toml` file that declares metadata, dependencies and tool settings
- A repeatable formatting step with Ruff, and a linting step that enforces a clear rule set
- A strict type check with mypy that runs in one command
- A working build that produces a wheel and a source distribution, and an understanding of what each file in the build is for
- A small text-statistics package with a console command and tests, checked by the same commands every time

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`, and on Windows also `py --list`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- Internet access to install packages from PyPI, the Python Package Index, which is free
- Tools installed inside the project's virtual environment in the steps below: `ruff` 0.6 or newer, `mypy` 1.10 or newer, and `build` 1.x (all free, open source)

**Prerequisites:**

- Day 9: Python Runtime and Syntax (`python -m`, scripts)
- Day 18: Modules and Packages (packages, `__init__.py`, `__main__.py`)
- Day 19 and Day 20: OOP I and II (classes used in a real package)
- Day 23: Files and Standard Library (the modules a package is likely to use)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Venv](#venv)
  - [Pip](#pip)
  - [Dependencies](#dependencies)
  - [Project layout](#project-layout)
  - [Formatting](#formatting)
  - [Linting](#linting)
  - [Type checking](#type-checking)
  - [Packaging concepts](#packaging-concepts)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a text statistics package](#project-a-text-statistics-package)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A script that runs on your laptop is not yet a project. A project runs on a colleague's machine, on a build server, and on a server that you will not log into. It needs its dependencies declared, its style checked, its types verified, and its package built the same way each time. The difference between a hobby script and a professional codebase is mostly a set of repeatable commands, and most of them are small. This day gives you the commands and the reasons for each one.

The tools in this day are the ones most Python teams use in 2026. A virtual environment isolates the packages of one project from every other project. `pip` installs them. `pyproject.toml` is the single file that describes the project, as defined in PEP 621. Ruff formats and lints the code, mypy checks the types, and the `build` frontend creates the wheel and source distribution that a package index accepts. Each step is simple alone. The skill is running them in order, every time, and writing them down so that anyone can repeat them.

The official references are the Python Packaging User Guide at packaging.python.org, the `venv` documentation at docs.python.org/3/library/venv.html, and the `pip` user guide at pip.pypa.io. The Ruff documentation at docs.astral.sh/ruff and the mypy documentation at mypy.readthedocs.io describe the two checking tools.

---

## Glossary

- **Virtual environment:** a folder with its own Python, its own `site-packages` and its own scripts, created with `python -m venv`.
- **pip:** the package installer that ships with Python. It downloads and installs packages from PyPI.
- **PyPI:** the Python Package Index, the public repository where most Python packages are published.
- **Dependency:** a package that your project needs in order to run or to develop.
- **Runtime dependency:** a package needed when the project runs.
- **Development dependency:** a package needed only to test, format, check or build the project.
- **Editable install:** an install that links the project's source into the environment, so changes take effect without reinstalling.
- **pyproject.toml:** the project file that declares metadata, build settings, dependencies and tool configuration.
- **src layout:** a project layout in which the package lives in a `src` folder, so the source is not imported by accident from the project root.
- **Formatter:** a tool that rewrites code to a standard style without changing its meaning. Ruff's formatter is one.
- **Linter:** a tool that reports suspicious or unclear code. Ruff's linter is one.
- **Type checker:** a tool that reads type annotations and reports mismatches without running the code. mypy is one.
- **Build backend:** the tool that turns a project into distribution files. setuptools is one.
- **Wheel:** a built, ready-to-install distribution file with the `.whl` extension.
- **sdist:** a source distribution, a tarball with the source and the build instructions.
- **Entry point:** a declared command name that runs a function in your package when installed.

---

## Concepts

The eight concepts follow the order of work on a real project: the environment, the installer, the dependencies, the layout, the formatter, the linter, the type checker, and the build.

### Venv

A virtual environment is a private room for one project. Think of a workshop where each project has its own bench, its own set of tools and its own labelled drawers. Tools you borrow from the shared storeroom are the global Python packages. A project that needs a specific version of a tool keeps its own copy in its own room, and it does not disturb the other rooms.

From first principles, `python -m venv .venv` creates a folder with a copy of the Python launcher, a `site-packages` folder for installed packages, and a `pyvenv.cfg` file that records the base interpreter. Activating the environment puts its `Scripts` folder on the PATH on Windows, or its `bin` folder on Linux and macOS, so that `python` and `pip` refer to the environment's copies. Activation is a convenience. You can always run a tool by its full path inside the environment, which is what the commands in this day do.

```bash
py -3.14 -m venv .venv                 # create the environment (Windows launcher)
source .venv/Scripts/activate          # Git Bash on Windows
.venv\Scripts\Activate.ps1             # PowerShell on Windows
python -c "import sys; print(sys.prefix)"   # shows the environment folder
```

The output of the last line points at the `.venv` folder when the environment is active. If it points at the global installation, the environment is not active, and packages you install will go to the global Python.

The common mistake is installing packages without activating the environment, which puts them in the global Python, so the project works on your machine and fails elsewhere. Check `sys.prefix` before installing. The second mistake is committing `.venv` to Git. Add it to `.gitignore`, because the environment is rebuilt from the project files and should never be shared as a folder. In the job market, a clean environment built from `pyproject.toml` is the first test of whether a project is reproducible.

### Pip

`pip` is the installer that fetches packages from PyPI and places them in the active environment. Think of a shop that delivers parts to your workshop. You name the parts and the version you want, the shop checks what else each part needs, and it delivers all of them. `pip` does the same with the dependencies of each package.

From first principles, `pip install name` downloads the newest compatible release and its dependencies. `pip install "name>=1.2,<2"` adds a version range. `pip install -e .` installs the project in editable mode, which reads the `pyproject.toml` in the current folder and links the source into the environment. `pip install -e ".[dev]"` also installs the optional group named `dev`. `pip list` shows what is installed, and `pip show name` shows the details of one package. `pip` resolves dependencies, but it does not write a lock file by itself, so the exact versions depend on the day you run it.

```bash
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -m pip list
```

The command `python -m pip` is preferred over `pip` on its own, because it runs the pip that belongs to the active interpreter. That avoids installing a package into a different Python by mistake.

The common mistake is running `pip install` with the global Python while the project environment is active, or the reverse. Using `python -m pip` removes that ambiguity. The second mistake is installing packages without writing them into `pyproject.toml`, so the next person cannot reproduce the setup. Every package the project needs should be declared. In the job market, reviewers check that a fresh environment can be built from the repository alone, and that is the purpose of declaring dependencies.

### Dependencies

A dependency is a promise that your project makes to a package. Think of a recipe that lists its ingredients. The recipe lists each ingredient with the quality it needs, such as "flour, plain, at least 500 grams". A version specifier does the same job for a package: `ruff>=0.6` says the code needs ruff at version 0.6 or newer.

From first principles, dependencies come in two groups. Runtime dependencies are listed under `dependencies` in `pyproject.toml`, and they are installed when someone installs your package. Development dependencies, such as formatters and type checkers, belong in an optional group such as `dev`, which installs only when a developer asks for it. For a library, use ranges such as `>=` or `<` to allow the user's environment to choose compatible versions. For an application that you deploy, pin exact versions in a lock file generated by a tool such as `pip-tools` or `uv`, so the deployed environment is reproducible.

```toml
[project]
dependencies = []

[project.optional-dependencies]
dev = ["ruff>=0.6", "mypy>=1.10", "build>=1.2"]
```

This project has no runtime dependencies, because the standard library covers its needs. That is a deliberate choice, and it is worth recording as a design decision.

The common mistake is pinning exact versions in a library's `pyproject.toml`, which forces every user of the library to match your versions and causes conflicts. The opposite mistake is leaving an application with open ranges and no lock file, so the same deployment installs different versions on two days. In the job market, dependency conflicts are a frequent cause of failed builds, and reviewers ask whether a new dependency is necessary before it is added.

### Project layout

A project layout is the map of folders in a repository. Think of a house with a clear separation between the rooms people live in and the storage room. The `src` layout puts the package in a `src` folder, and it keeps the project root for configuration, tests and documentation. Without `src`, running tests from the root may import the package from the folder, not from the installed copy, which hides packaging mistakes.

From first principles, `pyproject.toml` at the root is the single source of configuration. It has a `[build-system]` table that names the build backend, a `[project]` table with the name, version, Python requirement, dependencies and entry points, and `[tool.*]` tables for Ruff, mypy and other tools. The `[tool.setuptools.packages.find]` table tells setuptools to look in `src`. Tests live in a `tests` folder next to `src`, and they import the installed package by its name.

```
textstats/
├── .gitignore
├── README.md
├── pyproject.toml
├── src/
│   └── textstats/
│       ├── __init__.py
│       ├── cli.py
│       └── core.py
└── tests/
    ├── __init__.py
    └── test_core.py
```

The layout is the same for a ten-line utility and a large service, so a developer who knows one project can find their way around another.

The common mistake is mixing configuration files, data files and code in one folder with no structure, which makes the project hard to package. The second mistake is putting the package at the root without `src`, which lets tests pass against the working folder instead of the installed package. In the job market, the `src` layout is the common choice for libraries, and a reviewer notices quickly when a package has a loose layout.

### Formatting

A formatter rewrites code into one standard style. Think of a publisher's house style guide. Two writers can write the same book with different sentence rhythms, and the publisher's editor makes the layout uniform, without changing the meaning. The formatter does the same for indentation, line breaks, quotes and spacing.

From first principles, Ruff's formatter, run with `ruff format`, produces output that matches the Black style in almost every case. It is deterministic: running it twice gives the same result, and running it on formatted code changes nothing. Its settings live in `[tool.ruff]` in `pyproject.toml`, with the line length as the main choice. Formatting is a separate step from checking, and it should be run before a commit or in continuous integration, so that style discussions do not appear in code review.

```bash
python -m ruff format src tests
python -m ruff format --check src tests
```

The first command rewrites files. The second checks without changing anything, which is the form that a CI job uses.

The common mistake is formatting only some files, or formatting by hand, so the style drifts across the repository. Run the formatter over the whole project. A second mistake is disagreeing with the formatter and fighting its output with comments such as `# fmt: off`. Accept the standard style, and reserve exceptions for code that the formatter truly cannot handle. In the job market, teams with a formatter in CI spend less time on style and more time on design.

### Linting

A linter reports code that might be wrong or unclear. Think of a proofreader who points out a repeated word, a sentence that trails off, or a name that is never used. The proofreader flags the places where a careful reader would stop, and leaves the meaning to the author.

From first principles, Ruff's linter, run with `ruff check`, implements rules from several sources. The letter codes select groups: `E` and `F` come from pycodestyle and pyflakes, `I` checks import order, `B` from flake8-bugbear finds likely bugs, such as mutable default arguments from Day 16, and `UP` suggests newer syntax for the target Python version. The `select` list in `[tool.ruff.lint]` chooses the groups, and `ruff check --fix` applies safe automatic fixes. A rule that a team disagrees with can be ignored in the same table, with a comment that says why.

```bash
python -m ruff check src tests
python -m ruff check --fix src tests
```

The first command reports problems. The second applies the fixes Ruff can make safely, such as sorting imports or removing an unused name.

The common mistake is turning off rules to make the report quiet, which hides the problems the rules were chosen to catch. Fix the code, and ignore a rule only with a stated reason. The second mistake is running `--fix` without reading the changes. Review what it changed, especially on code you did not write. In the job market, a clean lint report in CI is a basic expectation, and reviewers use the rule codes to point to a fix quickly.

### Type checking

A type checker reads the annotations you wrote and reports the places where a value does not match its declared type. Think of an inspector who checks a blueprint against the building code before construction starts. The inspector builds nothing and finds the mismatches early, when they are cheap to fix.

From first principles, mypy reads the annotations in your files, infers types where you did not write them, and checks every call and assignment. The `strict = true` setting turns on the checks that catch the most problems, such as missing annotations, implicit `Any` and untyped functions. `mypy_path = "src"` tells mypy where to find the package when the code is not installed. mypy does not run the program. Annotations remain optional at runtime, so a passing check is evidence of consistency, not a guarantee against every error.

```bash
python -m mypy src tests
```

A clean run prints `Success: no issues found in N source files`. A failed run prints one line per problem, with the file and line.

The common mistake is adding `# type: ignore` to silence an error without understanding it, which removes the protection at that point. Add the ignore only when you can state why the checker is wrong, and include the error code, such as `# type: ignore[arg-type]`. The second mistake is running the checker on the source only and not on the tests, so the tests drift away from the types. In the job market, typed code is standard in serious teams, and a strict check in continuous integration is a common requirement.

### Packaging concepts

Packaging turns a project into files that someone else can install. Think of shipping a piece of furniture. The manufacturer packs the parts, includes the instructions and labels the box with its name and version. The buyer opens the box and assembles the furniture at home. A wheel is the assembled-ready box, and an sdist is the box of parts with the instructions for building it.

From first principles, a build frontend such as `python -m build` calls the build backend named in `pyproject.toml`. The backend reads the metadata, collects the package files, and writes a wheel, which is a zip file with a `.whl` extension, and an sdist, which is a compressed tarball with the source. The wheel name encodes the project name, version, Python tag and platform tag, so `textstats-0.1.0-py3-none-any.whl` is a pure-Python wheel that works on any platform. Entry points in `[project.scripts]` create a command in the user's environment that calls a function in your package. Publishing to PyPI is a separate step, and this day stops at the built files.

```bash
python -m build
ls dist
```

The `dist` folder holds the wheel and the sdist. Installing the wheel in a clean environment is the test that matters, because it checks what a user will actually receive.

The common mistake is building the package and testing only the source tree, which hides missing files in the distribution. Install the wheel into a new environment and run the command from there. The second mistake is forgetting that `dist`, `build` and `*.egg-info` are generated outputs and should not be committed. In the job market, release problems are often packaging problems, such as a missing data file or an entry point that points at the wrong function, and the test is always an install from the built artifact.

---

## Hands-on exercises

Work in a folder called `pro-lab`, using Python 3.14 or 3.12 or newer. Run every command from the folder that contains the project.

Exercise 1 creates and checks a virtual environment.

1. Create a folder `envcheck`, change into it, and create a virtual environment with `py -3.14 -m venv .venv` (or `python3.14 -m venv .venv` on Linux and macOS).
2. Activate it. Run `python -c "import sys; print(sys.prefix)"` and record the output.
3. Deactivate it, run the same command again, and record the difference. Write one sentence on why the paths differ.

Done when the first output points inside `.venv` and the second points at the global installation.

Exercise 2 installs a package with pip.

1. In the activated environment, run `python -m pip install ruff` and `python -m pip install mypy`.
2. Run `python -m pip list` and record the names and versions of the two packages.
3. Run `python -m pip show ruff` and note the `Location` line. Write one sentence on what the location shows.

Done when both packages appear in the list, and the location is inside your `.venv` folder.

Exercise 3 runs a formatter and a linter on messy code.

1. Create `messy.py` with an unused import, a line longer than 88 characters, and two imports in the wrong order.
2. Run `python -m ruff check messy.py` and record the rule codes that appear.
3. Run `python -m ruff format messy.py`, then `python -m ruff check --fix messy.py`, then check again. Record whether the second check is clean.

Done when the formatter changes the layout and the linter reports no problems after the fix.

Exercise 4 runs a strict type check.

1. Create `typed_demo.py` with a function `average(values: list[float]) -> float` that returns the mean.
2. Call it once with a list of strings, inside the file, so that the error is visible to mypy.
3. Run `python -m mypy --strict typed_demo.py`, read the error, fix the call, and rerun until mypy reports no issues.

Done when mypy reports `Success: no issues found in 1 source file`.

Exercise 5 builds a wheel and inspects it.

1. Install the build frontend with `python -m pip install build`.
2. In any small project with a `pyproject.toml`, run `python -m build`. Note the files in `dist`.
3. Open the wheel with `python -m zipfile -l dist/<name>.whl` and record the first five entries. Write one sentence on what the `.dist-info` folder contains.

Done when the wheel and sdist exist in `dist` and you can name the file that holds the metadata.

---

## Project: a text statistics package

The project is a small, installable package called `textstats`. It counts the most common words in a text, estimates a reading time, and exposes both through a `textstats` console command. The code uses only the standard library. The project's real subject is the workflow around the code: a virtual environment, a `pyproject.toml`, a src layout, Ruff for formatting and linting, a strict mypy check, tests, and a built wheel. The evidence is the set of command outputs and the wheel listing, all recorded in the findings file.

Suggested repo name: `100-days-software-day-24-professional-python`.

Repo layout:

```
100-days-software-day-24-professional-python/
├── .gitignore
├── README.md
├── pyproject.toml
├── src/
│   └── textstats/
│       ├── __init__.py
│       ├── cli.py
│       └── core.py
├── tests/
│   ├── __init__.py
│   └── test_core.py
└── docs/
    └── findings.md
```

Starter code for `pyproject.toml`:

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "textstats"
version = "0.1.0"
description = "Small text statistics helpers."
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
dev = ["ruff>=0.6", "mypy>=1.10", "build>=1.2"]

[project.scripts]
textstats = "textstats.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.mypy]
strict = true
mypy_path = "src"
files = ["src", "tests"]

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
disallow_incomplete_defs = false
```

The override relaxes one strict rule for the tests only. Library code must be fully annotated, and test methods may stay unannotated. That is a common team decision, and the project should state it in the findings file, as step 12 asks.

Starter code for `src/textstats/__init__.py`:

```python
"""Text statistics helpers: common words and reading time."""

from textstats.core import reading_minutes, top_words, words

__all__ = ["reading_minutes", "top_words", "words"]
__version__ = "0.1.0"
```

Starter code for `src/textstats/core.py`:

```python
import re
from collections import Counter

WORD = re.compile(r"[A-Za-z']+")


def words(text: str) -> list[str]:
    return [match.group(0).lower() for match in WORD.finditer(text)]


def top_words(text: str, n: int = 3) -> list[tuple[str, int]]:
    if n < 0:
        raise ValueError("n must be zero or greater")
    return Counter(words(text)).most_common(n)


def reading_minutes(text: str, wpm: int = 200) -> float:
    if wpm <= 0:
        raise ValueError("wpm must be positive")
    return round(len(words(text)) / wpm, 2)
```

Starter code for `src/textstats/cli.py`:

```python
import sys

from textstats.core import reading_minutes, top_words


def main(argv: list[str] | None = None) -> int:
    args = sys.argv[1:] if argv is None else argv
    text = " ".join(args)
    if not text.strip():
        print("usage: textstats WORDS...", file=sys.stderr)
        return 2
    for word, count in top_words(text):
        print(f"{word} {count}")
    print(f"reading time: {reading_minutes(text)} min")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Starter code for `tests/test_core.py`:

```python
import contextlib
import io
import unittest

from textstats import reading_minutes, top_words, words
from textstats.cli import main


class CoreTests(unittest.TestCase):
    def test_words_are_lowercase_and_keep_apostrophes(self):
        self.assertEqual(words("Don't stop, don't!"), ["don't", "stop", "don't"])

    def test_top_words_counts(self):
        self.assertEqual(top_words("a b a c a b", 2), [("a", 3), ("b", 2)])

    def test_top_words_rejects_negative_n(self):
        with self.assertRaises(ValueError):
            top_words("a", -1)

    def test_reading_minutes(self):
        self.assertEqual(reading_minutes("word " * 400, 200), 2.0)

    def test_reading_minutes_rejects_zero_speed(self):
        with self.assertRaises(ValueError):
            reading_minutes("word", 0)

    def test_cli_prints_counts(self):
        buffer = io.StringIO()
        with contextlib.redirect_stdout(buffer):
            code = main(["hello", "hello", "world"])
        self.assertEqual(code, 0)
        self.assertIn("hello 2", buffer.getvalue())

    def test_cli_empty_input_is_usage_error(self):
        with contextlib.redirect_stderr(io.StringIO()):
            self.assertEqual(main([""]), 2)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Create the repo folder, run `git init -b main`, and create the files above. Add `.venv/`, `dist/`, `build/`, `*.egg-info/` and `__pycache__/` to `.gitignore`.
2. Create the environment with `py -3.14 -m venv .venv` and activate it. Record the `sys.prefix` output in `docs/findings.md`.
3. Install the project in editable mode with `python -m pip install -e ".[dev]"`. Record the last line of the install output.
4. Run `python -m ruff format --check src tests`. If it reports files, run `python -m ruff format src tests` and record which files changed.
5. Run `python -m ruff check src tests` and fix every reported problem. Record the final line in `docs/findings.md`.
6. Run `python -m mypy` with the settings from `pyproject.toml` and fix every issue. Record the success line.
7. Run `python -m unittest discover -s tests -t .` and make every test pass. Record the count.
8. Run `textstats hello hello world` from the command line and paste the output into `docs/findings.md`.
9. Run `python -m build`, list `dist`, and open the wheel with `python -m zipfile -l dist/textstats-0.1.0-py3-none-any.whl`. Record the file names in `docs/findings.md`.
10. Create a second environment in a new folder, install the wheel with `python -m pip install dist/textstats-0.1.0-py3-none-any.whl`, and run `textstats hello`. Record that the command works outside the source tree.
11. Write a section in `docs/findings.md` titled "Why no runtime dependencies", in at least three sentences.
12. Commit with the message "Add textstats package with formatting, linting, typing and build" and push to GitHub.

Done when:

- `python -m ruff format --check src tests` and `python -m ruff check src tests` both pass
- `python -m mypy` reports no issues across `src` and `tests`
- `python -m unittest discover -s tests -t .` runs at least seven tests and passes
- `docs/findings.md` contains the environment path, the install line, the lint and type results, the test count, the CLI output, the wheel listing, the wheel test output, and the dependency section
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: `pyproject.toml`, the source and test files, `.gitignore`, `README.md` (with the setup and check commands in order), and `docs/findings.md`. Do not commit `.venv`, `dist` or `build`.

The package is small so that the workflow is the thing to study. A reviewer should be able to clone the repo, follow the README, and reproduce every check and the wheel without asking you a question.

---

## Check your understanding

1. A developer installs `ruff` with the global `pip`, then the project's tests fail on a teammate's machine that does not have ruff. Explain what went wrong, and state the change that makes the dependency reproducible.
2. A library's `pyproject.toml` pins `requests==2.31.0` as a runtime dependency. Explain the problem this causes for users of the library, and suggest the range that fits a library.
3. A `src` layout is used, and a test imports the package without installing it. Explain why the test might pass on the developer's machine but fail in CI, and state the command that prevents that.
4. A formatter and a linter both run in CI. Explain the difference between their jobs, and name one problem each would report that the other would not.
5. A team adds `# type: ignore` to every mypy error without comment. Explain what this does to the value of the type checker, and state the correct use.
6. A wheel installs and the `textstats` command is missing. Name the two places in `pyproject.toml` that could cause this, and the command that checks the entry point after installation.

### Answers

1. The project depends on a tool installed in the developer's global Python, so the teammate's environment does not have it. The change is to declare `ruff` in `[project.optional-dependencies] dev` and to install the project with `pip install -e ".[dev]"` in a virtual environment, so every machine gets the same declared tools.
2. Pinning an exact version in a library forces every user to use that version, which causes conflicts when another library needs a different one. A library should declare a range such as `requests>=2.31,<3`, which lets users choose compatible versions. Exact pins belong in an application's lock file.
3. In the src layout the package is not on the path by default, so a test that imports it works only after the project is installed. A developer may still see a pass because an older installed copy, or a stale build, is present on their machine, while a clean CI machine fails at the import. The check that catches this is installing the project into a fresh environment with `python -m pip install -e ".[dev]"` before running the tests, and in CI doing the same on every run.
4. The formatter rewrites layout, such as line length and spacing, and it never reports logic problems. The linter reports rule violations such as an unused import, a bare `except`, or a mutable default argument. Each one catches different things, which is why both run.
5. The type checker stops reporting real mismatches at those lines, so a later change that breaks a type goes unnoticed. The correct use is `# type: ignore[error-code]` on a specific line, with a comment that states why mypy is wrong for that case.
6. The `[project.scripts]` table might be missing the `textstats` entry, or the entry point might name the wrong module or function, such as `textstats.cli:run` when the function is `main`. The check is `python -m pip show -f textstats` to confirm the package is installed, followed by running `textstats --help` or `textstats hello` from a new shell, and checking that the script exists in the environment's `Scripts` or `bin` folder.

---

## Sources to read, watch and practice

*Read*

- Python Packaging User Guide, "Writing your pyproject.toml", at packaging.python.org
- Python documentation, module `venv`, at docs.python.org/3/library/venv.html
- pip user guide, "Installing packages", at pip.pypa.io
- PEP 621, "Storing project metadata in pyproject.toml", at peps.python.org/pep-0621/
- PEP 8, "Style Guide for Python Code", at peps.python.org/pep-0008/
- Ruff documentation, "Rules" and "Formatter", at docs.astral.sh/ruff
- mypy documentation, "Command line" and "Running mypy in strict mode", at mypy.readthedocs.io
- [Wikipedia: Software build](https://en.wikipedia.org/wiki/Software_build)
- [Wikipedia: Lint (software)](https://en.wikipedia.org/wiki/Lint_(software))

*Tutorials*

- GeeksforGeeks, "Python Virtual Environment" and "Python pip" tutorials, at geeksforgeeks.org
- W3Schools, "Python Virtualenv" page, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on modules and on running programs from the command line.
- The mypy channel and the Ruff documentation videos, which show type checking and linting on real code. Search each title in the official channel.

*Practice*

- The Python Packaging User Guide's tutorial on packaging a project at packaging.python.org. Follow it on a throwaway project to see the same build commands used today.

---

## Tomorrow

Day 25 is testing. You will write unit tests that check one function in isolation, use fixtures to set up shared state, write mocks to replace slow or external parts, and organize tests into integration suites that exercise several modules together. The `unittest` and `discover` commands from today are the runner for those tests, and the checks you set up here will run them on every change.

**Next:** [Day 25: Testing Python](Software-Day-25-Testing-Python.md)

# Day 18: Modules and Packages

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You have written functions in single files from Day 9 to Day 17. This is for anyone whose project has grown past one file, and who has hit an `ImportError`, a name clash with a standard library module, or a script that runs code when another file imports it.

**By the end of this day, you will have:**

- A working model of the `import` statement: how Python finds a module, runs it once, and caches it
- The ability to read and control a module's namespace, and to choose between `import x` and `from x import y` on purpose
- A correct use of `if __name__ == "__main__":` so that a file works as both a script and an importable module
- The ability to build a regular package with `__init__.py`, a `__main__.py` entry point and an `__all__` list
- A clear rule for choosing absolute or relative imports, and the habit of avoiding names that shadow the standard library
- A shapes-app package with tests and a command-line entry point, run with `python -m`

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2

**Prerequisites:**

- Day 9: Python Runtime and Syntax (`python -m`, the `__main__` guard, script mode)
- Day 16: Functions I (defining and calling functions)
- Day 17: Functions II (keyword arguments, closures used as helpers)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Imports](#imports)
  - [Namespaces](#namespaces)
  - [Name](#name)
  - [Package structure](#package-structure)
  - [Relative and absolute imports](#relative-and-absolute-imports)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a shapes app package](#project-a-shapes-app-package)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A program with one file is easy to hold in your head. A program with forty files needs structure, or no one can find anything. Modules and packages are how Python lets you split code into files and folders, and how those pieces find each other. Every library you install with `pip` is a package, every standard library feature you use is a module, and every test in a serious project imports the code it checks.

Most import problems come from where Python looks, what runs when a file loads, and which names the file exposes. A project that names a file `random.py` can break the standard library's `random` module for every file in that folder. A script that creates a database connection at import time runs that code whenever a test imports it. A package whose `__init__.py` imports half the world makes startup slow. This day gives you the model to avoid those problems.

The official reference is the Python tutorial section "Modules" at docs.python.org/3/tutorial/modules.html, followed by the language reference section "The import system" at docs.python.org/3/reference/import.html. The tutorial is short and covers most of what you need. The reference is the authority when you need an exact rule.

---

## Glossary

- **Module:** a Python file, or a built-in or compiled extension, that defines names. Its namespace is the set of names it defines.
- **Package:** a module that contains other modules, usually a folder with an `__init__.py` file.
- **Regular package:** a package with an `__init__.py` file, which Python recognizes directly.
- **Namespace package:** a package without an `__init__.py` file, which Python assembles from one or more folders. Use it rarely.
- **Import:** the process of finding, loading and binding a module or its names.
- **sys.path:** a list of folders that Python searches for modules, in order.
- **sys.modules:** a dictionary of modules that have already been imported, keyed by their full names.
- **Namespace (in this context):** the mapping from names to objects in a module, which you see with `vars(module)`.
- **`__name__`:** a module attribute that holds the module's full name, or `"__main__"` when the file is run directly.
- **`__init__.py`:** a file that runs when a package is imported, and that defines what the package exposes.
- **`__main__.py`:** a file that runs when a package is started with `python -m package_name`.
- **`__all__`:** a list of names that `from module import *` exports.
- **Relative import:** an import that starts with dots, such as `from .area import rectangle_area`, and resolves against the current package.
- **Absolute import:** an import that starts with the top-level package name, such as `from shapes_app.geometry import rectangle_area`.

---

## Concepts

The concepts follow an import from the moment Python sees the statement to the layout that makes imports easy to read.

### Imports

An import is a request for a module, and Python treats it like a library checkout. Think of borrowing a book from a library. You check the catalog, find the shelf, take the book, and the library keeps a record that you have it. If someone else asks for the same book, the library hands over the copy it already has rather than printing a new one.

From first principles, the first time a module is imported, Python searches for it along `sys.path`, which starts with the folder of the script being run (or the current directory for `python -m`), then the standard library and the installed packages. It compiles the module, creates a module object, stores it in `sys.modules`, and runs the module's top-level code. The second import finds the module in `sys.modules` and returns the same object without running the code again. This is why a module's top-level `print` appears only once in a process.

```python
import sys

import math

print("math" in sys.modules)     # True, math was just imported
print(sys.modules["math"] is math)  # True, the same object
print(sys.path[0])               # the folder Python searches first
```

The `sys.path[0]` value depends on how you ran the program. For a script, it is the script's folder. For `python -m`, it is the current working directory. Knowing this explains many "module not found" errors.

The common mistake is naming your own file after a standard library module, such as `random.py`, `json.py` or `email.py`. Python finds your file first when it searches the script's folder, so `import random` imports your file instead of the library, and the program fails in confusing ways. Rename the file. The second mistake is importing inside a function without a reason, which hides the dependency and delays the error. In the job market, reviewers ask for top-level imports except where a slow import is justified.

### Namespaces

A namespace is a labeled set of drawers. Each module is one set of drawers, and each name in the module is a drawer labeled with the name. Two modules can each have a drawer labeled `area` without conflict, because you open them as `geometry.area` and `reporting.area`.

From first principles, `import math` binds one name, `math`, in the current namespace to the module object. You reach the module's names with a dot: `math.pi`. `from math import pi` binds `pi` directly in the current namespace, and it does not bind `math`. `from math import *` binds every public name, which pollutes the namespace and makes it unclear where a name came from. The `as` clause renames the binding, as in `import numpy as np`, which is a common convention.

```python
import math
from math import sqrt as square_root

print(math.pi)             # 3.141592653589793
print(square_root(16))     # 4.0
print("pi" in dir())      # False, pi is not bound here
print(vars(math)["e"])     # 2.718281828459045
```

The `dir()` call lists the names in the current namespace. `pi` is not there, because only `math` and `square_root` were bound. The `vars(math)` call shows the module's own namespace as a dictionary.

The common mistake is using `from module import *` in production code. It makes it impossible to tell where a name came from, and it can overwrite names you already defined. A second mistake is importing two modules that define the same name with `from ... import`, so the second import silently replaces the first. Import the module and use the dot, unless a short name is clearly better. In the job market, code reviewers flag wildcard imports immediately, and linters such as Ruff have a rule for them.

### Name

`__name__` is a module attribute that tells the module how it was started. Think of a stage actor who knows whether they are rehearsing alone in the studio or performing in front of an audience. The same file can be the star of a show when it is run directly, and a supporting part when it is imported by another file.

From first principles, when Python runs a file as the main program, it sets `__name__` to the string `"__main__"`. When another module imports the file, `__name__` is the module's full name, such as `hello` or `shapes_app.cli`. The standard idiom `if __name__ == "__main__":` runs a block only in the first case. Code outside any function runs when the module is loaded, so anything that is not a definition should either sit behind that guard or live in a function that the guard calls.

```python
def main() -> None:
    print("running as", __name__)


if __name__ == "__main__":
    main()
```

Running `python hello.py` prints `running as __main__`. Importing the module from another file prints nothing, because the guard is false, and the file defines `main` without calling it.

The common mistake is putting a long-running action at the top level of a module, such as opening a network connection or starting a server. Every test that imports the module then starts the action, and the tests hang or fail. Put the action in a function and call it from the guard. In the job market, a clean entry point with `main` and the guard is expected in every script, and it makes the code testable.

### Package structure

A package is a folder of modules that Python can import as one name. Think of a department store with floors. Each floor is a package, each shelf is a module, and the store directory tells you where to find things. The `__init__.py` file is the directory at the entrance of the floor. It runs when the floor is first visited, and it decides which items are shown at the front.

From first principles, a regular package is a folder with an `__init__.py` file. Python imports the package by running that file, and it imports submodules when they are named. A file `geometry/area.py` is the module `geometry.area`. Inside `__init__.py`, a line such as `from .area import rectangle_area` makes the function available as `geometry.rectangle_area`. The `__all__` list controls what `from geometry import *` exports. A `__main__.py` file inside the package lets you run `python -m package_name` as an entry point.

```
shapes_app/
├── __init__.py
├── __main__.py
├── cli.py
├── reporting.py
└── geometry/
    ├── __init__.py
    ├── area.py
    └── perimeter.py
```

This layout separates the geometry rules from the reporting and the command line. Each folder has one job, and the file names say what is inside.

The common mistake is making `__init__.py` do too much work, such as reading files or opening connections, so importing the package has side effects. Keep `__init__.py` for exports and version constants. The second mistake is forgetting `__init__.py`. The folder then becomes a namespace package, which works for some tools but breaks others, and it is easy to miss in a code review. In the job market, package layout is one of the first things reviewers look at, and a clear `__init__.py` with a docstring is a sign of a maintained codebase.

### Relative and absolute imports

An import path can start from the top of the project or from the current package. Think of giving directions. "Go to Main Street, number 12" is an absolute address that works from anywhere. "Two doors down the hall" is a relative address that works only from where you are. Both are correct, and each fits a different situation.

From first principles, an absolute import names the module from the top-level package, such as `from shapes_app.geometry import rectangle_area`. A relative import begins with one or more dots, which count levels upward from the current package. `from .area import rectangle_area` means "in this package, the area module". `from ..other import thing` goes up one package. Relative imports work only inside a package. A file run directly as a script has no package context, so a relative import in it raises an error. Running it with `python -m` fixes that, because the module then knows its package.

```python
# inside shapes_app/reporting.py
from .geometry import circle_area, rectangle_area   # relative
# from shapes_app.geometry import circle_area       # absolute, also valid
```

The absolute form is clearer when the code is read outside its folder. The relative form is shorter and survives renaming the top-level package. Most teams standardize on absolute imports for the whole project and use relative imports only inside a package when the project asks for it.

The common mistake is writing a relative import in a file that is run directly, which fails with `ImportError: attempted relative import with no known parent package`. Run the file with `python -m package.module`, or switch to an absolute import. The second mistake is importing a module that depends on the current working directory, which works on one machine and fails on another. In the job market, import errors in CI are often caused by running from a different folder, and the fix is to use `python -m` from the project root.

---

## Hands-on exercises

Work in a folder called `modules-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 explores the module cache.

1. Create `cache_demo.py`. Import `math` and check that it is in `sys.modules`.
2. Create `printer.py` with a top-level `print("loaded")`. Import it twice from `cache_demo.py` and record how many times `loaded` prints.
3. Write one sentence explaining why the second import does not run the module again.

Done when `loaded` prints once and your sentence names `sys.modules`.

Exercise 2 practices namespaces.

1. Create `names.py`. Use `import math`, then `from math import sqrt as root`. Print `math.pi`, `root(81)`, and whether `"sqrt"` is in `dir()`.
2. Use `from math import *` in a second file, and list the names that appear in `dir()`. Write one sentence on why this is harder to read than the first file.

Done when you can explain the difference between `dir()` in the two files.

Exercise 3 uses `__name__` and the main guard.

1. Create `greeter.py` with a `main` function and the guard from the concepts section.
2. Run it with `python greeter.py`, then with `python -m greeter`, and record both outputs.
3. Create `importer.py` that imports `greeter` and prints `"imported"`. Run it and confirm that the greeting does not appear.

Done when the greeting appears in the first two runs but not in the third.

Exercise 4 builds a small package.

1. Create a folder `mathtools/` with an empty `__init__.py`, a `basic.py` that defines `add(a, b)`, and a `__main__.py` that prints `add(2, 3)`.
2. Run `python -m mathtools` from the parent folder and record the output.
3. Add an `__init__.py` line that imports `add` from `.basic` and lists it in `__all__`. Import `mathtools.add` from a new file and confirm it works.

Done when `python -m mathtools` prints `5` and the package-level import works.

Exercise 5 shows the relative import error.

1. Inside `mathtools/`, create `report.py` with `from .basic import add`.
2. Run `python mathtools/report.py` and record the error.
3. Run `python -m mathtools.report` from the parent folder and confirm the import works. Write one sentence on why the two runs differ.

Done when the first run fails with a relative import error and the second succeeds.

---

## Project: a shapes app package

The project is a small package with a geometry subpackage, a reporting module, a command-line interface, and a `__main__.py` entry point. The code is simple, and the structure is the lesson. The evidence is a passing test suite, a working `python -m shapes_app` command, and a findings file that shows the package layout and one import error you fixed.

Suggested repo name: `100-days-software-day-18-modules-and-packages`.

Repo layout:

```
100-days-software-day-18-modules-and-packages/
├── .gitignore
├── README.md
├── shapes_app/
│   ├── __init__.py
│   ├── __main__.py
│   ├── cli.py
│   ├── reporting.py
│   └── geometry/
│       ├── __init__.py
│       ├── area.py
│       └── perimeter.py
├── tests/
│   ├── __init__.py
│   └── test_geometry.py
└── docs/
    └── findings.md
```

Starter code for `shapes_app/__init__.py`:

```python
"""Shapes app: geometry helpers, text reports and a command-line entry point."""

__version__ = "0.1.0"
```

Starter code for `shapes_app/geometry/area.py`:

```python
import math


def rectangle_area(width: float, height: float) -> float:
    return width * height


def circle_area(radius: float) -> float:
    return math.pi * radius**2
```

Starter code for `shapes_app/geometry/perimeter.py`:

```python
def rectangle_perimeter(width: float, height: float) -> float:
    return 2 * (width + height)
```

Starter code for `shapes_app/geometry/__init__.py`:

```python
"""Geometry helpers. Import from here, not from the individual modules."""

from .area import circle_area, rectangle_area
from .perimeter import rectangle_perimeter

__all__ = ["circle_area", "rectangle_area", "rectangle_perimeter"]
```

Starter code for `shapes_app/reporting.py`:

```python
from .geometry import circle_area, rectangle_area, rectangle_perimeter


def rectangle_report(width: float, height: float) -> str:
    area = rectangle_area(width, height)
    perimeter = rectangle_perimeter(width, height)
    return f"rectangle {width}x{height}: area={area:.2f}, perimeter={perimeter:.2f}"


def circle_report(radius: float) -> str:
    return f"circle r={radius}: area={circle_area(radius):.2f}"
```

Starter code for `shapes_app/cli.py`:

```python
import sys

from shapes_app.reporting import circle_report, rectangle_report

USAGE = "usage: python -m shapes_app rect WIDTH HEIGHT | circle RADIUS"


def main(argv: list[str] | None = None) -> int:
    args = sys.argv[1:] if argv is None else argv
    try:
        if args[:1] == ["rect"] and len(args) == 3:
            print(rectangle_report(float(args[1]), float(args[2])))
            return 0
        if args[:1] == ["circle"] and len(args) == 2:
            print(circle_report(float(args[1])))
            return 0
    except ValueError:
        pass
    print(USAGE)
    return 1
```

Starter code for `shapes_app/__main__.py`:

```python
import sys

from shapes_app.cli import main

if __name__ == "__main__":
    sys.exit(main())
```

Starter code for `tests/test_geometry.py`:

```python
import contextlib
import io
import math
import unittest

from shapes_app.cli import main
from shapes_app.geometry import circle_area, rectangle_area, rectangle_perimeter
from shapes_app.reporting import circle_report, rectangle_report


class GeometryTests(unittest.TestCase):
    def test_rectangle_area_and_perimeter(self):
        self.assertEqual(rectangle_area(3, 4), 12)
        self.assertEqual(rectangle_perimeter(3, 4), 14)

    def test_circle_area(self):
        self.assertAlmostEqual(circle_area(1), math.pi)

    def test_reports(self):
        self.assertEqual(rectangle_report(3, 4), "rectangle 3x4: area=12.00, perimeter=14.00")
        self.assertEqual(circle_report(1), "circle r=1: area=3.14")

    def test_cli_rectangle(self):
        buffer = io.StringIO()
        with contextlib.redirect_stdout(buffer):
            code = main(["rect", "3", "4"])
        self.assertEqual(code, 0)
        self.assertIn("area=12.00", buffer.getvalue())

    def test_cli_bad_input_returns_usage(self):
        buffer = io.StringIO()
        with contextlib.redirect_stdout(buffer):
            code = main(["rect", "x", "4"])
        self.assertEqual(code, 1)
        self.assertIn("usage:", buffer.getvalue())

    def test_package_exports(self):
        import shapes_app.geometry as geometry

        self.assertIn("rectangle_area", geometry.__all__)


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python -m shapes_app rect 3 4` from the project root. Paste the output into `docs/findings.md`. Then run `python -m shapes_app circle 2` and paste that output too.
3. Run `python shapes_app/cli.py rect 3 4` and record the error. Explain in `docs/findings.md` why it fails, referring to the absolute import and the `sys.path[0]` value.
4. Run `python -m unittest` from the project root and make every test pass.
5. Add a function `square_area(side: float) -> float` to `area.py`, export it through `geometry/__init__.py`, and add a test that imports it from `shapes_app.geometry`.
6. Run `python -c "import sys, shapes_app; print(sorted(vars(shapes_app)))"` and paste the output into `docs/findings.md`. Explain in one sentence what `__version__` and `__all__` are doing.
7. Run `python -m mypy shapes_app tests` and fix every issue.
8. Write a paragraph in `docs/findings.md` that compares absolute and relative imports in this project, and states which one you used in `cli.py` and why.
9. Commit with the message "Add shapes app package with entry point and tests" and push to GitHub.

Done when:

- `python -m shapes_app rect 3 4` prints the rectangle report, and `python -m shapes_app` with bad input prints the usage line
- `python -m unittest` runs at least seven tests and passes
- `python -m mypy shapes_app tests` reports no issues
- `docs/findings.md` contains the command outputs, the error from step 3, the namespace output, and the import paragraph
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the package code, the tests, `docs/findings.md`, `README.md` (with the `python -m` commands), and `.gitignore`.

The package is small so the structure is easy to inspect. A reviewer should be able to see where each responsibility lives, why the entry point is a `__main__.py` and not a loose script, and why the tests import from the package rather than from file paths.

---

## Check your understanding

1. A student creates `random.py` in a project folder, and then `import random` fails with an unexpected error in an unrelated file. Explain the search order that causes this, and name the fix.
2. Two modules define a function named `parse`. One file does `from a import parse` and then `from b import parse`. Which function does the file call, and what would make the code clearer?
3. A file `tools.py` contains `result = expensive_setup()` at top level and no guard. Explain what happens when a test imports it, and rewrite the file so the setup runs only when the file is run as a script.
4. A package has an `__init__.py` that imports every module in the package. Name one problem this causes, and say what the file should contain instead.
5. A file `package/inner.py` contains `from .outer import helper`. Running `python package/inner.py` fails. Explain why, and give two ways to fix it.
6. Explain the difference between `import shapes_app.geometry` and `from shapes_app.geometry import rectangle_area` in terms of which names end up in the importing file's namespace.

### Answers

1. Python searches the script's folder first, so `import random` finds your `random.py` before the standard library module. Other files in the folder then get your file instead of the library. The fix is to rename the file, and to delete any leftover `__pycache__` files that were compiled from it.
2. The file calls the second `parse`, because `from b import parse` rebinds the name after `from a import parse`. Clearer code uses `import a` and `import b`, then `a.parse` and `b.parse`, so each name points to its module.
3. When a test imports `tools.py`, the `expensive_setup()` call runs at import time, which may be slow or have side effects. Rewrite the file with `def main(): result = expensive_setup()` and `if __name__ == "__main__": main()`, so the setup runs only when the file is executed directly.
4. Importing the package runs every submodule, which slows startup and can create circular imports. The `__init__.py` should contain only the names meant for callers, such as a docstring, a version and an `__all__` list, with imports limited to those names.
5. Running the file directly means it has no package context, so the relative import has no parent package to resolve against. Fix it by running `python -m package.inner` from the parent folder, or by changing the import to the absolute form `from package.outer import helper`.
6. `import shapes_app.geometry` binds only `shapes_app` in the importing file, and you reach the functions through the dotted path. `from shapes_app.geometry import rectangle_area` binds only `rectangle_area`, and the module names are not bound at all.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Modules", at docs.python.org/3/tutorial/modules.html
- Python language reference, "The import system", at docs.python.org/3/reference/import.html
- Python documentation, "Packages", section of the tutorial, at docs.python.org/3/tutorial/modules.html#packages
- PEP 8, "Imports" section, at peps.python.org/pep-0008/
- [Wikipedia: Modular programming](https://en.wikipedia.org/wiki/Modular_programming)
- [Wikipedia: Namespace](https://en.wikipedia.org/wiki/Namespace)

*Tutorials*

- GeeksforGeeks, "Python Modules" and "Python __name__ variable" tutorials, at geeksforgeeks.org
- W3Schools, "Python Modules" and "Python Packages" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on modules and libraries.

*Practice*

- Python Tutor at pythontutor.com. Step through a two-file example in single-file form to see how the module namespace is built, then compare it with the package layout in this day.

---

## Tomorrow

Day 19 begins object-oriented programming. You will define classes, create instances, store attributes, write methods, and use the special `__init__` method. You will also learn the difference between instance methods, class methods and static methods, and when each one belongs in a design. The packages you built today will hold the classes you write tomorrow, one class per module where that keeps the code clear.

**Next:** [Day 19: OOP I](Software-Day-19-OOP-I.md)

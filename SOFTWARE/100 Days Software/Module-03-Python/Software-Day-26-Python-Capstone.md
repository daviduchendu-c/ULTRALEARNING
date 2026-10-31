# Day 26: Python Capstone

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 6 hours, spread over one or two sittings
**Level:** intermediate. You have finished Days 9 to 25. This is for anyone who can write each piece on its own and now needs to put them into one service that another person can run, test, read and change.

**By the end of this day, you will have:**

- A multi-module command-line expense tracker with a service layer, a storage layer and a presentation layer, each in its own module
- Persistent data in SQLite through the standard library `sqlite3` module, with a schema, parameterized queries and transactions
- A small error hierarchy that separates user mistakes from bugs, and a CLI that reports user mistakes without a traceback
- Logging at the right levels, configured in one place and used by every module
- A test suite with unit tests that use a fake repository, and integration tests that run the real CLI against a real SQLite file
- Documentation a new developer can follow: a README with commands and a design note with the module map
- A Git history with feature branches, `--no-ff` merges and a version tag, built the way a small team would build it

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- Tools installed in the project's virtual environment: `pytest` 8.x, `ruff` 0.6 or newer, `mypy` 1.10 or newer (all free, open source)
- The sqlite3 module, which ships with Python. You do not install SQLite separately. Check it with `python -c "import sqlite3; print(sqlite3.sqlite_version)"`.

**Prerequisites:**

- Day 9 to Day 18: Runtime, types, collections, control flow, functions and modules
- Day 19 and Day 20: Classes, inheritance, composition and dataclasses
- Day 21 to Day 23: Protocols, errors and logging, and the standard library
- Day 24 and Day 25: The project layout, packaging, and testing with pytest

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Module map](#module-map)
  - [Persistence with SQLite](#persistence-with-sqlite)
  - [Service layer and errors](#service-layer-and-errors)
  - [Command-line interface](#command-line-interface)
  - [Logging](#logging)
  - [Tests at two levels](#tests-at-two-levels)
  - [Documentation and the Git workflow](#documentation-and-the-git-workflow)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an expense tracker](#project-an-expense-tracker)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every earlier day built one skill in isolation. This day asks you to combine them into a single tool that someone would actually use: a command that records what you spend, stores it safely between runs, reports totals for a month, and explains itself when something goes wrong. The code stays short, and the discipline shows in the boundaries. Each module does one job, the layers talk through small interfaces, the errors have names, and the tests check the behavior at the level where it matters.

Capstones teach something that exercises do not. A small project forces you to decide where a piece of logic belongs, which parts need tests, and what a new developer needs to read first. Those decisions are what a reviewer looks for in a paid job. This day also includes the Git workflow of a small team: a branch per feature, a merge that keeps the feature visible, and a tag that marks a release. The result is a repository whose history records the work.

The main references are the `sqlite3` documentation at docs.python.org/3/library/sqlite3.html, the `argparse` documentation at docs.python.org/3/library/argparse.html, the `logging` HOWTO at docs.python.org/3/howto/logging.html, and the Pro Git book's chapter on branching at git-scm.com/book/en/v2. Keep them open while you work.

---

## Glossary

- **Capstone:** a project that combines the skills of a whole course or module into one deliverable.
- **Layer:** a group of modules with one responsibility, such as storage, business rules or presentation.
- **Service layer:** the module that applies the rules of the application and calls storage to save or load data.
- **Repository:** an object that hides how data is stored behind methods such as `add` and `list_for_month`.
- **SQLite:** a small database stored in a single file, included with Python through the `sqlite3` module.
- **Schema:** the table definitions of a database.
- **Parameterized query:** a database command in which values are passed separately from the SQL text, using `?` placeholders.
- **Transaction:** a group of database changes that either all commit or all roll back.
- **CLI:** command-line interface, a program controlled by typed commands and options.
- **Subcommand:** a named action inside a CLI, such as `add` or `report`.
- **Exit code:** the number a program returns to the shell. Zero means success, and nonzero means a failure.
- **Feature branch:** a branch that holds the work for one change, merged into the main branch when it is complete.
- **Annotated tag:** a named, dated marker on one commit, used for releases.

---

## Concepts

The seven concepts describe the parts of the capstone in the order a reader meets them, from the layout of the code to the workflow that produced it.

### Module map

A module map is a plan of the rooms in a building before anyone builds the walls. Think of a restaurant. The kitchen prepares food, the dining room serves it, and the cash desk takes payment. Each area has its own staff and rules, and they meet at a small number of doors. A module map gives the same clarity to code: each module has a job, and each dependency points in one direction.

From first principles, the expense tracker has four layers. The `models` module holds the data types and the rule that turns text into a valid amount. The `storage` module knows how to save and load expenses in SQLite, and nothing else. The `service` module applies the application's rules, such as normalizing categories, and calls storage. The `cli` module reads command-line arguments, calls the service, and prints results. The arrows point one way: the CLI depends on the service, the service depends on storage and models, and storage depends on models. No module depends on the CLI, so the business rules can be used by a web front end later without change.

```
expenses/
├── __init__.py    version and package doc
├── __main__.py    python -m expenses entry point
├── errors.py      error types the CLI can report
├── models.py      Expense and parse_amount
├── storage.py     ExpenseRepository, the SQLite layer
├── service.py     ExpenseService, the rules
└── cli.py         argparse commands and exit codes
```

The common mistake is letting the CLI build SQL queries or the storage layer print messages. Each shortcut couples two layers that should be able to change separately. A second mistake is creating a module per concept before it is needed, which produces many small files with no clear boundary. Start with the layers, and split a module only when it has two reasons to change. In the job market, reviewers ask where a rule lives and how many layers a change would touch, and a clear module map answers both.

### Persistence with SQLite

Persistence is the act of keeping data after the program stops. Think of a filing cabinet. The cabinet keeps the folders after the office closes, and a clerk can find a folder by its label. SQLite is a filing cabinet that lives in one file, and it is part of the standard library, so the program needs nothing else installed.

From first principles, `sqlite3.connect(path)` opens or creates the file. `conn.execute(sql, params)` runs one statement, and the `?` placeholders receive the values separately, which prevents SQL injection. A connection used as a context manager commits the transaction when the block ends normally and rolls it back when an exception passes through. This project stores each amount as text, because SQLite's numeric types would hold a float, and money must stay exact. The repository converts text back to `Decimal` when it reads a row. The `CREATE TABLE IF NOT EXISTS` statement makes the schema safe to run every time the program starts.

```python
import sqlite3

conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE t (x TEXT)")
with conn:
    conn.execute("INSERT INTO t VALUES (?)", ("12.50",))
print(conn.execute("SELECT x FROM t").fetchall())   # [('12.50',)]
```

The `with conn:` block is a transaction. If the insert failed, the table would be left as it was before the block.

The common mistake is building SQL by joining strings with user input, which allows the input to change the query. Always use placeholders. The second mistake is storing money as a float in SQLite, which loses cents over many additions. Store the text form of a `Decimal`, and convert on read. In the job market, the boundary between code and data in a query is a standard review item, and the storage of money is a frequent source of incorrect totals.

### Service layer and errors

The service layer is the part of the application that decides what is allowed. Think of a bank teller. The teller checks the form, refuses an overdraft, and records the transaction. The vault stores the money, but it does not decide whether a withdrawal is reasonable. The service is the teller, and the repository is the vault.

From first principles, the service raises domain errors, which are subclasses of `ExpenseError`. The CLI catches that base class and prints the message to stderr with a nonzero exit code. Any other exception, such as a bug or a broken database file, is not caught by the CLI, so it appears with a traceback, which is the right behavior for a developer. This split keeps user mistakes quiet and bugs loud. The service never prints. It returns values and raises errors, which makes it easy to test.

```python
from expenses.errors import InvalidAmountError
from expenses.models import parse_amount

try:
    parse_amount("-5")
except InvalidAmountError as err:
    print("user error:", err)
```

The message is readable by a person, and the exception type is what the CLI uses to decide the exit code.

The common mistake is catching `Exception` in the CLI, which hides bugs as user errors and prints a misleading message. Catch the base class of your own errors only. A second mistake is letting the service read from the terminal with `input`, which makes it impossible to test without patching. Keep input and output in the CLI. In the job market, separating user errors from bugs is one of the most visible signs of a maintainable service.

### Command-line interface

A command-line interface is a set of doors into the application. Think of a hotel reception desk. Guests arrive with requests, the receptionist checks the booking, and passes the request to the right department. The desk does not cook or clean. The CLI parses arguments, calls the service and prints the answer.

From first principles, `argparse` builds a parser with a main command and subcommands. Each subcommand declares its own arguments, and the parser converts types, checks required values and produces help text. `type=date.fromisoformat` turns the text of an option into a date, and an invalid date makes `argparse` print an error and exit with status 2. The `main` function accepts an optional list of arguments, which lets tests call it without starting a new process. It returns an integer, and the entry point passes that integer to `sys.exit`.

```python
import argparse

parser = argparse.ArgumentParser(prog="demo")
commands = parser.add_subparsers(dest="command", required=True)
commands.add_parser("hello").add_argument("name")
print(parser.parse_args(["hello", "Ada"]))   # Namespace(command='hello', name='Ada')
```

The `required=True` on the subparsers means the program fails with a helpful message if no command is given, which is better than silently doing nothing.

The common mistake is calling `sys.exit` inside a function that tests call, which stops the test runner. Return an exit code instead, and let the entry point exit. A second mistake is printing from the service and the CLI both, which produces duplicate output. In the job market, CLI tools are judged by their help text, their exit codes and their error messages, and reviewers check all three.

### Logging

Logging records what the program did, so that someone can understand it later. Think of a flight recorder. It runs quietly during the flight, records every event with a time and a severity, and is read only when someone needs to know what happened. The recorder does not change the flight, and the severity setting decides how much is kept.

From first principles, each module calls `logging.getLogger(__name__)` once, and it logs at the right level: `info` for an action that changed data, `debug` for details that help diagnose a problem, and `warning` or `error` for things that went wrong. The configuration is done once, in the entry point, with `logging.basicConfig`. The CLI sets the level from the `-v` flag. Library modules never call `basicConfig` themselves, because the application owns the configuration. Logging a message with `%s` placeholders, rather than an f-string, keeps the formatting lazy.

```python
import logging

logger = logging.getLogger("expenses.service")
logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")
logger.info("recorded expense %d in %s", 3, "tea")
```

The output is `INFO expenses.service: recorded expense 3 in tea`. The logger name shows which module wrote the line, which is why `__name__` is the standard choice.

The common mistake is logging money amounts or personal notes at INFO, which places private data in the logs of every run. Log identifiers and categories, and keep sensitive details out. A second mistake is configuring logging inside each module, which causes duplicate lines and conflicting levels. In the job market, operators depend on logs to diagnose incidents, and a clear logger name per module is one of the first things they check.

### Tests at two levels

Tests at two levels give you speed and confidence. Think of a car factory. Each component is tested on a bench, which is fast and precise, and the finished car is taken on a test drive, which is slower and checks the whole system. Both are needed, and each answers a different question.

From first principles, the unit tests in this project use a `FakeRepo`, which is an in-memory class with the same method names as the real repository. The service tests check the rules, such as category normalization and the month check, without touching a file. The integration tests call the real `main` function with a temporary SQLite file, and they check the printed output and the exit codes. The integration tests are marked with `pytest.mark.integration`, so the fast run can skip them. Both groups use `tmp_path`, so no test writes to a fixed location.

```python
from expenses.errors import InvalidInputError
from expenses.service import ExpenseService


class FakeRepo:
    def add(self, amount, category, note, spent_on):
        raise AssertionError("not called in this test")


def test_empty_category_is_rejected():
    service = ExpenseService(FakeRepo())
    try:
        service.record("3", "  ", spent_on=None)
    except InvalidInputError:
        pass
```

The fake fails loudly if the service reaches storage when it should not, which is a useful property of a test double. The example is a fragment for illustration, and the project's full version is in the starter code.

The common mistake is using the real repository in every test, which makes the suite slow and hides rule bugs behind storage bugs. The second mistake is using only the fake, which never checks that the SQL works. Keep both levels. In the job market, a suite with a clear split between fast and integration tests is easy to run on every change, and teams notice quickly when the split is missing.

### Documentation and the Git workflow

Documentation and history are how a project explains itself to the next person. Think of a shared kitchen with a recipe book and a log on the wall. The recipe book says how to make each dish. The log shows who changed the recipe and why. Good projects have both, and they are kept up to date as the work changes.

From first principles, the README is the first file a reader sees, so it holds the install commands, the run commands and the test commands, in the order a new developer needs them. The design note describes the module map and the data flow in a short page. The Git history records each change as a commit with a clear message. A feature branch isolates one change, and `git merge --no-ff` creates a merge commit that shows the feature as a unit. An annotated tag such as `v0.1.0` names a release, and it is the reference that a user installs or a changelog points to.

```bash
git switch -c feature/storage
git add src/expenses/storage.py tests/integration/test_cli.py
git commit -m "Add SQLite repository with parameterized queries"
git switch main
git merge --no-ff feature/storage -m "Merge feature/storage"
git tag -a v0.1.0 -m "First release of the expense tracker"
```

The commands show one full cycle. Each step changes the repository in a way that is easy to inspect with `git log --graph`.

The common mistake is committing unrelated changes together, which makes the history hard to read and makes a revert risky. Commit one logical change at a time. A second mistake is documenting the code by copying it into the README, which goes out of date on the next change. Document commands and decisions, and let the code document its details. In the job market, a README that a new developer can follow in five minutes is one of the clearest signs of a professional project.

---

## Hands-on exercises

Work in a folder called `capstone-lab` for the exercises. Use Python 3.14 or 3.12 or newer, and create a virtual environment before each session.

Exercise 1 maps the layers.

1. On paper or in a text file, draw the four layers from the module map: models, storage, service and CLI. Draw one arrow for each import you expect.
2. Open the starter code, and check each arrow by reading the import lines. Note any import that goes the wrong way.
3. Write one sentence on which layer would change if you replaced SQLite with a JSON file.

Done when every import in the starter code follows one direction, and your sentence names only the storage layer.

Exercise 2 verifies persistence.

1. Create a repository in a temporary file, add one expense, and close the program.
2. Start a new Python process that opens the same file and lists the expenses for the month.
3. Record the amount as stored in the database with `sqlite3`, and confirm it is the text `"12.50"`, not a float.

Done when the second process sees the expense, and the stored amount is text.

Exercise 3 checks the error boundary.

1. Run the CLI with an invalid amount, such as `-5`, and record the message and the exit code.
2. Run it with a missing subcommand and record the message and the exit code.
3. Write one sentence on why the first case prints no traceback and the second prints argparse help.

Done when you can explain why user mistakes return 1 and argparse errors return 2.

Exercise 4 reads the logs.

1. Run `python -m expenses -v add 2 tea` and record the log line and the printed line.
2. Run the same command without `-v` and record the difference.
3. Write one sentence on which module wrote the log line, and how you can tell from its name.

Done when the verbose run shows the INFO line from the service module.

Exercise 5 runs the test levels.

1. Run `python -m pytest -m "not integration"` and record the number of tests selected.
2. Run `python -m pytest -m integration` and record the number selected.
3. Break one rule on purpose, such as the category normalization, and record which level of test fails first.

Done when you can state which level caught the broken rule, and why.

---

## Project: an expense tracker

The project is a command-line expense tracker with SQLite storage. It records an expense with an amount, a category, an optional note and a date, reports totals by category for a month, and removes an expense by its id. It has a service layer that applies the rules, a storage layer that uses parameterized SQL and transactions, a CLI with clear exit codes, logging controlled by a flag, two levels of tests, a README and a design note. The work is organized as feature branches merged with `--no-ff`, and it ends with a tagged release.

Suggested repo name: `100-days-software-day-26-python-capstone`.

Repo layout:

```
100-days-software-day-26-python-capstone/
├── .gitignore
├── README.md
├── pyproject.toml
├── docs/
│   ├── design.md
│   └── findings.md
├── src/
│   └── expenses/
│       ├── __init__.py
│       ├── __main__.py
│       ├── cli.py
│       ├── errors.py
│       ├── models.py
│       ├── service.py
│       └── storage.py
└── tests/
    ├── __init__.py
    ├── unit/
    │   ├── __init__.py
    │   ├── test_models.py
    │   └── test_service.py
    └── integration/
        ├── __init__.py
        └── test_cli.py
```

Starter code for `pyproject.toml`:

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "expenses"
version = "0.1.0"
description = "A command-line expense tracker with SQLite storage."
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.6", "mypy>=1.10"]

[project.scripts]
expenses = "expenses.cli:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers -ra"
markers = ["integration: tests that run the CLI against a real SQLite file"]

[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.mypy]
strict = true
mypy_path = "src"
files = ["src"]
```

Starter code for `src/expenses/__init__.py`:

```python
"""Expense tracker: models, SQLite storage, service rules and a CLI."""

__version__ = "0.1.0"
```

Starter code for `src/expenses/errors.py`:

```python
class ExpenseError(Exception):
    """Base class for errors that the CLI reports to the user."""


class InvalidAmountError(ExpenseError):
    """Raised when an amount is not a positive money value."""


class InvalidInputError(ExpenseError):
    """Raised when a category, month or other input is not usable."""


class NotFoundError(ExpenseError):
    """Raised when an expense id does not exist."""
```

Starter code for `src/expenses/models.py`:

```python
from dataclasses import dataclass
from datetime import date
from decimal import ROUND_HALF_UP, Decimal, InvalidOperation

from expenses.errors import InvalidAmountError

CENT = Decimal("0.01")


def parse_amount(text: str) -> Decimal:
    try:
        value = Decimal(text)
    except InvalidOperation as exc:
        raise InvalidAmountError(f"not an amount: {text!r}") from exc
    if not value.is_finite() or value <= 0:
        raise InvalidAmountError(f"amount must be a positive number: {text!r}")
    return value.quantize(CENT, rounding=ROUND_HALF_UP)


@dataclass(frozen=True)
class Expense:
    id: int
    amount: Decimal
    category: str
    note: str
    spent_on: date
```

Starter code for `src/expenses/storage.py`:

```python
import sqlite3
from collections.abc import Iterator
from contextlib import contextmanager
from datetime import date
from decimal import Decimal
from pathlib import Path

from expenses.errors import NotFoundError
from expenses.models import Expense

SCHEMA = """
CREATE TABLE IF NOT EXISTS expenses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    amount TEXT NOT NULL,
    category TEXT NOT NULL,
    note TEXT NOT NULL,
    spent_on TEXT NOT NULL
)
"""

Row = tuple[int, str, str, str, str]


class ExpenseRepository:
    def __init__(self, path: str | Path) -> None:
        self.path = str(path)
        with self._connect() as conn:
            conn.execute(SCHEMA)

    @contextmanager
    def _connect(self) -> Iterator[sqlite3.Connection]:
        conn = sqlite3.connect(self.path)
        try:
            with conn:
                yield conn
        finally:
            conn.close()

    def add(self, amount: Decimal, category: str, note: str, spent_on: date) -> Expense:
        with self._connect() as conn:
            cursor = conn.execute(
                "INSERT INTO expenses (amount, category, note, spent_on) VALUES (?, ?, ?, ?)",
                (str(amount), category, note, spent_on.isoformat()),
            )
            new_id = cursor.lastrowid
        if new_id is None:
            raise RuntimeError("insert did not return an id")
        return Expense(new_id, amount, category, note, spent_on)

    def list_for_month(self, year: int, month: int) -> list[Expense]:
        with self._connect() as conn:
            rows: list[Row] = conn.execute(
                "SELECT id, amount, category, note, spent_on FROM expenses "
                "WHERE substr(spent_on, 1, 7) = ? ORDER BY spent_on, id",
                (f"{year:04d}-{month:02d}",),
            ).fetchall()
        return [_row_to_expense(row) for row in rows]

    def delete(self, expense_id: int) -> None:
        with self._connect() as conn:
            cursor = conn.execute("DELETE FROM expenses WHERE id = ?", (expense_id,))
            removed = cursor.rowcount
        if removed == 0:
            raise NotFoundError(f"no expense with id {expense_id}")


def _row_to_expense(row: Row) -> Expense:
    expense_id, amount, category, note, spent_on = row
    return Expense(expense_id, Decimal(amount), category, note, date.fromisoformat(spent_on))
```

Starter code for `src/expenses/service.py`:

```python
import logging
from datetime import date
from decimal import Decimal

from expenses.errors import InvalidInputError
from expenses.models import Expense, parse_amount
from expenses.storage import ExpenseRepository

logger = logging.getLogger(__name__)


class ExpenseService:
    def __init__(self, repo: ExpenseRepository) -> None:
        self.repo = repo

    def record(
        self,
        amount_text: str,
        category: str,
        note: str = "",
        spent_on: date | None = None,
    ) -> Expense:
        amount = parse_amount(amount_text)
        cleaned = category.strip().lower()
        if not cleaned:
            raise InvalidInputError("category must not be empty")
        expense = self.repo.add(amount, cleaned, note.strip(), spent_on or date.today())
        logger.info("recorded expense %d in %s", expense.id, cleaned)
        return expense

    def monthly_totals(self, year: int, month: int) -> dict[str, Decimal]:
        if not 1 <= month <= 12:
            raise InvalidInputError(f"month out of range: {month}")
        totals: dict[str, Decimal] = {}
        for expense in self.repo.list_for_month(year, month):
            current = totals.get(expense.category, Decimal("0"))
            totals[expense.category] = current + expense.amount
        logger.debug("computed %d categories for %04d-%02d", len(totals), year, month)
        return dict(sorted(totals.items()))

    def remove(self, expense_id: int) -> None:
        self.repo.delete(expense_id)
        logger.info("removed expense %d", expense_id)
```

Starter code for `src/expenses/cli.py`:

```python
import argparse
import logging
import sys
from datetime import date
from decimal import Decimal

from expenses.errors import ExpenseError
from expenses.service import ExpenseService
from expenses.storage import ExpenseRepository

DEFAULT_DB = "expenses.db"


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="expenses", description="Track monthly expenses.")
    parser.add_argument("--db", default=DEFAULT_DB, help="path to the SQLite file")
    parser.add_argument("-v", "--verbose", action="store_true", help="log actions")
    commands = parser.add_subparsers(dest="command", required=True)

    add = commands.add_parser("add", help="record an expense")
    add.add_argument("amount")
    add.add_argument("category")
    add.add_argument("--note", default="")
    add.add_argument("--date", dest="spent_on", type=date.fromisoformat, default=None)

    report = commands.add_parser("report", help="totals by category for a month")
    report.add_argument("year", type=int)
    report.add_argument("month", type=int)

    remove = commands.add_parser("remove", help="delete an expense by id")
    remove.add_argument("expense_id", type=int)
    return parser


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    logging.basicConfig(
        level=logging.INFO if args.verbose else logging.WARNING,
        format="%(levelname)s %(name)s: %(message)s",
    )
    service = ExpenseService(ExpenseRepository(args.db))
    try:
        if args.command == "add":
            expense = service.record(args.amount, args.category, args.note, args.spent_on)
            print(f"added {expense.id}: {expense.amount} {expense.category}")
        elif args.command == "report":
            totals = service.monthly_totals(args.year, args.month)
            for category, total in totals.items():
                print(f"{category}: {total}")
            print(f"total: {sum(totals.values(), Decimal('0'))}")
        elif args.command == "remove":
            service.remove(args.expense_id)
            print(f"removed {args.expense_id}")
    except ExpenseError as err:
        print(f"error: {err}", file=sys.stderr)
        return 1
    return 0
```

Starter code for `src/expenses/__main__.py`:

```python
import sys

from expenses.cli import main

if __name__ == "__main__":
    sys.exit(main())
```

Starter code for `tests/unit/test_models.py`:

```python
from decimal import Decimal

import pytest

from expenses.errors import InvalidAmountError
from expenses.models import parse_amount


def test_parse_amount_rounds_half_up():
    assert parse_amount("12.345") == Decimal("12.35")


@pytest.mark.parametrize("text", ["abc", "-1", "0", "NaN", ""])
def test_parse_amount_rejects_bad_values(text):
    with pytest.raises(InvalidAmountError):
        parse_amount(text)
```

Starter code for `tests/unit/test_service.py`:

```python
import logging
from datetime import date
from decimal import Decimal

import pytest

from expenses.errors import InvalidInputError, NotFoundError
from expenses.models import Expense
from expenses.service import ExpenseService


class FakeRepo:
    """An in-memory stand-in for ExpenseRepository, used by the unit tests."""

    def __init__(self):
        self.rows = []
        self.next_id = 1

    def add(self, amount, category, note, spent_on):
        expense = Expense(self.next_id, amount, category, note, spent_on)
        self.next_id += 1
        self.rows.append(expense)
        return expense

    def list_for_month(self, year, month):
        return [e for e in self.rows if (e.spent_on.year, e.spent_on.month) == (year, month)]

    def delete(self, expense_id):
        before = len(self.rows)
        self.rows = [e for e in self.rows if e.id != expense_id]
        if len(self.rows) == before:
            raise NotFoundError(f"no expense with id {expense_id}")


def test_record_normalizes_category_and_amount():
    service = ExpenseService(FakeRepo())
    expense = service.record("3", "  Food ", spent_on=date(2026, 1, 5))
    assert expense.category == "food"
    assert expense.amount == Decimal("3.00")


def test_record_rejects_empty_category():
    service = ExpenseService(FakeRepo())
    with pytest.raises(InvalidInputError):
        service.record("3", "   ", spent_on=date(2026, 1, 5))


def test_monthly_totals_sum_per_category():
    service = ExpenseService(FakeRepo())
    service.record("3.50", "food", spent_on=date(2026, 1, 5))
    service.record("1.50", "food", spent_on=date(2026, 1, 9))
    service.record("10", "travel", spent_on=date(2026, 2, 1))
    assert service.monthly_totals(2026, 1) == {"food": Decimal("5.00")}


def test_monthly_totals_rejects_bad_month():
    service = ExpenseService(FakeRepo())
    with pytest.raises(InvalidInputError):
        service.monthly_totals(2026, 13)


def test_remove_unknown_id_raises():
    service = ExpenseService(FakeRepo())
    with pytest.raises(NotFoundError):
        service.remove(42)


def test_record_logs_at_info(caplog):
    service = ExpenseService(FakeRepo())
    with caplog.at_level(logging.INFO, logger="expenses.service"):
        service.record("2", "tea", spent_on=date(2026, 1, 5))
    assert "recorded expense 1 in tea" in caplog.text
```

Starter code for `tests/integration/test_cli.py`:

```python
import pytest

from expenses.cli import main

pytestmark = pytest.mark.integration


def test_add_then_report(tmp_path, capsys):
    db = str(tmp_path / "expenses.db")
    assert main(["--db", db, "add", "12.50", "Food", "--date", "2026-01-05"]) == 0
    assert main(["--db", db, "add", "3", "food", "--date", "2026-01-06"]) == 0
    capsys.readouterr()

    assert main(["--db", db, "report", "2026", "1"]) == 0
    out = capsys.readouterr().out
    assert "food: 15.50" in out
    assert "total: 15.50" in out


def test_remove_missing_id_reports_error(tmp_path, capsys):
    db = str(tmp_path / "expenses.db")
    assert main(["--db", db, "remove", "99"]) == 1
    assert "no expense with id 99" in capsys.readouterr().err


def test_invalid_amount_is_a_user_error(tmp_path, capsys):
    db = str(tmp_path / "expenses.db")
    assert main(["--db", db, "add", "-5", "food"]) == 1
    assert "positive" in capsys.readouterr().err


def test_data_survives_a_new_run(tmp_path, capsys):
    db = str(tmp_path / "expenses.db")
    main(["--db", db, "add", "4", "books", "--date", "2026-03-02"])
    capsys.readouterr()
    main(["--db", db, "report", "2026", "3"])
    assert "books: 4.00" in capsys.readouterr().out
```

The empty `__init__.py` files in `tests/`, `tests/unit/` and `tests/integration/` let pytest import each folder as a package.

Starter code for `README.md`:

```markdown
# Expense tracker

A command-line tool that records expenses in a local SQLite file.

## Setup

    py -3.14 -m venv .venv
    source .venv/Scripts/activate        # Git Bash; use .venv\Scripts\Activate.ps1 in PowerShell
    python -m pip install -e ".[dev]"

## Use

    expenses add 12.50 food --date 2026-01-05 --note lunch
    expenses report 2026 1
    expenses remove 3
    expenses -v add 2 tea                # -v logs each action

Use `--db PATH` to choose the database file. The default is `expenses.db`.

## Check

    python -m pytest -m "not integration"   # fast unit tests
    python -m pytest                        # all tests
    python -m ruff check src tests
    python -m mypy
```

Step-by-step tasks:

1. Create the repo folder, run `git init -b main`, and add `.gitignore` with `.venv/`, `dist/`, `build/`, `*.egg-info/`, `.pytest_cache/`, `__pycache__/` and `*.db`. Create the README and `pyproject.toml` first, and commit them as "Add project skeleton" on `main`.
2. Create the environment, activate it, and install with `python -m pip install -e ".[dev]"`. Record the `sys.prefix` line and the install's last line in `docs/findings.md`.
3. Create the source and test files, one feature at a time, on branches. For the storage layer, run `git switch -c feature/storage`, add `errors.py`, `models.py`, `storage.py`, and `tests/unit/test_models.py`. Commit with a message that says what the layer does, then `git switch main` and `git merge --no-ff feature/storage -m "Merge feature/storage"`. Run `git log --oneline --graph` and paste it into `docs/findings.md`.
4. Repeat step 3 for `feature/service` (`service.py` and `tests/unit/test_service.py`), and for `feature/cli` (`cli.py`, `__main__.py` and `tests/integration/test_cli.py`). Each branch must have at least one commit, and each merge must show a merge commit in the graph.
5. Run `python -m ruff format src tests`. Then run `python -m ruff check src tests`. The starter code has five lines longer than 88 characters, so wrap those calls over several lines, and record in `docs/findings.md` which lines you wrapped. Run `python -m ruff check src tests` again until it reports no issues.
6. Run `python -m mypy` and fix any issue. Record the success line.
7. Run `python -m pytest -m "not integration"` and `python -m pytest`. Record both counts.
8. Use the CLI end to end: add three expenses in two categories and one in a different month, then run `report` for each month, then remove one id. Paste the commands and outputs into `docs/findings.md`.
9. Run `python -m expenses -v add 2 tea` and record the log line in `docs/findings.md`. Name the module that wrote it.
10. Write `docs/design.md` with a module map, a data-flow sentence for one `add` command, and one paragraph on where the boundaries are and why the CLI does not build SQL.
11. Create a feature branch `feature/list-month`, add a `list` subcommand that prints each expense for a month with its id, amount, category and date, add a test, and merge it with `--no-ff`. Record the new command output in `docs/findings.md`.
12. Create an annotated tag with `git tag -a v0.1.0 -m "First release of the expense tracker"`. Push the branch, `main` and the tag to GitHub with `git push -u origin main --follow-tags`, after adding the remote.

Done when:

- `git log --oneline --graph --all` shows at least three feature branches merged with `--no-ff`, the `list-month` feature, and the `v0.1.0` tag
- `python -m mypy` reports no issues and `python -m ruff check src tests` reports no issues
- `python -m pytest` passes, and `python -m pytest -m "not integration"` runs only the unit tests
- `docs/findings.md` contains the graph, the end-to-end CLI transcript, the log line, and the counts
- `docs/design.md` names each module and the direction of the dependencies
- The GitHub repo shows the `main` branch, the tag and the merged history

Evidence to commit: all source, test and documentation files, `README.md`, `docs/design.md`, `docs/findings.md`, `pyproject.toml` and `.gitignore`. Do not commit `.venv`, the `expenses.db` file or the build folders.

The capstone is complete when a stranger can clone the repo, follow the README, and reproduce every command in `docs/findings.md`. Test that by cloning into a new folder once, and recording the result.

---

## Check your understanding

1. The CLI imports `sqlite3` and builds SQL directly. Name the layer boundary that this breaks, and explain what changes if you later switch to a JSON file.
2. A user enters `food'; DROP TABLE expenses; --` as a category. Explain why the parameterized query protects the table, and what the query would have done if it used an f-string.
3. The CLI catches `Exception` around the whole `main` body. Explain what happens to a bug such as a `KeyError` inside the service, and rewrite the handler to keep bugs loud.
4. A developer stores each amount as a SQLite `REAL`. Explain how totals can drift, and state the storage format the project uses instead.
5. A teammate adds a log line at INFO that prints the full note text for every expense. Name the privacy risk and the level and content that would fix it.
6. The history shows one large commit called "Build everything". Explain what a reviewer loses, and name the Git practice that would have kept the history readable.

### Answers

1. The CLI now depends on the storage details, so the presentation layer and persistence are coupled. Switching to a JSON file would require changing the CLI as well as the storage, instead of only the repository, which is the boundary the module map protects.
2. The placeholder sends the category to SQLite as a value, so the string is stored as text and never parsed as SQL. An f-string would place the text into the SQL statement itself, so the `DROP TABLE` would run as part of the query.
3. A `KeyError` in the service would be caught and printed as a generic error with exit code 1, hiding the bug. Catch only `ExpenseError` in the CLI, so unexpected exceptions propagate with a traceback and a nonzero exit.
4. `REAL` stores binary floating-point values, so additions accumulate small errors, and a total can differ from the sum of its parts by a cent. The project stores the text of a `Decimal`, which is exact, and converts it back on read.
5. Notes can contain private information, and the INFO level is on in many runs, so the text ends up in the logs. The fix is to log the identifier and category at INFO, and to move any text detail to DEBUG, or to leave it out entirely.
6. A reviewer cannot see when each feature was added, how the layers were built, or which change introduced a bug, and a revert would undo everything. The practice is one feature branch per change, with small commits and a `--no-ff` merge, so the graph shows each feature as a unit.

---

## Sources to read, watch and practice

*Read*

- Python documentation, module `sqlite3`, at docs.python.org/3/library/sqlite3.html
- Python documentation, module `argparse`, at docs.python.org/3/library/argparse.html
- Python Logging HOWTO, at docs.python.org/3/howto/logging.html
- Pro Git book, chapter "Git Branching", at git-scm.com/book/en/v2
- PEP 8, "Style Guide for Python Code", at peps.python.org/pep-0008/
- [Wikipedia: SQLite](https://en.wikipedia.org/wiki/SQLite)
- [Wikipedia: Command-line interface](https://en.wikipedia.org/wiki/Command-line_interface)
- [Wikipedia: Layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture)

*Tutorials*

- GeeksforGeeks, "Python SQLite" and "Python argparse" tutorials, at geeksforgeeks.org
- W3Schools, "Python MySQL" page for comparison with SQL databases, and the SQL tutorial at w3schools.com/sql/, which covers the SELECT and INSERT statements used here

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel, for a review of the earlier days.
- CS50's Introduction to Databases with SQL, on the CS50 YouTube channel, which covers the SQL behind this project.

*Practice*

- Learn Git Branching at learngitbranching.js.org, for merges and tags, as a visual check of the workflow used today.
- SQLBolt at sqlbolt.com, for interactive SQL lessons that match the queries in this project.

---

## Tomorrow

Module 3 is complete. Day 27 begins Module 4, the C++ track, with its toolchain: the compiler, the linker, translation units, header files, the build process and compiler warnings. The language is different, and the idea of a build is the same one you have just practiced: a set of inputs that must be turned into a program the same way every time.

**Next:** Day 27 is "C++ Toolchain" in the C++ module of the track index.

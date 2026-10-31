# Day 25: Testing Python

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You have packaged and checked a project on Day 24, and you have used `unittest` for small checks since Day 11. This is for anyone who has changed one function and broken three others, or who has written a test that passes today and fails tomorrow for no visible reason.

**By the end of this day, you will have:**

- A clear model of what a unit test checks, and how to keep one test to one behavior
- The ability to write parametrized pytest tests with fixtures that build shared state for each test
- A working understanding of test doubles, including mocks that record calls and patches that replace an object for the length of a test
- The ability to separate integration tests that touch files and other systems from fast unit tests, and to run each group on its own
- A repeatable test layout with pytest markers and configuration in `pyproject.toml`
- A todo-store repo with unit and integration tests, run by one command and checked by mypy and Ruff

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- Tools installed inside the project's virtual environment: `pytest` 8.x (free, open source, MIT licensed), `ruff` 0.6 or newer, and `mypy` 1.10 or newer
- The packaging workflow from Day 24: a `pyproject.toml`, a `src` layout, and `pip install -e ".[dev]"`

**Prerequisites:**

- Day 13 and Day 14: Lists, tuples, dictionaries and sets (the data the tests assert on)
- Day 19 and Day 20: OOP I and II (classes and their methods, which the tests exercise)
- Day 22: Exceptions and Debugging (`pytest.raises` relies on exceptions)
- Day 24: Professional Python (the project layout and the commands)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Unit tests](#unit-tests)
  - [Fixtures concepts](#fixtures-concepts)
  - [Mocks concepts](#mocks-concepts)
  - [Integration tests](#integration-tests)
  - [Test organization](#test-organization)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a todo store with unit and integration tests](#project-a-todo-store-with-unit-and-integration-tests)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A test is a second copy of your intention, written in code. When the code and the test agree, you have evidence that the behavior is what you meant. When they disagree, you have found a problem while it is still cheap to fix. Tests do not prove that a program is correct, but they make changes safe, they document behavior that the code alone does not explain, and they turn a bug report into a failing check that stays in the repository forever.

This day teaches the testing practices that working teams use most. Unit tests check one behavior in isolation, so a failure points to one place. Fixtures build the state each test needs, and parametrized tests run one check over many inputs. Test doubles replace parts that are slow, unreliable or outside your control, and integration tests check that the parts work together, including the files they read and write. The project is a small todo store with persistence, tested at both levels, and run by commands you can repeat on any machine.

The official references are the pytest documentation at docs.pytest.org, which is the most widely used test runner for Python, and the `unittest.mock` documentation at docs.python.org/3/library/unittest.mock.html. The pytest documentation's "How to use fixtures" and "How to parametrize fixtures and test functions" pages are the two best places to read after this day.

---

## Glossary

- **Test:** a function that runs code and checks its result against an expected value.
- **Unit test:** a test that checks one behavior of one unit of code, such as a function or a method, with its collaborators replaced when needed.
- **Integration test:** a test that checks several parts together, such as code that writes to a real file and reads it back.
- **Assertion:** a statement that a condition must hold, such as `assert total == 10`.
- **Fixture:** a function that prepares state for tests and provides it as an argument, in pytest.
- **Parametrize:** running one test function with several sets of inputs and expected outputs.
- **Test double:** any object that stands in for a real collaborator in a test.
- **Mock:** a test double that records how it was called, so the test can check the calls.
- **Stub:** a test double that returns fixed answers when called.
- **Patch:** a temporary replacement of an object or attribute, which is restored after the test.
- **Marker:** a label on a test, such as `integration`, that lets a runner select or skip it.
- **Flaky test:** a test that passes and fails on different runs without a code change, usually because it depends on timing, order or shared state.
- **Test discovery:** the process by which the test runner finds test files and functions by naming rules.

---

## Concepts

The five concepts follow the test pyramid from the bottom up: small unit tests, the fixtures and doubles that make them practical, the integration tests that check the seams, and the organization that keeps the whole suite manageable.

### Unit tests

A unit test is a small experiment. Think of a chemist who tests one substance at a time, under controlled conditions, so that a change in the result has one likely cause. A unit test calls one function with chosen inputs, and it checks one result. If it fails, the cause is almost always in that function.

From first principles, a good unit test has three parts: arrange the inputs, act by calling the code, and assert the outcome. The test name states the behavior, such as `test_add_rejects_blank_title`, so the failure message reads as a sentence. Each test should check one behavior, because a test that checks five things hides which one failed. Unit tests should run fast and in any order, so they do not touch the network or depend on the results of other tests. Pytest discovers them by name: files that start with `test_` and functions that start with `test_`.

```python
from todo_app.tasks import TodoStore


def test_add_strips_whitespace(tmp_path):
    store = TodoStore(tmp_path / "todos.json")
    task = store.add("  tea  ")
    assert task.title == "tea"
```

The three parts are visible: the store is arranged, `add` is called, and the title is asserted. The test uses a temporary path so that it does not write to a real file.

The common mistake is writing a test that asserts only that the code did not crash. The test passes for any output, so it proves little. Assert the value that the behavior promises. A second mistake is testing private implementation details, such as the order of internal calls, which breaks when the implementation is improved while the behavior stays correct. Test through the public interface. In the job market, reviewers ask what behavior a test protects, and a test whose name says nothing about behavior gets sent back.

### Fixtures concepts

A fixture is a set-up station. Think of a kitchen where each cook gets a clean bench, the same set of tools and the same ingredients before each dish. The bench is not shared between cooks, so one cook's mess does not spoil the next dish. A pytest fixture does the same for tests: it builds the state, hands it to each test that asks for it, and cleans it up afterward.

From first principles, a function decorated with `@pytest.fixture` is registered by name. A test that takes an argument with that name receives the value the fixture returns. Fixtures can depend on other fixtures, which builds a graph of set-up steps. The default scope is `function`, which means a fresh value for each test. Other scopes, such as `module` and `session`, share one value across tests, which is faster but risks shared state. The built-in `tmp_path` fixture gives each test a unique temporary folder, and `monkeypatch` replaces attributes for the duration of one test.

```python
import pytest


@pytest.fixture
def numbers():
    return [1, 2, 3]


def test_sum(numbers):
    assert sum(numbers) == 6


def test_length(numbers):
    assert len(numbers) == 3
```

Each test receives its own list, so a test that changes `numbers` does not affect the other.

The common mistake is using a session-scoped fixture that holds mutable data, so one test changes it and another test fails later, only when they run in a particular order. Start with function scope, and widen it only for expensive, read-only resources. A second mistake is writing a fixture that does the work of the test itself. A fixture should prepare state, not check it. In the job market, shared fixtures in a large suite are a common source of order-dependent failures, and reviewers ask about the scope of each one.

### Mocks concepts

A mock is a stand-in that records how it was used. Think of a stunt double in a film. The double takes the dangerous shot, and the director checks what the double did. In testing, a mock stands in for a dependency such as a network client, an email sender or a clock, and the test checks that the code called it the right way.

From first principles, `unittest.mock.Mock` accepts any call and returns another mock by default. It records each call, and methods such as `assert_called_once_with` check the recorded arguments. `patch` replaces an object for the duration of a `with` block or a test, then restores the original, even if the test fails. A stub is a simpler double that returns fixed values, and a fake is a working but simplified version, such as an in-memory store. Mocks are powerful, and they can hide mistakes: a mock accepts a method name that the real object does not have, so tests with mocks can pass while the real code fails. Check the interface of a mock against the real class when you can.

```python
from unittest.mock import Mock


def test_notify_called_once():
    notify = Mock()
    notify("done: ship")
    notify.assert_called_once_with("done: ship")
```

The test checks the call, not the internals of `notify`. In the project, the notifier is passed into the store, so the test can observe what the store asks the notifier to do.

The common mistake is mocking everything, which turns a test into a check of the mock setup rather than the behavior. Mock only the dependencies that are slow, external or unreliable. The second mistake is using `patch` on a name where the code uses it, rather than where it is defined. If a module imports a function with `from x import f`, patch `module.f`, not `x.f`. In the job market, teams mock network and clock dependencies, and they review mocks carefully because a wrong mock can hide a real failure.

### Integration tests

An integration test checks that parts work together. Think of a test drive of a car. Each part passed its bench tests, but the test drive checks that the engine, gearbox and wheels work as a system. Integration tests catch problems at the seams, such as a file format that the writer and the reader disagree about.

From first principles, an integration test uses real collaborators where they matter, such as the real file system, a real database or a real HTTP call to a local test server. These tests are slower and can fail for environmental reasons, such as a full disk or a missing permission, so they are kept separate from unit tests. A common pattern is to mark them with a pytest marker, such as `integration`, so the fast suite can run on every save and the full suite runs in continuous integration. Integration tests should still clean up after themselves, using `tmp_path` and not a fixed folder.

```python
from todo_app.tasks import TodoStore


def test_save_and_reload(tmp_path):
    path = tmp_path / "todos.json"
    store = TodoStore(path)
    store.add("one")
    store.save()

    reloaded = TodoStore(path)
    assert [t.title for t in reloaded.tasks] == ["one"]
```

This test uses the real file. A unit test would replace the file with a fake, but this one checks that the saved format can be read back, which is the seam that matters.

The common mistake is writing integration tests that depend on a shared database or on a file in the home folder, so they pass on one machine and fail on another. Create what the test needs and remove it afterward. A second mistake is making every test an integration test, which makes the suite slow enough that people stop running it. In the job market, the balance between fast unit tests and a smaller set of integration tests is one of the first things a reviewer checks when a suite gets slow.

### Test organization

Test organization is the map of the test suite. Think of a library catalog. Books are grouped by subject, each shelf has a label, and a patron can find a book without wandering the whole building. A test suite with the same structure lets a developer find the tests for one module and run only the slow ones when they need to.

From first principles, pytest finds tests under the folders named in `testpaths`, typically `tests`. A common layout puts unit tests in `tests/unit` and integration tests in `tests/integration`, with a `conftest.py` at the top that holds fixtures shared by both. Markers declared in `pyproject.toml` let you select groups from the command line: `pytest -m "not integration"` skips the slow group, and `pytest -m integration` runs only that group. The `--strict-markers` option makes pytest fail when a test uses a marker that is not declared, which catches typing mistakes.

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers -ra"
markers = ["integration: tests that read or write files"]
```

The `-ra` option prints a short summary of skipped and failed tests at the end of each run, which makes the output easier to read.

The common mistake is letting test files drift into the source folder, or naming tests in a way the runner does not find, so they never run. Run `pytest --collect-only` once to see what it finds. The second mistake is keeping tests that depend on a particular order in the same file as independent tests, which makes failures appear randomly. In the job market, a suite that is organized and quick to run is one of the most valued parts of a codebase, because it lets every change get checked.

---

## Hands-on exercises

Work in a folder called `testing-lab`, using Python 3.14 or 3.12 or newer. Create a virtual environment, activate it, and install pytest with `python -m pip install "pytest>=8"`.

Exercise 1 writes a parametrized unit test.

1. Create `text_utils.py` with a function `shout(text: str) -> str` that returns the text in uppercase with an exclamation mark.
2. Create `tests/test_text_utils.py` with one test that uses `@pytest.mark.parametrize` over three input and output pairs, including an empty string.
3. Run `python -m pytest -q` and record the number of passing cases.

Done when pytest reports every parametrized case as passed, and the count matches the number of pairs.

Exercise 2 uses a fixture.

1. Create a fixture `shopping` in `tests/conftest.py` that returns a new list `["milk", "eggs"]`.
2. Write two tests that each append an item to `shopping` and assert the length.
3. Run the tests in both orders, using `python -m pytest tests/test_x.py::test_a tests/test_x.py::test_b` and then the reverse, and record whether both pass.

Done when both tests pass in either order, because each receives its own list.

Exercise 3 uses a mock and a patch.

1. Create `weather.py` with a function `fetch_temperature(client) -> float` that calls `client.get_temp()` and returns the value.
2. Write a test that passes a `Mock` whose `get_temp` returns `21.5`, asserts the result, and asserts that `get_temp` was called once.
3. Write a second test that uses `unittest.mock.patch` to replace a function `read_sensor` in a module, and assert the replacement is used. Explain in one sentence why you patched the name where the code uses it.

Done when both tests pass and you can explain the difference between a stub and a mock in one sentence each.

Exercise 4 writes an integration test with a marker.

1. Add a marker `integration` to `pyproject.toml` under `[tool.pytest.ini_options]`, as in the concepts section.
2. Create `tests/integration/test_files.py` with a test that writes a text file in `tmp_path`, reads it back and asserts equality. Mark the module with `pytestmark = pytest.mark.integration`.
3. Run `python -m pytest -m "not integration"` and then `python -m pytest -m integration`. Record how many tests each command selected.

Done when the first command deselects the integration test and the second runs it.

Exercise 5 checks test discovery.

1. Create a file `tests/check_me.py` that contains a test function, and run `python -m pytest --collect-only -q`. Record whether the function is collected.
2. Rename the file to `tests/test_check_me.py` and run the same command. Record the difference.
3. Write one sentence on why the naming rule matters in a team that adds tests every week.

Done when you can explain why the renamed file is collected and the first one is not.

---

## Project: a todo store with unit and integration tests

The project is a small todo store with JSON persistence, built as a package with a `src` layout. The store assigns ids, validates titles, marks tasks done, notifies a callback, and saves and loads its state. The unit tests check each behavior with fixtures and a mock notifier. The integration tests check the file round trip, a corrupt file and a failed save. The commands are the evidence: the fast suite runs without the integration marker, the full suite runs everything, and mypy and Ruff check the source.

Suggested repo name: `100-days-software-day-25-testing-python`.

Repo layout:

```
100-days-software-day-25-testing-python/
├── .gitignore
├── README.md
├── pyproject.toml
├── src/
│   └── todo_app/
│       ├── __init__.py
│       └── tasks.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── unit/
│   │   ├── __init__.py
│   │   └── test_tasks_unit.py
│   └── integration/
│       ├── __init__.py
│       └── test_tasks_integration.py
└── docs/
    └── findings.md
```

Starter code for `pyproject.toml`:

```toml
[build-system]
requires = ["setuptools>=69"]
build-backend = "setuptools.build_meta"

[project]
name = "todo-app"
version = "0.1.0"
description = "A small todo store with persistence, used to practice testing."
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8", "ruff>=0.6", "mypy>=1.10"]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers -ra"
markers = ["integration: tests that read or write files"]

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

Starter code for `src/todo_app/__init__.py`:

```python
"""A small todo store with JSON persistence."""

__version__ = "0.1.0"
```

Starter code for `src/todo_app/tasks.py`:

```python
import json
from collections.abc import Callable
from dataclasses import asdict, dataclass
from datetime import UTC, datetime
from pathlib import Path


@dataclass
class Task:
    id: int
    title: str
    done: bool = False
    created: str = ""


def _utc_now() -> datetime:
    return datetime.now(UTC)


class TodoStore:
    def __init__(
        self,
        path: Path,
        clock: Callable[[], datetime] | None = None,
        notify: Callable[[str], None] | None = None,
    ) -> None:
        self.path = path
        self.clock = clock or _utc_now
        self.notify = notify
        self.tasks: list[Task] = []
        self._next_id = 1
        if path.exists():
            self.load()

    def add(self, title: str) -> Task:
        cleaned = title.strip()
        if not cleaned:
            raise ValueError("title must not be empty")
        task = Task(self._next_id, cleaned, created=self.clock().isoformat())
        self._next_id += 1
        self.tasks.append(task)
        return task

    def complete(self, task_id: int) -> None:
        for task in self.tasks:
            if task.id == task_id:
                task.done = True
                if self.notify is not None:
                    self.notify(f"done: {task.title}")
                return
        raise KeyError(task_id)

    def open_tasks(self) -> list[Task]:
        return [task for task in self.tasks if not task.done]

    def save(self) -> None:
        data = {"next_id": self._next_id, "tasks": [asdict(t) for t in self.tasks]}
        self.path.write_text(json.dumps(data, indent=2), encoding="utf-8")

    def load(self) -> None:
        data = json.loads(self.path.read_text(encoding="utf-8"))
        self.tasks = [Task(**item) for item in data["tasks"]]
        self._next_id = int(data["next_id"])
```

Starter code for `tests/conftest.py`:

```python
from datetime import UTC, datetime

import pytest

from todo_app.tasks import TodoStore


@pytest.fixture
def moment():
    return datetime(2026, 1, 5, 9, 15, tzinfo=UTC)


@pytest.fixture
def fixed_clock(moment):
    return lambda: moment


@pytest.fixture
def store(tmp_path, fixed_clock):
    return TodoStore(tmp_path / "todos.json", clock=fixed_clock)
```

Starter code for `tests/unit/test_tasks_unit.py`:

```python
from datetime import UTC, datetime
from unittest.mock import Mock

import pytest

from todo_app import tasks
from todo_app.tasks import TodoStore


def test_add_assigns_sequential_ids(store):
    first = store.add("write lesson")
    second = store.add("review code")
    assert (first.id, second.id) == (1, 2)


def test_add_records_clock_time(store):
    task = store.add("write lesson")
    assert task.created == "2026-01-05T09:15:00+00:00"


@pytest.mark.parametrize("title", ["", "   "])
def test_add_rejects_blank_title(store, title):
    with pytest.raises(ValueError):
        store.add(title)


@pytest.mark.parametrize("raw, expected", [("  tea ", "tea"), ("cake", "cake")])
def test_add_strips_whitespace(store, raw, expected):
    assert store.add(raw).title == expected


def test_complete_marks_task_done(store):
    task = store.add("ship")
    store.complete(task.id)
    assert store.open_tasks() == []


def test_complete_unknown_id_raises(store):
    with pytest.raises(KeyError):
        store.complete(99)


def test_notify_receives_message_on_complete(tmp_path, fixed_clock):
    notify = Mock()
    store = TodoStore(tmp_path / "todos.json", clock=fixed_clock, notify=notify)
    task = store.add("ship")
    store.complete(task.id)
    notify.assert_called_once_with("done: ship")


def test_default_clock_is_used_when_none_given(tmp_path, monkeypatch):
    fake = datetime(2030, 2, 3, 4, 5, tzinfo=UTC)
    monkeypatch.setattr(tasks, "_utc_now", lambda: fake)
    store = TodoStore(tmp_path / "todos.json")
    assert store.add("x").created == "2030-02-03T04:05:00+00:00"
```

Starter code for `tests/integration/test_tasks_integration.py`:

```python
import json
from pathlib import Path
from unittest.mock import patch

import pytest

from todo_app.tasks import TodoStore

pytestmark = pytest.mark.integration


def test_save_and_reload_round_trip(tmp_path, fixed_clock):
    path = tmp_path / "todos.json"
    store = TodoStore(path, clock=fixed_clock)
    store.add("one")
    store.add("two")
    store.complete(1)
    store.save()

    reloaded = TodoStore(path, clock=fixed_clock)
    assert [t.title for t in reloaded.open_tasks()] == ["two"]
    assert reloaded.add("three").id == 3


def test_missing_file_starts_empty(tmp_path, fixed_clock):
    store = TodoStore(tmp_path / "absent.json", clock=fixed_clock)
    assert store.tasks == []


def test_corrupt_file_raises(tmp_path):
    path = tmp_path / "bad.json"
    path.write_text("{bad", encoding="utf-8")
    with pytest.raises(json.JSONDecodeError):
        TodoStore(path)


def test_save_failure_is_reported(tmp_path, fixed_clock):
    store = TodoStore(tmp_path / "todos.json", clock=fixed_clock)
    store.add("one")
    with patch.object(Path, "write_text", side_effect=OSError("disk full")):
        with pytest.raises(OSError, match="disk full"):
            store.save()
```

The empty `__init__.py` files in `tests/`, `tests/unit/` and `tests/integration/` are required so that pytest imports each folder as a package without name clashes.

Step-by-step tasks:

1. Create the repo folder, run `git init -b main`, and create the files above. Add `.venv/`, `dist/`, `build/`, `*.egg-info/`, `.pytest_cache/` and `__pycache__/` to `.gitignore`.
2. Create the environment with `py -3.14 -m venv .venv`, activate it, and install the project with `python -m pip install -e ".[dev]"`. Record the final line in `docs/findings.md`.
3. Run `python -m pytest --collect-only -q` and paste the list of test names into `docs/findings.md`.
4. Run `python -m pytest` and make every test pass. Record the summary line.
5. Run `python -m pytest -m "not integration"` and record how many tests were deselected. Explain in one sentence why the fast run is the one to repeat on every save.
6. Add a unit test `test_open_tasks_excludes_done` that adds two tasks, completes one and checks the open list. Add a parametrized unit test that checks `store.add` rejects a title made only of tabs and spaces.
7. Add an integration test that saves a store with three tasks, loads it into a new store, and checks that the `created` timestamps match the originals.
8. Use `unittest.mock.patch` on `Path.write_text` in one unit test to check that `save` writes UTF-8 by asserting the `encoding` keyword. Explain in `docs/findings.md` why this test checks a call rather than a file.
9. Run `python -m ruff check src tests` and `python -m ruff format --check src tests`, fix every issue, and record the results.
10. Run `python -m mypy` and record the success line.
11. Write a section in `docs/findings.md` titled "What the mocks could hide", in at least three sentences, that names one way a mock could pass while the real code fails.
12. Commit with the message "Add todo store with unit and integration tests" and push to GitHub.

Done when:

- `python -m pytest` runs at least seventeen tests and passes
- `python -m pytest -m "not integration"` runs the unit tests only and passes
- `python -m mypy` reports no issues in `src`, and Ruff reports no issues in `src` and `tests`
- `docs/findings.md` contains the collection list, the summary lines, the mock explanation, and the section on hidden mock failures
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the source and test files, `pyproject.toml`, `.gitignore`, `README.md` (with the setup and test commands in order), and `docs/findings.md`.

The project is small so that the testing choices are visible. A reviewer should be able to tell from the file names which tests are fast, which touch the disk, and which use a double, and should be able to run the fast group in seconds.

---

## Check your understanding

1. A test named `test_add` calls `store.add("tea")` and asserts only that no exception is raised. Explain why this test is weak, and rewrite its assertion.
2. Two tests share a session-scoped fixture that returns a list. The second test fails only when run after the first. Explain the likely cause, and state the fixture scope change that fixes it.
3. A test patches `todo_app.tasks.Path.write_text` and asserts it was called. A reviewer says the test checks the implementation, not the behavior. Explain the argument, and state what behavior the test should check instead.
4. A developer mocks `datetime` in a module with `patch("datetime.datetime")`, but the module already imported `datetime` with `from datetime import datetime`. Explain why the patch has no effect, and name the place to patch.
5. The integration tests fail on a teammate's machine because they write to `C:\temp\todos.json`. Explain the problem, and name the pytest fixture that fixes it.
6. A pytest run prints "collected 0 items" after a developer adds a file named `checks.py` with test functions. Explain the naming rule that causes this, and give the fix.

### Answers

1. The test passes for any return value, because it checks only that nothing failed, so it does not protect the behavior that the title is stored. The better assertion checks the result: `assert store.add("tea").title == "tea"`.
2. The session-scoped fixture is shared, so the first test changes the list and the second sees the changed state. The fix is to use the default function scope, so each test receives a fresh list, or to avoid mutation of the shared value.
3. Patching `write_text` and asserting it was called ties the test to the way the code writes the file, so a valid refactor breaks it. The behavior to check is that saving and loading gives back the same tasks, which the integration test does through the real file.
4. The module already holds a reference to the original `datetime` class, so replacing the name in the `datetime` module does not change the module's own name. The patch must target the name where the module uses it, which is `module_under_test.datetime`.
5. The path is specific to one machine, and the folder may not exist or may be read-only on another. The fix is the built-in `tmp_path` fixture, which gives each test a unique temporary folder that pytest manages.
6. Pytest collects only files whose names match `test_*.py` or `*_test.py` by default. A file named `checks.py` is not collected, so its functions are never run. The fix is to rename the file to `test_checks.py`, and to run `python -m pytest --collect-only -q` to confirm.

---

## Sources to read, watch and practice

*Read*

- pytest documentation, "How to write and report assertions in tests" and "How to use fixtures", at docs.pytest.org
- pytest documentation, "How to parametrize fixtures and test functions", at docs.pytest.org
- pytest documentation, "Good Integration Practices", at docs.pytest.org
- Python documentation, module `unittest.mock`, at docs.python.org/3/library/unittest.mock.html
- Python documentation, module `unittest`, at docs.python.org/3/library/unittest.html
- [Wikipedia: Unit testing](https://en.wikipedia.org/wiki/Unit_testing)
- [Wikipedia: Integration testing](https://en.wikipedia.org/wiki/Integration_testing)
- [Wikipedia: Test double](https://en.wikipedia.org/wiki/Test_double)

*Tutorials*

- GeeksforGeeks, "Python Unit Testing" and "Python Mock Objects" tutorials, at geeksforgeeks.org
- W3Schools, "Python Testing" page, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segment on testing and the unit test examples.
- The pytest project's own talks and tutorials on its YouTube channel, which cover fixtures and parametrization with worked examples. Search the channel for "pytest fixtures".

*Practice*

- Exercism, the Python track at exercism.org/tracks/python, where each exercise ships with tests you run locally against your solution.

---

## Tomorrow

Day 26 is the capstone for Module 3. You will build a multi-module command-line service that stores its data in SQLite, logs its actions, and has tests, documentation and a Git workflow that another developer could follow. The packaging from Day 24, the errors from Day 22, the standard library from Day 23 and the testing from today all come together in one project.

**Next:** [Day 26: Python Capstone](Software-Day-26-Python-Capstone.md)

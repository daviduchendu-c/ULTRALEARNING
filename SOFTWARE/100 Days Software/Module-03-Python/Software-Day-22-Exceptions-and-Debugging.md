# Day 22: Exceptions and Debugging

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You have written classes, generators and context managers from Day 19 to Day 21. This is for anyone who has seen a long traceback and stopped reading after the first line, or who has added `print` statements to find a bug and then forgotten to remove them.

**By the end of this day, you will have:**

- The ability to read a Python traceback from the bottom up and find the line that caused the failure
- Precise use of `try`, `except`, `else` and `finally`, and the knowledge of which block runs when
- The ability to re-raise an error with a bare `raise`, and to chain a new error to its cause with `raise ... from`
- A custom exception hierarchy with useful attributes
- A clear rule for when to use `assert`, and when to raise an explicit exception instead
- Working use of the `pdb` debugger, including `breakpoint()`, and the logging module at the right levels
- A config-loader repo with tests for each error path, and a logging check

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 9: Python Runtime and Syntax (running scripts with `python`)
- Day 11: Numbers and Operators (conversions that can fail)
- Day 16 and Day 17: Functions (raising from functions)
- Day 21: Python Data Model (context managers, the `finally` idea)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Tracebacks](#tracebacks)
  - [Try except else finally](#try-except-else-finally)
  - [Raise](#raise)
  - [Custom exceptions](#custom-exceptions)
  - [Assertions](#assertions)
  - [Debugger](#debugger)
  - [Logging](#logging)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a config loader with errors and logs](#project-a-config-loader-with-errors-and-logs)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A program that fails gives you information. The traceback tells you where the program was, the message tells you what went wrong, and the logs tell you what was happening before the failure. Developers who read these carefully fix bugs in minutes. Developers who skip them spend hours guessing. The skill this day builds is reading errors, handling the ones you expect, and letting the others surface with enough context to act on.

Exceptions in Python are objects, and the mechanism that carries them through the call stack is the same mechanism you will use to model expected failures in your own code. A missing file, a malformed number and a failed network call are all exceptions. The question is which ones your code should catch, which ones it should translate into its own error types, and which ones it should let escape to the top. Answering that question is the core of good error handling.

The official references are the Python tutorial section "Errors and Exceptions" at docs.python.org/3/tutorial/errors.html and the language reference section "The try statement" at docs.python.org/3/reference/compound_stmts.html#the-try-statement. The `logging` module has its own how-to guide at docs.python.org/3/howto/logging.html, which is worth reading before the logging exercise.

---

## Glossary

- **Exception:** an object that signals an error and carries it up the call stack until a handler catches it.
- **Traceback:** the printed record of the call stack at the moment an exception was raised.
- **Frame:** the record of one function call in the stack, showing the file, line and function.
- **Handler:** an `except` clause that catches an exception of a given type.
- **Re-raise:** raising the current exception again, with a bare `raise` inside an `except` block.
- **Exception chaining:** linking one exception to the one that caused it, using `raise ... from` or the implicit context Python records.
- **Custom exception:** an exception class you define, usually by subclassing `Exception`.
- **Assertion:** an `assert` statement that checks a condition and raises `AssertionError` when it is false.
- **Breakpoint:** a point in code where execution pauses so that you can inspect it.
- **pdb:** the Python debugger, a standard library module for stepping through code.
- **Logger:** an object that records messages at levels such as DEBUG, INFO, WARNING, ERROR and CRITICAL.
- **Log level:** a number that ranks a message's importance, so a logger can filter by severity.

---

## Concepts

The seven concepts follow the life of an error: how to read it, how to catch it, how to raise it, how to name it, how to check your own assumptions, how to find it, and how to record it.

### Tracebacks

A traceback is a map of where the program was when it failed. Think of a set of footprints in snow. The last footprint is where the failure happened, and the trail shows how you got there. Read the traceback from the bottom up: the last line names the exception, and the lines above it show the calls that led to it.

From first principles, when an exception is not caught, Python prints each frame of the call stack, starting with the outermost call, and then the exception type and message on the final line. Each frame shows the file, the line number, the function name and the source line. The bottom frame is the one that raised the exception. When one exception occurs while handling another, Python prints both, with a message such as "During handling of the above exception, another exception occurred" or "The above exception was the direct cause of the following exception".

```python
def parse_port(text: str) -> int:
    return int(text)


def load(text: str) -> int:
    return parse_port(text)


load("eighty")
```

Running this prints a traceback whose last line is `ValueError: invalid literal for int() with base 10: 'eighty'`. The frames above it show that `load` called `parse_port`, which called `int`. The bottom frame in the file is the `int` call inside `parse_port`, which is the place to look first.

The common mistake is reading only the last line and changing the wrong code, or reading only the first line and missing the actual cause in a chained exception. The fix is to read the whole chain, and to find the first frame that belongs to your code. In the job market, a developer who reads tracebacks quickly is one of the most valued members of an on-call rotation, because the traceback is often the only evidence available from a production system.

### Try except else finally

A `try` block is a protected region. Think of a bridge inspection with a plan for each outcome. The inspector does the work in the `try` block. If a crack is found, the `except` block describes the response. If the work finishes cleanly, the `else` block records that the work is done. The `finally` block runs in every case, which is where the cleanup goes.

From first principles, Python runs the `try` block. If no exception occurs, it runs the `else` block and then the `finally` block. If an exception occurs, Python searches the `except` clauses in order, and runs the first one whose type matches. The `finally` block runs after the `try`, the `except` or the `else`, even if one of them returns or raises. An exception raised inside `else` is not caught by the `except` clauses above it, which is why `else` is the place for code that should not be protected by the handler.

```python
def read_number(text: str) -> int | None:
    try:
        value = int(text)
    except ValueError:
        print(f"not a number: {text!r}")
        return None
    else:
        print(f"parsed {value}")
        return value
    finally:
        print("done with", repr(text))


print(read_number("42"))
print(read_number("x"))
```

The output shows that the `else` runs only for the successful parse, and the `finally` runs for both. Notice that `return` inside `try` or `except` still lets `finally` run, which is why cleanup belongs there.

The common mistake is catching `Exception` or a bare `except:` around a large block, which hides bugs such as `TypeError` from a typo. Catch the narrowest exception type you expect, and keep the `try` block small. A bare `except:` also catches `KeyboardInterrupt` and `SystemExit`, so it stops Ctrl+C from working. In the job market, broad exception handlers are a common review comment, and the fix is to name the exception and keep the protected code short.

### Raise

`raise` starts an exception on purpose. Think of an alarm that a sensor triggers when a condition is wrong. The alarm carries a message and a type, and it travels up to whoever is listening. Raising is how a function says "I cannot do this safely", instead of returning a value that looks like success.

From first principles, `raise ValueError("message")` creates an exception object and starts unwinding the stack. A bare `raise` inside an `except` block re-raises the exception that is being handled, which keeps the original traceback. `raise NewError("text") from original` raises a new exception and records `original` as its cause, so the traceback shows both. Without `from`, Python still records the implicit context in `__context__`, but the message is different. Use `from` when you translate one exception into another, so the original detail is not lost.

```python
def parse_port(text: str) -> int:
    try:
        return int(text)
    except ValueError as exc:
        raise ValueError(f"port must be a number, got {text!r}") from exc


def ensure_port(port: int) -> int:
    if not 1 <= port <= 65535:
        raise ValueError(f"port out of range: {port}")
    return port


try:
    parse_port("abc")
except ValueError as err:
    print(err)               # port must be a number, got 'abc'
    print(err.__cause__)     # invalid literal for int() with base 10: 'abc'
```

The outer error is clear to the user, and the inner one is kept for the developer. That is the purpose of chaining.

The common mistake is raising a new exception inside `except` without `from`, which obscures the first cause, or re-raising with `raise err` instead of bare `raise`, which adds a new frame and changes the traceback. The second mistake is raising a generic `Exception`, which callers cannot catch selectively. Choose a specific type. In the job market, teams often define one exception class per library, and callers catch that class, not the internals.

### Custom exceptions

A custom exception is a named error for your own domain. Think of a set of specialized warning lights on a dashboard. A general warning light says "something is wrong", but a light labeled "low oil" tells the driver what to do. Custom exception classes let callers handle the cases they understand and let the rest pass through.

From first principles, an exception class is a subclass of `BaseException`, and in practice of `Exception`. Defining `class ConfigError(Exception): ...` creates a new type that `except ConfigError` can match. Subclasses form a hierarchy, so a handler for `ConfigError` also catches `MissingKeyError` if that class inherits from it. Exceptions can store attributes, such as the missing key, and a constructor can build a useful message from them.

```python
class ConfigError(Exception):
    """Raised when a configuration cannot be used."""


class MissingKeyError(ConfigError):
    def __init__(self, key: str) -> None:
        super().__init__(f"missing required key: {key}")
        self.key = key


try:
    raise MissingKeyError("port")
except ConfigError as err:
    print(type(err).__name__, err.key)   # MissingKeyError port
```

The handler catches the parent class and still has access to the child's attribute, because the child is an instance of the parent.

The common mistake is creating an exception class for every small case, which makes handlers hard to follow. Create a class when a caller will catch it separately or needs data from it. A second mistake is subclassing `BaseException` directly, which bypasses the normal handlers and surprises callers. In the job market, a clear error hierarchy is one of the signs of a maintainable library, and reviewers ask whether a new error class is needed or whether an existing one fits.

### Assertions

An assertion checks something that must be true if the code is correct. Think of a pilot's pre-flight checklist. It covers conditions that the pilot's own steps must have produced. If an item fails, something in the process is broken.

From first principles, `assert condition, message` evaluates the condition and raises `AssertionError` with the message when the condition is false. Assertions are removed when Python runs with the `-O` option, which skips them. That means they must never be the only check on user input, file contents or network data, because those checks would disappear in optimized runs. Use assertions for invariants that your own code guarantees, such as the result of an internal function being sorted, and use explicit `if` checks with `raise` for input validation.

```python
def average(values: list[float]) -> float:
    if not values:
        raise ValueError("values must not be empty")
    total = sum(values)
    assert len(values) > 0, "guard above should prevent this"
    return total / len(values)


print(average([2.0, 4.0]))   # 3.0
```

The explicit `ValueError` handles bad input from callers. The assertion documents an invariant that the guard already guarantees. If someone edits the guard away, the assertion catches the mistake during testing.

The common mistake is using `assert` to validate arguments from users or from a file, then running in production with `-O`, where the check silently disappears. The second mistake is an assertion with side effects, such as `assert queue.pop()`, which disappears entirely under `-O` and removes the side effect with it. In the job market, reviewers look for asserts on input and for assertions with side effects, because both are easy to miss in testing.

### Debugger

The debugger lets you pause a program and look around. Think of a film director who stops the scene to check the set. You can see every variable in place, step one line at a time, and resume when you have what you need. `pdb` is Python's standard debugger, and `breakpoint()` is the built-in way to start it.

From first principles, `breakpoint()`, available since Python 3.7, calls the function named by the `PYTHONBREAKPOINT` environment variable, which defaults to `pdb.set_trace`. The debugger stops at that line and gives you a prompt. Common commands are `n` for next line, `s` to step into a call, `c` to continue, `p name` to print a value, `l` to list the code around the current line, `w` to show the stack, and `q` to quit. You can also start a whole program under the debugger with `python -m pdb script.py`, which stops at the first line.

```python
def total_with_tax(prices: list[float], rate: float) -> float:
    subtotal = sum(prices)
    breakpoint()
    return subtotal * (1 + rate)


print(total_with_tax([2.5, 4.25], 0.1))
```

Run the file, type `p subtotal` at the prompt to see its value, `n` to move to the next line, and `c` to finish. Type `q` to quit at any point. In automated tests, set `PYTHONBREAKPOINT=0` to turn `breakpoint()` into a no-op, so a stray call does not stop the test run.

The common mistake is leaving `breakpoint()` in committed code, which stops a server or a test run unexpectedly. Search for it before every commit, and use a linter rule that flags it. The second mistake is debugging by changing code at random instead of forming a hypothesis, then checking the hypothesis with the debugger or a print. In the job market, debugging is a skill interviewers observe, and a systematic approach matters more than speed.

### Logging

Logging is a structured diary for a program. Think of a ship's log. Each entry is written with a time, a severity and a note, and the captain can read the whole log later to see what happened. Unlike `print`, the logging module lets you set severity levels, send messages to files or services, and turn detail up or down without changing the code.

From first principles, `logging.getLogger(__name__)` returns a logger named after the module. Each logger has a level, and it passes messages at or above that level to its handlers. The standard levels are DEBUG (10), INFO (20), WARNING (30), ERROR (40) and CRITICAL (50). `logging.basicConfig` sets up a simple handler for scripts. Calling `logger.exception("message")` inside an `except` block logs the message at ERROR with the traceback attached. Use `%s` placeholders in log calls, rather than f-strings, so the message is formatted only when it is actually emitted.

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(name)s: %(message)s")
logger = logging.getLogger("demo")

logger.debug("hidden, below INFO")
logger.info("loaded %d rows", 3)
logger.warning("disk %d%% full", 91)
```

Running the file prints the INFO and WARNING lines, and the DEBUG line is suppressed because the level is INFO. The `%%` in the warning prints a literal percent sign.

The common mistake is logging sensitive data, such as passwords or card numbers, which then ends up in files and dashboards. Log identifiers, not secrets. The second mistake is using `print` for diagnostics in a service, which cannot be filtered or routed. In the job market, every production service logs at levels that operators can change, and reviewers check that errors are logged once, at the place that handles them.

---

## Hands-on exercises

Work in a folder called `errors-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 reads tracebacks.

1. Create `trace.py` with the `parse_port` and `load` functions from the tracebacks section. Run it and copy the full traceback.
2. Label each frame in your copy with the function it names, and mark the frame that raised the error.
3. Write one sentence on which line you would change to fix the program, and why the top of the traceback is not the place to look first.

Done when you can name the failing call and the function that made it.

Exercise 2 practices `try`, `except`, `else` and `finally`.

1. Create `flow.py` with `read_number` from the concepts section.
2. Call it with `"42"`, `"x"` and `""`. Record the output and note which blocks ran for each input.
3. Change the `except` to catch `Exception` and then `TypeError`. Explain in one sentence why catching `Exception` is the weaker choice here.

Done when every output matches your prediction and your sentence names the risk of a broad handler.

Exercise 3 practices raise and chaining.

1. Create `chain.py` with `parse_port` that raises a `ValueError` with a clear message and `from` the original.
2. Call it with `"abc"` inside a `try` block, then print the message and `__cause__`.
3. Write a function that uses a bare `raise` after logging, and call it inside a `try` block. Record that the error still propagates.

Done when both the message and the cause are printed, and the bare `raise` propagates the error.

Exercise 4 builds a custom exception hierarchy.

1. Create `errors.py`. Define `ConfigError(Exception)` and `MissingKeyError(ConfigError)` with a `key` attribute.
2. Raise `MissingKeyError("port")` and catch it with `except ConfigError`.
3. Print the class name and the key attribute.

Done when the handler for the parent class catches the child and the key is available.

Exercise 5 uses the debugger and logging.

1. Create `debug_it.py` with `total_with_tax` and a `breakpoint()` call, as in the concepts section.
2. Run it, use `p subtotal`, `n` and `c` at the prompt, and record what each command printed.
3. Create `log_it.py` with `logging.basicConfig(level=logging.INFO)` and log one message at each of DEBUG, INFO and WARNING. Run it and record which lines appear.

Done when you have recorded the debugger output and the logging output shows only INFO and WARNING.

---

## Project: a config loader with errors and logs

The project is a small config loader. It parses a JSON text, requires a port key, validates the port, translates low-level errors into its own exception hierarchy with chaining, uses `try`, `except`, `else` and `finally` for file reading, and logs its decisions. The tests check each error path and each log message. The evidence is a passing test suite and a findings document that records one traceback you read and one debugging session.

Suggested repo name: `100-days-software-day-22-exceptions-and-debugging`.

Repo layout:

```
100-days-software-day-22-exceptions-and-debugging/
├── .gitignore
├── README.md
├── config_loader.py
├── tests/
│   ├── __init__.py
│   └── test_config_loader.py
└── docs/
    └── findings.md
```

Starter code for `config_loader.py`:

```python
import json
import logging

logger = logging.getLogger(__name__)


class ConfigError(Exception):
    """Raised when a configuration cannot be used."""


class MissingKeyError(ConfigError):
    """Raised when a required key is absent."""

    def __init__(self, key: str) -> None:
        super().__init__(f"missing required key: {key}")
        self.key = key


def parse_config(text: str) -> dict[str, object]:
    try:
        data = json.loads(text)
    except json.JSONDecodeError as exc:
        raise ConfigError("config is not valid JSON") from exc
    if not isinstance(data, dict):
        raise ConfigError("config must be a JSON object")
    return data


def require(config: dict[str, object], key: str) -> object:
    assert isinstance(config, dict), "internal: parse_config must return a dict"
    if key not in config:
        raise MissingKeyError(key)
    return config[key]


def load_port(text: str) -> int:
    config = parse_config(text)
    raw = require(config, "port")
    if not isinstance(raw, int) or isinstance(raw, bool):
        raise ConfigError("port must be an integer")
    if not 1 <= raw <= 65535:
        raise ConfigError(f"port out of range: {raw}")
    logger.info("port %d accepted", raw)
    return raw


def load_port_or_log(text: str) -> int:
    try:
        return load_port(text)
    except ConfigError:
        logger.exception("could not load port")
        raise


def read_file_safely(path: str) -> str | None:
    try:
        handle = open(path, encoding="utf-8")
    except FileNotFoundError:
        logger.warning("config file not found: %s", path)
        return None
    else:
        with handle:
            text = handle.read()
        return text
    finally:
        logger.debug("checked %s", path)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    print(load_port('{"port": 8080}'))
    try:
        load_port('{"host": "example"}')
    except ConfigError as err:
        print("error:", err)
```

Starter code for `tests/test_config_loader.py`:

```python
import json
import os
import tempfile
import unittest

from config_loader import (
    ConfigError,
    MissingKeyError,
    load_port,
    load_port_or_log,
    parse_config,
    read_file_safely,
)


class ConfigLoaderTests(unittest.TestCase):
    def test_valid_port(self):
        self.assertEqual(load_port('{"port": 8080}'), 8080)

    def test_invalid_json_chains_the_cause(self):
        with self.assertRaises(ConfigError) as ctx:
            parse_config("{not json")
        self.assertIsInstance(ctx.exception.__cause__, json.JSONDecodeError)

    def test_missing_key_is_specific(self):
        with self.assertRaises(MissingKeyError) as ctx:
            load_port('{"host": "example"}')
        self.assertEqual(ctx.exception.key, "port")
        self.assertIsInstance(ctx.exception, ConfigError)

    def test_port_must_be_integer(self):
        with self.assertRaises(ConfigError):
            load_port('{"port": "80"}')
        with self.assertRaises(ConfigError):
            load_port('{"port": true}')

    def test_port_range(self):
        with self.assertRaises(ConfigError):
            load_port('{"port": 70000}')

    def test_non_object_json(self):
        with self.assertRaises(ConfigError):
            parse_config("[1, 2, 3]")

    def test_missing_file_returns_none_and_warns(self):
        with self.assertLogs("config_loader", level="WARNING") as logs:
            self.assertIsNone(read_file_safely("no-such-file.json"))
        self.assertTrue(any("not found" in line for line in logs.output))

    def test_finally_logs_even_when_file_is_missing(self):
        with self.assertLogs("config_loader", level="DEBUG") as logs:
            read_file_safely("no-such-file.json")
        self.assertTrue(any("checked" in line for line in logs.output))

    def test_existing_file_is_read(self):
        with tempfile.TemporaryDirectory() as folder:
            path = os.path.join(folder, "config.json")
            with open(path, "w", encoding="utf-8") as handle:
                handle.write('{"port": 9000}')
            self.assertEqual(read_file_safely(path), '{"port": 9000}')

    def test_load_port_or_log_logs_and_reraises(self):
        with self.assertLogs("config_loader", level="ERROR") as logs:
            with self.assertRaises(ConfigError):
                load_port_or_log('{"port": 0}')
        self.assertTrue(any("could not load port" in line for line in logs.output))


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python config_loader.py` and paste the output into `docs/findings.md`. Explain in one sentence why the second call prints an error and no traceback.
3. Run `python -m unittest` and make every test pass.
4. Write a deliberate bug: change `load_port` so that it returns `raw + 1`. Run the tests and copy the failing assertion. Revert the change and explain in `docs/findings.md` how the failing message led you to the line.
5. Use the debugger on a failing input. Add a temporary `breakpoint()` in `load_port` just before the range check, run `python -m unittest tests.test_config_loader.ConfigLoaderTests.test_port_range` with `PYTHONBREAKPOINT` unset, inspect `raw` with `p raw`, and continue with `c`. Remove the breakpoint and record in `docs/findings.md` what you saw.
6. Add a function `load_port_file(path: str) -> int` that reads the file with `read_file_safely` and raises `ConfigError` when the file is missing. Add a test that the error is raised and chained or contextualized, and explain in one sentence which handler of the exception runs.
7. Run `python -m mypy config_loader.py tests/test_config_loader.py` and fix every issue.
8. Commit with the message "Add config loader with exception hierarchy and logging tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least eleven tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the program output, the deliberate-bug explanation, the debugger notes, and the sentence about the handler
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with run commands and the note that `PYTHONBREAKPOINT=0` disables breakpoints in CI), and `.gitignore`.

The loader is small so each error path is easy to trace. A reviewer should be able to start from a failing test, follow the traceback to the line in `load_port`, and see which log line the operator would read.

---

## Check your understanding

1. A traceback ends with `KeyError: 'port'`, and the frame above it is in your code. Explain which line you examine first, and what the `KeyError` tells you about the dictionary.
2. A developer wraps a 200-line function in `try: ... except Exception: pass`. Name the two concrete harms, and rewrite the handler to catch the expected error only.
3. Explain the difference between a bare `raise` and `raise err` inside an `except` block, using the traceback as evidence.
4. A validation check uses `assert port > 0` on a value from a user form. Explain why this is wrong, and write the replacement.
5. A logging call is written as `logger.debug(f"user {user.card_number} paid")`. Name the two problems, and rewrite the call.
6. Why does `finally` run even when a `return` appears in the `try` block, and why does that matter for a file that must be closed?

### Answers

1. Examine the line in your code that performs the lookup, and check whether the key is present. `KeyError: 'port'` means the dictionary has no `"port"` key, so the input does not match the expected shape, or the code assumed a key that was never set.
2. The first harm is that real bugs, such as a `TypeError` from a typo, are silently swallowed, and the program continues with wrong state. The second is that the function hides which operation failed, which makes debugging much harder. The rewrite catches only the expected error, such as `except FileNotFoundError:`, and keeps the protected code short.
3. A bare `raise` re-raises the same exception object with its original traceback. `raise err` raises the object again, and the traceback shows the new raise point as an additional frame, which can hide the original location. The original traceback is preserved only by the bare form.
4. The assertion is removed when Python runs with `-O`, so the check disappears in optimized runs, and the bad input reaches the rest of the program. The replacement is an explicit check that raises a specific error, such as `if port <= 0: raise ConfigError("port must be positive")`.
5. The f-string formats the message even when DEBUG is disabled, and it writes a card number into the log, which is sensitive data. The rewrite logs an identifier without the secret, using lazy formatting: `logger.debug("user %s paid", user.id)`.
6. The language guarantees that the `finally` block runs after the `try` block completes by any route, including `return`, `break`, `continue` or an exception. For a file, this means that a `finally` or a `with` block can close the handle even if the function returns early, which prevents leaked file descriptors.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Errors and Exceptions", at docs.python.org/3/tutorial/errors.html
- Python language reference, "The try statement", at docs.python.org/3/reference/compound_stmts.html#the-try-statement
- Python documentation, module `pdb`, at docs.python.org/3/library/pdb.html
- Python documentation, module `logging`, and the "Logging HOWTO", at docs.python.org/3/howto/logging.html
- PEP 3134, "Exception Chaining and Embedded Tracebacks", at peps.python.org/pep-3134/
- [Wikipedia: Exception handling](https://en.wikipedia.org/wiki/Exception_handling)
- [Wikipedia: Debugger](https://en.wikipedia.org/wiki/Debugger)

*Tutorials*

- GeeksforGeeks, "Python Exception Handling" and "Python Logging" tutorials, at geeksforgeeks.org
- W3Schools, "Python Try Except" page, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on exceptions and unit testing, which connect to today's work.

*Practice*

- Python Tutor at pythontutor.com. Step through the `try`, `except`, `else` and `finally` example, and watch which block runs for each input.

---

## Tomorrow

Day 23 is the standard library tour that most real programs depend on. You will work with `pathlib` for file paths, read and write text, binary and JSON and CSV files, handle dates and times with `datetime`, count and group with `collections`, match text with `re`, and run another program with `subprocess`. The file handling you practiced today, with its encodings and its `with` blocks, is the base for all of it.

**Next:** [Day 23: Files and Standard Library](Software-Day-23-Files-and-Standard-Library.md)

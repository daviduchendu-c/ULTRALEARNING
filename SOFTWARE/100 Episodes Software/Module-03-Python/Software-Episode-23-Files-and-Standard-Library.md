# Day 23: Files and Standard Library

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You can handle exceptions, use context managers and write classes. This is for anyone who has rebuilt a date parser, a CSV reader or a counting loop that the standard library already provides, and who wants the tools that real Python programs use every day.

**By the end of this day, you will have:**

- The ability to build and join file paths with `pathlib`, and to read and write files without relying on the working directory
- A correct choice between text and binary modes, with explicit encodings
- Working use of `json` and `csv` for structured data, including the rules for each format
- The ability to parse, compare and format dates and times with `datetime`, and to avoid naive and aware datetime mistakes
- Fluency with `collections.Counter`, `defaultdict` and `deque` for counting and grouping
- The ability to search and extract text with `re`, and to write patterns that match only what you intend
- A safe way to run another program with `subprocess`, using a list of arguments and no shell
- A log-analyzer repo with tests that use only the standard library

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 12: Strings and Unicode (encodings, `encode` and `decode`)
- Day 14: Dictionaries and Sets (counting and grouping by key)
- Day 22: Exceptions and Debugging (`with` blocks, error handling for files)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Pathlib](#pathlib)
  - [Text and binary](#text-and-binary)
  - [JSON](#json)
  - [CSV](#csv)
  - [Datetime](#datetime)
  - [Collections](#collections)
  - [Regex](#regex)
  - [Subprocess concept](#subprocess-concept)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a log analyzer](#project-a-log-analyzer)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

The Python standard library ships with the interpreter, and it covers most of what a small service or a data script needs: files, paths, dates, structured text formats, pattern matching, counting and running other programs. Using it well saves you from writing and maintaining your own parsers, and it means your code runs wherever Python runs, without an extra install. Reviewers in paid work expect you to reach for these modules first, and they notice when someone hand-rolls a CSV parser that the standard library already handles.

This day covers the modules that touch the outside world most often. Files and paths come first, because almost every program reads or writes them. Then the structured formats, JSON and CSV, followed by dates, collections, regular expressions and subprocesses. The project combines them into a log analyzer, which is the kind of small tool that appears in every operations team.

The official reference is the Python standard library documentation at docs.python.org/3/library/index.html. The modules used today are `pathlib` (docs.python.org/3/library/pathlib.html), `json`, `csv`, `datetime`, `collections`, `re` and `subprocess`. Each has a short overview at the top of its page, and those overviews are worth reading before the code.

---

## Glossary

- **Path:** an object from `pathlib` that represents a file or folder location and offers operations on it.
- **Text mode:** opening a file so that reads return `str` and writes take `str`, with an encoding applied.
- **Binary mode:** opening a file so that reads return `bytes` and writes take `bytes`, with no encoding.
- **Serialization:** turning an object into text or bytes for storage or transfer.
- **JSON:** JavaScript Object Notation, a text format for objects, lists, strings, numbers, booleans and null.
- **CSV:** comma-separated values, a text format for tables with one record per line.
- **Naive datetime:** a datetime with no time zone attached.
- **Aware datetime:** a datetime that carries a time zone or an offset from UTC.
- **Timedelta:** a difference between two dates or times, used for arithmetic.
- **Counter:** a dictionary subclass in `collections` that counts hashable items.
- **Regular expression:** a pattern language for matching text, implemented by the `re` module.
- **Subprocess:** a separate program started by your program, with its own process and output streams.
- **Shell:** a command interpreter, such as Bash or PowerShell, that parses a command string and runs it.

---

## Concepts

The eight concepts cover the standard way to get data in and out of a program, and the tools that turn raw text into counts, dates and results.

### Pathlib

A path is an address. Think of a postal address that depends on where the letter starts. A relative address such as "the third shelf" makes sense only from one room, while an absolute address works from anywhere. `pathlib` gives you path objects that join addresses with a `/` operator, check whether a location exists, and read the parts of a name.

From first principles, `Path("logs") / "events.csv"` builds a new path object on the current platform, using the right separator for Windows or Linux. The object has methods such as `exists`, `is_file`, `read_text`, `write_text`, `open`, `glob` and `mkdir`, and properties such as `name`, `stem`, `suffix` and `parent`. A relative path resolves against the current working directory, which depends on where you ran the program. To find a file next to your script, start from `Path(__file__).resolve().parent`.

```python
from pathlib import Path

base = Path(__file__).resolve().parent
data = base / "data" / "events.csv"
print(data.name, data.suffix, data.parent.name)
print(data.exists())
```

The output depends on where the file sits. The `name` is the last part, the `suffix` is the extension, and the `parent` is the folder that contains it.

The common mistake is building paths by joining strings with `+` or with `os.path.join` in new code, which works but is easier to get wrong with separators and with empty parts. The second mistake is relying on the working directory, so the program works from one folder and fails from another. Anchor paths to `__file__` when a file belongs to the project. In the job market, path bugs show up when a script runs under a scheduler whose working directory is different from the developer's.

### Text and binary

A file is a sequence of bytes on disk. Think of a book written in a foreign alphabet. Text mode is a translator who reads the book and hands you sentences. Binary mode gives you the raw pages as they are. You choose the translator when the content is text, and you skip it when the content is data such as images or archives.

From first principles, `open(path, "r", encoding="utf-8")` decodes bytes into `str` using the encoding you name, and it converts line endings according to the `newline` setting. `open(path, "rb")` returns raw bytes. Text mode without an explicit encoding uses the platform default, which varies between systems, so always name the encoding. For large files, iterating over the file object reads one line at a time, which keeps memory use flat, while `read()` loads the whole file.

```python
from pathlib import Path

path = Path("notes.txt")
path.write_text("café\n", encoding="utf-8")

with path.open("rb") as handle:
    print(handle.read())          # b'caf\xc3\xa9\n'

with path.open(encoding="utf-8") as handle:
    for line in handle:
        print(line.rstrip())      # café
```

The binary read shows the bytes that the UTF-8 encoding produced, and the text read decodes them back to the original line.

The common mistake is opening a binary file in text mode, which fails on bytes that are not valid text, or writing text to a file opened in binary mode, which raises `TypeError`. Check the mode against the content. The second mistake is reading a huge file with `read()` when a line loop would do. In the job market, data pipelines are built from line iteration for this reason, and reviewers ask why a file is loaded whole.

### JSON

JSON is a text format for simple data. Think of a shopping list written on paper that every store can read. It has objects, arrays, strings, numbers, booleans and null, and nothing else. Python's `json` module converts between these and the built-in types, so a dictionary becomes an object and a list becomes an array.

From first principles, `json.dumps` turns a Python object into a string, and `json.loads` turns a string into objects. The `dump` and `load` functions do the same with file objects. JSON has no type for dates, sets, tuples or decimals, so `dumps` raises `TypeError` for those values unless you pass a `default` function that converts them. Dictionary keys become strings in JSON, so an integer key comes back as a string after a round trip.

```python
import json

summary = {"errors": 2, "codes": ["404", "500"], "hours": {9: 2, 10: 2}}
text = json.dumps(summary, sort_keys=True)
print(text)                     # {"codes": ["404", "500"], "errors": 2, "hours": {"9": 2, "10": 2}}
print(json.loads(text)["hours"]["9"])   # 2, the integer key came back as a string
```

The printed text shows that the integer keys became strings, and the lookup must use `"9"` after the round trip.

The common mistake is storing a `Decimal` or a `datetime` directly with `json.dumps`, which fails, and then converting it to a float, which loses precision for money. Convert to a string with a clear format, such as ISO 8601 for dates, and convert back on load. A second mistake is assuming that `json.loads` returns the same key types it received. In the job market, JSON is the common format for APIs and configuration, and reviewers check that the data types in the payload match the documented contract.

### CSV

CSV is a table in plain text. Think of a paper spreadsheet printed one row per line, with commas between the columns. Each row is a record, and the first row often names the columns. The format looks simple, but quoting rules make it easy to break if you split lines yourself.

From first principles, the `csv` module handles quoting, embedded commas, embedded line breaks and escaped quotes. `csv.reader` returns each row as a list of strings. `csv.DictReader` uses the first row as keys and returns a dictionary for each record. Files should be opened with `newline=""`, which lets the module handle line endings correctly, and with an explicit encoding. Every value read from a CSV file is a string, so numbers and dates must be converted.

```python
import csv
from io import StringIO

text = 'time,level,message\n09:15,ERROR,"disk, full"\n'
rows = list(csv.DictReader(StringIO(text)))
print(rows[0]["message"])   # disk, full
print(int("9"))             # every CSV field is text until you convert it
```

The comma inside the quotes stays in the message, which a naive `split(",")` would break.

The common mistake is splitting lines on commas. It fails on the first quoted field that contains a comma. Use `csv`. The second mistake is forgetting that a CSV field is always a string, so comparing `"10" > "9"` gives `False` because string comparison is alphabetical. Convert before comparing. In the job market, CSV exports from spreadsheets and databases are everywhere, and quoting bugs are one of the most common data-import problems.

### Datetime

A date and time is a point on a calendar and a clock. Think of a train timetable. The times are written in the local zone of the station, and a traveler from another zone must convert them. A naive datetime is a timetable without a zone, and an aware datetime is one that says which zone it is in.

From first principles, `datetime` holds a date and a time. `timedelta` holds a duration. Subtracting two datetimes gives a `timedelta`, and adding a timedelta moves a datetime forward. Comparing or subtracting a naive datetime with an aware one raises `TypeError`. `datetime.fromisoformat` parses the ISO 8601 text that JSON and CSV files commonly use, and `isoformat` writes it back. `datetime.now(timezone.utc)` returns an aware value in UTC, which is the safe default for stored timestamps.

```python
from datetime import datetime, timedelta, timezone

start = datetime.fromisoformat("2026-01-05T09:15:00")
end = start + timedelta(minutes=47)
print(end.isoformat())                      # 2026-01-05T10:02:00
print((end - start).total_seconds())        # 2820.0

utc_now = datetime.now(timezone.utc)
print(utc_now.tzinfo is not None)           # True, the value is aware
```

The first two lines show naive arithmetic. The last line shows an aware value, which can be compared safely with other aware values.

The common mistake is mixing naive and aware datetimes, which raises `TypeError` in comparisons, or storing local times without a zone, which makes later conversions ambiguous. Store UTC and convert only for display. The second mistake is using `datetime.utcnow()`, which returns a naive value and is deprecated in newer Python releases. In the job market, time-zone bugs appear in every system that serves more than one region, and reviewers check how timestamps are stored.

### Collections

Counting and grouping are the most common data tasks in a program. Think of a sorting table for mail. You place each letter in a labeled bin, and at the end you count the bins. `collections` provides specialized bins, so you do not have to write the get-or-create logic yourself every time.

From first principles, `Counter(items)` builds a dictionary that maps each item to its count, and it provides `most_common` for ranking. `defaultdict(list)` creates an empty list the first time a missing key is accessed, which removes the need for `setdefault`. `deque` is a double-ended queue with fast appends and pops at both ends, which suits sliding windows and queues. Each of these is a dictionary or sequence with a different default behavior, and each is faster than the hand-written version for its job.

```python
from collections import Counter, defaultdict, deque

levels = Counter(["INFO", "ERROR", "INFO", "WARN", "ERROR", "ERROR"])
print(levels.most_common(2))        # [('ERROR', 3), ('INFO', 2)]

groups: defaultdict[str, list[str]] = defaultdict(list)
groups["ERROR"].append("ERR-404")
print(dict(groups))                 # {'ERROR': ['ERR-404']}

recent = deque(maxlen=3)
for value in [1, 2, 3, 4]:
    recent.append(value)
print(list(recent))                 # [2, 3, 4], the oldest was dropped
```

The `maxlen` argument makes the deque discard the oldest item automatically, which is a simple window of the last three values.

The common mistake is using a `Counter` and expecting its keys to be in insertion order when you print them. The order is the insertion order for a dictionary subclass, but `most_common` sorts by count, which is the usual purpose. The second mistake is using a `defaultdict` and accidentally creating keys by reading them. A lookup with `[]` creates the key, so use `.get` for reads that should not change the dictionary. In the job market, counting by category appears in every report, and the right `collections` type makes the code shorter and clearer.

### Regex

A regular expression is a pattern for text. Think of a stencil that you lay over a page. Only the cut-out shapes show through, and everything else is hidden. A pattern such as `ERR-\d{3}` matches the letters ERR, a dash and exactly three digits, and nothing else.

From first principles, the `re` module compiles a pattern and searches a string for matches. `re.search` finds the first match anywhere, `re.match` checks only the start, `re.fullmatch` requires the whole string to match, and `re.findall` returns all matches. A raw string such as `r"\d+"` keeps backslashes intact, so the regex engine sees `\d`. Parentheses capture groups, which `findall` returns when present. The `\b` anchor matches a word boundary, which stops `XERR-404` from matching `ERR-404`.

```python
import re

pattern = re.compile(r"\bERR-(\d{3})\b")
line = "ERR-404 not found; ERR-500 server error; XERR-123 ignored"
print(pattern.findall(line))         # ['404', '500']

match = re.search(r"(\d{4})-(\d{2})-(\d{2})", "released 2026-01-05")
print(match.groups())                # ('2026', '01', '05')
```

The first pattern returns the captured three-digit codes and skips the code with a letter in front. The second pattern uses groups to split a date.

The common mistake is writing `.*` where `.*?` is meant, which matches greedily across several items and returns too much. A second mistake is forgetting the raw string prefix, so `"\b"` means a backspace character, not a word boundary. Test regex with examples that should match and examples that should not. In the job market, log parsing and input validation depend on regular expressions, and reviewers ask for tests of each pattern, because a pattern that is too loose lets bad input through.

### Subprocess concept

A subprocess is another program that your program starts. Think of hiring a courier. You give the courier a package with clear instructions, the courier does the job and reports back, and you check the result. The courier does not take instructions from strangers who rewrite your note on the way.

From first principles, `subprocess.run` starts a program and waits for it to finish. Passing the command as a list of arguments, such as `["git", "status"]`, runs the program directly, without a shell. That matters for safety: if a user-supplied file name contains spaces, quotes or semicolons, a list keeps it as one argument. With `shell=True`, the command is parsed by the shell, and an attacker can add commands. `capture_output=True` collects stdout and stderr, `text=True` decodes them as strings, `check=True` raises `CalledProcessError` on a nonzero exit code, and `timeout` stops a program that runs too long.

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "--version"],
    capture_output=True,
    text=True,
    check=True,
)
print(result.stdout.strip())     # for example: Python 3.14.6
```

The command uses `sys.executable`, which is the interpreter running the script, so the subprocess runs the same Python. That avoids picking up a different `python` from the PATH.

The common mistake is passing user input to `shell=True`, which turns a file name into a command injection. The second mistake is ignoring the return code and assuming the program succeeded. Use `check=True` or check `returncode` explicitly. In the job market, build scripts, deployment tools and test runners all run subprocesses, and security reviewers check every call that uses a shell.

---

## Hands-on exercises

Work in a folder called `stdlib-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices paths and files.

1. Create `paths.py` that builds a path to `data/events.csv` relative to the script's folder, using `Path(__file__)`.
2. Write a text file with `write_text` using `encoding="utf-8"`, then read it back in text mode and in binary mode, and record both results.
3. Write one sentence on what would change if you used a relative path and ran the script from a different folder.

Done when the binary read shows the UTF-8 bytes and the text read returns the original string.

Exercise 2 practices JSON and CSV.

1. Create `formats.py`. Build a dictionary with a list and a nested integer-keyed dictionary, and write it with `json.dumps` and `sort_keys=True`.
2. Load it back with `json.loads` and show that the integer keys are now strings.
3. Write a CSV file with a field that contains a comma, read it with `csv.DictReader`, and confirm the field is intact.

Done when the key types change as described and the CSV field stays whole.

Exercise 3 practices datetime.

1. Create `times.py`. Parse two ISO timestamps 47 minutes apart and print their difference in seconds.
2. Create one aware datetime in UTC with `datetime.now(timezone.utc)` and one naive datetime, then try to subtract them in a `try` block and record the error.
3. Convert a UTC timestamp to a different zone using `astimezone` with a `timezone` offset that you choose, and print both.

Done when the difference is 2820.0 seconds and the mixed subtraction raises `TypeError`.

Exercise 4 practices collections and regex.

1. Create `counts.py`. Use `Counter` on a list of log levels and print `most_common(2)`.
2. Use `defaultdict(list)` to group error codes by level, and print the result.
3. Create `codes.py` with a compiled pattern `\bERR-(\d{3})\b` and test it on a line that contains `ERR-404`, `XERR-123` and `ERR-5000`. Record which codes match and explain each result.

Done when the `ERR-5000` line matches only the first three digits if you have omitted the closing boundary, and your explanation names the boundary that prevents that.

Exercise 5 runs a subprocess safely.

1. Create `run_it.py` that runs `[sys.executable, "--version"]` with `capture_output=True`, `text=True` and `check=True`.
2. Print the stdout and the return code.
3. Run a command that fails, such as `[sys.executable, "-c", "import sys; sys.exit(3)"]`, inside a `try` block that catches `CalledProcessError`. Record the return code.

Done when the version prints and the failing command raises an error with return code 3.

---

## Project: a log analyzer

The project is a small log analyzer that reads a CSV of events, parses timestamps, counts events by level, groups error codes with a regular expression, builds an hourly histogram, writes a JSON summary, reads the first bytes of a file in binary mode, and reports the Python version through a safe subprocess call. It uses only the standard library. The evidence is a passing test suite and a generated summary file that you commit as sample output.

Suggested repo name: `100-days-software-day-23-files-and-standard-library`.

Repo layout:

```
100-days-software-day-23-files-and-standard-library/
├── .gitignore
├── README.md
├── analyzer.py
├── data/
│   └── events.csv
├── tests/
│   ├── __init__.py
│   └── test_analyzer.py
└── docs/
    ├── findings.md
    └── summary.json
```

Starter data for `data/events.csv`:

```
time,level,message
2026-01-05T09:15:00,INFO,started
2026-01-05T09:20:00,ERROR,ERR-404 not found
2026-01-05T10:02:00,WARN,slow query
2026-01-05T10:05:00,ERROR,ERR-500 server error
```

Starter code for `analyzer.py`:

```python
import csv
import json
import re
import subprocess
import sys
from collections import Counter, defaultdict
from collections.abc import Iterable
from datetime import datetime
from pathlib import Path

ERROR_PATTERN = re.compile(r"\bERR-(\d{3})\b")


def read_events(path: Path) -> list[dict[str, str]]:
    with path.open(encoding="utf-8", newline="") as handle:
        return list(csv.DictReader(handle))


def parse_timestamp(text: str) -> datetime:
    return datetime.fromisoformat(text)


def count_by_level(events: list[dict[str, str]]) -> Counter[str]:
    return Counter(event["level"] for event in events)


def hour_histogram(events: list[dict[str, str]]) -> dict[int, int]:
    histogram: defaultdict[int, int] = defaultdict(int)
    for event in events:
        histogram[parse_timestamp(event["time"]).hour] += 1
    return dict(sorted(histogram.items()))


def error_codes(lines: Iterable[str]) -> list[str]:
    codes: list[str] = []
    for line in lines:
        codes.extend(ERROR_PATTERN.findall(line))
    return codes


def write_summary(summary: dict[str, object], path: Path) -> None:
    path.write_text(json.dumps(summary, indent=2, sort_keys=True) + "\n", encoding="utf-8")


def file_signature(path: Path) -> bytes:
    with path.open("rb") as handle:
        return handle.read(4)


def python_version() -> str:
    result = subprocess.run(
        [sys.executable, "--version"],
        capture_output=True,
        text=True,
        check=True,
    )
    return result.stdout.strip()


if __name__ == "__main__":
    base = Path(__file__).resolve().parent
    events = read_events(base / "data" / "events.csv")
    summary = {
        "levels": dict(count_by_level(events)),
        "hours": {str(hour): count for hour, count in hour_histogram(events).items()},
        "error_codes": error_codes(event["message"] for event in events),
        "python": python_version(),
    }
    write_summary(summary, base / "docs" / "summary.json")
    print(json.dumps(summary, indent=2, sort_keys=True))
```

Starter code for `tests/test_analyzer.py`:

```python
import csv
import json
import os
import tempfile
import unittest
from collections import Counter
from pathlib import Path

from analyzer import (
    count_by_level,
    error_codes,
    file_signature,
    hour_histogram,
    parse_timestamp,
    python_version,
    read_events,
    write_summary,
)

SAMPLE = [
    {"time": "2026-01-05T09:15:00", "level": "INFO", "message": "started"},
    {"time": "2026-01-05T09:20:00", "level": "ERROR", "message": "ERR-404 not found"},
    {"time": "2026-01-05T10:02:00", "level": "WARN", "message": "slow query"},
    {"time": "2026-01-05T10:05:00", "level": "ERROR", "message": "ERR-500 server error"},
]


class AnalyzerTests(unittest.TestCase):
    def test_read_events_from_csv(self):
        with tempfile.TemporaryDirectory() as folder:
            path = Path(folder) / "events.csv"
            with path.open("w", encoding="utf-8", newline="") as handle:
                writer = csv.DictWriter(handle, fieldnames=["time", "level", "message"])
                writer.writeheader()
                writer.writerows(SAMPLE)
            self.assertEqual(read_events(path), SAMPLE)

    def test_quoted_comma_survives_csv(self):
        with tempfile.TemporaryDirectory() as folder:
            path = Path(folder) / "quoted.csv"
            path.write_text('time,level,message\n09:15,ERROR,"disk, full"\n', encoding="utf-8")
            rows = read_events(path)
            self.assertEqual(rows[0]["message"], "disk, full")

    def test_count_by_level(self):
        self.assertEqual(count_by_level(SAMPLE), Counter({"INFO": 1, "ERROR": 2, "WARN": 1}))

    def test_hour_histogram(self):
        self.assertEqual(hour_histogram(SAMPLE), {9: 2, 10: 2})

    def test_error_codes_ignore_embedded_letters(self):
        self.assertEqual(error_codes(["ERR-404 x", "XERR-123", "ERR-500 and ERR-404"]), ["404", "500", "404"])

    def test_parse_timestamp_rejects_bad_text(self):
        with self.assertRaises(ValueError):
            parse_timestamp("not a time")

    def test_write_summary_round_trip(self):
        with tempfile.TemporaryDirectory() as folder:
            path = Path(folder) / "summary.json"
            write_summary({"errors": 2, "codes": ["404", "500"]}, path)
            self.assertEqual(json.loads(path.read_text(encoding="utf-8")), {"errors": 2, "codes": ["404", "500"]})

    def test_file_signature_reads_bytes(self):
        with tempfile.TemporaryDirectory() as folder:
            path = Path(folder) / "doc.bin"
            path.write_bytes(b"%PDF-1.4 body")
            self.assertEqual(file_signature(path), b"%PDF")

    def test_python_version_uses_running_interpreter(self):
        self.assertTrue(python_version().startswith("Python 3."))


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files and the `data/` folder above, and add `__pycache__/` to `.gitignore`.
2. Run `python analyzer.py` from the project root. The script writes `docs/summary.json`. Paste the console output into `docs/findings.md`.
3. Run `python -m unittest` and make every test pass.
4. Run `python analyzer.py` from a different folder, for example by changing to the parent folder and running `python 100-days-software-day-23-files-and-standard-library/analyzer.py`. Confirm that the summary still lands in the project's `docs/` folder. Write one sentence in `docs/findings.md` on why that works.
5. Add a function `top_codes(codes: list[str], n: int) -> list[tuple[str, int]]` that uses `Counter.most_common`. Add a test.
6. Add a function `filter_level(events: list[dict[str, str]], level: str) -> list[dict[str, str]]` that uses a list comprehension, and a test.
7. Run `python -m mypy analyzer.py tests/test_analyzer.py` and fix every issue.
8. Write a section in `docs/findings.md` titled "Safety notes" that explains why `python_version` passes a list and does not use `shell=True`, in at least three sentences.
9. Commit with the message "Add log analyzer using standard library modules" and push to GitHub, including `docs/summary.json` as sample output.

Done when:

- `python -m unittest` runs at least eleven tests and passes
- `python -m mypy` reports no issues
- `docs/summary.json` shows the level counts, the hours, the error codes and the Python version
- `docs/findings.md` contains the console output, the working-directory note, and the safety notes
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, `data/events.csv`, the tests, `docs/summary.json`, `docs/findings.md`, `README.md` (with run commands), and `.gitignore`.

The analyzer uses only modules from the standard library, which is itself a result worth recording. A reviewer should be able to clone the repo, run the script and the tests, and see the same summary without installing anything.

---

## Check your understanding

1. A script reads `data.csv` with a relative path and works on the developer's laptop but fails in a scheduled job. Explain the likely cause, and rewrite the path so it does not depend on the working directory.
2. A developer writes `path.read_text()` without an encoding on Windows and gets garbled accents in some files. Explain why, and state the one change that prevents it.
3. A CSV field `"Smith, J."` is read with `line.split(",")`. Explain the result, and name the module that handles this correctly.
4. Two timestamps, one naive and one aware, are subtracted. Predict the error, and state the rule for storing timestamps that avoids it.
5. A regex `r"ERR-(\d+)"` is applied to `"XERR-123"` and returns `123`. Explain why, and write a pattern that rejects it.
6. A developer runs `subprocess.run(f"git log {branch}", shell=True)` where `branch` comes from user input. Name the risk, and rewrite the call safely.

### Answers

1. The scheduled job starts with a different working directory, so the relative path resolves to a location where the file does not exist. Rewrite the path as `Path(__file__).resolve().parent / "data.csv"` or the equivalent for the project's data folder.
2. Without an explicit encoding, `open` uses the platform default, which on Windows is often not UTF-8, so bytes are decoded with the wrong codec and accents are garbled. Pass `encoding="utf-8"` to `read_text` and to every other file call.
3. The split breaks the quoted field into two parts, `"Smith` and ` J."`, so the record has the wrong number of columns. The `csv` module handles quoting and returns the field intact.
4. Python raises `TypeError` because a naive and an aware datetime cannot be compared or subtracted. The rule is to store timestamps as aware UTC values and convert them to local time only for display.
5. The pattern has no word boundary, so the `ERR` in `XERR` matches, and the regex finds the digits after it. A pattern with `\b` before the `E`, such as `r"\bERR-(\d+)\b"`, rejects it because no word boundary exists between `X` and `E`.
6. With `shell=True`, the shell parses the string, so a value such as `main; rm -rf ~` runs as a second command. The safe version passes a list, such as `subprocess.run(["git", "log", branch], check=True)`, so the branch name is always a single argument.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "The Python Standard Library" index, at docs.python.org/3/library/index.html
- Python documentation, modules `pathlib`, `json`, `csv`, `datetime`, `collections`, `re` and `subprocess`, at docs.python.org/3/library/
- Python documentation, "Regular Expression HOWTO", at docs.python.org/3/howto/regex.html
- RFC 8259, "The JavaScript Object Notation (JSON) Data Interchange Format", at datatracker.ietf.org/doc/html/rfc8259
- [Wikipedia: Comma-separated values](https://en.wikipedia.org/wiki/Comma-separated_values)
- [Wikipedia: ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)
- [Wikipedia: Regular expression](https://en.wikipedia.org/wiki/Regular_expression)

*Tutorials*

- GeeksforGeeks, "Python pathlib module", "Python JSON" and "Python Regular Expression" tutorials, at geeksforgeeks.org
- W3Schools, "Python File Handling", "Python JSON" and "Python RegEx" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on files and on regular expressions.

*Practice*

- regex101 at regex101.com, with the Python flavor selected. Paste the `ERR-(\d{3})` pattern and the sample lines, and watch which parts match and which do not.

---

## Tomorrow

Day 24 turns a script into a project that other people can install, test and maintain. You will create a virtual environment, install dependencies with `pip`, lay out a project with a `pyproject.toml`, format code with Ruff, check types with mypy, and read the packaging concepts that connect a repo to a published package. The analyzer you built today is the kind of tool that gains from those steps.

**Next:** [Day 24: Professional Python](Software-Day-24-Professional-Python.md)

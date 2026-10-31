# Day 12: Strings and Unicode

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** beginner. You know variables, types and operators from Day 9 to Day 11. This is for anyone who has sliced a name and broken an accent, or who has seen a file read as `Ã©` where `é` belongs.

**By the end of this day, you will have:**

- A working model of a Python `str` as an immutable sequence of Unicode code points
- The ability to index and slice strings correctly, including with negative indices and steps
- A practical toolkit of string methods, and the habit of using their return values
- Fluency with f-strings and format specifications for alignment, precision and grouping
- A clear explanation of the difference between characters, code points and bytes, and the correct use of `encode` and `decode`
- A small text-tools repo with tests that cover accents, emoji and file encodings

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2. On Windows PowerShell, check that the console shows accents correctly before the exercises (run `python -c "print('café')"`). If accents come out as question marks or odd symbols, set `$env:PYTHONUTF8 = "1"` for the session, which switches Python to UTF-8 mode

**Prerequisites:**

- Day 9: Python Runtime and Syntax (scripts, `python -m`, expressions)
- Day 10: Objects, Variables and Types (immutability, method return values)
- Day 11: Numbers and Operators (integer and float conversions)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Indexing](#indexing)
  - [Slicing](#slicing)
  - [Methods](#methods)
  - [Formatting](#formatting)
  - [F-strings](#f-strings)
  - [Encoding and decoding](#encoding-and-decoding)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a text tools lab](#project-a-text-tools-lab)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Text is the most common data a program handles. Names, emails, product descriptions, log lines, JSON payloads and HTML pages are all strings, and most of the bugs that users see in those strings come from one of three places. A slice that is off by one drops the last letter of a name. A method call whose result is discarded leaves the text unchanged. An encoding mismatch turns an accented letter into two strange characters. Each of these looks small, and each has happened in production systems that serve real customers.

This day builds the model that prevents those bugs. You will learn that a Python string is a sequence of Unicode code points, that indexing and slicing work on those code points, and that methods return new strings instead of changing the old one. You will learn the difference between a string and the bytes that represent it on disk or on a network, and you will write the encoding and decoding step explicitly. By the end you will be able to read and write text files without guessing.

The official references are the section "Text Sequence Type (str)" in the Python standard library documentation at docs.python.org/3/library/stdtypes.html, and the "How to" guide "Unicode HOWTO" at docs.python.org/3/howto/unicode.html. The Unicode HOWTO is written for programmers, and it is worth reading in full once.

---

## Glossary

- **Code point:** a number that Unicode assigns to one character, written as `U+0041` for `A` or `U+00E9` for `é`.
- **Unicode:** the standard that assigns code points to characters from almost every writing system, plus symbols and emoji.
- **Character:** in Python, one element of a `str`, which is one code point. A user-perceived character can be more than one code point.
- **Grapheme cluster:** what a reader sees as one character, which may be several code points, such as a letter with an accent written as two code points.
- **Encoding:** a rule that maps code points to bytes, such as UTF-8 or ASCII.
- **Decoding:** the reverse of encoding. It maps bytes back to code points using a named encoding.
- **UTF-8:** the dominant encoding on the web and in files. It uses one to four bytes per code point.
- **Byte string:** a `bytes` object, a sequence of byte values from 0 to 255, with no text meaning until decoded.
- **Index:** a position in a sequence, counted from 0 at the left, or from -1 at the right.
- **Slice:** a range of a sequence, written as `text[start:stop:step]`.
- **Format specification:** the part after the colon in an f-string or `format` call, such as `>10.2f`, which controls alignment, width and precision.
- **Normalization:** rewriting text into a standard form, such as NFC, so that equal-looking strings compare equal.

---

## Concepts

The concepts start with the mechanics of reading a string, then the methods and formatting you use to change it for display, and finish with the encoding boundary where text meets the outside world.

### Indexing

Think of a row of numbered lockers. Each locker holds exactly one item, and you open it by its number. Counting starts at zero on the left, so the first locker is 0. Python counts from the left starting at 0, and it also lets you count from the right with negative numbers, where -1 is the last locker.

From first principles, a `str` is an ordered sequence of code points. `text[i]` returns the code point at position `i` as a new string of length one, because Python does not have a separate character type. Indexing is fast, because the position is computed directly, but it does not know about grapheme clusters, so an accented letter written as two code points occupies two positions.

```python
name = "Ada"
print(name[0])    # A
print(name[-1])   # a
print(len(name))  # 3

word = "é"   # e followed by a combining acute accent
print(len(word))  # 2, though it looks like one letter
```

The last example shows the boundary. The string has two code points. Python counts code points, which can differ from the characters the reader sees on screen.

The common mistake is indexing past the end, which raises `IndexError`, or assuming that a loop over indices visits "characters" in the reader's sense. Use `for ch in text` when you want each code point, and treat grapheme-level work as a separate problem. The standard library does not split grapheme clusters, and the common third-party approach is the `regex` package with its `\X` pattern, or the `grapheme` package. In the job market, the difference matters for text editors, truncation of user names and emoji-heavy social content.

### Slicing

A slice is a window onto a sequence. Think of cutting a strip of film. You choose where the cut starts, where it stops, and how many frames you skip between kept frames. The stop position is excluded, which means `text[0:3]` keeps positions 0, 1 and 2.

From first principles, a slice `text[start:stop:step]` produces a new string. Missing values take defaults: `start` defaults to the beginning, `stop` to the end, and `step` to 1. A negative step walks backward. Slices never raise `IndexError` for out-of-range bounds. They clip to the available range and return what fits, which is useful and also a source of silent errors.

```python
title = "Python 3.14"
print(title[:6])       # Python
print(title[7:])       # 3.14
print(title[-4:])      # 3.14
print(title[::-1])     # 41.3 nohtyP
print(title[100:])     # empty string, no error
```

Because a slice creates a new string, `title[:6] = "Java"` raises `TypeError`. Strings are immutable, as Day 10 explained, so you build a new string instead.

The common mistake is using `text[:n]` to truncate a display label and cutting a word or an emoji in half. Truncating by code points is a fine default, but for user-visible labels, count characters with the grapheme approach above, or truncate at a word boundary. The job market uses this constantly in list views, tables and notifications, where a bad cut looks broken to the user.

### Methods

Methods are functions attached to an object, called with dot notation. A `str` has many methods, such as `upper`, `lower`, `strip`, `split`, `join`, `replace`, `startswith` and `find`. Think of them as tools in a workshop. Each tool takes the work piece and returns a new piece. The original piece stays on the bench unless you put the new one in its place.

From first principles, string methods never change the string they are called on. `text.upper()` returns a new uppercase string. Calling it without assigning the result is a silent no-op. `split` returns a list, `join` takes an iterable of strings and returns one string, and `find` returns the index or `-1`, while `index` raises `ValueError` when the substring is missing.

```python
raw = "  alice, bob ,carol  "
names = [part.strip() for part in raw.strip().split(",")]
print(names)                  # ['alice', 'bob', 'carol']
print(", ".join(names))       # alice, bob, carol
print("bob" in names)         # True
print(raw.find("zed"))        # -1
print("report.csv".endswith(".csv"))  # True
```

The first line strips the outer whitespace, splits on commas, and strips each part, which is a common pattern when reading user input. The `join` call is the reverse operation, and it is faster than repeated concatenation in a loop.

The common mistake is calling `replace` and expecting the original variable to change. `text.replace("a", "b")` returns a copy. Assign the result, as in `text = text.replace("a", "b")`. A second mistake is building a long string with `+=` in a loop, which is correct but slow for large inputs. Use `"".join(parts)` instead. In the job market, code review frequently catches discarded method results, and a linter with the right rule set will flag them.

### Formatting

Formatting turns values into text for people to read. Think of a printed receipt. The item names line up on the left, the prices line up on the right, and the numbers have the same number of decimal places. A format specification is the set of instructions that produces that layout.

From first principles, the format specification mini-language follows a colon inside braces, with the form `[fill][align][sign][width][grouping][.precision][type]`. Alignment is `<` for left, `>` for right and `^` for center. The width is the total number of characters. `.2f` gives two digits after the decimal point for a float. `,` inserts thousands separators. The same syntax works in `format(value, spec)`, in the `str.format` method, and in f-strings.

```python
value = 1234567.891
print(f"{value:,.2f}")        # 1,234,567.89
print(f"{'left':<10}|")       # left      |
print(f"{'right':>10}|")      # '     right|'
print(f"{'mid':^9}|")         # '   mid   |'
print("{:08.3f}".format(3.14159))  # 0003.142
```

The `str.format` method is older and still common in older codebases. Both forms are correct. For new code, f-strings are the usual choice.

The common mistake is formatting a number that is a string, or formatting a decimal string as a number, and getting `ValueError`. Convert to a number first, using `float` or `Decimal` from Day 11, and then format. Another common mistake is using `%` formatting from older code without knowing its rules. In the job market, you will read `%`-style logs in legacy services, so recognize it, even if you write new code with f-strings.

### F-strings

An f-string is a string literal with the letter `f` before the quotes. Inside it, every expression between braces is evaluated and its result is inserted. Think of a form letter with blanks, where the blanks are filled by expressions rather than by typed names.

From first principles, Python parses an f-string at compile time into a sequence of literal parts and expression parts. The expressions run in the scope where the f-string appears, so you can use local variables, attribute access and function calls inside them. The `=` specifier, such as `f"{total=}"`, prints both the name and the value, which is useful for debugging. The `!r` conversion applies `repr`, and `!s` applies `str`.

```python
user = "Ada"
count = 3
print(f"{user} has {count} messages")
print(f"{count * 2=}")          # count * 2=6
print(f"{user!r}")              # 'Ada'
print(f"{'nested ' + user.upper()}")  # nested ADA
```

Inside an f-string, you may not reuse the quote character that encloses it before Python 3.12. Python 3.12 relaxed that rule, and the examples here use the newer behavior. Target 3.12 or newer to write them.

The common mistake is building SQL, HTML or shell commands with f-strings. An f-string inserts whatever the variable contains, so a user-supplied name can change the meaning of the query. This is the classic injection bug. Use the parameter mechanism of the database driver, the escaping function of the template engine, or `subprocess` with a list of arguments. Day 23 covers `subprocess`. Security reviewers in the job market flag f-strings in query builders immediately.

### Encoding and decoding

A string is text, and a file or a network connection carries bytes. Think of a translation step. The text is in one language, and the bytes are a transcript in a code. To send text across a boundary, you encode it into bytes with an agreed code. To read it back, you decode the bytes with the same code. If the codes do not match, the message comes out garbled.

From first principles, Unicode gives each character a code point, and UTF-8 gives each code point a sequence of one to four bytes. ASCII characters, from 0 to 127, use one byte, so most English text in UTF-8 is the same size as before. Accented Latin letters use two bytes, most CJK characters use three, and emoji use four. `str.encode("utf-8")` produces a `bytes` object, and `bytes.decode("utf-8")` produces a `str`. Python 3 uses UTF-8 as the default for source files, and it is the default for many file operations on modern systems, but the reliable habit is to name the encoding explicitly every time.

```python
text = "café"
data = text.encode("utf-8")
print(len(text), len(data))     # 4 5
print(data)                     # b'caf\xc3\xa9'
print(data.decode("utf-8"))     # café
print("café".encode("latin-1"))  # b'caf\xe9'
```

The byte string `b'caf\xc3\xa9'` shows that `é` takes two bytes in UTF-8. Latin-1 uses one byte for `é`, which is why the same letter is different on disk depending on the encoding.

The common mistake is opening a file without naming its encoding. `open(path)` uses the locale's encoding when you do not pass one, and on Windows that is often not UTF-8. Write `open(path, encoding="utf-8")` every time. The second mistake is decoding with the wrong codec, which raises `UnicodeDecodeError` or, worse, produces mojibake such as `cafÃ©` when UTF-8 bytes are read as Latin-1. In the job market, encoding bugs appear when data moves between systems, such as an export from a spreadsheet, and the fix is always to know which encoding produced the bytes and to state it in code.

---

## Hands-on exercises

Work in a folder called `strings-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices indexing and slicing.

1. Create `slices.py`. Store the string `"Python 3.14"` and print its first character, its last character, the first six characters, and everything after the space.
2. Print the string reversed using a slice with a negative step.
3. Print `title[100:]` and confirm that it is an empty string and not an error. Write one sentence on why this behavior can hide bugs.

Done when every printed value matches what you predicted before running the file.

Exercise 2 practices methods and return values.

1. Create `methods.py`. Take the input `"  alice, bob ,carol  "`, and produce a list of clean names using `strip`, `split` and a list comprehension.
2. Join the list back into a string with `", "`.
3. Call `.upper()` on a name without assigning the result, print the name, and explain the output.

Done when the list of names prints as `['alice', 'bob', 'carol']` and your explanation names the immutability rule.

Exercise 3 practices f-strings and formatting.

1. Create `receipt.py`. Define a list of three tuples, each with an item name, a price as a float, and a quantity.
2. Print a header with left-aligned item and right-aligned numeric columns, using format specifications. Print one line per item with the total.
3. Print the grand total with a thousands separator, for example `f"{total:,.2f}"`.

Done when the columns line up in the terminal and the totals have two decimal places.

Exercise 4 demonstrates encoding and decoding.

1. Create `encode_check.py`. For each of `"cafe"`, `"café"`, `"naïve"`, `"👍🏽"` and `"é"`, print its length in code points and the length of its UTF-8 bytes.
2. Write the string `"café"` to a file called `sample.txt` with `encoding="utf-8"`. Read it back with the same encoding and confirm the text is equal.
3. Read the file back with `encoding="latin-1"` and record what you see. Write one sentence on why the output is wrong.

Done when the lengths you printed match the byte counts you predicted, and the round trip is exact.

Exercise 5 uses a grapheme-aware count.

1. Install the third-party package `regex` with `python -m pip install regex`. This is free and open source.
2. Create `graphemes.py`. Use `regex.findall(r"\X", text)` to count the grapheme clusters in `"é"`, `"👍🏽"` and `"café"`.
3. Compare the grapheme counts with the code point counts from Exercise 4, and write one sentence on when you would use each.

Done when the grapheme counts are 1, 1 and 4 for the three strings, respectively, and you can explain why `"👍🏽"` is one grapheme.

---

## Project: a text tools lab

The project is a small text-processing library with tests. It normalizes Unicode, builds URL slugs from titles with accents, counts words, and reports the difference between characters and bytes. The code is short. The discipline is in naming every encoding and testing the boundary cases. The evidence you commit is the passing test suite and a findings document with examples you ran.

Suggested repo name: `100-days-software-day-12-strings-and-unicode`.

Repo layout:

```
100-days-software-day-12-strings-and-unicode/
├── .gitignore
├── README.md
├── text_tools.py
├── encoding_report.py
├── formatting_demo.py
├── tests/
│   ├── __init__.py
│   └── test_text_tools.py
└── docs/
    └── findings.md
```

Starter code for `text_tools.py`:

```python
import re
import unicodedata


def normalize(text: str) -> str:
    return unicodedata.normalize("NFC", text)


def slugify(title: str) -> str:
    folded = unicodedata.normalize("NFKD", title)
    ascii_form = folded.encode("ascii", "ignore").decode("ascii")
    lowered = ascii_form.lower()
    return re.sub(r"[^a-z0-9]+", "-", lowered).strip("-")


def word_count(text: str) -> int:
    return len(text.split())


def first_chars(text: str, n: int) -> str:
    return text[:n]


def last_chars(text: str, n: int) -> str:
    return text[-n:] if n > 0 else ""


def read_text(path: str) -> str:
    with open(path, encoding="utf-8") as handle:
        return handle.read()


def write_text(path: str, text: str) -> None:
    with open(path, "w", encoding="utf-8") as handle:
        handle.write(text)
```

Starter code for `encoding_report.py`:

```python
def report(text: str) -> dict[str, int]:
    return {"chars": len(text), "bytes": len(text.encode("utf-8"))}


if __name__ == "__main__":
    for sample in ["cafe", "café", "naïve", "👍🏽", "é"]:
        result = report(sample)
        print(f"{sample!r:>12}  chars={result['chars']}  bytes={result['bytes']}")
```

Starter code for `formatting_demo.py`:

```python
items = [("Tea", 2.5, 3), ("Cake", 4.25, 1), ("Coffee", 3.0, 2)]

print(f"{'Item':<10}{'Price':>8}{'Qty':>5}{'Total':>10}")
grand_total = 0.0
for name, price, qty in items:
    total = price * qty
    grand_total += total
    print(f"{name:<10}{price:>8.2f}{qty:>5}{total:>10.2f}")
print(f"{'Grand total':<23}{grand_total:>10,.2f}")
```

Starter code for `tests/test_text_tools.py`:

```python
import os
import tempfile
import unittest

from encoding_report import report
from text_tools import first_chars, last_chars, normalize, read_text, slugify, word_count, write_text


class TextToolsTests(unittest.TestCase):
    def test_normalize_combines_accent(self):
        decomposed = "é"
        self.assertEqual(normalize(decomposed), "é")
        self.assertEqual(len(normalize(decomposed)), 1)

    def test_slugify_removes_accents(self):
        self.assertEqual(slugify("Café Déjà Vu!"), "cafe-deja-vu")

    def test_word_count_ignores_whitespace(self):
        self.assertEqual(word_count("  one two\nthree "), 3)

    def test_first_and_last_chars(self):
        self.assertEqual(first_chars("hello", 3), "hel")
        self.assertEqual(last_chars("hello", 2), "lo")
        self.assertEqual(last_chars("hello", 0), "")

    def test_report_counts_bytes(self):
        self.assertEqual(report("café"), {"chars": 4, "bytes": 5})
        self.assertEqual(report("é"), {"chars": 2, "bytes": 3})

    def test_file_round_trip(self):
        with tempfile.TemporaryDirectory() as folder:
            path = os.path.join(folder, "sample.txt")
            write_text(path, "naïve café")
            self.assertEqual(read_text(path), "naïve café")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python encoding_report.py`. Paste the output into `docs/findings.md`. Explain in one sentence why `"café"` has four characters and five bytes.
3. Run `python formatting_demo.py`. Paste the receipt into `docs/findings.md`, and add one sentence on which format specification controls the alignment of the price column.
4. Run `python -m unittest` and make all tests pass. Add one test for `slugify("  Hello,   World!  ")`, which should return `"hello-world"`.
5. Write a short function `truncate_words(text: str, limit: int) -> str` in `text_tools.py` that keeps whole words up to `limit` characters. Add a test for it. Explain in `docs/findings.md` why this is safer than `text[:limit]` for display.
6. Add a section to `docs/findings.md` titled "Encodings in use". Name the encoding used by `read_text` and `write_text`, and explain what would happen if a file saved as Latin-1 were read with that encoding.
7. Commit with the message "Add text tools lab with encoding tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least eight tests and passes
- `python encoding_report.py` shows the byte counts for all five samples
- `docs/findings.md` contains the outputs, the explanations, and the "Encodings in use" section
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: the code, the tests, `docs/findings.md`, `README.md` (with the run commands), and `.gitignore`.

The project keeps the code small so that the encoding and normalization decisions are visible. A reviewer who reads `docs/findings.md` should see that every file operation names its encoding, and that every claim about bytes was checked with a command.

---

## Check your understanding

1. A colleague writes `name[:10]` to truncate a display label, and a Vietnamese name with combining marks is cut in the middle of a letter. Explain why that happens, and name one approach that avoids it.
2. A function receives `"café"` from one system and `b"caf\xc3\xa9"` from another, and they are not equal in a comparison. Explain why, and show the one call that makes them comparable.
3. A student writes `title.replace("draft", "final")` and then prints `title`, which still says `draft`. Explain what happened and write the correct line.
4. Why is `"".join(parts)` preferred over repeated `+=` for building a large report? Name the reason in terms of how strings are stored.
5. A script opens a CSV exported from a spreadsheet with `open(path)` and reads `cafÃ©` where the file contains `café`. Explain the likely cause and the fix.
6. Why is an f-string a security risk when used to build a SQL query, and what should you use instead?

### Answers

1. `name[:10]` counts code points, and a combining mark is a separate code point, so the cut can fall between a base letter and its mark. Use a grapheme-aware library such as the `regex` package with `\X`, or truncate at a word boundary with `truncate_words`.
2. One value is text and the other is bytes, so they can never be equal. The bytes must be decoded with the correct encoding first, as in `b"caf\xc3\xa9".decode("utf-8") == "café"`.
3. Strings are immutable, so `replace` returns a new string and leaves `title` unchanged. The correct line is `title = title.replace("draft", "final")`.
4. Strings are immutable, so each `+=` creates a new string and copies the old contents, which makes the loop slow for large inputs. `join` computes the total length once and builds the result in one pass.
5. The file was saved as UTF-8, and the reader used a single-byte or Latin-1 style decoding, so each byte of the two-byte UTF-8 sequence for `é` became a separate character. The fix is to read with `encoding="utf-8"`, or to ask the exporter for the encoding it used and read with that.
6. An f-string inserts the variable's contents directly into the query text, so a value such as `' OR '1'='1` can change the query's meaning. Use the database driver's parameter placeholders, which send the values separately from the query text.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Unicode HOWTO", at docs.python.org/3/howto/unicode.html
- Python documentation, "Text Sequence Type (str)" section of "Built-in Types", at docs.python.org/3/library/stdtypes.html
- Python documentation, "Format Specification Mini-Language", at docs.python.org/3/library/string.html#format-specification-mini-language
- [Wikipedia: Unicode](https://en.wikipedia.org/wiki/Unicode)
- [Wikipedia: UTF-8](https://en.wikipedia.org/wiki/UTF-8)
- [Wikipedia: String (computer science)](https://en.wikipedia.org/wiki/String_(computer_science))

*Tutorials*

- GeeksforGeeks, "Python String Methods" and "Python f-strings" tutorials, at geeksforgeeks.org
- W3Schools, "Python Strings" and "Python String Formatting" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segments on strings and on working with files.

*Practice*

- Exercism, the Python track at exercism.org/tracks/python, where the exercises on strings and text practice slicing and methods with automated tests.

---

## Tomorrow

Day 13 moves from single strings to sequences of many items. Lists and tuples store ordered collections, and you will learn how to change them, copy them safely, unpack them into names, and build them with comprehensions. The mutability rule from Day 10 matters most here, because copying a list does not always copy everything inside it.

**Next:** [Day 13: Lists and Tuples](Software-Day-13-Lists-and-Tuples.md)

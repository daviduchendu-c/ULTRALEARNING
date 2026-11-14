# Day 19: OOP I

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You have written functions and organized them into modules on Day 16 to Day 18. This is for anyone who has passed the same three values through five functions and wished those values had a home of their own.

**By the end of this day, you will have:**

- A working model of a class as a blueprint and an instance as one object built from it
- The ability to store data in instance attributes and shared data in class attributes, and to tell them apart
- Fluency with instance methods, and an accurate understanding of what `self` is and what Python passes to it
- The ability to write an `__init__` method that sets up an instance and validates its input
- Correct use of `@classmethod` and `@staticmethod`, and a clear rule for when each one belongs in a design
- A library repo with classes, tests and a findings document that explains the difference between the three kinds of methods

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 10: Objects, Variables and Types (identity, references, mutability)
- Day 13 and Day 14: Lists, tuples, dictionaries and sets (the containers a class will hold)
- Day 16 to Day 18: Functions, scope and modules

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Classes](#classes)
  - [Instances](#instances)
  - [Attributes](#attributes)
  - [Methods](#methods)
  - [Init](#init)
  - [Class and static methods](#class-and-static-methods)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a library catalog](#project-a-library-catalog)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most programs model things. A library has books, a bank has accounts, a game has characters, and a service has requests. When the data about one thing and the functions that act on it are kept apart, the code becomes a web of parameters that must be passed everywhere, and a change to one field means a change in many places. Object-oriented programming groups the data and the behavior of one kind of thing into a class. The class is a blueprint, each object built from it is an instance, and the methods describe what the object can do.

This day introduces that model in Python, starting with the class statement and ending with the three kinds of methods. Python's version of object orientation is simpler than in some languages. There are no access modifiers, and every attribute is visible. The discipline comes from naming, validation and the choice of what belongs to an instance and what belongs to the class. Those choices are what reviewers judge when they read a class.

The official reference is the Python tutorial section "Classes" at docs.python.org/3/tutorial/classes.html, which is readable and covers this day's material. The language reference section "Class definitions" at docs.python.org/3/reference/compound_stmts.html#class-definitions is the exact specification.

---

## Glossary

- **Class:** a blueprint that defines the attributes and methods of a kind of object.
- **Instance:** one object created from a class.
- **Attribute:** a named value stored on an object or on a class.
- **Instance attribute:** an attribute stored on one instance, usually set in `__init__`.
- **Class attribute:** an attribute stored on the class and shared by all its instances.
- **Method:** a function defined inside a class, which receives the instance as its first argument.
- **self:** the conventional name of the first parameter of an instance method. It refers to the instance the method was called on.
- **cls:** the conventional name of the first parameter of a class method. It refers to the class.
- **`__init__`:** the method Python calls after creating an instance, used to set its initial attributes.
- **Constructor:** the operation that creates and initializes an object. In Python, the call to the class does this, and `__init__` sets up the new object.
- **@classmethod:** a decorator that makes a method receive the class as its first argument.
- **@staticmethod:** a decorator that makes a method receive no automatic first argument.
- **Decorator:** a function that takes a function and returns a changed version, written with `@name` above a `def`.
- **Invariant:** a condition that must hold for an object at all times, such as a book's year being in a valid range.

---

## Concepts

The concepts build the picture from the blueprint outward: the class, the object, the data, the behavior, the setup, and the methods that do not belong to one object.

### Classes

A class is a blueprint that describes a house without being one. Think of a cookie cutter. The cutter defines the shape of every cookie that comes out of it, but no cookie is the cutter, and you can make as many cookies as you like. A class defines the shape of its instances, and it does not hold the data for any one of them.

From first principles, the `class` statement runs when the module is loaded. It creates a class object and binds the class name to it, just as `def` binds a function name. The body of the class contains `def` statements for methods and assignments for class attributes. Creating an instance is done by calling the class, as in `Book("Dune", "Frank Herbert", 1965)`. Python creates a new object whose type is the class, and then runs `__init__` on it.

```python
class Book:
    pass


a = Book()
b = Book()
print(type(a) is Book)   # True
print(a is b)            # False, two separate objects
```

The class has no data and no behavior yet, but the two instances are already distinct objects, which is the main idea of this section.

The common mistake is treating a class as if it were the same as a module, and putting state in the class body that every instance then shares by accident. Attributes assigned in the class body are shared, while attributes assigned on `self` are per instance. The distinction is explained below. In the job market, reviewers look for this confusion when a class holds a list or dictionary at class level, because every instance then appends to the same object.

### Instances

An instance is one object built from a class. Think of the cookies again. Each cookie has the same shape, but each has its own crumbs, its own topping and its own position on the tray. Changing one cookie does not change the others, and the same is true of instances.

From first principles, each instance has its own namespace, which is a dictionary of attributes. The instance knows its class through `type(obj)` and `obj.__class__`. Two instances are equal only if the class defines an equality rule, which is a topic for Day 21. By default, `==` on two instances compares identity, which is the same rule you saw for plain objects on Day 10.

```python
class Counter:
    def __init__(self) -> None:
        self.value = 0


first = Counter()
second = Counter()
first.value += 5
print(first.value, second.value)   # 5 0
```

The two instances started from the same blueprint and were changed separately. This is the reason classes are useful for holding state that differs between objects, such as the balance of two bank accounts.

The common mistake is assuming that because two instances look the same when printed, they share state. Default printing shows the class name and an address, so two instances appear different even when their attributes match. Check attributes, not printouts. A second mistake is creating a new instance when you meant to pass the existing one, which makes a change disappear. In the job market, bugs of this kind show up in code that copies objects by accident.

### Attributes

Attributes are the named values that belong to an object. Think of a form attached to a file. The form has labeled fields, and each file has its own filled-in form. A class attribute is a field printed on a template that every form shares, such as the name of the office. An instance attribute is a field that each person fills in.

From first principles, Python looks up `obj.name` by first checking the instance's own namespace, then the class and its base classes. Assigning `obj.name = value` creates or updates the instance attribute, and it never changes the class attribute of the same name. That rule explains a common surprise: reading a class attribute through an instance works, but writing to it through an instance creates a new instance attribute that hides the class one.

```python
class Book:
    min_year = 1450              # class attribute, shared

    def __init__(self, title: str) -> None:
        self.title = title       # instance attribute, one per book


a = Book("Dune")
b = Book("Emma")
print(Book.min_year, a.min_year)   # 1450 1450, read from the class
a.min_year = 1900                  # creates an instance attribute on a
print(Book.min_year, a.min_year, b.min_year)  # 1450 1900 1450
```

After the assignment, `a` has its own `min_year`, and the class and the other instance are unchanged. This is the shadowing rule from Day 16, applied to attributes.

The common mistake is storing a mutable object, such as a list, as a class attribute, and then appending to it through an instance. Every instance then sees the change, because they all share the one list. Create mutable containers in `__init__` so that each instance has its own. The job market hits this bug in logging, caching and registry code, where a class-level dictionary grows across the whole program.

### Methods

A method is a function defined in a class. Think of a button on a machine. The button belongs to the machine and affects that machine's state. When you press it, the machine knows which machine it is. In Python, the instance is passed automatically as the first argument, named `self` by convention.

From first principles, `book.borrow("Ada")` is translated by Python into `Book.borrow(book, "Ada")`. The attribute lookup finds the function in the class, and it binds the instance to the first parameter. This is why every instance method must name at least one parameter for the instance, even if the method does not use it. The method can read and change instance attributes through `self`.

```python
class Book:
    def __init__(self, title: str) -> None:
        self.title = title
        self.borrowed_by: str | None = None

    def borrow(self, person: str) -> None:
        if self.borrowed_by is not None:
            raise RuntimeError(f"{self.title} is already borrowed")
        self.borrowed_by = person


dune = Book("Dune")
dune.borrow("Ada")
print(dune.borrowed_by)   # Ada
```

The method checks the state before changing it, which keeps the object valid. A method that breaks a rule the object promises is a bug, even if the code runs.

The common mistake is forgetting `self` in a method's parameter list, which makes Python pass the instance as the first explicit argument and produce a `TypeError` about the number of arguments. A second mistake is writing `self.borrowed_by = person` in the wrong place, such as in a class body, which creates a class attribute by accident. In the job market, method design is the main thing that reviewers read, because a method is where a class enforces its rules.

### Init

`__init__` is the method that sets up a new instance. Think of a new employee's first day, when the company hands over a badge, a desk and a set of accounts. `__init__` gives the new object its initial attributes and checks that they make sense.

From first principles, `Book("Dune", "Frank Herbert", 1965)` first creates an empty object of the class, and then calls `__init__` with that object as `self` and the arguments as the rest. `__init__` returns `None`, and its job is to set attributes. If it raises an exception, the instance is not returned to the caller. The method is a good place for validation, because an object that fails its own rules should never exist.

```python
class Book:
    min_year = 1450

    def __init__(self, title: str, year: int) -> None:
        if not self.min_year <= year <= 2026:
            raise ValueError(f"year out of range: {year}")
        self.title = title
        self.year = year


book = Book("Dune", 1965)
print(book.title, book.year)   # Dune 1965

try:
    Book("Old", 1000)
except ValueError as err:
    print("error:", err)
```

The validation runs before the attributes are set, so an invalid book never exists. That is the invariant idea from the glossary, enforced at the only place an instance is created.

The common mistake is returning a value from `__init__`, which raises a `TypeError`, or doing heavy work there, such as reading a large file. Keep `__init__` quick and deterministic, and put long operations in methods or in classmethods that build the instance from a source. In the job market, constructors that validate and fail early are a sign of a careful design, and reviewers ask what an invalid object would look like.

### Class and static methods

Some functions belong to a class but do not need one particular instance. Think of a library's front desk. The desk handles requests for the whole building, such as registering a new member, which is a class method. It also has a rule that any clerk can apply without knowing anything about one member, such as checking that an ISBN has the right length, which is a static method. Instance methods act on one member's record.

From first principles, `@classmethod` changes the first argument from the instance to the class, so the method receives `cls` and can call `cls(...)` to create an instance of the actual class, including subclasses. This is useful for alternative constructors. `@staticmethod` removes the automatic first argument entirely, so the method behaves like a plain function stored in the class namespace. It is useful when the logic is related to the class but does not read or change instance or class state.

```python
class Book:
    min_year = 1450

    def __init__(self, title: str, year: int) -> None:
        self.title = title
        self.year = year

    @classmethod
    def from_line(cls, line: str) -> "Book":
        title, year_text = (part.strip() for part in line.split("|"))
        return cls(title, int(year_text))

    @classmethod
    def is_valid_year(cls, year: int) -> bool:
        return cls.min_year <= year <= 2026

    @staticmethod
    def normalize_title(title: str) -> str:
        return " ".join(title.split())


book = Book.from_line("Dune | 1965")
print(book.title, book.year)                   # Dune 1965
print(Book.is_valid_year(1000))                # False
print(Book.normalize_title("  the   hobbit "))  # the hobbit
```

The `from_line` method uses `cls`, so a subclass of `Book` that calls `from_line` gets an instance of the subclass. The static method does not touch the class at all, so it could live in a module, but keeping it on the class groups related behavior together.

The common mistake is making every method an instance method, even when it does not use `self`, or making a static method that reads class state, which should be a class method. Ask what the method needs. If it needs one object, use an instance method. If it needs the class, use a class method. If it needs neither, a static method or a module function is correct. In the job market, reviewers ask this question about every method that does not use `self`, because the answer shows whether the design is clear.

---

## Hands-on exercises

Work in a folder called `oop-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices classes and instances.

1. Create `instances.py`. Define an empty class `Note`. Create two instances, assign each a different `text` attribute, and print both.
2. Print `type(first) is Note` and `first is second`.
3. Write one sentence on why two instances can have different data even though they came from one class.

Done when the two notes print their own text and your sentence names the blueprint idea.

Exercise 2 practices class and instance attributes.

1. Create `attributes.py`. Define a class `Book` with a class attribute `min_year = 1450` and an `__init__` that sets `title` on the instance.
2. Read `min_year` from the class and from an instance.
3. Assign `min_year` on one instance, then print the class value and both instance values. Write one sentence explaining the result.

Done when the other instance and the class still show 1450 after the assignment.

Exercise 3 practices methods.

1. Create `methods.py`. Add a `borrow(self, person)` method to `Book` that raises `RuntimeError` if the book is already borrowed, and a `give_back(self)` method that clears the borrower.
2. Borrow a book, try to borrow it again inside a `try` block, then give it back and borrow it again.
3. Record the outputs and check that each state change matches the rule.

Done when the second borrow raises the error and the final borrow succeeds after the return.

Exercise 4 practices `__init__` validation.

1. Create `validation.py`. Add a check in `Book.__init__` that rejects a year before 1450 or after 2026 with `ValueError`.
2. Create valid and invalid books, and record each result.
3. Write one sentence on why validating in `__init__` is better than validating in each method that uses the year.

Done when the invalid years raise and no invalid instance is created.

Exercise 5 practices class and static methods.

1. Create `factories.py`. Add a `@classmethod from_line(cls, line)` that parses `"Title | Year"`, and a `@staticmethod normalize_title(title)`.
2. Define a subclass `Ebook(Book)` with no body, and call `Ebook.from_line("Dune | 1965")`. Print its type name.
3. Run `python -m mypy factories.py` and fix any reported issue.

Done when the subclass call returns an `Ebook` and mypy is clean.

---

## Project: a library catalog

The project is a small library catalog with a `Book` class and a `Library` class. The book validates its own data, borrows and returns itself, and can be built from a text line through a class method. The library holds books and answers questions about them. The tests check each rule. The evidence is the passing test suite and a findings document that explains which methods are instance, class and static methods, and why.

Suggested repo name: `100-days-software-day-19-oop-i`.

Repo layout:

```
100-days-software-day-19-oop-i/
├── .gitignore
├── README.md
├── library.py
├── tests/
│   ├── __init__.py
│   └── test_library.py
└── docs/
    └── findings.md
```

Starter code for `library.py`:

```python
from typing import Self


class Book:
    """A book with a title, an author, a year and an optional borrower."""

    min_year = 1450

    def __init__(self, title: str, author: str, year: int) -> None:
        if not self.is_valid_year(year):
            raise ValueError(f"year out of range: {year}")
        self.title = title
        self.author = author
        self.year = year
        self.borrowed_by: str | None = None

    @classmethod
    def from_line(cls, line: str) -> Self:
        title, author, year_text = (part.strip() for part in line.split("|"))
        return cls(title, author, int(year_text))

    @classmethod
    def is_valid_year(cls, year: int, latest: int = 2026) -> bool:
        return cls.min_year <= year <= latest

    @staticmethod
    def normalize_title(title: str) -> str:
        return " ".join(title.split())

    def borrow(self, person: str) -> None:
        if self.borrowed_by is not None:
            raise RuntimeError(f"{self.title} is already borrowed by {self.borrowed_by}")
        self.borrowed_by = person

    def give_back(self) -> None:
        self.borrowed_by = None


class Library:
    """A collection of books with simple queries."""

    def __init__(self) -> None:
        self.books: list[Book] = []

    def add(self, book: Book) -> None:
        self.books.append(book)

    def by_author(self, author: str) -> list[Book]:
        return [book for book in self.books if book.author == author]

    def available(self) -> list[Book]:
        return [book for book in self.books if book.borrowed_by is None]


if __name__ == "__main__":
    library = Library()
    for line in ["Dune | Frank Herbert | 1965", "Emma | Jane Austen | 1815"]:
        library.add(Book.from_line(line))
    print([book.title for book in library.by_author("Jane Austen")])
    library.books[0].borrow("Ada")
    print(len(library.available()))
    print(Book.is_valid_year(1800), Book.normalize_title("  the   hobbit "))
```

Starter code for `tests/test_library.py`:

```python
import unittest

from library import Book, Library


class BookTests(unittest.TestCase):
    def test_from_line_builds_a_book(self):
        book = Book.from_line("Dune | Frank Herbert | 1965")
        self.assertEqual(book.title, "Dune")
        self.assertEqual(book.year, 1965)
        self.assertIsNone(book.borrowed_by)

    def test_invalid_year_is_rejected(self):
        with self.assertRaises(ValueError):
            Book("Old", "Anon", 1000)

    def test_is_valid_year_on_class_and_instance(self):
        self.assertTrue(Book.is_valid_year(1800))
        self.assertFalse(Book.is_valid_year(1000))
        book = Book("Dune", "Frank Herbert", 1965)
        self.assertFalse(book.is_valid_year(1000))

    def test_class_attribute_is_shared(self):
        self.assertEqual(Book.min_year, 1450)
        first = Book("A", "B", 1900)
        second = Book("C", "D", 1900)
        self.assertEqual(first.min_year, second.min_year)

    def test_instances_keep_their_own_borrower(self):
        first = Book("A", "B", 1900)
        second = Book("C", "D", 1900)
        first.borrow("Ada")
        self.assertEqual(first.borrowed_by, "Ada")
        self.assertIsNone(second.borrowed_by)

    def test_cannot_borrow_twice(self):
        book = Book("A", "B", 1900)
        book.borrow("Ada")
        with self.assertRaises(RuntimeError):
            book.borrow("Grace")

    def test_give_back_then_borrow_again(self):
        book = Book("A", "B", 1900)
        book.borrow("Ada")
        book.give_back()
        book.borrow("Grace")
        self.assertEqual(book.borrowed_by, "Grace")

    def test_static_method_normalizes_title(self):
        self.assertEqual(Book.normalize_title("  the   hobbit "), "the hobbit")

    def test_subclass_from_line_returns_subclass(self):
        class Ebook(Book):
            pass

        ebook = Ebook.from_line("Dune | Frank Herbert | 1965")
        self.assertIsInstance(ebook, Ebook)


class LibraryTests(unittest.TestCase):
    def test_by_author_and_available(self):
        library = Library()
        library.add(Book("Dune", "Frank Herbert", 1965))
        library.add(Book("Emma", "Jane Austen", 1815))
        library.books[0].borrow("Ada")
        self.assertEqual([b.title for b in library.by_author("Jane Austen")], ["Emma"])
        self.assertEqual([b.title for b in library.available()], ["Emma"])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python library.py` and paste the output into `docs/findings.md`. Explain in one sentence why `Library.books` is a list created in `__init__`.
3. Run `python -m unittest` and make every test pass.
4. Add a `Library.find_title(self, title: str) -> Book | None` method that returns the first book with that title. Add a test that covers a found and a missing title.
5. Add a method `Book.__init__` check that the author is not empty, raising `ValueError`. Add a test.
6. Write a table in `docs/findings.md` with one row for each method in `Book` and `Library`. For each row, name whether it is an instance method, a class method or a static method, and state in a few words why.
7. Run `python -m mypy library.py tests/test_library.py` and fix every issue.
8. Commit with the message "Add library catalog classes with tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least eleven tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs, the explanation of the books list, and the method table
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: `library.py`, the tests, `docs/findings.md`, `README.md` (with the run commands), and `.gitignore`.

The project is a single file with two classes, which is a deliberate size. A reviewer should be able to see what each class protects, which methods change state, and why the alternative constructor is a class method rather than a function outside the class.

---

## Check your understanding

1. A class has `members = []` in the class body, and `add_member` appends to `self.members`. Two library branches report the same members. Explain why, and show the corrected line.
2. An instance method is written as `def borrow(person):` with no `self`. Predict the error when `book.borrow("Ada")` is called, and explain how Python passes arguments to methods.
3. `Book.min_year = 1500` is run in a test, and it changes behavior in later tests. Explain why, and suggest how to make the test safe.
4. A developer puts a database connection in `__init__` for every book. Explain two problems this causes, and say what should happen instead.
5. A static method reads `cls.min_year`. Explain why this is wrong, and say which kind of method it should be.
6. Why is a class method a better alternative constructor than a plain function that returns `Book(...)`, when subclasses exist?

### Answers

1. `members` is a class attribute, and no `__init__` assigns `self.members`. So `self.members.append(...)` finds the class-level list through the instance and appends to that one shared list, and every branch sees the same members. The corrected version assigns `self.members = []` in `__init__`, so each instance gets its own list.
2. Calling `book.borrow("Ada")` passes the instance as the first positional argument and `"Ada"` as the second, so Python raises `TypeError` because `borrow` accepts only one argument. Python always passes the instance as the first argument, so the method must name it, conventionally `self`.
3. Assignment on the class changes the class attribute, which every instance sees when it reads `min_year` without an instance attribute of its own. The test should restore the original value in a `tearDown` or with `unittest.mock.patch.object`, so later tests see the default.
4. A connection in `__init__` is opened for every book, which wastes resources and can fail in a way that prevents a book from existing. The connection should be created once by the library or a service and passed in, or opened lazily when a method needs it.
5. The method reads class state through `cls`, so it must be a class method, which receives `cls` automatically. A static method has no access to `cls`, so the code would fail with `NameError` for `cls`.
6. A class method receives the actual class that was called, so `Ebook.from_line` creates an `Ebook`. A plain function always returns the base class, so subclasses cannot use it to build their own type.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Classes", at docs.python.org/3/tutorial/classes.html
- Python language reference, "Class definitions", at docs.python.org/3/reference/compound_stmts.html#class-definitions
- Python documentation, built-in functions `classmethod` and `staticmethod`, at docs.python.org/3/library/functions.html
- [Wikipedia: Object-oriented programming](https://en.wikipedia.org/wiki/Object-oriented_programming)
- [Wikipedia: Class (computer programming)](https://en.wikipedia.org/wiki/Class_(computer_programming))
- [Wikipedia: Method (computer programming)](https://en.wikipedia.org/wiki/Method_(computer_programming))

*Tutorials*

- GeeksforGeeks, "Python Classes and Objects" and "Python Class Method vs Static Method" tutorials, at geeksforgeeks.org
- W3Schools, "Python Classes and Objects" and "Python Class Methods" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segment on object-oriented programming.

*Practice*

- Python Tutor at pythontutor.com. Step through the class attribute example and the mutable class attribute bug, and watch which object each reference points to.

---

## Tomorrow

Day 20 extends the class model in two directions. Inheritance lets one class reuse and specialize another, and composition lets a class hold other objects and delegate work to them. You will meet polymorphism, where one call works on several types, the abstract base class idea, and the `dataclass` decorator that removes the boilerplate from data-holding classes. You will also weigh the trade-offs, because inheritance is often the wrong tool.

**Next:** [Day 20: OOP II](Software-Day-20-OOP-II.md)

# Day 20: OOP II

*100 Days of Software Engineering, Module 3: Python (Deep Foundation)*

**Time:** about 3 hours
**Level:** intermediate. You have written classes with attributes, methods and class-level helpers on Day 19. This is for anyone who has started to copy a class to make a slight variant, or who has built a class that does too many jobs and cannot be tested without everything else.

**By the end of this day, you will have:**

- A working model of inheritance as an "is-a" relationship, with method overriding and `super()`
- A working model of composition as a "has-a" relationship, and the judgment to choose it when inheritance would couple unrelated behavior
- The ability to write polymorphic code, where one call works on several kinds of object
- The ability to define an abstract base class with `abc`, and to explain why it refuses direct instantiation
- Fluency with `@dataclass`, including `frozen=True`, `field(default_factory=...)` and `__post_init__` for validation
- A payments repo with tests that compare an inheritance design and a composition design, and a written tradeoff analysis

**What you need:**

- Python 3.14 (any 3.12 or newer works). Check with `python --version`.
- Git 2.40 or newer and a GitHub account from Day 8
- A code editor with Python support, such as Visual Studio Code with the Microsoft Python extension (free)
- A terminal: Git Bash, PowerShell, or Ubuntu on WSL 2
- `mypy` 1.x, installed with `python -m pip install mypy` (free, open source)

**Prerequisites:**

- Day 11: Numbers and Operators (`Decimal` for money)
- Day 19: OOP I (classes, instances, methods, class and static methods)

---

In this guide you are going to cover the following:

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Concepts](#concepts)
  - [Inheritance](#inheritance)
  - [Composition](#composition)
  - [Polymorphism](#polymorphism)
  - [ABC concepts](#abc-concepts)
  - [Dataclasses](#dataclasses)
  - [Design tradeoffs](#design-tradeoffs)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a checkout with payment methods](#project-a-checkout-with-payment-methods)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Once a program has several kinds of thing, you face a choice. Should a new kind reuse the code of an existing kind, or should it hold the existing code and call it? Object-oriented designs answer that question with two relationships: inheritance and composition. Inheritance says a card payment is a payment method. Composition says a checkout has a payment method. Both are useful, and both are easy to misuse. A deep inheritance tree becomes hard to change, and a tangle of objects that hold each other becomes hard to follow.

This day teaches both relationships, the polymorphism that makes them useful, and the tools Python provides for them: the `abc` module for abstract base classes, and the `dataclass` decorator for classes whose main job is to hold data. It ends with the design tradeoffs, because the most valuable skill in this area is knowing when a class should not inherit at all.

The official references are the Python tutorial section "Inheritance" at docs.python.org/3/tutorial/classes.html#inheritance, the documentation for the module `abc` at docs.python.org/3/library/abc.html, and the documentation for the module `dataclasses` at docs.python.org/3/library/dataclasses.html.

---

## Glossary

- **Inheritance:** a relationship in which one class, the subclass, takes the attributes and methods of another class, the base class.
- **Base class:** a class that other classes inherit from. Also called a parent or superclass.
- **Subclass:** a class that inherits from a base class. Also called a child class.
- **Override:** redefining a method in a subclass so that the subclass version runs instead of the base version.
- **super():** a function that gives access to the base class's methods from inside a subclass.
- **Composition:** a design in which a class holds references to other objects and delegates work to them.
- **Delegation:** passing a task to a held object instead of doing it yourself.
- **Polymorphism:** the ability to call the same method on objects of different types and get a result appropriate to each type.
- **Abstract base class (ABC):** a class that declares methods that subclasses must implement, and that cannot be instantiated itself.
- **Abstract method:** a method declared with `@abstractmethod` that a subclass must override.
- **Dataclass:** a class decorated with `@dataclass`, which generates the `__init__`, `__repr__` and `__eq__` methods from its annotated fields.
- **Frozen:** a dataclass whose instances cannot be changed after creation.
- **Liskov substitution principle:** the rule that a subclass must be usable anywhere its base class is expected, without surprising the caller.

---

## Concepts

The six concepts cover the two relationships, the behavior they enable, the abstract contracts and data classes that support them, and the tradeoffs that decide between them.

### Inheritance

Inheritance is an "is-a" relationship. Think of a family tree. A child inherits the family name and some traits from a parent, and also has traits of their own. A subclass takes everything from its base class and adds or changes what it needs. A savings account is an account, with the same owner and balance, plus an interest rate.

From first principles, when you write `class Savings(Account):`, Python builds the subclass with the base class in its method resolution order, which is a list that decides where attribute lookup looks first. A method found in the subclass overrides the one in the base class. A method the subclass does not define is inherited. `super()` returns a proxy that searches the rest of that order, so a subclass can call the base class version of a method it has overridden. The common pattern is to call `super().__init__()` in a subclass constructor before adding its own attributes.

```python
class Account:
    def __init__(self, owner: str) -> None:
        self.owner = owner
        self.balance = 0.0

    def describe(self) -> str:
        return f"{self.owner}: {self.balance:.2f}"


class Savings(Account):
    def __init__(self, owner: str, rate: float) -> None:
        super().__init__(owner)
        self.rate = rate

    def describe(self) -> str:
        return super().describe() + f" at {self.rate:.1%}"


plan = Savings("Ada", 0.02)
print(plan.describe())   # Ada: 0.00 at 2.0%
```

`Savings` reuses the base constructor and the base description, and it adds the rate. It never copies code from `Account`.

The common mistake is inheriting for code reuse alone, when the two classes are not really the same kind of thing. A `Stack` that inherits from `list` exposes `insert` and `sort`, which break the stack rule. The second mistake is forgetting `super().__init__()` in a subclass constructor, which leaves the base attributes unset and produces an `AttributeError` later. In the job market, reviewers push back on inheritance hierarchies deeper than two or three levels, because each level adds a place where behavior can change unexpectedly.

### Composition

Composition is a "has-a" relationship. Think of a car. The car has an engine, a set of wheels and a radio. It does not inherit from the engine. It asks the engine to turn the wheels. When the engine is replaced by a better one, the car does not change its own design.

From first principles, a composed class stores references to other objects in its attributes and calls their methods to do work. The outer class defines its own interface, which is usually small, and it delegates the details to the parts. Composition lets you change a part at runtime by assigning a different one, and it lets you test the outer class with a fake part. The cost is that the outer class must forward the calls it needs, which is more code than inheriting the methods for free.

```python
class Checkout:
    def __init__(self, payment) -> None:
        self.payment = payment          # has-a payment method

    def pay(self, amount: float) -> str:
        return self.payment.charge(amount)   # delegates the work


class FakePayment:
    def charge(self, amount: float) -> str:
        return f"fake: {amount:.2f}"


print(Checkout(FakePayment()).pay(12.5))   # fake: 12.50
```

The checkout is testable with `FakePayment`, which needs no network and no card. That is one of the main reasons teams prefer composition in service code.

The common mistake is building a class that holds many objects and delegates every call, which becomes a second copy of the interface with no added value. Compose when the parts are distinct and the outer class adds rules. Do not compose when the outer class is just a wrapper. A second mistake is letting composed objects reach back into the outer object, which creates cycles that are hard to test. In the job market, dependency injection, which is the practice of passing parts in through the constructor as this example does, is standard in backend teams.

### Polymorphism

Polymorphism means the same message works on different objects. Think of a power socket. The socket accepts any plug that fits its standard, and the device behind it can be a kettle, a lamp or a laptop. The socket does not need to know the device. It needs only the shape of the plug.

From first principles, when code calls `method.charge(amount)`, Python looks up `charge` on the object's type at runtime. Each type can have its own version, and the caller does not need to know which one runs. Python also supports duck typing: any object with a `charge` method works, even without a shared base class. An abstract base class makes the contract explicit, so a reader and a type checker can see which methods are required.

```python
class CardPayment:
    def charge(self, amount: float) -> str:
        return f"card: {amount:.2f}"


class CashPayment:
    def charge(self, amount: float) -> str:
        return f"cash: {amount:.2f}"


for method in [CardPayment(), CashPayment()]:
    print(method.charge(10))
```

The loop calls `charge` on two different types with one line of code. Adding a third payment type requires no change to the loop, which is the practical benefit.

The common mistake is checking the type before calling a method, with `isinstance` chains that grow with every new type. That defeats polymorphism, because each new type requires a change in every chain. Put the behavior in the classes and call the method. A second mistake is assuming duck typing is always safe. A mistyped method name fails only when that line runs, which is why type checkers are useful with polymorphic code. In the job market, plugin systems and payment, storage and notification modules rely on polymorphism, and reviewers ask whether new cases require edits in many places.

### ABC concepts

An abstract base class declares what subclasses must provide, and it refuses to be built on its own. Think of a job description. It lists the duties every candidate must perform, but no one is hired for the job description itself. A person is hired for the role, and the role must have every duty filled.

From first principles, the `abc` module provides `ABC` as a base class and `@abstractmethod` as a decorator. A class with at least one unimplemented abstract method cannot be instantiated, and Python raises `TypeError` when you try. A subclass that implements every abstract method becomes concrete and can be created. The module also supports `register` for virtual subclasses, which is an advanced topic. For most code, `ABC` with `@abstractmethod` is enough. It makes the contract visible to readers and to mypy.

```python
from abc import ABC, abstractmethod


class PaymentMethod(ABC):
    @abstractmethod
    def charge(self, amount: float) -> str:
        """Charge the amount and return a receipt line."""


class CashPayment(PaymentMethod):
    def charge(self, amount: float) -> str:
        return f"cash: {amount:.2f}"


try:
    PaymentMethod()
except TypeError as err:
    print("error:", err)

print(CashPayment().charge(4))   # cash: 4.00
```

The base class cannot be created, and the subclass that implements the contract can be. The error message names the missing method.

The common mistake is writing an abstract method with a body that raises `NotImplementedError` instead of using `@abstractmethod`. That works at runtime, but the error appears only when the method is called, and the class can still be instantiated. Use `@abstractmethod` so Python checks the contract at creation time. A second mistake is putting too many abstract methods on one base class, which forces every subclass to implement methods it does not need. In the job market, abstract interfaces appear in code that must support several backends, such as storage or payment providers.

### Dataclasses

A dataclass is a class whose main job is to hold data. Think of a form with fixed fields, such as name, address and date of birth. The form has a standard layout, a standard way to compare two filled-in forms, and a standard printout. The `@dataclass` decorator generates those standard methods from the annotated fields.

From first principles, `@dataclass` reads the class annotations, in order, and writes an `__init__` that takes them as parameters. It also writes `__repr__` for readable printing, and `__eq__`, which compares the fields. With `frozen=True`, it adds `__setattr__` and `__delattr__` methods that raise `FrozenInstanceError`, and it makes the instances hashable by their fields. `field(default_factory=list)` gives each instance its own list, which is the fix for the mutable default trap from Day 10. A method named `__post_init__` runs after the generated `__init__`, and it is the place to validate fields.

```python
from dataclasses import dataclass, field
from decimal import Decimal


@dataclass(frozen=True)
class Item:
    name: str
    price: Decimal

    def __post_init__(self) -> None:
        if self.price < 0:
            raise ValueError(f"price must not be negative: {self.price}")


@dataclass
class Cart:
    items: list[Item] = field(default_factory=list)


first = Item("tea", Decimal("2.50"))
print(first == Item("tea", Decimal("2.50")))   # True, equality is generated
print(first)                                   # Item(name='tea', price=Decimal('2.50'))
```

The frozen item cannot be changed after creation, which makes it safe to share between parts of a program. The cart uses `default_factory` so that each cart has its own list.

The common mistake is using a dataclass for an object with complicated behavior and many methods, where a regular class reads better. Dataclasses fit data with a few fields and simple rules. A second mistake is forgetting `frozen=True` on values that should not change, such as a price or an identifier, which lets one part of a program alter data that another part depends on. In the job market, reviewers expect small value objects to be frozen dataclasses, and they check that money values are not floats.

### Design tradeoffs

Every design choice has a cost, and the cost usually shows up later. Think of a building with a strong foundation. The foundation takes longer to pour, and it makes later floors safe to build. A shallow design is quick to start and hard to extend. A deep design is slow to start and safe to extend, if it is the right design.

From first principles, inheritance couples the subclass to the internals of the base class. Changing a base method can break subclasses in ways the author did not expect, which is called the fragile base class problem. Composition couples the outer class to the interface of the parts, which is usually more stable, and it makes behavior easy to replace. Abstract base classes make a contract explicit at a cost of an extra layer. Dataclasses reduce boilerplate, but they generate equality and representation that you must check against your needs. The Liskov substitution principle asks whether a subclass can be used wherever the base class is used without surprises. If the answer is no, the inheritance is wrong, even if the code runs.

```python
class Bird:
    def fly(self) -> str:
        return "flying"


class Penguin(Bird):
    def fly(self) -> str:
        raise NotImplementedError("penguins cannot fly")


def take_off(bird: Bird) -> str:
    return bird.fly()


print(take_off(Bird()))        # flying
# take_off(Penguin())          # raises, which breaks the substitution rule
```

The penguin is a bird in biology, but it is not a valid substitute for a bird in code that expects every bird to fly. Designing the base class around flying is the mistake, and the fix is to split the behavior into smaller contracts.

The common mistake is choosing inheritance because it looks like the natural model of the domain, without asking whether the subclass can replace the base class in every case. Ask three questions before you inherit: is the relationship stable, can the subclass be used anywhere the base is used, and does the subclass need only a little of the base's behavior. If the answers are yes, inherit. If not, compose. In the job market, this judgment is one of the strongest signals in a design review.

---

## Hands-on exercises

Work in a folder called `oop-two-lab`, using Python 3.14 or 3.12 or newer.

Exercise 1 practices inheritance and `super()`.

1. Create `inherit.py`. Define `Account` with an owner and a balance, and a `describe` method.
2. Define `Savings(Account)` with a rate. Its constructor must call `super().__init__`, and its `describe` must extend the base result.
3. Print the description of one savings account, and write one sentence on what would happen if you forgot the `super().__init__` call.

Done when the description includes both the balance and the rate.

Exercise 2 practices composition with a fake part.

1. Create `compose.py`. Define a `Checkout` class that holds a payment object and delegates `pay(amount)` to it.
2. Define `FakePayment` with the same `charge` method and use it to test `Checkout`.
3. Write one sentence on why this design is easier to test than a checkout that creates its own payment object.

Done when `Checkout(FakePayment()).pay(12.5)` returns the fake receipt.

Exercise 3 practices polymorphism.

1. Create `poly.py`. Define two payment classes with a `charge(amount)` method that return different strings.
2. Write a loop that calls `charge` on a list containing one of each.
3. Add a third class that has the method but does not inherit from anything. Add it to the same list and confirm the loop still works.

Done when all three objects respond through the same loop, and you can explain why the third works without inheritance.

Exercise 4 practices abstract base classes.

1. Create `abstract.py`. Define `PaymentMethod(ABC)` with an abstract `charge` method.
2. Try to instantiate `PaymentMethod()` inside a `try` block and record the error.
3. Create a subclass that implements `charge` and instantiate it. Then create a subclass that does not implement `charge`, and record what happens when you try to instantiate it.

Done when the base class and the incomplete subclass both raise `TypeError`, and the complete subclass works.

Exercise 5 practices dataclasses.

1. Create `records.py`. Define a frozen dataclass `Item` with `name: str` and `price: Decimal`, and a `__post_init__` check that rejects negative prices.
2. Define a dataclass `Cart` with `items: list[Item]` using `field(default_factory=list)`.
3. Create two carts, add an item to one, and print both. Then try to assign `item.price = Decimal("1")` and record the error.

Done when the second cart is still empty, and the frozen item refuses assignment.

---

## Project: a checkout with payment methods

The project is a small checkout system. Payment methods are subclasses of an abstract base class, so the checkout can take any of them. The checkout itself is a composition: it holds a payment method and a receipt, and it delegates the charge. Items are frozen dataclasses with validation. The tests check inheritance, polymorphism, the abstract contract, composition with a fake, and the dataclass rules. The evidence is a passing test suite and a findings document with the tradeoff analysis.

Suggested repo name: `100-days-software-day-20-oop-ii`.

Repo layout:

```
100-days-software-day-20-oop-ii/
├── .gitignore
├── README.md
├── payments.py
├── tests/
│   ├── __init__.py
│   └── test_payments.py
└── docs/
    └── findings.md
```

Starter code for `payments.py`:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from decimal import ROUND_HALF_UP, Decimal


@dataclass(frozen=True)
class Item:
    name: str
    price: Decimal

    def __post_init__(self) -> None:
        if self.price < 0:
            raise ValueError(f"price must not be negative: {self.price}")


class PaymentMethod(ABC):
    """Anything that can take money for an amount."""

    @abstractmethod
    def charge(self, amount: Decimal) -> str:
        """Charge the amount and return a receipt line."""

    def describe(self) -> str:
        return type(self).__name__


class CardPayment(PaymentMethod):
    def __init__(self, card_last4: str) -> None:
        self.card_last4 = card_last4

    def charge(self, amount: Decimal) -> str:
        return f"card ****{self.card_last4}: {amount:.2f}"


class CashPayment(PaymentMethod):
    def charge(self, amount: Decimal) -> str:
        return f"cash: {amount:.2f}"


@dataclass
class Receipt:
    lines: list[str] = field(default_factory=list)

    def add(self, line: str) -> None:
        self.lines.append(line)

    def text(self) -> str:
        return "\n".join(self.lines)


class Checkout:
    """Composition: the checkout holds a payment method and a receipt."""

    def __init__(self, payment: PaymentMethod, tax_rate: Decimal = Decimal("0.10")) -> None:
        self.payment = payment
        self.tax_rate = tax_rate
        self.receipt = Receipt()

    def total(self, items: list[Item]) -> Decimal:
        subtotal = sum((item.price for item in items), Decimal("0"))
        return (subtotal * (1 + self.tax_rate)).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def pay(self, items: list[Item]) -> str:
        line = self.payment.charge(self.total(items))
        self.receipt.add(line)
        return line


def charge_all(methods: list[PaymentMethod], amount: Decimal) -> list[str]:
    return [method.charge(amount) for method in methods]


if __name__ == "__main__":
    items = [Item("tea", Decimal("2.50")), Item("cake", Decimal("4.25"))]
    checkout = Checkout(CardPayment("4242"))
    print(checkout.pay(items))
    print(charge_all([CashPayment(), CardPayment("1111")], Decimal("10.00")))
```

Starter code for `tests/test_payments.py`:

```python
import unittest
from dataclasses import FrozenInstanceError
from decimal import Decimal

from payments import CardPayment, CashPayment, Checkout, Item, PaymentMethod, charge_all


class FakePayment:
    def charge(self, amount: Decimal) -> str:
        return f"fake: {amount:.2f}"


class PaymentTests(unittest.TestCase):
    def test_abstract_base_cannot_be_created(self):
        with self.assertRaises(TypeError):
            PaymentMethod()

    def test_subclasses_implement_the_contract(self):
        self.assertEqual(CashPayment().charge(Decimal("4")), "cash: 4.00")
        self.assertEqual(CardPayment("4242").charge(Decimal("4")), "card ****4242: 4.00")

    def test_polymorphic_charge_all(self):
        lines = charge_all([CashPayment(), CardPayment("1111")], Decimal("10.00"))
        self.assertEqual(lines, ["cash: 10.00", "card ****1111: 10.00"])

    def test_checkout_totals_with_rounding(self):
        items = [Item("tea", Decimal("2.50")), Item("cake", Decimal("4.25"))]
        checkout = Checkout(FakePayment())
        self.assertEqual(checkout.total(items), Decimal("7.43"))

    def test_checkout_delegates_to_payment(self):
        items = [Item("tea", Decimal("2.50"))]
        checkout = Checkout(FakePayment())
        self.assertEqual(checkout.pay(items), "fake: 2.75")
        self.assertEqual(checkout.receipt.text(), "fake: 2.75")

    def test_item_equality_and_frozen(self):
        self.assertEqual(Item("tea", Decimal("2.50")), Item("tea", Decimal("2.50")))
        item = Item("tea", Decimal("2.50"))
        with self.assertRaises(FrozenInstanceError):
            item.price = Decimal("1")

    def test_negative_price_rejected(self):
        with self.assertRaises(ValueError):
            Item("refund", Decimal("-1"))

    def test_describe_uses_class_name(self):
        self.assertEqual(CashPayment().describe(), "CashPayment")


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, create the files above, and add `__pycache__/` to `.gitignore`.
2. Run `python payments.py` and paste the output into `docs/findings.md`. Explain in one sentence why the checkout total uses `ROUND_HALF_UP`.
3. Run `python -m unittest` and make every test pass. Note that the test `test_checkout_totals_with_rounding` depends on the rounding mode, and write one sentence on what would change with the default rounding of `Decimal`.
4. Add a third payment class `GiftCardPayment(PaymentMethod)` that takes a card code and charges it. Add a test that it works in `charge_all`.
5. Add a method `Checkout.refund(self, amount: Decimal) -> str` that delegates to a `refund` method on the payment. Decide whether `refund` belongs on the abstract base class or in a separate abstract class, and write the reason in `docs/findings.md`. Implement your choice and add a test.
6. Write a tradeoff analysis in `docs/findings.md` of at least 200 words. Compare the design you built, where `Checkout` holds a payment, with a design where `CardCheckout` inherits from `Checkout` and overrides the payment method. Name the coupling each design creates, and say which one you would choose for a shop that adds new payment methods every quarter.
7. Run `python -m mypy payments.py tests/test_payments.py` and fix every issue.
8. Commit with the message "Add checkout with payment methods, dataclasses and tests" and push to GitHub.

Done when:

- `python -m unittest` runs at least ten tests and passes
- `python -m mypy` reports no issues
- `docs/findings.md` contains the outputs, the rounding note, the refund decision, and the tradeoff analysis
- `git status` is clean after the commit, and the GitHub repo shows the same files

Evidence to commit: `payments.py`, the tests, `docs/findings.md`, `README.md` (with run commands), and `.gitignore`.

The project shows both relationships in one small codebase. A reviewer should be able to point to the line where inheritance gives polymorphism, the line where composition gives testability, and the paragraph in the findings file that explains why the design does not use inheritance for the checkout.

---

## Check your understanding

1. A `Stack` class inherits from `list` so it can reuse `append` and `pop`. A user calls `stack.insert(0, item)` and breaks the stack rule. Explain which relationship was misused, and name the alternative design.
2. A subclass overrides `__init__` and forgets to call `super().__init__()`. Predict the error when a base attribute is used later, and explain why it occurs.
3. Explain polymorphism using the `charge_all` function. What change would be needed if you added a new payment type, and what change would not be needed?
4. A base class declares `charge` with a body that raises `NotImplementedError`, and not with `@abstractmethod`. A subclass forgets to implement it. Explain when the error appears, and explain why `@abstractmethod` is stricter.
5. A developer writes `@dataclass` on a class with three fields and then adds fifteen methods that change the fields. Explain why a regular class may fit better, and name the dataclass feature that the code should still use.
6. Why does composition make a checkout easier to test than inheritance does, and what would the test double look like?

### Answers

1. The relationship is "is-a" inheritance, and a stack is not a list. The alternative is composition: the stack holds a list in a private attribute and exposes only `push` and `pop`, so `insert` is not part of its interface.
2. The base attribute is never set, so using it raises `AttributeError`. The subclass constructor must call `super().__init__()` so the base class sets its own attributes before the subclass adds its own.
3. `charge_all` calls `charge` on each method without knowing its type. A new payment type needs only to implement `charge` and be added to the list. The loop does not change, and no `isinstance` check is needed.
4. Without `@abstractmethod`, the class can be instantiated, and the missing method raises `NotImplementedError` only when `charge` is called, which may be in production. With `@abstractmethod`, Python raises `TypeError` at instantiation, so the missing method is caught as soon as the object is created.
5. Many methods that change fields suggest that the object has behavior, and a regular class makes the behavior and the state easier to read and protect. The feature to keep is `__post_init__` or the field declarations, and `frozen=True` where the data should not change after creation.
6. Composition lets the test pass a `FakePayment` into `Checkout`, which needs no network and no card. Inheritance would require the test to create a subclass or to patch the base class, which couples the test to the inheritance tree. The fake is a small class with the same `charge` method, as in the project.

---

## Sources to read, watch and practice

*Read*

- Python documentation, "Classes", section "Inheritance", at docs.python.org/3/tutorial/classes.html#inheritance
- Python documentation, module `abc`, at docs.python.org/3/library/abc.html
- Python documentation, module `dataclasses`, at docs.python.org/3/library/dataclasses.html
- PEP 557, "Data Classes", at peps.python.org/pep-0557/
- [Wikipedia: Inheritance (object-oriented programming)](https://en.wikipedia.org/wiki/Inheritance_(object-oriented_programming))
- [Wikipedia: Polymorphism (computer science)](https://en.wikipedia.org/wiki/Polymorphism_(computer_science))
- [Wikipedia: Composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance)
- [Wikipedia: Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)

*Tutorials*

- GeeksforGeeks, "Python Inheritance" and "Abstract Classes in Python" tutorials, at geeksforgeeks.org
- W3Schools, "Python Inheritance" and "Python Polymorphism" pages, at w3schools.com/python/

*Watch*

- CS50, "CS50's Introduction to Programming with Python", on the CS50 YouTube channel. Watch the segment on object-oriented design, and note the examples where classes are used for data.

*Practice*

- Python Tutor at pythontutor.com. Step through the inheritance example with `super()`, and the composition example with a fake payment, and watch how the object references change.

---

## Tomorrow

Day 21 opens the Python data model, the set of special methods that let your classes behave like built-in types. You will write `__repr__` and `__str__`, define equality and hashing rules, make objects iterable and iterators, write generators with `yield`, and build context managers with `with`. The dataclass from today already gives you `__repr__` and `__eq__`, and tomorrow you will see how those methods work underneath.

**Next:** [Day 21: Python Data Model](Software-Day-21-Python-Data-Model.md)

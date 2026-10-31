# Day 33: Classes and Objects

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You have written functions that take and return values, and you have used classes from the standard library such as `std::string` and `std::vector`. This day explains how to design your own class, what a constructor and a destructor must guarantee, and how access control keeps the rules of the class intact. It is for anyone whose classes are just a bag of public variables.

**By the end of this day, you will have:**

- Written a class with constructors, including one that delegates to another, and explained when an initializer list is required
- Explained what a destructor should do, and why a class that holds standard containers often needs none
- Used `public`, `private` and `const` member functions to protect a class's invariants
- A bank-account library whose rules, such as no overdraft, are enforced inside the class, with a test executable run by CTest
- A GitHub repo with the build and test results, and a short note listing each invariant and the function that enforces it

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Days 27 to 32: the build, functions, references, pointers, and the constructor and destructor order
- Day 28: integer types, which the project uses for money in cents

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Constructors](#constructors)
- [Destructors](#destructors)
- [Access control](#access-control)
- [Encapsulation](#encapsulation)
- [Member functions](#member-functions)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a bank account that protects its own rules](#project-a-bank-account-that-protects-its-own-rules)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A class is a promise. It says: here is a kind of thing, here is the data it holds, and here are the only ways to change that data. The promise is valuable because the rules it protects are often easy to break by accident. A bank balance must never go negative. A temperature setting must stay in a safe range. A list of songs must never contain a blank title. If any code anywhere can change the balance directly, the rule holds only as long as every programmer remembers it.

C++ classes let you move the rules inside the type. The constructor sets up a valid starting state. The member functions are the only way to change the state, and each one checks the rule before it acts. The access keywords hide the data from outside code. Together, these features are called encapsulation, and they are the basis of object-oriented design in C++, which the next day extends with inheritance.

The class rules are defined in the ISO C++ standard, and cppreference.com has a page for the class declaration and one for each special member function. Keep them open while you build the project.

---

## Glossary

- **Class:** a user-defined type that groups data members and member functions under one name.
- **Object:** an instance of a class, created from its definition.
- **Member variable:** a data value that belongs to each object of a class, also called a data member.
- **Member function:** a function declared inside a class, which can read and change that object's data.
- **Constructor:** a special member function that sets up a new object. It has the same name as the class and no return type.
- **Destructor:** a special member function, written `~ClassName()`, that runs when the object's lifetime ends.
- **Initializer list:** the part of a constructor, written after a colon, that initializes members before the constructor body runs.
- **Delegating constructor:** a constructor that calls another constructor of the same class in its initializer list.
- **Access specifier:** the keyword `public`, `private` or `protected` that controls who can use a member.
- **Encapsulation:** the practice of hiding data and exposing only operations that protect the data's rules.
- **Invariant:** a condition that is true for every valid object at every point between calls, such as a balance that is never negative.
- **Const member function:** a member function marked `const`, which promises not to change the object.

---

## Core concepts

The concepts follow the life of a class object. It is created by a constructor, used through member functions, protected by access control, and destroyed by its destructor. Encapsulation ties the access rules and the member functions together into one design.

### Constructors

A constructor is a setup routine that runs when an object is born. Think of a new employee's first day. Before anyone can ask the employee to do work, the desk, the badge and the login must exist. A constructor makes sure the object starts in a valid state, so no other code ever sees a half-built object.

In first principles, the compiler allocates space for the object, then calls the constructor, which initializes each member. The initializer list is the most reliable way to set members, because it runs before the constructor body. Members declared `const`, and members of class types that have no default constructor, must be set in the initializer list. Assigning them in the body is not enough.

```cpp
#include <iostream>
#include <string>
#include <utility>

class Counter {
public:
    Counter() : Counter("unnamed", 0) {}  // delegating constructor

    Counter(std::string name, int start)
        : name_(std::move(name)), value_(start) {}

    void tick() { ++value_; }

    void print() const {
        std::cout << name_ << ": " << value_ << '\n';
    }

private:
    std::string name_;
    int value_;
};

int main() {
    Counter a;
    Counter b("clicks", 10);
    a.tick();
    b.tick();
    a.print();
    b.print();
    return 0;
}
```

The output is `unnamed: 1` and `clicks: 11`. The default constructor delegates to the two-argument constructor, so the setup logic lives in one place. The `std::move` passes the string into the member without copying it, which Day 37 explains. Mark a single-argument constructor `explicit` so the compiler does not convert values to your type silently.

The common mistake is to leave a member uninitialized in a constructor that does not set it. A plain `int` member with no initializer contains garbage, and the bug shows up later in an unrelated function. Give every member a value in the initializer list or with a default member initializer, such as `int value_ = 0;`. In the market, uninitialized members are one of the most common sources of bugs in C++ code review comments.

### Destructors

A destructor is the cleanup routine that runs when an object dies. It is the other half of the lifetime rules from Day 32. Think of a hotel checkout. The guest returns the key card, settles the bill and clears the room, so the next guest starts clean. A destructor releases whatever the object acquired during its life.

The rule of zero is the modern default. If a class holds only members that clean up themselves, such as `std::string` and `std::vector`, the class needs no destructor at all. The compiler generates one that destroys each member in reverse order of declaration.

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <vector>

class Playlist {
public:
    explicit Playlist(std::string name) : name_(std::move(name)) {}

    void add(std::string song) {
        songs_.push_back(std::move(song));
    }

    void print() const {
        for (const std::string& song : songs_) {
            std::cout << name_ << ": " << song << '\n';
        }
    }

    // No destructor needed: std::string and std::vector clean up their own memory.

private:
    std::string name_;
    std::vector<std::string> songs_;
};

int main() {
    Playlist mix("mix");
    mix.add("one");
    mix.add("two");
    mix.print();
    return 0;
}
```

The output is `mix: one` and `mix: two`. The class has no destructor, and no memory leaks, because the members release their own storage when the `Playlist` is destroyed. A class that holds a raw pointer or a file handle does need a destructor, which the Day 32 buffer and file wrapper showed.

The common mistake is to write a destructor that throws an exception. A destructor runs during stack unwinding, and a second exception during unwinding ends the program. Destructors should not throw, and the standard marks them `noexcept` by default. The other mistake is to put logic with side effects that matter, such as saving to disk, in a destructor, where errors cannot be reported. In the market, teams that build database and storage code keep destructors minimal and expose explicit `close` functions that can report errors.

### Access control

Access control decides which code may use each member. `public` members are the interface, which any code can call. `private` members are the implementation, which only the class itself can use. Think of a bank teller window. Customers can ask for a deposit at the window, but they cannot walk into the vault. The vault is private, and the window is the public interface.

```cpp
#include <iostream>

class Thermostat {
public:
    void set_celsius(double value) {
        if (value < 5.0 || value > 30.0) {
            return;  // reject values outside the safe range
        }
        celsius_ = value;
    }

    double celsius() const {
        return celsius_;
    }

private:
    double celsius_ = 20.0;
};

int main() {
    Thermostat t;
    t.set_celsius(42.0);  // rejected: outside the range
    t.set_celsius(22.5);  // accepted
    std::cout << t.celsius() << '\n';
    // t.celsius_ = 99.0;  // error: 'celsius_' is private
    return 0;
}
```

The output is `22.5`. The commented line would not compile, and the compiler error names the private member. Because the only way to change the temperature is `set_celsius`, the range check runs every time, and no caller can bypass it.

The keyword `protected` is for members that subclasses need, and it is the subject of Day 34. For now, prefer `private` by default, and make a member public only when it is meant to be part of the interface.

The common mistake is to make everything `public` for convenience. Then every function in the program can break the rules of the class, and a bug can come from anywhere. The opposite mistake is to write a getter that returns a non-const reference to a private member, which gives callers the same power as a public variable. The rule is to return by value or by const reference. In the market, public data members are a common review finding in shared libraries, because they make future changes expensive.

### Encapsulation

Encapsulation is the design habit that ties access control to invariants. An invariant is a rule that must hold for every valid object at every point between calls, such as a balance that is never negative or a list that has no duplicates. Encapsulation means the class names its invariants, puts the data behind private members, and checks each invariant in every function that could break it.

Think of a sealed jar of jam. The jar protects the contents, and the lid is the interface. You can open the jar only in the ways the jar allows. The purpose is to keep the contents in the state the maker intended.

Encapsulation has a cost. Every operation has to go through the interface, and an interface that is too narrow forces awkward workarounds. The skill is choosing the operations that match the real use of the object, which is why the bank account in the project has `deposit` and `withdraw` and not a `set_balance` function.

The common mistake is to write a class where the invariant is checked in only some of the functions. A later function that forgets the check breaks the rule for everyone. The fix is to list the invariants at the top of the header, and to make every mutating function check them before it changes the data. You can spot the gap when a bug report says the value was valid in one function and invalid after a different call. In the market, payments, medical devices and access control code rely on encapsulated types for this reason.

### Member functions

A member function belongs to a class and can read or change the object it is called on. The object is passed to the function implicitly as a pointer named `this`. The `const` keyword after the parameter list promises that the function will not change the object. Think of the `const` on a member function as a sign on a shop door that says "browse only." The compiler checks that promise, so a const function that tries to change a member fails to compile.

```cpp
#include <iostream>

class Rectangle {
public:
    Rectangle(double width, double height) : width_(width), height_(height) {}

    double area() const {
        return width_ * height_;
    }

    void scale(double factor) {
        width_ *= factor;
        height_ *= factor;
    }

private:
    double width_;
    double height_;
};

int main() {
    const Rectangle fixed(3.0, 4.0);
    std::cout << fixed.area() << '\n';

    Rectangle r(2.0, 5.0);
    r.scale(2.0);
    std::cout << r.area() << '\n';
    // fixed.scale(2.0);  // error: discards const qualifier, because scale is not const
    return 0;
}
```

The output is `12` and then `40`. `area` is marked `const`, so it works on the `const` object `fixed`. `scale` is not const, so the compiler refuses to call it on `fixed`. The `width_` and `height_` members are initialized in the initializer list, which the rule for const members would require if they were const.

The common mistake is to forget `const` on a read-only function. Then callers that hold a `const` object cannot call it, and the design becomes awkward. Mark every function that does not change the object as `const`, and let the compiler check it. In the market, const-correct interfaces are a standard review item, because they make it safe to pass objects by const reference.

---

## Hands-on exercises

Work in a new folder called `classes-objects`. Run `git init -b main` in it.

Exercise 1 writes a class with a constructor.

1. Write a `Point` class with two `double` members, `x` and `y`, initialized in an initializer list.
2. Add a constructor with no arguments that sets the point to the origin by delegating to the two-argument constructor.
3. Add a member function `distance_to_origin() const` and print the distance for two points.

Done when the default point prints `0` and a point at 3, 4 prints `5`.

Exercise 2 protects a rule with access control.

1. Write a class `Battery` with a private `int` charge from 0 to 100.
2. Add `charge_by(int amount)` that clamps the result to the range, and a `level() const` getter.
3. Try to write `battery.charge_ = 500;` from `main` and read the compiler error. Then remove the line.

Done when the clamping is tested for values below 0 and above 100, and the error text is in your notes.

Exercise 3 uses a destructor.

1. Write a class with a `std::string` member and no destructor. Create it in a nested scope and print a message from the scope's end.
2. Run the program under the AddressSanitizer build in WSL, which is described in the project. Confirm the sanitizer reports no leaks.

Done when the sanitizer report is clean.

Exercise 4 tests const correctness.

1. Write a class with one `const` member function and one non-const member function.
2. Create a `const` object and call each function. Record which call fails to compile and what the error says.

Done when you have pasted the exact error text into your notes.

Exercise 5 writes an invariant comment.

1. Choose a simple class from the earlier exercises and write three invariants at the top of its header as comments.
2. For each invariant, name the function that enforces it and the test that checks it.

Done when every invariant has a function and a test named next to it.

---

## Project: a bank account that protects its own rules

The project is a bank account library. Its rules are that the balance is never negative, deposits and withdrawals must be positive, and an account must have an owner. A failed operation changes nothing and does not count as a transaction. The test checks each rule, and the demo shows the account in use.

Suggested repo name: `100-days-software-day-33-classes-and-objects`.

The repo layout:

```
100-days-software-day-33-classes-and-objects/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── invariants.md
├── src/
│   ├── bank_account.cpp
│   ├── bank_account.hpp
│   └── main.cpp
└── tests/
    └── bank_account_test.cpp
```

The header lists the public interface and keeps the data private. The money is stored as `std::int64_t` cents, which Day 28 introduced:

```cpp
// src/bank_account.hpp
#ifndef BANK_ACCOUNT_HPP
#define BANK_ACCOUNT_HPP

#include <cstdint>
#include <string>

// Invariants:
//   1. balance_cents_ is never negative.
//   2. owner_ is never empty.
//   3. transactions_ counts only successful deposits and withdrawals.
class BankAccount {
public:
    BankAccount(std::string owner, std::int64_t opening_cents);
    ~BankAccount();

    BankAccount(const BankAccount&) = delete;
    BankAccount& operator=(const BankAccount&) = delete;

    void deposit(std::int64_t cents);
    void withdraw(std::int64_t cents);

    std::int64_t balance_cents() const { return balance_cents_; }
    const std::string& owner() const { return owner_; }
    int transaction_count() const { return transactions_; }

    static int open_count();

private:
    std::string owner_;
    std::int64_t balance_cents_;
    int transactions_;
};

#endif
```

The class deletes its copy operations, so two objects cannot share one account by accident. The constructor and member functions enforce the invariants:

```cpp
// src/bank_account.cpp
#include "bank_account.hpp"

#include <stdexcept>
#include <utility>

namespace {
int open_accounts = 0;
}

BankAccount::BankAccount(std::string owner, std::int64_t opening_cents)
    : owner_(std::move(owner)), balance_cents_(opening_cents), transactions_(0) {
    if (owner_.empty()) {
        throw std::invalid_argument("owner must not be empty");
    }
    if (opening_cents < 0) {
        throw std::invalid_argument("opening balance must not be negative");
    }
    ++open_accounts;
}

BankAccount::~BankAccount() {
    --open_accounts;
}

void BankAccount::deposit(std::int64_t cents) {
    if (cents <= 0) {
        throw std::invalid_argument("deposit must be positive");
    }
    balance_cents_ += cents;
    ++transactions_;
}

void BankAccount::withdraw(std::int64_t cents) {
    if (cents <= 0) {
        throw std::invalid_argument("withdrawal must be positive");
    }
    if (cents > balance_cents_) {
        throw std::runtime_error("insufficient funds");
    }
    balance_cents_ -= cents;
    ++transactions_;
}

int BankAccount::open_count() {
    return open_accounts;
}
```

The demo uses the account and prints the result with two decimal places:

```cpp
// src/main.cpp
#include <iomanip>
#include <iostream>
#include <stdexcept>

#include "bank_account.hpp"

int main() {
    BankAccount alice("Alice", 10000);  // $100.00
    alice.deposit(2550);
    alice.withdraw(4000);

    std::cout << std::fixed << std::setprecision(2);
    std::cout << alice.owner() << " balance: " << alice.balance_cents() / 100.0 << '\n';
    std::cout << "transactions: " << alice.transaction_count() << '\n';

    try {
        alice.withdraw(999999);
    } catch (const std::runtime_error& error) {
        std::cout << "rejected: " << error.what() << '\n';
    }
    std::cout << "open accounts: " << BankAccount::open_count() << '\n';
    return 0;
}
```

The output is `Alice balance: 85.50`, `transactions: 2`, `rejected: insufficient funds`, and `open accounts: 1`. The test checks each invariant, including the one that a failed withdrawal leaves the balance and the transaction count alone:

```cpp
// tests/bank_account_test.cpp
#include <cstdlib>
#include <iostream>
#include <stdexcept>

#include "bank_account.hpp"

namespace {

int failures = 0;

void check(bool condition, const char* label) {
    if (!condition) {
        std::cerr << "FAILED: " << label << '\n';
        ++failures;
    }
}

}  // namespace

int main() {
    const int base = BankAccount::open_count();

    {
        BankAccount account("Bo", 500);
        check(BankAccount::open_count() == base + 1, "constructor counts the account");

        account.deposit(250);
        check(account.balance_cents() == 750, "deposit adds cents");

        bool overdraft = false;
        try {
            account.withdraw(1000);
        } catch (const std::runtime_error&) {
            overdraft = true;
        }
        check(overdraft, "overdraft throws");
        check(account.balance_cents() == 750, "failed withdrawal leaves balance unchanged");
        check(account.transaction_count() == 1, "failed withdrawal is not counted");

        bool zero_deposit = false;
        try {
            account.deposit(0);
        } catch (const std::invalid_argument&) {
            zero_deposit = true;
        }
        check(zero_deposit, "zero deposit throws");
    }
    check(BankAccount::open_count() == base, "destructor decrements the count");

    bool empty_owner = false;
    try {
        BankAccount nobody("", 0);
    } catch (const std::invalid_argument&) {
        empty_owner = true;
    }
    check(empty_owner, "empty owner throws");
    check(BankAccount::open_count() == base, "failed constructor leaves no count behind");

    if (failures == 0) {
        std::cout << "all bank account tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the library, the demo and the test, and registers the test with CTest:

```cmake
cmake_minimum_required(VERSION 3.25)
project(bank_account LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_library(bank_core STATIC src/bank_account.cpp)
target_include_directories(bank_core PUBLIC src)

add_executable(bank_demo src/main.cpp)
target_link_libraries(bank_demo PRIVATE bank_core)

add_executable(bank_account_test tests/bank_account_test.cpp)
target_link_libraries(bank_account_test PRIVATE bank_core)

enable_testing()
add_test(NAME bank_account_test COMMAND bank_account_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add bank account with invariants".
2. Build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Run `ctest --test-dir build --output-on-failure` and confirm it passes.
3. Run `./build/bank_demo` and check each line against the values in the section above.
4. Create a branch `feature/transfer`. Write a test first for a `static bool transfer(BankAccount& from, BankAccount& to, std::int64_t cents)` function that moves money only when `from` has enough funds. Then add the function to the header and the source file, and make the test pass. Keep the function free of extra state.
5. Remove the `= delete` lines from the copy operations and try to copy an account in `main`. Read the compiler error, then restore the deleted operations and record the message in `docs/invariants.md`.
6. Write `docs/invariants.md` with three short sections, one for each invariant. Each section names the function that enforces the rule and the test that checks it.
7. Merge with `git merge --no-ff feature/transfer -m "Add transfer between accounts"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes on `main`.
- The demo output matches the four lines in the section above.
- `docs/invariants.md` names a function and a test for each of the three invariants.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/invariants.md`, `README.md`, and the history on GitHub.

This project is short and sharp. A reviewer should be able to read the header and see the three invariants written down, then find each one enforced in the source file and checked in the test.

---

## Check your understanding

1. A class holds a `const int limit_` member and assigns it in the constructor body. Explain why this does not compile, and what the fix is.
2. A class holds a `std::vector<std::string>` and a raw `FILE*`. Which members need a destructor, and what does the compiler generate for the others?
3. A getter returns `std::string&` to a private name member. Explain how it breaks encapsulation, and show the smallest change that fixes it.
4. A function `void set_balance(int)` lets callers set any balance. What invariant does it threaten, and what would you replace it with?
5. Why does the bank account delete its copy constructor? Describe the bug that copying would create.
6. Why should a destructor not throw an exception?

### Answers

1. A `const` member must be initialized before the constructor body runs, so it must appear in the initializer list. The body is too late. Move the assignment into the initializer list, as `limit_(value)`.
2. The `FILE*` needs a destructor, because the class must close the file, and the raw pointer does not clean itself up. The `std::vector` and `std::string` members release their own storage, so the compiler's generated destructor handles them.
3. The caller can change the name without going through any check, so any rule the class wanted to enforce on the name can be broken. Return `const std::string&` instead, so callers can read but not change it.
4. It threatens the rule that the balance follows only deposits and withdrawals, and any range rule the class enforces. Replace it with `deposit` and `withdraw` functions that check their arguments and the balance before they change it.
5. Two objects would share no state, but both would think they own the same account, so a change through one would surprise the other. Deleting the copy operations makes the copy a compile error and forces the design to pass references or to define a clear copy meaning.
6. A destructor runs during stack unwinding after another exception. If it throws during that process, the program calls `std::terminate` and ends. Destructors are `noexcept` by default for this reason.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Class (computer programming)](https://en.wikipedia.org/wiki/Class_(computer_programming)) for the general idea of classes
- [Wikipedia: Encapsulation (computer programming)](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)) for the design principle
- [Wikipedia: Class invariant](https://en.wikipedia.org/wiki/Class_invariant) for the rule that every valid object must keep
- cppreference.com, the pages for "Classes", "Constructors", "Destructors" and "Member access control"
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on classes, which include the rule of zero and the rule of five

*Tutorials*

- learncpp.com, the chapters on classes, constructors, destructors, access specifiers and const member functions, written by Alex Allain
- GeeksforGeeks, search the site for "C++ classes and objects" and "C++ constructors" for the matching articles
- W3Schools, the C++ "Classes and Objects" pages, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on classes and constructors
- CppCon, the YouTube channel, for talks on class design. Search the channel for "rule of zero" to find them.

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, with the exercises on classes. Use the test suite to check each invariant you write.

---

## Tomorrow

Day 34 extends classes with inheritance and polymorphism. You will make a base class with virtual functions, override them in derived classes, and see why a virtual destructor matters. You will also weigh inheritance against composition, which is a more flexible way to share behavior. The access specifiers and invariants from today decide what a derived class may change, so that the bank rules stay intact when the design grows.

**Next:** [Day 34: Inheritance and Polymorphism](Software-Day-34-Inheritance-and-Polymorphism.md)

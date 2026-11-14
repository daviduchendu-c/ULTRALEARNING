# Day 28: Core Syntax and Types

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** beginner. You finished Day 27 and can build a project with CMake. You know what a variable is from another language, and this day shows where C++ asks you to be more exact. It is for anyone who has been surprised by a number that came out as `3` when they expected `3.5`.

**By the end of this day, you will have:**

- Declared variables with explicit types, using `const` where a value must not change
- Explained integer division, the `%` operator and the difference between `int` and `double`
- Read numbers and text from the keyboard with `std::cin` and `std::getline`, and printed formatted output with `std::cout`
- A tip and bill-splitting command-line program that uses fixed-width integer types, plus a small test executable run by CTest
- A GitHub repo with a passing test run and a short notes file on the type decisions you made

**What you need:**

- The toolchain from Day 27: GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Day 27: the build steps, `-std=c++20`, the warning flags and a working `CMakeLists.txt`
- Basic arithmetic and the idea of a variable in any language

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Variables](#variables)
- [Types](#types)
- [Const](#const)
- [Operators](#operators)
- [Conversions](#conversions)
- [Input and output](#input-and-output)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a bill-splitting calculator with tests](#project-a-bill-splitting-calculator-with-tests)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most arithmetic errors in early programs are not bugs in the logic. They are bugs in the types. A program adds two prices and prints `19`, because both numbers were integers and the fraction was dropped without a warning. Another program stores a count of bytes in an `int`, and it quietly wraps around past two billion. Python hides these choices. C++ makes you name them, and that is why the language is worth learning carefully.

This day covers the syntax you use in almost every line: declaring variables, choosing types, marking values as constant, applying operators and reading and writing data. The project turns these ideas into a real tool that splits a restaurant bill, and it avoids the classic floating-point trap by storing money as whole cents.

The rules here are defined by the ISO C++ standard. The Wikipedia article [C++](https://en.wikipedia.org/wiki/C%2B%2B) gives the big picture, and cppreference.com has a page for each type and operator you will meet.

---

## Glossary

- **Variable:** a named place in memory that holds a value of a given type.
- **Type:** the rule that says what values a variable can hold and what operations apply to it.
- **Literal:** a value written directly in code, such as `42`, `3.14` or `'a'`.
- **Initialization:** giving a variable its first value when it is declared.
- **Const:** a qualifier that says a value cannot change after it is initialized.
- **Operator:** a symbol such as `+`, `%` or `==` that combines values.
- **Integer division:** division between two integers, which keeps only the whole part of the result.
- **Conversion:** changing a value from one type to another, either implicitly by the compiler or explicitly with `static_cast`.
- **Narrowing:** a conversion that can lose information, such as from `double` to `int`.
- **Overflow:** a result too large for the type, such as adding 1 to the largest `int`.
- **Stream:** an object that moves data in or out, such as `std::cin` for input and `std::cout` for output.
- **Fixed-width integer:** a type with a size known for certain, such as `std::int32_t` for exactly 32 bits.

---

## Core concepts

The concepts in this day build on one another. A variable needs a type, a type decides which operators are legal, and conversions happen whenever two types meet. Input and output are where all of this becomes visible to a user.

### Variables

Think of a variable as a labeled box on a shelf. The label is the name, the box has a size, and the size is set by the type. You cannot put a pair of shoes into a box meant for a cup, and C++ checks the same kind of fit when you assign a value.

In first principles, a variable is a region of memory. Its declaration tells the compiler the name and the type, and its initialization gives the region a starting value. In C++, you can write the type explicitly or let the compiler deduce it with `auto`. Here is a complete program that declares several variables and prints them:

```cpp
#include <iostream>

int main() {
    int students = 24;
    double average_score = 78.5;
    char grade = 'B';
    bool passed = true;

    std::cout << students << ' ' << average_score << ' ' << grade << ' ' << passed << '\n';
    return 0;
}
```

The output is `24 78.5 B 1`. The `bool` prints as `1` because `std::cout` writes a `bool` as an integer unless you ask for `std::boolalpha`. That surprises people, and it is one of the small formatting rules you will meet in the input and output section.

Brace initialization, such as `int students{24};`, is the modern form. It refuses narrowing conversions, which makes it safer. A common mistake is to declare a variable without an initial value and read it later. The value is then whatever bytes were in that memory, which is undefined behavior. The `-Wall` flag warns about the obvious cases, as you saw on Day 27. The rule is to give every variable a value at the point you declare it. In the market, code review checklists at most C++ teams include this rule, and linters enforce it.

### Types

A type answers three questions: how much memory a value takes, which values it can represent, and which operations make sense. `int` is the everyday integer. `double` is a floating-point number with about 15 significant decimal digits. `bool` holds `true` or `false`, and `char` holds a single character code.

The sizes of basic types are not fixed by the standard. The standard guarantees minimum ranges, and the size of `int` is at least 16 bits. On the Windows and Linux compilers used in this module, `int` is 32 bits, but you should not depend on that. For code that must be exact, use the fixed-width types from `<cstdint>`, such as `std::int32_t` for exactly 32 bits and `std::int64_t` for exactly 64 bits.

Here is a program that shows the limit of a 32-bit integer and how a wider type avoids it:

```cpp
#include <cstdint>
#include <iostream>

int main() {
    std::int32_t small = 2147483647;
    std::int64_t big = static_cast<std::int64_t>(small) + 1;

    std::cout << "int32 max: " << small << '\n';
    std::cout << "widened:   " << big << '\n';
    std::cout << "sizeof(int) = " << sizeof(int) << " bytes\n";
    return 0;
}
```

The output on Windows and Linux is `int32 max: 2147483647`, then `widened:   2147483648`, then `sizeof(int) = 4 bytes`. The line `small + 1` on its own would overflow. Signed overflow is undefined behavior in C++, so the compiler may do anything, although in practice it often wraps. The safe habit is to choose a type large enough for the largest value you expect, and to test the edges.

For sizes and indexes, the standard provides `std::size_t` from `<cstddef>`. Containers return sizes as `std::size_t`, and the type is unsigned. Mixing signed and unsigned values in comparisons is a classic source of wrong results, which the `-Wsign-compare` warning catches inside `-Wall -Wextra`.

The common mistake is to choose `int` for money or for counts that can exceed two billion, such as bytes in a large file. The fix is a wider or fixed-width type and an explicit unit. Money is the clearest case, and the project in this day stores cents in an `std::int64_t`. In the market, payments code and financial systems use integer minor units almost everywhere for the same reason.

### Const

A `const` value is a promise. Once you initialize it, the compiler refuses any change. That promise helps two readers: the person who maintains the code, who knows the value is fixed, and the compiler, which can optimize using that knowledge.

Consider a retry limit:

```cpp
#include <iostream>

int main() {
    const int max_retries = 3;
    int attempts = 0;

    while (attempts < max_retries) {
        ++attempts;
        std::cout << "attempt " << attempts << '\n';
    }
    return 0;
}
```

If you add a line `max_retries = 5;` after the declaration, the compiler stops with an error like `assignment of read-only variable 'max_retries'`. The change is caught at compile time, and the program never runs with the wrong rule.

Use `const` by default for any value that is not meant to change. The common mistake is to skip it because it feels like extra typing, and then a later edit changes a value by accident. A useful signal is that a variable declared without `const` in a long function is easy to misread. In the market, `const` correctness is a standard review point in C++ codebases, and it becomes more important when many functions share data.

`const` also matters for pointers and member functions, which later days cover. For now, the rule is simple: if the value is decided once, declare it `const`.

### Operators

Operators combine values. C++ has the arithmetic operators `+`, `-`, `*`, `/` and `%`, the comparison operators `==`, `!=`, `<`, `>`, `<=` and `>=`, and the logical operators `&&`, `||` and `!`. Most behave as you expect, with one important exception: division between two integers discards the fraction.

```cpp
#include <iostream>

int main() {
    int a = 7;
    int b = 2;

    std::cout << "a / b   = " << a / b << '\n';
    std::cout << "a % b   = " << a % b << '\n';
    std::cout << "a / 2.0 = " << a / 2.0 << '\n';
    return 0;
}
```

The output is `a / b   = 3`, then `a % b   = 1`, then `a / 2.0 = 3.5`. Integer division truncates toward zero, so `7 / 2` is `3`. The `%` operator returns the remainder, which is what you need when you ask "how many cents are left over after splitting this bill." The `2.0` in the last line is a `double` literal, and the presence of a `double` in the expression changes the arithmetic.

Operator precedence follows mathematical convention: `*`, `/` and `%` bind tighter than `+` and `-`. Use parentheses whenever the intent is not obvious, because readers should not have to remember the table. Increment `++` also has two forms. `++i` increments first and returns the new value, and `i++` returns the old value. In a statement on its own, the difference does not matter, so prefer `++i` in loops.

The common mistake is to write `total / count` with two integers and expect a decimal result. The fix is to convert one side first, which the next section covers. You can spot the mistake when an average is always a whole number. In the market, average calculations that silently truncate show up in analytics dashboards and billing code, and teams find them through tests with values that do not divide evenly.

### Conversions

A conversion changes a value from one type to another. Some conversions are implicit: the compiler inserts them when you assign an `int` to a `double`, which is safe because every `int` fits in a `double`. Others lose information: a `double` converted to an `int` drops the fraction, and a large `int64` converted to `int32` can wrap.

Explicit conversion with `static_cast` tells the compiler and your reader that you meant the change. Here is a program that converts a price and shows the difference between an ordinary assignment and brace initialization:

```cpp
#include <iostream>

int main() {
    double price = 19.99;
    int whole = static_cast<int>(price);
    int from_braces{whole};

    std::cout << whole << ' ' << from_braces << '\n';
    return 0;
}
```

The output is `19 19`. The fraction `0.99` is gone, and `static_cast` made that loss deliberate. If you write `int bad{price};` instead, the compiler refuses with a narrowing conversion error. If you write `int bad = price;`, the compiler accepts it, and `-Wconversion` warns about it. The `-Wconversion` flag is not part of `-Wall -Wextra`, so it is worth adding to your project when you want stricter checks.

To convert money stored as a `double` into cents, use `std::llround` from `<cmath>` instead of a plain cast. Multiplying `19.99` by 100 gives `1998.9999999999998` in binary floating point, and a plain cast truncates it to `1998`. Rounding fixes that. The project uses this approach, and the test checks it.

The common mistake is to rely on implicit conversion inside an expression without thinking about the result type. The safe rule is to convert at the boundary, where input enters the program, and keep the internal type consistent. You can spot the problem by looking for values that are one unit off from the expected result. In the market, unit-conversion bugs in aerospace and robotics software have caused real losses, so teams enforce explicit conversions at interfaces.

### Input and output

Standard input and output use streams. `std::cin` reads from the keyboard, and `std::cout` writes to the terminal. The `>>` operator extracts a value from a stream, and the `<<` operator inserts one into it. The stream knows the type of the variable you give it, so `std::cin >> age` with an `int` reads an integer.

Input can fail. If the user types `twenty` when you expected a number, the stream enters a failed state, and later reads return nothing until you clear it. Check the result of every read:

```cpp
#include <iostream>
#include <string>

int main() {
    int age = 0;
    std::cout << "Age: ";
    if (!(std::cin >> age)) {
        std::cerr << "error: expected a whole number\n";
        return 1;
    }

    std::cin.ignore();  // discard the newline left by the previous read
    std::string name;
    std::cout << "Name: ";
    std::getline(std::cin, name);

    std::cout << name << " is " << age << '\n';
    return 0;
}
```

The `std::cin.ignore()` call matters. After `std::cin >> age`, the newline the user typed remains in the input buffer. A `std::getline` without that call returns an empty string immediately. This trap appears in almost every beginner program that mixes numeric and text input.

Formatting is set on the stream. `std::fixed` and `std::setprecision(2)` from `<iomanip>` print two decimal places, and `std::boolalpha` prints `true` or `false` instead of `1` or `0`. The formatting persists for later output on the same stream, which is useful and also a source of surprise when you forget you set it.

The common mistake is to ignore the failure state and let a program continue with a zero or garbage value. The check is cheap and the failure is common, because users make typing mistakes. In the market, input validation is the first defense in command-line tools, form handlers and embedded configuration parsers.

---

## Hands-on exercises

Work in a new folder called `core-syntax-types`. Run `git init -b main` in it.

Exercise 1 declares and prints variables.

1. Write a program with an `int`, a `double`, a `char` and a `bool`. Give each one an initial value.
2. Print all four on one line. Then add `std::boolalpha` before the `bool` and print again.
3. Compile with `g++ -std=c++20 -Wall -Wextra -Wpedantic -Werror`.

Done when the program compiles with no warnings and prints `true` or `false` for the `bool`.

Exercise 2 shows integer division.

1. Compute `17 / 5`, `17 % 5`, `17 / 5.0` and `17.0 / 5` in one program. Predict each result before you run it.
2. Compare your predictions with the output and write one sentence on any surprise.

Done when every prediction is either correct or explained in your notes.

Exercise 3 breaks the 32-bit limit.

1. Store `2147483647` in an `std::int32_t` and add 1 to a `std::int64_t` copy of it.
2. Print both values and the output of `sizeof(std::int32_t)` and `sizeof(std::int64_t)`.

Done when the widened value prints as `2147483648`.

Exercise 4 reads input safely.

1. Write a program that asks for an age with `std::cin >> age`, checks the result, and then reads a name with `std::getline`.
2. Run it and type `abc` at the age prompt. Confirm that the error message appears and the program exits with status 1. Check the status with `echo $?` in Git Bash or `$LASTEXITCODE` in PowerShell.
3. Run it again with valid input, first without `std::cin.ignore()`, then with it. Note the difference in the name output.

Done when you have recorded the exit code for bad input and the name output with and without `ignore`.

Exercise 5 uses `const`.

1. Declare `const double tax_rate = 0.08;` and use it to compute a tax amount from a price.
2. Try to assign a new value to `tax_rate` and read the compiler error. Then remove the line.

Done when you have pasted the exact error text into your notes.

---

## Project: a bill-splitting calculator with tests

The project is a command-line tool. You enter a bill, a tip percentage and a number of people, and it prints the total with tip, each person's share and any leftover cents. The arithmetic is done with integer cents, so the results are exact, and a small test executable checks the rules.

Suggested repo name: `100-days-software-day-28-core-syntax-and-types`.

The repo layout:

```
100-days-software-day-28-core-syntax-and-types/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── type-decisions.md
├── src/
│   ├── main.cpp
│   └── split.hpp
└── tests/
    └── split_test.cpp
```

The `split.hpp` header holds the arithmetic. It is a pure function, so the test can call it without reading the keyboard:

```cpp
// src/split.hpp
#ifndef SPLIT_HPP
#define SPLIT_HPP

#include <cstdint>
#include <stdexcept>

struct Split {
    std::int64_t total_cents;
    std::int64_t per_person_cents;
    std::int64_t remainder_cents;
};

// Tip is a whole percentage. Integer division truncates toward zero,
// which is correct here because every value is non-negative.
inline Split split_bill(std::int64_t bill_cents, int tip_percent, int people) {
    if (people <= 0) {
        throw std::invalid_argument("people must be at least 1");
    }
    if (tip_percent < 0 || bill_cents < 0) {
        throw std::invalid_argument("bill and tip must not be negative");
    }

    const std::int64_t tip_cents = bill_cents * tip_percent / 100;
    const std::int64_t total = bill_cents + tip_cents;
    const std::int64_t per_person = total / people;
    const std::int64_t remainder = total % people;

    return Split{total, per_person, remainder};
}

#endif
```

The `main.cpp` file reads the inputs, converts the bill to cents, and prints the results:

```cpp
// src/main.cpp
#include <cmath>
#include <cstdint>
#include <iomanip>
#include <iostream>
#include <stdexcept>

#include "split.hpp"

int main() {
    double bill_dollars = 0.0;
    int tip_percent = 0;
    int people = 0;

    std::cout << "Bill amount (for example 84.50): ";
    if (!(std::cin >> bill_dollars)) {
        std::cerr << "error: the bill must be a number\n";
        return 1;
    }

    std::cout << "Tip percent (for example 18): ";
    if (!(std::cin >> tip_percent)) {
        std::cerr << "error: the tip must be a whole number\n";
        return 1;
    }

    std::cout << "Number of people: ";
    if (!(std::cin >> people)) {
        std::cerr << "error: the number of people must be a whole number\n";
        return 1;
    }

    const auto bill_cents = static_cast<std::int64_t>(std::llround(bill_dollars * 100.0));

    try {
        const Split s = split_bill(bill_cents, tip_percent, people);
        std::cout << std::fixed << std::setprecision(2);
        std::cout << "Total with tip:   " << s.total_cents / 100.0 << '\n';
        std::cout << "Each person pays: " << s.per_person_cents / 100.0 << '\n';
        std::cout << "Cents left over:  " << s.remainder_cents << '\n';
    } catch (const std::invalid_argument& error) {
        std::cerr << "error: " << error.what() << '\n';
        return 1;
    }

    return 0;
}
```

The test file checks the rules with a small helper. It returns a non-zero exit code on failure, which is what CTest needs:

```cpp
// tests/split_test.cpp
#include <cstdlib>
#include <iostream>
#include <stdexcept>

#include "split.hpp"

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
    const Split even = split_bill(10000, 10, 4);  // $100.00 plus 10 percent
    check(even.total_cents == 11000, "total includes tip");
    check(even.per_person_cents == 2750, "even split per person");
    check(even.remainder_cents == 0, "no leftover cents");

    const Split uneven = split_bill(1000, 0, 3);  // $10.00 split three ways
    check(uneven.per_person_cents == 333, "uneven split per person");
    check(uneven.remainder_cents == 1, "one leftover cent");

    bool threw = false;
    try {
        split_bill(1000, 10, 0);
    } catch (const std::invalid_argument&) {
        threw = true;
    }
    check(threw, "zero people throws");

    if (failures == 0) {
        std::cout << "all split tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds both executables and registers the test with CTest. It applies the same warning policy to every target:

```cmake
cmake_minimum_required(VERSION 3.25)
project(split_bill LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_executable(split_app src/main.cpp)
target_include_directories(split_app PRIVATE src)

add_executable(split_test tests/split_test.cpp)
target_include_directories(split_test PRIVATE src)

enable_testing()
add_test(NAME split_test COMMAND split_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add bill splitter with integer cents".
2. Configure and build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Fix any warning before you go on.
3. Run the test with `ctest --test-dir build --output-on-failure`. Confirm it passes.
4. Run the app with the input `84.50`, `18` and `3`. Check the output: the total with tip should be 99.71, each person pays 33.23, and 2 cents are left over.
5. Create a branch `feature/rounding-check`. Add a test that checks a bill of `19.99` with 0 percent tip split one way gives 1999 cents. To convert the dollar amount in the test, use `std::llround(19.99 * 100.0)` and confirm the value is 1999, not 1998.
6. Write `docs/type-decisions.md` with three short sections: why money is stored in `std::int64_t` cents, why the tip is an `int` percentage, and what would go wrong if the bill were a `double` all the way through. Use one concrete number from your own test runs in each section.
7. Commit the branch, merge it into `main` with `git merge --no-ff feature/rounding-check -m "Add rounding check"`, and create an empty GitHub repository with the same name. Then run `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes with the test run shown in your terminal.
- The app prints 99.71, 33.23 and 2 for the sample input.
- The rounding test passes and `docs/type-decisions.md` quotes one real output from your runs.
- The build has no warnings under `-Wall -Wextra -Wpedantic -Werror`.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the three source files, the test file, `docs/type-decisions.md`, `README.md`, and the history on GitHub.

This project is short on code and heavy on decisions. The graded part is the type choice and the test that proves it. A reader should be able to see why `split_bill` takes cents and not dollars.

---

## Check your understanding

1. A program computes the average of `int` scores as `sum / count` and prints `81` for a true average of `81.5`. Explain why, and name two fixes.
2. You store the total of an invoice as a `double` and display it with two decimals. Under what conditions can a sum of prices differ from the expected cents, and what change makes the arithmetic exact?
3. A teammate writes `int size = some_vector.size();` and the compiler warns about a sign conversion. Explain what `size()` returns, and what the safest fix is.
4. Why does `std::cin >> age` followed by `std::getline` return an empty string, and what does `std::cin.ignore()` change?
5. A file contains the line `int bytes = file_size_in_bytes;` for a file 3 GB in size. Describe what can happen and which type you would choose.
6. Why is `const` on a local variable more than a style choice?

### Answers

1. Both operands are `int`, so the division truncates the fraction. Fix it by converting one operand, as in `sum / static_cast<double>(count)`, or by computing in cents or tenths with an integer and formatting the result with a decimal point.
2. Binary floating point cannot represent many decimal fractions exactly, so sums of values such as 0.10 can drift by a tiny amount, and rounding can produce a cent of difference. Store cents in a 64-bit integer and convert only for display.
3. `size()` returns `std::size_t`, which is unsigned. Assigning it to a signed `int` can change its value for very large sizes. The safest fix is to declare `std::size_t size = some_vector.size();` or use `auto`.
4. The number is read, but the newline the user typed stays in the input buffer. `std::getline` reads up to that newline, so it finds an empty line immediately. `std::cin.ignore()` discards one character, which removes the leftover newline.
5. A 3 GB value does not fit in a 32-bit `int`, which has a maximum around 2.1 billion, so it can overflow or wrap. Use `std::int64_t` or `std::uintmax_t` for byte counts.
6. A `const` declaration tells the reader and the compiler that the value is decided once. It turns accidental edits into compile errors and makes the code easier to review, which matters more as functions grow.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: C++](https://en.wikipedia.org/wiki/C%2B%2B) for the language overview
- [Wikipedia: Integer (computer science)](https://en.wikipedia.org/wiki/Integer_(computer_science)) and [Wikipedia: Floating-point arithmetic](https://en.wikipedia.org/wiki/Floating-point_arithmetic) for the number representation behind this day
- cppreference.com, the pages for "Fundamental types", "Integer types" and "Operator precedence"
- The C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, sections on arithmetic and on the use of `const`
- Stanford CS106L course materials, from the Stanford CS106L website, for a university treatment of types and streams

*Tutorials*

- learncpp.com, the chapters on variables, fundamental data types, const, operators and input and output, written by Alex Allain
- GeeksforGeeks, search the site for "C++ data types" and "C++ operators" to find the matching articles
- Microsoft Learn, search learn.microsoft.com for "C++ built-in types" for the Windows-side reference

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on data types and operators
- CppCon, the YouTube channel, for talks on floating-point and integer pitfalls, which you can search by title on the channel

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, which offers small exercises with a test suite. Complete the early exercises on arithmetic and strings and run the tests from your terminal.

---

## Tomorrow

Day 29 adds control flow and functions. You will write `if`, `switch`, `for` and `while`, define functions with parameters and return values, overload a function by its parameter types, and see the difference between passing a value and passing a reference. The types and `const` habits from today carry directly into function signatures, and the split calculator's `split_bill` function is the first step toward those ideas.

**Next:** [Day 29: Control Flow and Functions](Software-Day-29-Control-Flow-and-Functions.md)

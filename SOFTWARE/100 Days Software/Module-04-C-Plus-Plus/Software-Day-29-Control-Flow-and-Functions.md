# Day 29: Control Flow and Functions

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** beginner. You can declare variables and read input from Day 28, and you know what a function is from another language. This day covers how a program decides and repeats, and how it splits work into named pieces. It is for anyone whose programs grow into one long `main` that nobody wants to touch.

**By the end of this day, you will have:**

- Written conditions with `if`, `else if`, `else` and `switch`, and loops with `for`, `while` and `continue`
- Defined functions with parameters and return types, and overloaded one name for several parameter types
- Shown the difference between passing by value and passing by reference, with a program that proves it
- A grade-report command-line tool that reads scores from standard input, with a separate test executable run by CTest
- A GitHub repo with the build, the test output and a short note on the overload choices you made

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer, from Day 27
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Day 27: the build steps and the warning flags
- Day 28: variables, types, `const`, operators and `std::cin` with a check for failed reads

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Conditions](#conditions)
- [Loops](#loops)
- [Functions](#functions)
- [Overloads](#overloads)
- [Pass by value and reference](#pass-by-value-and-reference)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a grade report with overloads and tests](#project-a-grade-report-with-overloads-and-tests)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A program that only runs one line after another is little more than a shopping list. Real software decides based on input, repeats work over a collection of items, and hides details behind names so that the next reader can understand the top level without reading every line. Control flow gives the first two abilities, and functions give the third.

This day is about writing those pieces well. Most of the work in C++ is choosing how values move into and out of functions. Passing a value and passing a reference look almost identical in code, but they behave differently, and the wrong choice causes bugs that are hard to see. The project builds a grade report in which every rule lives in a small function that you can test.

The formal rules for statements and functions are in the ISO C++ standard. For a readable reference, cppreference.com has a page for each statement form and for function declarations.

---

## Glossary

- **Condition:** an expression that evaluates to `true` or `false`, used by `if`, `while` and `for`.
- **Branch:** one of the paths an `if` or `switch` can take.
- **Loop:** a statement that repeats a block while a condition holds.
- **Iteration:** one pass through the body of a loop.
- **Function:** a named block of code that takes parameters and may return a value.
- **Parameter:** a variable in a function declaration that receives a value from the caller.
- **Argument:** the value the caller passes in for a parameter.
- **Return type:** the type of value a function gives back, or `void` if it gives back nothing.
- **Overload:** a second function with the same name but different parameter types, chosen by the compiler at the call site.
- **Pass by value:** the function receives a copy of the argument, so changes do not reach the caller.
- **Pass by reference:** the function receives an alias for the caller's variable, so changes do reach the caller.
- **Lvalue reference:** a reference written with `&`, such as `int&`, that can bind to an existing variable.
- **Undefined behavior:** code the standard does not define. The compiler may produce any result, so it must be avoided.

---

## Core concepts

The concepts in this day form one path. Conditions and loops decide what runs and how often. Functions package that logic into named pieces. Overloads let one name serve several types, and the rules for passing arguments decide what a function can change. The sections follow that path.

### Conditions

A condition is a question the program asks about its data, and the answer picks a branch. Think of a sorting shelf at a post office. Each parcel goes to the first slot whose label matches, and the rest of the shelves are skipped. An `if` chain works the same way: the first true condition runs its block, and the others are skipped.

In first principles, the CPU compares values and jumps to one part of the code or another. C++ gives you the `if` statement to write that jump in readable form. Here is a grade classifier that uses an `if` chain:

```cpp
#include <iostream>

int main() {
    int score = 84;
    char grade = 'F';

    if (score >= 90) {
        grade = 'A';
    } else if (score >= 80) {
        grade = 'B';
    } else if (score >= 70) {
        grade = 'C';
    } else if (score >= 60) {
        grade = 'D';
    }

    std::cout << "score " << score << " gets " << grade << '\n';
    return 0;
}
```

The output is `score 84 gets B`. Order matters. If you swapped the first two checks, a score of 95 would match `score >= 80` first and get a `B`, which is wrong. Each condition is tested only after the ones above it fail, so put the most specific rule first.

Use `switch` when one integer or character value selects among several fixed cases. Each `case` ends with `break`, or the code falls through to the next case. Fall-through is sometimes useful and often a bug. Recent GCC versions and Clang warn about it under `-Wimplicit-fallthrough`, which `-Wextra` enables. The `[[fallthrough]];` attribute marks a fall-through as intentional. Here is a `switch` that counts grades:

```cpp
#include <iostream>

int main() {
    char grades[] = {'A', 'B', 'B', 'C', 'F'};
    int b_count = 0;

    for (char g : grades) {
        switch (g) {
            case 'A':
                break;
            case 'B':
                ++b_count;
                break;
            default:
                break;
        }
    }

    std::cout << "B grades: " << b_count << '\n';
    return 0;
}
```

The output is `B grades: 2`. The loop over `grades` is the subject of the next section, and the `default` case handles any value the other cases did not match.

The common mistake is to write two `if` statements where one `else if` was intended. The second `if` runs even when the first one matched, so a score of 95 can increment both counters. You can spot it when totals do not add up to the number of items. In the market, branching logic that drifts out of sync is one of the most common causes of billing and eligibility bugs, which is why teams write tests for each boundary.

### Loops

A loop repeats a block, and each repetition is an iteration. Picture a mail carrier who walks one street at a time until the bag is empty. The loop condition is the check at the start of each walk: if the bag still has mail, walk the next street. The bag is the state that changes, and without a change the loop never ends.

C++ provides `while`, `do while`, and `for`. The `for` loop is the most common because it groups the setup, the test and the update on one line. A range-based `for` loop, such as `for (char g : grades)`, walks through every element of an array or container without an index.

```cpp
#include <iostream>

int main() {
    int odd_total = 0;

    for (int n = 1; n <= 10; ++n) {
        if (n % 2 == 0) {
            continue;  // skip the rest of this iteration
        }
        odd_total += n;
    }

    std::cout << "sum of odd numbers up to 10: " << odd_total << '\n';

    int countdown = 3;
    while (countdown > 0) {
        std::cout << countdown << '\n';
        --countdown;
    }
    return 0;
}
```

The output is `sum of odd numbers up to 10: 25`, followed by `3`, `2` and `1` on separate lines. The `continue` statement skips the rest of one iteration and moves to the next. The `break` statement leaves the loop entirely. Both are useful, and both make a loop harder to read if you use many of them. Prefer one clear condition over a pile of `break` statements.

The common mistake is an off-by-one error, where a loop runs one time too many or too few. The usual cause is `<` where `<=` was intended, or starting at 1 when the list starts at 0. Test the first and last values by hand. A second mistake is changing the loop variable inside a range-based loop, which is not possible with `for (char g : grades)` because the variable is a copy. In the market, loops over large datasets are where performance problems and off-by-one bugs cost the most, so teams test boundary values in every loop they write.

### Functions

A function is a named piece of work with a clear input and output. Think of a recipe card with an ingredient list at the top and a finished dish at the end. The caller reads the name and the parameters and does not need the method. In C++, a function declaration gives the name, the return type and the parameter list, and a definition gives the body.

The definition of a function is independent of the place it is called. That allows you to test it on its own and to reuse it. Here is a small function that converts a grade to a letter:

```cpp
#include <iostream>

char letter_grade(int score) {
    if (score >= 90) {
        return 'A';
    }
    if (score >= 80) {
        return 'B';
    }
    if (score >= 70) {
        return 'C';
    }
    if (score >= 60) {
        return 'D';
    }
    return 'F';
}

int main() {
    std::cout << letter_grade(95) << letter_grade(72) << letter_grade(40) << '\n';
    return 0;
}
```

The output is `ACF`: `letter_grade(95)` returns `A`, `letter_grade(72)` returns `C`, and `letter_grade(40)` returns `F`. Each `return` ends the function immediately, so the early returns replace an `else` chain. Both styles are common, and early returns keep the rule readable.

Two details matter in the signature. The `void` return type means the function returns nothing. A function that promises a return value and does not provide one on every path produces undefined behavior, and `-Wall` warns about it with `-Wreturn-type`. The second detail is that each parameter has a type, and the compiler checks every call against that list.

The common mistake is to make a function do two jobs, such as parsing input and computing a result. That makes testing harder and the name less honest. A function named `letter_grade` should only compute a letter. You can spot the problem when a function name needs the word "and" to describe it. In the market, teams that keep functions focused tend to have shorter review cycles and fewer regressions.

### Overloads

An overload is a second function that shares a name with the first, but takes different parameter types. Think of a restaurant menu where "coffee" can be ordered as "small", "medium" or "large", and the waiter understands which size you mean from the word you use. The compiler does the same job by matching the argument types at each call.

Overloads let one name cover several types without forcing the caller to pick a different name. Here is a formatting function with two versions:

```cpp
#include <iostream>
#include <iomanip>
#include <sstream>
#include <string>

std::string format_score(int score) {
    return std::to_string(score);
}

std::string format_score(double score) {
    std::ostringstream out;
    out << std::fixed << std::setprecision(1) << score;
    return out.str();
}

int main() {
    std::cout << format_score(85) << '\n';
    std::cout << format_score(84.6) << '\n';
    return 0;
}
```

The output is `85` on the first line and `84.6` on the second. The compiler picks the `int` version for `85` and the `double` version for `84.6`, because each argument matches one of the declarations exactly. The chosen version is part of the function's signature, which is the name plus the parameter types. The return type does not take part in overload selection.

The rules become tricky when no version matches exactly. If you call `format_score(85u)` with an unsigned value, both the `int` and `double` versions require a conversion, and the compiler reports an ambiguous call. The error message lists both candidates. The fix is to add a version that matches exactly or to cast the argument. That limit is worth knowing, because overload resolution can seem magical until a call fails to compile.

The common mistake is to overload functions that do different things, so that a call with the same name behaves in a surprising way for one type. Keep overloads doing the same job on different types. In the market, the standard library uses overloads heavily, for example for stream output with `std::ostream::operator<<`, and the consistency of that design is why readers can guess what a call does.

### Pass by value and reference

When you pass an argument to a function, the parameter receives either a copy of the value or an alias for the original variable. That choice decides whether the function can change the caller's data. The two forms look nearly the same in the call, which makes the choice easy to miss.

Here is a program that shows both behaviors side by side:

```cpp
#include <iostream>

void try_to_double(int value) {
    value *= 2;
    std::cout << "inside the copy: " << value << '\n';
}

void really_double(int& value) {
    value *= 2;
}

int main() {
    int score = 40;

    try_to_double(score);
    std::cout << "after pass by value: " << score << '\n';

    really_double(score);
    std::cout << "after pass by reference: " << score << '\n';
    return 0;
}
```

The output is `inside the copy: 80`, then `after pass by value: 40`, and then `after pass by reference: 80`. The first function changes its own copy, and the caller's `score` is untouched. The copy is used inside the function, which keeps the compiler from warning that the parameter is set but never read. The second function uses `int&`, which means "a reference to an int," so `value` is another name for `score`, and the change reaches the caller.

Reference parameters have a second use: avoiding a copy. A `std::string` can be large, and copying one on every call wastes time. For read-only access, pass by `const` reference, written `const std::string&`. The function can read the value without copying it and cannot change it. Small types such as `int` and `double` are cheap to copy, so passing them by value is normal.

The common mistake is to take a non-const reference when the function only reads the value, which makes the signature lie about what the function does. The opposite mistake is to pass a large object by value out of habit, which makes every call slower. The rule to use is simple: pass small values by value, read large values by `const` reference, and use a non-const reference only when the function must change the caller's variable. You can spot the bug by looking for a function that changes a value you thought was safe. In the market, code reviewers look for these signatures first, because a misplaced `&` can silently change data in a shared structure.

---

## Hands-on exercises

Work in a new folder called `control-flow-functions`. Run `git init -b main` in it.

Exercise 1 uses an `if` chain.

1. Write a program that classifies a temperature as `cold` below 10, `mild` from 10 to 24, and `hot` at 25 or above.
2. Test it with 5, 10, 24, 25 and -3. Predict each output before running it.
3. Compile with `g++ -std=c++20 -Wall -Wextra -Wpedantic -Werror`.

Done when every boundary value gives the category you predicted.

Exercise 2 uses a loop with `continue`.

1. Print every multiple of 3 from 1 to 30 in one line, separated by spaces.
2. Count how many values in the range you printed.
3. Rewrite the loop so that it uses `continue` to skip values that are not multiples of 3.

Done when the printed list is `3 6 9 12 15 18 21 24 27 30` and the count is 10.

Exercise 3 writes functions with and without a return value.

1. Write `int square(int x)` that returns `x * x`, and `void print_square(int x)` that prints the result.
2. Call both from `main` and compare the output.
3. Remove the `return` from `square` and compile. Read the warning, then restore the return statement.

Done when you have pasted the warning text into your notes.

Exercise 4 proves the difference between value and reference.

1. Write the two functions from the pass by value and reference section.
2. Add a third function `void add_bonus(int& score, int bonus)` that adds `bonus` to `score`. Call it with 40 and 5 and print the result.

Done when the output shows the original value unchanged after the by-value call and the new value after the reference call.

Exercise 5 creates an overload and proves which one is chosen.

1. Write two versions of `describe` that take an `int` and a `double`. Each one prints a label that says which version was used.
2. Call `describe(3)`, `describe(3.0)` and `describe(3.0f)` and record which version runs for each call.

Done when you can explain the `3.0f` result in one sentence.

---

## Project: a grade report with overloads and tests

The project is a command-line tool. It reads whole-number scores from standard input until the input ends, then prints how many scores it read, the average, the lowest and highest score, and a count of each letter grade. Every rule lives in a function with a test.

Suggested repo name: `100-days-software-day-29-control-flow-and-functions`.

The repo layout:

```
100-days-software-day-29-control-flow-and-functions/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── overload-notes.md
├── src/
│   ├── grades.cpp
│   ├── grades.hpp
│   └── main.cpp
└── tests/
    └── grades_test.cpp
```

The header declares the functions and the statistics record. Declarations live here, and definitions live in `grades.cpp`, the same pattern as Day 27:

```cpp
// src/grades.hpp
#ifndef GRADES_HPP
#define GRADES_HPP

#include <string>

struct Stats {
    int count = 0;
    long long sum = 0;
    int min = 0;
    int max = 0;
};

char letter_grade(int score);
std::string format_score(int score);
std::string format_score(double score);
void add_score(Stats& stats, int score);
double average(const Stats& stats);

#endif
```

The definitions go in `grades.cpp`. The `letter_grade` function rejects scores outside 0 to 100 by throwing, so bad data cannot produce a silent `F`:

```cpp
// src/grades.cpp
#include "grades.hpp"

#include <iomanip>
#include <sstream>
#include <stdexcept>

char letter_grade(int score) {
    if (score < 0 || score > 100) {
        throw std::out_of_range("score must be between 0 and 100");
    }
    if (score >= 90) {
        return 'A';
    }
    if (score >= 80) {
        return 'B';
    }
    if (score >= 70) {
        return 'C';
    }
    if (score >= 60) {
        return 'D';
    }
    return 'F';
}

std::string format_score(int score) {
    return std::to_string(score);
}

std::string format_score(double score) {
    std::ostringstream out;
    out << std::fixed << std::setprecision(1) << score;
    return out.str();
}

void add_score(Stats& stats, int score) {
    if (stats.count == 0) {
        stats.min = score;
        stats.max = score;
    } else {
        if (score < stats.min) {
            stats.min = score;
        }
        if (score > stats.max) {
            stats.max = score;
        }
    }
    stats.sum += score;
    ++stats.count;
}

double average(const Stats& stats) {
    if (stats.count == 0) {
        return 0.0;
    }
    return static_cast<double>(stats.sum) / stats.count;
}
```

The `main.cpp` file reads the scores with a loop. A failed read that is not the end of input means the user typed something that is not a whole number, and the program stops with an error:

```cpp
// src/main.cpp
#include <iostream>
#include <stdexcept>

#include "grades.hpp"

int main() {
    Stats stats;
    int letter_a = 0;
    int letter_b = 0;
    int letter_c = 0;
    int letter_d = 0;
    int letter_f = 0;

    int score = 0;
    while (std::cin >> score) {
        try {
            const char grade = letter_grade(score);  // throws before any state changes
            add_score(stats, score);
            switch (grade) {
                case 'A':
                    ++letter_a;
                    break;
                case 'B':
                    ++letter_b;
                    break;
                case 'C':
                    ++letter_c;
                    break;
                case 'D':
                    ++letter_d;
                    break;
                default:
                    ++letter_f;
                    break;
            }
        } catch (const std::out_of_range& error) {
            std::cerr << "error: " << error.what() << '\n';
            return 1;
        }
    }

    if (!std::cin.eof()) {
        std::cerr << "error: input must contain only whole numbers\n";
        return 1;
    }

    if (stats.count == 0) {
        std::cout << "no scores entered\n";
        return 0;
    }

    std::cout << "count:   " << stats.count << '\n';
    std::cout << "average: " << format_score(average(stats)) << '\n';
    std::cout << "lowest:  " << format_score(stats.min) << '\n';
    std::cout << "highest: " << format_score(stats.max) << '\n';
    std::cout << "A: " << letter_a << "  B: " << letter_b
              << "  C: " << letter_c << "  D: " << letter_d
              << "  F: " << letter_f << '\n';
    return 0;
}
```

The test file checks the boundaries, the overloads, the reference update and the out-of-range rule:

```cpp
// tests/grades_test.cpp
#include <cstdlib>
#include <iostream>
#include <stdexcept>

#include "grades.hpp"

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
    check(letter_grade(90) == 'A', "90 is an A");
    check(letter_grade(89) == 'B', "89 is a B");
    check(letter_grade(60) == 'D', "60 is a D");
    check(letter_grade(59) == 'F', "59 is an F");

    check(format_score(85) == "85", "int overload");
    check(format_score(84.6) == "84.6", "double overload");

    Stats stats;
    add_score(stats, 90);
    add_score(stats, 80);
    add_score(stats, 70);
    check(stats.count == 3, "count after three scores");
    check(stats.min == 70 && stats.max == 90, "min and max update by reference");
    check(average(stats) == 80.0, "average of 90, 80, 70");

    bool threw = false;
    try {
        letter_grade(101);
    } catch (const std::out_of_range&) {
        threw = true;
    }
    check(threw, "101 is rejected");

    if (failures == 0) {
        std::cout << "all grade tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the report, the test and the shared source file, and registers the test with CTest:

```cmake
cmake_minimum_required(VERSION 3.25)
project(grade_report LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_library(grades STATIC src/grades.cpp)
target_include_directories(grades PUBLIC src)

add_executable(grade_report src/main.cpp)
target_link_libraries(grade_report PRIVATE grades)

add_executable(grades_test tests/grades_test.cpp)
target_link_libraries(grades_test PRIVATE grades)

enable_testing()
add_test(NAME grades_test COMMAND grades_test)
```

Step-by-step tasks:

1. Create the files above, add `build/` to `.gitignore`, and initialize the repo with `git init -b main`. Commit with the message "Add grade report with functions".
2. Configure and build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug`, then `cmake --build build`. Fix any warnings before you continue.
3. Run `ctest --test-dir build --output-on-failure`. Confirm the test passes.
4. Run the report with input `95 84 72 40 61`. In PowerShell, use `"95 84 72 40 61" | .\build\grade_report.exe`. In Git Bash, use `echo "95 84 72 40 61" | ./build/grade_report`. Check that the count is 5, the average is 70.4, and the letter counts are A 1, B 1, C 1, D 1, F 1.
5. Run the report with input `85 abc` and record the error message and the exit code.
6. Create a branch `feature/bonus-points`. Write a test first for a new function `void apply_bonus(Stats& stats, int bonus)` that adds `bonus` to `stats.sum` and leaves `count`, `min` and `max` alone. Commit the failing test, then add the declaration and definition until the test passes.
7. Write `docs/overload-notes.md` with three short sections: where the overloads are used and why `85` and `84.6` pick different versions, what happens with an unsigned argument and why the compiler reports ambiguity, and one place where you chose pass by `const` reference and why.
8. Merge the branch with `git merge --no-ff feature/bonus-points -m "Add bonus points"` into `main`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes.
- The sample input prints the expected count, average and letter counts from task 4.
- The invalid input prints the error and exits with status 1.
- `docs/overload-notes.md` names the exact overload that each call used.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/overload-notes.md`, `README.md`, and the history on GitHub.

This project is light on new ideas and heavy on structure. A reviewer should be able to read `main.cpp` and see only a loop and a report, while every rule sits in a function that a test checks.

---

## Check your understanding

1. A `for` loop prints the numbers 1 to 9 when you wanted 1 to 10. Name two places in the loop header where the boundary can go wrong, and say how you would check each one.
2. You write two `if` statements instead of an `if` and an `else if`, and a score of 95 is counted twice. Explain why the second `if` still runs, and what the fix looks like.
3. A function `void normalize(std::string& text)` is called on a string the caller still needs unchanged. What does the caller see, and how would you change the signature so the function only reads the text?
4. Why does `format_score(85u)` fail to compile when `format_score(int)` and `format_score(double)` both exist? Name one change that fixes the call without changing the callers.
5. A function is declared to return `int` but one path through it reaches the end without a `return`. Explain why this is undefined behavior, and why `-Wall` should catch it.
6. Your team wants to pass a large configuration object to many functions. Choose between pass by value, pass by `const` reference and pass by non-const reference, and justify your choice in one sentence.

### Answers

1. The start value and the comparison. A start of 1 with `<` stops one value early, and `<=` includes the last value. Check by printing the first and last values by hand, or by writing a test that counts iterations.
2. An `if` does not know about the `if` before it, so each one is tested independently, and a score of 95 satisfies both conditions. Use `else if`, so the chain stops at the first match.
3. The caller sees the text changed, because the reference is an alias for the caller's variable. Change the parameter to `const std::string&`, which allows reads without copies and forbids changes.
4. Both overloads need a conversion from `unsigned int`, so neither is better and the call is ambiguous. Cast the argument at the call site, for example `format_score(static_cast<int>(85u))`, or add a third overload for `unsigned int`.
5. A function that promises a value must return one on every path. If it does not, the value is not defined, and the compiler is allowed to generate any behavior. `-Wreturn-type` in `-Wall` warns about it.
6. Pass by `const` reference, because it avoids copying the large object and prevents the functions from changing it, which protects the caller's data.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Control flow](https://en.wikipedia.org/wiki/Control_flow) for the general concept
- [Wikipedia: Subroutine](https://en.wikipedia.org/wiki/Subroutine) for the idea of functions across languages
- cppreference.com, the pages for "if statement", "switch statement", "for loop" and "Function declaration"
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on functions and the parameter passing rules
- Stanford CS106L course materials, from the Stanford CS106L website, for a university treatment of functions and references

*Tutorials*

- learncpp.com, the chapters on conditional statements, loops, functions, overloading and pass by reference, written by Alex Allain
- GeeksforGeeks, search the site for "C++ functions" and "C++ function overloading" for the matching articles
- W3Schools, "C++ Functions" and "C++ Control Flow" pages, for quick examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on functions and references
- CS50 on YouTube, the CS50x lectures on loops and functions, for the same ideas in C

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, with the exercises on loops and functions. Run each exercise's test suite from your terminal before you move on.

---

## Tomorrow

Day 30 moves from single values to collections. You will store many items in `std::vector` and `std::array`, read and change `std::string` values, walk them with iterators, and use algorithms such as `std::sort` and `std::find_if`. The functions and reference rules from today decide how those collections are passed around, and the grade report's `Stats` record is the first step toward a container that holds every score.

**Next:** [Day 30: Arrays, Strings and Containers](Software-Day-30-Arrays-Strings-and-Containers.md)

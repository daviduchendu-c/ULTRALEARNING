# Day 38: Errors, Debugging and Testing

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 5 hours
**Level:** intermediate. You have written classes, containers and smart pointers from Days 32 to 37. This day adds the discipline that keeps that code correct: how to report errors without losing the cleanup guarantees, how to find a bug with a debugger and sanitizers, and how to lock behavior in place with unit tests. It is for anyone who has been fixing bugs by adding print statements and hoping.

**By the end of this day, you will have:**

- Thrown and caught exceptions with a custom error type that carries useful context, and explained why RAII keeps resources safe while the stack unwinds
- Used `assert` for conditions that indicate a programmer error, and explained why it differs from an exception
- Stepped through a program in a debugger, set a breakpoint and inspected a variable
- Built a project with AddressSanitizer and UndefinedBehaviorSanitizer, and read a sanitizer report that points to a bug
- A safe configuration parser with GoogleTest tests registered in CTest, plus a sanitizer build option
- A GitHub repo with the test output, a sanitizer report from a deliberate bug and a short note on each error strategy you used

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0 on Windows), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Ubuntu 24.04 in WSL 2 with `sudo apt install g++-14 gdb` for the sanitizer and debugger exercises. AddressSanitizer and UndefinedBehaviorSanitizer come with GCC on Linux.
- Git 2.40 or newer and a terminal
- Internet access at configure time, because CMake downloads GoogleTest 1.17.0 from GitHub
- A free GitHub account

**Prerequisites:**

- Days 27 to 37: the toolchain, classes, lifetime, RAII, containers, templates and smart pointers
- Day 31: pointers and the dangling pointer rule

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Exceptions](#exceptions)
- [Assertions](#assertions)
- [The debugger](#the-debugger)
- [Sanitizers](#sanitizers)
- [Unit testing](#unit-testing)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a safe configuration parser with tests and sanitizers](#project-a-safe-configuration-parser-with-tests-and-sanitizers)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Software fails in three ways. Some failures are expected, such as a user typing a word where a number belongs. Some are bugs in your own logic, such as an index one past the end. Some are bugs in the environment, such as a disk that is full. Each kind needs a different tool. Exceptions handle expected failures at a boundary, assertions catch broken assumptions during development, and sanitizers and debuggers find the bugs that the compiler cannot see.

Testing ties all of these together. A test states what the code must do, and it runs every time you change the code. Without tests, each change is a guess. With tests, a failing check tells you exactly which rule broke. This day adds a unit test framework to the project, so the rules you wrote in earlier days become executable.

The rules for exceptions and assertions are in the ISO C++ standard and in the GCC manual for the sanitizers. cppreference.com has pages for exceptions and for `assert`, and the GoogleTest documentation on GitHub covers the test framework.

---

## Glossary

- **Exception:** a signal that an operation cannot complete normally. C++ moves the program to the nearest matching `catch` block.
- **Throw:** the statement that raises an exception, written `throw value;`.
- **Catch:** a block that handles an exception of a given type.
- **Stack unwinding:** the process of destroying local objects as an exception leaves each function, which runs destructors and lets RAII release resources.
- **Exception safety:** a guarantee about what state an object is left in after an exception. The basic guarantee keeps the object valid. The strong guarantee also leaves it unchanged.
- **Assertion:** a check, written `assert(condition)`, that stops the program when a condition that must be true is false. Assertions are removed when `NDEBUG` is defined.
- **Breakpoint:** a marker that pauses a running program in the debugger.
- **Sanitizer:** an instrumentation that the compiler adds to detect a class of bugs at run time, such as memory errors or undefined behavior.
- **AddressSanitizer:** a sanitizer that detects out-of-bounds access, use-after-free and leaks. It is enabled with `-fsanitize=address`.
- **UndefinedBehaviorSanitizer:** a sanitizer that detects undefined behavior such as signed overflow. It is enabled with `-fsanitize=undefined`.
- **Unit test:** a test that checks one function or class in isolation, with a clear expected result.
- **Test fixture:** a shared setup that several tests in one group use.
- **GoogleTest:** a C++ unit test framework from Google, which provides `TEST`, `EXPECT_EQ` and `EXPECT_THROW`.

---

## Core concepts

The concepts split into two groups. Exceptions and assertions are ways to report and handle problems inside code. The debugger, sanitizers and unit tests are ways to find and prevent problems outside it.

### Exceptions

An exception is the way C++ reports that a function cannot do its job. Think of a fire alarm in a building. When it sounds, everyone stops and leaves by the nearest exit, and the people who know the building's procedures handle the emergency at the right floor. The `throw` statement sounds the alarm, and the `catch` block is the procedure for a particular kind of alarm.

In first principles, `throw` creates an object and starts searching up the call stack for a `catch` block whose type matches. Each function on the way is exited, and its local objects are destroyed. That destruction is what makes RAII work. A file opened by a guard closes, and a `unique_ptr` frees its object, even though the code that would have done it never ran.

```cpp
#include <iostream>
#include <initializer_list>
#include <stdexcept>
#include <string>

int parse_age(const std::string& text) {
    const int age = std::stoi(text);  // throws std::invalid_argument or std::out_of_range
    if (age < 0) {
        throw std::domain_error("age cannot be negative");
    }
    return age;
}

int main() {
    for (const std::string input : {"30", "-2", "abc"}) {
        try {
            const int age = parse_age(input);
            std::cout << "age " << age << '\n';
        } catch (const std::invalid_argument&) {
            std::cout << "not a number: " << input << '\n';
        } catch (const std::domain_error& error) {
            std::cout << "rejected: " << error.what() << '\n';
        }
    }
    return 0;
}
```

The output is `age 30`, then `rejected: age cannot be negative`, then `not a number: abc`. The value is computed before anything is printed, so a failed call leaves no partial line behind. The catch blocks are checked in order, and the first one whose type matches runs. Catch by const reference, so the exception object is not copied and derived types are not sliced. The standard library throws its own exception types, which derive from `std::exception`, and `what()` gives a readable message.

The design question is what to throw. Throw a type that carries the context a caller needs. The project defines `ConfigError`, which derives from `std::runtime_error` and records the line number. A caller can then print the line without parsing a message string.

The common mistake is to catch an exception and do nothing, which hides the failure and lets the program continue with bad state. The other mistake is to throw from a destructor, which can end the program during unwinding, as Day 33 explained. You can spot a swallowed error when a program behaves strangely and logs nothing. In the market, services report exceptions to logging systems at the boundary, where the caller can act, and they let destructors release resources quietly.

### Assertions

An assertion checks a condition that must be true if the code is correct. Think of a smoke detector on a stove. It only signals that something has already gone wrong, so you stop and find out why. An `assert` expresses a programmer's promise, such as "the caller checked that the list is not empty." If the promise is broken, the program stops during development, where the bug is cheap to find.

`assert` is removed when `NDEBUG` is defined, which is the default for release builds in CMake. That is why it is for bugs. A user's bad input must be handled with an exception or an error value, because an assertion may not run in production.

```cpp
#include <cassert>
#include <iostream>
#include <vector>

int average_of(const std::vector<int>& values) {
    assert(!values.empty() && "average_of needs at least one value");
    int total = 0;
    for (const int value : values) {
        total += value;
    }
    return total / static_cast<int>(values.size());
}

int main() {
    std::cout << average_of({4, 6, 8}) << '\n';
    return 0;
}
```

The output is `6`. If you call `average_of({})`, a Debug build stops with an assertion message that names the condition and the file and line. A Release build has no assertion, and the division by zero is undefined behavior. That is the trade-off: assertions find bugs early, and they must not be the only defense against bad input. The `&&` with a string in the assertion is a common idiom that puts a readable message next to the condition.

The common mistake is to put side effects inside an assertion, such as `assert(queue.pop())`, which vanish in release builds and change the behavior. Keep assertions free of side effects. The second mistake is to use `assert` for a condition that a user can cause. In the market, safety-critical systems use assertions heavily during development and still handle input errors explicitly.

### The debugger

A debugger runs your program under control. You can pause it, step one line at a time, look at variables and change their values. Think of a mechanic who stops an engine and checks each part instead of guessing from the noise. The debugger is the tool for bugs whose cause is not obvious from the output.

The GNU debugger, `gdb`, works on Linux and in WSL. The steps are to compile with debug symbols, which the Debug build type does with `-g`, start the program under `gdb`, set a breakpoint and run:

```text
gdb ./build/bug_demo
(gdb) break main
(gdb) run
(gdb) next
(gdb) print values[1]
(gdb) backtrace
(gdb) quit
```

The commands are `break` to set a breakpoint, `run` to start, `next` to step over a line, `print` to show a value and `backtrace` to show the call stack. On Windows, Visual Studio and VS Code with the C/C++ extension offer the same actions through buttons and a variables panel.

The program below has a bug that the debugger can show. The loop runs one step past the end of the vector:

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> values{3, 1, 4};
    int sum = 0;
    for (std::size_t i = 0; i <= values.size(); ++i) {  // bug: <= reads past the end
        sum += values[i];
    }
    std::cout << sum << '\n';
    return 0;
}
```

Set a breakpoint on the `sum += values[i]` line and step through the loop. When `i` is 3, `values.size()` is also 3, and `values[3]` reads memory the vector does not own. The debugger shows `i` and the size side by side, which makes the bug visible. The sanitizer section shows the same bug reported automatically.

What you see on screen depends on the library. Some builds read the garbage value silently, and others stop at once. The WinLibs GCC 16 build used to check this lesson enables libstdc++ bounds assertions by default, so the program aborts with `Assertion '__n < this->size()' failed` before it prints anything. You can ask for that check on any GCC build with `-D_GLIBCXX_ASSERTIONS`. Either way, the loop is wrong, and the fix is `i < values.size()`.

The common mistake is to stop the debugger at the wrong point, such as after the bad value was already written. Set the breakpoint before the state goes bad, and use watchpoints for a variable that changes unexpectedly. In the market, debugging core dumps and live services uses the same tools, with more time spent on the stack traces and less on stepping.

### Sanitizers

A sanitizer is a bug finder built into the compiler. It adds checks to your program, and it reports the first time a bad operation runs. Think of a metal detector at a building entrance. It does not stop every problem, but it finds the common ones on the way in, and it points to the place where the problem sits. The sanitizers in GCC and Clang catch memory errors and undefined behavior that the compiler cannot prove at compile time.

AddressSanitizer, enabled with `-fsanitize=address`, detects reads and writes outside allocated memory, use after free, and leaks at exit. UndefinedBehaviorSanitizer, enabled with `-fsanitize=undefined`, detects signed integer overflow, invalid shifts, misaligned access and several other kinds. Both add a small slowdown, which is acceptable for tests and not for production builds.

Build the bug program from the debugger section with AddressSanitizer in WSL:

```bash
g++-14 -std=c++20 -g -fsanitize=address -fno-omit-frame-pointer bug_demo.cpp -o bug_demo
./bug_demo
```

The report begins with a line such as `ERROR: AddressSanitizer: heap-buffer-overflow`, then the operation that failed, the line of code and the memory layout. The report points to the exact line, which is faster than any print statement. The output of the program does not appear, because the sanitizer stops the program at the first error.

The common mistake is to run the tests only in the normal build and assume the code is clean. A sanitizer finds bugs that tests miss, because it watches every access, not only the ones the test checks. The other mistake is to disable a sanitizer report instead of fixing the bug. In the market, sanitizer builds run in continuous integration for most native projects, and the reports are blocking.

### Unit testing

A unit test states one expected behavior and checks it. Think of a set of measuring tapes that you check against a standard length before you cut a board. Tests check only the rules you name, not the whole program. Each run shows that those rules still hold after a change, which is what makes refactoring safe.

GoogleTest provides the macros that make tests readable. `TEST(Suite, Name)` declares a test, `EXPECT_EQ(a, b)` checks equality and continues on failure, `ASSERT_TRUE(c)` stops the test on failure, and `EXPECT_THROW(statement, type)` checks that a type of exception is thrown. The `gtest_discover_tests` command in CMake registers each test with CTest, so `ctest` runs them all.

```cpp
#include <gtest/gtest.h>

#include "config.hpp"

TEST(ParseConfig, ReadsKeyValuePairs) {
    const auto config = parse_config("port = 8080\nworkers = 4\n");
    EXPECT_EQ(config.at("port"), 8080);
    EXPECT_EQ(config.at("workers"), 4);
}

TEST(ParseConfig, NonNumericValueThrows) {
    EXPECT_THROW(parse_config("port = abc\n"), ConfigError);
}
```

The common mistake is to write tests that check the implementation instead of the behavior, which break every time the code is rearranged. Test what the function promises, and keep each test to one rule. The second mistake is to write tests after a bug is found and not before. Write the failing test first, as Day 7 and Day 29 practiced, and then fix the code. In the market, unit tests run on every pull request, and teams that keep them fast and focused catch regressions before code is merged.

---

## Hands-on exercises

Work in a new folder called `errors-debugging-testing`. Run `git init -b main` in it.

Exercise 1 throws and catches.

1. Write a function that divides two integers and throws `std::domain_error` when the divisor is zero.
2. Call it from `main` inside a `try` block and print the message from `what()`.
3. Add a `std::unique_ptr` local in the function before the throw, and print a message from a destructor of a small class you wrote, to see the order.

Done when the error message prints and the destructor message appears before the catch message.

Exercise 2 uses an assertion.

1. Write a function that takes a `std::vector<int>` and returns its first element, with an `assert` that the vector is not empty.
2. Build it in Debug and call it with an empty vector. Record the assertion message.
3. Build in Release with `-DNDEBUG`, call it again, and note what happens instead.

Done when you can explain why the assertion and the exception are both needed.

Exercise 3 debugs the off-by-one program.

1. Compile the bug program from the debugger section in WSL with `g++-14 -std=c++20 -g`.
2. Run `gdb ./bug_demo`, set a breakpoint on `main`, and step with `next` until the loop starts. Print `i` and `values.size()` at each step.
3. Stop when `i` equals 3 and record what you see.

Done when your notes show the values of `i` and the size at the moment the loop reads past the end.

Exercise 4 runs a sanitizer.

1. Compile the bug program with `-fsanitize=address -g` and run it.
2. Read the report and copy the line number it names.
3. Fix the loop to use `<` and run again under the sanitizer. Confirm the report is gone.

Done when the bug line in the report matches the loop, and the fixed program runs clean.

Exercise 5 writes the first failing test.

1. Write a GoogleTest test for a function `int clamp_port(int)` that returns 1 for values below 1, 65535 for values above 65535, and the value itself otherwise. Write the test before the function.
2. Build and run it, and confirm it fails because the function does not exist yet or returns the wrong value.
3. Write the function and confirm the test passes.

Done when the test was red before the function existed and green after.

---

## Project: a safe configuration parser with tests and sanitizers

The project parses a simple configuration format with lines of `key = value`. Blank lines and lines starting with `#` are skipped. A bad line throws a `ConfigError` that carries the line number. The GoogleTest suite checks the parser, the port validation and the exception types, and an option turns on both sanitizers.

Suggested repo name: `100-days-software-day-38-errors-debugging-testing`.

The repo layout:

```
100-days-software-day-38-errors-debugging-testing/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── error-strategy.md
├── src/
│   ├── config.cpp
│   ├── config.hpp
│   └── main.cpp
└── tests/
    └── config_test.cpp
```

The header declares the error type and the parser. The exception carries the line number, which is useful context for the caller:

```cpp
// src/config.hpp
#ifndef CONFIG_HPP
#define CONFIG_HPP

#include <cstddef>
#include <map>
#include <stdexcept>
#include <string>

class ConfigError : public std::runtime_error {
public:
    ConfigError(std::size_t line, const std::string& message);
    std::size_t line() const { return line_; }

private:
    std::size_t line_;
};

// Parses "key = value" lines. Blank lines and lines starting with '#' are skipped.
// Throws ConfigError with the line number for any malformed line.
std::map<std::string, int> parse_config(const std::string& text);

bool is_valid_port(int port);

// Returns the port stored under key. The caller must check that the key exists.
int port_from(const std::map<std::string, int>& config, const std::string& key);

#endif
```

The implementation separates the two kinds of failure. Bad text throws `ConfigError`, which the caller can handle. A missing key passed to `port_from` is a programmer error, so it is an assertion, and an out-of-range port is bad data, so it throws `std::out_of_range`:

```cpp
// src/config.cpp
#include "config.hpp"

#include <cassert>
#include <exception>
#include <stdexcept>
#include <utility>

namespace {

std::string trim(const std::string& text) {
    const auto first = text.find_first_not_of(" \t\r");
    if (first == std::string::npos) {
        return "";
    }
    const auto last = text.find_last_not_of(" \t\r");
    return text.substr(first, last - first + 1);
}

}  // namespace

ConfigError::ConfigError(std::size_t line, const std::string& message)
    : std::runtime_error("line " + std::to_string(line) + ": " + message), line_(line) {}

std::map<std::string, int> parse_config(const std::string& text) {
    std::map<std::string, int> config;
    std::size_t line_number = 0;
    std::size_t start = 0;

    while (start <= text.size()) {
        const std::size_t end = text.find('\n', start);
        const std::string raw =
            text.substr(start, end == std::string::npos ? std::string::npos : end - start);
        ++line_number;

        const std::string line = trim(raw);
        if (!line.empty() && line[0] != '#') {
            const auto equals = line.find('=');
            if (equals == std::string::npos) {
                throw ConfigError(line_number, "missing '='");
            }

            const std::string key = trim(line.substr(0, equals));
            const std::string value_text = trim(line.substr(equals + 1));
            if (key.empty()) {
                throw ConfigError(line_number, "empty key");
            }
            if (config.contains(key)) {
                throw ConfigError(line_number, "duplicate key '" + key + "'");
            }

            std::size_t consumed = 0;
            int value = 0;
            try {
                value = std::stoi(value_text, &consumed);
            } catch (const std::exception&) {
                throw ConfigError(line_number, "value is not a number");
            }
            if (consumed != value_text.size()) {
                throw ConfigError(line_number, "trailing characters after the number");
            }
            config.emplace(key, value);
        }

        if (end == std::string::npos) {
            break;
        }
        start = end + 1;
    }
    return config;
}

bool is_valid_port(int port) {
    return port >= 1 && port <= 65535;
}

int port_from(const std::map<std::string, int>& config, const std::string& key) {
    assert(config.contains(key) && "caller must check that the key exists");
    const int port = config.at(key);
    if (!is_valid_port(port)) {
        throw std::out_of_range("port out of range for key '" + key + "'");
    }
    return port;
}
```

The demo parses one good file and one bad file, and it prints the error with its line number:

```cpp
// src/main.cpp
#include <iostream>
#include <string>

#include "config.hpp"

int main() {
    try {
        const auto good = parse_config("port = 8080\nworkers = 4\n");
        std::cout << "parsed: port=" << good.at("port") << " workers=" << good.at("workers") << '\n';

        parse_config("port = 8080\nworkers 4\n");
    } catch (const ConfigError& error) {
        std::cout << "error: " << error.what() << '\n';
        std::cout << "on line: " << error.line() << '\n';
    }
    return 0;
}
```

The GoogleTest file checks the parser rules, each error type and the port helpers:

```cpp
// tests/config_test.cpp
#include <gtest/gtest.h>

#include <stdexcept>

#include "config.hpp"

TEST(ParseConfig, ReadsKeyValuePairs) {
    const auto config = parse_config("port = 8080\nworkers = 4\n");
    EXPECT_EQ(config.at("port"), 8080);
    EXPECT_EQ(config.at("workers"), 4);
}

TEST(ParseConfig, SkipsCommentsAndBlankLines) {
    const auto config = parse_config("# a comment\n\nport=9000\n");
    EXPECT_EQ(config.size(), 1u);
    EXPECT_EQ(config.at("port"), 9000);
}

TEST(ParseConfig, MissingEqualsReportsItsLine) {
    try {
        parse_config("port 8080\n");
        FAIL() << "expected a ConfigError";
    } catch (const ConfigError& error) {
        EXPECT_EQ(error.line(), 1u);
    }
}

TEST(ParseConfig, NonNumericValueThrows) {
    EXPECT_THROW(parse_config("port = abc\n"), ConfigError);
}

TEST(ParseConfig, TrailingCharactersThrow) {
    EXPECT_THROW(parse_config("port = 80x\n"), ConfigError);
}

TEST(ParseConfig, DuplicateKeyThrows) {
    EXPECT_THROW(parse_config("a=1\na=2\n"), ConfigError);
}

TEST(PortCheck, AcceptsTheEdges) {
    EXPECT_TRUE(is_valid_port(1));
    EXPECT_TRUE(is_valid_port(65535));
}

TEST(PortCheck, RejectsOutOfRange) {
    EXPECT_FALSE(is_valid_port(0));
    EXPECT_FALSE(is_valid_port(70000));
}

TEST(PortFrom, ThrowsForOutOfRangeData) {
    const auto config = parse_config("port = 70000\n");
    EXPECT_THROW(port_from(config, "port"), std::out_of_range);
}
```

The `CMakeLists.txt` fetches GoogleTest, defines the warning policy as an interface target that only the project's own targets use, and adds a sanitizer option. The warning interface keeps GoogleTest's own build from failing on warnings it does not control:

```cmake
cmake_minimum_required(VERSION 3.25)
project(safe_config LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(ENABLE_SANITIZERS "Build with AddressSanitizer and UndefinedBehaviorSanitizer" OFF)

if(ENABLE_SANITIZERS AND NOT MSVC)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=address,undefined)
endif()

add_library(project_warnings INTERFACE)
target_compile_options(project_warnings INTERFACE
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.17.0
    GIT_SHALLOW TRUE
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

add_library(config_core STATIC src/config.cpp)
target_include_directories(config_core PUBLIC src)
target_link_libraries(config_core PRIVATE project_warnings)

add_executable(config_demo src/main.cpp)
target_link_libraries(config_demo PRIVATE config_core project_warnings)

enable_testing()
add_executable(config_test tests/config_test.cpp)
target_link_libraries(config_test PRIVATE config_core project_warnings GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(config_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/`, `build-asan/` and `docs/*.log` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add configuration parser with tests".
2. On Windows, build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. The first configure downloads GoogleTest, so it needs internet access. Run `ctest --test-dir build --output-on-failure` and confirm all tests pass.
3. Run `./build/config_demo` and check that it prints `parsed: port=8080 workers=4`, then the error on line 2.
4. In WSL, configure with `cmake -S . -B build-asan -G Ninja -DENABLE_SANITIZERS=ON -DCMAKE_BUILD_TYPE=Debug` and run `ctest --test-dir build-asan --output-on-failure`. Confirm there are no sanitizer reports.
5. Create a branch `feature/env-override`. Write a failing test first for a function `int port_or_default(const std::map<std::string, int>& config, const std::string& key, int fallback)` that returns the fallback when the key is missing and the stored port when it exists. Then write the function and make the test pass.
6. Add a deliberate bug on a scratch branch: change `config.contains(key)` to a copy that skips the check. Run the sanitizer build and the test, and record which tool reports a problem. Then discard the scratch change without committing it.
7. Write `docs/error-strategy.md` with three short sections: where you used exceptions and why, where you used an assertion and why, and one bug you found with a sanitizer or the debugger, with the report line that named it.
8. Merge with `git merge --no-ff feature/env-override -m "Add port fallback"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes on Windows, and the WSL sanitizer build reports nothing.
- The demo prints the two lines from task 3.
- `docs/error-strategy.md` names one real sanitizer or debugger finding from your runs.
- The GitHub repo shows the merge commit on `main`, with no scratch changes committed.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/error-strategy.md`, `README.md`, and the history on GitHub.

This project is the first one where tests and sanitizers decide whether the work is done. A reviewer should see the test names as a list of promised behaviors, and a sanitizer run that is clean.

---

## Check your understanding

1. A function throws an exception while holding a `std::unique_ptr` and a file handle. Explain which resources are released, in what order, and why the function needs no cleanup code.
2. A developer replaces `assert(config.contains(key))` with `if (!config.contains(key)) return 0;`. Explain what changes in Debug and Release builds, and which one is more appropriate for user input.
3. A program prints the correct output and runs cleanly in tests, but a sanitizer reports a heap buffer overflow. Explain why testing can miss the bug and what the sanitizer watches.
4. A test checks `port_from` by calling the function with a valid config and comparing the result. Describe one behavior it does not cover, and write the test that covers it.
5. Why does `catch (const std::exception& error)` catch a `ConfigError` as well as a `std::out_of_range`, and what would be lost by catching `std::exception` and not the specific types?
6. Why is it useful to write a test before the function it checks, and what does a test that never fails tell you?

### Answers

1. The `unique_ptr` deletes its object and the file handle closes, in the reverse order of their construction. Each object's destructor performs its cleanup as the stack unwinds past the function, so the function needs no try block or cleanup line.
2. In Debug, the assertion stops the program on a broken promise, which catches a programmer error. In Release, the assertion is removed, and the early return hides the problem. For user input, neither is right alone: the check should be a real error path such as an exception or an error value, because user input is not a programmer error.
3. Tests check only the paths they exercise, and an overflow may happen on a path the test never runs, or it may read memory that happens to hold the right value. The sanitizer instruments every memory access and reports the first one that leaves an allocation, even when the output looks correct.
4. The test does not cover an out-of-range port, which should throw `std::out_of_range`. A test for it is `EXPECT_THROW(port_from(parse_config("port = 0\n"), "port"), std::out_of_range);`.
5. `ConfigError` derives from `std::runtime_error`, which derives from `std::exception`, so the broader catch matches it. Catching `std::exception` loses the specific handling for each type, such as the line number from `ConfigError`, and it can hide bugs by catching errors the caller did not expect.
6. Writing the test first shows that the test can fail, which proves it checks something real. A test that never fails, even when the function is wrong, checks nothing, so it gives false confidence and should be rewritten.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Exception handling](https://en.wikipedia.org/wiki/Exception_handling) for the general concept
- [Wikipedia: Assertion (software development)](https://en.wikipedia.org/wiki/Assertion_(software_development)) for the role of assertions
- [Wikipedia: Debugger](https://en.wikipedia.org/wiki/Debugger) and [Wikipedia: GNU Debugger](https://en.wikipedia.org/wiki/GNU_Debugger) for the tools in this day
- [Wikipedia: Unit testing](https://en.wikipedia.org/wiki/Unit_testing) for the testing practice
- cppreference.com, the pages for "Exceptions", "assert" and "std::exception"
- The GCC manual, sections "Instrumentation Options" for `-fsanitize=address` and `-fsanitize=undefined`
- The GoogleTest primer and advanced guide in the googletest repository on GitHub, at github.com/google/googletest

*Tutorials*

- learncpp.com, the chapters on exceptions, assertions and debugging, written by Alex Allain
- GeeksforGeeks, search the site for "exception handling in C++" and "assert in C++" for the matching articles
- Microsoft Learn, search learn.microsoft.com for "Debug a C++ program in Visual Studio" for the Windows debugger view

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on exceptions and debugging
- CppCon, the YouTube channel, for talks on sanitizers and testing. Search the channel for "AddressSanitizer" to find them.

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, which uses GoogleTest-style test suites. Run each exercise under the sanitizer build to see clean and dirty runs side by side.
- GDB tutorial on the GNU website, search gnu.org for the GDB manual's "Sample Session" section, and practice the steps on the bug program.

---

## Tomorrow

Day 39 covers CMake and project structure. You used CMake from Day 27 and built targets, linked libraries and fetched GoogleTest today. Day 39 explains the target model in depth: library and executable targets, include paths that are public or private, dependencies from packages and FetchContent, presets for Debug and Release, and a GitHub Actions workflow that runs the build and the tests on every push. The sanitizer option from today becomes one preset that the workflow runs.

**Next:** [Day 39: CMake and Project Structure](Software-Day-39-CMake-and-Project-Structure.md)

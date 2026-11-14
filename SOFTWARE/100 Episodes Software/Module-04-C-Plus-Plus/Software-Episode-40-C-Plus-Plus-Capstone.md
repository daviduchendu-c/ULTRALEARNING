# Day 40: C++ Capstone

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 8 hours, spread over two sittings
**Level:** intermediate. You have finished the module's concepts from Day 27 to Day 39. This day is the project that brings them together: a command-line expense ledger with a library, an application, tests, sanitizers and a continuous integration workflow. It is for anyone who wants proof that they can build and ship a small C++ program, not only pass small exercises.

**By the end of this day, you will have:**

- Designed a small C++ program as a library and an application, with a clear public interface
- Used RAII for the file storage, smart pointers for ownership, containers for totals and lists, and exceptions for bad input
- Tested the parser, the formatter, the ledger rules and the file round trip with GoogleTest
- Built the project with Debug, Release and sanitizer presets, and read a sanitizer report from a deliberate bug
- Run the workflow from Day 39 on a fresh repository and kept it green on `main`
- A public GitHub repo with a README, a design note, a CLI demo and a workflow badge

**What you need:**

- Everything from Day 39: CMake 3.25 or newer (checked with CMake 4.3.3), Ninja 1.11 or newer, GCC 14 or newer (checked with g++ 16.1.0 on Windows), and Ubuntu 24.04 in WSL 2 with `sudo apt install g++-14 ninja-build`
- Git 2.40 or newer, and a free GitHub account
- Internet access for `FetchContent` and GitHub Actions

**Prerequisites:**

- Days 27 to 39. You should be able to write a class, a test and a CMake target without copying a template you do not understand.

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Designing the program](#designing-the-program)
- [Money and records](#money-and-records)
- [Storage and ownership](#storage-and-ownership)
- [The ledger and its rules](#the-ledger-and-its-rules)
- [Hands-on exercises](#hands-on-exercises)
- [Project: an expense ledger with tests, presets and CI](#project-an-expense-ledger-with-tests-presets-and-ci)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A capstone is the point where separate skills have to work together. Each earlier day gave you one tool: the compiler and the build, types and functions, classes, pointers, lifetime, containers, templates, smart pointers, errors and tests, and the CMake structure. A real program uses all of them at once, and most of the work is deciding where each tool belongs. This day is that work, in one project that you can finish in a weekend.

The program is an expense ledger. You add an expense with a date, a category, an amount and a description. The ledger stores the records in a CSV file, lists them, and totals them by category. The problem is small enough to read in one sitting, and it has the parts that real software has: input that can be wrong, money that must be exact, a file that must be read and written safely, and rules that must always hold.

Work through the design section first, then build the project in the order of the tasks. When the tests pass and the workflow is green, you have a repository that a stranger can clone, build and check in one command. That is the bar for the portfolio pieces this series is building toward.

The language rules come from the ISO C++ standard. The reference pages on cppreference.com cover every standard type used here, and the CMake and GitHub Actions documentation covers the build and the workflow.

---

## Glossary

- **Capstone:** a final project that combines the skills from a course.
- **Ledger:** a record of transactions, here a list of expenses with a total.
- **Record:** one row of stored data, here one expense.
- **Parser:** code that turns text into structured values, here a CSV line into an `Entry`.
- **Formatter:** code that turns values into text for people, here cents into `12.34`.
- **Storage:** the place a ledger's records live between runs, here a file or an in-memory list.
- **Abstract interface:** a base class with pure virtual functions, which a program uses to swap one storage for another.
- **Round trip:** writing a value out and reading it back, which must give the same value.
- **Invariant:** a rule that is true for every valid ledger, such as no negative amounts and no commas inside fields.

---

## Core concepts

The concepts follow the order of the design. You first decide the shape of the program, then the value types, then the storage and ownership, and finally the rules that tie them together.

### Designing the program

A design is a set of decisions written down before the code grows. Think of an architect's plan for a small house. The plan says where the walls go, which rooms connect, and where the plumbing runs. It leaves the paint color for later, and it can change as the builders learn more. A program design says which parts are libraries, which are applications, which functions are public, and which rules the library owns.

The design for this project splits the code in three. The library `ledger` holds the rules and the data types. The application `ledger_cli` reads the command line and calls the library. The tests use the library directly. The application contains no rules of its own. If a rule changes, such as allowing negative refunds, the change happens in the library, and the tests show it.

The public interface is the part of the library that the application and the tests use. It is small on purpose: an `Entry` type, the functions that parse and format amounts and records, a `Storage` interface, and a `Ledger` class with a handful of methods. Everything else is private to the library's source files.

The common mistake is to put the logic in `main`, where it cannot be tested without running the program. The fix is the split above. The second mistake is to make every helper public. A public helper becomes a promise that must be kept forever. In the market, the difference between a clean public interface and an accidental one decides how long a library can change safely, which is why review of the header is usually the first step in a code review.

### Money and records

Money needs exact arithmetic. Binary floating point cannot represent 0.10 exactly, so adding ten expenses of 0.10 can give a total that is a fraction of a cent off. The fix from Day 28 applies here: store amounts as whole cents in an `std::int64_t`, and convert to text only when a person reads it. The `Entry` struct stores the date, the category, the amount in cents and the description, and the amount type makes the unit explicit in the field name.

The text form of a record is a line of comma-separated values, such as `2026-10-31,food,1250,lunch with team`. The format is simple enough to read by eye and to parse with `std::string_view`, which avoids copies of the line. A parser must check every field. A date must have the shape `YYYY-MM-DD`, an amount must be a non-negative whole number of cents, and a line must have four fields. The parser throws `std::invalid_argument` with a short reason, and the ledger converts that into an error with the line number.

The user types amounts such as `12.5` or `12.50`. The function `parse_amount` accepts whole numbers, one decimal place and two decimal places, and returns an optional value. An invalid amount returns nothing, which the command-line program turns into a message. This is the Day 37 `std::optional` pattern in use: a missing result is normal for user input, so the function says so in its return type.

The common mistake is to round in the middle of a calculation, or to parse a float and then multiply by 100 and round. Both lose exactness at the edges. Parse the text directly into cents, as this project does. In the market, financial software stores money as integers in the smallest currency unit, and reviewers reject floating-point money at once.

### Storage and ownership

The ledger does not know where its records live. It talks to an abstract `Storage` interface with two functions: read the lines, and write the lines. The interface is the Day 34 pattern. The application gives the ledger a `FileStorage`, which uses `std::ifstream` and `std::ofstream` to read and write `ledger.csv`. The tests give the ledger a `MemoryStorage`, which keeps the lines in a vector, so the tests run without touching the disk.

Ownership is explicit. The ledger holds its storage in a `std::unique_ptr<Storage>`, so exactly one owner exists, and the storage is destroyed with the ledger. The `FileStorage` itself uses RAII: its streams open in a function and close when the function returns, on every path, including when an error is thrown. Day 32 explained why this matters, and the file handling in this project is the same idea with the standard streams.

A test that needs to look inside the storage keeps a raw pointer to it, and it does not own the storage. The pointer is valid only while the ledger is alive, which the test arranges by keeping the ledger in scope. That is the non-owning pointer rule from Day 31, applied to a real test.

The common mistake is to pass the file path into every function, so the rest of the code depends on the file system. The fix is the interface, which keeps the file behind one class. The second mistake is to ignore a failed write. The `FileStorage` checks the stream after the write and closes it before returning. In the market, storage interfaces like this one are how teams test code that touches disks, databases or network services without running those services.

### The ledger and its rules

The `Ledger` class owns the rules. It refuses an entry with a bad date, a comma inside a field, an empty category or a negative amount. It keeps the entries in a vector in the order they were added. It totals all entries, and it totals them by category in a `std::map`, which keeps the categories sorted for the listing. Each rule is a single check in one function, so a test can name the rule that failed.

The rules are checked in two places. `add` checks the entry before it is stored, so a bad entry never reaches the list. `load` checks each line as it reads, and it reports the first bad line with its number, so the user can fix the file. The two checks share the parser in `entry.cpp`, which keeps the rule for dates in one place.

The ledger is the part of the design that the tests exercise most. The tests add entries, check the totals, save to memory, load them back, feed in a bad line and check the line number, and try to add an invalid entry. Each test states one rule, which makes a failure easy to read.

The common mistake is to write the rules in the command-line code, where the tests cannot reach them, and then to write a second copy of the rules in the library. The fix is one place for each rule. The second mistake is to validate only on load and not on add, which lets a bad record into memory and writes it to the file. In the market, the library that owns the rules is the one that every interface, whether a command line, a web service or a mobile app, must call.

---

## Hands-on exercises

Work in a new folder called `expense-ledger`. Run `git init -b main` in it. Each exercise adds one piece, and the tests will tell you when a piece is wrong.

Exercise 1 writes the value types.

1. Create `include/ledger/entry.hpp` with the `Entry` struct and the five function declarations from the project.
2. Create `src/entry.cpp` and implement `parse_amount` and `format_cents` first. Compile with `-Wall -Wextra -Wpedantic -Werror`.
3. Check by hand: `parse_amount("12.5")` must give 1250, and `format_cents(5)` must give `0.05`.

Done when both functions compile with no warnings and the two checks match.

Exercise 2 writes the CSV parser.

1. Implement `to_csv`, `from_csv` and `is_valid_date`.
2. Round trip one entry through `to_csv` and `from_csv` and compare each field.
3. Feed `from_csv` a line with three fields and record the exception message.

Done when the round trip keeps every field and the three-field line throws `std::invalid_argument`.

Exercise 3 writes the storage interface.

1. Create `include/ledger/storage.hpp` with the abstract `Storage` class, `MemoryStorage` and `FileStorage`.
2. Implement `FileStorage` in `src/storage.cpp` so that a missing file is treated as an empty ledger.
3. Write a small program that writes two lines with `FileStorage`, closes it, and reads them back.

Done when the program reads back both lines in order.

Exercise 4 writes the ledger rules.

1. Implement `Ledger` with `load`, `add`, `save`, `entries`, `total_cents` and `totals_by_category`.
2. Add an entry with a comma in its category and read the exception message. Add an entry with a date of `31/10/2026` and read that message too.
3. Load a file with a bad second line and read the line number in the error.

Done when each of the three failures produces a clear message, and the bad entries never appear in `entries()`.

Exercise 5 runs a sanitizer on the finished library.

1. Configure with `-DENABLE_SANITIZERS=ON` in WSL and run the tests.
2. Introduce a deliberate bug: in `FileStorage::read_lines`, return a line from a vector that has already been destroyed, or read one past the end of a vector. Run the tests and copy the first line of the sanitizer report.
3. Remove the bug and run again.

Done when the report names the bug line and the clean run passes.

---

## Project: an expense ledger with tests, presets and CI

The project is a command-line expense ledger. It has a library that owns the rules, an application that parses the command line and calls the library, a GoogleTest suite, the three presets from Day 39, and the workflow from Day 39. The first commit must build and pass its tests, and the last commit must have a green workflow run on `main`.

Suggested repo name: `100-days-software-day-40-cpp-capstone`.

The repo layout:

```
100-days-software-day-40-cpp-capstone/
├── .github/
│   └── workflows/
│       └── ci.yml
├── .gitignore
├── CMakeLists.txt
├── CMakePresets.json
├── README.md
├── apps/
│   └── ledger/
│       ├── CMakeLists.txt
│       └── main.cpp
├── docs/
│   └── design.md
├── libs/
│   └── ledger/
│       ├── CMakeLists.txt
│       ├── include/
│       │   └── ledger/
│       │       ├── entry.hpp
│       │       ├── ledger.hpp
│       │       └── storage.hpp
│       └── src/
│           ├── entry.cpp
│           ├── ledger.cpp
│           └── storage.cpp
└── tests/
    ├── CMakeLists.txt
    └── ledger_test.cpp
```

Start from the Day 39 repository. Copy its `CMakePresets.json`, its `.github/workflows/ci.yml`, its root `CMakeLists.txt` and its `tests/CMakeLists.txt`. Change the project name in the root file to `expense_ledger`, change the two `add_subdirectory` lines to `libs/ledger` and `apps/ledger`, and in `tests/CMakeLists.txt` rename the test target to `ledger_test`, change its source to `ledger_test.cpp`, and link `ledger::ledger` in place of `textkit::textkit`. Then add the files below. The presets, the workflow and the warning policy stay unchanged.

The public header for the records declares the `Entry` struct and the parsing and formatting functions:

```cpp
// libs/ledger/include/ledger/entry.hpp
#ifndef LEDGER_ENTRY_HPP
#define LEDGER_ENTRY_HPP

#include <cstdint>
#include <optional>
#include <string>
#include <string_view>

struct Entry {
    std::string date;      // YYYY-MM-DD
    std::string category;
    std::int64_t cents;    // amounts are whole cents, never floating point
    std::string description;
};

bool is_valid_date(std::string_view text);
std::optional<std::int64_t> parse_amount(std::string_view text);
std::string format_cents(std::int64_t cents);
std::string to_csv(const Entry& entry);
Entry from_csv(std::string_view line);

#endif
```

The implementation parses the amount directly into cents, checks each CSV field, and throws `std::invalid_argument` with a short reason:

```cpp
// libs/ledger/src/entry.cpp
#include "ledger/entry.hpp"

#include <algorithm>
#include <cctype>
#include <cstddef>
#include <exception>
#include <stdexcept>

namespace {

bool all_digits(std::string_view text) {
    return !text.empty() && std::all_of(text.begin(), text.end(), [](char c) {
        return std::isdigit(static_cast<unsigned char>(c)) != 0;
    });
}

}  // namespace

bool is_valid_date(std::string_view text) {
    if (text.size() != 10 || text[4] != '-' || text[7] != '-') {
        return false;
    }
    for (std::size_t i = 0; i < text.size(); ++i) {
        if (i == 4 || i == 7) {
            continue;
        }
        if (!std::isdigit(static_cast<unsigned char>(text[i]))) {
            return false;
        }
    }
    return true;
}

std::optional<std::int64_t> parse_amount(std::string_view text) {
    const std::size_t dot = text.find('.');
    const std::string_view whole = (dot == std::string_view::npos) ? text : text.substr(0, dot);
    if (!all_digits(whole) || whole.size() > 12) {
        return std::nullopt;
    }

    std::int64_t cents = std::stoll(std::string(whole)) * 100;
    if (dot == std::string_view::npos) {
        return cents;
    }

    const std::string_view fraction = text.substr(dot + 1);
    if (fraction.empty() || fraction.size() > 2 || !all_digits(fraction)) {
        return std::nullopt;
    }
    const std::int64_t extra = std::stoll(std::string(fraction));
    cents += (fraction.size() == 1) ? extra * 10 : extra;
    return cents;
}

std::string format_cents(std::int64_t cents) {
    const std::int64_t whole = cents / 100;
    const std::int64_t fraction = cents % 100;
    std::string text = std::to_string(whole) + ".";
    if (fraction < 10) {
        text += '0';
    }
    text += std::to_string(fraction);
    return text;
}

std::string to_csv(const Entry& entry) {
    return entry.date + ',' + entry.category + ',' + std::to_string(entry.cents) + ',' +
           entry.description;
}

Entry from_csv(std::string_view line) {
    const std::size_t c1 = line.find(',');
    const std::size_t c2 = (c1 == std::string_view::npos) ? c1 : line.find(',', c1 + 1);
    const std::size_t c3 = (c2 == std::string_view::npos) ? c2 : line.find(',', c2 + 1);
    if (c3 == std::string_view::npos) {
        throw std::invalid_argument("expected four comma-separated fields");
    }

    Entry entry{};
    entry.date = std::string(line.substr(0, c1));
    entry.category = std::string(line.substr(c1 + 1, c2 - c1 - 1));
    const std::string cents_text(line.substr(c2 + 1, c3 - c2 - 1));
    entry.description = std::string(line.substr(c3 + 1));

    try {
        std::size_t used = 0;
        const long long value = std::stoll(cents_text, &used);
        if (used != cents_text.size() || value < 0) {
            throw std::invalid_argument("negative or partial number");
        }
        entry.cents = value;
    } catch (const std::exception&) {
        throw std::invalid_argument("amount is not a whole number of cents");
    }

    if (!is_valid_date(entry.date)) {
        throw std::invalid_argument("date must be YYYY-MM-DD");
    }
    return entry;
}
```

The storage header declares the interface and its two implementations. The memory version is in the header because it is tiny, and the file version has its own source file:

```cpp
// libs/ledger/include/ledger/storage.hpp
#ifndef LEDGER_STORAGE_HPP
#define LEDGER_STORAGE_HPP

#include <string>
#include <vector>

class Storage {
public:
    virtual ~Storage() = default;
    virtual std::vector<std::string> read_lines() const = 0;
    virtual void write_lines(const std::vector<std::string>& lines) = 0;
};

class MemoryStorage final : public Storage {
public:
    std::vector<std::string> read_lines() const override { return lines_; }
    void write_lines(const std::vector<std::string>& lines) override { lines_ = lines; }

private:
    std::vector<std::string> lines_;
};

class FileStorage final : public Storage {
public:
    explicit FileStorage(std::string path);
    std::vector<std::string> read_lines() const override;
    void write_lines(const std::vector<std::string>& lines) override;

private:
    std::string path_;
};

#endif
```

```cpp
// libs/ledger/src/storage.cpp
#include "ledger/storage.hpp"

#include <fstream>
#include <stdexcept>
#include <utility>

FileStorage::FileStorage(std::string path) : path_(std::move(path)) {}

std::vector<std::string> FileStorage::read_lines() const {
    std::vector<std::string> lines;
    std::ifstream in(path_);  // RAII: the stream closes when this function returns
    if (!in) {
        return lines;         // a missing file is an empty ledger
    }
    std::string line;
    while (std::getline(in, line)) {
        lines.push_back(line);
    }
    return lines;
}

void FileStorage::write_lines(const std::vector<std::string>& lines) {
    std::ofstream out(path_, std::ios::trunc);
    if (!out) {
        throw std::runtime_error("cannot write " + path_);
    }
    for (const std::string& line : lines) {
        out << line << '\n';
    }
    out.close();
    if (!out) {
        throw std::runtime_error("write failed for " + path_);
    }
}
```

The ledger header declares its error type and class. The class owns its storage and exposes read-only access to its entries:

```cpp
// libs/ledger/include/ledger/ledger.hpp
#ifndef LEDGER_LEDGER_HPP
#define LEDGER_LEDGER_HPP

#include <cstddef>
#include <cstdint>
#include <map>
#include <memory>
#include <stdexcept>
#include <string>
#include <vector>

#include "ledger/entry.hpp"
#include "ledger/storage.hpp"

class LedgerError : public std::runtime_error {
public:
    LedgerError(std::size_t line, const std::string& message);
    std::size_t line() const { return line_; }

private:
    std::size_t line_;
};

class Ledger {
public:
    explicit Ledger(std::unique_ptr<Storage> storage);

    void load();
    void add(Entry entry);
    void save() const;

    const std::vector<Entry>& entries() const { return entries_; }
    std::int64_t total_cents() const;
    std::map<std::string, std::int64_t> totals_by_category() const;

private:
    std::unique_ptr<Storage> storage_;
    std::vector<Entry> entries_;
};

#endif
```

```cpp
// libs/ledger/src/ledger.cpp
#include "ledger/ledger.hpp"

#include <stdexcept>
#include <utility>

namespace {

void validate(const Entry& entry) {
    if (!is_valid_date(entry.date)) {
        throw std::invalid_argument("date must be YYYY-MM-DD");
    }
    if (entry.category.empty() || entry.category.find(',') != std::string::npos) {
        throw std::invalid_argument("category must be non-empty and contain no commas");
    }
    if (entry.description.empty() || entry.description.find(',') != std::string::npos) {
        throw std::invalid_argument("description must be non-empty and contain no commas");
    }
    if (entry.cents < 0) {
        throw std::invalid_argument("amount must not be negative");
    }
}

}  // namespace

LedgerError::LedgerError(std::size_t line, const std::string& message)
    : std::runtime_error("line " + std::to_string(line) + ": " + message), line_(line) {}

Ledger::Ledger(std::unique_ptr<Storage> storage) : storage_(std::move(storage)) {
    if (!storage_) {
        throw std::invalid_argument("storage must not be null");
    }
}

void Ledger::load() {
    entries_.clear();
    std::size_t number = 0;
    for (const std::string& line : storage_->read_lines()) {
        ++number;
        if (line.empty()) {
            continue;
        }
        try {
            entries_.push_back(from_csv(line));
        } catch (const std::invalid_argument& error) {
            throw LedgerError(number, error.what());
        }
    }
}

void Ledger::add(Entry entry) {
    validate(entry);
    entries_.push_back(std::move(entry));
}

void Ledger::save() const {
    std::vector<std::string> lines;
    lines.reserve(entries_.size());
    for (const Entry& entry : entries_) {
        lines.push_back(to_csv(entry));
    }
    storage_->write_lines(lines);
}

std::int64_t Ledger::total_cents() const {
    std::int64_t total = 0;
    for (const Entry& entry : entries_) {
        total += entry.cents;
    }
    return total;
}

std::map<std::string, std::int64_t> Ledger::totals_by_category() const {
    std::map<std::string, std::int64_t> totals;
    for (const Entry& entry : entries_) {
        totals[entry.category] += entry.cents;
    }
    return totals;
}
```

The application parses the command line, and it turns every library error into a message and a non-zero exit code:

```cpp
// apps/ledger/main.cpp
#include <exception>
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

#include "ledger/entry.hpp"
#include "ledger/ledger.hpp"
#include "ledger/storage.hpp"

namespace {

void print_usage() {
    std::cerr << "usage:\n"
              << "  ledger add DATE CATEGORY AMOUNT DESCRIPTION...\n"
              << "  ledger list\n"
              << "  ledger totals\n";
}

std::string join_words(int first, int argc, char* argv[]) {
    std::string text;
    for (int i = first; i < argc; ++i) {
        if (i > first) {
            text += ' ';
        }
        text += argv[i];
    }
    return text;
}

}  // namespace

int main(int argc, char* argv[]) {
    if (argc < 2) {
        print_usage();
        return 1;
    }
    const std::string command = argv[1];

    try {
        Ledger ledger(std::make_unique<FileStorage>("ledger.csv"));
        ledger.load();

        if (command == "add" && argc >= 6) {
            const auto cents = parse_amount(argv[4]);
            if (!cents) {
                throw std::invalid_argument("amount must look like 12 or 12.34");
            }
            ledger.add(Entry{argv[2], argv[3], *cents, join_words(5, argc, argv)});
            ledger.save();
            std::cout << "added\n";
        } else if (command == "list") {
            for (const Entry& entry : ledger.entries()) {
                std::cout << entry.date << ' ' << entry.category << ' '
                          << format_cents(entry.cents) << ' ' << entry.description << '\n';
            }
        } else if (command == "totals") {
            for (const auto& [category, cents] : ledger.totals_by_category()) {
                std::cout << category << ": " << format_cents(cents) << '\n';
            }
            std::cout << "total: " << format_cents(ledger.total_cents()) << '\n';
        } else {
            print_usage();
            return 1;
        }
    } catch (const std::exception& error) {
        std::cerr << "error: " << error.what() << '\n';
        return 1;
    }
    return 0;
}
```

The test file covers every rule the library owns. It uses `MemoryStorage` for most tests and `FileStorage` for one round trip that touches the disk:

```cpp
// tests/ledger_test.cpp
#include <gtest/gtest.h>

#include <cstdint>
#include <cstdio>
#include <memory>
#include <optional>
#include <stdexcept>
#include <string>

#include "ledger/entry.hpp"
#include "ledger/ledger.hpp"
#include "ledger/storage.hpp"

TEST(AmountParsing, AcceptsWholeAndDecimalAmounts) {
    EXPECT_EQ(parse_amount("12.34"), std::optional<std::int64_t>(1234));
    EXPECT_EQ(parse_amount("12"), std::optional<std::int64_t>(1200));
    EXPECT_EQ(parse_amount("12.3"), std::optional<std::int64_t>(1230));
}

TEST(AmountParsing, RejectsEverythingElse) {
    EXPECT_FALSE(parse_amount("").has_value());
    EXPECT_FALSE(parse_amount("abc").has_value());
    EXPECT_FALSE(parse_amount("-1").has_value());
    EXPECT_FALSE(parse_amount("1.234").has_value());
    EXPECT_FALSE(parse_amount("12.").has_value());
}

TEST(Formatting, PadsCentsToTwoDigits) {
    EXPECT_EQ(format_cents(5), "0.05");
    EXPECT_EQ(format_cents(1200), "12.00");
    EXPECT_EQ(format_cents(1234), "12.34");
}

TEST(Dates, AcceptsOnlyTheIsoShape) {
    EXPECT_TRUE(is_valid_date("2026-10-31"));
    EXPECT_FALSE(is_valid_date("31-10-2026"));
    EXPECT_FALSE(is_valid_date("2026-1-1"));
}

TEST(CsvRecords, RoundTripKeepsEveryField) {
    const Entry original{"2026-10-31", "food", 1250, "lunch with team"};
    const Entry copy = from_csv(to_csv(original));
    EXPECT_EQ(copy.date, original.date);
    EXPECT_EQ(copy.category, original.category);
    EXPECT_EQ(copy.cents, original.cents);
    EXPECT_EQ(copy.description, original.description);
}

TEST(CsvRecords, ThreeFieldsAreRejected) {
    EXPECT_THROW(from_csv("2026-10-31,food,100"), std::invalid_argument);
}

TEST(LedgerRules, TotalsAndCategoriesAreCorrect) {
    Ledger ledger(std::make_unique<MemoryStorage>());
    ledger.add(Entry{"2026-10-31", "food", 1250, "lunch"});
    ledger.add(Entry{"2026-10-31", "travel", 300, "bus"});
    ledger.add(Entry{"2026-11-01", "food", 500, "coffee"});

    EXPECT_EQ(ledger.entries().size(), 3u);
    EXPECT_EQ(ledger.total_cents(), 2050);
    EXPECT_EQ(ledger.totals_by_category().at("food"), 1750);
}

TEST(LedgerRules, RejectsBadEntriesAndKeepsTheLedgerClean) {
    Ledger ledger(std::make_unique<MemoryStorage>());
    EXPECT_THROW(ledger.add(Entry{"2026-10-31", "food, out", 100, "x"}), std::invalid_argument);
    EXPECT_THROW(ledger.add(Entry{"31/10/2026", "food", 100, "x"}), std::invalid_argument);
    EXPECT_THROW(ledger.add(Entry{"2026-10-31", "food", -5, "x"}), std::invalid_argument);
    EXPECT_TRUE(ledger.entries().empty());
}

TEST(LedgerRules, SaveWritesCsvAndLoadReadsItBack) {
    auto storage = std::make_unique<MemoryStorage>();
    MemoryStorage* view = storage.get();  // non-owning; the ledger owns the storage
    Ledger ledger(std::move(storage));

    ledger.add(Entry{"2026-10-31", "food", 1250, "lunch"});
    ledger.save();
    ASSERT_EQ(view->read_lines().size(), 1u);
    EXPECT_EQ(view->read_lines()[0], "2026-10-31,food,1250,lunch");

    ledger.load();
    EXPECT_EQ(ledger.entries().size(), 1u);
}

TEST(LedgerRules, LoadReportsTheBadLineNumber) {
    auto storage = std::make_unique<MemoryStorage>();
    MemoryStorage* view = storage.get();
    Ledger ledger(std::move(storage));

    view->write_lines({"2026-10-31,food,100,ok", "not a record"});
    try {
        ledger.load();
        FAIL() << "expected a LedgerError";
    } catch (const LedgerError& error) {
        EXPECT_EQ(error.line(), 2u);
    }
}

TEST(FileStorageRoundTrip, SavedEntriesComeBackFromDisk) {
    const std::string path = "ledger-test.csv";
    {
        Ledger writer(std::make_unique<FileStorage>(path));
        writer.add(Entry{"2026-10-31", "food", 1250, "lunch"});
        writer.save();
    }
    Ledger reader(std::make_unique<FileStorage>(path));
    reader.load();
    ASSERT_EQ(reader.entries().size(), 1u);
    EXPECT_EQ(reader.entries()[0].description, "lunch");
    std::remove(path.c_str());
}
```

The library and the application each get a small `CMakeLists.txt`. The library installs its headers and archive, and the application renames its binary to `ledger`:

```cmake
# libs/ledger/CMakeLists.txt
add_library(ledger STATIC src/entry.cpp src/ledger.cpp src/storage.cpp)
add_library(ledger::ledger ALIAS ledger)

target_include_directories(ledger PUBLIC include)
target_link_libraries(ledger PRIVATE project_warnings)

install(TARGETS ledger ARCHIVE DESTINATION lib)
install(DIRECTORY include/ DESTINATION include)
```

```cmake
# apps/ledger/CMakeLists.txt
add_executable(ledger_cli main.cpp)
set_target_properties(ledger_cli PROPERTIES OUTPUT_NAME ledger)
target_link_libraries(ledger_cli PRIVATE ledger::ledger project_warnings)
```

Step-by-step tasks:

1. Create the layout. Start from the Day 39 files as described above, and add the library, application and test files. Add `build/`, `build-asan/` and `ledger.csv` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add expense ledger library and tests".
2. Run `cmake --preset debug`, `cmake --build --preset debug` and `ctest --preset debug --output-on-failure`. Confirm all test groups pass. If the release preset crashes inside `libstdc++` at run time, check `PATH` as described in Day 27: the compiler's `bin` folder must come before Git's.
3. Run the application from the build folder, in an empty working folder so that `ledger.csv` is created there. Add two expenses with `add 2026-10-31 food 12.50 lunch with team` and `add 2026-10-31 travel 3 bus fare`. Each prints `added`.
4. Run `list` and check that it prints both records, with the amounts as `12.50` and `3.00`. Run `totals` and check that it prints `food: 12.50`, `travel: 3.00` and `total: 15.50`.
5. Try `add 2026-10-31 food 1.234 too precise`. Read the error and record it in the README.
6. Run the release preset the same way. Then, in WSL, run `cmake --preset asan`, build and test, and confirm no sanitizer reports.
7. Create a branch `feature/monthly-totals`. Write a failing test first for `std::map<std::string, std::int64_t> totals_by_month() const` that groups totals by the first seven characters of the date. Implement the function in the library and expose it through the command `ledger months`, then make the tests pass and check the new command by hand.
8. Write `docs/design.md` with four short sections: the three layers and the rule each one owns, the reason amounts are stored as cents, the reason the file storage sits behind an interface, and one rule you checked in two places and why.
9. Create an empty public GitHub repository named `100-days-software-day-40-cpp-capstone`. Merge the branch with `git merge --no-ff feature/monthly-totals -m "Add monthly totals"`, then push with `git remote add origin <URL>` and `git push -u origin main`.
10. Open the Actions tab and confirm the workflow runs and passes on `main` for all three jobs. Add the workflow badge to the top of `README.md` and push a final commit with the message "Add CI badge".

Done when:

- `ctest --preset debug --output-on-failure` passes, with a test group for each rule in the test file.
- The CLI prints the exact `list` and `totals` output from task 4 for the two sample expenses.
- The WSL sanitizer build reports nothing.
- The GitHub repo shows the merge commit and the final badge commit on `main`, with a green workflow run.
- `docs/design.md` explains each layer and each rule with one concrete example.

Evidence to commit: all source, header, test, preset, workflow and CMake files, `docs/design.md`, `README.md` with the badge and the CLI examples, and the commit history on GitHub.

This is the capstone, so the bar is the whole repository. A reader should be able to clone the repo, run three commands, see a green run, and then read the library header and understand what the program promises.

---

## Check your understanding

1. The application stores amounts as `double` and prints them with two decimals. Explain how ten entries of `0.10` can total to a value that is not exactly one, and how the cents design fixes it.
2. A teammate moves the date check from `Ledger::add` into the command-line code. Describe which tests would still pass, which behavior is now unchecked, and where the rule belongs.
3. The test keeps a raw pointer to `MemoryStorage` after moving the `unique_ptr` into the ledger. Explain why this is safe in the test, and what would make it unsafe.
4. A user reports that the file was emptied after a failed write. Explain how the `FileStorage::write_lines` function prevents a partial write from being reported as success, and what it cannot prevent.
5. The workflow passes on Ubuntu but fails on Windows with a warning treated as an error. Explain where the warning policy is defined, why the fix belongs in the code and not in the workflow, and what changes in the presets if you disable `-Werror` on Windows only.
6. Why does the library expose `entries()` as a const reference, and what would be lost by returning a copy of the whole vector on every call?

### Answers

1. Binary floating point cannot represent 0.10 exactly, so each addition adds a tiny error, and the sum can be a few units of the last binary digit away from 1.00. Storing whole cents in an integer makes every addition exact, and the decimal point appears only when the value is formatted for display.
2. The command-line tests would still pass for valid input, but no test would check that the library rejects a bad date, because the library test would not have the rule. The rule belongs in `Ledger::add` and `validate`, where every caller, including tests and any future interface, goes through it.
3. The raw pointer is valid because the ledger owns the storage and the test keeps the ledger alive for the whole test. It would be unsafe if the ledger were destroyed or reset before the test used the pointer, because the pointer would then refer to freed memory.
4. The function opens the file with truncation, writes every line, closes the stream, and checks the stream state after closing, so a failure at write or close time throws an exception. It cannot protect against a crash or power loss in the middle of the write, because the file is truncated before the new content is written. A safer design writes a temporary file and renames it.
5. The warning policy is defined once in the root `CMakeLists.txt` through the `project_warnings` interface target. The fix belongs in the code because the warning shows a real problem that another compiler reports, and the workflow should run the same checks as a developer's machine. Disabling `-Werror` on Windows would let the same warning reach users, so the change would be a second policy to maintain.
6. A const reference lets callers read the entries without copying them, and it keeps the list private so that only `add` and `load` can change it. Returning a copy on every call would cost time and memory proportional to the number of entries each time someone asked for them.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Comma-separated values](https://en.wikipedia.org/wiki/Comma-separated_values) for the file format used for the records
- [Wikipedia: Fixed-point arithmetic](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) for the reasoning behind storing money as cents
- [Wikipedia: Ledger](https://en.wikipedia.org/wiki/Ledger) for the accounting idea behind the program
- cppreference.com, the pages for `std::optional`, `std::unique_ptr`, `std::map`, `std::ifstream`, `std::ofstream` and `std::stoll`
- The CMake documentation at cmake.org/cmake/help/latest, the pages for `cmake-presets` and `target_include_directories`
- The GitHub Actions documentation at docs.github.com/actions, the pages for workflow syntax and the status badge
- The C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on interfaces and resource management

*Tutorials*

- learncpp.com, the chapters on file input and output, classes and smart pointers, written by Alex Allain
- GeeksforGeeks, search the site for "C++ file handling" and "C++ CSV parsing" for the matching articles
- Microsoft Learn, search learn.microsoft.com for "C++ file I/O" for the Windows view of the streams

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on files and classes
- CppCon, the YouTube channel, for talks on interfaces and testing. Search the channel for "testable C++" to find them.

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, for the test-driven exercises that match this capstone's style. Solve two of them before you start the next module, and run each one under the sanitizer preset.

---

## Tomorrow

Module 5 starts with Day 41, complexity analysis, where you measure how the time and memory of an algorithm grow with its input. The ledger has a clear example of that question: the category totals use a `std::map`, and the total for each new entry costs a lookup in the map. The next module asks you to compare that cost with the alternatives, and to choose the data structure from the numbers instead of from habit.

**Next:** Day 41, the first day of the Data Structures and Algorithms module, which is not yet written.

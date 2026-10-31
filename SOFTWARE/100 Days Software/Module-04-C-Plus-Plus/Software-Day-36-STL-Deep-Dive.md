# Day 36: STL Deep Dive

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You have used `std::vector`, iterators and `<algorithm>` from Day 30, and class templates from Day 35. This day widens your view to the rest of the Standard Template Library: maps, sets, queues, stacks and lambdas, and how to choose among them. It is for anyone who reaches for a vector every time and then writes a loop to search it.

**By the end of this day, you will have:**

- Chosen a container for a given problem by naming its access pattern, its ordering and its cost
- Used `std::map`, `std::set`, `std::queue` and `std::stack` for real tasks, including insertion through `operator[]` and the `contains` check
- Written lambdas with captures and used them as comparators and predicates with `<algorithm>`
- Explained which container operations invalidate iterators, and which do not
- A log summary tool that parses a log file's lines, counts levels, collects components and checks bracket balance, with a test executable run by CTest
- A GitHub repo with the build, the test results and a table that maps each container to the problem it solved in your code

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Days 27 to 35: the build, classes, pointers, lifetime, inheritance and templates
- Day 30: vectors, strings, iterators and the basic algorithms

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Containers](#containers)
- [Iterators](#iterators)
- [Algorithms](#algorithms)
- [Lambdas](#lambdas)
- [Maps](#maps)
- [Sets](#sets)
- [Queues](#queues)
- [Stacks](#stacks)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a log summary tool with containers and lambdas](#project-a-log-summary-tool-with-containers-and-lambdas)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Most programs are data processing with a few rules attached. A log file must be grouped by level. A set of users must have no duplicates. A list of jobs must run in order. A chain of nested brackets must match. Each of these needs a different container, and the wrong choice does not fail loudly. It just makes the code slower or harder to read. A vector used for lookups turns a one-line map operation into a loop over every element, and a map used for a queue makes the order of processing a surprise.

The standard library has a container for each of these patterns, and the algorithms and lambdas make them concise. This day gives you a decision method, then walks through the containers you will reach for most often. The project is a log summary tool, and each part of it uses a different container so that the choice is visible in the code.

The container requirements are in the ISO C++ standard. cppreference.com lists, for each container, the complexity of its operations and the conditions under which its iterators stay valid. Those pages are the reference you should use when you are unsure.

---

## Glossary

- **Container:** a class template that stores a collection of elements, such as `std::vector` or `std::map`.
- **Sequence container:** a container that keeps elements in the order you insert them, such as `std::vector`, `std::list` and `std::deque`.
- **Associative container:** a container that orders or indexes elements by a key, such as `std::map` and `std::set`.
- **Container adaptor:** a container that wraps another container to offer a narrower interface, such as `std::queue` and `std::stack`.
- **Key:** the value an associative container uses to find an element.
- **Lambda:** an anonymous function written inline, with the form `[capture](parameters) { body }`.
- **Capture:** the list of outside variables a lambda can use, written between the square brackets.
- **Predicate:** a function that returns `true` or `false`, used by algorithms such as `std::count_if`.
- **Comparator:** a function that says whether one element comes before another, used for sorting and ordered containers.
- **Iterator invalidation:** the state in which an iterator no longer refers to a valid element, after certain container operations.
- **Complexity:** the cost of an operation as the size grows, written in big-O terms such as O(log n) or O(1).

---

## Core concepts

The concepts start with the container families and the decision method, then move through the iterator and algorithm rules, lambdas as the glue, and the four containers the project uses.

### Containers

Think of a filing system. A pile of papers on a desk is quick to add to, but finding one paper means checking each sheet. A filing cabinet with labeled folders takes longer to set up, but a folder is found in a few steps. A conveyor belt delivers papers in the order they arrive. The containers in the standard library are filing systems of these kinds, and each one has a cost profile.

The decision method is three questions. What is the access pattern: by position, by key, or in order of arrival? Does the container need to keep elements sorted? What does each operation cost in the worst case? The table below answers the common cases:

| Need | Container | Typical cost |
|------|-----------|--------------|
| A list you add to and read by index | `std::vector` | O(1) index, amortized O(1) at the end |
| A lookup from key to value, in sorted order | `std::map` | O(log n) per key |
| A set of unique items, in sorted order | `std::set` | O(log n) per item |
| A first-in, first-out line of work | `std::queue` | O(1) push and pop |
| A last-in, first-out stack of state | `std::stack` | O(1) push and pop |

The table is a starting point. A `std::unordered_map` has average O(1) lookups and no sorted order, and it is often the better choice when order does not matter. The common mistake is to choose a container by habit. The fix is to write down the access pattern first, then pick the container that matches. You can spot the mismatch in code review when a loop searches a vector for a key that is used every time. In the market, choosing the right container is one of the most common performance fixes in backend and data code.

### Iterators

Day 30 introduced iterators as bookmarks in a book. The standard library groups them into categories, and each category supports a different set of operations. The categories, from least to most capable, are input, forward, bidirectional and random access. A `std::vector` has random-access iterators, which allow `it + 3`. A `std::list` has bidirectional iterators, which allow only `++` and `--`. A `std::map` and a `std::set` also have bidirectional iterators.

The rule that matters most in practice is invalidation. Inserting into a `std::map` or `std::set` does not invalidate existing iterators, because the nodes do not move. Inserting into a `std::vector` can reallocate, which invalidates every iterator. The reference page for each container lists which operations invalidate which iterators, and reading that page before you store an iterator across a change is the habit that prevents the bug.

```cpp
#include <iostream>
#include <map>

int main() {
    std::map<int, const char*> names{{1, "one"}, {3, "three"}};

    auto it = names.find(1);
    names[2] = "two";  // insertion does not invalidate existing iterators
    names[4] = "four";

    std::cout << it->second << '\n';
    return 0;
}
```

The output is `one`. The map inserted two new elements after the iterator was taken, and the iterator still points at the element for key 1. The same code with a vector would be undefined behavior if the vector reallocated.

The common mistake is to assume all containers behave the same. The fix is to check the invalidation table for each container you use. In the market, the difference between stable and unstable iterators is a frequent source of bugs in event-driven code, where a callback modifies a container that a loop is walking.

### Algorithms

The algorithms from Day 30 work on any range, and they become more useful with lambdas. The two you will use most are `std::transform`, which applies a function to each element and writes the result to another range, and `std::accumulate`, which folds a range into one value. Both take iterators and a function, and both are defined in the `<numeric>` and `<algorithm>` headers.

```cpp
#include <algorithm>
#include <iostream>
#include <numeric>
#include <vector>

int main() {
    const std::vector<int> prices{100, 250, 75};
    std::vector<int> with_tax(prices.size());

    std::transform(prices.begin(), prices.end(), with_tax.begin(),
                   [](int price) { return price + price / 10; });

    const int total = std::accumulate(with_tax.begin(), with_tax.end(), 0);
    std::cout << "total with tax: " << total << '\n';
    return 0;
}
```

The output is `total with tax: 467`. The vector `with_tax` must be sized before `std::transform` writes to it, because the algorithm writes through the output iterator and does not grow the container. A `std::back_inserter` is the alternative when you do not know the size in advance.

The common mistake is to write `std::transform` into an empty vector without sizing it, which writes to memory the vector does not own. The fix is to size the output or use `std::back_inserter`. In the market, these algorithms replace most hand-written loops in mature code, because they state the intent and the compiler optimizes them well.

### Lambdas

A lambda is an anonymous function written where you need it. Think of a sticky note with a small rule on it, placed next to the work it applies to. The rule can refer to nearby variables, and the capture list says which ones. Lambdas are how algorithms receive custom behavior without a separate named function.

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

int main() {
    std::vector<std::string> names{"Grace", "Al", "Barbara"};
    const int min_length = 3;

    std::sort(names.begin(), names.end(),
              [](const std::string& a, const std::string& b) {
                  return a.size() < b.size();
              });

    const auto long_enough = std::count_if(
        names.begin(), names.end(),
        [min_length](const std::string& name) {
            return static_cast<int>(name.size()) >= min_length;
        });

    for (const std::string& name : names) {
        std::cout << name << '\n';
    }
    std::cout << "long enough: " << long_enough << '\n';
    return 0;
}
```

The output lists `Al`, `Grace` and `Barbara`, each on its own line, and then `long enough: 2`. The first lambda is a comparator that sorts by length. The second is a predicate with the capture `min_length`, which copies the value into the lambda. Capture by reference with `[&x]` when you want the lambda to see changes to `x`, and capture by value when you want a snapshot.

The common mistake is to capture by reference a local variable and then return the lambda from the function, which leaves a dangling reference. The fix is to capture by value when the lambda outlives the scope. Another mistake is to write a comparator that is not a strict weak ordering, such as one that returns true for equal elements, which makes `std::sort` undefined. You can spot it when sorting crashes on data with duplicates. In the market, custom comparators are where many sorting bugs live, and the rule that equal elements return false is worth memorizing.

### Maps

A `std::map` is a sorted collection of key and value pairs. Think of a dictionary with words in alphabetical order. You can look up a word by its spelling, and the entries always come out sorted. Keys are unique, and the value type can be anything. The map is implemented as a balanced tree, so lookups, inserts and erases take O(log n) time.

The `operator[]` form inserts a default value if the key is missing, which makes counting easy and also surprising:

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> stock{{"apple", 3}};
    stock["pear"] += 2;
    stock["apple"] -= 1;

    for (const auto& [item, count] : stock) {
        std::cout << item << ": " << count << '\n';
    }
    std::cout << "has kiwi: " << stock.contains("kiwi") << '\n';
    return 0;
}
```

The output is `apple: 2`, `pear: 2` and `has kiwi: 0`. The `stock["pear"]` expression created the entry with the value 0 and then added 2. The `contains` function from C++20 checks for a key without creating it, which is the right tool when you only want to ask a question. Using `operator[]` to ask a question creates an entry you did not mean to add.

The common mistake is to use `operator[]` for a lookup on a map that should not grow, which silently adds keys. The fix is `find` or `contains`, or `at`, which throws `std::out_of_range` when the key is missing. In the market, log aggregation, metrics and caching code use maps constantly, and an accidental insert is a common reason for memory growth.

### Sets

A `std::set` stores unique elements in sorted order. Think of a guest list where each name appears once and the printout is alphabetical. Inserting a duplicate does nothing, and the set reports whether the insert happened. Like a map, it is a balanced tree with O(log n) operations.

```cpp
#include <iostream>
#include <set>
#include <string>

int main() {
    std::set<std::string> tags;
    tags.insert("cpp");
    tags.insert("build");
    const bool inserted = tags.insert("cpp").second;  // duplicate, so false

    for (const std::string& tag : tags) {
        std::cout << tag << '\n';
    }
    std::cout << "size " << tags.size() << ", second insert added: " << inserted << '\n';
    return 0;
}
```

The output is `build`, `cpp`, then `size 2, second insert added: 0`. The structured binding `[it, inserted]` unpacks the pair that `insert` returns, and `it` points at the existing element. The set keeps the order itself, so you do not sort it.

The common mistake is to use a vector and call `std::sort` and `std::unique` every time you add an item. That is correct, but it costs O(n log n) on each change, and a set keeps the invariant for you. The opposite mistake is to use a set when the order of insertion matters. A set does not keep insertion order. In the market, sets appear in access control lists, deduplication stages and graph algorithms.

### Queues

A `std::queue` is a first-in, first-out line. Think of a checkout line at a shop, where the first customer to join is the first to leave. It is a container adaptor: it sits on top of another container, usually a `std::deque`, and exposes only `push`, `pop`, `front` and `empty`. The narrow interface is the point, because it stops callers from reaching into the middle of the line.

```cpp
#include <iostream>
#include <queue>
#include <string>

int main() {
    std::queue<std::string> jobs;
    jobs.push("resize");
    jobs.push("upload");
    jobs.push("notify");

    while (!jobs.empty()) {
        std::cout << "run " << jobs.front() << '\n';
        jobs.pop();
    }
    return 0;
}
```

The output is `run resize`, `run upload` and `run notify`, in that order. `front` reads the oldest item, and `pop` removes it without returning it, so you read the value before you pop. The common mistake is to call `front` on an empty queue, which is undefined behavior. The loop condition `!jobs.empty()` prevents it.

A queue does not allow priority. If some jobs must run first, use `std::priority_queue`, which Day 47 covers in the data structures module. In the market, queues connect producers and consumers in message systems, print spoolers and task runners.

### Stacks

A `std::stack` is a last-in, first-out container. Think of a stack of trays in a cafeteria, where you take the top tray first. Like the queue, it is an adaptor, and it exposes `push`, `pop`, `top` and `empty`. Stacks fit any problem where the most recent item must be handled first, such as undo history, parsing and checking nested structures.

```cpp
#include <iostream>
#include <stack>
#include <string>

int main() {
    std::stack<std::string> history;
    history.push("home");
    history.push("search");
    history.push("result");

    std::cout << "current page: " << history.top() << '\n';
    history.pop();
    std::cout << "after back: " << history.top() << '\n';
    return 0;
}
```

The output is `current page: result` and then `after back: search`. The stack gives the most recent page first, which is what a back button does. The project uses a stack to check whether brackets in a log message are balanced, which is a classic case where the last opened bracket must be the first closed.

The common mistake is to call `top` or `pop` on an empty stack. The fix is to check `empty()` first. Another mistake is to store pointers or references to stack elements and expect them to stay valid while the stack changes, which the reallocation rules for the underlying container can break. In the market, stacks drive expression evaluators, compilers and undo features in editors.

---

## Hands-on exercises

Work in a new folder called `stl-deep-dive`. Run `git init -b main` in it.

Exercise 1 chooses containers.

1. For five problems you can name from your own work or study, write the access pattern, the ordering need and the container you would pick.
2. Check one pick against the reference page on cppreference.com, and note the complexity of the operation you rely on most.

Done when each pick has a one-sentence reason in your notes.

Exercise 2 counts words with a map.

1. Write a program that reads words from standard input and counts each one with `std::map<std::string, int>`.
2. Print the counts in alphabetical order. Then call `contains` for a word that is not in the input and confirm the map size did not change.

Done when the counts are correct and the map size is unchanged after the `contains` check.

Exercise 3 removes duplicates with a set.

1. Read integers into a `std::vector<int>` with duplicates.
2. Copy them into a `std::set<int>` and print the size and the elements.
3. Sort the vector with `std::sort` and `std::unique` and compare the result with the set.

Done when the two results match and you can explain the cost difference in one sentence.

Exercise 4 processes a queue and a stack.

1. Push five strings into a `std::queue`, pop them, and print them in order.
2. Use a `std::stack` to reverse the same five strings. Print the reversed order.

Done when the queue prints in insertion order and the stack prints in reverse.

Exercise 5 writes lambdas with captures.

1. Write a lambda that captures a threshold by value and counts the values in a vector above it.
2. Write a second lambda that captures a counter by reference and increments it for each element passed to `std::for_each`.
3. Try to return the first lambda from a function that declares the threshold locally, and read the compiler warning or run the program under AddressSanitizer in WSL.

Done when the counter reflects the number of calls and you have noted the lifetime problem in step 3.

---

## Project: a log summary tool with containers and lambdas

The project reads log lines of the form `LEVEL component: message`, such as `ERROR db: connection refused`. It puts the lines into a queue, parses each one into a struct, counts the levels in a map, collects the component names in a set, filters the errors with a lambda, and checks bracket balance with a stack. The test checks each part.

Suggested repo name: `100-days-software-day-36-stl-deep-dive`.

The repo layout:

```
100-days-software-day-36-stl-deep-dive/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── container-table.md
├── src/
│   ├── log_summary.cpp
│   ├── log_summary.hpp
│   └── main.cpp
└── tests/
    └── log_summary_test.cpp
```

The header declares the entry type and the functions. Each function's signature names the container it uses:

```cpp
// src/log_summary.hpp
#ifndef LOG_SUMMARY_HPP
#define LOG_SUMMARY_HPP

#include <map>
#include <optional>
#include <queue>
#include <set>
#include <string>
#include <string_view>
#include <vector>

struct LogEntry {
    std::string level;
    std::string component;
    std::string message;
};

// Parses "LEVEL component: message". Returns nullopt for malformed lines.
std::optional<LogEntry> parse_line(std::string_view line);

// Parses every queued line in first-in, first-out order and skips malformed ones.
std::vector<LogEntry> drain(std::queue<std::string> pending);

// Counts entries per level. The map keeps levels in sorted order.
std::map<std::string, int> count_levels(const std::vector<LogEntry>& entries);

// Collects unique component names in sorted order.
std::set<std::string> unique_components(const std::vector<LogEntry>& entries);

// Returns the entries whose level matches exactly.
std::vector<LogEntry> filter_level(const std::vector<LogEntry>& entries, const std::string& level);

// True when every bracket in the text is closed in the right order.
bool brackets_balanced(std::string_view text);

#endif
```

The implementation uses each container for the job it is suited to. The stack in `brackets_balanced` holds open brackets, and the top is always the most recent one:

```cpp
// src/log_summary.cpp
#include "log_summary.hpp"

#include <algorithm>
#include <iterator>
#include <stack>
#include <utility>

std::optional<LogEntry> parse_line(std::string_view line) {
    const auto space = line.find(' ');
    if (space == std::string_view::npos) {
        return std::nullopt;
    }
    const auto colon = line.find(": ", space + 1);
    if (colon == std::string_view::npos) {
        return std::nullopt;
    }

    LogEntry entry;
    entry.level = std::string(line.substr(0, space));
    entry.component = std::string(line.substr(space + 1, colon - space - 1));
    entry.message = std::string(line.substr(colon + 2));

    if (entry.level.empty() || entry.component.empty()) {
        return std::nullopt;
    }
    return entry;
}

std::vector<LogEntry> drain(std::queue<std::string> pending) {
    std::vector<LogEntry> entries;
    while (!pending.empty()) {
        if (std::optional<LogEntry> entry = parse_line(pending.front())) {
            entries.push_back(std::move(*entry));
        }
        pending.pop();
    }
    return entries;
}

std::map<std::string, int> count_levels(const std::vector<LogEntry>& entries) {
    std::map<std::string, int> counts;
    for (const LogEntry& entry : entries) {
        ++counts[entry.level];
    }
    return counts;
}

std::set<std::string> unique_components(const std::vector<LogEntry>& entries) {
    std::set<std::string> names;
    for (const LogEntry& entry : entries) {
        names.insert(entry.component);
    }
    return names;
}

std::vector<LogEntry> filter_level(const std::vector<LogEntry>& entries, const std::string& level) {
    std::vector<LogEntry> result;
    std::copy_if(entries.begin(), entries.end(), std::back_inserter(result),
                 [&level](const LogEntry& entry) { return entry.level == level; });
    return result;
}

bool brackets_balanced(std::string_view text) {
    std::stack<char> open;
    for (const char c : text) {
        if (c == '(' || c == '[' || c == '{') {
            open.push(c);
        } else if (c == ')' || c == ']' || c == '}') {
            if (open.empty()) {
                return false;
            }
            const char expected = (c == ')') ? '(' : (c == ']') ? '[' : '{';
            if (open.top() != expected) {
                return false;
            }
            open.pop();
        }
    }
    return open.empty();
}
```

The demo puts six lines into a queue, one of them malformed, and prints the summary:

```cpp
// src/main.cpp
#include <iostream>
#include <queue>
#include <string>
#include <vector>

#include "log_summary.hpp"

int main() {
    const std::vector<std::string> raw{
        "INFO web: server started",
        "WARN db: slow query",
        "ERROR db: connection refused",
        "INFO web: request served",
        "malformed line",
        "ERROR auth: token expired",
    };

    std::queue<std::string> pending;
    for (const std::string& line : raw) {
        pending.push(line);
    }

    const std::vector<LogEntry> entries = drain(pending);
    std::cout << "parsed " << entries.size() << " of " << raw.size() << " lines\n";

    for (const auto& [level, count] : count_levels(entries)) {
        std::cout << level << ": " << count << '\n';
    }

    std::cout << "components:";
    for (const std::string& name : unique_components(entries)) {
        std::cout << ' ' << name;
    }
    std::cout << '\n';

    std::cout << "errors:\n";
    for (const LogEntry& entry : filter_level(entries, "ERROR")) {
        std::cout << "  " << entry.message << '\n';
    }

    std::cout << std::boolalpha;
    std::cout << "balanced: " << brackets_balanced("f(x[0]) + {y}")
              << ", unbalanced: " << brackets_balanced("(]") << '\n';
    return 0;
}
```

The test checks the parser, the queue order, the map and set results, the filter and the bracket rule:

```cpp
// tests/log_summary_test.cpp
#include <cstdlib>
#include <iostream>
#include <queue>
#include <string>
#include <vector>

#include "log_summary.hpp"

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
    const auto good = parse_line("ERROR db: connection refused");
    check(good.has_value(), "a valid line parses");
    check(good && good->level == "ERROR" && good->component == "db" &&
              good->message == "connection refused",
          "fields are split correctly");
    check(!parse_line("malformed line").has_value(), "a line without a colon is rejected");
    check(!parse_line("ERROR: x").has_value(), "a line without a component is rejected");

    std::queue<std::string> pending;
    pending.push("INFO web: first");
    pending.push("ERROR db: second");
    pending.push("malformed");
    const std::vector<LogEntry> entries = drain(pending);
    check(entries.size() == 2, "drain skips the malformed line");
    check(!entries.empty() && entries[0].component == "web", "drain keeps first-in order");

    const auto counts = count_levels(entries);
    check(counts.size() == 2, "two levels were counted");
    check(counts.count("INFO") == 1 && counts.at("ERROR") == 1, "each level counted once");

    check(unique_components(entries).size() == 2, "two unique components");
    check(filter_level(entries, "ERROR").size() == 1, "one error entry");

    check(brackets_balanced("{[()]}"), "nested brackets balance");
    check(!brackets_balanced("([)]"), "crossed brackets do not balance");
    check(!brackets_balanced("("), "an open bracket does not balance");
    check(brackets_balanced(""), "empty text balances");

    if (failures == 0) {
        std::cout << "all log summary tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the library, the demo and the test:

```cmake
cmake_minimum_required(VERSION 3.25)
project(log_summary LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_library(log_core STATIC src/log_summary.cpp)
target_include_directories(log_core PUBLIC src)

add_executable(log_demo src/main.cpp)
target_link_libraries(log_demo PRIVATE log_core)

add_executable(log_summary_test tests/log_summary_test.cpp)
target_link_libraries(log_summary_test PRIVATE log_core)

enable_testing()
add_test(NAME log_summary_test COMMAND log_summary_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add log summary with containers".
2. Build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Run `ctest --test-dir build --output-on-failure` and confirm it passes.
3. Run `./build/log_demo` and check the output: parsed 5 of 6 lines, ERROR 2, INFO 2, WARN 1, components auth db web, two error messages, and balanced true with unbalanced false.
4. Create a branch `feature/from-file`. Write a test first for a function `std::queue<std::string> load_lines(const std::string& path)` that reads a text file into a queue, and returns an empty queue for a missing file. Implement it and make the test pass.
5. Add a `sort_by_count` function that returns the levels as a `std::vector<std::pair<std::string, int>>` sorted by count, highest first, using a lambda comparator. Break ties alphabetically. Write a test for the tie rule.
6. Write `docs/container-table.md` with a table that maps each container in this project to the function that uses it, the access pattern and the reason it fits. Add one sentence on a container you would not use and why.
7. Merge with `git merge --no-ff feature/from-file -m "Load log lines from a file"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes on `main`.
- The demo prints the output listed in task 3.
- `docs/container-table.md` lists each container with its function and reason.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/container-table.md`, `README.md`, and the history on GitHub.

This project is short but shows five containers and two algorithms in one place. A reviewer should be able to see why each container is the one in its function signature.

---

## Check your understanding

1. A program searches a `std::vector` of one million records by ID inside a loop over another million events. Name the cost, and choose a container that makes the lookup cheaper.
2. A loop walks a `std::map` and inserts new keys into the same map. Explain which iterators stay valid, and which container's rule would make the same code unsafe.
3. A developer uses `operator[]` to check whether a key exists in a map. Describe what happens to the map, and name the function that checks without inserting.
4. Why does `std::set` not need a call to `std::sort`, and what does it cost to keep it sorted on each insert?
5. A comparator for `std::sort` returns `true` when two elements are equal. Explain why this is a bug, and what the correct rule is.
6. Why is `std::queue` a container adaptor instead of a separate container, and what does the narrow interface prevent?

### Answers

1. The loop does a linear search of one million records for each of a million events, which is on the order of 10 to the power of 12 comparisons. A `std::unordered_map` or a `std::map` keyed by ID reduces each lookup to average O(1) or O(log n).
2. Iterators into a `std::map` stay valid after insertions, because the nodes do not move. A `std::vector` would be unsafe, because an insertion can reallocate and invalidate every iterator.
3. `operator[]` inserts a default-constructed value for a missing key, so the check adds an entry to the map. Use `contains` or `find`, which check without inserting.
4. The set keeps its elements in order by its structure, so no separate sort is needed. Each insert costs O(log n) to find its place, while a vector would need a full sort after each change.
5. A strict weak ordering requires that a comparator return `false` for equal elements. Returning `true` for equal elements can cause `std::sort` to read outside the range or loop forever, which is undefined behavior. The correct rule is "less than" in the sense of strict ordering.
6. A queue is a `std::deque` with a narrow interface. The narrow interface prevents callers from reading or changing the middle of the line, which keeps the first-in, first-out rule enforced by the type.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Associative array](https://en.wikipedia.org/wiki/Associative_array) for the map concept across languages
- [Wikipedia: Queue (abstract data type)](https://en.wikipedia.org/wiki/Queue_(abstract_data_type)) and [Wikipedia: Stack (abstract data type)](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) for the two access patterns used in this day
- [Wikipedia: Standard Template Library](https://en.wikipedia.org/wiki/Standard_Template_Library) for the history and design of the library
- cppreference.com, the pages for `std::map`, `std::set`, `std::queue`, `std::stack`, lambda expressions and the `<algorithm>` header. Each container page lists its complexity and invalidation rules.
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on the standard library and the rule to prefer the library over hand-written loops

*Tutorials*

- learncpp.com, the chapters on the standard library containers and lambda expressions, written by Alex Allain
- GeeksforGeeks, search the site for "C++ map", "C++ set" and "C++ lambda expressions" for the matching articles
- W3Schools, the C++ "STL" and "Lambda" pages, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on the standard containers and lambdas
- CppCon, the YouTube channel, for talks on containers. Search the channel for "STL" or "container" to find them.

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, with exercises that use maps, sets and queues. Run each test suite from your terminal.

---

## Tomorrow

Day 37 covers modern C++: smart pointers, move semantics, `auto`, range-based `for`, and `std::optional`. You have already used `std::optional` in the log parser and moved values with `std::move`. Day 37 explains the rules behind both, and shows how `std::unique_ptr` and `std::shared_ptr` take over the ownership job that you did by hand in the Day 32 buffer. The containers from today store those smart pointers without any change to their behavior.

**Next:** [Day 37: Modern C++](Software-Day-37-Modern-C-Plus-Plus.md)

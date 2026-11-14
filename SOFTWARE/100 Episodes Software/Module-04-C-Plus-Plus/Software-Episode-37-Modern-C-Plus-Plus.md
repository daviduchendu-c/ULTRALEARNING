# Day 37: Modern C++

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You have written classes with destructors from Day 32, used containers and lambdas from Day 36, and seen `std::move` and `std::optional` in code. This day explains what those features do and how they replace most of the manual memory management from the earlier days. It is for anyone whose C++ still looks like the C++ of ten years ago.

**By the end of this day, you will have:**

- Used `std::unique_ptr` and `std::shared_ptr` to express ownership, and `std::weak_ptr` to observe without owning
- Explained move semantics: what a move constructor does, why a moved-from object must still be destroyable, and when `std::move` is only a cast
- Used `auto`, range-based `for` and structured bindings, and explained the copy each one can hide
- Used `std::optional` to return a value that may be missing, and `value_or` to supply a default
- A small document registry that owns its documents through smart pointers and moves them in without copying, with a test executable run by CTest
- A GitHub repo with the build, the test output and a note that records the ownership of each object in the registry

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Days 27 to 36: the build, classes, lifetime and RAII, containers and lambdas
- Day 35: templates, because `std::unique_ptr`, `std::shared_ptr` and `std::optional` are all class templates

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Smart pointers](#smart-pointers)
- [Move semantics](#move-semantics)
- [Auto](#auto)
- [Range-based for loops](#range-based-for-loops)
- [Optional values](#optional-values)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a document registry with smart pointers and moves](#project-a-document-registry-with-smart-pointers-and-moves)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Day 32 showed a class that owns heap memory and frees it in its destructor, which is RAII. That pattern is correct, but writing it for every resource is repetitive and easy to get wrong. Modern C++ provides standard types that do the job for you. `std::unique_ptr` owns one object and frees it when it goes out of scope. `std::shared_ptr` lets several owners share one object, and frees it when the last owner leaves. `std::optional` says that a value may be absent, without a null pointer.

The other large change is move semantics. Before C++11, returning a large object or putting one into a container meant copying it, and programmers avoided that by passing pointers everywhere. A move transfers the internal resources of an object to another object, leaving the source in a valid but unspecified state. Moves are why the vector grows quickly and why returning a big object by value is cheap.

This day teaches these tools in the order you need them. The rules come from the ISO C++ standard, and cppreference.com has a page for each smart pointer, for move semantics and for `std::optional`. Read the page for `std::unique_ptr` once, because its ownership rules are the ones you will use most.

---

## Glossary

- **Smart pointer:** a class template that owns a pointer and releases the object it points to, usually in its destructor.
- **Unique ownership:** the rule that exactly one owner exists, expressed by `std::unique_ptr`.
- **Shared ownership:** the rule that several owners share one object, expressed by `std::shared_ptr`, which keeps a reference count.
- **Reference count:** the number of `shared_ptr` objects that point to the same object. The object is destroyed when the count reaches zero.
- **Weak reference:** a non-owning reference that can check whether the object still exists, expressed by `std::weak_ptr`.
- **Move constructor:** a constructor that takes an rvalue reference and takes over the resources of its source.
- **Rvalue reference:** a reference written `T&&`, which can bind to a temporary or to an object explicitly marked with `std::move`.
- **Moved-from object:** an object whose resources have been taken by a move. It must still be safe to destroy and assign to.
- **Type deduction:** the compiler's choice of a variable's type from its initializer, written with `auto`.
- **Structured binding:** a declaration that unpacks a pair, tuple or struct into named variables, such as `auto [key, value] = pair;`.
- **Optional:** `std::optional<T>`, a value that either holds a `T` or holds nothing.

---

## Core concepts

The concepts follow the ownership question first, then the move that transfers ownership, then the features that make everyday code shorter and safer.

### Smart pointers

A smart pointer is a handle that owns something. Think of a rented car with a key card. The key card is the handle, and the rental agreement says who is responsible for returning the car. `std::unique_ptr` is one renter with one key. When the renter leaves the scope, the car is returned. `std::shared_ptr` is a shared fleet card held by several drivers. The car is returned only when the last driver gives up the card. A `std::weak_ptr` is a note that says where the car was parked, without holding a card.

In first principles, `unique_ptr` holds a raw pointer and deletes it in its destructor. It cannot be copied, because two owners would delete the same object. It can be moved, which transfers the single owner. `shared_ptr` holds a pointer and a control block with the reference count, and copying it increments the count. `weak_ptr` refers to the same control block, so it can check whether the object is gone without keeping it alive.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <utility>

struct Report {
    std::string name;

    explicit Report(std::string n) : name(std::move(n)) {
        std::cout << "open " << name << '\n';
    }

    ~Report() {
        std::cout << "close " << name << '\n';
    }
};

int main() {
    auto owned = std::make_unique<Report>("sales");
    std::shared_ptr<Report> shared = std::make_shared<Report>("audit");
    std::weak_ptr<Report> watcher = shared;

    std::cout << "watcher alive: " << !watcher.expired() << '\n';
    shared.reset();
    std::cout << "watcher alive after reset: " << !watcher.expired() << '\n';
    std::cout << "end of main\n";
    return 0;
}
```

The output is `open sales`, `open audit`, `watcher alive: 1`, `close audit`, `watcher alive after reset: 0`, `end of main`, and finally `close sales`. The `reset` call dropped the last strong reference, so the audit report closed immediately. The unique report lived until `main` returned. Use `std::make_unique` and `std::make_shared` rather than `new`, because they avoid a leak if an exception is thrown between the allocation and the smart pointer's construction.

The common mistake is to create two independent `shared_ptr` objects from one raw pointer, such as `std::shared_ptr<T>(p)` twice. Each one has its own control block, so the object is deleted twice. The fix is to copy the first `shared_ptr` or to use `shared_from_this` when an object must hand out shared ownership of itself. You can spot the bug as a crash at the end of `main`. In the market, teams standardize on `unique_ptr` as the default owner, and reserve `shared_ptr` for objects with a real shared lifetime, because reference counting has a cost on every copy.

### Move semantics

A move transfers the resources of one object to another. Think of moving house. You do not buy new furniture for the new house. You pack the furniture from the old one and place it in the new one, and the old house is empty. The furniture still exists, and it now belongs to the new home. A moved-from object is the empty old house: it is safe to use again, but its contents are no longer the ones it had.

In first principles, a move constructor takes an rvalue reference, which can bind to a temporary. It copies the pointers and sizes from the source, then sets the source's pointers to null or zero. The source's destructor then does nothing harmful. `std::move` is only a cast that turns an lvalue into an rvalue reference, so the compiler can choose the move constructor.

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <vector>

int main() {
    std::vector<std::string> source{"alpha", "beta"};
    std::vector<std::string> target = std::move(source);

    std::cout << "target has " << target.size() << " items\n";
    std::cout << "first is " << target[0] << '\n';
    return 0;
}
```

The output is `target has 2 items` and `first is alpha`. The vector's buffer moved from `source` to `target` without copying the strings. The `source` object is now empty in practice, but the standard promises only that it is valid and can be assigned to or destroyed. Do not read from a moved-from object as if it still held its old values.

The common mistake is to call `std::move` on a value you still need, such as a `const` object or a value you read later. A `const` object cannot be moved from, so the copy happens silently. The other mistake is to write `return std::move(local);`, which blocks an optimization the compiler would have done automatically. The rule is to return a local by name and let the compiler decide. You can spot the bug when an object is empty after a function call that should not have emptied it. In the market, move semantics are why modern C++ APIs return containers by value without performance penalties.

### Auto

`auto` tells the compiler to deduce a variable's type from its initializer. Think of a label printer that reads the item and prints the right label, so you do not write it by hand. The deduced type is exact, so `auto` does not change what the program does. It only saves you from writing a long type name, or from naming a type you cannot easily write, such as an iterator from a complex template.

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> scores{{"Ana", 91}, {"Ben", 78}};

    for (const auto& [name, score] : scores) {
        std::cout << name << ' ' << score << '\n';
    }

    auto total = 0;
    for (const auto& entry : scores) {
        total += entry.second;
    }
    std::cout << "total " << total << '\n';
    return 0;
}
```

The output is `Ana 91`, `Ben 78` and `total 169`. The `auto` in `auto total = 0;` is an `int`, because the initializer is an `int`. The structured binding `[name, score]` unpacks each pair. The `const auto&` form avoids a copy of each pair, which matters when the element is large.

The common mistake is to write `auto x = some_function_returning_a_reference();` and expect a reference, when `auto` deduces a value and copies. Use `auto&` or `const auto&` when you want a reference. Another mistake is to use `auto` where the type is important to the reader, such as a conversion result, which can hide a narrowing mistake. In the market, style guides differ on `auto`, but most teams use it for iterators, lambdas and structured bindings, and spell out types for numbers.

### Range-based for loops

A range-based `for` loop walks every element of a range without an index or an explicit iterator. Think of a conveyor belt that delivers each item to you in order. You say what you want to do with each item, and the belt handles the rest. The loop works with any type that has `begin` and `end`, including arrays, vectors, maps and your own containers.

The loop variable decides whether you copy, reference or modify the elements. Use `const auto&` to read large elements without copying. Use `auto&` to change them. A plain `auto` copies each element, which is correct for small types and wasteful for large ones.

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> values{1, 2, 3};

    for (int& value : values) {
        value *= 10;
    }
    for (const int value : values) {
        std::cout << value << '\n';
    }
    return 0;
}
```

The output is `10`, `20` and `30`, each on its own line. The first loop uses `int&`, so it changes the vector. The second loop reads each value through a copy, which is fine for `int`. The limit of the range-based loop is that it does not give you the index, so when you need the position, use an index loop or `std::views::enumerate` from C++23.

The common mistake is to modify a container while iterating over it with a range-based loop, such as adding elements to a vector inside the loop. That can invalidate the iterators the loop uses, which is undefined behavior. The fix is to collect changes and apply them after the loop. In the market, range-based loops are the default style in modern codebases, and reviewers look for copies of large objects in loop variables.

### Optional values

`std::optional<T>` holds either a value of type `T` or nothing. Think of a parcel locker that is either empty or holds one parcel. You check the locker before you open it. The type makes the possibility of absence visible in the signature, which a null pointer does not do as clearly.

An `optional` is a better return type than a null pointer when a missing result is normal and expected, such as a parse that can fail or a lookup that can miss. The caller must check the optional before using the value, or use `value_or` to supply a default:

```cpp
#include <exception>
#include <iostream>
#include <optional>
#include <string>

std::optional<int> parse_int(const std::string& text) {
    try {
        return std::stoi(text);
    } catch (const std::exception&) {
        return std::nullopt;
    }
}

int main() {
    if (auto n = parse_int("42")) {
        std::cout << "parsed " << *n << '\n';
    }
    std::cout << "fallback " << parse_int("x").value_or(-1) << '\n';
    return 0;
}
```

The output is `parsed 42` and `fallback -1`. The function returns a value on success and `std::nullopt` on failure. The `if` with an initializer checks the optional and gives access to the value. The `value_or` call provides the default without a branch.

The common mistake is to dereference an optional without checking it. `*n` on an empty optional is undefined behavior, and `value()` throws `std::bad_optional_access`, which is safer but still an error. Use `value_or`, a check, or `value()` only when an empty value is a real bug. You can spot the mistake when a program crashes only on the failure path. In the market, optional return types are standard in parsers, database lookups and configuration readers, because they make the failure path part of the signature.

---

## Hands-on exercises

Work in a new folder called `modern-cpp`. Run `git init -b main` in it.

Exercise 1 uses `unique_ptr`.

1. Write a class `Sensor` that prints its name in its constructor and destructor.
2. Create a `std::unique_ptr<Sensor>` with `std::make_unique`, then move it into a second `unique_ptr`. Print a message after the move.
3. Try to copy a `unique_ptr` and read the compiler error.

Done when the object is destroyed exactly once and the copy error text is in your notes.

Exercise 2 uses `shared_ptr` and `weak_ptr`.

1. Create a `shared_ptr` to a `Sensor`, make a copy of it in an inner scope, and check the `use_count()` at each step.
2. Create a `weak_ptr` from the shared pointer, reset the shared pointer, and check `expired()`.

Done when the use counts you print match your predictions.

Exercise 3 moves a vector.

1. Create a `std::vector<std::string>` with three long strings. Move it into a second vector.
2. Print the size of both vectors. Then assign a new value to the first one and print again.

Done when you can explain in one sentence why the moved-from vector can still be assigned.

Exercise 4 returns an optional.

1. Write `std::optional<double> average(const std::vector<int>& values)` that returns nothing for an empty vector.
2. Call it with an empty vector and a non-empty one. Use `value_or` for the empty case.

Done when the empty case prints your default and the non-empty case prints the average.

Exercise 5 compares copy and reference in a loop.

1. Write a loop over a `std::vector<std::string>` with `for (auto s : v)`, then with `for (const auto& s : v)`.
2. Time each loop over ten million iterations with `std::chrono`, or run each under a profiler if you have one.

Done when you can say which loop copies and which one does not.

---

## Project: a document registry with smart pointers and moves

The project is a registry of documents. A document is a title and a list of lines. The registry takes ownership of a document when it is published, shares that ownership with the caller, and lets anyone watch a document through a weak pointer. The first line of a document is returned as an optional, because an empty document has none.

Suggested repo name: `100-days-software-day-37-modern-cpp`.

The repo layout:

```
100-days-software-day-37-modern-cpp/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── ownership.md
├── src/
│   ├── document.cpp
│   ├── document.hpp
│   ├── main.cpp
│   ├── registry.cpp
│   └── registry.hpp
└── tests/
    └── registry_test.cpp
```

The document class has defaulted move operations, which are marked `noexcept` so that containers can use them during reallocation:

```cpp
// src/document.hpp
#ifndef DOCUMENT_HPP
#define DOCUMENT_HPP

#include <cstddef>
#include <optional>
#include <string>
#include <vector>

class Document {
public:
    explicit Document(std::string title);
    Document(std::string title, std::vector<std::string> lines);

    Document(const Document&) = default;
    Document& operator=(const Document&) = default;
    Document(Document&&) noexcept = default;
    Document& operator=(Document&&) noexcept = default;
    ~Document() = default;

    const std::string& title() const { return title_; }
    std::size_t line_count() const { return lines_.size(); }
    void append(std::string line);
    std::optional<std::string> first_line() const;

private:
    std::string title_;
    std::vector<std::string> lines_;
};

#endif
```

The implementation validates the title in the two-argument constructor, and the one-argument constructor delegates to it:

```cpp
// src/document.cpp
#include "document.hpp"

#include <stdexcept>
#include <utility>

Document::Document(std::string title)
    : Document(std::move(title), std::vector<std::string>{}) {}

Document::Document(std::string title, std::vector<std::string> lines)
    : title_(std::move(title)), lines_(std::move(lines)) {
    if (title_.empty()) {
        throw std::invalid_argument("title must not be empty");
    }
}

void Document::append(std::string line) {
    lines_.push_back(std::move(line));
}

std::optional<std::string> Document::first_line() const {
    if (lines_.empty()) {
        return std::nullopt;
    }
    return lines_.front();
}
```

The registry owns documents through `shared_ptr`. Publishing takes a `unique_ptr` and converts it, which is the normal direction: exclusive ownership becomes shared ownership once the object is ready to be shared:

```cpp
// src/registry.hpp
#ifndef REGISTRY_HPP
#define REGISTRY_HPP

#include <cstddef>
#include <map>
#include <memory>
#include <optional>
#include <string>

#include "document.hpp"

class Registry {
public:
    // Takes ownership of the draft and returns a shared handle to it.
    std::shared_ptr<Document> publish(std::unique_ptr<Document> draft);

    std::optional<std::string> first_line(const std::string& title) const;
    std::weak_ptr<Document> watch(const std::string& title) const;
    bool remove(const std::string& title);
    std::size_t size() const { return docs_.size(); }

private:
    std::map<std::string, std::shared_ptr<Document>> docs_;
};

#endif
```

```cpp
// src/registry.cpp
#include "registry.hpp"

#include <stdexcept>
#include <utility>

std::shared_ptr<Document> Registry::publish(std::unique_ptr<Document> draft) {
    if (!draft) {
        throw std::invalid_argument("draft must not be null");
    }
    if (docs_.contains(draft->title())) {
        throw std::invalid_argument("title already published: " + draft->title());
    }

    std::shared_ptr<Document> published = std::move(draft);
    docs_.emplace(published->title(), published);
    return published;
}

std::optional<std::string> Registry::first_line(const std::string& title) const {
    const auto it = docs_.find(title);
    if (it == docs_.end()) {
        return std::nullopt;
    }
    return it->second->first_line();
}

std::weak_ptr<Document> Registry::watch(const std::string& title) const {
    const auto it = docs_.find(title);
    if (it == docs_.end()) {
        return {};
    }
    return it->second;
}

bool Registry::remove(const std::string& title) {
    return docs_.erase(title) > 0;
}
```

The demo builds two documents, publishes them and removes one, then shows that a weak watcher sees the removal:

```cpp
// src/main.cpp
#include <iostream>
#include <memory>
#include <string>

#include "registry.hpp"

Document make_plan() {
    Document plan("Plan", {"step one", "step two"});
    return plan;
}

int main() {
    Registry registry;

    auto draft = std::make_unique<Document>("Notes");
    draft->append("first note");
    draft->append("second note");
    std::shared_ptr<Document> notes = registry.publish(std::move(draft));

    registry.publish(std::make_unique<Document>(make_plan()));
    std::weak_ptr<Document> watcher = registry.watch("Plan");

    std::cout << "published: " << notes->title() << '\n';
    std::cout << "documents: " << registry.size() << '\n';
    std::cout << "first line of Notes: " << registry.first_line("Notes").value_or("none") << '\n';
    std::cout << "first line of Missing: " << registry.first_line("Missing").value_or("none") << '\n';

    registry.remove("Plan");
    std::cout << "Plan expired after removal: " << watcher.expired() << '\n';
    return 0;
}
```

The output is `published: Notes`, `documents: 2`, `first line of Notes: first note`, `first line of Missing: none`, and `Plan expired after removal: 1`. The `Plan` document had no other owner after `remove`, so it was freed, and the weak pointer reports that.

The test covers the optional results, the duplicate rule, the moved document, the weak observer and the empty title:

```cpp
// tests/registry_test.cpp
#include <cstdlib>
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>

#include "registry.hpp"

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
    Document empty("Empty");
    check(!empty.first_line().has_value(), "an empty document has no first line");

    Document moved_from("A", {"x", "y"});
    Document moved = std::move(moved_from);
    check(moved.title() == "A" && moved.line_count() == 2, "a moved document keeps its contents");

    Registry registry;
    auto draft = std::make_unique<Document>("Notes");
    draft->append("first");
    auto published = registry.publish(std::move(draft));
    check(registry.size() == 1, "one document after publish");
    check(registry.first_line("Notes") == std::optional<std::string>("first"),
          "first line is returned through the optional");

    bool duplicate = false;
    try {
        registry.publish(std::make_unique<Document>("Notes"));
    } catch (const std::invalid_argument&) {
        duplicate = true;
    }
    check(duplicate, "a duplicate title is rejected");

    bool empty_title = false;
    try {
        Document bad("");
    } catch (const std::invalid_argument&) {
        empty_title = true;
    }
    check(empty_title, "an empty title is rejected");

    std::weak_ptr<Document> observer;
    {
        auto plan = registry.publish(std::make_unique<Document>("Plan"));
        observer = registry.watch("Plan");
        check(!observer.expired(), "watch sees a published document");
        check(registry.remove("Plan"), "remove reports success");
        check(!observer.expired(), "the caller's shared handle keeps the document alive");
    }
    check(observer.expired(), "the document is freed after the last owner leaves");

    if (failures == 0) {
        std::cout << "all registry tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the library, the demo and the test:

```cmake
cmake_minimum_required(VERSION 3.25)
project(modern_registry LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_library(registry_core STATIC src/document.cpp src/registry.cpp)
target_include_directories(registry_core PUBLIC src)

add_executable(registry_demo src/main.cpp)
target_link_libraries(registry_demo PRIVATE registry_core)

add_executable(registry_test tests/registry_test.cpp)
target_link_libraries(registry_test PRIVATE registry_core)

enable_testing()
add_test(NAME registry_test COMMAND registry_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add document registry with smart pointers".
2. Build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Run `ctest --test-dir build --output-on-failure` and confirm it passes.
3. Run `./build/registry_demo` and check each line against the output described above.
4. Create a branch `feature/archive`. Write a test first for `bool Registry::archive(const std::string& title)` that moves a document out of the registry and returns a `std::unique_ptr<Document>` through an output parameter of type `std::unique_ptr<Document>&`. Make the test pass with the move from `docs_` into the parameter.
5. In the test, check that the archived document's `title()` is correct and that `size()` dropped by one.
6. Write `docs/ownership.md` with three short sections: who owns each document while it is in the registry, who owns it after a caller keeps the shared handle, and what changes after `archive`. Quote one line from your demo output.
7. Merge with `git merge --no-ff feature/archive -m "Add archive with unique ownership"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes on `main`.
- The demo prints the five lines from task 3.
- `docs/ownership.md` names the owner of a document at three points in its life.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/ownership.md`, `README.md`, and the history on GitHub.

This project is short and ownership-focused. A reviewer should be able to read the registry header and say, for every pointer, who frees it and when.

---

## Check your understanding

1. A function returns `std::unique_ptr<Widget>` and the caller stores it in a `std::vector<std::unique_ptr<Widget>>`. Explain why this compiles, and what happens to each widget when the vector is destroyed.
2. Two `shared_ptr` objects are created from the same raw pointer `new Widget()`. Describe the failure, and the correct way to create the second owner.
3. A moved-from `std::string` is printed after the move. Explain why the output cannot be relied on, and what operations are safe on it.
4. A loop `for (auto item : items)` is slow for a vector of large structs. Name the copy and the fix.
5. A function returns `std::optional<Config>` and the caller uses `*result` without checking. Explain the risk and the two safer alternatives.
6. Why is `return std::move(local);` usually a mistake, and what should you write instead?

### Answers

1. `std::unique_ptr` can be moved, so returning it and storing it in the vector transfers ownership. When the vector is destroyed, it destroys each `unique_ptr`, which deletes each widget exactly once.
2. Each `shared_ptr` has its own control block, so each one believes it is the only owner. When both go out of scope, the object is deleted twice. The correct way is to copy the first `shared_ptr`, which shares the control block.
3. The standard promises only that a moved-from object is valid and can be destroyed or assigned to. The contents are unspecified, so the output can be empty or the old value. Assigning a new value or destroying the object is safe, and reading the old value is not.
4. Each iteration copies the struct into `item`. Use `const auto& item` to read without copying, or `auto& item` to change it in place.
5. If the optional is empty, `*result` is undefined behavior. Check with `if (result)` before using it, or use `result.value()`, which throws a clear exception, or `value_or` with a default.
6. It can block copy elision, which removes the copy for free, and it forces a move in its place when the compiler could have skipped both. Write `return local;` and let the compiler decide, because it treats a returned local as an rvalue automatically.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Smart pointer](https://en.wikipedia.org/wiki/Smart_pointer) for the general concept
- [Wikipedia: Reference counting](https://en.wikipedia.org/wiki/Reference_counting) for how `shared_ptr` counts owners
- [Wikipedia: Move semantics](https://en.wikipedia.org/wiki/Move_semantics) for the idea behind moves
- cppreference.com, the pages for `std::unique_ptr`, `std::shared_ptr`, `std::weak_ptr`, `std::optional`, "Move constructors", and "Range-based for loop"
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on resource management and smart pointers, including the rule to use `unique_ptr` as the default owner

*Tutorials*

- learncpp.com, the chapters on smart pointers, move semantics, `auto`, and `std::optional`, written by Alex Allain
- GeeksforGeeks, search the site for "C++ smart pointers", "C++ move semantics" and "C++ optional" for the matching articles
- W3Schools, the C++ "Smart Pointers" page, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on smart pointers and move semantics
- CppCon, the YouTube channel, for talks on move semantics. Search the channel for "move semantics" to find them.

*Practice*

- Compiler Explorer at godbolt.org. Paste the `make_plan` function from the demo and compare the output with and without `-fno-elide-constructors`. The comparison shows the copy the compiler removes.

---

## Tomorrow

Day 38 covers errors, debugging and testing. You will use exceptions with the RAII habit from Day 32, assertions for conditions that must never happen, the debugger, the AddressSanitizer and UndefinedBehaviorSanitizer, and a unit test framework, GoogleTest, wired into CTest. The smart pointers from today make the leak checks quiet, and the sanitizers are what will catch the bugs that the types do not prevent.

**Next:** [Day 38: Errors, Debugging and Testing](Software-Day-38-Errors-Debugging-and-Testing.md)

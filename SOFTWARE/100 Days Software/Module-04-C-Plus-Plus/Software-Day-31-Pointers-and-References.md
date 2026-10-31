# Day 31: Pointers and References

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You have passed values by reference since Day 29 and used iterators since Day 30. This day explains what those tools are built on: memory addresses. It is for anyone who has seen `*` and `&` in an error message and wanted to know what they actually do.

**By the end of this day, you will have:**

- Taken the address of a variable, dereferenced a pointer and changed the value through it
- Used `nullptr` to mark a pointer that points at nothing, and checked it before every use
- Explained how a reference differs from a pointer, and when each one is the right tool
- Used pointer arithmetic to walk an array and sum its values
- A small pointer library with a non-owning view type, a test executable run by CTest, and a README that records the safety rules
- A GitHub repo that shows a clean build and a test run, plus a short note on the dangling pointer you avoided

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- Optional for the safety exercise: Ubuntu 24.04 in WSL 2 with `sudo apt install g++-14 clang-18` and AddressSanitizer, which ships with GCC
- A free GitHub account

**Prerequisites:**

- Days 27 to 30: the build, functions, reference parameters, arrays, vectors and iterators
- Day 28: integer types and the `const` keyword

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Addresses](#addresses)
- [Dereferencing](#dereferencing)
- [Nullptr](#nullptr)
- [References](#references)
- [Pointer arithmetic](#pointer-arithmetic)
- [Safety](#safety)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a pointer toolkit with a non-owning view](#project-a-pointer-toolkit-with-a-non-owning-view)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every variable lives somewhere in memory, and that location has a number called an address. A pointer is a variable that stores an address. A reference is a second name for an existing variable. Both let a function or a data structure reach a value without copying it, and both are behind almost every bug in C and C++ programs that crash in production.

The tools are simple. The discipline is the hard part. A pointer can point at nothing, at memory that has been freed, or at a local variable whose lifetime ended when its function returned. A reference cannot be null, but it can still outlive the object it names. Modern C++ reduces the number of raw pointers you write, and the standard library hides most of them, but you will still read code that uses them, and you need to know when a raw pointer is the right tool.

The rules are in the ISO C++ standard, and cppreference.com has a page on pointers and one on references. Read both once while you work through this day.

---

## Glossary

- **Address:** the number that identifies a memory location, usually shown in hexadecimal.
- **Pointer:** a variable whose value is an address of another object, declared with `*`, such as `int* p`.
- **Dereference:** following a pointer to the object it points at, written `*p`.
- **Address-of operator:** the `&` operator applied to a variable, which gives its address.
- **Null pointer:** a pointer with no object, written `nullptr` in modern C++.
- **Reference:** another name for an existing object, declared with `&`, such as `int& r`.
- **Pointer arithmetic:** adding or subtracting an integer to a pointer, which moves it by whole elements.
- **Dangling pointer:** a pointer that holds the address of an object whose lifetime has ended.
- **Non-owning:** a pointer or view that refers to memory another object owns and must not free.
- **Use-after-free:** reading or writing memory through a pointer after it was released. This is undefined behavior.
- **Aliasing:** two names, such as a reference and the original variable, that refer to the same object.

---

## Core concepts

The concepts in this day form a path from the raw address to the safety rules. Addresses are the foundation, dereferencing is the way to use them, `nullptr` marks their absence, references are a safer name for the same idea, and pointer arithmetic is what makes arrays work. The safety section ties them together.

### Addresses

Think of a house on a street. The house has a value inside it, which is the furniture and the people, and it has an address, which tells a visitor where to find it. A pointer is a note that holds the address. The note points to the house, and copying the note does not copy the house.

In first principles, the computer's memory is a long row of bytes, each with an index. A variable occupies a few consecutive bytes, and the address is the index of the first one. The `&` operator asks for that index:

```cpp
#include <cstdint>
#include <iostream>

int main() {
    int score = 70;
    int* p = &score;

    std::cout << "score value: " << score << '\n';
    std::cout << "p points at value: " << *p << '\n';
    std::cout << "address printed as an unsigned number is not zero: "
              << (reinterpret_cast<std::uintptr_t>(p) != 0) << '\n';
    return 0;
}
```

The first two lines print `score value: 70` and `p points at value: 70`. The third line converts the address to `std::uintptr_t`, an integer type from `<cstdint>` that is large enough to hold a pointer, and prints `1` because the address is not zero. The address itself differs on every run, because the operating system chooses where the variable lives. Never write code that depends on a specific address.

The declaration `int* p` reads as "p is a pointer to int." The position of the star does not matter to the compiler, but `int* p, q;` declares only one pointer, because the star attaches to `p`. Write one declaration per line to avoid this trap.

The common mistake is to think a pointer owns the object it points at, which it never does. A pointer is a number that refers to an object owned by someone else, and the object may disappear while the pointer still holds the old number. You can spot the mistake when a pointer is returned from a function and the object was a local variable. In the market, the question "who owns this pointer's target" is the first thing reviewers ask in C++ code.

### Dereferencing

To use the value a pointer refers to, you dereference it with `*`. The expression `*p` names the object at the address, so you can read it or assign to it. Writing `*p = 85` changes the original `score`, because `p` leads to the same memory.

```cpp
#include <iostream>

int main() {
    int score = 70;
    int* p = &score;

    *p = 85;
    std::cout << "score after dereference write: " << score << '\n';

    const int* reader = &score;
    std::cout << "read through const pointer: " << *reader << '\n';
    return 0;
}
```

The output is `score after dereference write: 85` and `read through const pointer: 85`. The second pointer is a `const int*`, which means the pointed-to value cannot be changed through it. The pointer itself can still point somewhere else. `int* const` is the reverse: the pointer cannot change, but the value can. Read the declaration from right to left to decide which one you have.

Dereferencing an invalid pointer is the most common crash in C and C++. A pointer that holds garbage, a null value or a freed address causes undefined behavior when dereferenced. The program may crash immediately, continue with wrong data, or appear to work until a later change exposes the bug.

The common mistake is to dereference a pointer that was never assigned. An uninitialized pointer contains an arbitrary value, which is worse than a null pointer because it looks valid. Always initialize pointers, even if the initial value is `nullptr`. In the market, uninitialized pointers appear in embedded firmware and device drivers, where a wild write can damage a neighboring structure and the symptom appears far from the cause.

### Nullptr

A null pointer means "points at nothing." Before C++11, programmers used `NULL` or the literal `0`, which caused overload ambiguity and confusion. `nullptr` is a keyword of type `std::nullptr_t`, and it converts only to pointer types, so it is always clear what you meant.

Checking for null is the habit that prevents most pointer crashes. A function that may return nothing should return `nullptr`, and the caller should check the result before dereferencing:

```cpp
#include <iostream>

int* find_first_negative(int* first, int* last) {
    for (int* p = first; p != last; ++p) {
        if (*p < 0) {
            return p;
        }
    }
    return nullptr;
}

int main() {
    int readings[] = {12, 7, -3, 9};
    int* hit = find_first_negative(readings, readings + 4);

    if (hit != nullptr) {
        std::cout << "first negative: " << *hit << " at index " << (hit - readings) << '\n';
    } else {
        std::cout << "no negative values\n";
    }
    return 0;
}
```

The output is `first negative: -3 at index 2`. Change the array to all positive numbers and the program prints `no negative values`, without ever dereferencing a null pointer. The check costs one comparison, and it is much cheaper than a crash in production.

The limit of null checks is that they only catch null. A pointer to freed memory is not null and passes the check. A null pointer that is used without a check is still a bug, and compilers will not warn you in every case. Tools such as sanitizers, covered on Day 38, catch more of them.

The common mistake is to return a null pointer and let the caller forget to check. The fix is to return `std::optional`, which Day 37 covers, when a missing result is normal. Use a raw pointer when the function is a low-level building block and the convention is documented. In the market, C APIs return null pointers for failure, and wrapper code checks them before any use.

### References

A reference is a second name for an existing object. Think of a nickname for a person. The nickname always refers to the same person, it cannot be unset, and you cannot give it to someone else. In C++, a reference is declared with `&` and must be initialized to refer to an object when it is created.

Because a reference cannot be null and cannot be rebound to another object, it is safer than a pointer for most purposes. Use a reference for a parameter that must refer to an object, and use `const` references for read-only access to large values:

```cpp
#include <iostream>
#include <string>

void append_grade(std::string& report, char grade) {
    report += grade;
}

int main() {
    std::string report = "grades: ";
    append_grade(report, 'A');
    append_grade(report, 'B');

    const std::string& view = report;
    std::cout << view << " (length " << view.size() << ")\n";
    return 0;
}
```

The output is `grades: AB (length 10)`. The function changes `report` in place without copying it, and the `const std::string&` view reads the same object without a copy. Both references refer to the same string, so the change is visible to all of them.

The difference from a pointer is that a reference has no separate address you can inspect or change, and it cannot be null. A pointer can be reassigned, can be null and can be stored in an array. A reference is simpler, which makes it the first choice when the object must exist.

The common mistake is to return a reference to a local variable, which leaves a dangling reference when the function returns. The compiler warns about this in most cases under `-Wall` with the `-Wreturn-local-addr` check, and the fix is to return the value. Another mistake is to store a reference in a class member and let the referred object be destroyed first. In the market, reference-returning accessors on large objects are common, and reviewers check the lifetime of the object they refer to.

### Pointer arithmetic

A pointer into an array can move forward and backward. When you add 1 to an `int*`, the pointer moves by the size of one `int`, not by one byte. Think of rows of seats in a theater. Moving one seat forward takes you to the next seat, whatever its label. C++ does the byte arithmetic for you, as long as the pointer stays inside the same array.

```cpp
#include <iostream>

int main() {
    int values[] = {10, 20, 30, 40};
    int* p = values;

    std::cout << *p << ' ' << *(p + 2) << ' ' << p[3] << '\n';

    int* end = values + 4;
    int total = 0;
    for (int* q = values; q != end; ++q) {
        total += *q;
    }
    std::cout << "total " << total << '\n';
    return 0;
}
```

The output is `10 30 40` on the first line and `total 100` on the second. The expression `p[3]` is the same as `*(p + 3)`, which shows that array indexing is pointer arithmetic in disguise. The `end` pointer points one position past the last element, and you compare against it but never dereference it.

Pointer arithmetic is defined only inside one array, or one position past its end. Moving a pointer past that range and then dereferencing it is undefined behavior, even if the program appears to work. Pointers into different arrays cannot be subtracted to get a meaningful distance.

The common mistake is an off-by-one error in the end pointer. Using `<=` in a loop that compares to `end` reads one element past the array. The fix is to use a half-open range, where the start is included and the end is excluded, as the standard algorithms do. In the market, pointer loops over image buffers and audio samples are where this mistake causes visible glitches, and the fix is a test with the first and last element.

### Safety

Pointers and references are safe when you can answer three questions for each one: who owns the object, how long does the object live, and is the pointer checked before use. The rules below give you a short checklist.

Never return a pointer or reference to a local variable. Never store a pointer to an object that might be destroyed first. Never use a pointer after `delete`, after a vector reallocation or after the container that owns the object has been cleared. Always check a pointer that may be null before dereferencing it. Prefer references when the object must exist and prefer `std::optional` when a value may be missing.

The compiler helps with some of this. GCC and Clang warn about returning the address of a local under `-Wall`, and most dangling-reference bugs are caught by sanitizers at run time. The sanitizer for use-after-free is AddressSanitizer, and it is the subject of Day 38. The sample below shows the bug the rules prevent:

```text
int* make_value() {
    int local = 42;
    return &local;   // dangling: local is destroyed when the function returns
}
```

GCC reports `function returns address of local variable` under `-Wall`, and `-Werror` turns that warning into a failed build. Without the flags, the same code can run and print a garbage number, which is why the warning matters. The correct fix is to return the value by copy, or to take an output reference from the caller.

The common mistake is to believe that a pointer that worked in testing is safe. Dangling pointers often work for a while, because freed memory is not overwritten immediately. The fix is to run tests under a sanitizer. In the market, use-after-free bugs are among the most common root causes of security incidents in native code, which is why companies invest in sanitizers and safer types.

---

## Hands-on exercises

Work in a new folder called `pointers-references`. Run `git init -b main` in it.

Exercise 1 uses addresses and dereferencing.

1. Declare an `int` with the value 50 and a pointer to it. Use the pointer to add 25 to the value.
2. Print the value through the original variable and through the pointer. They should match.
3. Print the address with `static_cast<const void*>(p)` and run the program twice. Record whether the printed address changes.

Done when you can explain in one sentence why the printed address can change between runs.

Exercise 2 uses `nullptr` safely.

1. Write a function `int* find_value(int* first, int* last, int target)` that returns a pointer to the first matching element or `nullptr`.
2. Call it with a value that exists and with one that does not. Print the result only after checking for `nullptr`.

Done when both calls print the right message and the program never dereferences a null pointer.

Exercise 3 compares a reference and a pointer.

1. Write `void double_value(int& x)` and `void double_pointer(int* x)`. Each one doubles its argument.
2. Call `double_pointer` with `nullptr` in one run. Note what happens. Then add the check and run again.
3. Write one sentence on why the reference version does not need the check.

Done when you have recorded what happened with `nullptr` in the pointer version.

Exercise 4 walks an array with pointer arithmetic.

1. Create an array of eight integers. Walk it with a pointer from the first element to one past the last, and print each value.
2. Sum the values with the same loop and compare the result with the sum you compute by hand.

Done when the loop prints all eight values and stops exactly at the end.

Exercise 5 finds a dangling pointer on purpose, in WSL.

1. In WSL with GCC 14, write a function that returns a pointer to a local variable and calls it from `main`, then dereferences the result. Compile it without `-Werror` first, then with `-fsanitize=address -g`.
2. Run the program and read the AddressSanitizer report. Note the line where the stack object was created and the line where it was used.

Done when you can point to the two lines in the sanitizer report.

---

## Project: a pointer toolkit with a non-owning view

The project is a small library. It has three functions that work on raw pointers, a swap function that works on references, and a view class that wraps a pointer and a size. The view does not own the memory, and the tests check the boundary cases that cause the most bugs.

Suggested repo name: `100-days-software-day-31-pointers-and-references`.

The repo layout:

```
100-days-software-day-31-pointers-and-references/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── safety-rules.md
├── src/
│   ├── main.cpp
│   ├── pointer_tools.cpp
│   └── pointer_tools.hpp
└── tests/
    └── pointer_tools_test.cpp
```

The header declares the functions and the view. The view stores a pointer and a size, and it owns nothing:

```cpp
// src/pointer_tools.hpp
#ifndef POINTER_TOOLS_HPP
#define POINTER_TOOLS_HPP

#include <cstddef>

// Returns a pointer to the largest element in [first, last), or nullptr if the range is empty.
// When two elements tie, the first one wins.
const int* find_max_ptr(const int* first, const int* last);

// Sums the values in [first, last).
int sum_range(const int* first, const int* last);

// Exchanges the values that a and b refer to.
void swap_values(int& a, int& b);

// A non-owning view over an array of ints. It does not free the memory.
class IntView {
public:
    IntView(const int* data, std::size_t size);

    std::size_t size() const { return size_; }
    int at(std::size_t index) const;
    int sum() const { return sum_range(data_, data_ + size_); }

private:
    const int* data_;
    std::size_t size_;
};

#endif
```

The implementation checks the arguments that a pointer cannot check for itself, and it throws on bad input instead of dereferencing:

```cpp
// src/pointer_tools.cpp
#include "pointer_tools.hpp"

#include <cstddef>
#include <stdexcept>

const int* find_max_ptr(const int* first, const int* last) {
    if (first == last) {
        return nullptr;
    }

    const int* best = first;
    for (const int* p = first + 1; p != last; ++p) {
        if (*p > *best) {
            best = p;
        }
    }
    return best;
}

int sum_range(const int* first, const int* last) {
    int total = 0;
    for (const int* p = first; p != last; ++p) {
        total += *p;
    }
    return total;
}

void swap_values(int& a, int& b) {
    const int tmp = a;
    a = b;
    b = tmp;
}

IntView::IntView(const int* data, std::size_t size) : data_(data), size_(size) {
    if (data == nullptr && size != 0) {
        throw std::invalid_argument("null data with a nonzero size");
    }
}

int IntView::at(std::size_t index) const {
    if (index >= size_) {
        throw std::out_of_range("IntView index out of range");
    }
    return data_[index];
}
```

The `main.cpp` file shows the functions working on a small array:

```cpp
// src/main.cpp
#include <cstddef>
#include <iostream>

#include "pointer_tools.hpp"

int main() {
    const int readings[] = {12, 47, 31, 47, 9};
    const std::size_t n = sizeof(readings) / sizeof(readings[0]);

    const int* largest = find_max_ptr(readings, readings + n);
    std::cout << "largest: " << *largest << " at index " << (largest - readings) << '\n';
    std::cout << "sum: " << sum_range(readings, readings + n) << '\n';

    const IntView view(readings, n);
    std::cout << "view sum: " << view.sum() << ", view[2]: " << view.at(2) << '\n';

    int a = 1;
    int b = 2;
    swap_values(a, b);
    std::cout << "a=" << a << " b=" << b << '\n';

    const int* none = find_max_ptr(readings, readings);
    std::cout << (none == nullptr ? "empty range gives nullptr\n" : "unexpected\n");
    return 0;
}
```

The test file covers the tie rule, the empty range, the view bounds and the swap:

```cpp
// tests/pointer_tools_test.cpp
#include <cstdlib>
#include <iostream>
#include <stdexcept>

#include "pointer_tools.hpp"

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
    const int data[] = {12, 47, 31, 47, 9};
    const int* end = data + 5;

    check(find_max_ptr(data, end) == data + 1, "first of two 47s wins the tie");
    check(find_max_ptr(data, data) == nullptr, "empty range returns nullptr");
    check(sum_range(data, end) == 146, "sum of the sample");

    int x = 3;
    int y = 8;
    swap_values(x, y);
    check(x == 8 && y == 3, "swap exchanges values");

    const IntView view(data, 5);
    check(view.size() == 5, "view reports its size");
    check(view.sum() == 146, "view sum matches the range sum");
    check(view.at(4) == 9, "last element is reachable");

    bool threw = false;
    try {
        (void)view.at(5);
    } catch (const std::out_of_range&) {
        threw = true;
    }
    check(threw, "index past the end throws");

    bool null_threw = false;
    try {
        const IntView broken(nullptr, 3);
        (void)broken;
    } catch (const std::invalid_argument&) {
        null_threw = true;
    }
    check(null_threw, "null data with size throws");

    if (failures == 0) {
        std::cout << "all pointer tool tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the library, the demo and the test, and it adds an option that turns on AddressSanitizer for the WSL build:

```cmake
cmake_minimum_required(VERSION 3.25)
project(pointer_tools LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(ENABLE_ASAN "Build with AddressSanitizer" OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

if(ENABLE_ASAN AND NOT MSVC)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=address)
endif()

add_library(pointer_tools STATIC src/pointer_tools.cpp)
target_include_directories(pointer_tools PUBLIC src)

add_executable(pointer_demo src/main.cpp)
target_link_libraries(pointer_demo PRIVATE pointer_tools)

add_executable(pointer_tools_test tests/pointer_tools_test.cpp)
target_link_libraries(pointer_tools_test PRIVATE pointer_tools)

enable_testing()
add_test(NAME pointer_tools_test COMMAND pointer_tools_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add pointer toolkit with IntView".
2. Build on Windows with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Run `ctest --test-dir build --output-on-failure` and confirm the test passes.
3. Run `./build/pointer_demo` and check each line by hand: the largest value is 47 at index 1, the sum is 146, the view prints the same sum and `view[2]` is 31, and the swap prints `a=2 b=1` because `main` starts with `a = 1` and `b = 2`.
4. In WSL, build with `cmake -S . -B build-asan -G Ninja -DENABLE_ASAN=ON -DCMAKE_BUILD_TYPE=Debug` and run `ctest --test-dir build-asan --output-on-failure`. Confirm there are no sanitizer reports.
5. Create a branch `feature/bounded-max`. Write a test first for a function `int max_or(const int* first, const int* last, int fallback)` that returns the fallback on an empty range. Then implement it and make the test pass.
6. Write `docs/safety-rules.md` with three short sections: the ownership of each pointer in the library, the rule for returning a pointer from `find_max_ptr`, and one case where you would switch from a raw pointer to `std::vector` or `std::optional`.
7. Merge with `git merge --no-ff feature/bounded-max -m "Add bounded max helper"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- The Windows test passes, and the WSL test with AddressSanitizer passes with no reports.
- The demo output matches the values you calculated by hand.
- `docs/safety-rules.md` names the owner of the memory behind each pointer.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the library and test files, `docs/safety-rules.md`, `README.md`, and the history on GitHub.

This project is short and sharp. A reviewer should see that every function either checks its pointers or refuses to take them, and that the test proves the boundaries.

---

## Check your understanding

1. You write `int* p, q;` and later assign `q = &score;` and get a compile error. Explain what the declaration created.
2. A function returns `int*` to a local array. Describe what can go wrong at run time, and why it may look fine in a quick test.
3. Why does a reference parameter remove the need for a null check that a pointer parameter would need? Name one case where you still need a pointer.
4. An array loop uses `for (int* p = first; p <= last; ++p)` and reads garbage at the end. Explain what `last` points at and fix the loop.
5. A class stores an `IntView` that points at a `std::vector`. Later the vector is resized, and the view reads wrong values. Explain the cause and name the two fixes.
6. Why is `nullptr` clearer than `0` or `NULL` when you call an overloaded function that accepts both an `int` and a pointer?

### Answers

1. `int* p, q;` declares `p` as a pointer and `q` as a plain `int`, because the star belongs to `p` only. Assigning an address to `q` is a type error. Write `int* p; int* q;` or `int *p, *q;` to get two pointers.
2. The local array is destroyed when the function returns, so the pointer refers to memory that is no longer owned by any live object. Reading it may return the old values for a while, which is why a quick test can pass. Return the array by value, or take an output parameter from the caller.
3. A reference must refer to an object, so it cannot be null and the caller cannot pass nothing. You still need a pointer when the argument is optional, or when the function must accept an array, or when the value may be reassigned to point at another object.
4. `last` points one past the last element, so the condition `p <= last` includes an element that does not exist. Use `p != last` or `p < last`, which stops before the end.
5. The vector reallocates when resized, and the view still points at the old memory. Fix it by making the view store an index into the vector, or by recomputing the view after every change to the vector.
6. A call with `nullptr` matches only the pointer overload. A call with `0` could match the `int` overload, and `NULL` may expand to `0` on some platforms, which makes the choice unclear. `nullptr` removes the ambiguity.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Pointer (computer programming)](https://en.wikipedia.org/wiki/Pointer_(computer_programming)) for the general concept
- [Wikipedia: Dangling pointer](https://en.wikipedia.org/wiki/Dangling_pointer) for the lifetime bugs in this day
- [Wikipedia: Null pointer](https://en.wikipedia.org/wiki/Null_pointer) for the null concept
- cppreference.com, the pages for "Pointer declaration", "Reference declaration", "nullptr" and "Pointer arithmetic" in the expressions section
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the section on pointers and references, including the rule about owning and non-owning pointers
- AddressSanitizer documentation at the GCC manual, section "Instrumentation Options," under `-fsanitize=address`

*Tutorials*

- learncpp.com, the chapters on pointers, references, pointer arithmetic and null pointers, written by Alex Allain
- GeeksforGeeks, search the site for "C++ pointers" and "C++ references" for the matching articles
- W3Schools, the C++ "Pointers" and "References" pages, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on pointers and references
- CppCon, the YouTube channel, for talks on pointer safety. Search the channel for "pointers" and "lifetime" to find them.

*Practice*

- Compiler Explorer at godbolt.org. Paste the `find_max_ptr` function and compare the generated code with the same function written using `std::max_element`. The comparison shows that both compile to a similar loop.

---

## Tomorrow

Day 32 covers memory and lifetime. You will see where a variable lives on the stack and where `new` puts an object on the heap. You will learn how an object's lifetime starts and ends, why `new` and `delete` must match, and how RAII ties a resource's lifetime to a scope, so that the dangling pointer rules from today are enforced by the compiler instead of by memory discipline.

**Next:** [Day 32: Memory and Lifetime](Software-Day-32-Memory-and-Lifetime.md)

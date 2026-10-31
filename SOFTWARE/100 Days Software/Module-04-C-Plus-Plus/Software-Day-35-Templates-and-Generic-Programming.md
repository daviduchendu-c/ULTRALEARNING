# Day 35: Templates and Generic Programming

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You have written classes and used `std::vector`, `std::variant` and iterators. This day explains how one piece of code can work on many types, with the checking done by the compiler before the program runs. It is for anyone who has read a template error that stretched over forty lines and wondered what it was trying to say.

**By the end of this day, you will have:**

- Written a function template and a class template, and explained how the compiler creates a specific version for each type used
- Written a generic algorithm that works on any input range, including containers you did not write
- Used C++20 concepts to state what a type must support, and read the error a concept produces when a type fails
- Used `static_assert` to check template behavior at compile time
- A generic toolkit library, header-only, with a bounded stack class template, a range sum and a concept-constrained formatter, plus a test executable run by CTest
- A GitHub repo with the build, the test output and a note on one template error you caused and read

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Days 27 to 34: the build, functions, references, classes, inheritance and `std::variant`
- Day 30: `std::vector`, iterators and `<algorithm>`

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Function templates](#function-templates)
- [Class templates](#class-templates)
- [Generic algorithms](#generic-algorithms)
- [Compile-time concepts](#compile-time-concepts)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a header-only generic toolkit with tests](#project-a-header-only-generic-toolkit-with-tests)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Suppose you need a function that returns the larger of two numbers. You write one for `int`, then one for `double`, then one for `long`. The bodies are the same, and the copies drift apart over time. Templates solve that problem. You write the logic once, with a placeholder for the type, and the compiler produces a separate version for each type you use. The placeholder is checked when the code is compiled, which means a mistake shows up before the program runs.

Templates are the foundation of the standard library. `std::vector`, `std::map`, `std::sort` and `std::find` are all templates, and they work on any type that meets their requirements. Learning to write them lets you read the library source and the error messages that come from it. This day covers function templates, class templates and generic algorithms, then concepts, which state the requirements in a form the compiler can read and explain.

The template rules are in the ISO C++ standard. cppreference.com has pages for templates, concepts and the constraint syntax, and each standard library page shows the requirements its template places on its types.

---

## Glossary

- **Template:** a blueprint for a function or class in which one or more types are left open, to be filled in later.
- **Type parameter:** a placeholder for a type, declared with `typename T` or `class T`.
- **Template argument:** the actual type given to a type parameter, either written out or deduced by the compiler.
- **Instantiation:** the compiler's creation of a specific function or class from a template, for one set of template arguments.
- **Function template:** a template that defines a function with type parameters.
- **Class template:** a template that defines a class with type parameters, such as `std::vector<T>`.
- **Generic algorithm:** a function template that works on any sequence described by iterators, not on one container type.
- **Concept:** a named set of requirements on a type, declared with `concept` and checked by the compiler, introduced in C++20.
- **Constraint:** a requirement that a template argument must satisfy, written with a concept or a `requires` clause.
- **Header-only library:** a library whose templates live entirely in header files, because the compiler needs the full definition to create each instantiation.
- **Two-phase lookup:** the rule that the compiler checks some template code at definition time and some at instantiation time. It explains many confusing error messages.

---

## Core concepts

The concepts build on each other. A function template shows the basic idea, a class template applies it to a type with state, a generic algorithm applies it to a sequence, and concepts make the requirements explicit so that errors are short and readable.

### Function templates

A function template is a recipe with a blank for the ingredient type. Think of a cookie cutter. The shape is fixed, and you press it into dough of any thickness. Each time you use the cutter with a new type, the compiler makes a new cookie. The cutter itself never runs, and only the cookies remain.

In first principles, the compiler sees `larger<int>` or `larger<double>` at a call site. It deduces the type from the arguments when it can, then instantiates a copy of the function body for that type. Each instantiation is checked as normal code, which is why the errors appear at the call site rather than in the template body.

```cpp
#include <iostream>
#include <string>

template <typename T>
T larger(T a, T b) {
    return a > b ? a : b;
}

int main() {
    std::cout << larger(3, 9) << '\n';
    std::cout << larger(2.5, 1.5) << '\n';
    std::cout << larger(std::string("pear"), std::string("apple")) << '\n';
    return 0;
}
```

The output is `9`, `2.5` and `pear`. The compiler creates three instantiations, one for `int`, one for `double` and one for `std::string`. The `>` operator is what the template requires, and each type must provide it. A type without `>` causes a compile error at the call site.

The common mistake is to write a template whose body uses an operation that only some types support, and then pass a type that lacks it. The error message points inside the template body, and it can be long. The fix is a concept, covered later in this day, which states the requirement up front. You can spot the cause by looking for the operator named in the error. In the market, template libraries such as Eigen for linear algebra and Boost for utilities depend on the same pattern, which is why their error messages are notorious.

### Class templates

A class template is a blueprint for a whole class with type parameters. The best-known example is `std::vector<T>`, which holds elements of any type. Think of a set of labeled storage boxes that come in any size, where the label on each box names the type of item it holds.

```cpp
#include <iostream>
#include <string>

template <typename First, typename Second>
struct Pair {
    First first;
    Second second;
};

int main() {
    Pair<std::string, int> entry{"score", 90};
    Pair<int, double> ratio{3, 0.5};

    std::cout << entry.first << '=' << entry.second << '\n';
    std::cout << ratio.first * ratio.second << '\n';
    return 0;
}
```

The output is `score=90` and `1.5`. Each `Pair` is a different type, so `Pair<std::string, int>` and `Pair<int, double>` cannot be mixed by accident. The braces initialize the members in order, which works because `Pair` is an aggregate, a simple struct with public members.

Class templates with member functions need the definition in the header, because the compiler must see the whole template to create any member. The project places all of its templates in a header for this reason. A class template can also have a default type argument, such as `template <typename T = int>`, which lets callers omit the type when the default is right.

The common mistake is to put the definition of a template member in a `.cpp` file. Without the definition visible at the point of use, the linker cannot find the instantiated member, and the error looks like an undefined reference. The fix is to move the template definition into the header. In the market, container libraries and numerical libraries are almost entirely header-only for this reason.

### Generic algorithms

A generic algorithm is a function template that works on any sequence described by iterators, so one implementation serves a vector, a list and an array. Think of a bucket that can hold any kind of fruit. The bucket only needs to hand its pieces out one by one, whatever the fruit.

This is the design behind `<algorithm>`. The algorithm takes a pair of iterators, and it uses only the operations that the iterator category provides. The template does not care about the container type. Here is one algorithm that counts a value in any range:

```cpp
#include <iostream>
#include <list>
#include <vector>

template <typename Iterator, typename Value>
int count_equal(Iterator first, Iterator last, const Value& target) {
    int count = 0;
    for (; first != last; ++first) {
        if (*first == target) {
            ++count;
        }
    }
    return count;
}

int main() {
    std::vector<int> numbers{1, 2, 2, 3, 2};
    std::list<int> linked{2, 5, 2};

    std::cout << count_equal(numbers.begin(), numbers.end(), 2) << '\n';
    std::cout << count_equal(linked.begin(), linked.end(), 2) << '\n';
    return 0;
}
```

The output is `3` for the vector and `2` for the list. The same template works on two containers with different memory layouts, because it uses only `++`, `*` and comparison on iterators. A requirement such as random access, which `std::sort` needs, is a stronger rule on the iterator, and it is what made `std::map` unusable with `std::sort` on Day 30.

The common mistake is to write a generic algorithm that quietly assumes a stronger iterator category than the caller provides, such as using `iterator + n`. The code compiles for vectors and fails for lists, and the error appears at the call site. The fix is to state the requirement in a concept, or to use only operations that every iterator provides. In the market, the Standard Template Library is the reference example, and teams that write their own algorithms copy its iterator categories closely.

### Compile-time concepts

A concept is a named requirement that a template argument must satisfy. Before C++20, templates checked their requirements only by trying to compile the body, which made errors long and confusing. A concept moves that requirement into the function's declaration, so the compiler can reject the call with a short message that names the failed requirement.

Think of a job posting with a list of required skills. A candidate who does not meet the list is turned away at the door, and the reason is written on the form. Concepts are that posting for types. Each concept is a predicate that the compiler evaluates at compile time, and a `requires` expression describes the operations a type must support.

```cpp
#include <concepts>
#include <iostream>

template <std::floating_point T>
T average_of(T a, T b) {
    return (a + b) / 2;
}

int main() {
    std::cout << average_of(2.0, 3.0) << '\n';
    std::cout << average_of(1.0f, 2.0f) << '\n';
    // average_of(2, 3);  // error: constraints not satisfied, int is not floating point
    return 0;
}
```

The output is `2.5` and `1.5`. The commented line fails with a message stating that the constraint `std::floating_point` is not satisfied, which is much clearer than an error inside the template body. The standard library provides many concepts, such as `std::integral`, `std::totally_ordered` and `std::ranges::input_range`, and you can write your own with `requires`.

A concept can also describe operations rather than a single type trait. The project uses one that requires a type to be printable with `<<`. A concept is a compile-time check, so it adds nothing to the program's run time.

The common mistake is to write a concept that is too loose, which accepts types that then fail deep in the body, or too tight, which rejects types that would have worked. Test each concept with one type that should pass and one that should fail, which the project does with `static_assert`. In the market, concepts are how modern C++ libraries such as the ranges library document their requirements, and code reviewers ask for concepts on any public template.

---

## Hands-on exercises

Work in a new folder called `templates-generic`. Run `git init -b main` in it.

Exercise 1 writes a function template.

1. Write `template <typename T> T smaller(T a, T b)` that returns the smaller value.
2. Call it with two `int` values, two `double` values and two `std::string` values. Print each result.
3. Call it with an `int` and a `double` in one call. Read the compiler error and note the first line.

Done when the three working calls print the expected values and the error text for the mixed call is in your notes.

Exercise 2 writes a class template.

1. Write a `Box<T>` class template with one private member of type `T`, a constructor that sets it, and a getter that returns it by const reference.
2. Create a `Box<int>`, a `Box<std::string>` and a `Box<std::vector<int>>`, and print one value from each.

Done when all three boxes print their values and the code uses no template argument that you did not state explicitly.

Exercise 3 writes a generic algorithm.

1. Write `template <typename Iterator, typename Predicate> int count_if_true(Iterator first, Iterator last, Predicate pred)`.
2. Call it with a `std::vector<int>`, a `std::list<int>` and a plain array, using a lambda that counts even numbers.

Done when all three calls print the same count for the same data.

Exercise 4 adds a concept.

1. Write a concept `Summable` that requires `a + b` to produce a value of the same type.
2. Constrain a function `add_all` with it. Call it with `int` and `double`, then try a type that does not support `+`, such as a plain struct. Read the error.

Done when the failed call names the concept in its error message.

Exercise 5 checks templates at compile time.

1. Write three `static_assert` lines that check your templates from exercises 1 and 4 with constant values.
2. Add one `static_assert` that uses a concept, such as `static_assert(Summable<int>)`.

Done when the file compiles with no warnings, which proves the checks hold.

---

## Project: a header-only generic toolkit with tests

The project is a small header-only library with three templates and one concept. `BoundedStack<T>` holds up to a fixed number of values of any type. `sum_of` adds the values in any input range. `to_text` converts any type that can be printed. The test uses `static_assert` for compile-time checks and runtime checks for behavior.

Suggested repo name: `100-days-software-day-35-templates-and-generic-programming`.

The repo layout:

```
100-days-software-day-35-templates-and-generic-programming/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── template-errors.md
├── src/
│   ├── generic.hpp
│   └── main.cpp
└── tests/
    └── generic_test.cpp
```

The header holds all the templates. There is no `.cpp` file for them, because the compiler needs the full definitions:

```cpp
// src/generic.hpp
#ifndef GENERIC_HPP
#define GENERIC_HPP

#include <concepts>
#include <cstddef>
#include <ostream>
#include <ranges>
#include <sstream>
#include <stdexcept>
#include <string>
#include <utility>
#include <vector>

// Greatest common divisor by Euclid's algorithm. Inputs should be non-negative.
template <std::integral T>
constexpr T gcd_of(T a, T b) {
    while (b != 0) {
        const T remainder = a % b;
        a = b;
        b = remainder;
    }
    return a;
}

// Sums any input range, using the value type of the range.
template <std::ranges::input_range R>
std::ranges::range_value_t<R> sum_of(const R& range) {
    std::ranges::range_value_t<R> total{};
    for (const auto& value : range) {
        total += value;
    }
    return total;
}

// A stack with a fixed capacity. It throws instead of growing.
template <typename T>
class BoundedStack {
public:
    explicit BoundedStack(std::size_t capacity) : capacity_(capacity) {}

    void push(T value) {
        if (items_.size() >= capacity_) {
            throw std::overflow_error("stack is full");
        }
        items_.push_back(std::move(value));
    }

    T pop() {
        if (items_.empty()) {
            throw std::underflow_error("stack is empty");
        }
        T value = std::move(items_.back());
        items_.pop_back();
        return value;
    }

    const T& top() const {
        if (items_.empty()) {
            throw std::underflow_error("stack is empty");
        }
        return items_.back();
    }

    std::size_t size() const { return items_.size(); }
    bool empty() const { return items_.empty(); }

private:
    std::size_t capacity_;
    std::vector<T> items_;
};

// A concept for types that can be written to an output stream.
template <typename T>
concept Printable = requires(std::ostream& out, const T& value) {
    out << value;
};

template <Printable T>
std::string to_text(const T& value) {
    std::ostringstream out;
    out << value;
    return out.str();
}

// Clamps a value to [low, high]. Works for any totally ordered type.
template <std::totally_ordered T>
constexpr T clamp_to(T value, T low, T high) {
    if (value < low) {
        return low;
    }
    if (value > high) {
        return high;
    }
    return value;
}

#endif
```

The demo calls each template with a different type, so you can see the instantiations:

```cpp
// src/main.cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

#include "generic.hpp"

int main() {
    std::cout << "gcd(84, 36) = " << gcd_of(84, 36) << '\n';
    std::cout << "gcd(17L, 5L) = " << gcd_of(17L, 5L) << '\n';

    const std::vector<int> scores{90, 85, 77};
    std::cout << "sum of scores = " << sum_of(scores) << '\n';

    const std::vector<double> prices{1.25, 2.50};
    std::cout << "sum of prices = " << sum_of(prices) << '\n';

    BoundedStack<std::string> stack(2);
    stack.push("first");
    stack.push("second");
    std::cout << "top is " << stack.top() << '\n';

    std::cout << "text: " << to_text(42) << ' ' << to_text(std::string("hi")) << '\n';
    std::cout << "clamp 15 to [0, 10] = " << clamp_to(15, 0, 10) << '\n';

    try {
        stack.push("third");
    } catch (const std::overflow_error& error) {
        std::cout << "overflow: " << error.what() << '\n';
    }
    return 0;
}
```

The test uses `static_assert` for the compile-time checks, and runtime checks for the stack and the range sum:

```cpp
// tests/generic_test.cpp
#include <cstdlib>
#include <iostream>
#include <list>
#include <stdexcept>
#include <string>
#include <vector>

#include "generic.hpp"

// Compile-time checks. If one of these fails, the build fails.
static_assert(gcd_of(12, 18) == 6);
static_assert(gcd_of(17L, 5L) == 1L);
static_assert(clamp_to(15, 0, 10) == 10);
static_assert(Printable<int>);
static_assert(!Printable<std::vector<int>>);

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
    check(sum_of(std::vector<int>{}) == 0, "sum of an empty vector is zero");
    check(sum_of(std::list<int>{1, 2, 3}) == 6, "sum works on a list");
    check(to_text(3.5) == "3.5", "to_text formats a double");

    BoundedStack<int> stack(2);
    check(stack.empty(), "new stack is empty");
    stack.push(1);
    stack.push(2);
    check(stack.top() == 2, "top is the last value pushed");
    check(stack.pop() == 2 && stack.pop() == 1, "pop returns values in last-in order");

    bool overflow = false;
    try {
        stack.push(1);
        stack.push(2);
        stack.push(3);
    } catch (const std::overflow_error&) {
        overflow = true;
    }
    check(overflow, "pushing past capacity throws");

    bool underflow = false;
    try {
        BoundedStack<std::string> empty_stack(1);
        (void)empty_stack.pop();
    } catch (const std::underflow_error&) {
        underflow = true;
    }
    check(underflow, "popping an empty stack throws");

    if (failures == 0) {
        std::cout << "all generic tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` declares the library as an interface target, which has no compiled code of its own, and links the demo and the test against it:

```cmake
cmake_minimum_required(VERSION 3.25)
project(generic_toolkit LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_library(generic INTERFACE)
target_include_directories(generic INTERFACE src)

add_executable(generic_demo src/main.cpp)
target_link_libraries(generic_demo PRIVATE generic)

add_executable(generic_test tests/generic_test.cpp)
target_link_libraries(generic_test PRIVATE generic)

enable_testing()
add_test(NAME generic_test COMMAND generic_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add header-only generic toolkit".
2. Build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. The `static_assert` lines run at compile time, so a successful build already proves them. Run `ctest --test-dir build --output-on-failure`.
3. Run `./build/generic_demo` and check each line: `gcd(84, 36) = 12`, `gcd(17L, 5L) = 1`, `sum of scores = 252`, `sum of prices = 3.75`, `top is second`, `text: 42 hi`, `clamp 15 to [0, 10] = 10`, and the overflow message.
4. Create a branch `feature/sum-with-start`. Write a test first for a new function template `sum_from(R range, T start)` that adds `start` to the sum of the range. Keep the template in the header and use the same concept as `sum_of`. Then make the test pass.
5. Add a `static_assert` to the test that checks a type that should not work, such as `gcd_of(1.5, 2.0)`, fails the constraint. You can express this with `requires` in a separate test, or by checking `std::integral<double>` is false. Record the error text from a deliberate failing call in `docs/template-errors.md`.
6. Write `docs/template-errors.md` with three short sections: one error you read that named a concept, one error that pointed inside a template body, and what you changed to make the second kind easier to read.
7. Merge with `git merge --no-ff feature/sum-with-start -m "Add sum from a start value"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- The build succeeds, which proves the `static_assert` checks, and `ctest` passes.
- The demo prints the eight lines from task 3.
- `docs/template-errors.md` quotes at least one real error message from your builds.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the header, the demo and test files, `docs/template-errors.md`, `README.md`, and the history on GitHub.

This project is short but dense. A reviewer should see that the templates are in the header, that the requirements are stated as concepts, and that the compile-time checks are part of the build.

---

## Check your understanding

1. You write a template that calls `.size()` on its argument, and a caller passes an `int`. Explain where the error appears, and why the error message points inside the template body.
2. A class template member function is defined in `widget.cpp`, and the test links but fails with an undefined reference. Explain why, and state the fix.
3. A generic algorithm uses `iterator + 3`. It works with a `std::vector` and fails with a `std::list`. Explain why, and name the iterator category the algorithm requires.
4. Why does a concept produce a clearer error than an unconstrained template?
5. A concept `Printable` accepts `std::string` but rejects `std::vector<int>`. Explain how the compiler decides, and what a `requires` expression checks.
6. Why does `sum_of` declare its return type as `std::ranges::range_value_t<R>` instead of `int`?

### Answers

1. The error appears at the call site, because the compiler instantiates the template for `int` there and finds that `int` has no `size()` member. The message points inside the body because the failing expression is in the body, and the compiler shows that line too.
2. Template members must be defined where the compiler can see them to instantiate them. A definition in a separate `.cpp` file is not available to the test's compile step, so no instantiation exists and the linker cannot find the symbol. Move the definition into the header.
3. `iterator + 3` needs random-access iterators, which a `std::list` does not provide. The algorithm requires random access, so the iterator category is random access. The fix is to use `std::next(first, 3)`, which works with every category, or to declare the stronger requirement with a concept.
4. A concept checks the requirement in the function declaration, before the body is instantiated. The compiler reports which constraint failed, in one line, instead of showing the body's failed expression.
5. The compiler evaluates the `requires` expression for the type. It asks whether `out << value` compiles for `std::string`, which it does, and for `std::vector<int>`, which has no `<<` operator in the standard library, so the concept is false.
6. The return type should match the value type of the range, so that a range of `double` returns a `double` and a range of `int` returns an `int`. A fixed `int` return would truncate sums of doubles.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Generic programming](https://en.wikipedia.org/wiki/Generic_programming) for the idea across languages
- [Wikipedia: Template (C++)](https://en.wikipedia.org/wiki/Template_(C%2B%2B)) for the C++ mechanism
- [Wikipedia: C++20](https://en.wikipedia.org/wiki/C%2B%2B20) for the release that added concepts, which this day uses
- cppreference.com, the pages for "Templates", "Constraints and concepts", "Class template argument deduction" and the `<concepts>` header
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on templates and concepts

*Tutorials*

- learncpp.com, the chapters on function templates, class templates and template argument deduction, written by Alex Allain
- GeeksforGeeks, search the site for "C++ templates" and "C++20 concepts" for the matching articles
- W3Schools, the C++ "Templates" page, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on templates
- CppCon, the YouTube channel, for talks on concepts and generic code. Search the channel for "concepts" to find them.

*Practice*

- Compiler Explorer at godbolt.org. Paste the `larger` function from this day, select GCC 14 or newer with `-std=c++20`, and compare the generated code for `int` and `double`. The two instantiations appear as separate functions.

---

## Tomorrow

Day 36 is the STL deep dive. You have used `std::vector`, `std::sort` and iterators, and the containers in the standard library are all class templates like the `BoundedStack` from today. Day 36 covers maps, sets, queues and stacks, the iterator invalidation rules for each one, and lambdas, which you used in this day's exercises and which the algorithms depend on. The generic-programming habits from today help you read the requirements listed on each container's reference page.

**Next:** [Day 36: STL Deep Dive](Software-Day-36-STL-Deep-Dive.md)

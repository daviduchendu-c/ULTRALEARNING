# Day 34: Inheritance and Polymorphism

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You can write classes with constructors, destructors and private members from Day 33. This day adds the next layer: one class can be a kind of another, and a single call can do different work depending on the object it reaches. It is for anyone who has seen a class hierarchy in a framework and wondered how the right function gets called.

**By the end of this day, you will have:**

- Declared a base class with `virtual` functions, a pure virtual interface and a virtual destructor, and explained why each one is needed
- Marked overrides with `override` and `final`, and explained what the compiler checks for you
- Compared an inheritance design with a composition design, and chosen between them for a given problem
- A shape-report library with an abstract `Shape` interface, a `std::variant` version of the same report, and a test that proves both give the same answer
- A GitHub repo with the build, the test and the report output, plus a short note on the design choice

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- A free GitHub account

**Prerequisites:**

- Days 27 to 33: the build, pointers and references, lifetime, and classes with access control
- Day 30: `std::vector` and iterators

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Virtual functions](#virtual-functions)
- [Overriding](#overriding)
- [Abstract interfaces](#abstract-interfaces)
- [Composition tradeoffs](#composition-tradeoffs)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a shape report with inheritance and composition](#project-a-shape-report-with-inheritance-and-composition)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Software often has many kinds of one thing. A payment can be a card, a bank transfer or a voucher. A sensor can be a camera, a lidar or a temperature probe. A shape can be a circle or a rectangle. Each kind has its own rules, and the code that uses them should not need to know which kind it has. Inheritance and polymorphism are the C++ tools for that situation. A base class states what every kind can do, and derived classes say how they do it.

The tools are powerful and easy to misuse. A deep hierarchy couples code that should be independent. A missing `virtual` on a destructor causes silent memory leaks. A derived class that does not match the base contract breaks the code that trusts the base. This day shows the right tools, the traps, and a second way to solve the same problem with composition, so you can compare them on real code instead of in theory.

The rules for classes with virtual functions are defined in the ISO C++ standard. cppreference.com has pages on derived classes, virtual functions and abstract classes, and the C++ Core Guidelines discuss when to prefer each design.

---

## Glossary

- **Base class:** a class that other classes derive from, which defines the shared interface.
- **Derived class:** a class that inherits from a base class, written `class Circle : public Shape`.
- **Inheritance:** a relationship in which a derived class has the members of its base class and can add or change behavior.
- **Polymorphism:** the ability to call one function name on objects of different types and have the right code run for each type.
- **Virtual function:** a member function that a derived class can replace, and that is chosen at run time through the object's actual type.
- **Override:** a derived class function that replaces a virtual function from its base.
- **Pure virtual function:** a virtual function declared with `= 0`, which the derived class must implement. A class with one is abstract.
- **Abstract class:** a class with at least one pure virtual function. It cannot be instantiated.
- **Dynamic dispatch:** the run-time lookup that picks the override for a virtual call.
- **Slicing:** copying a derived object into a base object, which keeps only the base part and loses the derived behavior.
- **Composition:** building a class from other objects it contains, instead of inheriting from them.
- **Variant:** `std::variant`, a type that holds exactly one of several listed types, chosen at run time.

---

## Core concepts

The concepts form a chain. Virtual functions make polymorphism possible, `override` and `final` keep the chain honest, abstract interfaces describe what each kind must do, and composition is the alternative when inheritance costs more than it returns.

### Virtual functions

A virtual function is a promise that the actual type of the object, not the type of the pointer or reference, decides which function runs. Think of a remote control with one button labeled "power." The button is the same for every device, but each device reacts in its own way. The remote sends the press, and the device at the other end decides what it means.

In first principles, a class with virtual functions has a hidden table of function addresses, usually called the vtable. Each object carries a small pointer to the table for its type. When you call a virtual function through a base pointer, the program follows that pointer to find the right function. Without the `virtual` keyword, the compiler picks the function from the pointer's static type, and the derived version is never reached.

```cpp
#include <iostream>

class Animal {
public:
    virtual ~Animal() = default;

    virtual void speak() const {
        std::cout << "...\n";
    }

    void introduce() const {
        std::cout << "I am an animal. ";
        speak();
    }
};

class Dog : public Animal {
public:
    void speak() const override {
        std::cout << "Woof\n";
    }
};

class Cat : public Animal {
public:
    void speak() const override {
        std::cout << "Meow\n";
    }
};

int main() {
    Dog dog;
    Cat cat;
    const Animal* pets[] = {&dog, &cat};

    for (const Animal* pet : pets) {
        pet->speak();
    }
    dog.introduce();
    return 0;
}
```

The output is `Woof`, `Meow`, and then `I am an animal. Woof`. The loop calls `speak` through base pointers, and each call reaches the right override. The `introduce` function is not virtual, but it calls `speak`, which is, so the dispatch still happens inside it. The virtual destructor is the subject of the next section, and it is required here so that deleting a `Dog` through an `Animal` pointer destroys it correctly.

The common mistake is to forget `virtual` on a base function. The code compiles and calls the base version every time, which is silent and hard to spot. Compilers can warn with `-Woverloaded-virtual`, and `override` catches most cases, as the next section explains. The cost of a virtual call is an extra pointer lookup, which is small, but it prevents some compiler optimizations, which matters in tight loops. In the market, plugin systems, GUI toolkits and device drivers use virtual functions to let code run against interfaces the core team never saw.

### Overriding

An override is a derived class function that replaces a virtual function from its base. The word `override` tells the compiler that you meant to replace one, and the compiler checks that a matching virtual function exists. If you misspell the name or change a parameter type, the compiler reports an error instead of silently creating a new function. That single keyword prevents a whole class of bugs.

The `final` keyword has two uses. On a function, it stops further overrides. On a class, it stops further derivation. The next example shows both:

```cpp
#include <iostream>

class Base {
public:
    virtual ~Base() = default;
    virtual void run() const {
        std::cout << "base run\n";
    }
};

class Worker : public Base {
public:
    void run() const override {
        std::cout << "worker run\n";
    }
};

class Specialist final : public Worker {
public:
    void run() const override {
        std::cout << "specialist run\n";
    }
};

int main() {
    Specialist specialist;
    const Base& ref = specialist;
    ref.run();
    return 0;
}
```

The output is `specialist run`. The reference has the static type `Base`, and the call goes to the most derived override. Because `Specialist` is marked `final`, no class can derive from it, and the hierarchy is closed. If you wrote `void run() const override` with a `const` missing in one class, the compiler would stop with an error, because the signatures no longer match.

The common mistake is to declare a function with the same name in a derived class but a different signature, believing it overrides the base. The result is an overload, not an override, and the base version still runs through base pointers. The fix is `override`. You can spot the problem when the derived function is never called through a base pointer. Teams that enforce `override` in code review avoid this kind of bug entirely. In the market, large C++ codebases such as game engines and browsers enable the `-Wsuggest-override` warning to catch the omission.

### Abstract interfaces

An abstract interface is a base class with only pure virtual functions. It describes what a kind of object must do, without saying how. Think of a job description. It lists the duties of a driver, and any candidate who can perform them is acceptable. The company does not care how the driver learned to drive.

Declaring a function `= 0` makes the class abstract, which means no object of that type can be created. Any derived class must implement every pure virtual function before it can be instantiated. Code that takes the interface by reference or pointer works with every implementation.

```cpp
#include <iostream>
#include <string>

class Logger {
public:
    virtual ~Logger() = default;
    virtual void write(const std::string& message) = 0;
};

class ConsoleLogger final : public Logger {
public:
    void write(const std::string& message) override {
        std::cout << "[console] " << message << '\n';
    }
};

void record_start(Logger& logger) {
    logger.write("started");
}

int main() {
    ConsoleLogger console;
    record_start(console);
    return 0;
}
```

The output is `[console] started`. The function `record_start` depends only on `Logger`, so a test could pass a fake logger that stores messages in memory, and the production code would not change. A declaration such as `Logger logger;` does not compile, because `Logger` is abstract.

The trap here is slicing. If you copy a derived object into a base object by value, only the base part is copied, and the virtual behavior is lost. For example, `Logger copy = console;` would not compile for this abstract class, but for a concrete base it would compile and silently drop the derived data. The rule is to pass polymorphic objects by pointer or reference, never by value. In the market, interface-based design is the standard for test doubles, plugin APIs and hardware abstraction layers.

### Composition tradeoffs

Composition builds a class by holding other objects, instead of inheriting from them. A car has an engine, and the car uses it without being an engine itself. Think of a restaurant kitchen. The chef has a stove, a knife and a fridge, and uses them.

Inheritance says "is a," and composition says "has a." Both are useful, and the right choice depends on what must stay true. Inheritance couples the derived class to every detail of the base, so a change to the base can break the derived class. Composition couples less, because the containing class only uses the public interface of its parts.

```cpp
#include <iostream>

class Engine {
public:
    void start() const {
        std::cout << "engine on\n";
    }
};

class Car {
public:
    void drive() const {
        engine_.start();
        std::cout << "car moving\n";
    }

private:
    Engine engine_;
};

int main() {
    Car car;
    car.drive();
    return 0;
}
```

The output is `engine on` and then `car moving`. The `Car` owns an `Engine` as a member, and it calls the engine through its public interface. If the engine needs a different implementation, the car can hold a different type, or a pointer to an interface, without changing its own code.

The tradeoffs are concrete. Inheritance gives you dynamic dispatch and a shared interface with virtual calls. Composition gives you looser coupling, and with `std::variant` a closed set of types with no virtual calls at all. The variant approach is the one the project compares, because a closed set of shapes often fits it better than an open hierarchy. The common mistake is to inherit from a class just to reuse a few functions. That creates a "is a" relationship that does not exist, and a later change breaks the wrong code. The fix is to ask whether the new class could be used in every place the base class is used. If not, compose. In the market, most mature C++ codebases favor composition for reuse and reserve inheritance for true interfaces and frameworks.

---

## Hands-on exercises

Work in a new folder called `inheritance-polymorphism`. Run `git init -b main` in it.

Exercise 1 calls virtual functions through a base pointer.

1. Write a base class `Vehicle` with a virtual function `wheels() const` that returns 4. Write a derived class `Bicycle` that overrides it to return 2.
2. Put one of each in an array of `const Vehicle*` and print the wheel count for each.
3. Remove the `virtual` keyword from the base function, rebuild, and record the printed numbers.

Done when you can explain why the numbers changed when `virtual` was removed.

Exercise 2 uses `override` to catch a mistake.

1. In a derived class, write `int wheels() override` with the `const` left out, so the signature does not match the base's `wheels() const`.
2. Read the compiler error and note the exact text.
3. Fix the signature and compile again.

Done when the error text is in your notes and the fixed version builds with no warnings.

Exercise 3 demonstrates the virtual destructor.

1. Write a base class with a non-virtual destructor and a derived class that allocates an `int` array in its constructor and frees it in its destructor.
2. Delete a derived object through a base pointer. Run the program under the AddressSanitizer build in WSL and read the leak report.
3. Add `virtual` to the base destructor and run again.

Done when you have both reports and can say which destructor ran in each case.

Exercise 4 builds an abstract interface and a fake.

1. Write a `Storage` interface with pure virtual `save(const std::string&)` and `load() const`.
2. Write a `MemoryStorage` class that keeps the value in a member string, and a function that saves and loads through the interface.
3. Try to create a `Storage` object directly and read the compiler error.

Done when the function works with `MemoryStorage` and you have the error text in your notes.

Exercise 5 compares inheritance and composition.

1. Write two designs for a `Printer` that uses a `Logger`: one that inherits from `Logger`, and one that holds a `Logger` member.
2. Write one sentence for each design on what changes when the logger is replaced.

Done when your two sentences name a concrete change for each design.

---

## Project: a shape report with inheritance and composition

The project builds one report two ways. The inheritance version uses an abstract `Shape` interface with three derived classes. The composition version uses a `std::variant` over three plain structs. Both produce the same total area, and the test checks that they agree. The demo prints the report using the inheritance version.

Suggested repo name: `100-days-software-day-34-inheritance-and-polymorphism`.

The repo layout:

```
100-days-software-day-34-inheritance-and-polymorphism/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── design-choice.md
├── src/
│   ├── main.cpp
│   ├── report.cpp
│   ├── report.hpp
│   ├── shape.cpp
│   ├── shape.hpp
│   └── variant_shapes.hpp
└── tests/
    └── shape_test.cpp
```

The header declares the abstract base and the three derived classes. The destructor is virtual, and each concrete class is marked `final`:

```cpp
// src/shape.hpp
#ifndef SHAPE_HPP
#define SHAPE_HPP

#include <string>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string name() const = 0;
};

class Circle final : public Shape {
public:
    explicit Circle(double radius);
    double area() const override;
    std::string name() const override;

private:
    double radius_;
};

class Rectangle final : public Shape {
public:
    Rectangle(double width, double height);
    double area() const override;
    std::string name() const override;

private:
    double width_;
    double height_;
};

class Triangle final : public Shape {
public:
    Triangle(double base, double height);
    double area() const override;
    std::string name() const override;

private:
    double base_;
    double height_;
};

#endif
```

The implementation validates each size, so a zero or negative dimension throws before any area is computed:

```cpp
// src/shape.cpp
#include "shape.hpp"

#include <stdexcept>

namespace {

constexpr double kPi = 3.141592653589793;

void require_positive(double value, const char* what) {
    if (!(value > 0.0)) {
        throw std::invalid_argument(std::string(what) + " must be positive");
    }
}

}  // namespace

Circle::Circle(double radius) : radius_(radius) {
    require_positive(radius, "radius");
}

double Circle::area() const {
    return kPi * radius_ * radius_;
}

std::string Circle::name() const {
    return "circle";
}

Rectangle::Rectangle(double width, double height) : width_(width), height_(height) {
    require_positive(width, "width");
    require_positive(height, "height");
}

double Rectangle::area() const {
    return width_ * height_;
}

std::string Rectangle::name() const {
    return "rectangle";
}

Triangle::Triangle(double base, double height) : base_(base), height_(height) {
    require_positive(base, "base");
    require_positive(height, "height");
}

double Triangle::area() const {
    return 0.5 * base_ * height_;
}

std::string Triangle::name() const {
    return "triangle";
}
```

The report functions take pointers to the base class, so they work with any shape, present or future. They do not own the shapes:

```cpp
// src/report.hpp
#ifndef REPORT_HPP
#define REPORT_HPP

#include <string>
#include <vector>

#include "shape.hpp"

// The shapes are not owned by these functions. The caller keeps them alive.
double total_area(const std::vector<const Shape*>& shapes);
const Shape* largest(const std::vector<const Shape*>& shapes);
std::string summary(const std::vector<const Shape*>& shapes);

#endif
```

```cpp
// src/report.cpp
#include "report.hpp"

#include <iomanip>
#include <sstream>

double total_area(const std::vector<const Shape*>& shapes) {
    double sum = 0.0;
    for (const Shape* shape : shapes) {
        sum += shape->area();
    }
    return sum;
}

const Shape* largest(const std::vector<const Shape*>& shapes) {
    const Shape* best = nullptr;
    for (const Shape* shape : shapes) {
        if (best == nullptr || shape->area() > best->area()) {
            best = shape;
        }
    }
    return best;
}

std::string summary(const std::vector<const Shape*>& shapes) {
    std::ostringstream out;
    out << std::fixed << std::setprecision(2);
    for (const Shape* shape : shapes) {
        out << shape->name() << ": " << shape->area() << '\n';
    }
    out << "total: " << total_area(shapes) << '\n';
    return out.str();
}
```

The composition version holds plain structs in a `std::variant`. No virtual calls are involved, and the `std::visit` call picks the right branch at compile time for each alternative:

```cpp
// src/variant_shapes.hpp
#ifndef VARIANT_SHAPES_HPP
#define VARIANT_SHAPES_HPP

#include <type_traits>
#include <variant>
#include <vector>

namespace value {

struct Circle {
    double radius;
};

struct Rectangle {
    double width;
    double height;
};

struct Triangle {
    double base;
    double height;
};

using AnyShape = std::variant<Circle, Rectangle, Triangle>;

inline double area_of(const AnyShape& shape) {
    return std::visit(
        [](const auto& s) -> double {
            using T = std::decay_t<decltype(s)>;
            if constexpr (std::is_same_v<T, Circle>) {
                return 3.141592653589793 * s.radius * s.radius;
            } else if constexpr (std::is_same_v<T, Rectangle>) {
                return s.width * s.height;
            } else {
                return 0.5 * s.base * s.height;
            }
        },
        shape);
}

inline double total_area(const std::vector<AnyShape>& shapes) {
    double sum = 0.0;
    for (const AnyShape& shape : shapes) {
        sum += area_of(shape);
    }
    return sum;
}

}  // namespace value

#endif
```

The demo prints the report for three shapes:

```cpp
// src/main.cpp
#include <iostream>
#include <vector>

#include "report.hpp"
#include "shape.hpp"

int main() {
    const Circle circle(2.0);
    const Rectangle rect(3.0, 4.5);
    const Triangle tri(6.0, 2.0);

    const std::vector<const Shape*> shapes{&circle, &rect, &tri};
    std::cout << summary(shapes);
    std::cout << "largest: " << largest(shapes)->name() << '\n';
    return 0;
}
```

The test checks each area, the largest shape, the validation rule, and that the variant version agrees with the inheritance version:

```cpp
// tests/shape_test.cpp
#include <cmath>
#include <cstdlib>
#include <iostream>
#include <stdexcept>
#include <vector>

#include "report.hpp"
#include "shape.hpp"
#include "variant_shapes.hpp"

namespace {

int failures = 0;

void check(bool condition, const char* label) {
    if (!condition) {
        std::cerr << "FAILED: " << label << '\n';
        ++failures;
    }
}

bool near(double a, double b) {
    return std::fabs(a - b) < 1e-9;
}

}  // namespace

int main() {
    const Rectangle rect(3.0, 4.0);
    const Triangle tri(10.0, 3.0);
    const Circle circle(1.0);

    check(near(rect.area(), 12.0), "rectangle area");
    check(near(tri.area(), 15.0), "triangle area");
    check(near(circle.area(), 3.141592653589793), "circle area");

    const std::vector<const Shape*> shapes{&rect, &tri, &circle};
    const double inheritance_total = total_area(shapes);
    check(near(inheritance_total, 12.0 + 15.0 + 3.141592653589793),
          "total area through base pointers");
    check(largest(shapes) == &tri, "the triangle with area 15 is largest");
    check(largest({}) == nullptr, "an empty list has no largest shape");

    const std::vector<value::AnyShape> values{
        value::Rectangle{3.0, 4.0},
        value::Triangle{10.0, 3.0},
        value::Circle{1.0},
    };
    check(near(value::total_area(values), inheritance_total),
          "variant version matches the inheritance version");

    bool threw = false;
    try {
        const Rectangle bad(0.0, 1.0);
    } catch (const std::invalid_argument&) {
        threw = true;
    }
    check(threw, "a zero width throws");

    if (failures == 0) {
        std::cout << "all shape tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the report library, the demo and the test:

```cmake
cmake_minimum_required(VERSION 3.25)
project(shape_report LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror;-Wsuggest-override>"
)

add_library(shape_core STATIC src/shape.cpp src/report.cpp)
target_include_directories(shape_core PUBLIC src)

add_executable(shape_demo src/main.cpp)
target_link_libraries(shape_demo PRIVATE shape_core)

add_executable(shape_test tests/shape_test.cpp)
target_link_libraries(shape_test PRIVATE shape_core)

enable_testing()
add_test(NAME shape_test COMMAND shape_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add shape report with inheritance and variant versions".
2. Build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Run `ctest --test-dir build --output-on-failure` and confirm it passes.
3. Run `./build/shape_demo`. Check the output: circle 12.57, rectangle 13.50, triangle 6.00, total 32.07, and largest rectangle.
4. Create a branch `feature/hexagon`. Write a test first for a new `Hexagon` class that derives from `Shape` and computes its area from one side with `area = 3 * sqrt(3) / 2 * side * side`. Add the class, make the test pass, and make sure the existing report code needs no change.
5. Add a `Hexagon` to the variant version as well, and check that the two totals still agree in the test.
6. Write `docs/design-choice.md` with three short sections: which version you would extend for a plugin system that other teams write, which one you would choose for a fixed set of shapes that never grows, and what a slicing bug would look like in each design.
7. Merge with `git merge --no-ff feature/hexagon -m "Add hexagon shape"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes, and the inheritance and variant totals match in the test.
- The demo prints the four lines from task 3.
- `docs/design-choice.md` explains the choice in terms of who extends the code.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/design-choice.md`, `README.md`, and the history on GitHub.

This project is light on new code and heavy on design. A reviewer should see that both versions compute the same answer, and that the choice between them is written down with a reason.

---

## Check your understanding

1. A base class has a non-virtual function `describe()` that calls a virtual function `name()`. A derived class overrides `name()`. What does `describe()` print through a base pointer, and why?
2. A derived class writes `void area() const` where the base has `virtual double area() const`. Explain what the compiler does without `override`, and what it does with `override`.
3. You delete a derived object through a base pointer, and the base class has no virtual destructor. Describe the undefined behavior, and the fix.
4. A class has a pure virtual function. Why can you not create an object of that type, and what does a derived class have to do?
5. A teammate says a `Square` should inherit from `Rectangle` because a square is a rectangle. Describe one operation that would break the substitution rule, and propose a composition or separate-class design instead.
6. Why is a `std::variant` a better fit than inheritance when the set of alternatives is closed and known at compile time?

### Answers

1. It prints the derived version of `name()`, because `describe()` calls the virtual function on `this`, and the virtual call goes to the object's actual type. Dynamic dispatch works inside non-virtual functions too.
2. Without `override`, the compiler creates a new function that does not replace the base one, so calls through a base pointer still reach the base version. With `override`, the compiler checks the signature and reports an error if it does not match.
3. Deleting through a base pointer without a virtual destructor calls only the base destructor, so the derived part is not destroyed correctly and any resource it owns leaks. The fix is `virtual ~Base() = default;`.
4. A pure virtual function has no implementation in the base, so the object would have a missing function. The derived class must override every pure virtual function before it can be instantiated.
5. A `Rectangle` allows setting width and height independently, so a `Square` that keeps them equal breaks any code that sets them separately. The fix is a separate `Square` class that shares a `Shape` interface, or a composition that stores one side length.
6. A closed set can be handled by `std::visit` with no virtual calls, and the compiler checks that every alternative is handled. Adding a new alternative shows every place that needs an update, which inheritance does not do.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Polymorphism (computer science)](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)) for the general concept
- [Wikipedia: Inheritance (object-oriented programming)](https://en.wikipedia.org/wiki/Inheritance_(object-oriented_programming)) and [Wikipedia: Composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance) for the design tradeoff
- [Wikipedia: Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) for the rule behind the square and rectangle example
- cppreference.com, the pages for "Derived classes", "Virtual functions", "Abstract class" and "std::variant"
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on class hierarchies, including the rules about virtual destructors and `override`

*Tutorials*

- learncpp.com, the chapters on inheritance, virtual functions, pure virtual functions and object slicing, written by Alex Allain
- GeeksforGeeks, search the site for "C++ inheritance" and "C++ polymorphism" for the matching articles
- W3Schools, the C++ "Inheritance" and "Polymorphism" pages, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on inheritance and virtual functions
- CppCon, the YouTube channel, for talks on composition and variants. Search the channel for "std::variant" to find them.

*Practice*

- Compiler Explorer at godbolt.org. Paste the `Animal` example and look for the vtable reference in the generated code. The comparison shows how a virtual call becomes an indirect call.

---

## Tomorrow

Day 35 covers templates and generic programming. You saw one form of generic code today with `std::variant` and `std::visit`, and templates are the compile-time version of the same idea. You will write function and class templates, use concepts from C++20 to state what a type must support, and write a generic algorithm that works on any container. The virtual calls from today give you the run-time form of polymorphism, and templates give you the compile-time form, so you can compare their costs directly.

**Next:** [Day 35: Templates and Generic Programming](Software-Day-35-Templates-and-Generic-Programming.md)

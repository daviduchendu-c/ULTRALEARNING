# Day 27: C++ Toolchain

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** beginner. You have written code in another language, such as Python or JavaScript, and you know what a function and a file are. C++ adds a build step, and this day is about that step. It is for anyone who has seen a wall of red compiler text and wondered where to start reading.

**By the end of this day, you will have:**

- A working C++20 compiler and CMake setup, verified with a version command
- A three-file program built by hand with `g++`, so you can see each stage of the build
- A deliberate compile error, a linker error and a warning, each read and fixed
- A CMake project that enables warnings and treats them as errors
- A GitHub repo with a build log and a short explanation of each stage

**What you need:**

- A C++ compiler that supports C++20: GCC 14 or newer. This guide was checked with g++ 16.1.0 from a MinGW-w64 build on Windows 11. Install it with WinLibs (winlibs.com) or MSYS2 (msys2.org, then `pacman -S mingw-w64-ucrt-x86_64-gcc`). Ubuntu 24.04 in WSL 2 also works with `sudo apt install g++-14`.
- CMake 3.25 or newer. This guide was checked with CMake 4.3.3. Install it from cmake.org or with `winget install Kitware.CMake`.
- Ninja 1.11 or newer, the build tool that CMake drives. Install it with `winget install Ninja-build.Ninja`.
- Git 2.40 or newer, from `git --version`
- A terminal. Git Bash works, and so does PowerShell.
- A PATH that puts your compiler's `bin` folder first. Git for Windows ships its own MinGW runtime, `libstdc++-6.dll`. If a program built with the compiler crashes inside `libstdc++` at run time, the wrong copy of that DLL was loaded. Move the compiler's `bin` folder ahead of Git's in `PATH`, open a new terminal, and run again.
- Visual Studio Code with the CMake Tools extension (optional but useful for later days)
- A free GitHub account

**Prerequisites:**

- Days 1 to 6 of the track, which cover the command line, Git and the commit workflow
- Basic programming in any language: variables, functions and `if` statements
- Knowing how to change directories and list files in a terminal

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Source files and header files](#source-files-and-header-files)
- [Translation units](#translation-units)
- [The compiler](#the-compiler)
- [The linker](#the-linker)
- [The build process](#the-build-process)
- [Warnings](#warnings)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a three-file program with a CMake build and warnings as errors](#project-a-three-file-program-with-a-cmake-build-and-warnings-as-errors)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Python runs your code line by line through an interpreter that is already installed on the machine. C++ works differently. Before a C++ program can run, a compiler turns each source file into machine code, and a linker joins those pieces into one executable. Each step can fail, and each failure message looks different. Most beginners lose their first afternoon not to C++ itself but to the toolchain: the wrong compiler, a header that cannot be found, or a warning that everyone ignores.

This day gives you the build process as a set of visible steps. You will compile by hand, break the build on purpose, and then move the same project into CMake, the build system that most C++ teams use. Later days assume you can read a compiler error, so the time spent here pays off through the whole module.

The C++ language is standardized by ISO, and the current versions are C++20 and C++23. This module targets C++20 because every compiler named above supports its core features well. For the official overview, read the Wikipedia article [C++](https://en.wikipedia.org/wiki/C%2B%2B), then the compiler documentation for your toolchain.

---

## Glossary

- **Compiler:** a program that turns C++ source code into machine code for one platform. Examples are GCC (`g++`), Clang (`clang++`) and MSVC (`cl.exe`).
- **Linker:** a program that joins compiled pieces into one executable and resolves the names one piece uses from another.
- **Source file:** a text file with the `.cpp` extension (or `.cc`, `.cxx`) that holds code the compiler reads directly.
- **Header file:** a text file, usually `.hpp` or `.h`, that holds declarations shared by several source files.
- **Translation unit:** one source file after the preprocessor has pasted in every header it includes. This is the unit the compiler works on.
- **Object file:** the compiled output of one translation unit, usually `.o` on Linux and MinGW, or `.obj` on MSVC.
- **Declaration:** a statement that tells the compiler a name exists and what type it has, such as `double circle_area(double radius);`.
- **Definition:** the code that gives a name its behavior, such as the body of `circle_area`.
- **Preprocessor:** the first stage of compilation. It handles lines that start with `#`, such as `#include` and `#define`.
- **Build system:** a tool that runs the compiler and linker in the right order and rebuilds only what changed. CMake is a build system generator, and Ninja is the tool that runs the build.
- **Warning:** a message from the compiler that the code is legal but probably wrong.
- **Undefined reference:** a linker error that means a name was declared and used, but no object file defines it.

---

## Core concepts

C++ builds a program in stages, and each stage has its own job and its own kind of error. The concepts below follow the path a source file takes, from text on disk to a running program, and then the tools that enforce quality along the way.

### Source files and header files

Think of a source file as a recipe card and a header file as the index card that lists the names of the recipes in a shared cookbook. Anyone who wants to use a recipe reads the index card to learn its name and inputs, without copying the full method into every binder. In C++, a header declares what exists, and a source file defines how it works.

That split is a matter of first principles. The compiler reads one source file at a time and must know the type of every name before it uses that name. A header gives it that information. If you call `circle_area` in `main.cpp`, the compiler needs to see `double circle_area(double radius);` first, so the function call can be checked against the parameter types. The definition can live in a different file, and the linker connects the call to that definition later.

Here is the split in practice. Create a folder called `hello-toolchain` with these files. The header holds only declarations:

```cpp
// include/geometry.hpp
#ifndef GEOMETRY_HPP
#define GEOMETRY_HPP

double circle_area(double radius);
double rectangle_area(double width, double height);

#endif
```

The include guard, the `#ifndef` block, stops the header from being read twice in the same translation unit. Without it, including the header from two places in one file would declare the same functions twice. Modern compilers accept `#pragma once` as an alternative, but the include guard works everywhere, so this guide uses it.

The source file holds the definitions:

```cpp
// src/geometry.cpp
#include "geometry.hpp"

namespace {
constexpr double pi = 3.141592653589793;
}

double circle_area(double radius) {
    return pi * radius * radius;
}

double rectangle_area(double width, double height) {
    return width * height;
}
```

The program that uses both files is short. It includes the header, never the source file:

```cpp
// src/main.cpp
#include <iostream>

#include "geometry.hpp"

int main() {
    std::cout << "Circle area: " << circle_area(2.0) << '\n';
    std::cout << "Rectangle area: " << rectangle_area(3.0, 4.5) << '\n';
    return 0;
}
```

The anonymous `namespace` makes `pi` visible only inside this file. That keeps names from leaking into other files, which matters once a project grows. The common mistake is to put function bodies in a header. That works for one include but produces duplicate definition errors when two source files include the same header, so the rule is to keep bodies in `.cpp` files unless the function is a template or is marked `inline`. You will see how to spot that error in the warnings section. In the market, every C++ library ships headers and compiled code this way, from the standard library headers in `<vector>` to the SDK headers used by game engines.

### Translation units

A translation unit is the compiler's view of one source file after the preprocessor has finished. The preprocessor copies the text of every `#include` into the file, expands every macro, and removes the `#if` blocks that are false. What the compiler sees is one long file, and that file is the translation unit.

This matters because the compiler handles each translation unit independently. It does not know what is inside `geometry.cpp` while it compiles `main.cpp`. It only knows what the headers say. That is why the header is the contract between files, and why changing a function body in `geometry.cpp` does not require recompiling `main.cpp`, while changing a declaration in `geometry.hpp` does.

You can see the expanded translation unit with one flag. Run `g++ -std=c++20 -E -Iinclude src/main.cpp -o build/main.ii` and open `build/main.ii`. It will contain thousands of lines from the standard library, and your `main` function will be at the end. A file that looks short in your editor can be enormous after preprocessing. Compile time grows with that size, which is one reason large projects keep headers small.

The common mistake is to include a header that you do not need, which slows every build that touches the file. The fix is to include only what a file uses and to prefer forward declarations when a full definition is not needed. You can spot the problem when a small change causes a long rebuild. In the market, build speed work at companies that ship large C++ codebases, such as game studios and browser teams, starts with exactly this kind of include audit.

### The compiler

The compiler reads one translation unit and produces an object file. It runs several passes: it parses the text into a tree of statements and expressions, checks types, applies optimizations if you asked for them, and generates machine code for the target processor. Each pass can reject the program, and a rejection is what you call a compile error.

The flag `-std=c++20` tells the compiler which language rules to apply. Without it, GCC uses its default, which can change between versions. Pinning the standard in your build is a habit that prevents one machine from accepting code that another rejects.

To compile one file without linking, use `-c`:

```text
g++ -std=c++20 -Wall -Wextra -Wpedantic -Iinclude -c src/geometry.cpp -o build/geometry.o
```

The `-c` flag stops after producing the object file. The `-o` flag names the output. The `-I` flag adds `include` to the list of folders the compiler searches for headers. A file that compiles here but not in your editor often has a different `-I` path, so check that first.

Here is a compile error to read. Replace the line `return width * height;` in `geometry.cpp` with `return width * height` and delete the semicolon. The compiler reports something like this:

```text
src/geometry.cpp:14:5: error: expected ';' before '}' token
```

The message gives the file, the line and column, and the reason. Read the first error only. Later errors often follow from the first, and fixing the first one clears most of the rest. Fix the semicolon and the error goes away. That habit, top-down and one error at a time, is most of what you need to read compiler output.

The common mistake is to ignore the file and line and look only at the message text. Look at the location first. The line the compiler points to is often where the mistake is, but the real cause can be a missing `#include` a few lines higher.

Compilers appear in the market as part of every toolchain you will meet. Visual Studio uses MSVC, Apple's developer tools ship Clang, and Linux distributions package GCC. Knowing the flag names for two of them makes it easier to read build logs written for the third.

### The linker

The linker takes all the object files and the libraries, then resolves every name that one object file uses but another defines. It does not read C++ code. It works with symbol names and addresses, which is why a linker error can appear even when every source file compiles cleanly.

Suppose `main.cpp` calls `rectangle_area`, but `geometry.cpp` does not define it. The compile step succeeds, because the header declares the function and the compiler trusts that a definition exists somewhere. The link step fails:

```text
main.cpp:(.text+0x2e): undefined reference to `rectangle_area(double, double)'
```

The message says a name was used but never defined. The fix is to define the function in one source file, then link that file's object into the program. Check that the source file is in the `add_executable` list in CMake or in your manual `g++` command. A forgotten source file is the most common cause of undefined references in a new project.

A different linker error is multiple definitions. If you put `double rectangle_area(double, double) { ... }` into `geometry.hpp` and include that header in two `.cpp` files, each object file contains its own copy of the function, and the linker reports that the symbol is defined more than once. This is the duplicate definition problem from the source files section, and the linker is where it appears.

Link the program by naming all the object files:

```text
g++ build/main.o build/geometry.o -o build/hello
```

On Windows, the output is `build\hello.exe`. Run it with `./build/hello` in Git Bash or `.\build\hello.exe` in PowerShell.

Teams that ship libraries use the same mechanism. A shared library exposes symbols, and a program that links against it gets the same undefined reference errors when a symbol is missing. In the market, the linker is the reason build engineers learn platform-specific symbol visibility rules.

### The build process

The build process is the sequence that connects source to program: preprocess each file, compile it to an object file, then link all object files into an executable. A build system runs the sequence and skips steps whose inputs have not changed. Doing it by hand for three files is manageable. Doing it for three hundred files is not, which is why CMake exists.

CMake does not compile anything itself. You write a `CMakeLists.txt` file that describes targets, which are the things you want to build, and their sources and options. CMake reads that description and generates build files for a backend. This guide uses Ninja, which runs the commands quickly and reports each one. The two commands you will use daily are:

```bash
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

The first command configures the project and writes the Ninja files into the `build` folder. The second runs the build. A `Debug` build keeps debugging symbols and turns off most optimizations. A `Release` build does the opposite. Keep both in mind, because later days show a Debug build for debugging and a Release build for speed.

Here is the `CMakeLists.txt` for the project. It lists both source files, adds the include folder, and enables the same warnings you used by hand:

```cmake
cmake_minimum_required(VERSION 3.25)
project(hello_toolchain LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_executable(hello src/main.cpp src/geometry.cpp)
target_include_directories(hello PRIVATE include)

target_compile_options(hello PRIVATE
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)
```

The generator expressions in `target_compile_options` choose the right flags for the compiler in use, so the same file works with MSVC and with GCC or Clang. Quote each generator expression and separate its flags with semicolons, as the snippet does. Without the quotes, CMake splits the expression at each semicolon, the angle brackets no longer match, and the compiler receives a malformed flag such as `-Werror>`. `CMAKE_EXPORT_COMPILE_COMMANDS` writes a `compile_commands.json` file that editors and tools such as `clangd` use to understand your code. `CMAKE_CXX_EXTENSIONS OFF` turns off GCC-specific extensions so that the code builds under the standard alone, which helps portability.

The common mistake is to edit source files and forget that CMake caches its configuration. If you add a new `.cpp` file and the build does not see it, rerun the configure command. The fix is to rerun `cmake -S . -B build` and build again. In the market, every serious C++ project uses a build system like this, and most have a CI job that performs a clean configure and build on every change.

### Warnings

A warning is the compiler saying "this is legal, but I think you did not mean it." Warnings catch uninitialized variables, comparisons between signed and unsigned integers, and missing returns, among others. They cost nothing at run time, and the compiler can only warn about what you turn on.

The flags that matter most for this module are `-Wall` and `-Wextra`, which enable a broad set of checks, and `-Wpedantic`, which rejects code that uses compiler extensions instead of standard C++. `-Werror` turns every warning into an error. Teams use it so that warnings cannot pile up unnoticed.

The next program contains a bug that a warning catches:

```cpp
// src/uninitialized.cpp
#include <iostream>

int main() {
    int total;
    std::cout << total << '\n';
    return 0;
}
```

Compile it with `g++ -std=c++20 -Wall -Wextra -Wpedantic -Werror src/uninitialized.cpp -o build/uninitialized`. The compiler reports:

```text
src/uninitialized.cpp:5:18: error: 'total' is used uninitialized [-Werror=uninitialized]
```

The bracketed name, `-Werror=uninitialized`, tells you which warning became an error. Removing `-Werror` would turn it back into a warning, but do not remove it to make the build pass. Fix the code by giving `total` a value: `int total = 0;`. The value of an uninitialized `int` is whatever bytes happened to be in memory, so the output is unpredictable. That is undefined behavior, and the compiler's warning is a gift.

Warnings have a failure mode too. Turning on every warning in an old codebase can produce thousands of messages, and developers stop reading them. The practical approach is to enable warnings on new code, fix them in batches, and treat any new warning as a failed build. The `/W4 /WX` pair in the CMake file is MSVC's equivalent of the GCC flags, and it makes the same promise.

In the market, teams that run CI with warnings as errors report fewer production crashes from uninitialized values. The setting is common in embedded firmware, where a bug in a sensor driver can be hard to diagnose after release.

---

## Hands-on exercises

Work in a new folder called `hello-toolchain`. Run `git init -b main` in it before you start, so that every change is tracked from the first file.

Exercise 1 checks the toolchain.

1. Run `g++ --version` and note the first line. It should show version 14 or newer.
2. Run `cmake --version` and `ninja --version`. CMake should be 3.25 or newer, and Ninja 1.11 or newer.
3. Create a file `check.cpp` with `#include <iostream>` and a `main` that prints `"toolchain ok"`. Compile it with `g++ -std=c++20 -Wall -Wextra -Wpedantic -Werror check.cpp -o check` and run it.

Done when `./check` prints `toolchain ok` with no warnings during the build.

Exercise 2 builds three files by hand.

1. Create `include/geometry.hpp`, `src/geometry.cpp` and `src/main.cpp` exactly as shown in the source files section.
2. Make a `build` folder. Compile each source file to an object file with `-c`, one command per file. You should end with `build/geometry.o` and `build/main.o`.
3. Link them with `g++ build/main.o build/geometry.o -o build/hello`, then run the program.

Done when the program prints a circle area of 12.5664 for radius 2.0 and a rectangle area of 13.5 for 3.0 by 4.5.

Exercise 3 produces a linker error and fixes it.

1. Delete the definition of `rectangle_area` from `geometry.cpp`, leaving the declaration in the header.
2. Compile both files and link them. Read the undefined reference message.
3. Restore the function and link again.

Done when you have written down the exact undefined reference line, and the link succeeds after the fix.

Exercise 4 reads a warning.

1. Create `src/uninitialized.cpp` from the warnings section and compile it with the flags shown.
2. Read the warning and note the bracketed flag name.
3. Fix the variable and compile again with the same flags.

Done when the second compile produces no output at all.

Exercise 5 moves the project to CMake.

1. Write the `CMakeLists.txt` from the build process section.
2. Run `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug`, then `cmake --build build`.
3. Open `build/compile_commands.json` and find the entry for `src/main.cpp`. Read the `command` field and confirm it contains `-std=c++20` and `-Wall`.

Done when `cmake --build build` finishes with no warnings and `compile_commands.json` lists both source files.

---

## Project: a three-file program with a CMake build and warnings as errors

The project is a small geometry program. Its code is deliberately short, so the work goes into the toolchain: separate files, a build that works from a clean folder, a warning policy, and a log that explains each stage.

Suggested repo name: `100-days-software-day-27-cpp-toolchain`.

The repo layout:

```
100-days-software-day-27-cpp-toolchain/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── include/
│   └── geometry.hpp
├── src/
│   ├── geometry.cpp
│   └── main.cpp
├── tests/
│   └── check_build.sh
└── docs/
    └── build-log.md
```

The starter files are the three from the source files section, plus the `CMakeLists.txt` from the build process section. Use the same contents shown earlier. Add this `.gitignore`:

```text
build/
*.o
*.ii
```

The `tests/check_build.sh` script proves the build works from nothing. It is short, and it runs in Git Bash or WSL:

```bash
#!/usr/bin/env bash
set -euo pipefail
rm -rf build
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/hello
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, add the files, and commit with the message "Add three-file geometry program".
2. Run `bash tests/check_build.sh`. Confirm the output ends with the two area lines and that no warnings appear.
3. Create a branch `feature/add-triangle` with `git switch -c feature/add-triangle`. Add a function `triangle_area(double base, double height)` to the header and the source file, and call it from `main.cpp`. The area is `0.5 * base * height`. Build and run.
4. Introduce a warning on purpose: in `main.cpp`, add `int unused = 3;`. Run the build and confirm it fails with a `-Werror` message. Remove the line and confirm the build passes.
5. Break the link on purpose by removing `src/geometry.cpp` from `add_executable`. Run the build, copy the undefined reference line into `docs/build-log.md`, then restore the file.
6. Write `docs/build-log.md` with four short sections: the compile step, the link step, the warning you triggered and fixed, and the undefined reference you triggered and fixed. Each section names the command and the one-line result.
7. Write a `README.md` with the two commands a new reader needs to build and run the program, plus the minimum compiler and CMake versions.
8. Commit, merge `feature/add-triangle` into `main` with `git switch main` and `git merge --no-ff feature/add-triangle -m "Add triangle area"`, and create an empty GitHub repository with the name above. Then run `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `bash tests/check_build.sh` runs from a deleted `build` folder and prints the two area lines and the triangle area.
- The `-Werror` build fails on the unused variable and passes after you remove it.
- `docs/build-log.md` names the undefined reference message and the warning flag from your own runs.
- `compile_commands.json` exists in `build/` and lists all source files.
- The GitHub repo shows `main` with the merge commit from the feature branch.

Evidence to commit: `CMakeLists.txt`, the three source files, `tests/check_build.sh`, `docs/build-log.md`, `README.md`, and the commit history on GitHub.

This project is light on logic and heavy on tooling, because the skill is reading the build. A reviewer should be able to read `docs/build-log.md` and see that you can produce an error, read it, and fix it.

---

## Check your understanding

1. You add a new function `square_area` to `geometry.cpp` and call it from `main.cpp`, but you forget to declare it in `geometry.hpp`. Which stage fails, and what does the error say?
2. You compile `geometry.cpp` successfully, then change its function body. Which object files must be rebuilt, and why does a change to the `#include` list in `main.cpp` have a different effect?
3. A teammate puts a function body in a header and includes that header in two `.cpp` files. Which stage reports the problem, and why does the compile step alone not catch it?
4. Why does the build use `-std=c++20` explicitly instead of trusting the compiler default?
5. The team turns on `-Werror` in a codebase with 4,000 existing warnings. Describe a rollout plan that keeps the build green and still stops new warnings.
6. Why is `CMAKE_CXX_EXTENSIONS OFF` useful, and what does the flag change?

### Answers

1. The compile step fails for `main.cpp`, with a message like `error: 'square_area' was not declared in this scope`. The compiler must see a declaration before it can check a call. The fix is to add the declaration to the header.
2. Only `geometry.o` needs rebuilding, because the body is a definition used inside that translation unit. A change to the include list in `main.cpp` affects `main.o`, because the preprocessor pastes the headers into that file and the compiler must reread them.
3. The linker reports multiple definitions, because each object file has its own copy of the function. The compile step accepts each file on its own, since each one is valid when read alone.
4. The default standard depends on the compiler and its version, so the same code can compile on one machine and fail on another. Pinning the standard makes the build reproducible.
5. Turn on warnings for new code first, using a separate warning level or a compile option only on new files. Fix existing warnings in small batches, and then turn `-Werror` on for the whole project. CI blocks any pull request that adds a warning during the rollout.
6. With extensions on, GCC accepts non-standard features such as some GNU keywords, so code can build with GCC and fail on MSVC or Clang. Turning extensions off makes the compiler enforce the standard, which makes the build portable.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: C++](https://en.wikipedia.org/wiki/C%2B%2B) for the language history and standardization
- [Wikipedia: Compiler](https://en.wikipedia.org/wiki/Compiler) and [Wikipedia: Linker (computing)](https://en.wikipedia.org/wiki/Linker_(computing)) for the core terms
- cppreference.com, the page "Translation phases" in the language reference, for the exact stages of compilation
- GCC documentation at gcc.gnu.org/onlinedocs, the sections on "Warning Options" and "Invoking GCC"
- CMake documentation at cmake.org/cmake/help/latest, the pages for `cmake-buildsystem` and `add_executable`
- Stanford CS106L course notes, available on the Stanford CS106L website, for a university-level view of compilation and headers

*Tutorials*

- learncpp.com, the chapters on the preprocessor, header files and multiple-file programs, written by Alex Allain
- GeeksforGeeks. Search the site for "C++ compilation process" and "linker" to find the matching articles.
- Microsoft Learn. Search learn.microsoft.com for the C++ command-line compile tutorial, which walks through `cl` and the linker.

*Watch*

- The Cherno, the YouTube series "C++ Series" (search the title on YouTube), for the header and linker episodes
- CS50 on YouTube, the CS50x lectures, if you want a gentle introduction to compiled languages and the C toolchain

*Practice*

- Compiler Explorer at godbolt.org. Paste the `geometry.cpp` file and set the compiler to GCC 14 to see the object code generated for each function. Use the "Compile to binary" option to see the link step.

---

## Tomorrow

Day 28 starts writing actual C++ code. You will declare variables, choose types, use `const`, read input with `std::cin` and print with `std::cout`. The builds from today are the foundation: every program you write from Day 28 onward runs through the same compile, link and warning steps, and the `-Wall -Wextra -Wpedantic -Werror` habit you set up here carries over.

**Next:** [Day 28: Core Syntax and Types](Software-Day-28-Core-Syntax-and-Types.md)

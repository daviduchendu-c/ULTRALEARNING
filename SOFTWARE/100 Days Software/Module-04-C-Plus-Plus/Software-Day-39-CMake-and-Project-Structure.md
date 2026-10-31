# Day 39: CMake and Project Structure

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 5 hours
**Level:** intermediate. You have built single-folder projects with CMake since Day 27, and you have linked libraries and fetched GoogleTest in Days 38 and earlier. This day turns that into a project that a team can grow: separate libraries, applications and tests, clear include paths, dependencies that are declared once, presets for each build mode, and a continuous integration workflow that builds and tests every change. It is for anyone whose CMake file has grown into a single long list of files that nobody dares to touch.

**By the end of this day, you will have:**

- Declared library, executable and interface targets, and explained the difference between them
- Set include directories as `PUBLIC`, `PRIVATE` or `INTERFACE`, and explained what each one leaks to dependents
- Declared a dependency with `FetchContent` and linked it to one target only
- Written `CMakePresets.json` with Debug, Release and sanitizer presets, and built and tested each one with a single command
- Written a GitHub Actions workflow that builds and tests on Ubuntu and Windows
- A multi-target repository with a library, a command-line app, tests and CI, all in one GitHub repo

**What you need:**

- CMake 3.25 or newer (checked with CMake 4.3.3), which supports the presets format used here
- Ninja 1.11 or newer
- GCC 14 or newer (checked with g++ 16.1.0 on Windows), and Ubuntu 24.04 in WSL 2 for the sanitizer preset
- Git 2.40 or newer
- Internet access, for `FetchContent` and for GitHub Actions
- A free GitHub account, with a repository that can run Actions

**Prerequisites:**

- Days 27 to 38: the build steps, targets, warnings as errors, GoogleTest and sanitizers
- Day 6 and Day 8: Git branches, pushing to GitHub and pull requests

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Targets](#targets)
- [Libraries](#libraries)
- [Include paths](#include-paths)
- [Dependencies](#dependencies)
- [Builds](#builds)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a multi-target text tool with presets and continuous integration](#project-a-multi-target-text-tool-with-presets-and-continuous-integration)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A small C++ program can live in one folder with one `CMakeLists.txt` that lists its files. A real project outgrows that. Once there are two programs that share code, a test suite and a dependency on a third-party library, the build description has to say which parts depend on which, and it has to say it once. Without that structure, every new file changes the build in several places, and a change in one place breaks another.

CMake is built for that job. It describes a project as targets, which are the libraries, executables and tests you build, and it records how each target uses the others. Modern CMake puts all the rules on the target, so an include path or a compile flag travels with the target that needs it. This day shows the target model, the way include paths and dependencies flow between targets, and the presets and continuous integration that make the build repeatable.

The CMake language and its target model are documented in the official CMake manual at cmake.org. The C++ standard itself says nothing about builds, so the rules here are CMake's and your compiler's.

---

## Glossary

- **Target:** a thing CMake builds, such as a library, an executable or an interface library, created with `add_library` or `add_executable`.
- **Library target:** a target that produces a static or shared library for other targets to link against.
- **Executable target:** a target that produces a program.
- **Interface library:** a target that has no output of its own. It carries usage requirements, such as include paths and compile flags, to the targets that link it.
- **Usage requirement:** a property, such as an include directory, that a target gives to the targets that depend on it.
- **PUBLIC, PRIVATE, INTERFACE:** the keywords that say whether a usage requirement applies to the target itself, to its dependents, or to both.
- **Dependency:** another target or package that a target needs in order to build.
- **FetchContent:** a CMake module that downloads and builds a dependency at configure time.
- **Preset:** a named set of configure, build and test options stored in `CMakePresets.json`.
- **Out-of-source build:** a build whose output goes to a separate folder, such as `build/`, so the source tree stays clean.
- **Continuous integration:** an automated process that builds and tests the project on every change, usually on a server such as GitHub Actions.
- **Install rule:** a CMake instruction that copies built files to a location a user can install from.

---

## Core concepts

The concepts form the target model in order: what a target is, how libraries are shaped, how include paths flow to dependents, how external code enters the project, and how the build modes and automation tie it together.

### Targets

A target is the unit of a CMake project. Think of the parts of a kitchen. The oven, the sink and the fridge are separate appliances, each with its own power and plumbing, and they are wired into the house once. A target is an appliance. It has sources, compile options, include paths and link dependencies, and CMake connects it to the others.

In first principles, `add_executable` and `add_library` create a target with a name, and every later command that names the target adds properties to it. `target_sources`, `target_include_directories`, `target_compile_options` and `target_link_libraries` are the commands that fill those properties. The older style used global variables such as `CMAKE_CXX_FLAGS`, which apply to every target and cannot be scoped. The modern style sets properties on the target, so the rule is local.

Here is a small target layout, which the project in this day builds on:

```cmake
cmake_minimum_required(VERSION 3.25)
project(text_tools LANGUAGES CXX)

add_library(textkit src/textkit.cpp)
target_include_directories(textkit PUBLIC include)

add_executable(textcli apps/textcli/main.cpp)
target_link_libraries(textcli PRIVATE textkit)
```

The file declares two targets. The executable depends on the library, and CMake builds the library first. The `textcli` target does not list the library's source file, because the library is a separate target with its own sources.

The common mistake is to list every source file in one executable, which works until a second program needs the same code and the sources are copied. The fix is to move shared code into a library target and link both programs to it. In the market, large C++ codebases such as game engines and browsers have hundreds of targets, and the target graph is the main map of the code.

### Libraries

A library target groups code that several programs or tests use. Think of a toolbox. The tools are used in many jobs, so they are kept together and shared. CMake offers two main kinds: static libraries, which are copied into each program that uses them, and shared libraries, which are loaded at run time. The `add_library` command chooses the kind with the keywords `STATIC`, `SHARED` or `INTERFACE`, and without a keyword it follows the `BUILD_SHARED_LIBS` variable.

Static libraries are simpler to ship in a small project, and they avoid run-time loading issues on Windows, where a shared library must be found next to the program or on the path. Shared libraries save space when many programs use the same code, and they allow the library to be replaced without rebuilding the programs. Days 29 to 38 used a static library in the same way, and this day keeps that choice.

The header-only libraries from Day 35 use `INTERFACE`. An interface library has no sources of its own. It carries include directories and compile options, and the targets that link it receive them. The project uses an interface library to hold the warning settings, so every target shares one policy.

The common mistake is to build a library and then forget to mark its include directory as `PUBLIC`, so programs that link it cannot find its headers. The error is a missing header, which looks like a problem with the program rather than with the library. In the market, library packaging decisions such as static or shared linking are made per platform, and build engineers document them carefully.

### Include paths

An include path tells the compiler where to find headers. The question is who else needs that path. Think of the directions to a building. The owner of the building knows the address, and visitors need it too. A `PUBLIC` include path is the address printed on the business card: the library needs it, and so do the programs that link it. A `PRIVATE` path is a shortcut the library uses internally, which visitors do not need. An `INTERFACE` path is for users only, which is what an interface library provides.

```cmake
add_library(textkit src/textkit.cpp)
target_include_directories(textkit
    PUBLIC  include    # headers that programs using textkit include
    PRIVATE src        # internal headers that only textkit's sources include
)
```

The keyword does what the table says. With `PUBLIC`, the include directory is added to the library itself and to every target that links it, so `#include "textkit/textkit.hpp"` works in the app without extra setup. With `PRIVATE`, the directory affects only the library's own sources. The rule is to make the public headers `PUBLIC`, keep internal headers `PRIVATE`, and never add a global include directory with `include_directories`, which leaks to every target in the directory.

The common mistake is to use `include_directories` at the top of the file, which is simpler and wrong for every target that should not see those headers. The second mistake is to make everything `PUBLIC`, so internal headers leak into the public interface and can be included by accident. The fix is to think about the interface first. In the market, the difference between public and private include paths decides which headers a library promises to keep stable, and it is part of the versioning contract.

### Dependencies

A dependency is a piece of code your target needs and does not write yourself. Think of ingredients you buy instead of growing. You need to know which supplier, which version and which label. CMake offers three ways to bring one in. `find_package` looks for a library already installed on the machine. `FetchContent` downloads the source at configure time and builds it with your project. The third way is a package manager such as vcpkg or Conan, which is out of scope for this day.

`FetchContent` is the simplest route for a learning project, because it needs no installation, only the URL and a tag. The tag is a fixed version, such as `v1.17.0`, so every build gets the same code. Here is GoogleTest declared once and used by one target:

```cmake
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.17.0
    GIT_SHALLOW TRUE
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

add_executable(textkit_test tests/textkit_test.cpp)
target_link_libraries(textkit_test PRIVATE textkit GTest::gtest_main)
```

The dependency is linked `PRIVATE` to the test, so nothing else sees GoogleTest. The library `textkit` does not depend on GoogleTest, and the application does not either. The keyword matters: if the library linked GoogleTest `PUBLIC`, every program that used the library would also receive the test framework, which is a size and build-time cost for no reason.

The common mistake is to use a floating tag such as `main` for a dependency. The next build may fetch different code, and a bug can appear without any change in your repo. Pin a tag or a commit hash. The second mistake is to declare a dependency in every subdirectory. Declare it once, and link it where it is used. In the market, dependency pinning is part of supply-chain security, and teams review version changes the way they review code.

### Builds

A build is one run of the compiler and linker over the targets, with a chosen configuration. The two configurations you have used are Debug and Release. Debug keeps symbols and turns off optimization, so debugging works and the program is slower. Release turns on optimization and removes assertions, so the program is fast and the bugs that only appear with optimization can hide. Sanitizer builds are a third kind, with extra checks that slow the program.

CMake supports two styles. Single-configuration generators, such as Ninja, use one build type per build folder, chosen with `CMAKE_BUILD_TYPE` at configure time. Multi-configuration generators, such as Visual Studio, build several types from one folder, chosen at build time. The project in this day uses one folder per preset, which works the same way on both styles of generator.

Presets make the build choice repeatable. A `CMakePresets.json` file records the generator, the build folder and the cache variables for each named configuration. The commands become short:

```bash
cmake --preset debug
cmake --build --preset debug
ctest --preset debug
```

The first command configures the build into `build/debug`. The second builds it, and the third runs the tests with the output shown on failure. The same three commands run on Windows and Linux, and the continuous integration workflow uses them unchanged, which means the local build and the server build match. Install rules are the last piece. The `install` command copies a library's headers and binaries to a prefix folder, which a user can install from or package later.

The common mistake is to build in the source folder, which mixes generated files with source files, and then cannot clean the build without risk. The fix is an out-of-source build folder, which the presets enforce. In the market, reproducible builds and the ability to run the same commands in CI are standard expectations for any team, because they are the basis of trust in a release.

---

## Hands-on exercises

Work in a new folder called `cmake-structure`. Run `git init -b main` in it.

Exercise 1 creates two targets.

1. Write a library `greet` with one function `std::string greeting(std::string_view name)`, in `src/greet.cpp` with its header in `include/greet/greet.hpp`.
2. Write an executable `hello` in `apps/hello/main.cpp` that prints the greeting, and link it to `greet`.
3. Build with `cmake -S . -B build -G Ninja` and `cmake --build build`, then run `./build/apps/hello/hello` or the equivalent on Windows.

Done when the program prints the greeting and the build has no warnings.

Exercise 2 changes an include path from global to target-level.

1. Replace any global `include_directories` line with `target_include_directories`, using `PUBLIC` for `include` and `PRIVATE` for `src`.
2. Remove the `PUBLIC` keyword from the `include` path, rebuild the app, and read the error. Restore it.

Done when you have the missing-header error text in your notes.

Exercise 3 adds a presets file.

1. Write `CMakePresets.json` with version 3, and presets named `debug` and `release`, each using the Ninja generator and a `binaryDir` under `build/`.
2. Run `cmake --preset debug`, `cmake --build --preset debug` and `ctest --preset debug` in order. Then do the same for `release`.

Done when both presets build, and the build folders are separate.

Exercise 4 fetches a dependency.

1. Add `FetchContent` for GoogleTest pinned to `v1.17.0`, and link it `PRIVATE` to a test executable.
2. Write one test that checks the greeting function and register it with `gtest_discover_tests`.
3. Run `ctest --preset debug --output-on-failure`.

Done when the test passes and `ctest` lists it by name.

Exercise 5 runs the install rule.

1. Add `install(TARGETS greet ARCHIVE DESTINATION lib)` and `install(DIRECTORY include/ DESTINATION include)` to the library's CMake file.
2. Run `cmake --install build/debug --prefix build/install-test` and list the installed files.

Done when the header and the library file appear under the install prefix.

---

## Project: a multi-target text tool with presets and continuous integration

The project is a small repository that a team could grow. It has a library, `textkit`, with word counting and slug generation. It has a command-line app, `textcli`, that uses the library. It has a GoogleTest suite, and it has presets for Debug, Release and sanitizers. A GitHub Actions workflow builds and tests it on every push.

Suggested repo name: `100-days-software-day-39-cmake-and-project-structure`.

The repo layout:

```
100-days-software-day-39-cmake-and-project-structure/
├── .github/
│   └── workflows/
│       └── ci.yml
├── .gitignore
├── CMakeLists.txt
├── CMakePresets.json
├── README.md
├── apps/
│   └── textcli/
│       └── main.cpp
├── docs/
│   └── build-matrix.md
├── libs/
│   └── textkit/
│       ├── CMakeLists.txt
│       ├── include/
│       │   └── textkit/
│       │       └── textkit.hpp
│       └── src/
│           └── textkit.cpp
└── tests/
    ├── CMakeLists.txt
    └── textkit_test.cpp
```

The root `CMakeLists.txt` sets the standard, adds the sanitizer option, defines the warning policy as an interface target, and adds the subdirectories. The `include(CTest)` line creates the `BUILD_TESTING` option, which lets a user skip the tests:

```cmake
cmake_minimum_required(VERSION 3.25)
project(text_tools LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(ENABLE_SANITIZERS "Build with AddressSanitizer and UndefinedBehaviorSanitizer" OFF)

if(ENABLE_SANITIZERS AND NOT MSVC)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=address,undefined)
endif()

# Shared warning policy. Targets opt in by linking this interface target.
add_library(project_warnings INTERFACE)
target_compile_options(project_warnings INTERFACE
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_subdirectory(libs/textkit)
add_subdirectory(apps/textcli)

include(CTest)
if(BUILD_TESTING)
    add_subdirectory(tests)
endif()
```

The library's own `CMakeLists.txt` declares the public header folder, the private warning link and the install rules. The namespaced alias lets the rest of the project use `textkit::textkit`, which reads clearly and fails if the name is misspelled:

```cmake
# libs/textkit/CMakeLists.txt
add_library(textkit STATIC src/textkit.cpp)
add_library(textkit::textkit ALIAS textkit)

target_include_directories(textkit PUBLIC include)
target_link_libraries(textkit PRIVATE project_warnings)

install(TARGETS textkit ARCHIVE DESTINATION lib)
install(DIRECTORY include/ DESTINATION include)
```

The public header declares the functions. Its namespace is the library's name, so callers can see where each function comes from:

```cpp
// libs/textkit/include/textkit/textkit.hpp
#ifndef TEXTKIT_TEXTKIT_HPP
#define TEXTKIT_TEXTKIT_HPP

#include <cstddef>
#include <string>
#include <string_view>

namespace textkit {

// Counts the whitespace-separated words in the text.
std::size_t word_count(std::string_view text);

// Lowercases letters and digits, and joins each run of other characters with one hyphen.
std::string slugify(std::string_view text);

}  // namespace textkit

#endif
```

The implementation is in the library's private source file, and it includes only the public header:

```cpp
// libs/textkit/src/textkit.cpp
#include "textkit/textkit.hpp"

#include <cctype>

namespace textkit {

std::size_t word_count(std::string_view text) {
    std::size_t count = 0;
    bool in_word = false;
    for (const char c : text) {
        const bool space = std::isspace(static_cast<unsigned char>(c)) != 0;
        if (!space && !in_word) {
            ++count;
        }
        in_word = !space;
    }
    return count;
}

std::string slugify(std::string_view text) {
    std::string slug;
    bool pending_hyphen = false;
    for (const char c : text) {
        const auto uc = static_cast<unsigned char>(c);
        if (std::isalnum(uc)) {
            if (pending_hyphen && !slug.empty()) {
                slug.push_back('-');
            }
            pending_hyphen = false;
            slug.push_back(static_cast<char>(std::tolower(uc)));
        } else {
            pending_hyphen = true;
        }
    }
    return slug;
}

}  // namespace textkit
```

The application uses the library through its public interface and reads standard input:

```cpp
// apps/textcli/main.cpp
#include <iostream>
#include <string>

#include "textkit/textkit.hpp"

int main() {
    std::string all;
    std::string line;
    while (std::getline(std::cin, line)) {
        all += line;
        all += '\n';
    }

    std::cout << "words: " << textkit::word_count(all) << '\n';
    std::cout << "slug: " << textkit::slugify(all) << '\n';
    return 0;
}
```

The application's `CMakeLists.txt` links the library and the warning policy:

```cmake
# apps/textcli/CMakeLists.txt
add_executable(textcli main.cpp)
target_link_libraries(textcli PRIVATE textkit::textkit project_warnings)
```

The test folder fetches GoogleTest, pinned to a tag, and links it privately to the test only:

```cmake
# tests/CMakeLists.txt
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.17.0
    GIT_SHALLOW TRUE
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

include(GoogleTest)

add_executable(textkit_test textkit_test.cpp)
target_link_libraries(textkit_test PRIVATE textkit::textkit GTest::gtest_main project_warnings)
gtest_discover_tests(textkit_test)
```

```cpp
// tests/textkit_test.cpp
#include <gtest/gtest.h>

#include "textkit/textkit.hpp"

TEST(WordCount, EmptyTextHasNoWords) {
    EXPECT_EQ(textkit::word_count(""), 0u);
}

TEST(WordCount, IgnoresRepeatedSpaces) {
    EXPECT_EQ(textkit::word_count("  a  b "), 2u);
}

TEST(Slugify, JoinsWordsWithHyphens) {
    EXPECT_EQ(textkit::slugify("Hello, World!  C++20"), "hello-world-c-20");
}

TEST(Slugify, PunctuationOnlyIsEmpty) {
    EXPECT_EQ(textkit::slugify("!!!"), "");
}

TEST(Slugify, KeepsExistingHyphenBoundaries) {
    EXPECT_EQ(textkit::slugify("already-slug"), "already-slug");
}
```

The presets file defines three configurations, each with its own build folder, and a build and test preset for each one:

```json
{
  "version": 3,
  "cmakeMinimumRequired": { "major": 3, "minor": 25, "patch": 0 },
  "configurePresets": [
    {
      "name": "debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    },
    {
      "name": "release",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    },
    {
      "name": "asan",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/asan",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug", "ENABLE_SANITIZERS": "ON" }
    }
  ],
  "buildPresets": [
    { "name": "debug", "configurePreset": "debug" },
    { "name": "release", "configurePreset": "release" },
    { "name": "asan", "configurePreset": "asan" }
  ],
  "testPresets": [
    { "name": "debug", "configurePreset": "debug", "output": { "outputOnFailure": true } },
    { "name": "release", "configurePreset": "release", "output": { "outputOnFailure": true } },
    { "name": "asan", "configurePreset": "asan", "output": { "outputOnFailure": true } }
  ]
}
```

The workflow builds and tests each preset on two operating systems. It installs Ninja first, because hosted runners do not all provide it by default. A separate job runs the sanitizer preset on Ubuntu:

```yaml
# .github/workflows/ci.yml
name: ci

on:
  push:
  pull_request:

jobs:
  build-and-test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-24.04, windows-2022]
        preset: [debug, release]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - name: Install Ninja on Linux
        if: runner.os == 'Linux'
        run: sudo apt-get update && sudo apt-get install -y ninja-build

      - name: Install Ninja on Windows
        if: runner.os == 'Windows'
        run: choco install ninja -y

      - name: Configure
        run: cmake --preset ${{ matrix.preset }}

      - name: Build
        run: cmake --build --preset ${{ matrix.preset }}

      - name: Test
        run: ctest --preset ${{ matrix.preset }}

  sanitizers:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4

      - name: Install Ninja
        run: sudo apt-get update && sudo apt-get install -y ninja-build

      - name: Configure with sanitizers
        run: cmake --preset asan

      - name: Build with sanitizers
        run: cmake --build --preset asan

      - name: Test with sanitizers
        run: ctest --preset asan
```

The `docs/build-matrix.md` file explains the presets to a reader in a few lines: which preset is for daily work, which one is for release checks, and which one is for memory bugs.

Step-by-step tasks:

1. Create the layout and files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Restructure into library, app and tests".
2. Run `cmake --preset debug`, `cmake --build --preset debug` and `ctest --preset debug --output-on-failure`. Confirm the five tests pass.
3. Run `./build/debug/apps/textcli/textcli` and type `Hello, World!  C++20`, then press Ctrl+Z and Enter on Windows or Ctrl+D on Linux. Check that it prints `words: 3` and `slug: hello-world-c-20`.
4. Run the release preset the same way. Then, in WSL, run `cmake --preset asan`, build and test. Confirm there are no sanitizer reports.
5. Create a branch `feature/truncate`. Write a failing test first for `std::string truncate(std::string_view text, std::size_t max_length)` that returns the text unchanged when it fits, and the first `max_length` characters otherwise. Add the declaration to the public header, implement it in the library, and make the test pass.
6. Push the repository to GitHub with `git remote add origin <URL>` and `git push -u origin main`, then open the Actions tab. Confirm that every job in the workflow passes. If a job fails, read its log and fix the cause in a new commit.
7. Write `docs/build-matrix.md` with a table of the three presets, the command for each, and the job that runs it. Commit it.
8. Merge the branch with `git merge --no-ff feature/truncate -m "Add truncate helper"`, push, and confirm the workflow passes on `main`.

Done when:

- All three presets build, and `ctest` passes for each one locally.
- The app prints the two lines from task 3.
- The GitHub Actions workflow passes on `main` for every job.
- `docs/build-matrix.md` describes each preset and where it runs.

Evidence to commit: the root and library `CMakeLists.txt`, `CMakePresets.json`, the source, public header and test files, the workflow file, `docs/build-matrix.md`, `README.md`, and the Actions run link in the README.

This project is light on new C++ and heavy on structure. A reviewer should see a target graph in the CMake files, a public header that is small and stable, a dependency that is pinned, and a green workflow badge on the default branch.

---

## Check your understanding

1. A library uses `target_include_directories(lib PRIVATE include)`, and an app that links the library cannot find its header. Explain why, and state the keyword that fixes it.
2. A test target and the app both link GoogleTest through the library's `PUBLIC` link. Describe the cost, and the change that keeps the test framework private.
3. Why does a `FetchContent` declaration with `GIT_TAG main` make a build less reproducible, and what should the tag be instead?
4. A team builds in the source folder with a single `CMakeLists.txt`. Name two problems this causes and the fix.
5. The presets file gives the same commands on Windows and Linux. Explain why that works, and what the workflow gains from using the same commands.
6. A library sets `CMAKE_CXX_FLAGS` globally for warnings. Describe a case where this breaks a third-party dependency, and how an interface target avoids it.

### Answers

1. `PRIVATE` makes the include directory apply only to the library's own sources, so dependents do not get it. The fix is `PUBLIC` for the directory that holds the public headers, which adds it to the library and to every target that links it.
2. Every program that uses the library also receives GoogleTest, which increases build time, link size and the set of dependencies those programs must have. The fix is to link GoogleTest `PRIVATE` in the test target only, so the library and its users never see it.
3. A branch name moves over time, so two builds at different times can fetch different code, and a bug can appear with no change in your repository. Use a release tag such as `v1.17.0` or a full commit hash.
4. Generated files mix with source files, which makes the source tree dirty and hard to clean, and a clean build risks deleting work. The fix is an out-of-source build folder, which the presets set with `binaryDir`.
5. CMake presets describe the configure, build and test steps in one file, and the commands read the file on each platform. The workflow gains the same steps as a developer's local machine, so a green local run predicts a green CI run.
6. Setting `CMAKE_CXX_FLAGS` affects every target in the project, including third-party code that may fail under the flags, such as `-Werror` with a warning the dependency triggers. An interface target applies the warnings only to targets that link it, which keeps the policy local to the project's own code.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: CMake](https://en.wikipedia.org/wiki/CMake) for the tool and its history
- [Wikipedia: Build automation](https://en.wikipedia.org/wiki/Build_automation) for the general practice
- [Wikipedia: Continuous integration](https://en.wikipedia.org/wiki/Continuous_integration) for the workflow idea
- The CMake documentation at cmake.org/cmake/help/latest, the pages for `cmake-buildsystem`, `cmake-presets`, `target_include_directories`, `FetchContent` and `install`
- The GitHub Actions documentation at docs.github.com/actions, the pages for workflow syntax and matrix builds
- The GoogleTest quickstart with CMake in the googletest repository on GitHub

*Tutorials*

- The CMake tutorial at cmake.org/cmake/help/latest/guide/tutorial, which walks through a library and an executable with tests
- GeeksforGeeks, search the site for "CMake tutorial" and "CMake C++" for the matching articles
- Microsoft Learn, search learn.microsoft.com for "CMake projects in Visual Studio" for the Windows view of presets

*Watch*

- Jason Turner's C++ Weekly on YouTube, search the channel for "CMake" to find the episodes on target-based design
- CppCon, the YouTube channel, for talks on CMake and build systems. Search the channel for "modern CMake" to find them.

*Practice*

- Compiler Explorer at godbolt.org, which supports CMake-free compilation. Compare the flags the compiler receives in its settings with the flags CMake passes in `compile_commands.json` from one of your builds.

---

## Tomorrow

Day 40 is the C++ capstone. You will build a small command-line expense ledger that stores records in a CSV file. It uses the RAII file handle and the buffer from Day 32, the smart pointers from Day 37, the containers from Day 36, the error types from Day 38 and the target layout from today. Its tests run on every push through the workflow you wrote today, so the capstone is a complete project from the first line of code to a green build.

**Next:** [Day 40: C++ Capstone](Software-Day-40-C-Plus-Plus-Capstone.md)

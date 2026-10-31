# Day 32: Memory and Lifetime

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** intermediate. You have used pointers and references from Day 31 and know that an object can be destroyed while a pointer still holds its address. This day explains where objects live, when they die, and how a C++ class can make cleanup automatic. It is for anyone who has leaked memory in a loop and wondered where it went.

**By the end of this day, you will have:**

- Explained the difference between automatic storage on the stack and dynamic storage on the heap, with one example of each
- Allocated and freed an object with `new` and `delete`, and an array with `new[]` and `delete[]`, and matched each pair correctly
- Traced the constructor and destructor order for nested scopes, and explained why it is reversed
- Written an RAII guard that releases a resource on every exit path, including when an exception is thrown
- A small library with a deep-copying integer buffer, a file wrapper that closes itself and a test executable run by CTest
- A GitHub repo with a sanitizer run in WSL and a short note on the lifetime rules you verified

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- Ubuntu 24.04 in WSL 2 with `sudo apt install g++-14`, for the AddressSanitizer run. GCC ships the sanitizer runtime, so no extra package is needed for it.
- A free GitHub account

**Prerequisites:**

- Days 27 to 31: the build, functions, references, pointers and the null-pointer habit
- Day 28: integer types

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Stack and heap](#stack-and-heap)
- [New and delete](#new-and-delete)
- [Object lifetime](#object-lifetime)
- [RAII](#raii)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a resource library with a deep-copy buffer and a file wrapper](#project-a-resource-library-with-a-deep-copy-buffer-and-a-file-wrapper)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every program holds data for a while and then lets it go. The question is who decides when that happens. In a language with a garbage collector, a runtime checks for unused objects and frees them, at a time you do not control. C++ leaves that choice to you, and the language gives you rules that make the decision predictable. If you follow the rules, memory is released exactly once, at the right time. If you break them, you get leaks, double frees or use-after-free bugs.

This day covers the two places objects can live, the lifetime rules that say when they begin and end, and RAII, the C++ idiom that connects a resource's lifetime to a scope. RAII is the reason well-written C++ rarely calls `delete` by hand. Learning it now makes the containers and smart pointers in later days feel natural.

The lifetime rules are defined in the ISO C++ standard. cppreference.com has pages on storage duration and on object lifetime, and both are worth reading alongside this day.

---

## Glossary

- **Stack:** the region of memory that holds local variables and function call data. It grows and shrinks as functions are called and return.
- **Heap:** the region of memory that a program requests at run time with `new`. Its objects stay alive until something frees them.
- **Automatic storage:** the lifetime of a local variable, which begins when its scope is entered and ends when it is left.
- **Dynamic storage:** the lifetime of an object created with `new`, which ends only when `delete` runs.
- **Allocation:** asking the system for memory, which `new` does for you.
- **Deallocation:** returning memory to the system, which `delete` does for you.
- **Constructor:** a function that sets up an object when its lifetime begins.
- **Destructor:** a function that cleans up an object when its lifetime ends, written `~ClassName()`.
- **Leak:** memory that was allocated and never freed, so no code can reach it again.
- **Double free:** calling `delete` twice on the same pointer, which is undefined behavior.
- **RAII:** Resource Acquisition Is Initialization. A resource is acquired in a constructor and released in a destructor, so the scope controls it.
- **Deep copy:** a copy that allocates its own memory and copies the contents, instead of copying the pointer.

---

## Core concepts

The concepts form a sequence. You first need to know where objects live, then how to create and destroy them by hand, then how the language ends their lifetime for you, and finally how to use that behavior to manage a resource safely.

### Stack and heap

Think of the stack as a stack of plates in a cafeteria. You add a plate on top and take the top plate off, in that order, so the last plate you added is the first one you remove. Local variables work the same way. When a function is called, its variables are pushed onto the stack, and when it returns, they are popped off in reverse order. The cost is almost nothing, and nobody has to remember to clean up.

The heap is more like a warehouse with rented shelves. You ask for a shelf of a given size, you get its address, and the shelf stays yours until you give it back. Nothing returns the shelf for you. That freedom is what makes the heap useful, because the size can be decided at run time and the object can outlive the function that created it. It is also what makes the heap dangerous.

```cpp
#include <iostream>

int main() {
    int on_stack = 10;           // automatic storage: released when main returns
    int* on_heap = new int(20);  // dynamic storage: lives until delete

    std::cout << on_stack << ' ' << *on_heap << '\n';
    delete on_heap;              // return the memory to the system
    return 0;
}
```

The output is `10 20`. The variable `on_stack` needs no cleanup. The heap object needs an explicit `delete`. If you removed that line, the program would still print the same output, and it would leak four bytes. A single leak is small, but a leak in a loop that runs for a week adds up.

The common mistake is to take the address of a stack variable and keep it after the function returns. That pointer refers to a stack slot that has been popped, and it is the dangling pointer from Day 31. The rule is simple: heap memory lives until you free it, and stack memory lives until its scope ends. In the market, the division between stack and heap matters most in embedded firmware and game engines, where allocations inside a frame loop cause visible stutter.

### New and delete

The `new` operator allocates an object on the heap and returns a pointer to it. The `delete` operator destroys the object and frees its memory. Each `new` needs exactly one matching `delete`, and each `new[]` needs exactly one matching `delete[]`. Mixing them is undefined behavior, because the runtime stores extra bookkeeping for arrays that `delete` does not know how to read.

```cpp
#include <iostream>

int main() {
    int* values = new int[4]{1, 2, 3, 4};

    int total = 0;
    for (int i = 0; i < 4; ++i) {
        total += values[i];
    }
    std::cout << "total " << total << '\n';

    delete[] values;  // must match new[]
    return 0;
}
```

The output is `total 10`. The braces after `new int[4]` initialize the array, so the values are not garbage. Forgetting the `[]` in `delete[]` is the classic array mistake, because the code compiles and often appears to work, which is what makes it dangerous.

The limit of `new` and `delete` is that every path through the function must reach the `delete`. An early `return` or an exception skips it, and the memory leaks. The next two sections show how C++ removes that burden from you. For now, learn the rule and the matching pairs, and notice how many places in your code would need a `delete`.

The common mistake is a double free. If two pointers refer to the same heap object and both call `delete`, the second call corrupts the heap. Setting the pointer to `nullptr` after `delete` makes the second `delete` harmless, because `delete nullptr` does nothing. You can spot a double free by a crash that happens only in some runs. In the market, native code reviews look for every `new` and check the matching `delete` by hand, which is slow, and the reason C++ teams avoid raw `new` today.

### Object lifetime

An object's lifetime begins when its constructor finishes and ends when its destructor starts. Between those points, the object exists and can be used. A local object's destructor runs when its scope ends, and the order is reversed from the order of construction. The last object created is the first one destroyed, as with the stack of plates.

The next program prints each constructor and destructor call, so you can see the order:

```cpp
#include <iostream>
#include <string>
#include <utility>

class Noisy {
public:
    explicit Noisy(std::string name) : name_(std::move(name)) {
        std::cout << "construct " << name_ << '\n';
    }

    ~Noisy() {
        std::cout << "destruct " << name_ << '\n';
    }

private:
    std::string name_;
};

int main() {
    Noisy outer("outer");
    {
        Noisy inner("inner");
        std::cout << "in inner scope\n";
    }
    std::cout << "back in main\n";
    return 0;
}
```

The output is `construct outer`, `construct inner`, `in inner scope`, `destruct inner`, `back in main`, and then `destruct outer`. The inner object dies at the closing brace of its block, and the outer object dies when `main` returns. The `std::move` in the constructor moves the string into the member, which avoids a copy. Day 37 explains move semantics in full.

Lifetime rules also apply to members. A class's members are constructed in the order they are declared, and destroyed in the reverse order. This matters when one member depends on another. The rule is to declare members in the order their dependencies are constructed, and to keep each destructor simple.

The common mistake is to think a destructor runs at a random time, or not at all when an exception is thrown. In fact it runs on schedule, even during unwinding. Local objects in a scope are destroyed when the stack unwinds, which is what makes RAII work. The mistake that does cause real trouble is to create an object with `new` and forget that it has no automatic destruction. You can spot it when a log shows the constructor message but never the destructor one. In the market, lifetime bugs in long-running servers often come from objects that outlive the thread that created them.

### RAII

RAII stands for Resource Acquisition Is Initialization, and the name is clumsy but the idea is simple. A resource is any object you must return: heap memory, a file handle, a network socket, a lock. RAII says that you acquire the resource in a constructor and release it in a destructor. The object's scope then controls the resource, so the cleanup happens on every exit path without a line of cleanup code in the function.

Think of a library card or a rented locker. You get the card when you enter, and you hand it back when you leave, whether you finish your work normally or leave in a hurry. The library does not depend on your memory to take the book back, because the card is tied to your visit.

```cpp
#include <iostream>
#include <stdexcept>

class Guard {
public:
    explicit Guard(const char* name) : name_(name) {
        std::cout << "acquire " << name_ << '\n';
    }

    ~Guard() {
        std::cout << "release " << name_ << '\n';
    }

    Guard(const Guard&) = delete;
    Guard& operator=(const Guard&) = delete;

private:
    const char* name_;
};

void work(bool fail) {
    Guard lock("work");
    if (fail) {
        throw std::runtime_error("failed midway");
    }
    std::cout << "work done\n";
}

int main() {
    try {
        work(false);
        work(true);
    } catch (const std::exception& e) {
        std::cout << "caught: " << e.what() << '\n';
    }
    return 0;
}
```

The output is `acquire work`, `work done`, `release work`, `acquire work`, `release work`, and then `caught: failed midway`. The second call to `work` throws, and the guard is still released before the exception reaches `main`. The function has no cleanup code at all, so there is nothing to forget. The deleted copy operations prevent two guards from releasing the same resource.

The common mistake is to write a class that owns a resource but forgets the copy rules. If the class is copied, both copies try to release the same resource, which is a double free in disguise. The fix is to delete the copy operations, as the guard does, or to implement a deep copy, as the project does. The rule of thumb from the C++ community is that a class that manages a resource must define or delete the destructor, the copy constructor and the copy assignment, and must consider the move operations too.

In the market, RAII is the reason C++ code in databases, browsers and trading systems can handle exceptions without leaking. Companies that run long-lived C++ services usually have a style guide that bans raw `new` in application code and requires RAII wrappers instead.

---

## Hands-on exercises

Work in a new folder called `memory-lifetime`. Run `git init -b main` in it.

Exercise 1 allocates and frees one object.

1. Allocate an `int` with `new` and set it to 42. Print it, then free it with `delete`.
2. Run the program under the AddressSanitizer build in WSL, described in the project. Confirm the sanitizer reports no leaks.
3. Remove the `delete` line and run again. Record whether the sanitizer reports a leak, and the number of bytes it lists.

Done when you have the leak report from step 3 in your notes.

Exercise 2 traces constructor order.

1. Write a class `Trace` that prints its name in the constructor and the destructor, as in the object lifetime section.
2. Create three `Trace` objects in `main`, one inside a nested block, and predict the full output before running it.
3. Compare the output with your prediction and explain the one line that surprised you.

Done when the prediction and the output agree, or the difference is explained.

Exercise 3 forgets a `delete[]`.

1. Allocate an array with `new int[100]` and do not free it. Then allocate another with `new int[100]` and free it with `delete` instead of `delete[]`.
2. Build and run in WSL with `-fsanitize=address`. Read the report and note which of the two errors it names.

Done when you can explain the difference between `delete` and `delete[]` in one sentence.

Exercise 4 writes a guard that releases on an exception.

1. Write a `Guard` class as in the RAII section.
2. Write a function that creates a guard, then calls a function that throws an exception on its third step.
3. Catch the exception in `main` and check that the release message appears before the catch message.

Done when the release message appears before the caught message.

Exercise 5 breaks a copy on purpose.

1. Write a class that owns an `int*` allocated in its constructor and freed in its destructor, with no copy operations.
2. Copy an object into another and run the program under the sanitizer. Read the double free report.
3. Add a deleted copy constructor and assignment, and show the compiler error when you try the copy again.

Done when you have both reports in your notes and can say which one the deleted copy operations prevent.

---

## Project: a resource library with a deep-copy buffer and a file wrapper

The project is a small library with two resource types. `IntBuffer` owns a heap array and copies it deeply. `FileHandle` owns a C file and closes it in its destructor. The demo prints the lifetime events, the test checks them, and the WSL build runs AddressSanitizer on both.

Suggested repo name: `100-days-software-day-32-memory-and-lifetime`.

The repo layout:

```
100-days-software-day-32-memory-and-lifetime/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── lifetime-rules.md
├── src/
│   ├── file_handle.cpp
│   ├── file_handle.hpp
│   ├── int_buffer.cpp
│   ├── int_buffer.hpp
│   └── main.cpp
└── tests/
    └── resource_test.cpp
```

The buffer header declares the class. It tracks how many buffers are alive, which lets the test check that destructors run:

```cpp
// src/int_buffer.hpp
#ifndef INT_BUFFER_HPP
#define INT_BUFFER_HPP

#include <cstddef>

class IntBuffer {
public:
    explicit IntBuffer(std::size_t size);
    IntBuffer(const IntBuffer& other);
    IntBuffer& operator=(const IntBuffer& other);
    ~IntBuffer();

    std::size_t size() const { return size_; }
    int& at(std::size_t index);
    int at(std::size_t index) const;

    static int live_count();

private:
    int* data_;
    std::size_t size_;
    static int live_;
};

#endif
```

The implementation allocates in the constructors, copies deeply, and frees in the destructor. The copy assignment allocates the new memory before it frees the old, so a failed allocation leaves the object unchanged:

```cpp
// src/int_buffer.cpp
#include "int_buffer.hpp"

#include <algorithm>
#include <stdexcept>

int IntBuffer::live_ = 0;

IntBuffer::IntBuffer(std::size_t size) : data_(new int[size]{}), size_(size) {
    ++live_;
}

IntBuffer::IntBuffer(const IntBuffer& other)
    : data_(new int[other.size_]), size_(other.size_) {
    std::copy(other.data_, other.data_ + other.size_, data_);
    ++live_;
}

IntBuffer& IntBuffer::operator=(const IntBuffer& other) {
    if (this != &other) {
        int* fresh = new int[other.size_];
        std::copy(other.data_, other.data_ + other.size_, fresh);
        delete[] data_;
        data_ = fresh;
        size_ = other.size_;
    }
    return *this;
}

IntBuffer::~IntBuffer() {
    delete[] data_;
    --live_;
}

int& IntBuffer::at(std::size_t index) {
    if (index >= size_) {
        throw std::out_of_range("IntBuffer index out of range");
    }
    return data_[index];
}

int IntBuffer::at(std::size_t index) const {
    if (index >= size_) {
        throw std::out_of_range("IntBuffer index out of range");
    }
    return data_[index];
}

int IntBuffer::live_count() {
    return live_;
}
```

The file wrapper owns a `std::FILE*`. It cannot be copied, because two owners would close the same file:

```cpp
// src/file_handle.hpp
#ifndef FILE_HANDLE_HPP
#define FILE_HANDLE_HPP

#include <cstdio>
#include <string>

class FileHandle {
public:
    FileHandle(const char* path, const char* mode);
    ~FileHandle();

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    void write_line(const std::string& line);

private:
    std::FILE* file_;
};

#endif
```

```cpp
// src/file_handle.cpp
#include "file_handle.hpp"

#include <stdexcept>
#include <string>

FileHandle::FileHandle(const char* path, const char* mode)
    : file_(std::fopen(path, mode)) {
    if (file_ == nullptr) {
        throw std::runtime_error(std::string("cannot open ") + path);
    }
}

FileHandle::~FileHandle() {
    if (file_ != nullptr) {
        std::fclose(file_);
    }
}

void FileHandle::write_line(const std::string& line) {
    (void)std::fputs(line.c_str(), file_);
    (void)std::fputc('\n', file_);
}
```

The demo prints the counts at each scope, then writes a file and reads it back:

```cpp
// src/main.cpp
#include <fstream>
#include <iostream>
#include <string>

#include "file_handle.hpp"
#include "int_buffer.hpp"

int main() {
    {
        IntBuffer a(3);
        a.at(0) = 7;
        {
            IntBuffer b = a;  // deep copy
            b.at(0) = 99;
            std::cout << "inside: a=" << a.at(0) << " b=" << b.at(0)
                      << " live=" << IntBuffer::live_count() << '\n';
        }
        std::cout << "after inner scope: live=" << IntBuffer::live_count() << '\n';
    }
    std::cout << "after outer scope: live=" << IntBuffer::live_count() << '\n';

    {
        FileHandle out("lifetime-demo.txt", "w");
        out.write_line("first line");
        out.write_line("second line");
    }  // the destructor closes the file here

    std::ifstream in("lifetime-demo.txt");
    std::string line;
    while (std::getline(in, line)) {
        std::cout << "read back: " << line << '\n';
    }
    return 0;
}
```

The test file checks the deep copy, the counts, the bounds, the file write and the failed open:

```cpp
// tests/resource_test.cpp
#include <cstdio>
#include <cstdlib>
#include <fstream>
#include <iostream>
#include <stdexcept>
#include <string>

#include "file_handle.hpp"
#include "int_buffer.hpp"

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
    const int before = IntBuffer::live_count();

    {
        IntBuffer original(2);
        original.at(1) = 5;
        IntBuffer copy = original;
        copy.at(1) = 6;

        check(original.at(1) == 5, "copy constructor makes a deep copy");
        check(copy.at(1) == 6, "the copy has its own value");
        check(IntBuffer::live_count() == before + 2, "two buffers alive in scope");

        IntBuffer target(1);
        target = original;
        check(target.size() == 2 && target.at(1) == 5, "copy assignment copies size and values");
    }
    check(IntBuffer::live_count() == before, "destructors release every buffer");

    bool index_threw = false;
    try {
        IntBuffer small(2);
        (void)small.at(2);
    } catch (const std::out_of_range&) {
        index_threw = true;
    }
    check(index_threw, "index past the end throws");

    const char* path = "resource-test.txt";
    {
        FileHandle file(path, "w");
        file.write_line("hello from RAII");
    }
    {
        std::ifstream in(path);
        std::string line;
        std::getline(in, line);
        check(line == "hello from RAII", "file handle writes and closes");
    }
    std::remove(path);

    bool open_threw = false;
    try {
        FileHandle missing("no-such-directory/none.txt", "r");
    } catch (const std::runtime_error&) {
        open_threw = true;
    }
    check(open_threw, "a missing file throws from the constructor");

    if (failures == 0) {
        std::cout << "all resource tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the library, the demo and the test. The `ENABLE_ASAN` option turns on AddressSanitizer for the WSL build. The definition for MSVC stops the C runtime from rejecting `fopen`:

```cmake
cmake_minimum_required(VERSION 3.25)
project(resource_lab LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(ENABLE_ASAN "Build with AddressSanitizer" OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)
add_compile_definitions($<$<CXX_COMPILER_ID:MSVC>:_CRT_SECURE_NO_WARNINGS>)

if(ENABLE_ASAN AND NOT MSVC)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=address)
endif()

add_library(resource_lab STATIC src/int_buffer.cpp src/file_handle.cpp)
target_include_directories(resource_lab PUBLIC src)

add_executable(resource_demo src/main.cpp)
target_link_libraries(resource_demo PRIVATE resource_lab)

add_executable(resource_test tests/resource_test.cpp)
target_link_libraries(resource_test PRIVATE resource_lab)

enable_testing()
add_test(NAME resource_test COMMAND resource_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/`, `build-asan/` and `*.txt` output files from the demo and the test to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add resource library with RAII types".
2. On Windows, run `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug`, then `cmake --build build`, then `ctest --test-dir build --output-on-failure`. Confirm the test passes.
3. Run `./build/resource_demo` and check that the inside line shows `a=7 b=99 live=2`, the inner scope shows `live=1`, and the outer scope shows `live=0`.
4. In WSL, build with `cmake -S . -B build-asan -G Ninja -DENABLE_ASAN=ON -DCMAKE_BUILD_TYPE=Debug` and run `ctest --test-dir build-asan --output-on-failure`. Confirm there are no sanitizer reports and no leaks.
5. Create a branch `feature/buffer-fill`. Write a test first for `void fill(IntBuffer& buffer, int value)` that sets every element to the value. Then implement the function and make the test pass. Run the WSL sanitizer build again.
6. Write `docs/lifetime-rules.md` with three short sections: which object owns each allocation in the library, why the copy assignment allocates before it deletes, and why `FileHandle` cannot be copied. Use one line from your demo output in the first section.
7. Merge with `git merge --no-ff feature/buffer-fill -m "Add buffer fill"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- The Windows test passes and the WSL test with AddressSanitizer reports no errors or leaks.
- The demo output shows the three live counts from task 3.
- `docs/lifetime-rules.md` names the owner of each resource.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the library, demo and test files, `docs/lifetime-rules.md`, `README.md`, and the history on GitHub.

This project is short but covers the rule that matters most in native code: every resource has one owner, and the owner's destructor releases it. A reviewer should see that the copy operations are either deep or deleted, and never implicit.

---

## Check your understanding

1. A function returns a pointer to an object it created with `new`. The caller forgets to delete it. Name the bug, say when the memory is released, and name one way to prevent it.
2. Two locals `a` and `b` are created in that order, and then the function returns. Predict the destruction order and explain why.
3. A class owns an `int*` and has a destructor that calls `delete`, but no copy constructor. Copy an object and describe what goes wrong when both copies are destroyed.
4. A function opens a file, calls a function that throws, and never closes the file. Explain why an RAII wrapper fixes this without adding any cleanup code to the function.
5. You see `delete` on memory allocated by `new int[10]`. Describe the undefined behavior, and why it may not crash in a quick test.
6. Why does the copy assignment in `IntBuffer` allocate the new memory before deleting the old memory?

### Answers

1. The bug is a leak, because no code can reach the memory after the pointer is lost, and the memory is never released. It stays allocated until the process ends. Prevent it by owning the memory with a RAII type, such as `std::unique_ptr` from Day 37, or by returning a value instead of a pointer.
2. `b` is destroyed first, then `a`, because objects are destroyed in the reverse order of construction. The last object created is the first to die, as with a stack of plates.
3. Both copies point at the same memory. The first destructor frees it, and the second destructor frees it again. That is a double free, which is undefined behavior and often corrupts the heap. The fix is a deep copy or a deleted copy constructor.
4. The wrapper's destructor runs during stack unwinding when the exception leaves the function, so the file is closed on every exit path. The function does not need a try block or a cleanup line.
5. Using `delete` instead of `delete[]` is undefined behavior, because the runtime stores the array length separately and `delete` does not read it. Some runtimes happen to free the block anyway, so a quick test can pass, and that still does not make the code correct. AddressSanitizer reports the mismatch.
6. If the allocation fails and throws, the old memory is still held, so the object remains valid and unchanged. If the old memory were deleted first, a failed allocation would leave the object pointing at freed memory.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Stack-based memory allocation](https://en.wikipedia.org/wiki/Stack-based_memory_allocation) for the stack in general
- [Wikipedia: Memory leak](https://en.wikipedia.org/wiki/Memory_leak) and [Wikipedia: Resource acquisition is initialization](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) for the two core ideas of this day
- cppreference.com, the pages for "Storage duration", "Object lifetime", "new expression" and "delete expression"
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on resource management, including rules R.1 (manage resources automatically with RAII) and R.3 (a raw pointer is non-owning)
- AddressSanitizer documentation at the GCC manual, section "Instrumentation Options," for the leak and use-after-free reports used in the exercises

*Tutorials*

- learncpp.com, the chapters on dynamic memory allocation, object lifetime and destructors, written by Alex Allain
- GeeksforGeeks, search the site for "new and delete in C++" and "RAII in C++" for the matching articles
- Microsoft Learn, search learn.microsoft.com for "C++ memory management" for the Windows view of the heap

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on the stack, the heap and destructors
- CppCon, the YouTube channel, for talks on RAII. Search the channel for "RAII" to find them.

*Practice*

- Compiler Explorer at godbolt.org. Paste the `Noisy` class and compare the compiled code with and without the destructor. The comparison shows where the compiler inserts the destructor calls.

---

## Tomorrow

Day 33 covers classes and objects. You have written classes in this day, with constructors, destructors and members. Day 33 explains access control with `public` and `private`, const member functions, initializer lists, and encapsulation as a way to protect the invariants that the buffer relies on. The lifetime rules from today decide when those invariants begin and end.

**Next:** [Day 33: Classes and Objects](Software-Day-33-Classes-and-Objects.md)

# Day 30: Arrays, Strings and Containers

*100 Days of Software Engineering, Module 4: C++ (Deep Foundation)*

**Time:** about 4 hours
**Level:** beginner to intermediate. You can write functions and loops from Days 28 and 29, and you have seen a `for` loop over a fixed set of values. This day moves from single values to collections of them. It is for anyone who has tried to store a list of words or scores and found that the plain C++ tools feel awkward.

**By the end of this day, you will have:**

- Used a built-in array, `std::array`, `std::vector` and `std::string`, and explained when each one fits
- Walked a collection with iterators and range-based `for`, and explained what makes an iterator invalid
- Used `std::sort`, `std::find` and `std::count_if` from `<algorithm>` on real data
- A word-frequency command-line tool that reads a text file, counts words and prints the most common ones, with a test executable run by CTest
- A GitHub repo with the test output, a sample run on a real text file and a short note on the container choices

**What you need:**

- GCC 14 or newer (checked with g++ 16.1.0), CMake 3.25 or newer (checked with CMake 4.3.3), and Ninja 1.11 or newer
- Git 2.40 or newer
- A terminal, either Git Bash or PowerShell
- Any plain-text file of at least a few hundred words, such as a README from one of your earlier repos or a public-domain book from Project Gutenberg (gutenberg.org)
- A free GitHub account

**Prerequisites:**

- Days 27 to 29: the build steps, functions, overloads and the difference between passing by value and by reference
- Day 28: `std::cin`, `std::cout` and integer types

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Core concepts](#core-concepts)
- [Arrays](#arrays)
- [Strings](#strings)
- [Vectors](#vectors)
- [Fixed-size arrays](#fixed-size-arrays)
- [Iterators](#iterators)
- [Algorithms](#algorithms)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a word-frequency counter with tests](#project-a-word-frequency-counter-with-tests)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

A program that handles one score is simple. A program that reads a book, a log file or a list of orders has to hold many values, find some of them, and sort or count them. Every real program does this, and the tools C++ offers for it are the containers and the algorithms that work on them. Choosing the right one is most of the skill, and choosing the wrong one usually shows up as slow code or memory bugs.

The standard library gives you a family of containers with a shared design. Each one stores elements in a specific way, and each one has rules about when its elements stay valid. This day explains the containers you will use most, the iterator idea that connects them to algorithms, and the mistakes that come from misreading those rules. The project counts words in a text file, which needs a string, a dynamic list, sorting and counting, all in one small tool.

The container and algorithm rules are part of the ISO C++ standard library. The reference pages on cppreference.com describe the exact guarantees for each container, so you should keep one open while you work.

---

## Glossary

- **Array:** a fixed number of values of one type stored side by side in memory, written as `int scores[5]`.
- **Container:** an object that stores a collection of values and manages their memory, such as `std::vector` or `std::map`.
- **String:** a sequence of characters. In C++, `std::string` manages its own memory and grows as needed.
- **Vector:** a dynamic array, `std::vector`, that grows when you add elements.
- **Capacity:** the number of elements a vector can hold before it must allocate more memory.
- **Iterator:** an object that points at an element in a container and moves through the elements. It works like a generalized pointer.
- **Algorithm:** a function from `<algorithm>` that works on any range described by iterators, such as `std::sort`.
- **Range:** a pair of iterators, a beginning and an end, that describes a sequence of elements.
- **Invalidation:** the state of an iterator or reference that no longer points at a valid element, usually after the container changes its memory.
- **Sentinel:** the value of `end()`, which points one position past the last element and must never be dereferenced.
- **Tokenizing:** splitting text into pieces such as words, usually by whitespace or punctuation.

---

## Core concepts

The concepts below move from the simplest collection to the most flexible one, then to the tools that work on any collection. Each section shows the same question: how do I store these values, and how do I reach each one?

### Arrays

A built-in array is a row of boxes with fixed labels. Think of an egg carton with exactly twelve cups. You know where each egg goes, and you cannot add a thirteenth cup without buying a new carton. In C++, a built-in array has a fixed size set at compile time, and its elements sit next to each other in memory.

Here is an array that holds five scores and totals them:

```cpp
#include <iostream>

int main() {
    int scores[5] = {90, 85, 77, 64, 100};
    int total = 0;

    for (int i = 0; i < 5; ++i) {
        total += scores[i];
    }

    std::cout << "total: " << total << '\n';
    std::cout << "third score: " << scores[2] << '\n';
    return 0;
}
```

The output is `total: 416` and `third score: 77`. Indexes start at 0, so `scores[2]` is the third element. The loop uses the literal `5` twice, once in the declaration and once in the condition. If you change one and forget the other, the program reads past the end, which is undefined behavior. C++ does not check array bounds for you.

A second trap is passing an array to a function. The array quietly becomes a pointer to its first element, and the function loses the size. The safer habit is to pass a container that knows its size, which the next sections introduce. Built-in arrays still matter because the C++ ABI and many C libraries use them directly.

The common mistake is an off-by-one index, such as `scores[5]` on a five-element array. The compiler does not catch it, and the program may print a random value or crash later. Sanitizers catch it on Day 38. In the market, buffer overruns on built-in arrays are the root of many security advisories, which is why modern code prefers the containers below.

### Strings

A `std::string` is a text buffer that manages its own memory. Think of a notepad that grows its page when you write past the end, and copies its contents to a larger page when needed. You do not have to count characters in advance or free the memory yourself.

```cpp
#include <iostream>
#include <string>

int main() {
    std::string first = "Ada";
    std::string last = "Lovelace";
    std::string full = first + " " + last;

    std::cout << full << " has " << full.size() << " characters\n";
    std::cout << "first letter: " << full[0] << '\n';
    return 0;
}
```

The output is `Ada Lovelace has 12 characters` and `first letter: A`. The `+` operator joins strings, and `size()` returns the number of characters. `full[0]` is an unchecked index, so `full.at(0)` is the safer choice when the index comes from user input, because `at` throws `std::out_of_range` if the index is wrong.

Strings also interact with the `const` and reference rules from Day 29. Pass a `std::string` into a function as `const std::string&` to avoid copying it. A `std::string_view`, which is C++17, is a non-owning view over characters and avoids copies even more. The project uses `std::string_view` for input text, so the function does not own or copy the file contents.

The common mistake is to mix `char*` strings with `std::string` and to forget that a `char*` pointing at a temporary string becomes a dangling pointer when the temporary is destroyed. The fix is to store a `std::string`. You can spot the bug when a pointer to text works in one run and crashes in another. In the market, string handling bugs show up in web servers and logging code, and teams prefer owning types so that lifetimes are obvious.

### Vectors

`std::vector` is the default container for a list of values. It is a dynamic array: it stores elements next to each other, like a built-in array, but it grows when you call `push_back`. Think of a train that adds a car when it needs one. When the train has no room left, the vector moves all its cars to a longer track, which takes time.

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> scores{90, 85, 77};
    scores.push_back(64);
    scores.push_back(100);

    int total = 0;
    for (int s : scores) {
        total += s;
    }

    std::cout << "count: " << scores.size() << '\n';
    std::cout << "average: " << total / static_cast<int>(scores.size()) << '\n';
    return 0;
}
```

The output is `count: 5` and `average: 83`. The integer division truncates `83.2` to `83`, which is the Day 28 lesson again. The `static_cast` is needed because `size()` returns `std::size_t`, an unsigned type, and mixing it with `int` triggers a sign warning.

A vector has a capacity that grows in steps, often doubling. If you know the final size, call `reserve` first to avoid repeated moves. When a vector grows, it relocates its elements to the new memory, which costs time in proportion to how many there are. Moves are cheaper than copies for large objects, and Day 37 explains when a type can be moved.

The common mistake is to hold a reference or iterator to an element and then call `push_back`. If the vector reallocates, the reference and iterator now point at freed memory. The fix is to finish using the reference before the change, or to store an index instead. You can spot the bug when a value changes in a way the code does not explain. In the market, vectors are the default container in game engines, simulation code and data pipelines, where this invalidation rule is a common source of crashes.

### Fixed-size arrays

`std::array` is a built-in array wrapped in a class. It has a fixed size known at compile time, like an egg carton, but it also knows its own size, copies like any other value, and works with the standard algorithms. Use it when the number of elements never changes and the set is known when you write the code.

```cpp
#include <array>
#include <iostream>

int main() {
    constexpr std::array<const char*, 3> days{"Mon", "Tue", "Wed"};

    for (const char* day : days) {
        std::cout << day << '\n';
    }
    std::cout << "size is " << days.size() << '\n';
    return 0;
}
```

The output lists the three days and then `size is 3`. The `constexpr` keyword means the array is built at compile time. The size is part of the type, so `std::array<int, 3>` and `std::array<int, 4>` are different types, which lets the compiler catch mismatches at build time.

The limitation is that `std::array` cannot grow or shrink. For a list of unknown length, use `std::vector`. For a fixed-size buffer on a small embedded device, `std::array` is the usual choice because it never allocates memory.

The common mistake is to choose `std::array` for data that changes length, which forces you to track the count yourself. The fix is to switch to a vector. In the market, `std::array` appears in performance-sensitive code, such as audio buffers, graphics math and robotics sensor frames, where a fixed size avoids allocations.

### Iterators

An iterator is a position inside a container. Think of a bookmark in a book. You can move it forward, look at the page it marks, and check whether you have reached the end. Every container in the standard library gives you iterators with the same basic operations, which is what lets one algorithm work on many containers.

Each container has `begin()`, which points at the first element, and `end()`, which points one position past the last element. The `end()` position is a sentinel. It marks the end of the range, and you must never dereference it. A range is the pair `[begin, end)`, and every algorithm works on that half-open range.

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<int> scores{90, 85, 77};

    for (auto it = scores.begin(); it != scores.end(); ++it) {
        std::cout << *it << ' ';
    }
    std::cout << '\n';

    for (int s : scores) {
        std::cout << s << ' ';
    }
    std::cout << '\n';
    return 0;
}
```

Both loops print `90 85 77` on one line each. The first loop uses an explicit iterator, and `*it` reads the element it points at. The second is a range-based `for` loop, which is shorter and is the form you should prefer unless you need the iterator itself. Range-based `for` uses iterators behind the scenes.

Iterators can become invalid. Adding an element to a vector may reallocate its memory, which invalidates every iterator and reference into it. Erasing an element invalidates iterators at or after the point of erasure. Different containers give different guarantees, which is why the reference page for each container matters. A `std::list` keeps its iterators valid across most changes, but at the cost of slower access by position.

The common mistake is to keep an iterator across an insertion or erase and then dereference it. The fix is to use the iterator returned by the call, such as the result of `erase`, or to recompute it after the change. You can spot the bug when a loop works with small vectors and crashes with large ones, because the larger one reallocates. In the market, this is one of the most common bugs in C++ code review.

### Algorithms

Algorithms are functions that work on any range described by two iterators. Think of a sorting machine on a conveyor belt. It only needs each box to have a position and to be comparable, whatever the box holds. The standard algorithms in `<algorithm>` follow the same idea.

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> scores{64, 100, 77, 90, 85};

    std::sort(scores.begin(), scores.end());

    auto found = std::find(scores.begin(), scores.end(), 90);
    if (found != scores.end()) {
        std::cout << "found 90 at index " << (found - scores.begin()) << '\n';
    }

    const auto passing = std::count_if(scores.begin(), scores.end(),
                                       [](int s) { return s >= 70; });
    std::cout << "passing: " << passing << '\n';
    return 0;
}
```

The output is `found 90 at index 3` and `passing: 4`. After sorting, the values are `64 77 85 90 100`, so 90 sits at index 3. The lambda `[](int s) { return s >= 70; }` is an anonymous function, which Day 36 covers in depth. `std::find` and `std::count_if` return an iterator or a count, and the `end()` check tells you whether the search succeeded.

Algorithms do not change the container's size. `std::sort` reorders elements, and `std::remove` moves elements to the end without shrinking the vector. To actually remove them, combine `std::remove` with `erase`, which is the erase-remove idiom. C++20 offers `std::erase_if` as a shorter form.

The common mistake is to call `std::sort` on a `std::map` or `std::set`, which does not compile, because `std::sort` needs random-access iterators and those two containers only provide bidirectional ones. Those containers already keep their elements in key order. Another is to assume that `std::remove` shrinks the vector. The fix is to read the algorithm's documentation on cppreference.com before you use it. You can spot the bug when the vector length is the same after a removal. In the market, the standard algorithms replace hand-written loops in most mature codebases, because they are tested and expressive.

---

## Hands-on exercises

Work in a new folder called `arrays-strings-containers`. Run `git init -b main` in it.

Exercise 1 uses a built-in array.

1. Declare an array of six temperatures and print the largest value by looping over the array with a `for` loop.
2. Print the array size as `sizeof(temps) / sizeof(temps[0])` and compare it with the number of elements you wrote.

Done when the largest value matches what you expect and the size prints as 6.

Exercise 2 uses `std::string`.

1. Read a full name with `std::getline` into a `std::string`. Print its length and its first and last characters.
2. Use `at(index)` with an index of 100 and catch the `std::out_of_range` exception. Print the message from `what()`.

Done when the program prints the length and catches the exception without crashing.

Exercise 3 uses `std::vector`.

1. Create an empty `std::vector<int>` and push the numbers 1 to 10 into it with a loop.
2. Print `size()` and `capacity()` after the loop. Then call `reserve(100)` on a new vector and push the same numbers. Compare the capacity values.

Done when you can explain in one sentence why `reserve` helps.

Exercise 4 uses algorithms.

1. Create a vector of ten integers that includes duplicates. Sort it with `std::sort`.
2. Remove duplicates with `std::unique` and `erase`, the erase-remove idiom. Print the result.
3. Count how many values are even with `std::count_if`.

Done when the duplicates are gone and the even count is correct for your data.

Exercise 5 demonstrates iterator invalidation on purpose.

1. Create a vector with three elements and store an iterator to the first element.
2. Push 20 values into the vector. Do not dereference the old iterator yet.
3. Write a note on whether the iterator is still valid and why. Then compile with `-fsanitize=address` in WSL, if you have it set up, and read the report if the old iterator is used.

Done when your notes explain why the vector reallocates and what the old iterator might point to.

---

## Project: a word-frequency counter with tests

The project reads a text file, splits it into words, counts each word and prints the total, the number of unique words and the five most common words. It uses a string, a vector, sorting and counting, and it puts each step in its own function so the test can check the rules.

Suggested repo name: `100-days-software-day-30-arrays-strings-containers`.

The repo layout:

```
100-days-software-day-30-arrays-strings-containers/
├── .gitignore
├── CMakeLists.txt
├── README.md
├── docs/
│   └── container-choices.md
├── src/
│   ├── main.cpp
│   ├── text_stats.cpp
│   └── text_stats.hpp
└── tests/
    └── text_stats_test.cpp
```

The header declares the structure and functions. The `WordCount` struct holds one word and how often it appears:

```cpp
// src/text_stats.hpp
#ifndef TEXT_STATS_HPP
#define TEXT_STATS_HPP

#include <cstddef>
#include <string>
#include <string_view>
#include <vector>

struct WordCount {
    std::string word;
    std::size_t count;
};

std::string normalize_word(std::string_view raw);
std::vector<std::string> tokenize(std::string_view text);
std::vector<WordCount> count_words(std::vector<std::string> words);
std::vector<WordCount> top_n(const std::vector<WordCount>& counts, std::size_t n);

#endif
```

The implementation uses `std::string_view` for input, so the functions never copy the whole file. It lowercases each word and keeps apostrophes so that contractions survive:

```cpp
// src/text_stats.cpp
#include "text_stats.hpp"

#include <algorithm>
#include <cctype>
#include <cstddef>
#include <iterator>
#include <string>
#include <string_view>
#include <utility>
#include <vector>

std::string normalize_word(std::string_view raw) {
    std::string result;
    result.reserve(raw.size());
    for (char c : raw) {
        const auto uc = static_cast<unsigned char>(c);
        if (std::isalnum(uc) || c == '\'') {
            result.push_back(static_cast<char>(std::tolower(uc)));
        }
    }
    return result;
}

std::vector<std::string> tokenize(std::string_view text) {
    std::vector<std::string> words;
    std::size_t start = 0;

    while (start < text.size()) {
        while (start < text.size() && std::isspace(static_cast<unsigned char>(text[start]))) {
            ++start;
        }
        std::size_t end = start;
        while (end < text.size() && !std::isspace(static_cast<unsigned char>(text[end]))) {
            ++end;
        }
        if (end > start) {
            std::string word = normalize_word(text.substr(start, end - start));
            if (!word.empty()) {
                words.push_back(std::move(word));
            }
        }
        start = end;
    }
    return words;
}

std::vector<WordCount> count_words(std::vector<std::string> words) {
    std::sort(words.begin(), words.end());

    std::vector<WordCount> counts;
    for (auto it = words.begin(); it != words.end();) {
        const std::string current = *it;
        const auto next = std::find_if(it, words.end(),
                                       [&current](const std::string& w) { return w != current; });
        counts.push_back(WordCount{current, static_cast<std::size_t>(std::distance(it, next))});
        it = next;
    }
    return counts;
}

std::vector<WordCount> top_n(const std::vector<WordCount>& counts, std::size_t n) {
    std::vector<WordCount> sorted = counts;
    std::sort(sorted.begin(), sorted.end(),
              [](const WordCount& a, const WordCount& b) {
                  if (a.count != b.count) {
                      return a.count > b.count;
                  }
                  return a.word < b.word;
              });
    if (sorted.size() > n) {
        sorted.resize(n);
    }
    return sorted;
}
```

The `main.cpp` file reads the file named on the command line, prints the summary and returns a non-zero code on any error:

```cpp
// src/main.cpp
#include <cstddef>
#include <fstream>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

#include "text_stats.hpp"

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cerr << "usage: wordstats <file.txt>\n";
        return 1;
    }

    std::ifstream in(argv[1]);
    if (!in) {
        std::cerr << "error: cannot open " << argv[1] << '\n';
        return 1;
    }

    std::stringstream buffer;
    buffer << in.rdbuf();
    const std::string text = buffer.str();

    const std::vector<std::string> words = tokenize(text);
    const std::vector<WordCount> counts = count_words(words);
    const std::vector<WordCount> best = top_n(counts, 5);

    std::cout << "total words:  " << words.size() << '\n';
    std::cout << "unique words: " << counts.size() << '\n';
    std::cout << "top words:\n";
    for (const WordCount& entry : best) {
        std::cout << "  " << entry.word << ": " << entry.count << '\n';
    }
    return 0;
}
```

The test file checks the normalizing, the tokenizing, the counts and the tie-breaking rule:

```cpp
// tests/text_stats_test.cpp
#include <cstdlib>
#include <iostream>
#include <string>
#include <vector>

#include "text_stats.hpp"

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
    check(normalize_word("Hello,") == "hello", "strip punctuation and lowercase");
    check(normalize_word("don't") == "don't", "keep apostrophes");
    check(normalize_word("...").empty(), "punctuation only becomes empty");

    const std::vector<std::string> words = tokenize("The cat, the  hat.\n\tTHE end");
    check(words.size() == 6, "six tokens from the sample");

    const std::vector<WordCount> counts = count_words(words);
    check(counts.size() == 4, "four unique words");

    const std::vector<WordCount> best = top_n(counts, 1);
    check(best.size() == 1 && best[0].word == "the" && best[0].count == 3,
          "the is the most common word");

    const std::vector<WordCount> top3 = top_n(counts, 3);
    check(top3.size() == 3 && top3[1].word == "cat" && top3[2].word == "end",
          "ties are broken alphabetically");

    check(tokenize("").empty(), "empty input gives no words");

    if (failures == 0) {
        std::cout << "all text stats tests passed\n";
        return EXIT_SUCCESS;
    }
    return EXIT_FAILURE;
}
```

The `CMakeLists.txt` builds the tool and the test from the same source file, and registers the test with CTest:

```cmake
cmake_minimum_required(VERSION 3.25)
project(word_frequency LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    "$<$<CXX_COMPILER_ID:MSVC>:/W4;/WX>"
    "$<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall;-Wextra;-Wpedantic;-Werror>"
)

add_library(text_stats STATIC src/text_stats.cpp)
target_include_directories(text_stats PUBLIC src)

add_executable(wordstats src/main.cpp)
target_link_libraries(wordstats PRIVATE text_stats)

add_executable(text_stats_test tests/text_stats_test.cpp)
target_link_libraries(text_stats_test PRIVATE text_stats)

enable_testing()
add_test(NAME text_stats_test COMMAND text_stats_test)
```

Step-by-step tasks:

1. Create the files above. Add `build/` to `.gitignore`. Initialize the repo with `git init -b main` and commit with the message "Add word frequency counter".
2. Build with `cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug` and `cmake --build build`. Fix any warnings before you continue.
3. Run `ctest --test-dir build --output-on-failure`. Confirm the test passes.
4. Download a plain-text file, such as a chapter of a public-domain book from Project Gutenberg, and save it as `sample.txt` in a folder outside the repo. Run the tool on it with `./build/wordstats path/to/sample.txt`. Record the total words, the unique words and the top five.
5. Create a branch `feature/ignore-short-words`. Write a test first that checks `top_n` ignores words shorter than three letters. Then add a `min_length` parameter, make the test pass, and keep the default value compatible with the existing tests.
6. Write `docs/container-choices.md` with three short sections: why `tokenize` returns a vector, why `top_n` takes a const reference, and what would change if the file were a gigabyte in size. Use one real number from your sample run in the first section.
7. Merge the branch with `git merge --no-ff feature/ignore-short-words -m "Add minimum word length"`, create an empty GitHub repository with the same name, and push with `git remote add origin <URL>` and `git push -u origin main`.

Done when:

- `ctest --test-dir build --output-on-failure` passes.
- The sample run prints a total, a unique count and five words, and you can check one of them by hand.
- `docs/container-choices.md` quotes one number from your sample run.
- The GitHub repo shows the merge commit on `main`.

Evidence to commit: `CMakeLists.txt`, the source and test files, `docs/container-choices.md`, `README.md`, and the history on GitHub. Do not commit `sample.txt` if it is copyrighted. Note its source in the README instead.

This project is light on new syntax and heavy on choices. The reader should see why each function returns the container it does, and why the test checks the tie-breaking rule that a naive version would get wrong.

---

## Check your understanding

1. A function receives a `std::vector<int>` by value, but it only reads the values. What does each call cost, and what is the fix?
2. You store `auto first = scores.begin();`, call `push_back` many times, then dereference `first`. Explain what can go wrong, and name two fixes.
3. Suppose a later edit removes the lowercasing step, and the counter now reports `the` and `The` as two different words. Which function is responsible, and which existing test should have caught it?
4. A teammate uses `std::remove` on a vector and then prints `size()`, which has not changed. Explain why, and show the complete call that actually removes the elements.
5. A data pipeline holds a fixed buffer of exactly 1024 sensor readings that is allocated once and never resized. Compare `std::vector` and `std::array` for this case and justify your choice.
6. Why does `top_n` take `const std::vector<WordCount>&` and return a new vector, instead of sorting the caller's data in place?

### Answers

1. Each call copies the whole vector, which costs time and memory proportional to its size. Pass `const std::vector<int>&` to read without copying.
2. The vector may reallocate on `push_back`, which frees the old memory. The iterator then points at freed memory, and dereferencing it is undefined behavior. Fix it by reserving capacity before the loop, or by storing an index and recomputing the iterator after the changes.
3. `normalize_word` is responsible, because it is the only place that changes case. The test "strip punctuation and lowercase" checks `normalize_word("Hello,") == "hello"`, so removing the `std::tolower` call makes that test fail.
4. `std::remove` moves the kept elements to the front and returns an iterator to the new logical end. It does not shrink the vector. The complete call is `scores.erase(std::remove(scores.begin(), scores.end(), value), scores.end());`.
5. Use `std::array<Reading, 1024>` because the size is known at compile time and never changes. It avoids heap allocation and makes the size part of the type. `std::vector` would work, but it adds an allocation and a capacity field that this case does not need.
6. The function does not change the caller's data, which keeps it safe to call from anywhere and matches the const reference rule from Day 29. Returning a new vector makes the result explicit, and the caller decides whether to keep the original order.

---

## Sources to read, watch and practice

*Read*

- [Wikipedia: Array data structure](https://en.wikipedia.org/wiki/Array_data_structure) for the general idea of arrays
- [Wikipedia: Dynamic array](https://en.wikipedia.org/wiki/Dynamic_array) for how a vector grows
- [Wikipedia: Iterator](https://en.wikipedia.org/wiki/Iterator) for the iterator concept across languages
- cppreference.com, the pages for `std::vector`, `std::array`, `std::string`, `std::string_view` and the `<algorithm>` header. Read the "Iterator invalidation" notes on the vector page.
- C++ Core Guidelines at isocpp.github.io/CppCoreGuidelines, the sections on containers and the rule against raw arrays

*Tutorials*

- learncpp.com, the chapters on arrays, vectors, strings and iterators, written by Alex Allain
- GeeksforGeeks, search the site for "C++ vector", "C++ string" and "C++ algorithm library" for the matching articles
- W3Schools, the C++ "Arrays" and "Strings" pages, for short examples

*Watch*

- The Cherno, the YouTube "C++ Series" (search the title on YouTube), for the episodes on arrays, vectors and iterators
- CppCon, the YouTube channel, for talks on containers and iterator invalidation. Search the channel for "iterator invalidation" to find them.

*Practice*

- Exercism, the C++ track at exercism.org/tracks/cpp, with the exercises on strings and vectors. Run the test suite for each one from your terminal.

---

## Tomorrow

Day 31 covers pointers and references in detail. You have used references since Day 29 and iterators today, and both are built on the same idea of pointing at memory. You will learn addresses, dereferencing, `nullptr`, pointer arithmetic over arrays, and the safety rules that keep a pointer from outliving the memory it points at. The invalidation bugs from today are the reason those rules matter.

**Next:** [Day 31: Pointers and References](Software-Day-31-Pointers-and-References.md)

# C++ Mastery: From Foundations to Expert — A Complete Revamp Course

## Course Overview

Welcome to **C++ Mastery**. This comprehensive course is designed specifically for experienced developers (6+ years) who want to revamp, solidify, and deepen their knowledge of C++. Whether you've been working in an older codebase and want to modernize your skills, or you're preparing for rigorous senior-level technical interviews, this course provides a deep dive into the language's mechanics, modern features, and best practices.

### What Makes This Course Unique?
- **Deep and Thorough**: We don't gloss over details. We explain the "why" behind memory layouts, virtual tables, template instantiations, and compiler optimizations.
- **Code-Heavy**: Every concept is paired with working, compilable C++ code.
- **Interview-Focused**: Tricky interview questions with detailed answers are included after each major section.
- **Modern**: Covers standard features from C++11 up through C++23.
- **Practical**: Real-world use cases, common pitfalls, and idiomatic best practices.

---

## Complete Table of Contents

### Module 01: C++ Fundamentals Refresher
*Types, expressions, control flow, functions, initialization rules, and compilation model.*
[View Module 01](./01-fundamentals-refresher.md)

### Module 02: Pointers, References & Memory Management
*Raw pointers, references, smart pointers (`std::unique_ptr`, `std::shared_ptr`), RAII, memory layout (stack vs. heap), and custom allocators.*
[View Module 02](./02-pointers-references-memory.md)

### Module 03: Object-Oriented Programming Deep Dive
*Classes, inheritance, polymorphism, virtual dispatch mechanisms (vtables), multiple inheritance, and object slicing.*
[View Module 03](./03-oop-deep-dive.md)

### Module 04: Templates & Generic Programming
*Function and class templates, SFINAE, `if constexpr`, C++20 Concepts, variadic templates, and template metaprogramming.*
[View Module 04](./04-templates-generic-programming.md)

### Module 05: The Standard Template Library
*Containers (vector, map, etc.), custom iterators, standard algorithms, `std::pmr`, `std::variant`, and C++20 Ranges.*
[View Module 05](./05-stl-mastery.md)

### Module 06: Modern C++ Features
*A dedicated tour of features introduced in C++11, C++14, C++17, C++20, and C++23 (e.g., structured bindings, modules, `constexpr`, designated initializers).*
[View Module 06](./06-modern-cpp-features.md)

### Module 07: Move Semantics & Perfect Forwarding
*Rvalue references, move constructors/assignments, `std::move`, `std::forward`, forwarding references, and copy elision.*
[View Module 07](./07-move-semantics-forwarding.md)

### Module 08: Multithreading & Concurrency
*Threads, mutexes, atomics, memory models, asynchronous tasks (`std::async`, futures), and C++20 Coroutines.*
[View Module 08](./08-multithreading-concurrency.md)

### Module 09: Lambda Expressions & Functional C++
*Closures, lambda captures, `std::function`, `std::invoke`, and functional programming idioms.*
[View Module 09](./09-lambdas-functional-cpp.md)

### Module 10: Error Handling & Exception Safety
*Exception mechanisms, exception guarantees (basic, strong, no-throw), `noexcept` specifier, and modern alternatives like `std::expected`.*
[View Module 10](./10-error-handling-exceptions.md)

### Module 11: Design Patterns & Best Practices
*SOLID principles in C++, GoF patterns modernized for C++ (e.g., Type Erasure, CRTP, Pimpl), and idiomatic C++.*
[View Module 11](./11-design-patterns-best-practices.md)

### Module 12: Performance, Optimization & Master Interview Bank
*Cache friendliness, profiling, compiler optimizations, zero-overhead abstractions, and a curated list of 110 senior interview Q&A.*
[View Module 12](./12-performance-interview-mastery.md)

### Module 13: Low-Latency, HFT & Ultra-High-Performance Systems
*Zero-allocation hot paths, Linux OS core isolation (`isolcpus`, `nohz_full`), Kernel Bypass (Solarflare EF_VI / DPDK), Lock-Free SPSC Queues, Limit Order Book (L2/L3) design, Fixed-Point arithmetic, and 50+ Quant/HFT interview Q&A.*
[View Module 13](./13-low-latency-hft-systems.md)

---

## How to Use This Course

### Prerequisites
- Solid understanding of general programming concepts.
- Prior experience with C++ or another systems programming language (C, Rust) is highly recommended.
- Familiarity with command-line tools and building software.

### Suggested Learning Path
1. **Sequential Deep Dive**: If you are revamping your entire C++ knowledge base, proceed sequentially from Module 01 to Module 12.
2. **Modernization Focus**: If your foundation is solid but you want to catch up on modern features, start with Module 06, then focus on Modules 07, 04 (Concepts), and 05 (Ranges).
3. **Interview Prep**: Review Modules 02, 03, 07, and 08 heavily, then spend time on Module 12.

> 💡 **Tip:** Don't just read the code! Copy the examples, compile them, modify them, and see how the compiler reacts.

---

## C++ Standards Timeline

C++ has evolved significantly from its early days. Understanding this timeline helps contextualize why certain features exist.

- **C++98 / C++03**: The original ISO standard. Introduced templates and the STL.
- **C++11**: The modern revolution. Introduced `auto`, lambdas, move semantics, smart pointers, and a standardized memory model.
- **C++14**: Incremental improvements, generic lambdas, relaxed `constexpr`.
- **C++17**: Structured bindings, `std::optional`/`std::variant`, fold expressions, filesystem API.
- **C++20**: A major release introducing Concepts, Ranges, Coroutines, and Modules.
- **C++23**: Library extensions, `std::expected`, `std::mdspan`, `deducing this`, and printing facilities.

---

## Compilation Guide

To get the most out of this course, you should compile the examples using a modern C++ compiler.

### GCC (g++)
Compile with C++20 support and standard warnings:
```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic -O2 example.cpp -o example
```
*For C++23, use `-std=c++2b` or `-std=c++23` depending on your GCC version.*

### Clang (clang++)
```bash
clang++ -std=c++20 -Wall -Wextra -Wpedantic -O2 example.cpp -o example
```

### Microsoft Visual C++ (MSVC)
Using the Developer Command Prompt:
```cmd
cl /std:c++20 /W4 /EHsc /O2 example.cpp
```
*For C++23 features in MSVC, use `/std:c++latest`.*

> ⚠️ **Warning:** Some C++20 (like Modules) and C++23 features may still be experimental or partially implemented depending on your compiler version. Always check your compiler's documentation for the current status.

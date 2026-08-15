<p align="center">
  <img src="https://img.shields.io/badge/C++-00599C?style=for-the-badge&logo=c%2B%2B&logoColor=white" alt="C++" />
  <img src="https://img.shields.io/badge/Standard-C++23-blue?style=for-the-badge" alt="C++23" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="MIT License" />
  <img src="https://img.shields.io/badge/Questions-300%2B-orange?style=for-the-badge" alt="300+ Questions" />
</p>

<h1 align="center">🚀 C++ Mastery: From Foundations to Expert</h1>

<p align="center">
  <b>A comprehensive, self-contained C++ course covering beginner to advanced topics — designed for experienced developers who want to revamp, deepen, and solidify their C++ knowledge.</b>
</p>

<p align="center">
  <i>12 in-depth modules • 300+ interview questions with answers • C++11 through C++23 • Hundreds of compilable code examples</i>
</p>

---

## 🎯 Who Is This For?

This course is specifically designed for:

- **Experienced C++ developers** (3–10+ years) looking to fill knowledge gaps and refresh fundamentals
- **Senior engineers preparing for interviews** at FAANG, quant firms, game studios, and systems companies
- **Mid-level developers** wanting to level up to senior/staff positions
- **Embedded/systems programmers** transitioning to modern C++ (C++17/20/23)
- **Anyone** who wants a single, comprehensive reference for all of C++ — from the basics to the deepest corners

> 💡 **Not a "Hello World" course.** This assumes you can write C++ already. We go deep — explaining the *why*, the *how*, the *gotchas*, and the *interview traps* behind every concept.

---

## 📚 Course Modules

| Module | Title | What You'll Master |
|:------:|-------|-------------------|
| **00** | [Course Overview](./00-course-overview.md) | Learning roadmap, C++ standards timeline, compilation guide |
| **01** | [Fundamentals Refresher](./01-fundamentals-refresher.md) | Type system, storage classes, casts, expressions, operators, ADL, namespaces, preprocessor |
| **02** | [Pointers, References & Memory](./02-pointers-references-memory.md) | Raw pointers, pointer arithmetic, function pointers, smart pointers (`unique_ptr`, `shared_ptr`, `weak_ptr`), RAII, memory layout |
| **03** | [OOP Deep Dive](./03-oop-deep-dive.md) | Classes, Rule of 0/3/5, operator overloading, `<=>`, inheritance, virtual dispatch, vtable internals, RTTI, SOLID |
| **04** | [Templates & Generic Programming](./04-templates-generic-programming.md) | Function/class templates, specialization, variadic templates, fold expressions, SFINAE, C++20 concepts, CRTP, metaprogramming |
| **05** | [STL Mastery](./05-stl-mastery.md) | All containers (internals & complexity), iterators, 60+ algorithms, C++20 ranges & views, `optional`/`variant`/`any` |
| **06** | [Modern C++ Feature Tour](./06-modern-cpp-features.md) | Every major feature from **C++11 → C++23**: structured bindings, `if constexpr`, coroutines, modules, `std::expected`, deducing `this` |
| **07** | [Move Semantics & Perfect Forwarding](./07-move-semantics-forwarding.md) | Value categories, `std::move`, `std::forward`, reference collapsing, copy elision (RVO/NRVO), common pitfalls |
| **08** | [Multithreading & Concurrency](./08-multithreading-concurrency.md) | `std::thread`, mutexes, condition variables, atomics, **C++ memory model**, async/futures, thread pools, **C++20 coroutines** |
| **09** | [Lambda & Functional C++](./09-lambdas-functional-cpp.md) | Capture mechanics, generic lambdas, `std::function`, higher-order functions, recursive lambdas, overloaded pattern |
| **10** | [Error Handling & Exception Safety](./10-error-handling-exceptions.md) | Exception guarantees, `noexcept`, scope guards, `std::expected` (C++23), error codes vs exceptions, defensive programming |
| **11** | [Design Patterns & Best Practices](./11-design-patterns-best-practices.md) | GoF patterns in modern C++, CRTP, pimpl, type erasure, NVI, copy-and-swap, Core Guidelines, anti-patterns |
| **12** | [Performance & Interview Mastery](./12-performance-interview-mastery.md) | CPU cache, SoA/AoS, compiler optimizations, profiling tools, **100+ curated interview questions** with detailed answers |

---

## 🧠 What Makes This Course Different?

<table>
<tr>
<td width="50%">

### 📖 Self-Contained Explanations
Every concept is explained from first principles with the *"why"* behind it. No hand-waving, no "left as an exercise."

### 💻 Compilable Code Examples
Hundreds of code snippets — all designed to be copy-pasted and compiled. Each example demonstrates a real concept, not toy code.

### ⚠️ Gotcha Callouts
Common pitfalls, undefined behavior traps, and subtle bugs are explicitly flagged with warnings and explanations.

</td>
<td width="50%">

### 🎯 Interview-Focused
200+ interview questions organized by topic and difficulty (Easy → Expert). Includes output prediction, design scenarios, and coding challenges.

### 🔬 Internals Explained
vtable layout, `shared_ptr` control blocks, `vector` growth strategy, hash table bucket mechanics, cache line effects — we show what's under the hood.

### 📐 Modern C++ First
Everything is taught with modern C++ idioms (C++17/20/23) while explaining legacy approaches for codebases that need them.

</td>
</tr>
</table>

---

## 🗺️ Suggested Learning Path

```
Week 1-2   ➤  Modules 00, 01, 02   (Foundations: types, pointers, memory)
Week 3-4   ➤  Modules 03, 04       (OOP & Templates)
Week 5-6   ➤  Modules 05, 06       (STL & Modern C++)
Week 7-8   ➤  Modules 07, 08       (Move Semantics & Multithreading)
Week 9-10  ➤  Modules 09, 10       (Lambdas & Error Handling)
Week 11-12 ➤  Modules 11, 12       (Design Patterns & Interview Prep)
```

> **For interview prep**: Prioritize Modules **02** (Pointers), **03** (OOP/vtable), **07** (Move Semantics), **08** (Multithreading), and **12** (Interview Bank).

---

## 🔴 Interview Priority Guide

| Priority | Topics | Module(s) |
|----------|--------|-----------|
| 🔴 **Critical** | Pointers & Smart Pointers | [02](./02-pointers-references-memory.md) |
| 🔴 **Critical** | Virtual Functions & vtable | [03](./03-oop-deep-dive.md) |
| 🔴 **Critical** | Move Semantics & `std::forward` | [07](./07-move-semantics-forwarding.md) |
| 🔴 **Critical** | Multithreading & Atomics | [08](./08-multithreading-concurrency.md) |
| 🟡 **Important** | Templates, SFINAE & Concepts | [04](./04-templates-generic-programming.md) |
| 🟡 **Important** | STL Containers & Algorithms | [05](./05-stl-mastery.md) |
| 🟡 **Important** | Design Patterns | [11](./11-design-patterns-best-practices.md) |
| 🟢 **Good to Know** | Modern C++ Features | [06](./06-modern-cpp-features.md) |
| 🟢 **Good to Know** | Lambdas & Functional C++ | [09](./09-lambdas-functional-cpp.md) |
| 🟢 **Good to Know** | Exception Safety | [10](./10-error-handling-exceptions.md) |

---

## 🛠️ How to Compile the Examples

All code examples target **C++17** minimum, with many requiring **C++20** or **C++23**.

```bash
# GCC (C++20)
g++ -std=c++20 -Wall -Wextra -pedantic -O2 example.cpp -o example -pthread

# Clang (C++20)
clang++ -std=c++20 -Wall -Wextra -pedantic -O2 example.cpp -o example -pthread

# MSVC (C++20)
cl /std:c++20 /EHsc /W4 example.cpp

# For C++23 features
g++ -std=c++2b ...
clang++ -std=c++2b ...
cl /std:c++latest ...
```

> 💡 Use [Compiler Explorer (godbolt.org)](https://godbolt.org) to try examples online without any local setup.

---

## 📊 Course Stats

| Metric | Value |
|--------|-------|
| Total Modules | 13 (including overview) |
| Content Size | ~350 KB of pure technical content |
| Interview Questions | **300+** (including 110-question master bank in Mod 12) |
| C++ Standards Covered | C++98 → C++23 |
| Code Examples | Hundreds of compilable snippets |
| Topics Covered | 60+ major C++ topics |

---

## 🤝 Contributing

Found an error? Have a better explanation? Want to add more interview questions?

1. Fork the repository
2. Create a feature branch (`git checkout -b fix/vtable-explanation`)
3. Commit your changes (`git commit -m 'Fix vtable diagram explanation'`)
4. Push to the branch (`git push origin fix/vtable-explanation`)
5. Open a Pull Request

---

## ⭐ Star This Repo

If you found this course helpful, please consider giving it a ⭐ — it helps others discover it!

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](./LICENSE) file for details.

---

<p align="center">
  <b>Built with ❤️ for the C++ community</b><br/>
  <i>Happy coding, and good luck with your interviews! 🎉</i>
</p>

# Module 01 — C++ Fundamentals Refresher

This module provides a comprehensive refresher on C++ fundamentals, specifically tailored for experienced engineers looking to deepen their understanding of the language's core mechanics, historical context, and modern idioms (C++11 through C++23).

---

## 1. Type System Deep Dive

C++ features a statically typed, nominally typed system with strong type checking (though C-style casts provide loopholes).

### Fundamental Types
C++ does not dictate the exact size of fundamental types, only their minimum sizes and relative sizes (e.g., `sizeof(char) <= sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long)`).

```cpp
#include <iostream>
#include <limits>

int main() {
    // Sizes are platform-dependent (LP64 vs LLP64)
    std::cout << "char: " << sizeof(char) << " bytes\n"; // Always 1
    std::cout << "int: " << sizeof(int) << " bytes\n";   // Usually 4
    std::cout << "double: " << sizeof(double) << " bytes\n"; // Usually 8
    
    // Ranges
    std::cout << "int max: " << std::numeric_limits<int>::max() << "\n";
}
```

### Fixed-Width Integers
Included via `<cstdint>`, these guarantee exact bit widths. Prefer these when hardware boundaries, network protocols, or binary file formats are involved.

```cpp
#include <cstdint>

int8_t   a = 10;   // Exactly 8 bits
int16_t  b = 20;   // Exactly 16 bits
int32_t  c = 30;   // Exactly 32 bits
int64_t  d = 40;   // Exactly 64 bits
uint32_t e = 50U;  // Unsigned 32-bit
```

> [!WARNING]
> `int8_t` and `uint8_t` are often typedefs for `signed char` and `unsigned char`. Printing them via `std::cout` will print characters, not integers! Cast to `int` or use `+` before printing: `std::cout << +my_uint8;`.

### Type Conversions (Casts)
Modern C++ eschews C-style casts `(Type)var` in favor of named casts for clarity and safety.

```cpp
class Base { virtual void dummy() {} };
class Derived : public Base { int a; };

void type_conversions() {
    // 1. implicit conversion
    int i = 42;
    double d = i;

    // 2. static_cast: Well-defined, compile-time conversions
    double pi = 3.14;
    int rounded = static_cast<int>(pi);

    // 3. dynamic_cast: Safe downcasting in polymorphic hierarchies (RTTI)
    Base* b = new Derived();
    Derived* dev = dynamic_cast<Derived*>(b); // null if invalid cast

    // 4. const_cast: Removes const/volatile qualifiers
    const int val = 10;
    int* ptr = const_cast<int*>(&val); // Modifying *ptr is UB if original var is const!

    // 5. reinterpret_cast: Low-level bit reinterpretation
    int hex = 0xDEADBEEF;
    float* fptr = reinterpret_cast<float*>(&hex); 
}
```

### Type Aliases
Always prefer `using` over `typedef`. It is more readable, especially for function pointers, and supports templates.

```cpp
// Old C-style
typedef void (*FuncPtr)(int, double);
typedef std::map<std::string, std::vector<int>> MyMap_t;

// Modern C++ (using)
using FuncPtrNew = void(*)(int, double);
using MyMapNew_t = std::map<std::string, std::vector<int>>;

// using supports templates (typedef does not!)
template <typename T>
using MyVector = std::vector<T>;
```

### auto and decltype
`auto` deduces types, dropping `const` and references by default (decay).
`decltype` yields the exact type, including references and qualifiers.

```cpp
const int cx = 42;
const int& rx = cx;

auto a = rx;       // int (const and reference stripped)
auto& b = rx;      // const int&
decltype(rx) c = cx; // const int&
```

### Type Traits Basics
`<type_traits>` allows introspecting and modifying types at compile-time.

```cpp
#include <type_traits>

static_assert(std::is_integral_v<int>);
static_assert(std::is_same_v<std::remove_const_t<const int>, int>);
```

---

## 2. Variables & Storage

### Storage Classes & Duration
Storage duration determines the lifetime of an object.
- **Automatic**: Created on block entry, destroyed on exit (default).
- **Static**: Initialized once, lasts for program lifetime.
- **Dynamic**: Managed manually via `new`/`delete` or smart pointers.
- **Thread**: Lasts for the duration of a thread (`thread_local`).

```cpp
void storage_example() {
    int auto_var = 1;                  // Automatic
    static int static_var = 1;         // Static, persists across calls
    thread_local int thread_var = 1;   // Thread-local, unique to each thread
}
```

### Linkage
- **Internal Linkage**: Visible only within the current translation unit (`static` global variables, anonymous namespaces).
- **External Linkage**: Visible across translation units (default for globals, `extern`).

### Initialization
C++ has a notoriously complex initialization syntax.

```cpp
int a;            // Default initialization (uninitialized if primitive local)
int b{};          // Value initialization (zero-initialized)
int c(10);        // Direct initialization
int d = 10;       // Copy initialization
int e{10};        // List initialization (prevents narrowing conversions!)

struct Aggregate { int x; double y; };
Aggregate agg{1, 2.0}; // Aggregate initialization
```

### Structured Bindings (C++17)
Allows unpacking tuples, pairs, and aggregates seamlessly.

```cpp
#include <tuple>
#include <map>

std::tuple<int, double, char> get_data() { return {1, 3.14, 'c'}; }

void bindings() {
    auto [i, d, c] = get_data(); // Structured binding

    std::map<int, std::string> m = {{1, "one"}, {2, "two"}};
    for (const auto& [key, val] : m) {
        // Iterate with structured bindings
    }
}
```

---

## 3. Expressions & Operators

### Operator Precedence & Associativity
Most operators group left-to-right, but Assignment, Unary, and Conditional (`?:`) group right-to-left.

> [!TIP]
> Do not memorize the whole table; use parentheses to make intent explicit. Know that `&&` binds tighter than `||`, and `==` binds tighter than `&`.

### Short-Circuit Evaluation
Logical `&&` and `||` short-circuit. If the left operand dictates the outcome, the right is not evaluated.

```cpp
bool ptr_check(int* p) {
    // If p is nullptr, p->value is never evaluated. Safe!
    return p != nullptr && p->value > 0;
}
```

### Sequence Points & Undefined Behavior
Modifying a variable multiple times without an intervening sequence point is Undefined Behavior (UB) prior to C++11, and still highly discouraged.

```cpp
int i = 0;
i = ++i; // UB in C++03, well-defined in C++11 but awful style.
// func(i++, i++) is UB because argument evaluation order is unspecified.
```

### Comma Operator Tricks
Evaluates left expression, discards result, evaluates right expression.

```cpp
int a = (1, 2, 3); // a is 3.

// Often used in for loops:
for(int i=0, j=10; i<j; ++i, --j) { /* ... */ }
```

### Bitwise Operations
Critical for embedded or high-performance systems.

```cpp
int set_bit(int val, int bit) { return val | (1 << bit); }
int clear_bit(int val, int bit) { return val & ~(1 << bit); }
int toggle_bit(int val, int bit) { return val ^ (1 << bit); }
bool check_bit(int val, int bit) { return (val & (1 << bit)) != 0; }
```

---

## 4. Control Flow

### `if constexpr` (C++17)
Discards false branches at compile time. Essential for template metaprogramming.

```cpp
template <typename T>
void print_type(T t) {
    if constexpr (std::is_integral_v<T>) {
        // This branch only compiles if T is an integer
        std::cout << "Integer: " << t << '\n';
    } else {
        std::cout << "Not an integer\n";
    }
}
```

### `switch` with `[[fallthrough]]`
C++ switch cases fall through by default. C++17 added `[[fallthrough]]` to suppress compiler warnings and signal intent.

```cpp
switch (val) {
    case 1:
        do_one();
        [[fallthrough]];
    case 2:
        do_two();
        break;
}
```

### Range-based `for` Loops
Iterate over any container providing `begin()` and `end()`.

```cpp
std::vector<int> v = {1, 2, 3};
for (auto& elem : v) { elem *= 2; }       // Modify by ref
for (const auto& elem : v) { /* Read */ } // Read by const ref
```

### `goto`
Almost always avoided, but valid for multi-level loop breaking or centralized cleanup in legacy C-style codebases (though RAII is the modern C++ alternative to cleanup).

---

## 5. Functions

### Function Overloading & Resolution
Functions can share names if parameter lists differ. The compiler resolves using:
1. Exact match
2. Promotion (e.g., `short` -> `int`)
3. Standard conversions (e.g., `int` -> `double`)
4. User-defined conversions

### Default Arguments Pitfalls
Evaluated at the call site. Cannot be redefined.
> [!WARNING]
> Virtual functions with default arguments are a massive footgun. The default argument is statically bound (based on the pointer type), but the function is dynamically bound.

### Inline, Constexpr, Consteval
- `inline`: Suggests replacing function call with body. Mostly used today to allow multiple definitions of a function across translation units (ODR).
- `constexpr`: Function *can* be evaluated at compile time if inputs are compile-time constants.
- `consteval` (C++20): Function *must* be evaluated at compile time (immediate function).

### Function Pointers vs `std::function`
`std::function` (from `<functional>`) is a type-erased wrapper that can store any callable (function pointers, lambdas, functors). It has overhead (possible heap allocation). Use function pointers or templates where performance is critical.

### Variadic Functions
C-style (using `va_list`) is unsafe. Modern C++ uses variadic templates (parameter packs).

```cpp
template<typename T>
void print_all(T t) { std::cout << t << '\n'; }

template<typename T, typename... Args>
void print_all(T t, Args... args) {
    std::cout << t << ", ";
    print_all(args...); // Recursive pack expansion
}

// C++17 Fold Expressions make it cleaner:
template<typename... Args>
void print_fold(Args... args) {
    (std::cout << ... << args) << '\n';
}
```

---

## 6. Namespaces

### `using` Declarations vs Directives
- Declaration: `using std::cout;` (Brings a single name).
- Directive: `using namespace std;` (Brings all names. **NEVER use in header files!** Leads to global namespace pollution).

### ADL (Argument-Dependent Lookup)
Koenig Lookup. The compiler looks for functions in the namespaces of their arguments.

```cpp
namespace MyLib {
    struct MyStruct {};
    void process(MyStruct m) {}
}

int main() {
    MyLib::MyStruct s;
    process(s); // ADL finds MyLib::process without explicit scope!
}
```

### Inline Namespaces
Used for versioning libraries. Members of an inline namespace are exposed as if they belong to the parent namespace.

```cpp
namespace Lib {
    inline namespace V2 { void foo() {} }
    namespace V1 { void foo() {} }
}
// Lib::foo() calls Lib::V2::foo()
```

### Anonymous Namespaces
Use instead of `static` for internal linkage.
```cpp
namespace {
    int internal_variable = 42; // Only visible in this file
}
```

---

## 7. Preprocessor

Executes before actual compilation. Understands text manipulation, not C++ syntax.

### Macros
```cpp
#define PI 3.14159                  // Object-like
#define MIN(a, b) ((a)<(b)?(a):(b)) // Function-like (Notice the parentheses!)
#define LOG(fmt, ...) printf(fmt, __VA_ARGS__) // Variadic
```

### Include Guards vs `#pragma once`
Include guards `#ifndef HEADER_H ...` are standard. `#pragma once` is non-standard but supported by virtually all modern compilers and prevents macro name collisions.

### Predefined Macros
`__FILE__`, `__LINE__` are preprocessor macros. `__func__` is a compiler-defined local variable representing the function name.

### C++20 Modules Brief
Modules (`import <iostream>;`) are designed to replace the preprocessor `#include`. They solve slow compile times by compiling modular interfaces once, and prevent macro leakage across translation units.

---

## Interview Questions & Answers

**Q1: What is the exact size of `int` in C++?**
**A1:** The C++ standard does not specify an exact size, only that it is at least 16 bits and `sizeof(short) <= sizeof(int) <= sizeof(long)`. On most modern 32/64-bit systems, it is 32 bits (4 bytes).

**Q2: Differentiate between `static_cast` and `reinterpret_cast`.**
**A2:** `static_cast` performs well-defined, safe conversions (like `int` to `double` or upcasting). It can fail at compile time. `reinterpret_cast` blindly reinterprets the bit pattern of one type as another (like `int*` to `long`), circumventing the type system, often leading to undefined behavior if misused (strict aliasing violations).

**Q3: Why should you avoid `using namespace std;` in header files?**
**A3:** It injects the entirety of the `std` namespace into the global namespace of any file that includes the header. This defeats the purpose of namespaces and leads to name collisions.

**Q4: What is Argument-Dependent Lookup (ADL)?**
**A4:** ADL allows the compiler to find unqualified function calls by searching the namespaces of the function's arguments. E.g., `std::cout << "Hi"` uses ADL to find `std::operator<<`.

**Q5: What is the difference between `const` and `constexpr` for variables?**
**A5:** `const` implies read-only at runtime (value cannot change after initialization). `constexpr` implies the value is known and fixed at *compile-time*.

**Q6: What happens if you define a virtual function with a default parameter?**
**A6:** It's highly discouraged. Virtual functions bind dynamically (at runtime based on object type), but default arguments bind statically (at compile time based on the pointer/reference type). A base class pointer to a derived object will use the Base class's default argument, but execute the Derived class's function body!

**Q7: Explain the output difference between `a++` and `++a` as an expression.**
**A7:** `++a` (pre-increment) increments the value and returns a reference to the updated variable. `a++` (post-increment) creates a copy of the old value, increments the original, and returns the old copy (by value).

**Q8: What is a structured binding, and when was it introduced?**
**A8:** Introduced in C++17, structured bindings (`auto [x, y] = my_pair;`) allow easy unpacking of tuples, pairs, structs, and arrays into distinct variables.

**Q9: What does `[[fallthrough]]` achieve?**
**A9:** Introduced in C++17, it suppresses compiler warnings for intentional switch-case fallthroughs, signaling to both compiler and human readers that the omission of `break;` is deliberate.

**Q10: Contrast Anonymous Namespaces with `static` globals.**
**A10:** Both provide internal linkage (variables/functions are not visible outside the translation unit). However, anonymous namespaces are the modern C++ way, can contain custom types (classes), and apply to template specializations, whereas `static` is a C-legacy limited mostly to functions and variables.

**Q11: What is the purpose of `decltype`?**
**A11:** `decltype(expr)` inspects the declared type of an entity or the type and value category of an expression. Unlike `auto`, it does not strip `const` or reference qualifiers.

**Q12: How does `if constexpr` differ from a regular `if`?**
**A12:** Evaluated at compile-time. The discarded branch is not fully compiled (e.g., it can contain code that would be a compilation error for a specific template instantiation).

**Q13: Why is list initialization `int x{3.14};` safer than `int x = 3.14;`?**
**A13:** List initialization (using braces) strictly forbids narrowing conversions. `int x{3.14}` will result in a compiler error, while `int x = 3.14` will silently truncate to `3`.

**Q14: Explain Short-Circuit Evaluation.**
**A14:** For logical AND (`&&`) and OR (`||`), the second operand is not evaluated if the result can be determined solely from the first operand.

**Q15: What is the rule regarding the `thread_local` storage specifier?**
**A15:** `thread_local` dictates that the variable has thread storage duration. Each thread gets its own independent copy of the variable, initialized when the thread starts (or on first use) and destroyed when the thread terminates.

# Module 06 — Modern C++ Feature Tour (C++11 through C++23)

Welcome to Module 06! C++ has undergone a massive transformation starting with C++11. This module is a comprehensive guide to the modern features introduced from C++11 through C++23. It focuses on the "why", practical usage, and how these features make C++ safer, faster, and more expressive.

---

## 1. C++11 — The Revolution

C++11 was a paradigm shift that modernized the language, making it feel almost like a new language.

### Core Language Features

#### `auto` Keyword
**Why:** Replaces verbose type declarations and prevents accidental implicit conversions.
```cpp
// Before
std::vector<int>::const_iterator it = vec.begin();
// C++11
auto it = vec.begin(); 
```
**Gotcha:** `auto` strips references and `const`/`volatile` qualifiers. Use `auto&` or `const auto&` if you want a reference.

#### Range-based `for` loops
**Why:** Simplifies iterating over containers and arrays.
```cpp
std::vector<int> numbers = {1, 2, 3, 4, 5};
for (const auto& num : numbers) {
    std::cout << num << " ";
}
```

#### `nullptr`
**Why:** Replaces the `NULL` macro (often defined as `0`). `nullptr` is of type `std::nullptr_t` and prevents ambiguous function calls between integer and pointer overloads.
```cpp
void func(int);
void func(int*);

func(NULL);    // Calls func(int) in pre-C++11, ambiguous or unexpected!
func(nullptr); // Unambiguously calls func(int*)
```

#### Strongly typed enums (`enum class`)
**Why:** Traditional enums pollute the surrounding scope and implicitly convert to integers. `enum class` solves both.
```cpp
enum class Color { Red, Green, Blue };
Color c = Color::Red; // Must be qualified
// int i = c; // Error: No implicit conversion to int
```

#### Lambda Expressions
**Why:** Allows writing inline, anonymous function objects. Perfect for standard library algorithms.
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
int multiplier = 2;
// Capture multiplier by value, modify elements inline
std::for_each(vec.begin(), vec.end(), [multiplier](int& n) {
    n *= multiplier;
});
```

#### Move Semantics and Rvalue References (`&&`)
**Why:** Eliminates unnecessary deep copies. Allows resources (like dynamically allocated memory) to be "stolen" from temporary objects (rvalues).
```cpp
class HugeData {
    int* data;
public:
    // Move constructor
    HugeData(HugeData&& other) noexcept : data(other.data) {
        other.data = nullptr; // Leave moved-from object in valid state
    }
};
```

#### Smart Pointers (`std::unique_ptr`, `std::shared_ptr`)
**Why:** Replaces raw pointers for ownership management, practically eliminating memory leaks.
```cpp
#include <memory>
// Unique ownership: strict ownership, zero overhead
std::unique_ptr<int> p1 = std::make_unique<int>(42); // make_unique is C++14

// Shared ownership: reference counted
std::shared_ptr<int> p2 = std::make_shared<int>(100);
```

#### Uniform Initialization (Brace Initialization)
**Why:** Provides a consistent syntax to initialize any object and prevents narrowing conversions.
```cpp
int x{5};
double y{5.5};
// int z{y}; // Error: Narrowing conversion prevented!
```

#### `constexpr` (Basics)
**Why:** Evaluates expressions and functions at compile-time, improving runtime performance.
```cpp
constexpr int getArraySize() { return 10; }
int arr[getArraySize()]; // Valid!
```

#### `static_assert`
**Why:** Compile-time assertions. Fails the build if a condition is false.
```cpp
static_assert(sizeof(void*) == 8, "Requires 64-bit architecture");
```

#### Variadic Templates
**Why:** Allows templates to accept an arbitrary number of arguments.
```cpp
template<typename T>
void print(T arg) { std::cout << arg << '\n'; }

template<typename T, typename... Args>
void print(T arg, Args... args) {
    std::cout << arg << ' ';
    print(args...);
}
```

#### Thread Support Library
**Why:** Standardized multithreading (`std::thread`, `std::mutex`, `std::async`).
```cpp
#include <thread>
void doWork() {}
std::thread t(doWork);
t.join();
```

### Other Notable C++11 Features
- **Type Traits (`<type_traits>`)**: Compile-time type introspection.
- **`std::array`, `std::forward_list`, `std::unordered_map`**: Hash tables and static arrays.
- **Delegating Constructors**: Constructors calling other constructors in the same class.
- **`override` / `final`**: Explicit intent for virtual functions. `override` prevents silent bugs when signatures mismatch.
- **Raw String Literals**: `R"(C:\Path\To\File)"` avoids escaping backslashes.
- **User-Defined Literals**: `10_km`, `5_h`.
- **`noexcept`**: Specifies that a function does not throw exceptions (enables move optimization in containers).
- **Trailing Return Types**: `auto func() -> int`.
- **`decltype`**: Deduces the exact type of an expression.
- **`std::initializer_list`**: Allows functions to accept `{1, 2, 3}`.
- **`= default` / `= delete`**: Explicitly generating or forbidding special member functions.

---

## 2. C++14 — The Polish

C++14 smoothed out C++11's rough edges and added minor but highly useful features.

#### Generic Lambdas
**Why:** Allows `auto` parameters in lambdas.
```cpp
auto add = [](auto a, auto b) { return a + b; };
std::cout << add(5, 5.5); // Works with mixed types
```

#### Lambda Capture Initializers
**Why:** Allows capturing by move into a lambda (e.g., `std::unique_ptr`).
```cpp
auto ptr = std::make_unique<int>(10);
auto lambda = [p = std::move(ptr)]() {
    std::cout << *p;
};
```

#### Return Type Deduction
**Why:** `auto` can be used as the return type for normal functions.
```cpp
auto multiply(int a, double b) {
    return a * b; // Deduces double
}
```

#### Variable Templates
**Why:** Templates for variables, not just functions or classes.
```cpp
template<typename T>
constexpr T pi = T(3.1415926535897932385);
```

#### Binary Literals & Digit Separators
**Why:** Readability.
```cpp
int bin = 0b1010'0011'1100;
long money = 1'000'000;
```

#### `std::make_unique`
**Why:** C++11 forgot `make_unique`. Always prefer it over `new` to prevent memory leaks in complex expressions.

#### Relaxed `constexpr`
**Why:** `constexpr` functions can now have loops, `if` statements, and local variables.

#### `[[deprecated]]` Attribute
```cpp
[[deprecated("Use newFunc() instead")]]
void oldFunc();
```

#### `std::exchange`
**Why:** Replaces the old value with a new one and returns the old value. Extremely useful for move constructors.

---

## 3. C++17 — The Productivity Boost

C++17 drastically improved daily programming ergonomics.

#### Structured Bindings
**Why:** Unpack tuples, pairs, and structs directly into variables.
```cpp
std::map<int, std::string> m = {{1, "A"}};
for (auto& [key, value] : m) {
    std::cout << key << ": " << value << '\n';
}
```

#### `if` / `switch` with Initializer
**Why:** Limits the scope of variables to the `if` block.
```cpp
if (auto it = m.find(1); it != m.end()) {
    std::cout << "Found: " << it->second;
} // 'it' is destroyed here
```

#### `std::optional`, `std::variant`, `std::any`
**Why:** Safe alternatives to pointers for optional values, unions, and `void*`.
```cpp
std::optional<int> divide(int a, int b) {
    if (b == 0) return std::nullopt;
    return a / b;
}
```

#### `std::string_view`
**Why:** A non-owning, lightweight view of a string. Prevents unnecessary allocations.
```cpp
void print(std::string_view sv) {
    std::cout << sv;
}
// print("hello") does NOT allocate memory!
```

#### Fold Expressions
**Why:** Vastly simplifies variadic templates.
```cpp
template<typename... Args>
auto sum(Args... args) {
    return (... + args); // Unpacks into arg1 + arg2 + ...
}
```

#### Class Template Argument Deduction (CTAD)
**Why:** No need to specify template types if they can be deduced from the constructor.
```cpp
std::pair p(1, 3.14); // Deduces std::pair<int, double>
```

#### `if constexpr`
**Why:** Compile-time `if` statements. Replaces SFINAE (enable_if) hacks.
```cpp
template <typename T>
void process(T t) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Integer processing\n";
    } else {
        std::cout << "Other processing\n";
    }
}
```

#### `std::filesystem`
**Why:** A standard way to interact with paths, files, and directories.
```cpp
#include <filesystem>
namespace fs = std::filesystem;
fs::path p = "/tmp/test.txt";
```

#### Other C++17 Features
- **Inline Variables**: Define variables in headers without ODR (One Definition Rule) violations.
- **Nested Namespaces**: `namespace A::B::C { }`.
- **Attributes**: `[[nodiscard]]`, `[[maybe_unused]]`, `[[fallthrough]]`.
- **Parallel Algorithms**: `std::sort(std::execution::par, vec.begin(), vec.end());`.

---

## 4. C++20 — The Major Release

C++20 is arguably the biggest update since C++11, introducing the "Big Four".

### The Big Four

#### 1. Concepts
**Why:** Adds constraints to templates, providing readable error messages instead of endless template spew.
```cpp
#include <concepts>

template<std::integral T>
T add(T a, T b) {
    return a + b;
}
// add(1.5, 2.5); // Compile error: constraints not satisfied
```

#### 2. Ranges (`std::ranges`)
**Why:** Replaces `begin()/end()` iterators with range objects and introduces composable "views" (lazy evaluation).
```cpp
#include <ranges>
#include <vector>

std::vector<int> v = {1, 2, 3, 4, 5, 6};
auto even_squares = v 
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::transform([](int n) { return n * n; });
```

#### 3. Coroutines
**Why:** Asynchronous programming made native. Functions that can pause and resume (`co_await`, `co_yield`, `co_return`).
*(Note: C++20 provides the language framework; standard library types like `std::generator` arrived in C++23).*

#### 4. Modules
**Why:** Replaces `#include` header files. Dramatically speeds up compilation times and eliminates macro pollution.
```cpp
// math.ixx
export module math;
export int add(int a, int b) { return a + b; }

// main.cpp
import math;
int main() { return add(2, 3); }
```

### Other C++20 Features

#### Three-way Comparison (Spaceship Operator `<=>`)
**Why:** Generates all 6 comparison operators automatically.
```cpp
struct Point {
    int x, y;
    auto operator<=>(const Point&) const = default;
};
```

#### Designated Initializers
**Why:** Clearer struct initialization (borrowed from C).
```cpp
struct Config { int a; double b; };
Config c = {.a = 1, .b = 2.0};
```

#### `std::format`
**Why:** Fast, type-safe, Python-like string formatting replacing `printf` and `std::stringstream`.
```cpp
#include <format>
std::string s = std::format("Hello {}! You have {} messages.", "Alice", 5);
```

#### `std::span`
**Why:** A non-owning view of a contiguous sequence (array, vector). Safer than passing pointer + size.
```cpp
void process(std::span<int> data) { /* ... */ }
```

#### Constexpr Improvements, `consteval`, and `constinit`
- Virtual functions, `dynamic_cast`, string, and vector can now be used in `constexpr`.
- `consteval`: Function *must* be evaluated at compile time.
- `constinit`: Variable *must* be initialized at compile time (solves the Static Initialization Order Fiasco).

#### `std::jthread`
**Why:** A self-joining thread. When it goes out of scope, it automatically joins and supports cancellation via stop tokens.

---

## 5. C++23 — The Latest

C++23 is largely a library-focused release that polishes C++20.

#### `std::expected`
**Why:** Represents either an expected value or an error. A much cleaner alternative to throwing exceptions or returning pairs.
```cpp
#include <expected>
enum class Error { NotFound, AccessDenied };

std::expected<int, Error> get_value(bool success) {
    if (success) return 42;
    else return std::unexpected(Error::NotFound);
}
```

#### Deducing `this` (Explicit Object Parameters)
**Why:** Simplifies CRTP (Curiously Recurring Template Pattern) and avoids duplicating code for `const` and non-const overloads.
```cpp
struct MyStruct {
    template <typename Self>
    auto& get_x(this Self&& self) {
        return std::forward<Self>(self).x;
    }
    int x = 5;
};
```

#### `std::print` / `std::println`
**Why:** Formats and prints directly to the console without the overhead of `std::cout`.
```cpp
#include <print>
std::println("Hello, C++{}!", 23);
```

#### Multidimensional Subscript Operator `[]`
**Why:** You can now pass multiple arguments to `operator[]`.
```cpp
struct Grid {
    int operator[](int x, int y) { return x * y; }
};
```

#### `std::flat_map`, `std::flat_set`
**Why:** Contiguous-memory drop-in replacements for `std::map`. They are vectors sorted internally. Highly cache-friendly and usually faster for lookups than `std::map`.

#### `std::generator`
**Why:** The first standard library coroutine type! Makes writing lazily evaluated sequences trivial.
```cpp
#include <generator>
std::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}
```

#### `std::mdspan`
**Why:** A multi-dimensional view over contiguous memory (e.g., viewing a 1D vector as a 2D matrix).

#### Monadic Operations on `std::optional`
**Why:** Functional chaining for optionals (`and_then`, `transform`, `or_else`).
```cpp
std::optional<int> get_opt();
auto result = get_opt().transform([](int n) { return n * 2; })
                       .or_else([] { return std::optional<int>(0); });
```

#### Other C++23 Features
- `if consteval`: Replacement for `std::is_constant_evaluated()`.
- `[[assume(expr)]]`: Optimization hint to the compiler.
- `static operator()`: Lambdas and functors with no state can be static.
- `std::stacktrace`: Standardized way to dump the call stack.
- `std::views::zip`: Zip multiple ranges together.
- `std::unreachable()`: Tells the compiler a code path is impossible, allowing optimizations.

---

## 🔑 Interview Questions & Answers

**Q1. What is the difference between `auto` and `decltype`?**
*Answer:* `auto` strips reference and const/volatile qualifiers during type deduction (unless explicitly specified like `const auto&`). `decltype` deduces the exact type, preserving all qualifiers and references. 

**Q2. Explain move semantics. How does `std::move` work?**
*Answer:* Move semantics allow stealing resources from temporary objects (rvalues) instead of making deep copies. `std::move` doesn't actually move anything; it is a cast that converts an lvalue into an rvalue reference (`T&&`), allowing a move constructor or move assignment operator to be invoked.

**Q3. What is the Rule of Five?**
*Answer:* If a class defines a destructor, it likely manages a resource and thus should explicitly define (or `= delete`) the copy constructor, copy assignment operator, move constructor, and move assignment operator.

**Q4. Why should you prefer `std::make_unique` over raw `new`?**
*Answer:* It prevents memory leaks. In `foo(std::unique_ptr<T>(new T()), bar())`, if `bar()` throws, the memory allocated by `new T()` is leaked before the unique_ptr constructor runs. `std::make_unique` ensures the memory is safely wrapped immediately. Also, it avoids typing the type name twice.

**Q5. Explain the difference between `constexpr` and `const`.**
*Answer:* `const` means the value cannot be modified after initialization. `constexpr` means the value *must* be known at compile time and can be used in compile-time contexts (like array sizes or template arguments).

**Q6. What problem do strongly typed enums (`enum class`) solve?**
*Answer:* Traditional enums pollute the surrounding namespace (enumerators leak out) and implicitly convert to integers, which can lead to logical bugs. `enum class` solves both by requiring scope resolution (`Color::Red`) and preventing implicit integer conversions.

**Q7. What is `std::string_view`? What are its gotchas?**
*Answer:* It is a non-owning reference to a string (just a pointer and a length). It is very fast to pass around. The main gotcha is dangling references: if the underlying string is destroyed or modified, the `string_view` becomes invalid. Also, it is not guaranteed to be null-terminated.

**Q8. Explain SFINAE and how C++20 Concepts replace it.**
*Answer:* SFINAE (Substitution Failure Is Not An Error) is a template metaprogramming technique where invalid template substitutions are silently discarded rather than causing hard errors. It relies on complex `<type_traits>` like `std::enable_if`. Concepts (C++20) replace this by allowing explicitly defined constraints (e.g., `requires std::integral<T>`), leading to vastly better compiler error messages and cleaner code.

**Q9. What are Structured Bindings?**
*Answer:* A C++17 feature that allows you to unpack public data members of structs, pairs, tuples, or arrays into distinct variables in one line: `auto [x, y, z] = get_vector3();`.

**Q10. How does `std::variant` differ from `std::any`?**
*Answer:* `std::variant` is a type-safe union that can hold exactly one value from a *pre-specified* list of types (e.g., `std::variant<int, float, string>`). `std::any` can hold *any* type whatsoever, functioning somewhat like a safe, type-aware `void*`.

**Q11. What is the purpose of `if constexpr`?**
*Answer:* It evaluates the condition at compile time. If the condition is false, the compiler completely discards the branch. It is highly useful in template programming to conditionally compile code based on type traits without needing multiple template overloads.

**Q12. What are C++20 Ranges and Views?**
*Answer:* Ranges provide algorithms that take range objects directly rather than `begin()/end()` iterator pairs. Views are lazy, composable adapters (like `filter`, `transform`) that don't allocate memory or mutate the underlying data.

**Q13. What is the "Static Initialization Order Fiasco" and how does `constinit` solve it?**
*Answer:* It occurs when global/static variables in different translation units depend on each other's initialization, but C++ does not guarantee the order of initialization across translation units. C++20 `constinit` forces the variable to be initialized at compile-time, ensuring it is ready before any dynamic initialization happens.

**Q14. What is `std::expected` in C++23 used for?**
*Answer:* It is a vocabulary type for error handling without exceptions. It holds either the expected valid return value or an error value, forcing the caller to explicitly check for success before accessing the value.

**Q15. Why use `std::span` instead of passing `std::vector` by reference?**
*Answer:* Passing `std::vector<int>&` ties the function exclusively to vectors. `std::span<int>` accepts a vector, a raw array, a `std::array`, or any contiguous block of memory, making the function much more flexible while maintaining bounds safety.

---
*Created as part of the C++ Mastery Course.*

# Module 04 — Templates & Generic Programming

Welcome to Module 04. Templates and generic programming are what elevate C++ from a mere object-oriented language to a multi-paradigm powerhouse. If you want to write highly reusable, performant, and type-safe code, mastering templates is non-negotiable.

In this module, we will dive deep into everything from basic function templates to C++20 Concepts and advanced metaprogramming techniques.

---

## 1. Function Templates

Function templates allow you to write generic functions that can operate on different data types. The compiler generates (instantiates) a specific version of the function for each type it is called with.

### 1.1 Template Syntax, Instantiation, and Deduction

```cpp
#include <iostream>

// Template definition
template <typename T>
T myMax(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    // Implicit instantiation (Template Argument Deduction)
    std::cout << myMax(5, 10) << '\n';       // Instantiates myMax<int>
    std::cout << myMax(5.5, 10.2) << '\n';   // Instantiates myMax<double>

    // Explicit instantiation
    std::cout << myMax<double>(5, 10.2) << '\n'; // Forces T to be double
    return 0;
}
```

> 💡 **Tip:** Prefer `typename` over `class` in template parameter lists. They mean the exact same thing, but `typename` makes it clearer that the parameter can be *any* type (including primitive types like `int`), not just a class.

### 1.2 Template Argument Deduction (TAD) Rules

When you call a template function without explicitly specifying the template arguments, the compiler attempts to deduce them from the function arguments.

- **Exact Match Required:** If a function template has a single template parameter `T` used for multiple arguments, they must be the *exact* same type.
- **Reference Stripping:** If the template parameter takes `T` by value, references are stripped. `const` and `volatile` qualifiers are also stripped.
- **Array-to-Pointer Decay:** When passing by value, arrays decay to pointers. When passing by reference, arrays retain their type and size.

```cpp
template <typename T> void byValue(T param) {}
template <typename T> void byReference(T& param) {}

int main() {
    const int x = 5;
    int arr[3] = {1, 2, 3};

    byValue(x);     // T is deduced as int (const is stripped)
    byValue(arr);   // T is deduced as int* (array decays)

    byReference(x);   // T is deduced as const int
    byReference(arr); // T is deduced as int[3] (no decay)
}
```

### 1.3 Overloading vs Specialization

You can overload a function template with a non-template function. Overload resolution always prefers a non-template function over a template function if both are exact matches.

```cpp
template <typename T>
void print(T a) { std::cout << "Template: " << a << '\n'; }

void print(int a) { std::cout << "Non-template: " << a << '\n'; }

int main() {
    print(5);     // Calls non-template
    print<>(5);   // Forces template instantiation
    print(5.5);   // Calls template
}
```

> ⚠️ **Warning:** Avoid specializing function templates. Overload them instead. Specializations do not participate in overload resolution the way you might expect, which can lead to counterintuitive behavior.

### 1.4 Return Type Deduction

In C++14, you can use `auto` as the return type for functions, allowing the compiler to deduce it.

```cpp
template <typename T1, typename T2>
auto add(T1 a, T2 b) {
    return a + b; // Return type deduced based on the result of a + b
}

// C++11 required trailing return types:
template <typename T1, typename T2>
auto add11(T1 a, T2 b) -> decltype(a + b) {
    return a + b;
}
```

---

## 2. Class Templates

Class templates allow you to define a blueprint for a class where some types or values are generalized.

### 2.1 Syntax and Instantiation

```cpp
template <typename T>
class Box {
private:
    T value;
public:
    Box(T val) : value(val) {}
    T getValue() const; // Declaration only
};

// Member function definition outside the class
template <typename T>
T Box<T>::getValue() const {
    return value;
}
```

### 2.2 Template Parameters

Templates can take three types of parameters:
1. **Type parameters** (`typename T`)
2. **Non-type parameters** (NTTPs like `int N`, pointers, enums. Since C++20, even floating-point types and structural class types).
3. **Template template parameters** (A template parameter that is itself a template).

```cpp
// Non-type template parameter
template <typename T, std::size_t N>
class StaticArray {
    T data[N];
public:
    std::size_t size() const { return N; }
};

// Template template parameter
template <template <typename, typename> class Container, typename T, typename Alloc = std::allocator<T>>
class Wrapper {
    Container<T, Alloc> data;
};
```

### 2.3 Default Template Arguments

```cpp
template <typename T = int, std::size_t N = 10>
class Buffer { /* ... */ };

Buffer<> b; // Buffer<int, 10>
```

### 2.4 Class Template Argument Deduction (CTAD) - C++17

Before C++17, you always had to specify template arguments for classes (`std::pair<int, double> p(1, 2.0);`). C++17 introduced CTAD.

```cpp
std::pair p(1, 2.0); // Deduced as std::pair<int, double>
std::vector v{1, 2, 3}; // Deduced as std::vector<int>
Box b{42}; // Deduced as Box<int>
```

**Deduction Guides:** Sometimes the compiler needs help with CTAD.

```cpp
template <typename T>
struct Wrapper {
    T val;
    Wrapper(T v) : val(v) {}
};

// Deduction guide (though implicit guide works here, this is the syntax)
template <typename T> Wrapper(T) -> Wrapper<T>;
```

---

## 3. Template Specialization

Sometimes a generic template implementation doesn't work or isn't optimal for a specific type. You can specialize the template.

### 3.1 Full Specialization

Provides a complete replacement for a specific set of template arguments.

```cpp
template <typename T>
struct Traits {
    static void print() { std::cout << "Generic traits\n"; }
};

// Full specialization for int
template <>
struct Traits<int> {
    static void print() { std::cout << "Traits for int\n"; }
};
```

### 3.2 Partial Specialization (Class Templates Only)

Allows you to specialize for a subset of types (e.g., all pointers).

```cpp
template <typename T>
struct Traits {
    static const bool is_pointer = false;
};

// Partial specialization for any pointer type
template <typename T>
struct Traits<T*> {
    static const bool is_pointer = true;
};
```

> 🔑 **Key Insight:** Function templates CANNOT be partially specialized. If you need partial specialization for a function, either use overloading, or delegate the function call to a static member function of a class template (which *can* be partially specialized).

### 3.3 Tag Dispatch Pattern

Tag dispatch is a technique used before `if constexpr` and Concepts to choose different function overloads at compile time based on type traits.

```cpp
#include <iterator>

// Implementation for random access iterators
template <typename Iter>
void advance_impl(Iter& it, int n, std::random_access_iterator_tag) {
    it += n;
}

// Implementation for forward iterators
template <typename Iter>
void advance_impl(Iter& it, int n, std::forward_iterator_tag) {
    while (n--) ++it;
}

template <typename Iter>
void my_advance(Iter& it, int n) {
    // Use iterator_category as a tag to dispatch to the correct overload
    advance_impl(it, n, typename std::iterator_traits<Iter>::iterator_category());
}
```

---

## 4. Variadic Templates

Introduced in C++11, variadic templates accept an arbitrary number of template arguments.

### 4.1 Parameter Packs and Expansion

```cpp
// Base case for recursion
void print() {}

// Variadic template
template <typename T, typename... Args> // Args is a template parameter pack
void print(T first, Args... args) {     // args is a function parameter pack
    std::cout << first << ' ';
    print(args...);                     // Pack expansion
}
```

You can get the number of elements in a pack using `sizeof...(Args)`.

### 4.2 Fold Expressions (C++17)

Fold expressions drastically simplify variadic templates by removing the need for recursion.

```cpp
template <typename... Args>
auto sum(Args... args) {
    return (... + args); // Unary left fold: (((arg1 + arg2) + arg3) + ...)
}

template <typename... Args>
void print_all(Args... args) {
    // Binary fold over the comma operator
    (..., (std::cout << args << " "));
    std::cout << '\n';
}
```

### 4.3 std::apply and std::make_from_tuple (C++17)

`std::apply` invokes a Callable with a tuple of arguments.

```cpp
#include <tuple>

void f(int a, double b, const char* c) { /* ... */ }

int main() {
    std::tuple t{1, 2.5, "hello"};
    std::apply(f, t); // Unpacks the tuple and calls f(1, 2.5, "hello")
}
```

---

## 5. SFINAE (Substitution Failure Is Not An Error)

When the compiler tries to instantiate a template during overload resolution, if substituting the template parameter results in an invalid type or expression, the compiler doesn't throw a hard error. Instead, it silently discards that overload and looks for others.

### 5.1 std::enable_if

```cpp
#include <type_traits>
#include <iostream>

// Enabled only if T is an integral type
template <typename T, typename std::enable_if_t<std::is_integral_v<T>, int> = 0>
void process(T val) {
    std::cout << "Processing integral: " << val << '\n';
}

// Enabled only if T is a floating-point type
template <typename T, typename std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
void process(T val) {
    std::cout << "Processing float: " << val << '\n';
}
```

### 5.2 The void_t Trick

`std::void_t` is a metaprogramming tool used to detect ill-formed types. It maps any sequence of types to `void`.

```cpp
#include <type_traits>

template <typename, typename = std::void_t<>>
struct has_type_member : std::false_type {};

// This specialization is chosen if T::type is well-formed
template <typename T>
struct has_type_member<T, std::void_t<typename T::type>> : std::true_type {};

struct A { using type = int; };
struct B {};

// has_type_member<A>::value is true
// has_type_member<B>::value is false
```

---

## 6. Concepts (C++20)

Concepts revolutionized template programming. They replace SFINAE with a clean, readable syntax for constraining template parameters.

### 6.1 Defining Concepts and requires Clauses

```cpp
#include <concepts>
#include <iostream>

// Defining a concept
template <typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>; // Requires a + b to be valid and convertible to T
};

// Constraining a template using a concept
template <Addable T>
T add(T a, T b) {
    return a + b;
}

// Alternative syntax using requires clause
template <typename T>
requires Addable<T>
T add2(T a, T b) {
    return a + b;
}
```

### 6.2 Standard Library Concepts

The `<concepts>` header provides many built-in concepts: `std::integral`, `std::floating_point`, `std::same_as`, `std::derived_from`, etc.

```cpp
#include <concepts>
#include <iostream>

void print_int(std::integral auto val) { // Abbreviated function template
    std::cout << val << '\n';
}
```

### 6.3 Concepts vs SFINAE Comparison

- **Readability:** Concepts are vastly more readable than `enable_if`.
- **Compilation Time:** Concepts compile faster.
- **Error Messages:** Concept failures produce clear, direct error messages ("constraint not satisfied"), whereas SFINAE failures produce massive compiler vomit.
- **Subsumption:** Concepts allow the compiler to order overloads based on constraint strength (a more constrained template is preferred).

---

## 7. Template Metaprogramming (TMP)

TMP uses the C++ compiler as an interpreter to compute values or generate types at compile time.

### 7.1 Compile-Time Computation

```cpp
template <int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template <>
struct Factorial<0> {
    static constexpr int value = 1;
};

// Computed at compile time
constexpr int f5 = Factorial<5>::value; // 120
```

> 💡 **Tip:** While classic recursive TMP is important to understand, modern C++ prefers `constexpr` and `consteval` functions for value computations, as they use normal C++ syntax.

### 7.2 `if constexpr` (C++17)

`if constexpr` allows conditional compilation within a template. Code inside discarded branches is not instantiated.

```cpp
#include <iostream>
#include <type_traits>

template <typename T>
void print_info(T val) {
    if constexpr (std::is_pointer_v<T>) {
        std::cout << "Pointer to " << *val << '\n';
    } else {
        std::cout << "Value: " << val << '\n';
    }
}
```

---

## 8. Advanced Topics

### 8.1 Two-Phase Name Lookup & Dependent Names

When parsing a template, the compiler does a "two-phase" lookup:
1. **Phase 1 (Definition time):** Non-dependent names (names that don't depend on a template parameter) are looked up and bound immediately.
2. **Phase 2 (Instantiation time):** Dependent names (names that depend on `T`) are looked up.

Because of this, the compiler doesn't know if `T::Nested` is a type or a static member variable. You must use `typename` to disambiguate types, and `template` to disambiguate member templates.

```cpp
#include <vector>

template <typename T>
void func() {
    // typename is required because vector<T>::iterator is a dependent type
    typename std::vector<T>::iterator it;
}
```

### 8.2 Curiously Recurring Template Pattern (CRTP)

CRTP is a technique where a derived class inherits from a base template class instantiated with the derived class itself. It's used for static polymorphism (avoiding `virtual` function overhead).

```cpp
#include <iostream>

template <typename Derived>
struct Base {
    void interface() {
        // Cast to derived and call implementation
        static_cast<Derived*>(this)->implementation();
    }
};

struct Derived1 : Base<Derived1> {
    void implementation() { std::cout << "Derived1\n"; }
};
```

---

## 9. Interview Questions

1. **What is Template Argument Deduction (TAD) and when does it fail?**
   *Answer:* TAD is the compiler's ability to deduce the type `T` from the function arguments. It fails when deduction is ambiguous (e.g., `std::max(5, 5.5)` where `T` could be `int` or `double`), or when the type is in a non-deduced context (like to the left of the scope resolution operator `::`).

2. **Why should you prefer overloading over specializing function templates?**
   *Answer:* Function template specializations don't participate in overload resolution. The compiler first resolves overloads among base templates and non-templates, and *only then* looks for specializations of the chosen base template. This leads to unintuitive behaviors. Overloading provides clean, predictable resolution.

3. **What is SFINAE? Give an example.**
   *Answer:* Substitution Failure Is Not An Error. During overload resolution, if replacing a template parameter with an actual type produces invalid code, the compiler silently ignores that overload rather than emitting an error. Example: Using `std::enable_if` in a template signature to disable the function for floating-point types.

4. **Explain the `typename` keyword disambiguator.**
   *Answer:* Inside a template, if an identifier depends on a template parameter and refers to a type (a dependent type), the compiler assumes it's a value/variable by default. You must prefix it with `typename` (e.g., `typename T::value_type`) to tell the compiler it's a type.

5. **What is CTAD and when was it introduced?**
   *Answer:* Class Template Argument Deduction, introduced in C++17. It allows omitting template arguments for class templates when instantiating them, letting the compiler deduce them from constructor arguments (e.g., `std::vector v{1, 2, 3};`).

6. **What are fold expressions?**
   *Answer:* Introduced in C++17, fold expressions perform an operation over all elements of a parameter pack without needing recursive template instantiations. E.g., `(... + args)` sums all arguments.

7. **How do Concepts improve upon SFINAE?**
   *Answer:* Concepts (C++20) provide a built-in language feature for constraining templates. They compile faster, provide significantly clearer error messages, read like English, and support subsumption (compiler can rank overloads by constraint specificity). SFINAE is a hack relying on compiler behavior.

8. **What is `if constexpr` and why is it useful?**
   *Answer:* Introduced in C++17, `if constexpr` evaluates a condition at compile time. The compiler completely discards the branch that evaluates to false, meaning the discarded code doesn't even need to be well-formed for the current template type. It replaces many SFINAE and tag dispatch use cases.

9. **Explain the CRTP pattern and its primary use case.**
   *Answer:* Curiously Recurring Template Pattern: `class Derived : public Base<Derived>`. It enables static polymorphism. The base class can cast `this` to `Derived*` to call derived methods. It provides polymorphism without the runtime overhead of virtual dispatch (vtable).

10. **What is the `void_t` idiom?**
    *Answer:* `std::void_t` is a template alias that maps any number of types to `void`. It is heavily used in SFINAE to detect whether a class has a specific member (type, function, or data) at compile time by utilizing the fact that substitution will fail if the member doesn't exist.

11. **Can you specialize a class template method without specializing the whole class?**
    *Answer:* Yes, you can explicitly specialize a member function of a class template for a specific template argument, without specializing the entire class.

12. **What does `sizeof...(Args)` do?**
    *Answer:* It returns the number of elements in a template parameter pack at compile time.

13. **What is the One Definition Rule (ODR) and how does it apply to templates?**
    *Answer:* The ODR states a class, template, or inline function can have multiple identical definitions across translation units. Since templates must be instantiated by the compiler, their full definitions usually reside in header files. As long as the definitions are token-for-token identical across translation units, the linker will collapse them into a single instance without ODR violations.

14. **What is `extern template`?**
    *Answer:* Introduced in C++11, it tells the compiler *not* to instantiate a template in the current translation unit, because it is explicitly instantiated elsewhere. This drastically reduces compile times for heavily used templates like `std::vector<int>`.

15. **Explain the difference between a template type parameter and a non-type template parameter.**
    *Answer:* A type parameter (`typename T`) expects a type (like `int`, `std::string`). A non-type parameter (`int N`) expects a compile-time constant value. Non-type parameters are often used for fixed sizes (e.g., `std::array<int, 5>`).

---

## 10. Quick Reference / Cheat Sheet

| Feature | Syntax Snippet | C++ Version |
| :--- | :--- | :--- |
| **Function Template** | `template <typename T> void f(T x);` | 98 |
| **Class Template** | `template <typename T> class C {};` | 98 |
| **Variadic Template** | `template <typename... Ts> void f(Ts... args);` | 11 |
| **Alias Template** | `template <typename T> using Vec = std::vector<T>;` | 11 |
| **Variable Template** | `template <typename T> constexpr T pi = T(3.14);` | 14 |
| **Fold Expressions** | `return (... + args);` | 17 |
| **if constexpr** | `if constexpr (std::is_same_v<T, int>) { ... }` | 17 |
| **CTAD** | `std::pair p(1, 2.0);` | 17 |
| **Concepts** | `template <std::integral T> void f(T x);` | 20 |

---
*End of Module 04*

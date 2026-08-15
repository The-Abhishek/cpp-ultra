# Module 07 — Move Semantics & Perfect Forwarding

Welcome to one of the most critical, powerful, and often misunderstood topics in modern C++: **Move Semantics and Perfect Forwarding**. Introduced in C++11, these features fundamentally changed how C++ code is written, drastically improving performance by eliminating unnecessary deep copies, and enabling perfect forwarding for generic programming.

In this comprehensive module, we will explore value categories in depth, understand what `std::move` and `std::forward` *actually* do, build our own move-aware types, and learn how to avoid common performance pitfalls.

---

## 1. Value Categories (The Complete Taxonomy)

Before understanding *how* to move, you must understand *what* can be moved. In C++11, the simple "lvalue vs rvalue" model was expanded into a finer-grained taxonomy based on two properties:
1. **Has identity**: Does the expression have an identifiable memory location (can you take its address)?
2. **Can be moved from**: Is it safe to steal resources from the object the expression evaluates to?

Based on these properties, C++ defines three primary value categories and two mixed categories:

### The Primary Categories

*   **lvalue** (left value): Has identity, cannot be moved from.
    *   Examples: Named variables, dereferenced pointers `*p`, functions, string literals `"hello"`, pre-increment `++x`.
*   **prvalue** (pure rvalue): Has no identity, can be moved from.
    *   Examples: Literals (except strings) `42`, `true`, temporary objects created by value returning functions `str.substr(0, 5)`, the result of arithmetic `x + y`, lambdas.
*   **xvalue** (eXpiring value): Has identity, CAN be moved from.
    *   Examples: The result of `std::move(x)`, casting to an rvalue reference `static_cast<T&&>(x)`, accessing a member of an rvalue `MyStruct().member`.

### The Mixed Categories

*   **glvalue** (generalized lvalue): Has identity. (lvalue + xvalue)
*   **rvalue**: Can be moved from. (prvalue + xvalue)

> [!TIP]
> The most important distinction for move semantics is the **rvalue** category. If an expression is an rvalue (either a prvalue temporary or an xvalue cast), it can bind to an rvalue reference, meaning we can steal its resources!

### Determining Value Categories with `decltype`

You can use `decltype` to inspect the value category of an expression `E`:
*   If `decltype((E))` is `T&`, `E` is an lvalue.
*   If `decltype((E))` is `T&&`, `E` is an xvalue.
*   If `decltype((E))` is `T`, `E` is a prvalue.

```cpp
#include <iostream>
#include <type_traits>

template<typename T>
struct ValueCategory {
    static constexpr const char* name = "prvalue";
};

template<typename T>
struct ValueCategory<T&> {
    static constexpr const char* name = "lvalue";
};

template<typename T>
struct ValueCategory<T&&> {
    static constexpr const char* name = "xvalue";
};

#define CATEGORY(expr) ValueCategory<decltype((expr))>::name

int main() {
    int x = 10;
    
    std::cout << "x: " << CATEGORY(x) << '\n';               // lvalue
    std::cout << "10: " << CATEGORY(10) << '\n';             // prvalue
    std::cout << "std::move(x): " << CATEGORY(std::move(x)) << '\n'; // xvalue
    std::cout << "x + 5: " << CATEGORY(x + 5) << '\n';       // prvalue
    std::cout << "++x: " << CATEGORY(++x) << '\n';           // lvalue
    std::cout << "x++: " << CATEGORY(x++) << '\n';           // prvalue
}
```

---

## 2. Rvalue References (`T&&`)

An **rvalue reference** (`T&&`) is a new type of reference introduced in C++11. Its primary purpose is to bind to rvalues (temporaries or objects explicitly marked for moving), signaling that it is safe to scavenge their resources.

### Binding Rules

Understanding what binds to what is crucial:

| Reference Type | Binds to lvalue? | Binds to const lvalue? | Binds to rvalue? | Binds to const rvalue? |
| :--- | :--- | :--- | :--- | :--- |
| `T&` | Yes | No | No | No |
| `const T&` | Yes | Yes | Yes | Yes |
| `T&&` | No | No | Yes | No (practically) |
| `const T&&` | No | No | No | Yes (rarely used) |

> [!NOTE]
> `const T&` is the universal catcher for read-only access. It can bind to temporaries. `T&&` is specifically for mutable rvalues that we intend to modify (move from).

### Rvalue References Extend Lifetime

Just like `const T&`, binding a prvalue to an rvalue reference extends the lifetime of the temporary until the reference itself goes out of scope.

```cpp
std::string generateString() { return "Temporary String"; }

void lifetimeExtension() {
    // The temporary string's lifetime is extended to match 'ref'
    std::string&& ref = generateString(); 
    std::cout << ref << '\n'; 
} // temporary destroyed here
```

### ⚠️ KEY INSIGHT: Named Rvalue References are Lvalues!

This is the most common pitfall for beginners. **If it has a name, it's an lvalue.**

```cpp
void process(std::string&& rval_ref) {
    // rval_ref is an lvalue here! It has a name and an address.
    // If we want to pass it to another function taking an rvalue reference,
    // we MUST use std::move again.
    
    // consume(rval_ref); // ERROR: consume expects std::string&&, got lvalue
    consume(std::move(rval_ref)); // OK
}
```
Why? Because `rval_ref` could be used multiple times in the `process` function. If it implicitly acted as an rvalue, the first use might silently steal its guts, leaving subsequent uses with a hollow shell. The compiler forces you to explicitly say `std::move` to acknowledge "I am done with this object, steal away."

---

## 3. Move Semantics

### The Problem Solved
Before C++11, returning large objects by value or adding them to containers involved expensive deep copies.

```cpp
std::vector<int> createLargeVector() {
    std::vector<int> v(1000000, 42);
    return v; // C++98: Expensive deep copy! (Without RVO)
}
```

### Move Constructor and Move Assignment

Move semantics allows us to "steal" the guts of a temporary instead of copying them. We implement this via the **move constructor** and **move assignment operator**.

```cpp
class DynamicArray {
private:
    int* data_;
    size_t size_;

public:
    // Default Constructor
    DynamicArray(size_t size) : size_(size), data_(new int[size]) {}

    // Destructor
    ~DynamicArray() { delete[] data_; }

    // Copy Constructor (Deep Copy)
    DynamicArray(const DynamicArray& other) : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // Move Constructor (Steal Resources)
    // Takes a non-const rvalue reference
    DynamicArray(DynamicArray&& other) noexcept 
        : size_(other.size_), data_(other.data_) { // Steal pointers/values
        
        // Leave 'other' in a valid but unspecified (empty) state
        other.size_ = 0;
        other.data_ = nullptr;
    }

    // Move Assignment Operator
    DynamicArray& operator=(DynamicArray&& other) noexcept {
        if (this != &other) { // Prevent self-assignment
            delete[] data_;   // Free existing resource
            
            size_ = other.size_; // Steal new resource
            data_ = other.data_;
            
            other.size_ = 0;     // Nullify source
            other.data_ = nullptr;
        }
        return *this;
    }
};
```

### What `std::move` Really Does

> [!IMPORTANT]
> `std::move` DOES NOT MOVE ANYTHING. 
> `std::move` is simply a cast. It unconditionally casts its argument to an rvalue reference (`T&&`), turning an lvalue into an xvalue. This allows it to bind to move constructors or move assignment operators.

```cpp
std::string a = "Hello";
std::string b = std::move(a); // std::move(a) casts 'a' to std::string&&
// Because it's an rvalue, std::string's move constructor is called.
// The move constructor actually does the moving.
```

### Moved-From State

The C++ standard dictates that after an object of a standard library type has been moved from, it is placed in a **valid but unspecified state**.
*   **Valid:** You can safely destroy it or assign a new value to it without undefined behavior.
*   **Unspecified:** You cannot assume its value (e.g., a moved-from `std::string` is usually empty, but the standard doesn't guarantee it).

### `noexcept` and the Strong Exception Guarantee

Notice the `noexcept` on the move operations. This is critical for `std::vector` reallocations. 
If a vector resizes, it needs to move elements to the new buffer. If the move constructor might throw, vector will fall back to copying elements to maintain the **strong exception guarantee** (if an exception occurs during resize, the original vector remains intact). 

> [!TIP]
> Always mark your move constructors and move assignment operators as `noexcept` if they don't allocate memory or call throwing code!

### Rule of Five and Compiler Generation

If you don't define any copy operations, move operations, or a destructor, the compiler will implicitly generate move operations for you (doing a member-wise move).
If you define *any* of the Rule of Five (Destructor, Copy Ctor, Copy Assign, Move Ctor, Move Assign), implicit generation of move operations is suppressed.

---

## 4. Perfect Forwarding

### The Forwarding Problem

Imagine writing a generic wrapper function that passes its arguments to another function.
In C++98, to support both lvalues and rvalues without unnecessary copies, you had to overload your wrapper for every combination of `const T&` and `T&`. For `N` arguments, you needed `2^N` overloads!

### Forwarding References (Universal References)

In C++11, if a template parameter `T` is used in the form `T&&` *exactly*, it becomes a **forwarding reference** (also coined "universal reference" by Scott Meyers).

```cpp
template <typename T>
void wrapper(T&& arg) { // T&& is a forwarding reference!
    target(std::forward<T>(arg));
}
```

A forwarding reference can bind to *anything*: lvalues, rvalues, const, non-const. 
> [!WARNING]
> `T&&` is ONLY a forwarding reference in a deduced context.
> `void foo(std::vector<T>&& v)` -> Rvalue reference (not `T&&` exactly).
> `template<class T> class MyClass { void foo(T&& x); };` -> Rvalue reference (`T` is deduced at class level, not function level).

### Reference Collapsing Rules

How does `T&&` bind to an lvalue? Through **reference collapsing**. If you take a reference to a reference, the compiler collapses them into a single reference based on this rule: *If there is any lvalue reference, it becomes an lvalue reference. Otherwise, it's an rvalue reference.*

*   `T& &`   -> `T&`
*   `T& &&`  -> `T&`
*   `T&& &`  -> `T&`
*   `T&& &&` -> `T&&`

If we pass an `int` lvalue to `wrapper`:
`T` is deduced as `int&`. `T&&` becomes `int& &&`, which collapses to `int&`.
If we pass an `int` rvalue to `wrapper`:
`T` is deduced as `int`. `T&&` becomes `int&&`, which remains `int&&`.

### `std::forward`

`std::forward<T>(arg)` conditionally casts `arg` to an rvalue reference *only if* `arg` was initialized with an rvalue. 
If `arg` was initialized with an lvalue, `std::forward` returns an lvalue reference.

**Implementing `make_unique` as an example:**

```cpp
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    // We use std::forward to perfectly preserve the value category of each argument
    // as it is passed to T's constructor.
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

### `auto&&`

`auto&&` acts exactly like a forwarding reference. It's heavily used in range-based for loops and generic lambdas.

```cpp
auto lambda = [](auto&& arg) {
    target(std::forward<decltype(arg)>(arg));
};
```

---

## 5. Advanced Move Topics

### Move-Only Types

Types that manage exclusive resources cannot be copied, only moved. Examples: `std::unique_ptr`, `std::thread`, `std::fstream`.

```cpp
std::unique_ptr<int> p1 = std::make_unique<int>(10);
// std::unique_ptr<int> p2 = p1; // ERROR: use of deleted copy constructor
std::unique_ptr<int> p3 = std::move(p1); // OK: p1 transfers ownership to p3
```

### `push_back` vs `emplace_back`

*   `push_back(T&&)` takes an object of type `T`. If you pass constructor arguments, a temporary `T` is created, then *moved* into the vector.
*   `emplace_back(Args&&...)` perfectly forwards the arguments directly to the constructor of `T` *in-place* inside the vector's memory. This avoids the temporary object and the move operation entirely.

### Sink Parameters vs Forwarding References

If a function *always* needs to take ownership of a parameter (a "sink" parameter), taking it by value is often the cleanest modern C++ idiom:

```cpp
class Employee {
    std::string name_;
public:
    // Takes 'name' by value, then moves it.
    // If caller passes lvalue: 1 copy, 1 move.
    // If caller passes rvalue: 0 copies, 2 moves.
    Employee(std::string name) : name_(std::move(name)) {}
};
```
Perfect forwarding (`template<typename S> Employee(S&& name)`) is optimal (0 copies, 1 move for rvalue) but severely clutters the interface and can cause issues with overload resolution. Pass-by-value + move is usually preferred for sinks.

---

## 6. Copy Elision (RVO and NRVO)

Copy elision is a compiler optimization where an unnecessary copy/move is completely omitted, constructing the object directly in its final destination.

### RVO (Return Value Optimization) / Mandatory Elision (C++17)

Since C++17, if you return a prvalue (a temporary) of the same type as the return type, copy elision is **mandatory**. No copy or move constructor is required to even exist!

```cpp
std::mutex getMutex() {
    return std::mutex{}; // OK in C++17! std::mutex is non-copyable and non-movable.
    // The mutex is constructed directly in the caller's stack frame.
}
```

### NRVO (Named Return Value Optimization)

If you return a *named* local variable, the compiler *may* elide the copy/move. This is not guaranteed, but virtually all compilers do it.

```cpp
std::string makeString() {
    std::string local = "Hello";
    return local; // NRVO usually happens here. 
}
```

> [!CAUTION]
> NEVER write `return std::move(local);` if returning a local variable of the same type as the return type.
> This forces the compiler to treat it as an xvalue and call the move constructor, completely disabling NRVO! 
> Let the compiler do its job: it will automatically treat it as an rvalue if NRVO cannot be applied.

---

## 7. Common Mistakes and Pitfalls

1.  **Using `std::move` on `const` objects:**
    ```cpp
    const std::string s = "Hello";
    std::string s2 = std::move(s); 
    // std::move(s) returns const std::string&&.
    // This doesn't match the move constructor T(T&&), but it DOES match 
    // the copy constructor T(const T&). It silently copies!
    ```

2.  **Using an object after `std::move`:**
    ```cpp
    std::vector<int> v = {1, 2, 3};
    process(std::move(v));
    v.push_back(4); // Dangerous! 'v' is in an unspecified state.
    ```

3.  **`std::move` in a loop over members:**
    If you `std::move` a member variable multiple times in a loop, you steal it on the first iteration, and subsequent iterations get nothing.

4.  **Perfect forwarding in generic lambdas:**
    Don't forget the `decltype`!
    ```cpp
    auto f = [](auto&& x) { target(std::forward<decltype(x)>(x)); };
    ```

---

## 8. Interview Questions & Detailed Answers

**Q1: What exactly does `std::move` do? Does it move memory?**
**A:** `std::move` does absolutely nothing at runtime. It is purely a compile-time cast. It casts its argument to an rvalue reference (`T&&`). This allows the compiler to select the move constructor or move assignment operator, which are the functions that actually perform the resource transfer (moving).

**Q2: What is the difference between an lvalue and an rvalue?**
**A:** An lvalue represents an object that occupies an identifiable location in memory (it has an address/identity). You can assign to it (if not const). An rvalue represents a temporary value or an object that is about to be destroyed, meaning its resources can safely be scavenged. It typically does not have a persistent memory address.

**Q3: Is a named rvalue reference an lvalue or an rvalue?**
**A:** It is an lvalue. Any variable that has a name is an lvalue, regardless of its type. This is why you must use `std::move(ref)` if you want to forward a named rvalue reference to another function taking an rvalue reference.

**Q4: Will this code compile? `void foo(int&& x); int main() { int a = 5; foo(a); }`**
**A:** No. `foo` expects an rvalue reference, but `a` is an lvalue. Rvalue references cannot bind to lvalues. You would need to call `foo(std::move(a));`.

**Q5: Why should move constructors be marked `noexcept`?**
**A:** Standard library containers like `std::vector` provide the strong exception guarantee. During reallocation (resizing), `std::vector` needs to move elements. If the element's move constructor is not marked `noexcept`, `std::vector` cannot guarantee that an exception won't be thrown midway through the move. Therefore, it will fall back to using the copy constructor instead, severely degrading performance.

**Q6: What is the Rule of Five?**
**A:** If a class requires a user-defined destructor, copy constructor, or copy assignment operator (usually because it manages a raw resource), it almost certainly requires user-defined move constructors and move assignment operators to be efficient. The five are: Destructor, Copy Ctor, Copy Assign, Move Ctor, Move Assign.

**Q7: Explain the concept of "valid but unspecified state".**
**A:** When an object is moved from, the standard dictates it is left in a valid but unspecified state. "Valid" means it maintains its invariants well enough that its destructor can safely run, or it can be assigned a new value without crashing. "Unspecified" means you cannot rely on it having any particular value (e.g., a moved-from string might be empty, but you shouldn't assume its size is 0 without checking).

**Q8: What is the "Forwarding Problem"?**
**A:** It's the problem of writing a generic wrapper function that passes arguments to another function exactly as they were received (preserving lvalueness, rvalueness, and constness) without creating exponential combinations of overloads and without causing unnecessary copies.

**Q9: What makes a template parameter `T&&` a forwarding reference (universal reference) as opposed to a normal rvalue reference?**
**A:** `T&&` is a forwarding reference ONLY when type deduction is taking place directly on `T` at the point of the function call. For example, in `template<typename T> void f(T&& param);`. If `T` is already deduced (e.g., `std::vector<T>&&` or a member function `void push(T&&)` in a class template `vector<T>`), it is just an rvalue reference.

**Q10: What are the reference collapsing rules?**
**A:** When a reference to a reference is formed (usually during template instantiation or typedefs), they collapse. The rule is: `&` + `&` -> `&`, `&` + `&&` -> `&`, `&&` + `&` -> `&`, `&&` + `&&` -> `&&`. Basically, if there's any lvalue reference, it collapses to an lvalue reference.

**Q11: How does `std::forward` work?**
**A:** `std::forward<T>(arg)` looks at the template parameter `T`. If `T` is an lvalue reference (meaning the original argument passed to the forwarding reference was an lvalue), `std::forward` returns an lvalue reference. If `T` is a non-reference or rvalue reference (meaning the original argument was an rvalue), `std::forward` casts `arg` to an rvalue reference, allowing it to be moved.

**Q12: Is it a good idea to write `return std::move(local_variable);`?**
**A:** Almost always NO. This is an anti-pattern. Returning a local variable by value automatically triggers NRVO (Named Return Value Optimization), which elides the copy entirely. If NRVO fails, the compiler implicitly treats the return as a move. Explicitly using `std::move` creates an xvalue, which disables NRVO and forces a move operation instead of elision, pessimizing performance.

**Q13: When *should* you use `std::move` in a return statement?**
**A:** You should only use it when returning a local variable whose type differs from the function's return type (though implicit conversions might handle this), or when returning a member of a local variable (e.g., `return std::move(local_struct.string_member);`), or when returning an object passed in by value as a parameter.

**Q14: What happens if you `std::move` a `const` object?**
**A:** It compiles, but it doesn't move. `std::move(const T)` results in a `const T&&`. This cannot bind to a move constructor `T(T&&)` because of the `const`. Instead, it binds to the copy constructor `T(const T&)`, resulting in a silent copy.

**Q15: What is the difference between `std::vector::push_back` and `emplace_back`?**
**A:** `push_back` takes an already-constructed object (by const lvalue reference or rvalue reference) and copies/moves it into the vector's memory. `emplace_back` takes the *arguments* needed to construct the object and perfectly forwards them to construct the object directly in-place inside the vector's memory, avoiding temporary creation and move operations entirely.

**Q16: Can you have a class that is movable but not copyable?**
**A:** Yes. `std::unique_ptr` and `std::thread` are prime examples. You achieve this by explicitly deleting the copy constructor and copy assignment operator (`= delete;`), while defining or defaulting the move operations.

**Q17: What does `decltype(auto)` do, and when is it useful regarding perfect forwarding?**
**A:** `decltype(auto)` deduces a return type while preserving references. `auto` strips references. If you are writing a perfect forwarding wrapper around a function that returns an `int&`, returning `auto` will return a copy (`int`). Returning `decltype(auto)` will correctly return `int&`.

**Q18: Explain the difference between prvalue, xvalue, and glvalue.**
**A:** 
- **prvalue** (pure rvalue): Has no identity, can be moved from (e.g., `42`, a temporary object).
- **xvalue** (expiring value): Has identity, can be moved from (e.g., the result of `std::move(obj)`).
- **glvalue** (generalized lvalue): Has identity. It encompasses lvalues and xvalues.

**Q19: Analyze this code: `std::string s = "A"; std::vector<std::string> v; v.push_back(std::move(s)); std::cout << s;`**
**A:** The code is valid but the output is unspecified. `s` is moved into the vector. Standard library types are left in a valid but unspecified state after moving. For `std::string`, it will likely print an empty string, but relying on that exact output is poor practice.

**Q20: How do you implement perfect forwarding for a generic constructor?**
**A:** 
```cpp
class Wrapper {
    std::string data;
public:
    template <typename T, typename = std::enable_if_t<!std::is_same_v<std::decay_t<T>, Wrapper>>>
    Wrapper(T&& arg) : data(std::forward<T>(arg)) {}
};
```
Note the SFINAE (`enable_if_t`) to prevent the generic forwarding constructor from accidentally hijacking the copy/move constructors!

---

## Quick Reference / Cheat Sheet

*   **`std::move(x)`**: Casts `x` to an rvalue reference (`T&&`). Use when you are *done* with an object and want to transfer ownership.
*   **`std::forward<T>(x)`**: Casts `x` to an rvalue reference *only if* `x` was initialized with an rvalue. Use inside templates to perfectly forward parameters.
*   **Forwarding Reference**: `T&&` where `T` is a template parameter deduced *at the call site*.
*   **Named Rvalue Reference**: `T&& ref`. `ref` itself is an lvalue!
*   **Move Constructor Signature**: `Class(Class&& other) noexcept;`
*   **Rule of Zero**: If you don't manage raw resources, write NO custom destructors/copy/move operations. The compiler will generate optimal ones.
*   **NRVO Rule**: Never `return std::move(local_variable);` if the types match.

## References
*   Effective Modern C++ by Scott Meyers (Items 23 - 30)
*   C++ Reference: Value Categories (cppreference.com)
*   C++ Reference: `std::move` and `std::forward`

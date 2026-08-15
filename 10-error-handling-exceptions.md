# Module 10 — Error Handling & Exception Safety

Welcome to Module 10. Robust software must deal gracefully with the inevitable: things go wrong. Files vanish, networks partition, memory runs out, and users provide invalid input. C++ provides a multi-faceted approach to error handling, blending the classic C-style approaches with object-oriented exception handling and modern functional paradigms.

In this module, we will explore the mechanisms of exception handling, the critical concept of Exception Safety Guarantees, the foundational role of RAII, and the latest C++ features like `std::expected` (C++23) that provide powerful alternatives to exceptions.

---

## 1. Exception Fundamentals

Exceptions provide a mechanism to decouple the detection of an error from its handling. When a function detects a problem it cannot resolve, it "throws" an exception, which propagates up the call stack until a matching "catch" handler is found.

### The `throw`, `try`, `catch` Mechanism

The core mechanism is built around three keywords:

*   **`throw`**: Used to signal an anomalous situation (an exception).
*   **`try`**: Defines a block of code in which exceptions might be thrown and caught.
*   **`catch`**: Defines a block of code that handles a specific type of exception thrown within the associated `try` block.

```cpp
#include <iostream>
#include <stdexcept>

double divide(double numerator, double denominator) {
    if (denominator == 0.0) {
        // Signal an error by throwing an exception
        throw std::invalid_argument("Division by zero");
    }
    return numerator / denominator;
}

int main() {
    try {
        double result = divide(10.0, 0.0);
        std::cout << "Result: " << result << '\n';
    } catch (const std::invalid_argument& e) {
        // Handle the specific error
        std::cerr << "Error: " << e.what() << '\n';
    } catch (const std::exception& e) {
        // Catch any other standard exceptions
        std::cerr << "Standard exception: " << e.what() << '\n';
    } catch (...) {
        // Catch-all handler (use sparingly)
        std::cerr << "Unknown exception occurred.\n";
    }
    return 0;
}
```

> [!TIP]
> **Catch Order Matters:** Handlers are evaluated in the order they appear. Always order your `catch` blocks from the most derived types to the most base types. If you place `catch (const std::exception&)` before `catch (const std::invalid_argument&)`, the base class handler will catch the `invalid_argument`, and the derived handler will never execute.

### Exception Object Types & The Standard Hierarchy

You can throw any type in C++ (even an `int` or a `char*`), but best practice dictates throwing objects that derive from `std::exception`. The standard library provides a rich hierarchy defined in `<stdexcept>`.

*   **`std::exception`**: The base class. Provides the `virtual const char* what() const noexcept` method.
*   **`std::logic_error`**: Errors resulting from faulty logic (should theoretically be preventable by code inspection).
    *   `std::invalid_argument`
    *   `std::domain_error`
    *   `std::length_error`
    *   `std::out_of_range`
*   **`std::runtime_error`**: Errors that cannot be easily predicted in advance and occur at runtime.
    *   `std::range_error`
    *   `std::overflow_error`
    *   `std::underflow_error`
    *   `std::system_error` (Since C++11)

### Catching: By Value vs. By Reference vs. By Pointer

**Always catch by `const` reference.**

1.  **Catching by Value:** `catch (std::exception e)` causes *object slicing*. If you throw a `std::runtime_error`, catching it by value as `std::exception` slices off the derived parts, losing the overridden `what()` behavior and any derived state.
2.  **Catching by Pointer:** `catch (std::exception* e)` requires allocating the exception on the heap (e.g., `throw new std::runtime_error(...)`), which is dangerous because throwing memory-allocation exceptions during an out-of-memory error will fail. It also forces the catcher to manage the memory (delete the pointer).
3.  **Catching by Reference:** `catch (const std::exception& e)` avoids slicing, avoids unnecessary copies, and utilizes polymorphism correctly.

```cpp
try {
    throw std::runtime_error("Disk full");
} catch (const std::exception& e) { // Correct: Catch by const reference
    std::cerr << e.what(); 
}
```

### Re-throwing Exceptions

If a `catch` block cannot fully handle an exception, or needs to perform some logging before passing it on, it can re-throw the exception.

*   **`throw;`** (Without an operand): Re-throws the *current* exception object exactly as it is, preserving its dynamic type.
*   **`throw e;`**: Throws a *new* exception object, which is a copy of the static type of `e` in the catch block. This leads to object slicing.

```cpp
try {
    // ... something throws std::out_of_range
} catch (const std::exception& e) {
    log_error(e.what());
    throw; // CORRECT: Re-throws the original std::out_of_range
    // throw e; // WRONG: Throws a new, sliced std::exception object
}
```

### Stack Unwinding

When an exception is thrown, the C++ runtime searches up the call stack for a matching `catch` block. During this search, as it exits each function's scope, it invokes the destructors of all fully-constructed local objects in that scope. This process is called **stack unwinding**.

Stack unwinding is the foundation of C++ resource management (RAII), ensuring that locks are released, memory is freed, and files are closed, even when errors occur.

> [!WARNING]
> **Exceptions in Destructors:** If a destructor throws an exception *while stack unwinding is already in progress*, the C++ runtime immediately calls `std::terminate()`, halting the program. Destructors must essentially be `noexcept` (implicitly true since C++11).

### `noexcept` Specifier

The `noexcept` specifier (introduced in C++11, replacing deprecated `throw()` specifications) declares that a function is guaranteed not to throw an exception.

```cpp
void safe_function() noexcept {
    // I promise not to throw
}
```

If a `noexcept` function attempts to propagate an exception outward, the runtime immediately calls `std::terminate()`. `noexcept` is a critical tool for compiler optimization (especially for move constructors).

### Handling the Unhandled

If an exception propagates to the top of the stack (out of `main()` or an unjoined thread) without being caught, `std::terminate()` is called. You can set a custom terminate handler using `std::set_terminate()`.

```cpp
#include <exception>
#include <cstdlib>
#include <iostream>

void my_terminate() {
    std::cerr << "Unhandled exception! Terminating gracefully.\n";
    std::abort();
}

int main() {
    std::set_terminate(my_terminate);
    throw std::runtime_error("Fatal error");
    return 0; // Never reached
}
```

### Transporting Exceptions: `std::exception_ptr`

C++11 introduced `std::exception_ptr`, a smart pointer type that manages an exception object. It allows you to catch an exception in one thread, store it, and rethrow it in another thread.

```cpp
#include <exception>
#include <thread>
#include <iostream>

std::exception_ptr global_exception_ptr = nullptr;

void worker() {
    try {
        throw std::runtime_error("Error in background thread");
    } catch (...) {
        global_exception_ptr = std::current_exception(); // Capture exception
    }
}

int main() {
    std::thread t(worker);
    t.join();

    if (global_exception_ptr) {
        try {
            std::rethrow_exception(global_exception_ptr); // Rethrow in main thread
        } catch (const std::exception& e) {
            std::cout << "Caught from thread: " << e.what() << '\n';
        }
    }
    return 0;
}
```

### `std::nested_exception`

When handling an exception, you sometimes want to wrap it in a new exception to add context, without losing the original stack trace.

```cpp
#include <exception>
#include <iostream>
#include <stdexcept>

void read_file() {
    throw std::runtime_error("File not found");
}

void load_config() {
    try {
        read_file();
    } catch (...) {
        // Wrap the current exception
        std::throw_with_nested(std::runtime_error("Failed to load configuration"));
    }
}

void print_exception(const std::exception& e, int level =  0) {
    std::cerr << std::string(level, ' ') << "exception: " << e.what() << '\n';
    try {
        std::rethrow_if_nested(e);
    } catch(const std::exception& nested_e) {
        print_exception(nested_e, level+1);
    } catch(...) {}
}

int main() {
    try {
        load_config();
    } catch (const std::exception& e) {
        print_exception(e);
    }
}
```

---

## 2. Exception Safety Guarantees

Exception safety is not about "preventing" exceptions; it's about ensuring your program remains in a valid, predictable state when an exception *does* occur. Dave Abrahams defined four levels of exception safety guarantees.

### 1. No-throw (or Nothrow) Guarantee
The strongest guarantee. The operation is guaranteed to succeed and never throw an exception.
*   **Examples:** Built-in types, valid swap operations, standard library move constructors/assignment operators (usually).
*   **Mechanism:** Marked with `noexcept`.

### 2. Strong Exception Guarantee (Commit or Rollback)
If the operation fails (throws), the program state remains exactly as it was before the operation was attempted. It's atomic.
*   **Examples:** `std::vector::push_back` (if the type's copy/move constructor doesn't throw).
*   **Mechanism:** Typically achieved using the "Copy-and-Swap Idiom".

### 3. Basic Exception Guarantee
If the operation fails, the program remains in a valid (though possibly unknown) state, and no resources (memory, file handles) are leaked. Invariants are preserved.
*   **Examples:** Most standard library operations provide this at a minimum.

### 4. No Guarantee (Avoid!)
If an exception occurs, the program is left in an invalid state, resources are leaked, or invariants are violated.

### Writing Exception-Safe Code: The Copy-and-Swap Idiom

The Copy-and-Swap idiom is the gold standard for achieving the Strong Exception Guarantee, particularly in assignment operators.

```cpp
#include <algorithm>
#include <cstddef>

class MyVector {
    int* data_;
    size_t size_;

public:
    // Constructor
    MyVector(size_t n) : data_(new int[n]()), size_(n) {}

    // Destructor
    ~MyVector() { delete[] data_; }

    // Copy constructor (might throw std::bad_alloc)
    MyVector(const MyVector& other) : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    // Nothrow swap
    friend void swap(MyVector& first, MyVector& second) noexcept {
        // Enable ADL (Argument Dependent Lookup)
        using std::swap;
        swap(first.size_, second.size_);
        swap(first.data_, second.data_);
    }

    // Assignment operator providing Strong Exception Guarantee
    MyVector& operator=(MyVector other) { // Note: pass by value! (creates a copy)
        // 'other' is a temporary copy. If copying throws, state of *this is unmodified (Strong Guarantee).
        swap(*this, other); // Nothrow swap.
        return *this;
        // The old data is automatically destroyed when 'other' goes out of scope.
    }
};
```

### Conditional `noexcept`

You can make `noexcept` conditional upon template parameters using the `noexcept(expression)` operator.

```cpp
template <typename T>
void swap_wrapper(T& a, T& b) noexcept(noexcept(std::swap(a, b))) {
    std::swap(a, b);
}
```
This states: `swap_wrapper` is `noexcept` *if and only if* `std::swap(a, b)` is `noexcept`.

---

## 3. RAII and Exception Safety

**Resource Acquisition Is Initialization (RAII)** is the bedrock of C++ exception safety. It relies on a simple rule: *bind the lifecycle of a resource to the lifecycle of a local object.*

When an exception is thrown, stack unwinding guarantees that destructors of local objects are called. Therefore, if resources (memory, locks, file handles) are released in the destructor of a local object, they are guaranteed to be released safely, even during exceptions.

### Why RAII is Essential

Without RAII, you end up with C-style error handling mixed with exceptions, leading to leaks:

```cpp
// BAD: Prone to resource leaks if process() throws
void bad_func() {
    int* data = new int[100];
    process(data); // If this throws, 'delete[] data' is never reached!
    delete[] data;
}

// GOOD: RAII handles the resource
void good_func() {
    std::vector<int> data(100);
    process(data); // If this throws, vector's destructor cleans up the memory.
}
```

### Scope Guards

Sometimes writing a full RAII class for a single localized action is overkill. Scope guards (popularized by Alexandrescu and generic libraries) execute a lambda function upon exiting the scope.

While C++ doesn't have `std::scope_exit` natively yet (expected in `std::experimental`), it's trivial to write:

```cpp
template <typename F>
class ScopeExit {
    F func;
    bool active;
public:
    explicit ScopeExit(F f) : func(std::move(f)), active(true) {}
    ~ScopeExit() { if (active) func(); }
    void release() { active = false; }
    
    // Delete copy/move semantics for safety
    ScopeExit(const ScopeExit&) = delete;
    ScopeExit& operator=(const ScopeExit&) = delete;
};

// Helper function for template deduction
template <typename F>
ScopeExit<F> make_scope_exit(F f) { return ScopeExit<F>(std::move(f)); }

void process_file() {
    FILE* file = fopen("data.txt", "r");
    if (!file) return;

    auto cleanup = make_scope_exit([&] { 
        fclose(file); 
        std::cout << "File closed safely.\n";
    });

    // ... code that might throw ...
    // fclose is guaranteed to be called.
}
```

### `std::uncaught_exceptions()` (C++17)

To create advanced scope guards like `ScopeSuccess` or `ScopeFailure`, you need to know *why* the scope is exiting: normally, or due to an exception. C++17 introduced `std::uncaught_exceptions()`, which returns the number of active exceptions (exceptions currently in flight during stack unwinding).

```cpp
#include <exception>
#include <iostream>

class Transaction {
    int initial_exceptions;
public:
    Transaction() : initial_exceptions(std::uncaught_exceptions()) {}
    ~Transaction() {
        if (std::uncaught_exceptions() > initial_exceptions) {
            std::cout << "Rolling back transaction due to exception.\n";
        } else {
            std::cout << "Committing transaction.\n";
        }
    }
};
```

---

## 4. Error Handling Alternatives

Exceptions are not always the right tool. They have a performance cost (when thrown), they disrupt control flow invisibly, and they are forbidden in many constrained environments (embedded, game development).

### Error Codes

The traditional approach.

```cpp
enum class ErrorCode { Success, InvalidInput, NetworkFailure };

ErrorCode fetch_data(int& out_data) {
    if (/* network fails */ false) return ErrorCode::NetworkFailure;
    out_data = 42;
    return ErrorCode::Success;
}
```
*Drawback*: Callers often forget to check the return value, leading to silent failures.

### `std::error_code` and `std::system_error`

Introduced in C++11, `std::error_code` encapsulates OS-specific error codes (like `errno` or Windows `GetLastError()`) into a standardized interface using `std::error_category`.

```cpp
#include <system_error>
#include <iostream>

void print_error() {
    std::error_code ec = std::make_error_code(std::errc::permission_denied);
    std::cout << "Error: " << ec.message() << " (Category: " << ec.category().name() << ")\n";
}
```

### The Expected Pattern: `std::expected` (C++23)

C++23 introduces `std::expected<T, E>`. It represents an object that contains *either* a value of type `T` (success) *or* an error of type `E` (failure). This is a functional approach to error handling.

> [!IMPORTANT]
> `std::expected` forces the caller to acknowledge the possibility of an error, unlike exceptions which can be silently ignored until they crash the program.

```cpp
#include <expected>
#include <string>
#include <iostream>

// Return either a parsed int, or a string explaining the error
std::expected<int, std::string> parse_int(const std::string& str) {
    try {
        return std::stoi(str);
    } catch (...) {
        return std::unexpected("Failed to parse integer");
    }
}

int main() {
    auto result1 = parse_int("42");
    if (result1) {
        std::cout << "Value: " << *result1 << '\n';
    }

    auto result2 = parse_int("hello");
    if (!result2) {
        std::cerr << "Error: " << result2.error() << '\n';
    }
    return 0;
}
```

### Monadic Operations on `std::expected`

C++23 also provides monadic operations (`and_then`, `transform`, `or_else`) for `std::expected`, allowing chaining of operations without deeply nested `if/else` error checks.

```cpp
// Example of monadic chaining (pseudocode for concept)
// auto final_result = parse_int("42")
//                     .transform([](int x) { return x * 2; })
//                     .and_then(validate_positive)
//                     .or_else(log_error);
```

### `std::optional`

If the only "error" is the absence of a value (e.g., searching for an item that isn't there), `std::optional<T>` (C++17) is the correct choice, rather than throwing an exception or returning pointers.

---

## 5. Modern Error Handling Best Practices

### When to use what?

1.  **Exceptions**: Use for truly *exceptional* events. Disk full, database connection lost, out of memory. If it happens rarely and you can't handle it locally, throw.
2.  **`std::expected`**: Use for expected failures. Parsing user input, looking up a file that might not exist. If failure is a normal part of the domain logic, use expected values.
3.  **`std::optional`**: Use when a function might legitimately return "nothing" (e.g., `find_user_by_id`).
4.  **Error Codes / `std::error_code`**: Use in low-level system programming, C APIs, or strict environments where exceptions are disabled (`-fno-exceptions`).

### Performance Cost of Exceptions

C++ compilers implement the "Zero-Cost Exception Model" (Itanium ABI).
*   **Zero Cost when NOT thrown**: `try` blocks do not add runtime overhead on the "happy path".
*   **High Cost when thrown**: Stack unwinding and RTTI lookups are extremely slow. Throwing an exception is orders of magnitude slower than a simple `return`.
*   **Conclusion**: Do not use exceptions for control flow.

### Exceptions across DLL Boundaries

Throwing exceptions across module/DLL boundaries (e.g., from a Windows DLL to an executable) is highly dangerous. If the DLL and the executable are compiled with different compilers, different standard libraries, or different memory allocators, stack unwinding will corrupt memory and crash. Always catch exceptions at the DLL boundary and convert them to C-style error codes.

### Constructors and Destructors

*   **Constructors**: Throwing from a constructor is the *only* way to signal that object creation failed. Because the object was never fully constructed, its destructor will *not* be called. However, destructors for fully-constructed member variables and base classes *will* be called.
*   **Destructors**: Never, ever throw exceptions out of a destructor. It leads to `std::terminate`.

---

## 6. Assertions and Debugging

Errors fall into two broad categories: runtime errors (the environment failed) and logic errors (the programmer made a mistake). Assertions are used to catch logic errors.

### The `assert` Macro

Defined in `<cassert>`. It evaluates an expression. If it is false, it prints an error message and calls `std::abort()`.

```cpp
#include <cassert>

void process_buffer(const char* buffer, size_t size) {
    assert(buffer != nullptr && "Buffer must not be null");
    assert(size > 0 && "Size must be positive");
    // ...
}
```
**Crucial Point:** `assert` is completely removed in Release builds (when `NDEBUG` is defined). Never put side-effecting code inside an `assert`!

### `static_assert`

Evaluated at *compile time*. Used to enforce conditions on types and constants.

```cpp
static_assert(sizeof(int) >= 4, "int must be at least 32 bits");

template <typename T>
void do_something(T value) {
    static_assert(std::is_integral_v<T>, "T must be an integral type");
}
```

### `[[assume]]` (C++23)

Allows the programmer to tell the compiler optimizer that a certain condition is true. If it's false at runtime, it's Undefined Behavior (UB).

```cpp
void optimize_me(int x) {
    [[assume(x > 0)]]; // Compiler can now optimize assuming x is positive
    // ...
}
```

### `std::unreachable()` (C++23)

Explicitly marks a code path as unreachable. Helps the optimizer. If execution reaches it, it is UB.

```cpp
#include <utility>

enum class Color { Red, Green, Blue };

int get_id(Color c) {
    switch (c) {
        case Color::Red: return 1;
        case Color::Green: return 2;
        case Color::Blue: return 3;
    }
    std::unreachable(); // We know no other colors exist
}
```

---

## 7. Interview Questions & Detailed Answers

**Q1: What happens if an exception is thrown inside a constructor? Does the destructor get called?**
**A:** If a constructor throws, the object is considered "not fully constructed," so its destructor is **not** called. However, stack unwinding guarantees that the destructors for any base classes and member variables that were *already fully constructed* prior to the exception being thrown *will* be called in reverse order of construction. This is why member variables should manage their own resources (RAII).

**Q2: Why should you avoid throwing exceptions from a destructor?**
**A:** Destructors are called during stack unwinding, which occurs when an exception is already in flight. If a destructor throws a second exception while the first is unhandled, the C++ runtime cannot resolve the situation and immediately calls `std::terminate()`, crashing the application.

**Q3: Explain the "Copy-and-Swap" idiom and what exception safety guarantee it provides.**
**A:** It provides the **Strong Exception Guarantee** (commit or rollback). In an assignment operator (`operator=`), you take the parameter by value (creating a temporary copy). If the copy constructor throws, the original object is untouched. If the copy succeeds, you swap the contents of the temporary object with `*this` using a non-throwing `swap` function. When the function exits, the temporary object is destroyed, cleaning up the old data.

**Q4: What is object slicing in the context of exceptions? How do you prevent it?**
**A:** Object slicing occurs when a derived exception object is caught by value as a base class exception type (e.g., `catch (std::exception e)`). The derived parts of the object are sliced off, and polymorphic behavior (like overridden `what()` methods) is lost. Prevent this by always catching by `const` reference: `catch (const std::exception& e)`.

**Q5: What is the difference between `throw;` and `throw e;` inside a catch block?**
**A:** `throw;` (without an operand) re-throws the exact same exception object that was caught, preserving its dynamic type and preventing object slicing. `throw e;` throws a *new* copy of `e` based on the static type of `e` in the catch block, which causes object slicing if `e` is a base reference to a derived object.

**Q6: What is the Zero-Cost Exception Model?**
**A:** It refers to the implementation technique (Itanium ABI) where `try` blocks and the potential to throw exceptions incur zero runtime overhead as long as no exception is actually thrown. The cost is shifted entirely to the "throw" path, making the process of throwing and unwinding the stack relatively slow.

**Q7: How does `std::expected` differ from throwing exceptions?**
**A:** `std::expected<T, E>` represents a value or an error natively in the return type. It does not perform stack unwinding and is fast and predictable. It forces the caller to explicitly handle the possibility of failure in the control flow, unlike exceptions which are invisible in the function signature and can be accidentally ignored.

**Q8: Explain the four levels of Exception Safety Guarantees.**
**A:**
1.  **Nothrow:** Guaranteed never to throw.
2.  **Strong:** If an exception occurs, state is rolled back to exactly what it was before the call.
3.  **Basic:** If an exception occurs, no resources are leaked, invariants are maintained, but state may have changed.
4.  **No Guarantee:** Operations may leak resources or corrupt state if exceptions occur.

**Q9: When should you use `std::optional` vs `std::expected`?**
**A:** Use `std::optional` when the failure mode is simply the absence of a value (e.g., searching for a key in a map). Use `std::expected` when you need to return detailed information about *why* the operation failed (an error code, an error string).

**Q10: What does `noexcept` do, and why is it particularly important for move constructors?**
**A:** `noexcept` specifies that a function promises not to throw exceptions. If a move constructor is marked `noexcept`, standard containers like `std::vector` will use it during reallocation. If it is not `noexcept`, `std::vector` will fall back to using the copy constructor to ensure the Strong Exception Guarantee during reallocation, which degrades performance.

**Q11: Can you catch exceptions across thread boundaries automatically?**
**A:** No. Exceptions are bound to the thread that threw them. To pass an exception to another thread, you must catch it, store it using `std::current_exception()` into a `std::exception_ptr`, pass that pointer to the other thread, and rethrow it using `std::rethrow_exception()`.

**Q12: What is the purpose of `std::terminate()`?**
**A:** It is the function called by the runtime when an exception cannot be handled. This occurs if an exception escapes `main()`, escapes a thread, or is thrown during stack unwinding. It usually calls `std::abort()` to crash the program.

**Q13: Why is putting functional logic inside an `assert()` macro a critical bug?**
**A:** `assert(do_something_important())` will execute normally in Debug builds. However, in Release builds (when `NDEBUG` is defined), the `assert` macro evaluates to an empty statement. The function `do_something_important()` will never be called, completely altering the program's logic in production.

**Q14: Explain `std::uncaught_exceptions()` (C++17).**
**A:** It returns the number of exception objects currently being processed (i.e., the number of exceptions in flight during stack unwinding). It is used to determine if a destructor is being called normally or due to an exception, which is useful for implementing transactional scope guards (Commit on success, Rollback on throw).

**Q15: Why is throwing exceptions across DLL boundaries generally forbidden?**
**A:** Different DLLs might be compiled with different compilers, standard library versions, or memory allocators. An exception object allocated in one module and caught in another might cause ABI mismatch, memory corruption when the catch block tries to free the object, or failure in stack unwinding.

---

## 8. Quick Reference / Cheat Sheet

| Feature | Syntax | Use Case |
| :--- | :--- | :--- |
| **Throw** | `throw std::runtime_error("msg");` | Signal an exceptional failure. |
| **Catch by Ref** | `catch (const std::exception& e)` | Standard way to catch to avoid slicing. |
| **Rethrow** | `throw;` | Propagate the current exception upward. |
| **Nothrow Spec** | `void func() noexcept;` | Guarantee function won't throw. |
| **Exception Ptr** | `std::exception_ptr p;` | Store exception for cross-thread transport. |
| **ScopeGuard** | Custom class w/ lambda | Execute code on scope exit (RAII alternative). |
| **Expected (C++23)**| `std::expected<int, ErrorCode>` | Functional error handling, value OR error. |
| **Optional (C++17)**| `std::optional<int>` | Return a value or "nothing". |
| **Assert** | `assert(ptr != nullptr);` | Debug-only logic checks. |
| **Static Assert** | `static_assert(sizeof(int)==4);`| Compile-time logic checks. |

---

## 9. References

*   *Effective C++* and *More Effective C++* by Scott Meyers
*   *Exceptional C++* by Herb Sutter (for Copy-and-Swap and Exception Safety Guarantees)
*   [cppreference.com - Exception Handling](https://en.cppreference.com/w/cpp/language/exceptions)
*   [cppreference.com - std::expected](https://en.cppreference.com/w/cpp/utility/expected)
*   WG21 (C++ Standards Committee) Papers on Error Handling (P0709 "Zero-overhead deterministic exceptions")

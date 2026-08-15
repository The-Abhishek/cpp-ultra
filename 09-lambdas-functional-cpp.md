# Module 09 — Lambda Expressions & Functional C++

Lambdas were introduced in C++11 and have fundamentally transformed the way C++ is written. They provide a concise, inline syntax for defining anonymous function objects (functors) and are a cornerstone of modern C++, particularly when working with algorithms and functional programming paradigms.

In this comprehensive module, we will explore everything from basic lambda syntax to advanced functional programming patterns, memory footprints, and the evolution of lambdas up through C++23.

---

## 1. Lambda Fundamentals

### The Lambda Syntax
The general syntax of a lambda expression is:
```cpp
[captures](params) mutable noexcept -> return_type { body }
```

Let's break down each component:
- **`[captures]`**: The capture clause. Specifies which variables from the surrounding scope are accessible inside the lambda and whether they are captured by value or reference.
- **`(params)`**: The parameter list. Identical to a normal function's parameter list. If empty, the parentheses can optionally be omitted (though keeping them is common practice).
- **`mutable`**: By default, lambdas captured by value are `const`. The `mutable` keyword allows the lambda body to modify variables captured by value.
- **`noexcept`**: Specifies that the lambda will not throw exceptions. Optional.
- **`-> return_type`**: The trailing return type. Often omitted, allowing the compiler to deduce the return type from the `return` statements in the body.
- **`{ body }`**: The actual code to execute.

### Example: Basic Syntax
```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    auto add = [](int a, int b) -> int {
        return a + b;
    };
    
    std::cout << "Sum: " << add(5, 3) << '\n'; // Outputs 8
    return 0;
}
```

### Implicit vs Explicit Return Types
If you don't specify a trailing return type, the compiler deduces it:
- In C++11, if the body consists of a single `return` statement, the return type is deduced from that statement; otherwise, it's `void`.
- In C++14 and later, the compiler can deduce the return type from multiple `return` statements, provided they all deduce to the same type.

```cpp
auto multiply = [](double a, double b) { return a * b; }; // Deduced as double

auto complex_logic = [](int x) {
    if (x > 0) return true;
    return false; // Both paths return bool, deduces bool
};
```

### Immediately Invoked Lambda Expressions (IIFE)
An IIFE is a lambda that is defined and immediately executed. This is extremely useful for complex `const` initialization.

```cpp
#include <iostream>

int main() {
    int x = 10;
    
    // Instead of deferring initialization, use an IIFE
    const int result = [&]() {
        int temp = x * 2;
        if (temp > 15) return temp - 5;
        return temp;
    }(); // <-- Notice the () at the end
    
    std::cout << "Result: " << result << '\n';
}
```
> [!TIP]
> IIFEs are an excellent way to enforce `const` correctness on variables that require complex logic to initialize.

### Lambda vs Functor (Under the Hood)
A lambda is syntactic sugar for a compiler-generated class (a closure type) that overloads the `operator()`.

When you write:
```cpp
int multiplier = 3;
auto scale = [multiplier](int val) { return val * multiplier; };
```

The compiler essentially generates a class similar to this:
```cpp
class __Lambda_Compiler_Generated {
    int multiplier; // Captured state
public:
    __Lambda_Compiler_Generated(int m) : multiplier(m) {}
    
    // By default, operator() is const
    int operator()(int val) const {
        return val * multiplier;
    }
};

int multiplier = 3;
auto scale = __Lambda_Compiler_Generated(multiplier);
```

### Size of Lambdas
The `sizeof` a lambda depends exactly on its captured state, plus potential padding.

```cpp
#include <iostream>

int main() {
    auto empty_lambda = []() {};
    std::cout << sizeof(empty_lambda) << '\n'; // Usually 1 byte (empty class)

    int a = 5;
    auto capture_one = [a]() {};
    std::cout << sizeof(capture_one) << '\n'; // Usually 4 bytes

    double b = 3.14;
    auto capture_mixed = [a, b]() {};
    std::cout << sizeof(capture_mixed) << '\n'; // Usually 16 bytes (padding for alignment)
}
```

---

## 2. Capture Modes

Captures define the closure's state.

### Capture by Value `[=]`
Captures all used variables from the enclosing scope by copying them. The closure type contains a member variable for each captured variable.
By default, these copies are `const`.

```cpp
int x = 10;
auto print_x = [=]() {
    // x = 20; // ERROR: x is read-only
    std::cout << x << '\n';
};
```

If you need to modify the captured *copies* (without affecting the original variables), use `mutable`.
```cpp
int x = 10;
auto increment_copy = [=]() mutable {
    x++;
    std::cout << "Inside: " << x << '\n'; // 11
};
increment_copy();
std::cout << "Outside: " << x << '\n'; // 10 (original unmodified)
```

### Capture by Reference `[&]`
Captures all used variables by reference. The closure stores references to the original variables.

```cpp
int x = 10;
auto increment_ref = [&]() {
    x++;
};
increment_ref();
std::cout << "Outside: " << x << '\n'; // 11
```

### Mixed Captures
You can specify default captures and then explicitly capture specific variables differently.

- `[=, &x]` : Default by value, `x` by reference.
- `[&, x]` : Default by reference, `x` by value.

```cpp
int a = 1, b = 2, c = 3;
auto mixed = [=, &a]() {
    a++; // Allowed (ref)
    // b++; // Error (const value)
    return a + b + c;
};
```

### Capturing `this` and `*this`
Inside a member function, capturing `[=]` or `[&]` implicitly captures the `this` pointer by value. This can be dangerous if the lambda outlives the object.

```cpp
struct MyStruct {
    int data = 42;
    auto get_lambda() {
        return [=]() { return data; }; // Captures `this` implicitly
    }
};
```
In C++17, you can explicitly capture a copy of the entire object using `[*this]`.

```cpp
struct MyStruct {
    int data = 42;
    auto get_safe_lambda() {
        return [*this]() { return data; }; // Captures a COPY of the object
    }
};
```

### Init Captures / Generalized Captures (C++14)
C++14 introduced generalized captures, allowing you to declare and initialize new variables within the capture clause. This is crucial for capturing move-only types like `std::unique_ptr`.

```cpp
#include <memory>
#include <iostream>

int main() {
    auto ptr = std::make_unique<int>(100);
    
    // ptr is moved into the lambda's state variable 'p'
    auto process = [p = std::move(ptr)]() {
        std::cout << *p << '\n';
    };
    
    process();
    // std::cout << *ptr; // Undefined Behavior: ptr is empty
}
```

### Capturing Structured Bindings (C++20)
In C++17, you could not capture structured bindings. C++20 fixed this.

```cpp
#include <tuple>
#include <iostream>

int main() {
    std::tuple<int, double> t{42, 3.14};
    auto [i, d] = t;
    
    // C++20 allows capturing i and d
    auto l = [i, d]() { std::cout << i << " " << d << '\n'; };
    l();
}
```

### Dangling Capture References (Common Bug!)
> [!WARNING]
> If a lambda captures by reference and outlives the scope of the captured variables, invoking the lambda results in Undefined Behavior (dangling reference).

```cpp
#include <functional>

std::function<int()> get_dangling() {
    int local_var = 10;
    return [&]() { return local_var; }; // BUG: returning reference to local variable
}
// Calling get_dangling()() leads to UB.
```

### Capture Pitfalls with Loops
When lambdas are created inside loops and deferred for later execution, capturing by reference can cause all lambdas to observe the final state of the loop variable.

```cpp
#include <vector>
#include <functional>
#include <iostream>

int main() {
    std::vector<std::function<void()>> funcs;
    for (int i = 0; i < 3; ++i) {
        // [=] creates a copy of i for each lambda.
        // [&] would result in all lambdas printing 3 (or UB if i goes out of scope).
        funcs.push_back([i]() { std::cout << i << " "; });
    }
    for (auto& f : funcs) f(); // Prints 0 1 2
}
```

---

## 3. Generic Lambdas (C++14)

C++14 introduced generic lambdas by allowing `auto` in the parameter list. The compiler implements this using a template `operator()`.

```cpp
auto print = [](const auto& val) {
    std::cout << val << '\n';
};

print(5);       // Instantiates print.operator()<int>
print("Hello"); // Instantiates print.operator()<const char[6]>
```

### Variadic Generic Lambdas
You can combine `auto` with parameter packs.

```cpp
auto print_all = [](auto&&... args) {
    (std::cout << ... << args) << '\n'; // C++17 fold expression
};

print_all(1, " ", 2.5, " ", "Test");
```

### Interaction with Templates
Under the hood, a generic lambda:
```cpp
auto gen = [](auto x) { return x; };
```
Becomes:
```cpp
struct __Generic_Lambda {
    template <typename T>
    auto operator()(T x) const { return x; }
};
```

---

## 4. Lambda Features by Standard

A quick history of lambdas across C++ versions:

### C++11
- Basic lambda syntax.
- Deduce return type for single-statement bodies.

### C++14
- Generic lambdas (`auto` parameters).
- Generalized captures / Init captures (`[x = expr]`).
- Return type deduction for multi-statement bodies.

### C++17
- `constexpr` lambdas (can be used at compile time).
- Explicit `[*this]` capture.

```cpp
constexpr auto add = [](int a, int b) { return a + b; };
static_assert(add(2, 3) == 5, "Math is broken!");
```

### C++20
- Template syntax for lambdas (`[]<typename T>(T x) { ... }`), useful for extracting type information.
- Lambdas in unevaluated contexts (e.g., `decltype`).
- Default constructible and assignable stateless lambdas.

```cpp
// C++20 template lambda
auto generic_vector_processor = []<typename T>(std::vector<T>& vec) {
    T temp = vec.front();
    // ...
};
```

### C++23
- Static lambdas (`static operator()`). Cannot have captures.
- Deducing `this` in lambdas.

```cpp
// C++23 static lambda
auto f = [] static { return 42; }; // No 'this' pointer generated
```

---

## 5. `std::function`

`std::function` (from `<functional>`) is a general-purpose, polymorphic function wrapper. It can store, copy, and invoke any *Callable* target: functions, lambda expressions, bind expressions, or other function objects.

### Type Erasure Mechanism
`std::function` uses a technique called **Type Erasure**. It hides the actual type of the callable object and exposes only the signature.

```cpp
#include <functional>
#include <iostream>

void do_something(const std::function<void(int)>& callback) {
    callback(42);
}

int main() {
    do_something([](int x) { std::cout << x << '\n'; });
}
```

### Performance Overhead
> [!WARNING]
> `std::function` comes with performance overhead:
> 1. **Heap Allocation:** If the stored closure is larger than what Small Object Optimization (SOO) allows (typically around 16-32 bytes), `std::function` will allocate memory on the heap.
> 2. **Virtual Dispatch / Indirect Call:** Invoking the target involves indirect function calls (often through function pointers or vtables), preventing compiler inlining.

### When to use `std::function` vs Templates
- **Use Templates (e.g., `template<typename F> void call(F f)`)**: When performance is critical, when the function is short and inlineable, and you don't need to store the callable in a homogeneous container.
- **Use `std::function`**: When you need to store callables across API boundaries (where templates aren't an option), or when storing different types of closures in a container (e.g., `std::vector<std::function<...>>`).

### `std::move_only_function` (C++23)
`std::function` requires its target to be copyable. This means it cannot store a lambda that captures a `std::unique_ptr`.
C++23 introduces `std::move_only_function`, which only requires the target to be movable.

```cpp
#include <memory>
#include <functional>

int main() {
    auto ptr = std::make_unique<int>(10);
    auto l = [p = std::move(ptr)]() { return *p; };
    
    // std::function<int()> f = std::move(l); // ERROR: l is not copyable
    
    // C++23 move_only_function handles this:
    // std::move_only_function<int()> mf = std::move(l); // OK
}
```

---

## 6. Functional Programming in C++

C++ supports functional paradigms through higher-order functions (functions taking/returning functions).

### `std::bind` and `std::placeholders`
Historically used to partially apply functions.

```cpp
#include <functional>
#include <iostream>

void print_diff(int a, int b) { std::cout << a - b << '\n'; }

int main() {
    using namespace std::placeholders;
    auto subtract_5 = std::bind(print_diff, _1, 5);
    subtract_5(15); // Outputs 10
}
```

**Why lambdas are better than `std::bind`**:
Lambdas are almost always superior to `std::bind`. They are more readable, easier to debug, compile faster, and can be inlined by the compiler (whereas `std::bind` often creates an opaque wrapper).

```cpp
// Lambda equivalent (Preferred)
auto subtract_5_lambda = [](int a) { print_diff(a, 5); };
```

### `std::invoke` (C++17)
Uniformly calls any callable (function pointer, lambda, member function pointer, functor) with given arguments.

```cpp
#include <functional>

struct Foo {
    void print(int i) const { std::cout << i << '\n'; }
};

int main() {
    Foo foo;
    std::invoke(&Foo::print, foo, 42); // Calls foo.print(42)
}
```

### Currying and Partial Application
You can achieve currying (translating a function taking multiple arguments into a sequence of functions taking single arguments) using nested lambdas.

```cpp
auto add = [](int a) {
    return [a](int b) {
        return a + b;
    };
};

int sum = add(5)(10); // 15
```

### Map, Filter, Reduce with Ranges (C++20)
C++20 Ranges library brings elegant functional data pipelines to C++.

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <ranges>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6};

    auto even = [](int i) { return i % 2 == 0; };
    auto square = [](int i) { return i * i; };

    // Filter and Map
    auto result = v | std::views::filter(even)
                    | std::views::transform(square);

    for (int i : result) {
        std::cout << i << " "; // 4 16 36
    }
}
```

---

## 7. Advanced Lambda Patterns

### Recursive Lambdas
A lambda cannot capture itself by `auto` because its type isn't fully defined at the time of capture.

**Approach 1: `std::function`** (High overhead)
```cpp
std::function<int(int)> factorial = [&](int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
};
```

**Approach 2: Passing itself via generic lambda** (Zero overhead)
```cpp
auto factorial = [](auto&& self, int n) -> int {
    return n <= 1 ? 1 : n * self(self, n - 1);
};
std::cout << factorial(factorial, 5); // 120
```

**Approach 3: Deducing `this` (C++23)** (The cleanest solution)
```cpp
auto factorial = [](this auto const& self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
std::cout << factorial(5); // 120
```

### The Overloaded Lambda Pattern (for `std::visit`)
A brilliant pattern combining parameter pack expansion and `using` declarations to create inline visitors for `std::variant`.

```cpp
#include <variant>
#include <iostream>

// The overload pattern struct
template<class... Ts> struct overload : Ts... { using Ts::operator()...; };

// Explicit deduction guide (not needed in C++20, but good for C++17)
template<class... Ts> overload(Ts...) -> overload<Ts...>;

int main() {
    std::variant<int, float, std::string> var = "Hello";

    std::visit(overload{
        [](int i) { std::cout << "int: " << i << '\n'; },
        [](float f) { std::cout << "float: " << f << '\n'; },
        [](const std::string& s) { std::cout << "string: " << s << '\n'; }
    }, var);
}
```

### Lambda and Perfect Forwarding
If a lambda accepts forwarding references (`auto&&`), you should use `std::forward` via `decltype`.

```cpp
auto forward_wrapper = [](auto&& arg) {
    target_function(std::forward<decltype(arg)>(arg));
};
```
In C++20, template syntax makes this cleaner:
```cpp
auto forward_wrapper = []<typename T>(T&& arg) {
    target_function(std::forward<T>(arg));
};
```

---

## Interview Questions & Answers

1. **What is a lambda expression in C++ and how is it implemented by the compiler?**
   **Answer:** A lambda is an inline, anonymous function object (functor). The compiler implements it by generating a unique, unnamed class (a closure type) that overrides `operator()`. Captures are implemented as member variables of this generated class, and the constructor initializes them.

2. **What is the difference between `[=]` and `[&]`?**
   **Answer:** `[=]` captures all used variables from the surrounding scope by value (creating copies). `[&]` captures all used variables by reference. 

3. **What does the `mutable` keyword do in a lambda?**
   **Answer:** By default, lambdas captured by value are `const`, meaning you cannot modify the copied variables inside the lambda body. `mutable` removes the `const` qualification from the generated `operator()`, allowing modifications to the captured values.

4. **Explain the dangling reference problem with lambdas.**
   **Answer:** If a lambda captures variables by reference (`[&]` or `[&x]`) and the lambda's execution outlives the scope of the captured variables (e.g., returning the lambda from a function), invoking it will access destroyed memory, resulting in Undefined Behavior.

5. **How do you capture a `std::unique_ptr` in a lambda?**
   **Answer:** Since `std::unique_ptr` is move-only, it cannot be captured by value `[=]`. You must use C++14 generalized/init captures to move it: `[ptr = std::move(my_ptr)]() { ... }`.

6. **What is the difference between capturing `[=]` in a member function in C++11 vs `[*this]` in C++17?**
   **Answer:** In C++11, `[=]` implicitly captures the `this` pointer by value, meaning the lambda still relies on the original object's lifetime. C++17's `[*this]` captures a *copy* of the entire object, allowing the lambda to be safely executed even if the original object is destroyed.

7. **How does `std::function` differ from a template parameter for accepting callbacks?**
   **Answer:** `std::function` uses type erasure to store any callable with a specific signature, which involves heap allocations (for large closures) and virtual/indirect dispatch. Templates resolve the exact type at compile time, allowing for inline optimizations and zero-overhead execution, but they cannot be easily stored in heterogeneous containers.

8. **What is an Immediately Invoked Lambda Expression (IIFE) and why use it?**
   **Answer:** An IIFE is a lambda that is defined and immediately executed (e.g., `[&](){ return 5; }()`). It is primarily used to perform complex initialization logic for `const` variables, ensuring immutability without deferring initialization or creating helper functions.

9. **Can a lambda capture itself? How do you write a recursive lambda?**
   **Answer:** A lambda cannot capture itself directly using `auto` because its type isn't deduced until after the definition. You can write recursive lambdas using `std::function`, passing the lambda to itself as an `auto` parameter (C++14), or by using the Deducing `this` feature (C++23) with `[](this auto const& self) { ... }`.

10. **What are generic lambdas (C++14)?**
    **Answer:** Generic lambdas use `auto` in their parameter lists. The compiler implements this by making the generated class's `operator()` a templated member function.

11. **Why is `std::bind` considered mostly obsolete in modern C++?**
    **Answer:** Lambdas provide the same partial application functionality but are vastly more readable, easier to step through in a debugger, and more easily optimized/inlined by the compiler. `std::bind` often creates convoluted compiler errors and opaque runtime wrappers.

12. **What does the C++20 syntax `[]<typename T>(std::vector<T>& v)` achieve?**
    **Answer:** It introduces template parameter lists to lambdas. This allows you to explicitly name and extract types (like `T`) from the arguments without needing `decltype`, making generic lambda code much cleaner and easier to constrain (e.g., with Concepts).

13. **Explain the Overloaded Lambda pattern using `std::visit`.**
    **Answer:** It's a technique to handle `std::variant` cleanly. You define a struct that inherits from multiple lambdas (using a variadic template parameter pack) and brings their `operator()` into scope via `using`. You then pass an aggregate initialization of this struct containing inline lambdas to `std::visit`.

14. **What is the default capture of global variables in a lambda?**
    **Answer:** Global variables, static local variables, and `constexpr` variables are not captured. They are accessed directly by the lambda since they have static storage duration and their lifetimes span the entire program.

15. **What is a `static` lambda in C++23?**
    **Answer:** A lambda prefixed with `static` (e.g., `[] static { }`) guarantees that it captures no state. The compiler generates a static `operator()` instead of a member function, meaning there is no hidden `this` pointer for the closure object itself, resulting in slightly better performance for stateless callables.

16. **How does the size of a lambda change with different captures?**
    **Answer:** The size is determined by the size of the captured state. An empty capture `[]` typically results in a 1-byte closure. Capturing pointers or references typically adds 8 bytes each (on 64-bit systems), and capturing large objects by value adds the size of those objects.

---

## Quick Reference / Cheat Sheet

| Feature | Syntax / Example | Notes |
|---------|------------------|-------|
| Empty Lambda | `[](){}` | Does nothing. |
| Value Capture | `[=](){}` | Copies all used local variables. |
| Ref Capture | `[&](){}` | References all used local variables. |
| Mixed Capture | `[=, &x](){}` | Default by value, `x` by ref. |
| Mutable | `[x]() mutable { x++; }` | Allows modifying value-captured copies. |
| Init Capture | `[p = std::move(ptr)](){}`| C++14. Moves objects into lambda state. |
| Generic Lambda | `[](auto x){}` | C++14. `x` can be any type. |
| Template Lambda| `[]<typename T>(T x){}` | C++20. Explicit type extraction. |
| Static Lambda | `auto f = [] static {};` | C++23. No state, no closure `this`. |

---

## References & Further Reading
- [cppreference.com - Lambda expressions](https://en.cppreference.com/w/cpp/language/lambda)
- [cppreference.com - std::function](https://en.cppreference.com/w/cpp/utility/functional/function)
- *Effective Modern C++* by Scott Meyers (Items 31-34 focus specifically on Lambdas)
- *C++ Primer, 5th Edition* by Stanley B. Lippman

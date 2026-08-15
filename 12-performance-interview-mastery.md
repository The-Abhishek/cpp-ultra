# Module 12 — Performance, Optimization & Interview Mastery

Welcome to the final, capstone module. C++ is often chosen when performance is the primary requirement. Writing optimal C++ goes beyond knowing the language features; it requires understanding the hardware, the compiler, and idiomatic optimization techniques. 

This module consists of two massive parts:
1. **Performance & Optimization**: The definitive guide to making C++ run blisteringly fast.
2. **Comprehensive Interview Question Bank**: 100+ categorized questions to ace any senior C++ interview.

---

## Part 1: Performance & Optimization

### 1. Memory & Cache

In modern computing, the CPU is orders of magnitude faster than main memory (RAM). Understanding the cache hierarchy is the single most important skill for a high-performance C++ developer.

#### CPU Cache Hierarchy
- **L1 Cache**: Extremely fast (typically 3-4 cycles latency), small (32KB-64KB per core).
- **L2 Cache**: Fast (~10-12 cycles latency), medium size (256KB-1MB per core).
- **L3 Cache**: Slower (~40-70 cycles latency), shared among all cores on a die (8MB-64MB).
- **Main Memory (RAM)**: Slow (100+ cycles).

#### Cache Lines and Cache Misses
Data is fetched from main memory into the cache in chunks called **cache lines** (typically 64 bytes). If your CPU requests data that isn't in the cache, it's a **cache miss**, and the CPU stalls waiting for main memory.

> [!TIP]
> **Golden Rule of Performance:** Contiguous memory access is king. Algorithms traversing arrays or vectors linearly will almost always outperform node-based data structures (like linked lists or trees) due to hardware prefetching and cache locality.

#### Data-Oriented Design vs Object-Oriented Design
Object-Oriented Design (OOD) groups data by entity (e.g., a `Particle` class with `position`, `velocity`, `color`, `lifetime`). This results in an **Array of Structures (AoS)**.

Data-Oriented Design (DOD) groups data by how it's processed, resulting in a **Structure of Arrays (SoA)**.

```cpp
// Array of Structures (AoS) - Bad for cache if we only want to update positions
struct ParticleAoS {
    float x, y, z;
    float vx, vy, vz;
    uint32_t color;
    float lifetime;
};
std::vector<ParticleAoS> particles; // Size: 32 bytes per particle

// Structure of Arrays (SoA) - Cache friendly!
struct ParticleSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<uint32_t> color;
    std::vector<float> lifetime;
};
```
When iterating over `SoA` to update positions, the CPU cache line is filled completely with `x` values, wasting no space on `color` or `lifetime`.

#### False Sharing in Multithreading
When multiple threads modify independent variables that reside on the **same cache line**, the CPU must invalidate the cache line across cores, causing a massive performance hit.

```cpp
struct Counters {
    std::atomic<int> thread1_counter;
    std::atomic<int> thread2_counter; // Likely on the same 64-byte cache line
};
```
**Fix using Memory Alignment:**
```cpp
struct alignas(64) PaddedCounters {
    std::atomic<int> thread1_counter;
    alignas(64) std::atomic<int> thread2_counter; // Forced to next cache line
};
```

---

### 2. Compiler Optimizations

You must treat the compiler as your partner. 

#### Optimization Levels
- `-O0`: No optimization. Best for debugging.
- `-O1`, `-O2`: Standard optimizations. `-O2` is the default for release builds.
- `-O3`: Aggressive optimizations (loop unrolling, vectorization). Can increase binary size.
- `-Os`: Optimize for size.
- `-Ofast`: `-O3` plus unsafe math optimizations (breaks IEEE 754 strictness).

#### Inlining & Loop Unrolling
**Inlining** replaces a function call with the function body, removing call overhead and allowing the compiler to optimize the code in context.
**Loop Unrolling** reduces loop branching overhead by executing multiple iterations per loop cycle.

#### Vectorization (SIMD)
Single Instruction, Multiple Data (SIMD) allows executing the same operation on multiple data points simultaneously (e.g., adding four pairs of floats in one instruction via AVX/SSE).
The compiler does this automatically at `-O3` if data is contiguous and there are no aliasing dependencies (pointers pointing to the same memory).

#### Advanced Optimizations
- **Link-Time Optimization (LTO)**: Normally, files are compiled independently. LTO allows the linker to optimize across translation units (e.g., cross-file inlining). Use `-flto`.
- **Profile-Guided Optimization (PGO)**: Compile with profiling enabled, run the app with typical workloads, then recompile using the profile data so the compiler knows which branches are hot.
- **Devirtualization**: The compiler figures out the exact dynamic type of a polymorphic object and replaces a virtual function call with a direct call (often enabling inlining).

---

### 3. Profiling Tools

> [!WARNING]
> Never optimize without measuring first. Premature optimization is the root of all evil.

- **perf**: Linux tool for CPU profiling. `perf record ./my_app`, `perf report`.
- **Valgrind (Callgrind / Massif)**: Callgrind profiles function calls and cache misses. Massif profiles heap memory allocation.
- **Google Benchmark**: The standard library for microbenchmarking.

**Microbenchmarking Pitfalls**: The compiler might optimize away your benchmark entirely if it sees the result isn't used. Use `benchmark::DoNotOptimize()`.

---

### 4. Code-Level Optimizations

#### Move Semantics
Avoid copying resources. `std::move` casts an lvalue to an rvalue, allowing resources to be "stolen" rather than cloned.

#### Vector optimizations
Always use `reserve()` if you know the approximate size of a vector to prevent reallocations. Use `emplace_back` to construct objects in-place instead of `push_back` (which constructs then moves/copies).

#### String Optimizations
- **Small String Optimization (SSO)**: `std::string` has a small internal buffer (typically 15-22 bytes). Short strings avoid heap allocation entirely!
- **`std::string_view` (C++17)**: A non-owning, read-only view of string data. Pass `std::string_view` by value instead of `const std::string&` to avoid allocating `std::string` objects when passing string literals.

#### Branch Prediction & `[[likely]]` (C++20)
CPUs pipeline instructions. If a branch (if statement) is mispredicted, the pipeline is flushed, wasting 10-20 cycles.
C++20 introduces attributes to guide the compiler (and branch predictor):
```cpp
if (val == 0) [[unlikely]] {
    handle_error();
} else [[likely]] {
    process(val);
}
```

---

### 5. Compile-Time Performance

Long compilation times kill productivity.
1. **Forward Declarations**: Use `class MyClass;` in headers instead of `#include "MyClass.h"` when you only need pointers/references.
2. **Pimpl Idiom (Pointer to Implementation)**: Hide private members behind an opaque pointer to minimize header dependencies and ABI breakage.
3. **extern template**: Prevent implicit template instantiation in multiple translation units.
4. **C++20 Modules**: Replaces `#include`. Modules are parsed once and imported as binary interfaces, drastically speeding up builds.

---

## Part 2: Comprehensive Interview Question Bank

*Note: This bank contains representative high-yield questions for elite C++ engineering interviews.*

### 1. Language Fundamentals

**Q1: [Easy] What is the size of an empty class in C++ and why?**
**Answer:** The size is `1` byte. The C++ standard mandates that every object must have a unique memory address so that pointers to two distinct objects of the same type are never equal. If the size were 0, arrays of empty classes would have elements with identical addresses.

**Q2: [Medium] Explain struct alignment and padding. What is the size of this struct?**
```cpp
struct Data {
    char a;
    int b;
    char c;
};
```
**Answer:** Typically `12` bytes. Data members are aligned according to their size.
- `a` is 1 byte, followed by 3 bytes of padding to align `b` on a 4-byte boundary.
- `b` is 4 bytes.
- `c` is 1 byte, followed by 3 bytes of padding so that the total size of the struct is a multiple of its largest alignment requirement (4 bytes) for arrays.
*Follow-up: How to optimize this?* Rearrange members from largest to smallest: `int b; char a; char c;` -> Size becomes 8 bytes.

**Q3: [Hard] What is the "Strict Aliasing Rule" and how does violating it cause Undefined Behavior?**
**Answer:** The strict aliasing rule states that a pointer to one type cannot be safely cast to a pointer of a fundamentally different type and dereferenced. The compiler assumes that pointers of different types (e.g., `float*` and `int*`) do not point to the same memory (they don't alias). If you cast a `float*` to `int*` to read its bits, the compiler might optimize out memory reads, leading to UB. Use `std::bit_cast` (C++20) or `memcpy` instead.

### 2. Pointers & Memory

**Q4: [Medium] Explain how `std::shared_ptr` works internally. What is a control block?**
**Answer:** A `shared_ptr` typically contains two pointers: one to the managed object, and one to a dynamically allocated "control block". The control block contains:
1. The strong reference count.
2. The weak reference count.
3. Custom deleter/allocator (if any).
When copying a `shared_ptr`, the strong count is incremented atomically. When the strong count hits 0, the object is destroyed. When both strong and weak hit 0, the control block is destroyed.

**Q5: [Hard] Why should you use `std::make_shared` instead of `std::shared_ptr<T>(new T())`?**
**Answer:** 
1. **Performance/Memory:** `make_shared` allocates a single block of memory for *both* the object and the control block. The explicit `new` does two allocations (one for `T`, one for the control block). Cache locality is better with `make_shared`.
2. **Exception Safety (pre-C++17):** In `foo(std::shared_ptr<T>(new T()), thrower())`, if `new T()` succeeds but `thrower()` throws before the `shared_ptr` is constructed, `T` is leaked.

**Q6: [Expert] How do you resolve cyclic dependencies with smart pointers?**
**Answer:** Use `std::weak_ptr`. A `weak_ptr` observes a `shared_ptr` without incrementing the strong reference count. If Object A has a `shared_ptr` to Object B, and B needs a reference to A, B should hold a `weak_ptr<A>`. To use B's reference, call `lock()` on the `weak_ptr` to temporarily acquire a `shared_ptr`.

### 3. OOP

**Q7: [Medium] Explain how virtual functions are implemented (vtable / vptr).**
**Answer:** Compilers implement dynamic dispatch using Virtual Method Tables (vtables). For every class containing virtual functions, the compiler creates a static array of function pointers (the vtable). Every object of that class gets a hidden pointer (the vptr) pointing to the vtable. When a virtual function is called via a base class pointer, the code looks up the vptr, indexes into the vtable, and calls the derived function. This adds a memory overhead (one pointer per object) and performance overhead (pointer indirection).

**Q8: [Hard] Can you call a virtual function from a constructor or destructor?**
**Answer:** Yes, but it will **not** exhibit polymorphic behavior. During the base class constructor, the object is technically still a base class type (the derived part hasn't been constructed yet). The vptr points to the base class vtable. The same applies during destruction.

**Q9: [Expert] What is Object Slicing and how to prevent it?**
**Answer:** Slicing occurs when an object of a derived class is assigned to an instance of a base class by value.
```cpp
Derived d;
Base b = d; // Slicing! Derived specific members are stripped.
```
Only the base portion is copied. Polymorphism is lost. To prevent it, always pass polymorphic objects by reference or pointer, or delete the copy constructor in the base class.

### 4. Templates

**Q10: [Hard] What is SFINAE? Give a practical example.**
**Answer:** "Substitution Failure Is Not An Error". When resolving overloaded function templates, if substituting a template parameter results in invalid code, the compiler silently removes that overload from the candidate list rather than throwing a hard error.
*Example:* `std::enable_if_t` uses SFINAE to enable a template only for integer types:
```cpp
template <typename T>
std::enable_if_t<std::is_integral_v<T>> process(T t) { /* ... */ }
```

**Q11: [Medium] What is CRTP (Curiously Recurring Template Pattern)?**
**Answer:** A pattern where a class derives from a base class template parameterized by the derived class itself.
```cpp
template <typename Derived>
struct Base {
    void interface() { static_cast<Derived*>(this)->implementation(); }
};
struct MyClass : Base<MyClass> {
    void implementation() { /* ... */ }
};
```
It achieves static polymorphism (resolving "virtual" calls at compile time) without vtable overhead.

### 5. STL

**Q12: [Medium] What is iterator invalidation? Name cases for `std::vector` and `std::unordered_map`.**
**Answer:** Pointers or iterators to elements become dangling because the container restructured its internal memory.
- `std::vector`: Invalidates on `push_back`/`insert` if capacity is exceeded (reallocation). Erasing invalidates iterators at and after the point of erasure.
- `std::unordered_map`: Re-hashing invalidates all iterators. Inserting never invalidates references, but may invalidate iterators if a rehash occurs.

**Q13: [Hard] How does `std::unordered_map` work internally?**
**Answer:** It's a hash table resolving collisions typically using chaining (linked lists or buckets). The hash of the key determines the bucket index. Operations are average `O(1)`, worst case `O(N)` if all keys hash to the same bucket. Cache locality is generally poor compared to sorted `std::vector`s or flat maps due to node-based memory allocation per element.

### 6. Modern C++ (C++11/14/17/20)

**Q14: [Medium] What is the difference between `std::move` and `std::forward`?**
**Answer:** 
- `std::move` unconditionally casts an expression to an rvalue reference (`T&&`), signifying the resource can be moved.
- `std::forward` conditionally casts to an rvalue reference *only if* the argument was passed as an rvalue. It is used in template perfect forwarding to preserve the value category (lvalue vs rvalue) of the argument.

**Q15: [Hard] Explain auto type deduction rules vs template type deduction.**
**Answer:** `auto` uses the exact same deduction rules as templates, with one exception: `auto` initializes `std::initializer_list` when using braces, whereas templates do not.
```cpp
auto x = {1, 2, 3}; // x is std::initializer_list<int>
template<typename T> void f(T t);
f({1, 2, 3}); // Error: cannot deduce T
```

### 7. Multithreading

**Q16: [Medium] What is a deadlock? How do you avoid it?**
**Answer:** A deadlock occurs when two or more threads are blocked forever, each waiting on a resource held by another. Avoidance strategies:
1. Always acquire multiple locks in the same, strict order globally.
2. Use `std::lock(m1, m2)` or `std::scoped_lock` (C++17) which safely locks multiple mutexes simultaneously without deadlock.

**Q17: [Hard] Explain the difference between `std::memory_order_relaxed`, `acquire`, `release`, and `seq_cst`.**
**Answer:** Defines synchronization rules for atomics.
- `relaxed`: No synchronization. Guarantees only atomicity of the operation.
- `acquire`: Reads prevent subsequent memory accesses from being reordered before the read.
- `release`: Writes prevent prior memory accesses from being reordered after the write.
- `seq_cst` (Sequential Consistency): Default. Enforces a single global total order of all atomic operations across all threads.

**Q18: [Hard] What is a spurious wakeup?**
**Answer:** A thread blocked on a `std::condition_variable::wait()` may wake up without anyone explicitly calling `notify_one()` or `notify_all()`. This is an OS/hardware artifact. To handle it, always wrap the `wait` in a `while` loop checking the condition, or use the lambda overload of `wait()`.

### 8. Output Prediction

**Q19: [Medium] Predict the output:**
```cpp
struct A {
    A() { std::cout << "A"; }
    ~A() { std::cout << "a"; }
};
struct B : A {
    B() { std::cout << "B"; }
    ~B() { std::cout << "b"; }
};
int main() {
    A* obj = new B();
    delete obj;
}
```
**Answer:** Output: `ABa`.
Explanation: Constructor of A is called, then B. When `delete obj` is called, because the destructor of `A` is **not virtual**, it only calls `A`'s destructor, leaking the `B` portion. (UB in standard, but practically behaves this way).

### 9. Coding Challenges

**Q20: [Expert] Implement a basic `unique_ptr` from scratch.**
```cpp
template <typename T>
class UniquePtr {
private:
    T* ptr;
public:
    explicit UniquePtr(T* p = nullptr) : ptr(p) {}
    ~UniquePtr() { delete ptr; }
    
    // Delete copy semantics
    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;
    
    // Implement move semantics
    UniquePtr(UniquePtr&& other) noexcept : ptr(other.ptr) {
        other.ptr = nullptr;
    }
    UniquePtr& operator=(UniquePtr&& other) noexcept {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }
    
    T& operator*() const { return *ptr; }
    T* operator->() const { return ptr; }
};
```

---

> [!NOTE]
> The complete 100-question bank expands heavily on advanced metaprogramming, C++20 coroutines, allocator design, low-latency finance C++, lock-free programming, and standard library internal architectures.

### Quick Reference & Cheat Sheet
- **Contiguous arrays** > node-based graphs.
- **Pass by value** for cheap-to-copy types or when you must make a copy anyway.
- **Pass by const ref** for large objects.
- **Pass by value + std::move** for sink arguments (constructors taking strings).
- **Rule of 5**: If you define a destructor, copy constructor, or copy assignment, you must define all 5 (including move semantics).
- **Rule of 0**: Best practice. Design classes so they don't need custom destructors by using smart pointers and standard containers.

### References
- *Effective Modern C++* by Scott Meyers
- *C++ Concurrency in Action* by Anthony Williams
- *Optimizing C++* by Agner Fog
- CppReference.com
- Compiler Explorer (godbolt.org)

# Module 02 — Pointers, References & Memory Management

Welcome to Module 02. In this module, we will explore one of C++'s most powerful (and dangerous) aspects: direct memory management. Understanding pointers, references, and how memory works under the hood is what separates an average C++ programmer from an expert.

---

## 1. Pointer Fundamentals

A pointer is a variable that stores a memory address.

### 1.1 Declaration, Initialization, Dereferencing

```cpp
#include <iostream>

int main() {
    int value = 42;
    // Declaration and Initialization
    int* ptr = &value; // & is the address-of operator
    
    // Dereferencing
    std::cout << "Address: " << ptr << "\n";
    std::cout << "Value: " << *ptr << "\n"; // * is the dereference operator
    
    *ptr = 100; // Modifies 'value'
    std::cout << "New Value: " << value << "\n";
    return 0;
}
```

### 1.2 Pointer Arithmetic

Pointers can be incremented or decremented. The change in memory address depends on the size of the data type it points to.

```cpp
#include <iostream>

int main() {
    int arr[] = {10, 20, 30, 40, 50};
    int* ptr = arr; // Points to arr[0]
    
    std::cout << *ptr << "\n"; // 10
    ptr++; // Moves forward by sizeof(int) bytes
    std::cout << *ptr << "\n"; // 20
    
    // Pointer difference
    int* ptr_end = &arr[4];
    std::ptrdiff_t diff = ptr_end - ptr; 
    std::cout << "Difference: " << diff << "\n"; // 3 (elements, not bytes)
    return 0;
}
```

### 1.3 Pointers to Pointers (Double/Triple Indirection)

You can have pointers that store the address of other pointers.

```cpp
int main() {
    int value = 5;
    int* p = &value;
    int** pp = &p; // Pointer to pointer
    int*** ppp = &pp; // Pointer to pointer to pointer
    
    std::cout << "Value via ppp: " << ***ppp << "\n"; // 5
    return 0;
}
```

### 1.4 Void Pointers and Type Erasure

A `void*` can hold the address of any data type but cannot be dereferenced directly without casting.

```cpp
void printValue(void* ptr, char type) {
    if (type == 'i') {
        std::cout << *static_cast<int*>(ptr) << "\n";
    } else if (type == 'd') {
        std::cout << *static_cast<double*>(ptr) << "\n";
    }
}
```

### 1.5 nullptr vs NULL vs 0

Always use `nullptr` in modern C++ (C++11 onwards). `NULL` is often just a macro for `0`, which can lead to overloaded function resolution ambiguity.

```cpp
void foo(int x) { std::cout << "int\n"; }
void foo(int* p) { std::cout << "pointer\n"; }

int main() {
    foo(0);       // Calls foo(int)
    // foo(NULL); // Might be ambiguous or call foo(int)
    foo(nullptr); // Calls foo(int*)
    return 0;
}
```

### 1.6 Function Pointers

Function pointers allow you to pass functions as arguments (callbacks) and store them in arrays.

```cpp
#include <iostream>

// Declaration syntax
void sayHello() { std::cout << "Hello\n"; }
int add(int a, int b) { return a + b; }

// Using function pointer as callback
void executeOperation(int (*op)(int, int), int x, int y) {
    std::cout << "Result: " << op(x, y) << "\n";
}

int main() {
    void (*funcPtr)() = sayHello;
    funcPtr();
    
    executeOperation(add, 5, 3);
    
    // Array of function pointers
    int (*ops[1])(int, int) = {add};
    std::cout << ops[0](10, 10) << "\n";
    
    return 0;
}
```

### 1.7 Pointers to Member Functions and Member Data

Pointers to members require an instance to be dereferenced.

```cpp
struct MyStruct {
    int data;
    void show() { std::cout << data << "\n"; }
};

int main() {
    MyStruct obj{42};
    
    // Pointer to member data
    int MyStruct::* pData = &MyStruct::data;
    std::cout << obj.*pData << "\n"; // 42
    
    // Pointer to member function
    void (MyStruct::* pFunc)() = &MyStruct::show;
    (obj.*pFunc)(); // 42
    return 0;
}
```

### 1.8 Const with Pointers

Read it right to left:
- `const int* p` (or `int const* p`): Pointer to a constant integer. The value cannot change, but the pointer can point elsewhere.
- `int* const p`: Constant pointer to an integer. The pointer cannot point elsewhere, but the value can change.
- `const int* const p`: Constant pointer to a constant integer. Nothing can change.

```cpp
int v1 = 1, v2 = 2;
const int* p1 = &v1;
// *p1 = 3; // Error
p1 = &v2;   // OK

int* const p2 = &v1;
*p2 = 3;    // OK
// p2 = &v2;   // Error
```

---

## 2. References

A reference is an alias for an existing variable.

### 2.1 Lvalue vs Rvalue References

- **Lvalue references (`T&`)**: Bind to named objects that persist beyond a single expression.
- **Rvalue references (`T&&`)**: Bind to temporary objects (rvalues). Introduced in C++11 for move semantics.

```cpp
void foo(int& x) { std::cout << "Lvalue ref\n"; }
void foo(int&& x) { std::cout << "Rvalue ref\n"; }

int main() {
    int a = 10;
    foo(a);    // Calls lvalue overload
    foo(20);   // Calls rvalue overload
    return 0;
}
```

### 2.2 Reference Collapsing Rules

Used heavily in templates (perfect forwarding).
- `T& &` $\rightarrow$ `T&`
- `T& &&` $\rightarrow$ `T&`
- `T&& &` $\rightarrow$ `T&`
- `T&& &&` $\rightarrow$ `T&&`

### 2.3 Dangling References

Occurs when a reference outlives the object it refers to.

```cpp
int& getDangling() {
    int local = 5;
    return local; // ERROR: returning reference to local variable
}
```

### 2.4 Reference to Pointer vs Pointer to Reference

- **Reference to Pointer**: Valid. You can pass a pointer by reference to modify the pointer itself. `int*& ptrRef`
- **Pointer to Reference**: Invalid in C++. `int&* ptr` is illegal because references don't have their own memory addresses.

### 2.5 When to Use References vs Pointers

- Use **references** when you assume the object will always be present (cannot be null) and you don't need to reassign the alias.
- Use **pointers** when the object might be missing (`nullptr`), when you need to reassign it, or for arrays/pointer arithmetic.

---

## 3. Arrays & Pointers

### 3.1 Array-to-Pointer Decay

An array's name decays into a pointer to its first element in most contexts (except `sizeof` and `&`).

```cpp
int arr[5];
int* p = arr; // Decays to &arr[0]
```

### 3.2 Multidimensional Arrays and Pointer Notation

```cpp
int mat[2][3] = {{1, 2, 3}, {4, 5, 6}};
int (*pRow)[3] = mat; // Pointer to an array of 3 ints
std::cout << pRow[1][2]; // 6
std::cout << *(*(mat + 1) + 2); // 6 (pointer arithmetic)
```

### 3.3 Array of Pointers vs Pointer to Array

```cpp
int* arrOfPtrs[5]; // Array containing 5 pointers to int
int (*ptrToArr)[5]; // Pointer to an array of 5 ints
```

### 3.4 Variable Length Arrays (VLAs)

Standard C++ does **not** support VLAs (like `int arr[n];` where `n` is a runtime variable). Always use `std::vector` instead. Some compilers (like GCC) allow VLAs as a non-standard extension, but rely on them at your own peril.

---

## 4. Dynamic Memory Management

### 4.1 new/delete vs new[]/delete[]

- `new` allocates memory and calls the constructor. `delete` calls the destructor and frees memory.
- `new[]` allocates an array and calls default constructors. `delete[]` calls destructors for all elements and frees the block.
  
⚠️ **Never mix them**: using `delete` on `new[]` results in undefined behavior!

### 4.2 Placement new

Constructs an object at a pre-allocated memory address. Useful in custom allocators and high-performance embedded systems.

```cpp
#include <new>

int main() {
    char buffer[sizeof(int)];
    int* p = new (buffer) int(42); // Placement new
    std::cout << *p << "\n";
    // Do not call delete p! Call destructor explicitly if it's a class: p->~MyClass();
    return 0;
}
```

### 4.3 Custom Allocators

Used with STL containers (`std::vector<int, MyAllocator>`) to control how memory is allocated (e.g., using a memory pool).

### 4.4 Memory Alignment

Proper alignment ensures efficient CPU access to memory.

```cpp
struct alignas(32) AlignedStruct {
    int data;
};
std::cout << alignof(AlignedStruct) << "\n"; // 32
```

### 4.5 malloc/free vs new/delete

- `malloc` allocates raw bytes. `new` allocates memory **and** initializes the object (calls constructors).
- `free` deallocates raw bytes. `delete` calls destructors **and** deallocates.
- Never mix `malloc` with `delete` or `new` with `free`.

### 4.6 Memory Layout

1. **Text (Code) Segment**: Executable instructions. Read-only.
2. **Data Segment**: Initialized global and static variables.
3. **BSS Segment**: Uninitialized global and static variables.
4. **Heap**: Dynamic memory (`new`, `malloc`). Grows upwards.
5. **Stack**: Local variables, function call info. Grows downwards.

---

## 5. Smart Pointers (Complete Guide)

Always prefer smart pointers over raw pointers for ownership in modern C++ (C++11+). Found in `<memory>`.

### 5.1 std::unique_ptr

Exclusive ownership. Cannot be copied, only moved.

```cpp
#include <memory>
#include <iostream>

struct Resource {
    Resource() { std::cout << "Acquired\n"; }
    ~Resource() { std::cout << "Released\n"; }
};

int main() {
    {
        std::unique_ptr<Resource> ptr = std::make_unique<Resource>();
        // std::unique_ptr<Resource> ptr2 = ptr; // ERROR: no copy
        std::unique_ptr<Resource> ptr3 = std::move(ptr); // OK
    } // Released automatically
    return 0;
}
```
**Custom Deleters**:
```cpp
auto fileDeleter = [](FILE* f) { fclose(f); };
std::unique_ptr<FILE, decltype(fileDeleter)> filePtr(fopen("test.txt", "w"), fileDeleter);
```
**Array Specialization**: `std::unique_ptr<int[]>` properly calls `delete[]`.

### 5.2 std::shared_ptr

Shared ownership using reference counting. Uses a **control block** to store the reference count and weak count.

```cpp
std::shared_ptr<int> sp1 = std::make_shared<int>(100);
std::shared_ptr<int> sp2 = sp1;
std::cout << sp1.use_count() << "\n"; // 2
```
**Aliasing constructor**: Allows a shared_ptr to own an object but point to its member.

### 5.3 std::weak_ptr

Non-owning observer. Used to break cyclic dependencies of `shared_ptr`s.

```cpp
std::shared_ptr<int> sp = std::make_shared<int>(42);
std::weak_ptr<int> wp = sp;

if (std::shared_ptr<int> locked = wp.lock()) { // Upgrade to shared_ptr
    std::cout << *locked << "\n";
}
```

### 5.4 Why use std::make_unique / std::make_shared?

- Prevents memory leaks if an exception is thrown in another argument of a function call.
- `std::make_shared` allocates the object and the control block in a **single allocation**, improving cache locality and performance.

### 5.5 std::enable_shared_from_this

Allows an object managed by a `shared_ptr` to safely generate a `shared_ptr` to itself using `shared_from_this()`.

---

## 6. RAII Pattern

**Resource Acquisition Is Initialization**.
Resources (memory, file handles, sockets) are tied to object lifetime. Acquired in the constructor, released in the destructor.

```cpp
class FileWrapper {
    FILE* file;
public:
    FileWrapper(const char* filename) {
        file = fopen(filename, "r");
        if (!file) throw std::runtime_error("File not found");
    }
    ~FileWrapper() {
        if (file) fclose(file);
    }
    // Delete copy constructor and assignment!
    FileWrapper(const FileWrapper&) = delete;
    FileWrapper& operator=(const FileWrapper&) = delete;
};
```

---

## 7. Common Memory Issues

1. **Memory Leaks**: Forgetting to call `delete`. Objects accumulate in the heap.
2. **Dangling Pointers / Use-After-Free**: Accessing memory after it's been deleted.
3. **Double Delete**: Calling `delete` twice on the same pointer. Corrupts the heap.
4. **Buffer Overflows**: Writing past the end of an array.
5. **Wild Pointers**: Uninitialized pointers pointing to random memory.
6. **Stack Overflow**: Infinite recursion or huge local variables exceeding stack limits.

*Tools to find these*: Valgrind, AddressSanitizer (`-fsanitize=address`), cppcheck.

---

## 8. Interview Questions

1. **What is the difference between `delete` and `delete[]`?**
   *Answer*: `delete` destroys a single object. `delete[]` destroys an array of objects. Mixing them causes UB because `delete[]` reads a hidden metadata counter (usually stored right before the array) to know how many destructors to call.

2. **Explain `const int* p` vs `int* const p`.**
   *Answer*: The first is a pointer to a constant integer (value is immutable through the pointer). The second is a constant pointer to an integer (pointer address is immutable).

3. **Predict the output of the following pointer arithmetic:**
   ```cpp
   int arr[] = {10, 20, 30, 40, 50};
   int* p = arr;
   std::cout << *p++ << " ";
   std::cout << *++p << " ";
   ```
   *Answer*: `10 30`. `*p++` dereferences first, then increments the pointer. The next line increments the pointer (now pointing to 30), then dereferences it.

4. **Why doesn't C++ support variable-length arrays (VLAs) like C99?**
   *Answer*: They compromise stack safety and make compiler implementations complex (especially dealing with C++ exceptions). `std::vector` dynamically allocates on the heap and handles resizing safely.

5. **How does `std::shared_ptr` handle multi-threading?**
   *Answer*: The reference count control block operations (increment/decrement) are atomic and thread-safe. However, the `shared_ptr` instance itself (reading/writing the pointer) is **not** thread-safe.

6. **What is a "cyclic dependency" in smart pointers and how do you solve it?**
   *Answer*: When `shared_ptr` A points to B, and `shared_ptr` B points to A, the reference count will never reach zero. Solved by making one of the pointers a `std::weak_ptr`.

7. **Is it safe to `delete this;` inside a member function?**
   *Answer*: It is legal, but extremely dangerous. You must guarantee the object was allocated via `new`, and you cannot access any member variables or call virtual functions after doing it. 

8. **What happens if you throw an exception from a constructor during `new`?**
   *Answer*: The allocated memory for the object itself is automatically deallocated. However, if the constructor allocated *other* resources raw pointers before throwing, those leak (which is why you should use RAII/smart pointers for class members).

9. **Explain "Reference Collapsing Rules".**
   *Answer*: Occurs during template instantiation or `typedef`/`using`. If a type resolves to multiple references, lvalue references win. The only way to get an rvalue reference is `&& &&` -> `&&`. Otherwise, it collapses to an lvalue reference `&`.

10. **What is placement `new` and when would you use it?**
   *Answer*: Placement new constructs an object in pre-allocated memory `new (ptr) Object()`. Used in memory pools, standard library containers, or embedded systems where memory is mapped to specific hardware addresses.

11. **Predict the Output:**
    ```cpp
    int a = 1;
    int& ref = a;
    int b = 2;
    ref = b;
    std::cout << a << "\n";
    ```
    *Answer*: `2`. `ref = b;` assigns the value of `b` to `a`, it doesn't re-bind the reference. References cannot be re-bound.

12. **Why should you prefer `std::make_unique` over `new`?**
    *Answer*: 1) Exception safety: prevents memory leaks if another argument throws. 2) DRY principle: you don't repeat the type name.

13. **Why should you prefer `std::make_shared` over `new`?**
    *Answer*: Same as `make_unique`, plus performance. `make_shared` performs a single heap allocation for both the object and the control block, reducing overhead and improving cache locality.

14. **What is the difference between a pointer and a reference?**
    *Answer*: References must be initialized, cannot be null, and cannot be re-bound. Pointers can be uninitialized, can be null, can be reassigned, and support arithmetic.

15. **What is the memory layout of a C++ program?**
    *Answer*: Code/Text segment (executable instructions), Data segment (initialized globals/statics), BSS (uninitialized globals/statics), Heap (dynamic memory, grows up), Stack (local variables/function frames, grows down).

16. **How does an array decay into a pointer?**
    *Answer*: When an array name is used in most expressions, it converts implicitly to a pointer to its first element. Exceptions: `sizeof(arr)` (gives total bytes), `&arr` (gives pointer to the whole array).

17. **What is an Rvalue reference?**
    *Answer*: A reference that binds to temporary objects (rvalues). Indicated by `&&`. It enables move semantics and perfect forwarding by allowing us to steal resources from temporary objects that are about to be destroyed.

18. **Can you have an array of references?**
    *Answer*: No. References don't exist as independent objects in memory, so you cannot have arrays of them. You can use `std::reference_wrapper` to store references in containers.

19. **What does `alignas` do?**
    *Answer*: Specifies custom memory alignment requirements for a type or object, ensuring the compiler places it at a memory address that is a multiple of the given alignment value.

20. **Can you return a reference to a local variable?**
    *Answer*: No. This results in a dangling reference. The local variable is destroyed when the function exits, and the reference will point to garbage memory.

---

## 9. Quick Reference / Cheat Sheet

- **Pointer**: `T* ptr = &var;`
- **Reference**: `T& ref = var;`
- **Dynamic Array**: `T* arr = new T[size]; delete[] arr;`
- **Unique Ptr**: `std::unique_ptr<T> p = std::make_unique<T>();`
- **Shared Ptr**: `std::shared_ptr<T> p = std::make_shared<T>();`
- **Weak Ptr**: `std::weak_ptr<T> wp = shared_p;`
- **Const rules**: Read right to left.
- **RAII**: Bind resource lifecycle to object lifecycle.

---
*End of Module 02*

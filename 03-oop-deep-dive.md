# Module 03 — Object-Oriented Programming Deep Dive

Welcome to the **C++ Object-Oriented Programming (OOP) Deep Dive**. C++ is a multi-paradigm language, but its OOP features are incredibly powerful and form the core of many large-scale applications. In this module, we will explore C++ OOP concepts from the ground up to expert-level intricacies.

---

## 1. Classes & Objects

### 1.1 Class vs Struct
In C++, the `class` and `struct` keywords are nearly identical. The only difference is the default access level and default inheritance mode.
- **`class`**: Defaults to `private` access and `private` inheritance.
- **`struct`**: Defaults to `public` access and `public` inheritance.

**Convention:** Use `struct` for plain old data (POD) structures or aggregates where all members are public. Use `class` when you want to enforce encapsulation, maintain invariants, and provide an interface.

```cpp
// Conventionally used for plain data
struct Point {
    int x;
    int y;
}; // Members are public by default

class BankAccount {
    double balance; // Private by default
public:
    void deposit(double amount) { balance += amount; }
};
```

### 1.2 Access Specifiers
C++ provides three access specifiers to control encapsulation:
- **`public`**: Accessible from anywhere the object is visible.
- **`protected`**: Accessible from within the class and its derived classes.
- **`private`**: Accessible only from within the class (and friends).

### 1.3 The `this` Pointer
Every non-static member function has an implicit parameter called `this`, which is a pointer to the object on which the function is called.
- In a non-const member function of `class T`, `this` is of type `T* const`.
- In a const member function, `this` is of type `const T* const`.

```cpp
class Box {
    int length;
public:
    Box(int length) {
        this->length = length; // Disambiguate member from parameter
    }
    Box* getPointer() {
        return this; // Return pointer to current object
    }
    // Const member function: 'this' is const Box* const
    int getLength() const { return length; } 
};
```

### 1.4 `sizeof` for Classes
The size of a class is influenced by its members, padding (alignment), and the presence of virtual functions.

- **Empty Class**: The size of an empty class is at least **1 byte**. This ensures that distinct objects of the same class have distinct memory addresses.
- **Padding and Alignment**: Compilers insert padding to align data members according to architecture requirements (e.g., 4-byte boundaries).
- **vptr**: If a class has virtual functions, the compiler adds a pointer to the virtual table (`vptr`), increasing the size by 4 or 8 bytes depending on the architecture.

```cpp
class Empty {};
class Data { char c; int i; }; // Likely 8 bytes on 64-bit due to padding after 'char'
class VirtualData { char c; int i; virtual ~VirtualData() {} }; // Likely 16 bytes: 8 for vptr + 8 for data/padding

#include <iostream>
int main() {
    std::cout << sizeof(Empty) << "\n"; // 1
    std::cout << sizeof(Data) << "\n";  // 8
    std::cout << sizeof(VirtualData) << "\n"; // 16
}
```

### 1.5 Aggregate Types
An aggregate is an array or a class with:
- No user-declared or inherited constructors.
- No private or protected non-static data members.
- No virtual functions.
- No virtual, private, or protected base classes.

Aggregate initialization allows brace-initialization (`{}`).

```cpp
struct Employee {
    int id;
    std::string name;
};
Employee emp1 = {1, "Alice"}; // Aggregate initialization
```

---

## 2. Constructors & Destructors

### 2.1 Types of Constructors and Destructors
- **Default Constructor**: Can be called with no arguments.
- **Parameterized Constructor**: Takes arguments to initialize members.
- **Copy Constructor**: Initializes an object from another object of the same type.
- **Move Constructor**: Initializes an object by transferring ownership of resources from a temporary (rvalue).
- **Destructor**: Called when the object's lifetime ends to clean up resources.

```cpp
class String {
    char* data;
public:
    // Default
    String() : data(nullptr) {}
    
    // Parameterized
    String(const char* str) { /* allocate and copy */ }
    
    // Copy
    String(const String& other) { /* deep copy */ }
    
    // Move
    String(String&& other) noexcept : data(other.data) {
        other.data = nullptr; // Steal resource, leave other in valid state
    }
    
    // Destructor
    ~String() { delete[] data; }
};
```

### 2.2 Delegating Constructors
Introduced in C++11, a constructor can call another constructor of the same class to avoid code duplication.

```cpp
class Widget {
    int x, y;
public:
    Widget(int x, int y) : x(x), y(y) {}
    Widget() : Widget(0, 0) {} // Delegates to parameterized constructor
};
```

### 2.3 `explicit` Constructors and Conversion Operators
By default, a constructor callable with a single argument acts as an implicit conversion operator. The `explicit` keyword prevents this.

```cpp
class Vector {
public:
    explicit Vector(int size) {}
};

Vector v1 = 10; // Error! Implicit conversion disabled.
Vector v2(10);  // OK
```
Similarly, conversion operators can be marked `explicit`.

### 2.4 Member Initializer Lists
Data members must be initialized in the **Member Initializer List**.
> **IMPORTANT:** Members are initialized in the order they are **declared in the class**, NOT in the order they appear in the initializer list.

```cpp
class Foo {
    int y;
    int x; // Initialized first if declared first? No, y is declared first!
public:
    // WARNING: 'y' is initialized before 'x' because it's declared first.
    // If you wrote: Foo(int val) : x(val), y(x) {}, y would get garbage!
    Foo(int val) : y(val), x(y) {} 
};
```

### 2.5 In-class Member Initializers (NSDMI)
C++11 allows initializing members directly where they are declared.

```cpp
class Server {
    int port = 8080;
    std::string ip{"127.0.0.1"};
public:
    Server() = default; // Uses in-class initializers
    Server(int p) : port(p) {} // ip uses in-class, port uses init-list
};
```

### 2.6 The Rule of Zero, Three, Five
- **Rule of Three (C++98)**: If a class requires a user-defined destructor, copy constructor, or copy assignment operator, it almost certainly requires all three.
- **Rule of Five (C++11)**: Extend Rule of Three to include move constructor and move assignment operator.
- **Rule of Zero**: Classes should be designed so they don't need custom destructors, copy/move constructors, or assignment operators. Rely on standard library types like `std::string` and `std::unique_ptr` that handle their own resources.

### 2.7 Virtual Destructors
> **CAUTION:** If a class is meant to be derived from and manipulated polymorphically (via base class pointers), its destructor **MUST** be `virtual`.

```cpp
class Base {
public:
    virtual ~Base() { std::cout << "Base dest\n"; }
};
class Derived : public Base {
public:
    ~Derived() { std::cout << "Derived dest\n"; }
};

Base* b = new Derived();
delete b; // If ~Base() wasn't virtual, ~Derived() wouldn't be called -> Memory Leak!
```

### 2.8 Destructor Order in Inheritance
Destruction happens in the exact reverse order of construction.
Construction: Base -> Member Objects -> Derived
Destruction: Derived -> Member Objects -> Base

---

## 3. Operator Overloading

### 3.1 Overloadable vs Non-overloadable
- **Overloadable**: `+`, `-`, `*`, `/`, `=`, `<>`, `()`, `[]`, `->`, `new`, `delete`, etc.
- **Non-overloadable**: `.`, `.*`, `::`, `?:` (ternary), `sizeof`, `typeid`.

### 3.2 Member vs Non-member (Friend)
- Use member functions if the left-hand operand is an object of the class (e.g., `=`, `[]`, `()`, `->`).
- Use non-member (often `friend`) functions if the left operand is a different type, or for symmetric operators like `+`.

### 3.3 `operator<<` and `operator>>`
Must be implemented as non-member functions because the left operand is `std::ostream` or `std::istream`.

```cpp
class Point {
    int x, y;
public:
    Point(int x, int y) : x(x), y(y) {}
    friend std::ostream& operator<<(std::ostream& os, const Point& p) {
        return os << "(" << p.x << ", " << p.y << ")";
    }
};
```

### 3.4 Spaceship Operator `<=>` (C++20)
The three-way comparison operator allows the compiler to generate all six relational operators automatically.

```cpp
#include <compare>
struct Value {
    int data;
    auto operator<=>(const Value&) const = default;
};
```

### 3.5 Functors, `[]`, and `->`
- `operator()` creates a functor, treating an object like a function.
- `operator[]` is used for array-like access.
- `operator->` is essential for implementing smart pointers.

```cpp
class SmartPtr {
    int* ptr;
public:
    SmartPtr(int* p = nullptr) : ptr(p) {}
    ~SmartPtr() { delete ptr; }
    int* operator->() { return ptr; }
    int& operator*() { return *ptr; }
};
```

### 3.6 Copy-and-Swap Idiom
A robust way to implement the assignment operator that provides strong exception guarantee.

```cpp
class Array {
    int* data;
    size_t size;
public:
    // Copy constructor
    Array(const Array& other) { /* ... */ }
    
    // Friend swap function
    friend void swap(Array& first, Array& second) noexcept {
        std::swap(first.data, second.data);
        std::swap(first.size, second.size);
    }
    
    // Assignment operator (takes argument by value!)
    Array& operator=(Array other) {
        swap(*this, other);
        return *this;
    } // other's destructor cleans up old data
};
```

---

## 4. Inheritance

### 4.1 Types of Inheritance
- **Single**: One base class.
- **Multiple**: Multiple base classes.
- **Multilevel**: `C` derives from `B`, which derives from `A`.
- **Hierarchical**: Multiple classes derive from one base class.
- **Hybrid**: A mix of the above.

### 4.2 Virtual Inheritance and the Diamond Problem
In multiple inheritance, if two classes `B` and `C` inherit from `A`, and `D` inherits from `B` and `C`, `D` will contain two copies of `A`. Virtual inheritance solves this.

```cpp
class Animal { public: int age; };
class Mammal : virtual public Animal {};
class Winged : virtual public Animal {};
class Bat : public Mammal, public Winged {}; 
// Bat has only one instance of Animal::age
```

### 4.3 Access Control in Inheritance
- `public` inheritance: Models "is-a". Public members stay public, protected stay protected.
- `protected` inheritance: Models "implemented-in-terms-of". Public and protected members become protected.
- `private` inheritance: Models "implemented-in-terms-of". Public and protected members become private.

### 4.4 Object Slicing
If a derived object is passed or assigned to a base object by **value**, the derived-specific parts are "sliced" off.
> **TIP:** Always pass polymorphic objects by pointer or reference to prevent slicing.

```cpp
class Base { public: int b; virtual void f() {} };
class Derived : public Base { public: int d; void f() override {} };

void func(Base obj) {} // SLICING occurs!
Derived d;
func(d); // Only the Base part is copied to obj.
```

---

## 5. Polymorphism

### 5.1 Compile-time vs Runtime
- **Compile-time (Static)**: Function overloading, Operator overloading, Templates, CRTP (Curiously Recurring Template Pattern). Resolved during compilation.
- **Runtime (Dynamic)**: Virtual functions via base class pointers/references. Resolved at runtime using the `vtable`.

### 5.2 The `vtable` Mechanism Explained
Every class that declares or inherits a virtual function has a virtual table (`vtable`) created by the compiler. The `vtable` is an array of function pointers pointing to the most-derived implementations of the virtual functions.
Every instance of such a class contains a hidden pointer, the `vptr`, which points to the class's `vtable`.
When a virtual function is called:
1. Follow the object's `vptr` to the `vtable`.
2. Look up the index of the function being called.
3. Call the function pointer at that index.

### 5.3 Pure Virtual Functions & Abstract Classes
A function ending with `= 0` is a pure virtual function. A class with at least one pure virtual function is an **Abstract Class** and cannot be instantiated.

```cpp
class Shape {
public:
    virtual double area() const = 0; // Pure virtual
    virtual ~Shape() {}
};
```
An **Interface Class** is an abstract class with *only* pure virtual functions and no data members.

### 5.4 `override` and `final` (C++11)
- `override`: Ensures the function actually overrides a base class virtual function. Catches spelling or signature mismatch errors at compile time.
- `final`: Prevents a virtual function from being overridden further, or a class from being inherited.

### 5.5 Virtual Function Calls in Constructors/Destructors
> **WARNING:** Never call virtual functions in constructors or destructors. During construction, the `vptr` is set to the current class being constructed. It does not look down to derived classes because they haven't been constructed yet (or have already been destroyed).

```cpp
class Base {
public:
    Base() { foo(); } // Calls Base::foo(), NOT Derived::foo()
    virtual void foo() { std::cout << "Base\n"; }
};
```

### 5.6 RTTI: `dynamic_cast` and `typeid`
Runtime Type Information allows querying the type at runtime.
- `dynamic_cast`: Safely downcasts a polymorphic base pointer/reference to a derived pointer/reference. Returns `nullptr` (for pointers) or throws `std::bad_cast` (for references) if the cast fails.

```cpp
Base* b = new Derived();
if (Derived* d = dynamic_cast<Derived*>(b)) {
    // Cast succeeded
}
```

---

## 6. Special Topics

### 6.1 Friend Functions and Classes
A `friend` has access to private and protected members of the class. Friendship is not mutual, not inherited, and not transitive.

### 6.2 Static Members
Static members belong to the class rather than an object.
- Static data members must be defined outside the class (unless `inline` in C++17 or `const` integral types).
- Static member functions do not have a `this` pointer and can only access other static members.

### 6.3 Mutable Keyword
Allows a data member to be modified even inside a `const` member function. Useful for caching, lazy evaluation, or mutexes.

```cpp
class Cache {
    mutable int computedValue = 0;
    mutable bool isValid = false;
public:
    int getValue() const {
        if (!isValid) {
            computedValue = 42; // Allowed because of mutable
            isValid = true;
        }
        return computedValue;
    }
};
```

### 6.4 Copy Elision and RVO/NRVO
The compiler is allowed to omit copy/move constructors, constructing the object directly in the target memory.
- **RVO (Return Value Optimization)**: Returning a temporary object. Mandatory since C++17.
- **NRVO (Named Return Value Optimization)**: Returning a local variable.

---

## 7. SOLID Principles in C++

### 7.1 Single Responsibility Principle (SRP)
A class should have only one reason to change.
```cpp
// Bad: Class handles logic AND logging
class User {
    void saveToDatabase() { /* DB logic */ }
};

// Good: Separate responsibilities
class User {};
class UserRepository { void save(User u) {} };
```

### 7.2 Open/Closed Principle (OCP)
Classes should be open for extension but closed for modification.
```cpp
class Shape { public: virtual double area() const = 0; };
class Rectangle : public Shape { /* ... */ };
class Circle : public Shape { /* ... */ };
// We can add Triangle without modifying Shape or existing classes
```

### 7.3 Liskov Substitution Principle (LSP)
Subtypes must be substitutable for their base types without altering correctness.
```cpp
// Bad: A Square modifying width also modifies height, breaking Rectangle invariants.
class Rectangle { virtual void setWidth(int w); };
class Square : public Rectangle { void setWidth(int w) override { width = height = w; } };
```

### 7.4 Interface Segregation Principle (ISP)
Clients should not be forced to depend on interfaces they do not use.
```cpp
// Bad: Fat interface
class IWorker { virtual void work()=0; virtual void eat()=0; };

// Good: Segregated
class IWorkable { virtual void work()=0; };
class IFeedable { virtual void eat()=0; };
```

### 7.5 Dependency Inversion Principle (DIP)
High-level modules should not depend on low-level modules; both should depend on abstractions.
```cpp
class ILogger { public: virtual void log(const std::string&) = 0; };
class FileLogger : public ILogger { /*...*/ };

class App {
    ILogger& logger; // Depends on abstraction, not concrete FileLogger
public:
    App(ILogger& l) : logger(l) {}
};
```

---

## 8. Interview Questions & Detailed Answers

**Q1: What is the exact difference between `class` and `struct` in C++?**
**A:** In C++, the only difference is the default visibility. Members and base classes of a `struct` are `public` by default, whereas for a `class`, they are `private` by default. Everything else (templates, inheritance, methods, etc.) is identical.

**Q2: How does the compiler resolve virtual function calls internally? (Explain the vtable)**
**A:** The compiler creates a static array of function pointers called the `vtable` for any class containing virtual functions. Every object of this class contains a hidden pointer called the `vptr` pointing to this `vtable`. When a virtual function is called via a pointer or reference, the compiler generates code to dereference the object's `vptr` to find the `vtable`, looks up the correct index for the function, and calls the function pointer found there.

**Q3: What is the "Diamond Problem" and how does C++ solve it?**
**A:** It occurs in multiple inheritance when a class `D` inherits from `B` and `C`, and both `B` and `C` inherit from `A`. `D` ends up with two copies of `A`'s subobject. This creates ambiguity. C++ solves this using `virtual inheritance` (`class B : virtual public A`). With virtual inheritance, only a single shared instance of the base class `A` is created within `D`.

**Q4: Why should destructors be virtual in polymorphic base classes?**
**A:** If you delete a derived object through a base class pointer and the base class destructor is not virtual, the compiler resolves the destructor call statically based on the pointer type. Only the base class destructor will run, skipping the derived class's destructor and leading to memory or resource leaks.

**Q5: Can a constructor be virtual? Why or why not?**
**A:** No. A virtual call relies on the `vptr` to resolve the function type at runtime. During the execution of a constructor, the object is not fully formed, and the `vptr` points to the `vtable` of the class currently being constructed, not the final derived class. Therefore, the concept of a virtual constructor doesn't logically apply. (To achieve polymorphic creation, we use the "Virtual Constructor Idiom", like a virtual `clone()` method).

**Q6: What happens if you call a virtual function inside a constructor?**
**A:** The virtual mechanism is effectively disabled. The function called will be the one defined in the class currently being constructed (or its base classes). It will not call overrides in derived classes because the derived class hasn't been constructed yet, and the `vptr` is currently set to the `vtable` of the class whose constructor is running.

**Q7: Explain the Rule of Three, Rule of Five, and Rule of Zero.**
**A:** 
- **Rule of Three**: If a class manages resources manually and needs a custom destructor, copy constructor, or copy assignment operator, it likely needs all three.
- **Rule of Five**: With C++11, if you define any of the three, you should also define the move constructor and move assignment operator for optimization.
- **Rule of Zero**: The best practice is to design classes that rely entirely on RAII members (like `std::string`, `std::unique_ptr`) so that no custom destructor, copy, or move operations are needed at all; the compiler-generated defaults will do the right thing.

**Q8: What is "Object Slicing"?**
**A:** Object slicing occurs when a derived class object is assigned to or passed by value to a base class object. Since the target object only has memory allocated for the base class members, any derived-specific members are "sliced off" and lost. This is prevented by passing objects by reference or pointer.

**Q9: Explain the `explicit` keyword.**
**A:** It is used on single-argument constructors (or constructors where all but one argument have defaults) and conversion operators to prevent the compiler from using them for implicit type conversions. It forces the programmer to explicitly cast or construct the object.

**Q10: What does the `mutable` keyword do?**
**A:** It allows a specific class data member to be modified even if the object itself is declared `const`, or if it is being accessed from inside a `const` member function. It is typically used for internal implementation details like caching, memoization, or synchronization primitives (mutexes) that don't change the logical state of the object.

**Q11: What is the size of an empty class and why?**
**A:** It is usually 1 byte (though compiler-dependent, it must be > 0). The C++ standard requires that two distinct objects of the same type must have different memory addresses. If empty classes had size 0, an array of them would all have the same address.

**Q12: In what order are class members initialized?**
**A:** They are initialized in the exact order they are *declared in the class definition*, NOT in the order they appear in the constructor's initialization list.

**Q13: How does the Copy-and-Swap idiom work?**
**A:** It's an elegant way to write an exception-safe assignment operator. The operator takes its argument *by value* (which uses the copy constructor). Then, it swaps the internals of `*this` with the temporary copy. When the function exits, the temporary goes out of scope and its destructor cleans up the old resources previously held by `*this`.

**Q14: What is the difference between early binding and late binding?**
**A:** Early (static) binding happens at compile time, resolving function calls based on the static type of the pointer/object (e.g., normal functions, overloaded functions). Late (dynamic) binding happens at runtime using the `vtable`, resolving function calls based on the actual dynamic type of the object pointed to (e.g., virtual functions).

**Q15: What is a Pure Virtual Function?**
**A:** A virtual function declared with `= 0` at the end. It provides no implementation in the base class (though it technically can have one out-of-line) and forces any concrete derived class to override it. A class with at least one pure virtual function becomes abstract.

**Q16: How do you prevent a class from being inherited in C++11?**
**A:** Append the `final` keyword after the class name in its declaration: `class MyClass final { ... };`

**Q17: What does the `override` keyword do and why use it?**
**A:** It explicitly states that a function is meant to override a virtual function in a base class. If there is a signature mismatch, spelling error, or if the base function isn't virtual, the compiler will generate an error. It prevents subtle bugs.

**Q18: What is CRTP (Curiously Recurring Template Pattern)?**
**A:** It is an idiom where a class `Derived` derives from a class template instantiated with `Derived` itself: `class Derived : public Base<Derived>`. It is used to achieve static polymorphism (resolving polymorphic behavior at compile time rather than using virtual functions and vtables), avoiding runtime overhead.

**Q19: Can a `friend` function be `virtual`?**
**A:** No. A friend function is not a member function of the class; it is an external function that is granted access. Since it's not a member, it cannot be virtual.

**Q20: What is the difference between `dynamic_cast` and `static_cast`?**
**A:** `static_cast` performs conversions at compile-time and performs no runtime checks, making it fast but potentially unsafe if you downcast incorrectly. `dynamic_cast` is used for safe downcasting of polymorphic types at runtime. It checks RTTI to verify the cast is valid; if it fails, it returns `nullptr` (for pointers) or throws `std::bad_cast` (for references).

**Q21: Does the size of an object increase if it has a static data member?**
**A:** No. Static data members exist independently of any object instance. They are stored in the data segment (BSS or initialized data segment), so they do not contribute to `sizeof(Class)`.

---
*End of Module 03*

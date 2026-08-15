# Module 11 — Design Patterns & Best Practices in Modern C++

Welcome to Module 11. For experienced C++ developers, knowing syntax is only the beginning. True mastery comes from knowing *how to organize code* to make it robust, maintainable, and performant. In this module, we will deep-dive into Design Patterns, Modern C++ Idioms, and Best Practices using Modern C++ (C++11 through C++23) semantics. We focus on zero-overhead abstractions, move semantics, and type-safe idioms.

---

## 1. Creational Patterns

Creational patterns abstract the instantiation process. They help make a system independent of how its objects are created, composed, and represented.

### 1.1. Singleton

The Singleton pattern ensures a class has only one instance and provides a global point of access to it.
In Modern C++, the **Meyer's Singleton** is the gold standard because it is thread-safe (since C++11) and guarantees lazy initialization without locks.

```cpp
class DatabaseConnection {
public:
    // Meyer's Singleton: thread-safe in C++11 and later
    static DatabaseConnection& getInstance() {
        static DatabaseConnection instance;
        return instance;
    }

    // Delete copy/move constructors and assignment operators
    DatabaseConnection(const DatabaseConnection&) = delete;
    DatabaseConnection& operator=(const DatabaseConnection&) = delete;
    DatabaseConnection(DatabaseConnection&&) = delete;
    DatabaseConnection& operator=(DatabaseConnection&&) = delete;

    void query(const std::string& q) {
        // execute query
    }

private:
    DatabaseConnection() = default; // Private constructor
    ~DatabaseConnection() = default; // Private destructor
};
```

> [!WARNING]
> **Why Singletons are Controversial:** Singletons introduce global state, making testing difficult (mocking a singleton is hard without dependency injection) and obscuring dependencies. Use them sparingly, typically for loggers or hardware access abstractions.

### 1.2. Factory Method (with `std::unique_ptr`)

The Factory Method defines an interface for creating an object, but lets subclasses decide which class to instantiate. In modern C++, factories should almost always return `std::unique_ptr` to express clear ownership transfer.

```cpp
#include <memory>
#include <iostream>

class Document {
public:
    virtual ~Document() = default;
    virtual void open() const = 0;
};

class PdfDocument : public Document {
public:
    void open() const override { std::cout << "Opening PDF.\n"; }
};

class WordDocument : public Document {
public:
    void open() const override { std::cout << "Opening Word.\n"; }
};

class DocumentCreator {
public:
    virtual ~DocumentCreator() = default;
    // Factory method returning unique_ptr
    virtual std::unique_ptr<Document> createDocument() const = 0;
};

class PdfCreator : public DocumentCreator {
public:
    std::unique_ptr<Document> createDocument() const override {
        return std::make_unique<PdfDocument>();
    }
};
```

### 1.3. Abstract Factory

Provides an interface for creating families of related or dependent objects without specifying their concrete classes.

```cpp
class Button { public: virtual ~Button() = default; virtual void paint() = 0; };
class WinButton : public Button { public: void paint() override { /* Win logic */ } };
class MacButton : public Button { public: void paint() override { /* Mac logic */ } };

class ScrollBar { public: virtual ~ScrollBar() = default; virtual void paint() = 0; };
class WinScrollBar : public ScrollBar { public: void paint() override { /* Win logic */ } };
class MacScrollBar : public ScrollBar { public: void paint() override { /* Mac logic */ } };

class GUIFactory {
public:
    virtual ~GUIFactory() = default;
    virtual std::unique_ptr<Button> createButton() = 0;
    virtual std::unique_ptr<ScrollBar> createScrollBar() = 0;
};

class WinFactory : public GUIFactory {
public:
    std::unique_ptr<Button> createButton() override { return std::make_unique<WinButton>(); }
    std::unique_ptr<ScrollBar> createScrollBar() override { return std::make_unique<WinScrollBar>(); }
};
```

### 1.4. Builder (Fluent API Style)

Builder separates the construction of a complex object from its representation. The fluent API style allows method chaining.

```cpp
#include <string>
#include <iostream>

class HttpRequest {
public:
    std::string method, url, body;
    int timeout_ms = 0;

    void print() const { std::cout << method << " " << url << "\n"; }
};

class RequestBuilder {
private:
    HttpRequest request;
public:
    RequestBuilder& method(const std::string& m) { request.method = m; return *this; }
    RequestBuilder& url(const std::string& u) { request.url = u; return *this; }
    RequestBuilder& body(const std::string& b) { request.body = b; return *this; }
    RequestBuilder& timeout(int t) { request.timeout_ms = t; return *this; }
    
    HttpRequest build() { return std::move(request); }
};

// Usage:
// auto req = RequestBuilder().method("GET").url("https://api.com").timeout(5000).build();
```

### 1.5. Prototype (Clone with Covariant Return Types)

Prototype allows copying existing objects without making the code dependent on their classes. Covariant return types allow an overriding virtual function to return a pointer/reference to a derived class.

```cpp
#include <memory>

class Shape {
public:
    virtual ~Shape() = default;
    // Base clone returns unique_ptr<Shape>
    virtual std::unique_ptr<Shape> clone() const = 0;
};

class Circle : public Shape {
    int radius;
public:
    Circle(int r) : radius(r) {}
    
    // Modern C++ standard doesn't support covariant return types with smart pointers directly.
    // We implement covariant raw clone, then wrap in smart pointer.
private:
    virtual Circle* do_clone() const { return new Circle(*this); }
public:
    std::unique_ptr<Shape> clone() const override {
        return std::unique_ptr<Shape>(do_clone());
    }
};
```

---

## 2. Structural Patterns

Structural patterns deal with object composition, creating relationships between objects to form larger structures.

### 2.1. Adapter

Adapter allows objects with incompatible interfaces to collaborate.
- **Class Adapter**: Uses multiple inheritance.
- **Object Adapter**: Uses composition (preferred).

```cpp
// Target interface
class MediaPlayer {
public:
    virtual void play(const std::string& audioType, const std::string& fileName) = 0;
    virtual ~MediaPlayer() = default;
};

// Adaptee
class AdvancedMediaPlayer {
public:
    void playVlc(const std::string& fileName) { /* ... */ }
    void playMp4(const std::string& fileName) { /* ... */ }
};

// Object Adapter
class MediaAdapter : public MediaPlayer {
    std::unique_ptr<AdvancedMediaPlayer> advancedMusicPlayer;
public:
    MediaAdapter() : advancedMusicPlayer(std::make_unique<AdvancedMediaPlayer>()) {}
    
    void play(const std::string& audioType, const std::string& fileName) override {
        if(audioType == "vlc") advancedMusicPlayer->playVlc(fileName);
        else if(audioType == "mp4") advancedMusicPlayer->playMp4(fileName);
    }
};
```

### 2.2. Bridge (Pimpl Idiom as Bridge)

Bridge decouples an abstraction from its implementation. The Pimpl (Pointer to Implementation) idiom in C++ is a classic example of Bridge.

```cpp
// Widget.h
#include <memory>
class Widget {
public:
    Widget();
    ~Widget(); // Must be defined in .cpp where Impl is visible
    void draw();
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// Widget.cpp
struct Widget::Impl {
    void draw() { /* implementation details hidden from header */ }
};
Widget::Widget() : pImpl(std::make_unique<Impl>()) {}
Widget::~Widget() = default; 
void Widget::draw() { pImpl->draw(); }
```

### 2.3. Composite

Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.

```cpp
#include <vector>
#include <memory>
#include <iostream>

class Graphic {
public:
    virtual void draw() const = 0;
    virtual ~Graphic() = default;
};

class Line : public Graphic {
public:
    void draw() const override { std::cout << "Line\n"; }
};

class Picture : public Graphic {
    std::vector<std::unique_ptr<Graphic>> children;
public:
    void add(std::unique_ptr<Graphic> g) { children.push_back(std::move(g)); }
    void draw() const override {
        for (const auto& child : children) {
            child->draw();
        }
    }
};
```

### 2.4. Decorator

Attach additional responsibilities to an object dynamically. Provides a flexible alternative to subclassing.

```cpp
class Coffee {
public:
    virtual double cost() const = 0;
    virtual ~Coffee() = default;
};

class SimpleCoffee : public Coffee {
public:
    double cost() const override { return 1.0; }
};

class CoffeeDecorator : public Coffee {
protected:
    std::unique_ptr<Coffee> coffee;
public:
    CoffeeDecorator(std::unique_ptr<Coffee> c) : coffee(std::move(c)) {}
};

class MilkDecorator : public CoffeeDecorator {
public:
    MilkDecorator(std::unique_ptr<Coffee> c) : CoffeeDecorator(std::move(c)) {}
    double cost() const override { return coffee->cost() + 0.5; }
};
```

### 2.5. Facade

Provides a simplified interface to a complex body of code.

```cpp
class CPU { public: void freeze() {} void jump(long position) {} void execute() {} };
class Memory { public: void load(long position, const char* data) {} };
class HardDrive { public: char* read(long lba, int size) { return nullptr; } };

class ComputerFacade {
    CPU cpu; Memory memory; HardDrive hd;
public:
    void start() {
        cpu.freeze();
        memory.load(0, hd.read(0, 1024));
        cpu.jump(0);
        cpu.execute();
    }
};
```

### 2.6. Flyweight

Use sharing to support large numbers of fine-grained objects efficiently. Often implemented using a factory with a map caching `std::shared_ptr`.

```cpp
#include <string>
#include <unordered_map>
#include <memory>

class Character {
    char symbol; // Intrinsic state
public:
    Character(char c) : symbol(c) {}
    void display(int fontSize) { /* fontSize is extrinsic */ }
};

class CharacterFactory {
    std::unordered_map<char, std::shared_ptr<Character>> characters;
public:
    std::shared_ptr<Character> getCharacter(char key) {
        if (characters.find(key) == characters.end()) {
            characters[key] = std::make_shared<Character>(key);
        }
        return characters[key];
    }
};
```

### 2.7. Proxy

Provide a surrogate or placeholder to control access to an object. `std::shared_ptr` is technically a proxy for the raw pointer it manages.

```cpp
class Image {
public:
    virtual void display() = 0;
    virtual ~Image() = default;
};

class RealImage : public Image {
    std::string filename;
public:
    RealImage(const std::string& f) : filename(f) { loadFromDisk(); }
    void display() override { /* display image */ }
private:
    void loadFromDisk() { /* Expensive operation */ }
};

class ProxyImage : public Image {
    std::string filename;
    std::unique_ptr<RealImage> realImage;
public:
    ProxyImage(const std::string& f) : filename(f) {}
    void display() override {
        if (!realImage) {
            realImage = std::make_unique<RealImage>(filename);
        }
        realImage->display();
    }
};
```

---

## 3. Behavioral Patterns

Behavioral patterns define how objects interact and assign responsibilities.

### 3.1. Observer (with `std::function`)

Instead of classic OOP inheritance interfaces, Modern C++ often uses `std::function` callbacks.

```cpp
#include <functional>
#include <vector>

class Subject {
    using ObserverCb = std::function<void(int)>;
    std::vector<ObserverCb> observers;
    int state;
public:
    void attach(ObserverCb obs) { observers.push_back(std::move(obs)); }
    void setState(int s) {
        state = s;
        notify();
    }
    void notify() {
        for (auto& obs : observers) obs(state);
    }
};
```

### 3.2. Strategy

Defines a family of algorithms.
**Dynamic (Inheritance):** Uses virtual functions. Overhead of vtable at runtime.
**Static (Templates):** Zero-overhead at runtime.

```cpp
// Static Strategy via Templates
struct FastSort {
    template <typename Container>
    void operator()(Container& c) const { /* Fast sort logic */ }
};

template <typename SortingStrategy>
class Context {
    SortingStrategy strategy;
public:
    template <typename Container>
    void sort(Container& c) {
        strategy(c); // Resolved at compile time
    }
};
```

### 3.3. Command (with Lambdas)

Encapsulates a request as an object. In modern C++, a command is often just a lambda wrapped in `std::function`.

```cpp
#include <queue>
#include <functional>

class Invoker {
    std::queue<std::function<void()>> commands;
public:
    void addCommand(std::function<void()> cmd) { commands.push(std::move(cmd)); }
    void executeAll() {
        while(!commands.empty()) {
            commands.front()();
            commands.pop();
        }
    }
};
```

### 3.4. Iterator

C++ STL has deeply embedded the Iterator pattern. Rather than building OOP iterators (`first()`, `next()`, `isDone()`), C++ relies on the STL protocol (`begin()`, `end()`, `operator++`, `operator*`).

### 3.5. Visitor (std::variant-based)

Classic double-dispatch Visitor requires intrusive `accept()` methods. Modern C++ uses `std::variant` and `std::visit`.

```cpp
#include <variant>
#include <iostream>

struct Circle { void draw() const { std::cout << "Circle\n"; } };
struct Square { void draw() const { std::cout << "Square\n"; } };

using Shape = std::variant<Circle, Square>;

// Overloaded pattern for std::visit
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

void drawShape(const Shape& s) {
    std::visit(overloaded{
        [](const Circle& c) { c.draw(); },
        [](const Square& sq) { sq.draw(); }
    }, s);
}
```

### 3.6. State, Template Method, Chain of Resp., Mediator
These exist but are often replaced by state machines libraries (like Boost.Statechart), functional chaining, or simple lambdas in modern C++.

---

## 4. Modern C++ Idioms

Idioms are C++-specific implementations of common patterns and mechanisms.

### 4.1. RAII (Resource Acquisition Is Initialization)
The absolute core of C++. Acquire resources in constructors, release in destructors. Guarantees safety in the face of exceptions.

### 4.2. CRTP (Curiously Recurring Template Pattern)
Static polymorphism. A class `Derived` derives from a template class `Base<Derived>`.

```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

class MyClass : public Base<MyClass> {
public:
    void implementation() { /* ... */ }
};
```

### 4.3. NVI (Non-Virtual Interface)
Public interfaces should be non-virtual. Virtual methods should be private/protected. Allows the base class to enforce pre- and post-conditions.

```cpp
class Base {
public:
    void doWork() { // Public non-virtual
        // Pre-condition logic
        doWorkImpl();
        // Post-condition logic
    }
private:
    virtual void doWorkImpl() = 0; // Private virtual
};
```

### 4.4. Copy-and-Swap Idiom
Provides strong exception safety and handles self-assignment for classes managing resources.

```cpp
class MyResource {
    int* data;
public:
    // ... constructors / destructor ...
    
    // Copy constructor
    MyResource(const MyResource& other) : data(new int(*other.data)) {}
    
    friend void swap(MyResource& first, MyResource& second) noexcept {
        std::swap(first.data, second.data);
    }
    
    // Unified assignment operator takes by value
    MyResource& operator=(MyResource other) {
        swap(*this, other);
        return *this;
    }
};
```

### 4.5. Scope Guard
A lightweight RAII wrapper that executes a lambda on scope exit. `std::unique_ptr` with custom deleter can act as one.

```cpp
// Defer macro implementation via RAII
auto guard = std::unique_ptr<void, std::function<void(void*)>>(
    (void*)1, [](void*){ /* cleanup logic */ });
```

### 4.6. Type Erasure
Implementing polymorphic behavior without inheritance, akin to `std::any` or `std::function`. Uses template constructors and internal abstract bases.

### 4.7. Policy-Based Design
Using templates to mix and match behaviors (policies) at compile time (popularized by Andrei Alexandrescu).

```cpp
template <typename MemoryPolicy, typename LockPolicy>
class SmartPointer : public MemoryPolicy, public LockPolicy { ... };
```

### 4.8. SFINAE-based Dispatch & Concepts
Using `std::enable_if` or C++20 `requires` to select overloads based on type traits.

```cpp
template<typename T>
requires std::is_integral_v<T>
void process(T val) { /* fast integer path */ }
```

---

## 5. Best Practices (C++ Core Guidelines)

1. **Ownership semantics**: 
   - Use `std::unique_ptr` for exclusive ownership.
   - Use `std::shared_ptr` for shared ownership.
   - Use raw pointers (`T*`) or references (`T&`) to denote non-owning observation. **Never use raw pointers to manage memory.**
2. **const correctness**: Mark functions, parameters, and local variables `const` by default. It aids reasoning and enables compiler optimizations.
3. **Value semantics**: Prefer passing and returning by value and relying on move semantics instead of heap-allocating everything. Objects should "behave like ints".
4. **Small Buffer Optimization (SBO)**: Types like `std::string` and `std::function` avoid heap allocation for small payloads. Design custom types to utilize union buffers for small sizes.
5. **Rule of Zero / Five**: If a class manages no manual resources, let the compiler generate the Big Five (Rule of Zero). If it manages a resource, implement all five (destructor, copy ctor, copy assign, move ctor, move assign).
6. **Prefer Composition over Inheritance**: Inheritance builds tight coupling. Use it mainly to model "is-a" relationships and dynamic interfaces, not for code reuse.

---

## 6. Anti-Patterns to Avoid

- **God Class**: A class that controls too much. Split using Single Responsibility Principle.
- **Raw new/delete**: Always use `std::make_unique` or `std::make_shared`.
- **Overuse of Inheritance**: Especially deep class hierarchies.
- **Exception-unsafe code**: Functions that leak resources or leave objects in an invalid state if an exception is thrown. Use RAII.
- **Premature Optimization**: Unnecessary use of complex template metaprogramming when a simple runtime check suffices. SBO when not needed.
- **Macro Abuse**: Using `#define` for constants or functions. Use `constexpr` and inline functions instead.
- **Static Initialization Order Fiasco**: Relying on the initialization order of global static variables defined in different translation units. Use Meyer's Singleton (function-local statics) instead.

---

## 7. Interview Questions

1. **What is the Static Initialization Order Fiasco, and how does Meyer's Singleton solve it?**
   *Answer*: Global statics in different `.cpp` files are initialized in an undefined order. If Static A depends on Static B, and A initializes first, it crashes. Meyer's Singleton uses a static local variable inside a function. C++ guarantees local statics are initialized the first time control passes through their declaration, resolving the order naturally.

2. **Why is `std::make_shared` preferred over `std::shared_ptr<T>(new T)`?**
   *Answer*: Efficiency and exception safety. `make_shared` performs a single memory allocation for both the control block and the object, whereas the raw pointer version requires two allocations. Also, if a function argument throws before the `shared_ptr` constructor completes, the raw pointer could leak.

3. **Explain the Pimpl idiom and its benefits.**
   *Answer*: "Pointer to Implementation." It hides the private members of a class in a forward-declared struct accessed via a unique_ptr. It breaks compilation dependencies, drastically reducing build times, and maintains ABI stability since the class size doesn't change when private members change.

4. **What is CRTP and when would you use it?**
   *Answer*: The Curiously Recurring Template Pattern (`class Derived : public Base<Derived>`). It provides static polymorphism. The base class can cast `this` to the Derived type and call its methods. Used when you want polymorphic behavior (like interfaces) but cannot afford virtual function call overhead in performance-critical loops.

5. **Why should public methods usually be non-virtual (NVI Idiom)?**
   *Answer*: The Non-Virtual Interface idiom separates interface from implementation. A public non-virtual method allows the base class to enforce invariants, lock mutexes, or log before and after calling a private/protected virtual method that derived classes implement.

6. **What is the Rule of Zero?**
   *Answer*: If your class manages resources exclusively via RAII types (like `std::unique_ptr`, `std::vector`, `std::string`), you should not declare any custom destructors, copy, or move constructors. The compiler-generated defaults will do the right thing, reducing bugs.

7. **How do you implement covariant return types with smart pointers?**
   *Answer*: C++ does not natively support overriding a virtual method returning `unique_ptr<Base>` with one returning `unique_ptr<Derived>`. You implement a private virtual method returning a raw pointer (`Base* clone() const`), and provide a public non-virtual method that calls the private virtual one and wraps the result in a smart pointer.

8. **What is Type Erasure? Name a standard library component that uses it.**
   *Answer*: Type erasure provides a non-templated interface to templated behavior, allowing you to store heterogeneous objects uniformly. `std::function` and `std::any` use type erasure internally via inheritance and template constructors.

9. **Explain the Copy-and-Swap idiom and its main advantage.**
   *Answer*: You define the assignment operator by taking the argument *by value* (which uses the copy constructor), and then swapping its contents with `this`. It automatically handles self-assignment and provides the strong exception guarantee (if copying fails, the current object remains untouched).

10. **Why are Singletons considered anti-patterns by many developers?**
    *Answer*: They hide dependencies (a class can call `Singleton::getInstance()` anywhere, so its API doesn't reflect its dependencies), introduce global mutable state, and make unit testing extremely difficult since the state persists across tests.

11. **Difference between `std::visit` on a `std::variant` and traditional Visitor pattern?**
    *Answer*: Traditional visitor is intrusive (requires modifying classes to add `accept` methods) and handles pointers/polymorphism. `std::variant` is non-intrusive, works with value semantics, and resolution is done at compile time, leading to better performance and no vtable overhead.

12. **How does Small Buffer Optimization (SBO) work in `std::string`?**
    *Answer*: `std::string` contains a small internal array (e.g., 15-22 bytes). If the string is short enough, it stores the characters directly in this array, avoiding a heap allocation (`new`). If it grows larger, it dynamically allocates memory and points to it.

13. **When would you use `std::weak_ptr`?**
    *Answer*: To break cyclical references between `std::shared_ptr`s, which would cause memory leaks. Also used to observe a `shared_ptr` without extending its lifetime, like in a cache.

14. **What is tag dispatching?**
    *Answer*: A metaprogramming technique where empty structs (tags) are used as function arguments to select different overloaded implementations at compile time based on type traits. E.g., `std::advance` uses `std::random_access_iterator_tag` vs `std::forward_iterator_tag`.

15. **Explain "Value Semantics" vs "Reference Semantics".**
    *Answer*: Value semantics means assignment copies the state (objects are independent, like `int`). Reference semantics means assignment copies the pointer (objects share state, like Java objects). Modern C++ favors value semantics for easier reasoning and better cache locality, optimizing it with move semantics.

# 🎬 Episode 01 Script: "What Actually Happens in RAM When You Write C++? (Pointers, Memory & Vtables)"

- **Target Video Length**: 18–20 minutes
- **Playlists**: *C++ Ultra Masterclass*, *Elite C++ Interview Prep*
- **GitHub Module Reference**: [Module 02: Memory Management](file:///C:/Users/abhit/.gemini/antigravity/scratch/cpp-mastery-course/02-pointers-references-memory.md) & [Module 03: OOP & Vtables](file:///C:/Users/abhit/.gemini/antigravity/scratch/cpp-mastery-course/03-oop-deep-dive.md)

---

## ⏱️ Video Breakdown & Teleprompter Script

---

### [0:00 – 1:15] 🎯 Hook: The Cost of a Virtual Function Call

**[VISUAL: Full facecam with sleek dark background. Text overlay: "15ns Latency Trap"]**

> *"If you ask an average C++ programmer how a virtual function works, they'll tell you: 'It enables runtime polymorphism.' But if you ask a senior engineer at a high-frequency trading firm or a game engine studio, they will tell you that a virtual function is an indirect memory load through an array of function pointers that can destroy your CPU's branch target buffer and prevent function inlining.*
>
> *Welcome to C++ Ultra. I'm Abhishek, and in this series, we are tearing down C++ from bare metal to modern C++23. Today, we are not looking at syntax. We are looking at RAM, CPU registers, and the exact assembly generated when you create pointers, allocate memory, and call virtual methods.*
>
> *By the end of this video, you will understand the exact byte-by-byte memory layout of every C++ object you ever create."*

---

### [1:15 – 4:30] 🧠 Act 1: The Memory Layout of a Process

**[VISUAL: Screen switch to Excalidraw / Diagram showing Text, Data, BSS, Heap, Stack]**

> *"Let's start where your program lives: the 64-bit virtual address space. When your operating system boots a C++ binary, it partitions memory into five distinct segments:*
>
> 1. *The **Text Segment**: Read-only executable machine instructions.*
> 2. *The **Data Segment**: Initialized global and static variables.*
> 3. *The **BSS Segment**: Uninitialized static variables (zeroed out by the OS on startup).*
> 4. *The **Heap**: Dynamic memory managed via `malloc`, `new`, or your custom arena allocators.*
> 5. *The **Stack**: High-memory addresses growing downwards, handling local variables and stack frames.*
>
> *Now, let's look at something that trips up 80% of candidates in technical interviews: Memory Alignment and Padding."*

**[VISUAL: Show code snippet in VS Code / Godbolt]**

```cpp
struct BadLayout {
    char a;    // 1 byte
    double b;  // 8 bytes
    int c;     // 4 bytes
    short d;   // 2 bytes
};
```

> *"If you ask a beginner what `sizeof(BadLayout)` is, they add 1 + 8 + 4 + 2 and say 15 bytes. But let's run `sizeof`: it prints **24 bytes**!*
>
> *Why? Because modern 64-bit CPUs read memory in aligned multi-byte words. A `double` requires an 8-byte boundary. The compiler injects 7 invisible padding bytes between `a` and `b`, plus 2 trailing padding bytes at the end so array indexing stays aligned.*
>
> *By simply rearranging the fields by descending size—`double b; int c; short d; char a;`—the struct shrinks from 24 bytes down to **16 bytes**. In a vector of 10 million elements, that simple reordering saves **80 megabytes of memory** and drastically improves L1 cache line residency."*

---

### [4:30 – 10:00] 🔬 Act 2: Inside the Virtual Table (vtable) Engine

**[VISUAL: Diagram showing Base class pointer pointing to Derived object in RAM, with vptr pointing to static vtable array]**

> *"Now let's examine dynamic dispatch. How does the C++ compiler know which function to call when you have a `Base* ptr = new Derived();`?*
>
> *Let's break down the hidden plumbing:*
> 1. *When a class declares or inherits a `virtual` function, the compiler constructs a static table in the read-only data segment called the **vtable**.*
> 2. *Inside the vtable is an array of function pointers pointing to the most-derived implementation of each virtual method.*
> 3. *Every single object of that class gets injected with an invisible hidden member pointer: the **vptr** (virtual pointer).*
>
> *When you call `ptr->speak()`, the CPU executes three sequential steps:*
> - *Step 1: Dereference `ptr` to read the object's `vptr`.*
> - *Step 2: Add the fixed compile-time index offset for `speak()` inside the vtable.*
> - *Step 3: Dereference the function pointer and jump execution to the code address.*
>
> *Let's look at the generated x86-64 assembly in Godbolt to prove it."*

**[VISUAL: Godbolt Compiler Explorer side-by-side with assembly highlighted]**

```assembly
# Calling non-virtual function:
call    Base::non_virtual()     # Direct jump: can be inlined!

# Calling virtual function:
mov     rax, QWORD PTR [rdi]    # Load vptr from object
mov     rax, QWORD PTR [rax]    # Fetch function pointer from vtable[0]
call    rax                     # Indirect call: cannot inline, risks BTB stall!
```

---

### [10:00 – 14:00] ⚠️ Act 3: The 3 Deadliest Virtual Function Traps

> *"Now, let's cover the three classic interview traps that interviewers at FAANG and quant firms love to test:*

#### Trap #1: Calling Virtual Functions in Constructors
> *"What happens if you call a virtual method inside a base class constructor?*
> *The answer: Polymorphism is disabled! During `Base` construction, the `Derived` subobject hasn't been constructed yet. C++ sets the object's `vptr` to point to `Base`'s vtable until construction finishes.*

#### Trap #2: Non-Virtual Base Class Destructor
> *"If you delete a derived object through a `Base*` without a `virtual ~Base()`, the compiler only executes `Base::~Base()`. The derived destructor never runs $\rightarrow$ leading to leaked memory, open socket leaks, and Undefined Behavior.*

#### Trap #3: Default Parameter Binding on Virtual Functions
> *"Look at this tricky code:"*

```cpp
struct Base {
    virtual void print(int x = 10) { std::cout << "Base: " << x; }
};
struct Derived : Base {
    void print(int x = 20) override { std::cout << "Derived: " << x; }
};

Base* p = new Derived();
p->print(); // Prints: "Derived: 10"!
```
> *"Why did it print 10 instead of 20? Because virtual function resolution is dynamic at **runtime**, but default arguments are bound statically at **compile time** from the pointer's type (`Base*`)!"*

---

### [14:00 – 17:30] 💻 Act 4: Live Coding a Custom `UniquePtr` from Scratch

**[VISUAL: VS Code live coding session with typing sound / smooth code reveal]**

> *"To truly understand memory management, you must be able to write your own smart pointer from scratch on a whiteboard. Let's build a minimal `UniquePtr` in 30 seconds:"*

```cpp
template <typename T>
class UniquePtr {
    T* ptr_{nullptr};
public:
    explicit UniquePtr(T* p = nullptr) noexcept : ptr_(p) {}
    ~UniquePtr() noexcept { delete ptr_; }

    // 1. Delete copy semantics (Unique ownership!)
    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;

    // 2. Implement Move semantics
    UniquePtr(UniquePtr&& other) noexcept : ptr_(other.ptr_) {
        other.ptr_ = nullptr;
    }
    UniquePtr& operator=(UniquePtr&& other) noexcept {
        if (this != &other) {
            delete ptr_;
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    T& operator*() const noexcept { return *ptr_; }
    T* operator->() const noexcept { return ptr_; }
};
```

---

### [17:30 – End] 🎯 Outro & Viewer Challenge

**[VISUAL: Facecam with course GitHub repository scrolling on the side]**

> *"Here is your interview challenge for today:*
> *Suppose you have a polymorphic hierarchy where you need the performance of direct function calls with ZERO vtable overhead. How do you implement static polymorphism using CRTP (Curiously Recurring Template Pattern)?*
>
> *Leave your answer in the comments below!*
>
> *The entire code, slides, and all **350+ categorized senior C++ interview questions** are available for free on our GitHub repository linked in the description below.*
>
> *Hit that subscribe button, give this video a like, and in the next episode, we are building a sub-100 nanosecond lock-free SPSC queue for High-Frequency Trading.*
>
> *See you in Episode 2!"*

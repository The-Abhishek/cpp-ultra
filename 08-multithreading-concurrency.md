# Module 08 — Multithreading & Concurrency

Welcome to Module 08. Multithreading and concurrency are among the most complex and critical areas of modern C++ development. With multi-core processors being the norm, writing efficient, bug-free concurrent code is an essential skill for senior C++ developers. This module dives deep into the C++ threading library, synchronization primitives, the memory model, atomic operations, and modern asynchronous patterns.

---

## 1. Thread Basics

### std::thread Creation, Joining, Detaching

The `std::thread` class (introduced in C++11) represents a single thread of execution.

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void worker_function() {
    std::cout << "Worker thread started\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    std::cout << "Worker thread finished\n";
}

int main() {
    std::thread t(worker_function);
    
    // t.join() blocks the calling thread until t finishes execution.
    // Must be called before t's destructor, otherwise std::terminate is called.
    t.join();
    
    std::thread t2([] {
        std::cout << "Detached thread running...\n";
    });
    // t2.detach() separates the thread of execution from the thread object.
    t2.detach();
    
    return 0;
}
```

> [!WARNING]
> If a `std::thread` object is destructed while it is still "joinable" (i.e., you haven't called `join()` or `detach()`), the program will call `std::terminate()`.

### Passing Arguments to Threads

Arguments to the thread function are passed by value by default. If you need to pass by reference, use `std::ref()`.

```cpp
#include <iostream>
#include <thread>

void update_value(int val, int& ref) {
    val++;
    ref++;
}

int main() {
    int v = 10;
    int r = 10;
    
    // Passing r by std::ref to ensure it's passed by reference
    std::thread t(update_value, v, std::ref(r));
    t.join();
    
    std::cout << "v: " << v << " (unchanged)\n";
    std::cout << "r: " << r << " (changed to 11)\n";
    return 0;
}
```

### Thread Identification & Ownership

Threads cannot be copied, only moved. This ensures a 1:1 relationship between a `std::thread` object and an OS thread.

```cpp
#include <iostream>
#include <thread>
#include <vector>

int main() {
    std::cout << "Main thread ID: " << std::this_thread::get_id() << "\n";
    std::cout << "Hardware concurrency: " << std::thread::hardware_concurrency() << "\n";
    
    std::thread t1([]{ std::cout << "Thread 1\n"; });
    // std::thread t2 = t1; // ERROR: std::thread is not copyable
    std::thread t3 = std::move(t1); // OK: Ownership transferred to t3
    
    if (!t1.joinable()) {
        std::cout << "t1 is no longer joinable.\n";
    }
    
    t3.join();
    return 0;
}
```

### std::jthread (C++20)

`std::jthread` automatically joins on destruction and supports cooperative cancellation via `std::stop_token`.

```cpp
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    std::jthread jt([](std::stop_token stoken) {
        while (!stoken.stop_requested()) {
            std::cout << "jthread working...\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(200));
        }
        std::cout << "Stop requested, jthread exiting.\n";
    });
    
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    jt.request_stop(); // Request cooperative cancellation
    
    // No need to call jt.join(); it joins automatically on destruction.
    return 0;
}
```

---

## 2. Mutual Exclusion

### Mutexes & Locks

`std::mutex` provides exclusive, non-recursive ownership semantics. Never use mutexes directly; always use RAII wrappers to prevent deadlocks in case of exceptions.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>

std::mutex mtx;
int shared_counter = 0;

void increment(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        // std::lock_guard provides strict scoped locking
        std::lock_guard<std::mutex> lock(mtx);
        ++shared_counter;
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(increment, 1000);
    }
    for (auto& t : threads) t.join();
    std::cout << "Final counter: " << shared_counter << "\n";
    return 0;
}
```

### std::unique_lock and std::scoped_lock

`std::unique_lock` is more flexible than `std::lock_guard`: it can be unlocked/locked manually and supports deferred locking, but has slight overhead.

`std::scoped_lock` (C++17) is a variadic lock guard that uses a deadlock-avoidance algorithm to lock multiple mutexes simultaneously.

```cpp
#include <mutex>

std::mutex m1, m2;

void thread_safe_transfer() {
    // Locks both m1 and m2 without risk of deadlock (avoids lock ordering issues)
    std::scoped_lock lock(m1, m2);
    // Perform transfer...
}
```

### Deadlock and Double-Checked Locking

A deadlock occurs when two or more threads wait indefinitely for each other to release locks.

Double-Checked Locking (DCLP) was historically broken in C++ until C++11 introduced thread-safe static initialization and the memory model.

```cpp
// Modern, thread-safe Singleton in C++11 onwards (Magic Statics)
class Singleton {
public:
    static Singleton& get_instance() {
        static Singleton instance; // Thread-safe in C++11+
        return instance;
    }
private:
    Singleton() = default;
};
```

### std::shared_mutex (Reader-Writer Lock)

Introduced in C++17, `std::shared_mutex` allows multiple readers or a single writer.

```cpp
#include <iostream>
#include <shared_mutex>
#include <thread>

class ThreadSafeConfig {
    mutable std::shared_mutex rw_mtx;
    int config_value = 0;
public:
    int read() const {
        std::shared_lock<std::shared_mutex> lock(rw_mtx); // Shared lock (readers)
        return config_value;
    }
    
    void write(int v) {
        std::unique_lock<std::shared_mutex> lock(rw_mtx); // Exclusive lock (writer)
        config_value = v;
    }
};
```

---

## 3. Condition Variables

`std::condition_variable` allows threads to wait until a specific condition becomes true. It must be used with `std::unique_lock<std::mutex>`.

### The Lost Wakeup and Spurious Wakeups
- **Spurious wakeup:** A thread might be woken up even if `notify()` was not called. Always use a while loop to check the condition, or use the predicate version of `wait()`.
- **Lost wakeup:** If `notify_one()` is called before the waiting thread reaches `wait()`, the notification is lost.

### Producer-Consumer Pattern

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> data_queue;
bool finished = false;

void producer() {
    for (int i = 0; i < 5; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            data_queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        cv.notify_one();
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    {
        std::lock_guard<std::mutex> lock(mtx);
        finished = true;
    }
    cv.notify_all();
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        // Wait until queue is not empty or production is finished
        cv.wait(lock, []{ return !data_queue.empty() || finished; });
        
        while (!data_queue.empty()) {
            int val = data_queue.front();
            data_queue.pop();
            lock.unlock(); // Unlock while processing
            std::cout << "Consumed: " << val << "\n";
            lock.lock();   // Re-lock
        }
        
        if (finished && data_queue.empty()) break;
    }
}

int main() {
    std::thread p(producer);
    std::thread c(consumer);
    p.join(); c.join();
    return 0;
}
```

---

## 4. Atomic Operations

`std::atomic<T>` provides operations that are guaranteed to be atomic (indivisible).

### Compare-and-Swap (CAS)

CAS is the fundamental building block of lock-free data structures.
- `compare_exchange_weak`: May fail spuriously (even if the expected value matches). Used in loops.
- `compare_exchange_strong`: Guarantees success if the value matches. More expensive on some architectures.

```cpp
#include <atomic>
#include <iostream>

std::atomic<int> current_value(10);

void multiply_by_two() {
    int expected = current_value.load();
    int desired;
    do {
        desired = expected * 2;
        // If current_value == expected, it sets current_value = desired and returns true.
        // Otherwise, it updates expected to the current_value and returns false.
    } while (!current_value.compare_exchange_weak(expected, desired));
}
```

### std::atomic_flag

The only atomic type guaranteed to be lock-free. Used to build spinlocks.

```cpp
class Spinlock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // Spin
        }
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

---

## 5. Memory Model & Memory Ordering

The C++ memory model defines how memory operations in different threads interact. It uses the "happens-before" and "synchronizes-with" relationships.

### Memory Orders
1. `std::memory_order_seq_cst` (Default): Sequential consistency. A single total modification order for all sequentially consistent operations.
2. `std::memory_order_acquire` & `std::memory_order_release`: Used in pairs. A release operation synchronizes with an acquire operation on the same atomic variable.
3. `std::memory_order_relaxed`: No synchronization or ordering guarantees, only atomicity.

```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> ready(false);
int data = 0;

void producer() {
    data = 42; // Non-atomic write
    // Release ensures all prior writes are visible to threads that acquire 'ready'
    ready.store(true, std::memory_order_release);
}

void consumer() {
    // Acquire ensures we see all writes that happened before the release
    while (!ready.load(std::memory_order_acquire));
    assert(data == 42); // Guaranteed to never fire
}
```

---

## 6. Asynchronous Programming

### std::async, std::future, std::promise

`std::async` runs a function asynchronously and returns a `std::future` to retrieve the result.

```cpp
#include <iostream>
#include <future>
#include <thread>

int compute_heavy_task(int x) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return x * x;
}

int main() {
    // std::launch::async forces a new thread.
    // std::launch::deferred executes lazily when get() is called.
    std::future<int> fut = std::async(std::launch::async, compute_heavy_task, 10);
    
    std::cout << "Doing other work...\n";
    
    // get() blocks until the result is ready. Can only be called once.
    int result = fut.get();
    std::cout << "Result: " << result << "\n";
    return 0;
}
```

### std::promise

A `std::promise` provides a way to store a value or an exception that is later acquired asynchronously via a `std::future`.

```cpp
#include <iostream>
#include <future>
#include <thread>

void calculate(std::promise<int> prom) {
    try {
        prom.set_value(42);
    } catch (...) {
        prom.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();
    
    std::thread t(calculate, std::move(prom));
    
    std::cout << "Result from promise: " << fut.get() << "\n";
    t.join();
    return 0;
}
```

---

## 7. Thread Pools

A basic thread pool maintains a queue of tasks and a pool of worker threads.

```cpp
#include <iostream>
#include <vector>
#include <thread>
#include <queue>
#include <functional>
#include <mutex>
#include <condition_variable>

class ThreadPool {
public:
    ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock, [this] { 
                            return this->stop || !this->tasks.empty(); 
                        });
                        if (this->stop && this->tasks.empty()) return;
                        
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    template<class F>
    void enqueue(F&& f) {
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            tasks.emplace(std::forward<F>(f));
        }
        condition.notify_one();
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) worker.join();
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop = false;
};
```

---

## 8. Coroutines (C++20)

Coroutines are stackless functions that can be suspended and resumed. They use `co_await`, `co_yield`, and `co_return`.

### Generator Example

```cpp
#include <iostream>
#include <coroutine>

struct Generator {
    struct promise_type {
        int current_value;
        std::suspend_always yield_value(int value) {
            current_value = value;
            return {};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        Generator get_return_object() { 
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)}; 
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;

    Generator(std::coroutine_handle<promise_type> h) : handle(h) {}
    ~Generator() { if (handle) handle.destroy(); }

    bool move_next() {
        handle.resume();
        return !handle.done();
    }
    int current_value() { return handle.promise().current_value; }
};

Generator counter(int max) {
    for (int i = 0; i < max; ++i) {
        co_yield i; // Suspends coroutine and yields value
    }
}

int main() {
    auto gen = counter(5);
    while (gen.move_next()) {
        std::cout << gen.current_value() << " ";
    }
    return 0;
}
```

---

## 9. Common Concurrency Bugs

1. **Data Race**: When two threads access the same memory location concurrently, at least one is a write, and there is no synchronization. Causes Undefined Behavior (UB).
2. **Race Condition**: A semantic error where the timing of thread execution causes incorrect program behavior.
3. **Deadlock**: Threads form a cycle of waiting for locks held by each other.
4. **False Sharing**: Threads modifying independent variables that reside on the same cache line, causing severe cache invalidation overhead. Align data to cache lines (e.g., `alignas(64)`) to prevent this.
5. **ABA Problem**: In lock-free programming, a memory location is read twice, has the same value, but was modified and changed back in between. Pointer tagging or Hazard Pointers mitigate this.

---

## 10. Interview Questions (25+)

**Q1: What is the difference between a process and a thread?**
A: A process is an independent execution environment with its own memory space. Threads exist within a process and share the same memory space (heap, data), but have their own stack and registers. Context switching between threads is faster.

**Q2: What happens if you forget to join or detach a `std::thread`?**
A: The destructor of `std::thread` checks if it is `joinable()`. If it is, it calls `std::terminate()`, crashing the program. C++20 `std::jthread` solves this by auto-joining in the destructor.

**Q3: Explain a Data Race vs. a Race Condition.**
A: A data race is a low-level hardware/compiler issue where multiple threads access the same memory concurrently without synchronization (at least one write). It is UB. A race condition is a high-level logic flaw where the output depends on the non-deterministic timing of threads (e.g., interleaving bank transactions).

**Q4: How does `std::lock_guard` differ from `std::unique_lock`?**
A: `std::lock_guard` is a strict RAII wrapper—it locks on construction and unlocks on destruction. `std::unique_lock` allows deferred locking, manual locking/unlocking, and condition variable integration, but carries a slight performance and memory overhead.

**Q5: What is a deadlock and how do you prevent it in C++?**
A: Deadlock occurs when threads wait on each other indefinitely. Prevention techniques:
- Lock ordering: Always acquire locks in the exact same order across all threads.
- Use `std::scoped_lock` (or `std::lock`), which uses deadlock-avoidance algorithms to lock multiple mutexes safely.
- Avoid holding locks during user code callbacks.

**Q6: What is a spurious wakeup? How do you handle it?**
A: Condition variables may wake up a waiting thread even if `notify()` wasn't called (due to OS scheduling/POSIX implementation details). Handle it by always wrapping the `wait()` call in a `while` loop checking a boolean predicate, or use the lambda predicate version of `std::condition_variable::wait`.

**Q7: Why must `std::condition_variable::wait` take a `std::unique_lock`?**
A: `wait()` needs to atomically unlock the mutex and put the thread to sleep, then re-lock it when woken up. `std::unique_lock` provides the `unlock()` and `lock()` methods required for this, whereas `lock_guard` does not.

**Q8: Explain Compare-And-Swap (CAS).**
A: CAS atomically compares the contents of a memory location with an expected value, and if they match, modifies the contents to a new desired value. It's the foundation of lock-free data structures. In C++, this is `compare_exchange_weak`/`strong`.

**Q9: When should you use `compare_exchange_weak` vs `strong`?**
A: Use `weak` inside loops because it can fail spuriously (due to CPU architecture reasons, like LL/SC instructions on ARM), but is cheaper. Use `strong` when you calculate the desired value outside a loop and want to try exchanging exactly once.

**Q10: What is the ABA problem?**
A: Thread 1 reads value A. Thread 2 changes A to B, then back to A. Thread 1 performs a CAS expecting A, succeeds, but the context has changed (e.g., in a lock-free stack, node A was deleted and reallocated). Mitigated by double-width CAS (adding a version counter), Hazard Pointers, or epoch-based reclamation.

**Q11: What is False Sharing?**
A: When multiple threads modify independent variables that happen to share the same CPU cache line (typically 64 bytes). The CPU hardware invalidates the entire cache line across cores, causing a massive performance hit. Fixed by `alignas(64)` padding between variables.

**Q12: What does `std::memory_order_acquire` and `std::memory_order_release` do?**
A: They form a synchronization pair. A `store(release)` ensures that all prior memory writes (even non-atomic ones) in the current thread become visible to another thread that performs a `load(acquire)` on the same atomic variable. It prevents compiler and CPU instruction reordering across the atomic operation.

**Q13: What is the difference between `notify_one()` and `notify_all()`?**
A: `notify_one()` unblocks one waiting thread. `notify_all()` unblocks all of them. Use `notify_all()` when multiple threads might be waiting on different conditions governed by the same CV, or when state changes such that multiple threads can now proceed (e.g., a shared reader-writer lock).

**Q14: Explain `std::async` vs `std::thread`.**
A: `std::thread` is a low-level OS thread wrapper; it does not return values easily and exceptions terminate the program. `std::async` is a higher-level abstraction that returns a `std::future`, allowing easy retrieval of return values and propagation of exceptions.

**Q15: What is the "Lost Wakeup" problem?**
A: If Thread A calls `notify_one()` before Thread B has actually executed `cv.wait()`, the notification is lost. Thread B will block forever. Solved by always maintaining shared state (a boolean flag) protected by the mutex, and checking it before waiting.

**Q16: How does `std::shared_mutex` work?**
A: It provides a Multiple-Readers / Single-Writer lock. Readers use `std::shared_lock<std::shared_mutex>` (concurrent access). Writers use `std::unique_lock<std::shared_mutex>` (exclusive access). Useful when reads heavily outnumber writes.

**Q17: What is Thread Local Storage (TLS)?**
A: Using the `thread_local` keyword, a variable gets a separate instance for every thread. It is initialized upon thread creation and destroyed when the thread exits. Useful for per-thread caches or random number generators.

**Q18: What is a lock-free data structure?**
A: A concurrent data structure where the suspension of one thread never prevents other threads from making progress. They use atomic operations (like CAS) instead of mutexes.

**Q19: What is a wait-free data structure?**
A: A stronger guarantee than lock-free: *every* thread is guaranteed to make progress and complete its operation within a bounded number of steps, regardless of what other threads are doing. Extremely difficult to implement.

**Q20: Why is double-checked locking an anti-pattern in C++03 but safe in C++11?**
A: In C++03, there was no memory model. The compiler/CPU could reorder the object initialization and pointer assignment, causing another thread to see a non-null pointer to an uninitialized object. C++11 introduced the memory model and thread-safe static initialization.

**Q21: How do you pass arguments to a `std::thread` by reference?**
A: You must wrap the argument in `std::ref()` or `std::cref()`. By default, `std::thread` constructor copies/moves arguments into the thread's internal storage.

**Q22: Explain the significance of `std::this_thread::yield()`.**
A: It provides a hint to the OS scheduler that the current thread is willing to relinquish its remaining CPU time slice, allowing other threads to run. Used in spinlocks to reduce CPU hogging while waiting.

**Q23: How do you handle exceptions in worker threads?**
A: If an exception escapes a `std::thread` function, `std::terminate` is called. You must catch exceptions inside the thread. Alternatively, use `std::promise` to pass `std::current_exception()` to the caller, or use `std::async`/`std::future` which does this automatically.

**Q24: What is `std::packaged_task`?**
A: It wraps any callable target so that it can be invoked asynchronously. Its return value or exception is stored in a shared state accessible through a `std::future`. It bridges the gap between `std::thread` and `std::future`.

**Q25: Describe how C++20 coroutines differ from threads.**
A: Threads are preemptively scheduled by the OS, require a large dedicated stack (e.g., 1MB), and context switching is costly. Coroutines are cooperatively scheduled by the application, are stackless (state allocated on the heap), and suspension/resumption is as cheap as a function call.

---
*End of Module 08*

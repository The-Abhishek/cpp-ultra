# Design Task Scheduler (like cron)

## 1. Requirements & Use Cases
- **Actors:** User/Application
- **Use Cases:** Schedule one-time task, schedule recurring task, cancel task.

## 2. Terminology & Entities
- **Cron [Time-based job scheduler]:** Standard tool for recurring tasks.
- **Priority Queue [Data structure where elements are dequeued based on priority]:** Used to efficiently find the next task to run.
- **Entities:** Scheduler, Task, RecurringTask, TaskQueue.

## 3. Design Patterns & Principles
- **Command Pattern:** Task encapsulation. Why? Wraps the execution logic into an object that can be stored and executed later.
- **Observer Pattern:** Task completion notifications.
- **SOLID Principles:** Liskov Substitution - RecurringTask can be used wherever a Task is expected.

## 4. ASCII Class Diagram
```text
+-------------------+        +----------------+
|    Scheduler      |<>------|     Task       |
+-------------------+        +----------------+
| - taskQueue (MinH)|        | - executeTime  |
| - workerThread    |        | + execute()    |
| + schedule()      |        +----------------+
+-------------------+               ^
                                    |
                             +----------------+
                             | RecurringTask  |
                             +----------------+
```

## 5. Deep Dive: Min-Heap Based Scheduler
- **Why min-heap?**
  - Insertion: O(log n)
  - Peek next task: O(1) (root of the heap)
  - Extraction: O(log n)
- **Worker thread logic:** 
  - Sleeps until the root element's execution time using a `condition_variable`.
  - Wakes up early if a new task with an earlier time is scheduled.
- **Recurring:** Executes, calculates next time, re-inserts into heap.

## 6. Full Working C++ Implementation
```cpp
#include <iostream>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <chrono>

using TimePoint = std::chrono::time_point<std::chrono::system_clock>;

struct Task {
    TimePoint executeTime;
    std::function<void()> func;
    
    // Min-heap ordering (earliest time first)
    bool operator>(const Task& other) const {
        return executeTime > other.executeTime;
    }
};

class TaskScheduler {
    std::priority_queue<Task, std::vector<Task>, std::greater<Task>> taskQueue;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop;
    std::thread worker;

    void run() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx);
            
            if (stop && taskQueue.empty()) break;
            
            if (taskQueue.empty()) {
                cv.wait(lock);
            } else {
                auto nextTask = taskQueue.top();
                auto now = std::chrono::system_clock::now();
                
                if (nextTask.executeTime <= now) {
                    taskQueue.pop();
                    lock.unlock();
                    nextTask.func(); // Execute command
                } else {
                    cv.wait_until(lock, nextTask.executeTime);
                }
            }
        }
    }

public:
    TaskScheduler() : stop(false) {
        worker = std::thread(&TaskScheduler::run, this);
    }

    ~TaskScheduler() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            stop = true;
        }
        cv.notify_one();
        if (worker.joinable()) worker.join();
    }

    void schedule(std::function<void()> taskFunc, int delaySec) {
        auto executeTime = std::chrono::system_clock::now() + std::chrono::seconds(delaySec);
        
        std::lock_guard<std::mutex> lock(mtx);
        taskQueue.push({executeTime, taskFunc});
        cv.notify_one(); // Wake up thread if it's sleeping
    }
};

int main() {
    TaskScheduler scheduler;
    
    std::cout << "Scheduling task for 2 seconds...\n";
    scheduler.schedule([](){ std::cout << "Task 1 Executed!\n"; }, 2);
    
    std::cout << "Scheduling task for 1 second...\n";
    scheduler.schedule([](){ std::cout << "Task 2 Executed!\n"; }, 1);
    
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return 0;
}
```

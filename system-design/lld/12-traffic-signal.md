# Design Traffic Signal Controller

## 1. Requirements & Use Cases
- **Actors:** System (automated), Traffic Admin, Sensors
- **Use Cases:** Cycle through signals (Green→Yellow→Red), handle emergency vehicles, pedestrian crossing.

## 2. Terminology & Entities
- **FSM [Finite State Machine]:** A model of computation with a finite number of states. Used for light transitions.
- **Deadlock [A situation where no progress can be made]:** Must be prevented at intersections.
- **Entities:** TrafficController, Intersection, Signal, Road, Timer.

## 3. Design Patterns & Principles
- **State Pattern:** Signal states (GREEN/YELLOW/RED). Why? Avoids massive switch statements for transitions.
- **Observer Pattern:** Notify connected signals when a state changes. Why? Easy synchronization.
- **Strategy Pattern:** Timing strategy (fixed-time, adaptive). Why? Allows swapping algorithms based on time of day.
- **SOLID Principles:** Open/Closed Principle applied to timing strategies—can add new ones without altering the controller.

## 4. ASCII Class Diagram
```text
+-------------------+       +-----------------+
| TrafficController |<------| TimingStrategy  |
+-------------------+       +-----------------+
| - signals         |       | + getDuration() |
| - start()         |       +-----------------+
+-------------------+
        |
        v
+-------------------+       +-----------------+
|      Signal       |------>|   SignalState   |
+-------------------+       +-----------------+
| - id              |       | + nextState()   |
| - currentState    |       +-----------------+
| - changeState()   |
+-------------------+
```

## 5. Deep Dive: Signal Synchronization
Ensuring conflicting directions don't both go green:
- Pair opposing directions (North-South, East-West).
- Use a central controller that explicitly turns off one pair before activating the other, including a clearance interval (all-red state).

## 6. Full Working C++ Implementation
```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>
#include <memory>
#include <mutex>

enum class Light { RED, GREEN, YELLOW };

class Signal {
    std::string direction;
    Light state;
    std::mutex mtx;
public:
    Signal(std::string dir) : direction(dir), state(Light::RED) {}
    
    void setState(Light newState) {
        std::lock_guard<std::mutex> lock(mtx);
        state = newState;
        std::string color = (state == Light::RED) ? "RED" : (state == Light::GREEN) ? "GREEN" : "YELLOW";
        std::cout << "Signal " << direction << " is now " << color << "\n";
    }
    
    Light getState() {
        std::lock_guard<std::mutex> lock(mtx);
        return state;
    }
};

class IntersectionController {
    std::shared_ptr<Signal> nsSignal; // North-South
    std::shared_ptr<Signal> ewSignal; // East-West
    bool running;
public:
    IntersectionController() : running(true) {
        nsSignal = std::make_shared<Signal>("North-South");
        ewSignal = std::make_shared<Signal>("East-West");
    }
    
    void runCycle() {
        while (running) {
            // NS Green, EW Red
            ewSignal->setState(Light::RED);
            nsSignal->setState(Light::GREEN);
            std::this_thread::sleep_for(std::chrono::seconds(2)); // Green time
            
            // NS Yellow
            nsSignal->setState(Light::YELLOW);
            std::this_thread::sleep_for(std::chrono::seconds(1)); // Yellow time
            
            // NS Red, EW Green
            nsSignal->setState(Light::RED);
            ewSignal->setState(Light::GREEN);
            std::this_thread::sleep_for(std::chrono::seconds(2)); // Green time
            
            // EW Yellow
            ewSignal->setState(Light::YELLOW);
            std::this_thread::sleep_for(std::chrono::seconds(1)); // Yellow time
            
            running = false; // Stop after 1 cycle for demo
        }
    }
};

int main() {
    IntersectionController controller;
    std::cout << "Starting Traffic Signal Cycle:\n";
    controller.runCycle();
    return 0;
}
```

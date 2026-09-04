# Elevator System LLD

## 1. Requirements Collection
*   **Actors**: Passenger, Building Admin.
*   **Use Cases**: Request elevator from a floor, select destination inside, elevator scheduling.
*   **Entities**: `ElevatorSystem`, `Elevator`, `Floor`, `Request`, `Direction`.

## 2. Terminology & Concepts
*   **SCAN (Elevator Algorithm)**: Goes in one direction until all requests in that direction are fulfilled, then reverses.
*   **LOOK**: Similar to SCAN, but reverses at the *last request* in a direction, not at the physical end of the building.
*   **SSTF (Shortest Seek Time First)**: Always serves the closest request next. Can cause starvation.
*   **Thread Safety**: Utilizing `std::mutex` and `std::condition_variable` to handle asynchronous passenger requests without race conditions.

## 3. Class Design & ASCII Diagram
```text
+-------------------+      manages     +-------------------+
|  ElevatorSystem   |----------------->|    Elevator       |
+-------------------+                  +-------------------+
| - elevators: List |                  | - id: int         |
| - mtx: mutex      |                  | - currFloor: int  |
+-------------------+                  | - dir: Direction  |
| + requestElevator()                  | - state: State    |
| + step()          |                  | - upQueue: pq     |
+-------------------+                  | - downQueue: pq   |
                                       +-------------------+
                                       | + addRequest()    |
                                       | + move()          |
                                       +-------------------+
        |
        | uses Strategy
        v
+-------------------+
|  IElevatorAlgo    |
+-------------------+
| + schedule()      |
+-------------------+
```

## 4. Design Patterns & SOLID Principles
*   **Strategy Pattern**: For `IElevatorAlgo` (SCAN, LOOK). Allows changing dispatch logic without touching the Elevator class.
*   **State Pattern**: Elevator states (IDLE, MOVING_UP, MOVING_DOWN). Transitions dictate behavior.
*   **Observer Pattern**: Display boards on floors observe the elevator's current floor.

**SOLID Checklist**:
*   **SRP**: `Elevator` handles movement, `ElevatorSystem` handles routing requests.
*   **OCP**: New algorithms can be added by implementing `IElevatorAlgo`.

## 5. Full C++ Implementation (SCAN Algorithm focus)
```cpp
#include <iostream>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <chrono>

using namespace std;

enum class Direction { UP, DOWN, IDLE };
enum class State { IDLE, MOVING, STOPPED };

struct Request {
    int floor;
    Direction dir;
    Request(int f, Direction d) : floor(f), dir(d) {}
};

class Elevator {
private:
    int id;
    int currentFloor;
    Direction direction;
    State state;
    
    // Min-heap for UP requests
    priority_queue<int, vector<int>, greater<int>> upRequests;
    // Max-heap for DOWN requests
    priority_queue<int, vector<int>, less<int>> downRequests;
    
    mutex mtx;

public:
    Elevator(int _id) : id(_id), currentFloor(0), direction(Direction::IDLE), state(State::IDLE) {}

    void addRequest(int targetFloor) {
        lock_guard<mutex> lock(mtx);
        if (targetFloor > currentFloor) {
            upRequests.push(targetFloor);
            if (direction == Direction::IDLE) direction = Direction::UP;
        } else if (targetFloor < currentFloor) {
            downRequests.push(targetFloor);
            if (direction == Direction::IDLE) direction = Direction::DOWN;
        }
    }

    void run() {
        while (true) {
            int nextFloor = -1;
            {
                lock_guard<mutex> lock(mtx);
                if (direction == Direction::UP && !upRequests.empty()) {
                    nextFloor = upRequests.top();
                    upRequests.pop();
                } else if (direction == Direction::DOWN && !downRequests.empty()) {
                    nextFloor = downRequests.top();
                    downRequests.pop();
                } else {
                    // Switch direction if needed (LOOK algorithm logic)
                    if (direction == Direction::UP && !downRequests.empty()) {
                        direction = Direction::DOWN;
                    } else if (direction == Direction::DOWN && !upRequests.empty()) {
                        direction = Direction::UP;
                    } else {
                        direction = Direction::IDLE;
                        state = State::IDLE;
                    }
                }
            }

            if (nextFloor != -1) {
                state = State::MOVING;
                cout << "Elevator " << id << " moving from " << currentFloor << " to " << nextFloor << "\n";
                this_thread::sleep_for(chrono::milliseconds(500)); // Simulate movement
                currentFloor = nextFloor;
                cout << "Elevator " << id << " reached floor " << currentFloor << "\n";
            } else {
                this_thread::sleep_for(chrono::milliseconds(100)); // Idle wait
            }
        }
    }
};

int main() {
    Elevator e1(1);
    
    // Run elevator in a background thread
    thread t1(&Elevator::run, &e1);

    // Simulate requests
    e1.addRequest(5);
    e1.addRequest(3);
    e1.addRequest(8);
    
    this_thread::sleep_for(chrono::seconds(2));
    e1.addRequest(2); // Should go down after reaching 8

    t1.detach(); // For demonstration purposes
    this_thread::sleep_for(chrono::seconds(5));
    
    return 0;
}
```

## 6. Interview Tips & Follow-up Questions
*   **Q: How do you prevent thread starvation when adding requests?**
    *   A: The mutex lock scope is kept minimal (only around queue modifications). Real implementations might use condition variables to wake sleeping elevators.
*   **Q: Why LOOK over SCAN?**
    *   A: SCAN goes to the very top/bottom floor even if there are no requests there. LOOK reverses at the highest/lowest requested floor, saving time and energy.
*   **Tip**: Always mention hardware constraints—Elevators are slow, software is fast. Therefore, optimizing for seek time (elevator travel time) is paramount.


# Design Movie Ticket Booking (BookMyShow)

## 1. Requirements & Use Cases
- **Actors:** Customer, Admin
- **Use Cases:** Search movies, view showtimes, select seats, book, pay, cancel.

## 2. Terminology & Entities
- **Concurrency [Executing multiple computations simultaneously]:** Crucial for preventing double booking.
- **ACID [Atomicity, Consistency, Isolation, Durability]:** DB properties needed for transactions.
- **Entities:** Movie, Cinema, Screen, Show, Seat, Booking, Payment.

## 3. Design Patterns & Principles
- **Strategy Pattern:** Pricing varies (normal, premium, VIP). Why? Allows adding new pricing models without changing existing code.
- **Observer Pattern:** Send notifications upon booking confirmation. Why? Decouples booking logic from notification delivery.
- **Locking Pattern:** Prevent double-booking. Why? Handles concurrent access to the same seat.
- **SOLID Principles:** Single Responsibility applied by separating Payment, Booking, and Notification logic.

## 4. ASCII Class Diagram
```text
+----------------+        +-----------------+       +----------------+
|    Movie       |<-------|      Show       |------>|      Screen    |
+----------------+        +-----------------+       +----------------+
| - title        |        | - startTime     |       | - screenId     |
| - duration     |        | - endTime       |       | - totalSeats   |
+----------------+        +-----------------+       +----------------+
                                 |
                                 v
                          +-----------------+
                          |     Seat        |
                          +-----------------+
                          | - seatId        |
                          | - status        |
                          | - lock()        |
                          | - unlock()      |
                          +-----------------+
```

## 5. Deep Dive: Concurrent Seat Booking
How to prevent two users booking the same seat:
- **Pessimistic locking:** Lock seat row in DB during selection (e.g., `SELECT ... FOR UPDATE`). Blocks other reads but prevents dirty reads.
- **Optimistic locking:** Version check on commit. Fails and retries if the version changed. Good for read-heavy systems.
- **Temporary hold:** Redis TTL-based seat hold. Holds seat for 10 mins during payment to avoid deadlocks.

## 6. Full Working C++ Implementation
```cpp
#include <iostream>
#include <string>
#include <vector>
#include <mutex>
#include <memory>
#include <thread>

enum class SeatStatus { AVAILABLE, RESERVED, BOOKED };

class Seat {
    std::string id;
    SeatStatus status;
    std::mutex mtx;
public:
    Seat(std::string id) : id(id), status(SeatStatus::AVAILABLE) {}
    
    std::string getId() const { return id; }
    
    // Thread-safe reserve
    bool reserve() {
        std::lock_guard<std::mutex> lock(mtx);
        if (status == SeatStatus::AVAILABLE) {
            status = SeatStatus::RESERVED;
            return true;
        }
        return false;
    }
    
    void book() {
        std::lock_guard<std::mutex> lock(mtx);
        if (status == SeatStatus::RESERVED) {
            status = SeatStatus::BOOKED;
        }
    }
};

class Show {
    std::string id;
    std::vector<std::shared_ptr<Seat>> seats;
public:
    Show(std::string id) : id(id) {}
    void addSeat(std::shared_ptr<Seat> seat) { seats.push_back(seat); }
    
    bool bookSeat(const std::string& seatId) {
        for (auto& seat : seats) {
            if (seat->getId() == seatId) {
                if (seat->reserve()) {
                    std::cout << "Thread " << std::this_thread::get_id() << " reserved seat " << seatId << "\n";
                    // Simulate payment delay
                    std::this_thread::sleep_for(std::chrono::milliseconds(100));
                    seat->book();
                    std::cout << "Thread " << std::this_thread::get_id() << " booked seat " << seatId << "\n";
                    return true;
                }
            }
        }
        std::cout << "Thread " << std::this_thread::get_id() << " failed to reserve seat " << seatId << "\n";
        return false;
    }
};

int main() {
    std::shared_ptr<Show> show = std::make_shared<Show>("SHOW_1");
    show->addSeat(std::make_shared<Seat>("A1"));
    
    auto user1 = [&]() { show->bookSeat("A1"); };
    auto user2 = [&]() { show->bookSeat("A1"); };
    
    std::thread t1(user1);
    std::thread t2(user2);
    
    t1.join();
    t2.join();
    
    return 0;
}
```

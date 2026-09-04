# Design a Hotel Management System

## Step 1: Requirements Clarification
*   **Actors:** Guest, Receptionist, Admin
*   **Use Cases:** Search rooms, book room, check-in, check-out, cancel booking
*   **Scale:** Moderate [QPS (Queries Per Second) < 100], strong consistency needed for bookings.

> **Interview Tip:** Always ask about scale and concurrency early on to establish whether you need strong locks or if optimistic concurrency control is sufficient.

## Step 2: Object Identification
*   **Entities:** `Hotel`, `Room`, `Booking`, `Guest`, `Payment`
*   **Enums:** `RoomType` (Single, Double, Suite), `RoomStatus` (AVAILABLE, RESERVED, OCCUPIED, MAINTENANCE)

## Step 3: Class Diagram (ASCII)
```text
+---------------+       1..* +------------+       +----------------+
|     Hotel     |----------->|    Room    |       |  RoomStatus    |
+---------------+            +------------+       +----------------+
| -name         |            | -id        |       | AVAILABLE      |
| -location     |            | -type      |       | RESERVED       |
| -rooms        |            | -status    |       | OCCUPIED       |
| +searchRoom() |            | -price     |       | MAINTENANCE    |
+---------------+            +------------+       +----------------+
                                   ^
                                   | 1..1
+---------------+       1..* +------------+       +----------------+
|    Guest      |<-----------|  Booking   |------>|    Payment     |
+---------------+            +------------+ 1..1  +----------------+
| -id           |            | -bookingId |       | -amount        |
| -name         |            | -guest     |       | -status        |
| -email        |            | -room      |       | +process()     |
+---------------+            | -checkIn   |       +----------------+
                             | -checkOut  |
                             +------------+
```

## Step 4: Core APIs / Interfaces
*   `searchRooms(RoomType type, Date from, Date to)`
*   `bookRoom(Guest guest, Room room, Date from, Date to)`
*   `checkIn(Booking booking)`
*   `checkOut(Booking booking)`

## Step 5: Design Patterns
*   **State Pattern:** Room states (AVAILABLE, RESERVED, OCCUPIED). *Why?* Centralizes state transitions and prevents illegal state changes (e.g., checking into a maintenance room).
*   **Strategy Pattern:** Pricing strategy (Weekday, Weekend, Holiday). *Why?* Allows changing pricing algorithms dynamically.
*   **Observer Pattern:** Booking notifications. *Why?* Decouples the core booking logic from email/SMS notification services.

## Step 6: Code Implementation (C++)

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <mutex>
#include <memory>
#include <stdexcept>

// Terminology:
// [Concurrency] Ensuring thread-safety using mutexes when booking a room to avoid double booking.
// [SOLID] Single Responsibility Principle used throughout class definitions.

enum class RoomType { SINGLE, DOUBLE, SUITE };
enum class RoomStatus { AVAILABLE, RESERVED, OCCUPIED, MAINTENANCE };

class Room {
private:
    std::string id;
    RoomType type;
    RoomStatus status;
    double basePrice;
    std::mutex roomMutex;

public:
    Room(std::string id, RoomType type, double price) 
        : id(id), type(type), status(RoomStatus::AVAILABLE), basePrice(price) {}

    std::string getId() const { return id; }
    RoomStatus getStatus() const { return status; }
    RoomType getType() const { return type; }

    bool reserve() {
        std::lock_guard<std::mutex> lock(roomMutex); // Thread safety
        if (status == RoomStatus::AVAILABLE) {
            status = RoomStatus::RESERVED;
            return true;
        }
        return false;
    }

    void checkIn() {
        std::lock_guard<std::mutex> lock(roomMutex);
        if (status == RoomStatus::RESERVED) {
            status = RoomStatus::OCCUPIED;
        } else {
            throw std::runtime_error("Room must be reserved before check-in.");
        }
    }

    void checkOut() {
        std::lock_guard<std::mutex> lock(roomMutex);
        if (status == RoomStatus::OCCUPIED) {
            status = RoomStatus::AVAILABLE;
        }
    }
};

class Guest {
public:
    std::string name;
    std::string email;
    Guest(std::string n, std::string e) : name(n), email(e) {}
};

class PricingStrategy {
public:
    virtual double calculatePrice(double basePrice, int days) = 0;
    virtual ~PricingStrategy() = default;
};

class RegularPricing : public PricingStrategy {
public:
    double calculatePrice(double basePrice, int days) override {
        return basePrice * days;
    }
};

class Booking {
private:
    std::string bookingId;
    Guest guest;
    std::shared_ptr<Room> room;
    int durationDays;
    std::shared_ptr<PricingStrategy> pricing;

public:
    Booking(std::string id, Guest g, std::shared_ptr<Room> r, int days, std::shared_ptr<PricingStrategy> p)
        : bookingId(id), guest(g), room(r), durationDays(days), pricing(p) {}

    void confirm() {
        if (room->reserve()) {
            std::cout << "Booking confirmed for " << guest.name << ". Total cost: $" 
                      << pricing->calculatePrice(100.0, durationDays) << "\n";
        } else {
            std::cout << "Booking failed. Room unavailable.\n";
        }
    }
    
    void checkIn() {
        room->checkIn();
        std::cout << guest.name << " checked in.\n";
    }

    void checkOut() {
        room->checkOut();
        std::cout << guest.name << " checked out.\n";
    }
};

// Hotel Manager
class Hotel {
private:
    std::vector<std::shared_ptr<Room>> rooms;

public:
    void addRoom(std::shared_ptr<Room> room) {
        rooms.push_back(room);
    }

    std::shared_ptr<Room> searchAvailableRoom(RoomType type) {
        for (auto& room : rooms) {
            if (room->getType() == type && room->getStatus() == RoomStatus::AVAILABLE) {
                return room;
            }
        }
        return nullptr;
    }
};

int main() {
    Hotel hotel;
    auto room1 = std::make_shared<Room>("101", RoomType::SINGLE, 100.0);
    hotel.addRoom(room1);

    Guest guest("John Doe", "john@example.com");
    auto pricing = std::make_shared<RegularPricing>();

    auto availableRoom = hotel.searchAvailableRoom(RoomType::SINGLE);
    if (availableRoom) {
        Booking booking("B001", guest, availableRoom, 3, pricing);
        booking.confirm();
        booking.checkIn();
        booking.checkOut();
    }

    return 0;
}
```

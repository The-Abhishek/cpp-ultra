# Parking Lot LLD

## 1. Requirements Collection
*   **Actors**: Driver (user parking vehicle), Admin (manages parking lot).
*   **Use Cases**:
    *   Enter lot and get a ticket.
    *   Exit lot, calculate fee, and pay.
    *   Check spot availability.
    *   Add/remove parking spots.
*   **Entities**: `ParkingLot`, `ParkingFloor`, `ParkingSpot`, `Vehicle`, `Ticket`, `Payment`.

## 2. Terminology & Concepts
*   **LLD (Low-Level Design)**: Detailed design of individual components, classes, and their relationships.
*   **Design Patterns**: Reusable solutions to common software design problems.
*   **Concurrency**: Handling multiple operations simultaneously, critical for enter/exit points.
*   **SOLID Principles**: Five principles of object-oriented design (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion).

## 3. Class Design & ASCII Diagram
```text
+-------------------+       1..n +-------------------+       1..n +-------------------+
|    ParkingLot     |----------->|   ParkingFloor    |----------->|   ParkingSpot     |
+-------------------+            +-------------------+            +-------------------+
| - instance: Lot*  |            | - floorId: int    |            | - spotId: int     |
| - floors: vector  |            | - spots: vector   |            | - type: SpotType  |
| - mtx: shared_mtx |            +-------------------+            | - isFree: bool    |
+-------------------+            | + getFreeSpot()   |            | - vehicle: Veh*   |
| + getInstance()   |            +-------------------+            +-------------------+
| + enter(Vehicle)  |                                             | + assign()        |
| + exit(Ticket)    |                                             | + remove()        |
+-------------------+                                             +-------------------+
        |
        | uses
        v
+-------------------+       generates      +-------------------+
| IPricingStrategy  |<---------------------|     Ticket        |
+-------------------+                      +-------------------+
| + calculate()     |                      | - ticketId: str   |
+-------------------+                      | - entryTime: time |
        ^                                  | - spot: Spot*     |
        | implements                       +-------------------+
+-------------------+
| HourlyPricing     |
+-------------------+
```

## 4. Design Patterns & SOLID Principles
*   **Singleton Pattern**: Ensure only one `ParkingLot` instance manages everything.
*   **Strategy Pattern**: `IPricingStrategy` allows flexible fee calculations (hourly, flat rate) without changing core logic (Open/Closed Principle).
*   **Factory Pattern**: Creating different types of `Vehicle` and `ParkingSpot`.
*   **Observer Pattern**: Useful for updating electronic display boards when a spot is taken/freed.

**SOLID Checklist**:
*   **SRP (Single Responsibility Principle)**: `Ticket` handles entry info, `Payment` handles checkout.
*   **OCP (Open/Closed Principle)**: Add new pricing via `IPricingStrategy` without modifying `ParkingLot`.
*   **DIP (Dependency Inversion Principle)**: Depend on abstractions (`IPricingStrategy`), not concretions.

## 5. Full C++ Implementation
```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <mutex>
#include <shared_mutex>
#include <chrono>

using namespace std;

// Enums
enum class VehicleType { CAR, TRUCK, MOTORCYCLE };
enum class SpotType { COMPACT, LARGE, MOTORCYCLE };

// Entities
class Vehicle {
    string licensePlate;
    VehicleType type;
public:
    Vehicle(string plate, VehicleType t) : licensePlate(plate), type(t) {}
    VehicleType getType() const { return type; }
    string getPlate() const { return licensePlate; }
};

class ParkingSpot {
    int id;
    SpotType type;
    bool isFree;
    Vehicle* currentVehicle;
public:
    ParkingSpot(int _id, SpotType _type) : id(_id), type(_type), isFree(true), currentVehicle(nullptr) {}
    bool getIsFree() const { return isFree; }
    SpotType getType() const { return type; }
    
    void assignVehicle(Vehicle* v) {
        isFree = false;
        currentVehicle = v;
    }
    
    void removeVehicle() {
        isFree = true;
        currentVehicle = nullptr;
    }
    int getId() const { return id; }
};

class Ticket {
    string id;
    chrono::system_clock::time_point entryTime;
    ParkingSpot* spot;
public:
    Ticket(string _id, ParkingSpot* _spot) : id(_id), spot(_spot) {
        entryTime = chrono::system_clock::now();
    }
    ParkingSpot* getSpot() const { return spot; }
    auto getEntryTime() const { return entryTime; }
};

// Strategy Pattern for Pricing
class IPricingStrategy {
public:
    virtual double calculateFee(Ticket* ticket) = 0;
    virtual ~IPricingStrategy() = default;
};

class HourlyPricing : public IPricingStrategy {
public:
    double calculateFee(Ticket* ticket) override {
        auto now = chrono::system_clock::now();
        auto duration = chrono::duration_cast<chrono::hours>(now - ticket->getEntryTime()).count();
        return (duration == 0 ? 1 : duration) * 10.0; // $10 per hour
    }
};

// Singleton ParkingLot
class ParkingLot {
private:
    static ParkingLot* instance;
    static mutex initMutex;
    
    vector<ParkingSpot*> spots;
    IPricingStrategy* pricingStrategy;
    mutable shared_mutex rwMutex; // Thread-safety (Read-Write Lock)

    ParkingLot() {
        pricingStrategy = new HourlyPricing();
        // Initialize dummy spots
        for(int i=0; i<10; ++i) spots.push_back(new ParkingSpot(i, SpotType::COMPACT));
    }

public:
    static ParkingLot* getInstance() {
        if (!instance) {
            lock_guard<mutex> lock(initMutex);
            if (!instance) instance = new ParkingLot();
        }
        return instance;
    }

    Ticket* enter(Vehicle* v) {
        unique_lock<shared_mutex> lock(rwMutex); // Exclusive write lock
        
        for (auto spot : spots) {
            if (spot->getIsFree()) { // simplified type check
                spot->assignVehicle(v);
                cout << "Vehicle " << v->getPlate() << " parked at spot " << spot->getId() << "\n";
                return new Ticket("TKT-" + to_string(spot->getId()), spot);
            }
        }
        cout << "Parking Full!\n";
        return nullptr;
    }

    double exit(Ticket* ticket) {
        unique_lock<shared_mutex> lock(rwMutex);
        
        ParkingSpot* spot = ticket->getSpot();
        double fee = pricingStrategy->calculateFee(ticket);
        spot->removeVehicle();
        cout << "Vehicle exited. Fee: $" << fee << "\n";
        return fee;
    }
    
    ~ParkingLot() {
        delete pricingStrategy;
        for(auto spot : spots) delete spot;
    }
};

ParkingLot* ParkingLot::instance = nullptr;
mutex ParkingLot::initMutex;

int main() {
    ParkingLot* lot = ParkingLot::getInstance();
    Vehicle car1("ABC-123", VehicleType::CAR);
    
    Ticket* t1 = lot->enter(&car1);
    if (t1) {
        lot->exit(t1);
        delete t1;
    }
    return 0;
}
```

## 6. Interview Tips & Follow-up Questions
*   **Q: How do you handle concurrency for `enter` and `exit` operations?**
    *   A: Use `std::shared_mutex` (Read-Write Lock) to allow multiple readers to check availability, but exclusive locks (`std::unique_lock`) when modifying spot state during entry/exit.
*   **Q: What if the ticket is lost?**
    *   A: We can query the database for active parking sessions by license plate, which requires a secondary index on `license_plate` in our active tickets mapping.
*   **Tip**: Call out that singletons are controversial but appropriate here for hardware controllers (like a gate).


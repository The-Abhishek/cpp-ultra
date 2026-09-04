# LLD (Low-Level Design) — Universal Framework

> Use this 6-step framework for every LLD interview.
> LLD = **OOP design + working code + design patterns + extensibility.**

---

## The 6-Step LLD Framework

```
┌──────────────────────────────────────────────────────────────────────┐
│                    LLD INTERVIEW (30-45 min)                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: Requirements & Use Cases ──────────────── [2-3 min]  🎯    │
│      ↓   (What does the system DO?)                                  │
│                                                                      │
│  Step 2: Core Entities & Relationships ─────────── [3 min]    📦    │
│      ↓   (Nouns → Classes, Verbs → Methods)                         │
│                                                                      │
│  Step 3: Design Patterns & Class Diagram ───────── [5 min]    📐    │
│      ↓   (Which pattern? WHY this pattern?)                          │
│                                                                      │
│  Step 4: Interface & Class Definitions ─────────── [5 min]    🔌    │
│      ↓   (C++ headers: abstract classes, interfaces)                 │
│                                                                      │
│  Step 5: Core Logic Implementation ─────────────── [15 min]   💻    │
│      ↓   (Working C++ code for critical flows)                       │
│                                                                      │
│  Step 6: Extensibility & Follow-ups ────────────── [3-5 min]  🔄    │
│         (How to add features? How to scale?)                         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step 1 — Requirements & Use Cases (2-3 min)

**WHY:** Prevents over-engineering. Clarify scope before writing any class.

### Gathering Template:
```
┌─────────────────────────────────────────────────────────┐
│  USE CASE IDENTIFICATION                                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Actors:  Who uses the system?                          │
│  ────────                                               │
│  • Primary:   Customer, Admin, Driver, ...              │
│  • System:    Timer, Scheduler, External API            │
│                                                         │
│  Core Use Cases (must implement):                       │
│  ──────────────────────────────                         │
│  UC1: [Actor] can [action] → [result]                   │
│  UC2: [Actor] can [action] → [result]                   │
│  UC3: System [triggers] when [condition]                │
│                                                         │
│  Constraints:                                           │
│  ───────────                                            │
│  • Concurrent access? Thread safety needed?             │
│  • Real-time updates? Observer needed?                  │
│  • Multiple strategies? Strategy pattern?               │
│  • State transitions? State machine?                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** Ask "Should I focus on the core domain logic, or also 
> include persistence/database layer?" This shows architectural awareness.

---

## Step 2 — Core Entities & Relationships (3 min)

**WHY:** Nouns in requirements become Classes. Verbs become Methods.

### Entity Discovery:
```
Requirements sentence:  "A user parks a car in a parking spot and gets a ticket."

Nouns (→ Classes):      User, Car, ParkingSpot, Ticket, ParkingLot
Verbs (→ Methods):      park(), getTicket(), calculateFee(), exit()
Adjectives (→ Enums):   VehicleType (Car, Truck, Motorcycle)
                        SpotSize (Small, Medium, Large)
```

### Relationship Types:
| Relationship | UML | C++ | Example |
|-------------|-----|-----|---------|
| **Has-a** (Composition) | ◆──→ | `unique_ptr<Engine>` in Car | Car has-a Engine (Engine dies with Car) |
| **Has-a** (Aggregation) | ◇──→ | `vector<Student*>` in School | School has-a Student (Student exists independently) |
| **Is-a** (Inheritance) | △──→ | `: public Vehicle` | Car is-a Vehicle |
| **Uses** (Dependency) | - - → | Method parameter | ParkingLot uses PaymentProcessor |
| **Implements** | △- -→ | `: public IStrategy` | FlatPricing implements IPricingStrategy |

---

## Step 3 — Design Patterns & Class Diagram (5 min)

**WHY:** Shows you know *when* and *why* to apply patterns, not just their names.

### Pattern Selection Guide:
```
┌──────────────────────────────────────────────────────────────────┐
│                PATTERN SELECTION DECISION TREE                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Multiple algorithms for same task?                              │
│  (pricing, sorting, routing)                                     │
│      → YES → STRATEGY PATTERN                                   │
│                                                                  │
│  Object has distinct states with transitions?                    │
│  (order: placed→paid→shipped, elevator: idle→moving→stopped)    │
│      → YES → STATE PATTERN                                      │
│                                                                  │
│  Need to notify multiple observers of changes?                   │
│  (display boards, notification systems)                          │
│      → YES → OBSERVER PATTERN                                   │
│                                                                  │
│  Need to create objects without specifying exact class?          │
│  (create Vehicle without knowing if Car or Truck)                │
│      → YES → FACTORY PATTERN                                    │
│                                                                  │
│  Only one instance should exist?                                 │
│  (database connection pool, configuration manager)               │
│      → YES → SINGLETON PATTERN                                  │
│                                                                  │
│  Tree/hierarchical structure? (file system, UI components)       │
│      → YES → COMPOSITE PATTERN                                  │
│                                                                  │
│  Need to add behavior dynamically without subclassing?           │
│  (adding toppings to pizza, adding features to stream)           │
│      → YES → DECORATOR PATTERN                                  │
│                                                                  │
│  Chain of handlers, pass request along until handled?            │
│  (logging levels, ATM cash dispensing, middleware)                │
│      → YES → CHAIN OF RESPONSIBILITY                            │
│                                                                  │
│  Undo/redo support? Encapsulate actions as objects?              │
│  (game moves, editor commands)                                   │
│      → YES → COMMAND PATTERN                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Class Diagram (ASCII):
```
Example: Parking Lot System

                    ┌─────────────────────────┐
                    │     «interface»          │
                    │   IPricingStrategy       │
                    ├─────────────────────────┤
                    │ + calculateFee(          │
                    │     duration): double    │
                    └────────────△────────────┘
                         ┌───────┴───────┐
                  ┌──────┴──────┐ ┌──────┴──────┐
                  │ HourlyPrice │ │  FlatPrice  │
                  └─────────────┘ └─────────────┘

  ┌──────────────┐         ┌──────────────────┐
  │  ParkingLot  │◆───────▶│   ParkingFloor   │
  ├──────────────┤  1..*   ├──────────────────┤
  │ - floors_    │         │ - spots_         │
  │ - pricing_   │         │ + findSpot()     │
  │ + enter()    │         └──────┬───────────┘
  │ + exit()     │                │ 1..*
  └──────────────┘         ┌──────┴───────────┐
                           │   ParkingSpot    │
                           ├──────────────────┤
                           │ - id_            │
                           │ - size_          │
                           │ - occupied_      │
                           │ + canFit()       │
                           │ + park()         │
                           │ + unpark()       │
                           └──────────────────┘
                                  ◇
                                  │ 0..1
                           ┌──────┴───────────┐
                           │    Vehicle        │
                           ├──────────────────┤
                           │ - plate_         │
                           │ - type_          │
                           └────────△─────────┘
                              ┌─────┼─────┐
                           ┌──┴──┐┌─┴──┐┌─┴───┐
                           │ Car ││Bike ││Truck│
                           └─────┘└────┘└─────┘
```

---

## Step 4 — Interface & Class Definitions (5 min)

**WHY:** Define contracts first (interfaces), then implement. Shows SOLID thinking.

### SOLID Principles Checklist:

| Principle | Meaning | How to Apply |
|-----------|---------|-------------|
| **S** — Single Responsibility | Each class does ONE thing | `PricingService` only calculates price, doesn't manage spots |
| **O** — Open/Closed | Open for extension, closed for modification | Add new `IPricingStrategy` without changing `ParkingLot` |
| **L** — Liskov Substitution | Subtypes must be substitutable | Any `Vehicle` subclass works where `Vehicle` is expected |
| **I** — Interface Segregation | Small, focused interfaces | `IPayable` separate from `ITrackable` |
| **D** — Dependency Inversion | Depend on abstractions, not concretions | `ParkingLot` depends on `IPricingStrategy`, not `HourlyPricing` |

### Template:
```cpp
// Step 4a: Define enums for type safety
enum class VehicleType { MOTORCYCLE, CAR, TRUCK };
enum class SpotSize    { SMALL, MEDIUM, LARGE };

// Step 4b: Define interfaces (abstract classes in C++)
class IPricingStrategy {
public:
    virtual ~IPricingStrategy() = default;
    virtual double calculateFee(
        std::chrono::minutes duration
    ) const = 0;  // pure virtual = interface method
};

// Step 4c: Define core entities
class Vehicle {
protected:
    std::string plate_;
    VehicleType type_;
public:
    Vehicle(std::string plate, VehicleType type) 
        : plate_(std::move(plate)), type_(type) {}
    virtual ~Vehicle() = default;
    
    VehicleType getType() const { return type_; }
    const std::string& getPlate() const { return plate_; }
};

class Car : public Vehicle {
public:
    Car(std::string plate) : Vehicle(std::move(plate), VehicleType::CAR) {}
};
```

---

## Step 5 — Core Logic Implementation (15 min)

**WHY:** This is where you prove you can actually code the design, not just draw boxes.

### What to implement:
1. **Happy path** — the main flow works correctly
2. **Edge cases** — lot full, invalid input, duplicate entry
3. **Thread safety** — `std::mutex` / `std::shared_mutex` where needed
4. **Error handling** — `std::optional`, exceptions, error codes

### Concurrency Patterns:
```cpp
// Reader-Writer Lock (multiple readers, exclusive writer)
class ThreadSafeResource {
    mutable std::shared_mutex mtx_;  // shared = readers, unique = writer
    Data data_;
public:
    Data read() const {
        std::shared_lock lock(mtx_);  // multiple readers OK
        return data_;
    }
    void write(Data d) {
        std::unique_lock lock(mtx_);  // exclusive access
        data_ = std::move(d);
    }
};
```

---

## Step 6 — Extensibility & Follow-ups (3-5 min)

**WHY:** Shows you designed for change, not just for today's requirements.

### Extensibility Checklist:
```
┌─────────────────────────────────────────────────────────┐
│  EXTENSIBILITY QUESTIONS TO ANSWER                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  "How to add a new vehicle type?"                       │
│  → Just create a new subclass of Vehicle                │
│  → No changes to ParkingLot (Open/Closed Principle)     │
│                                                         │
│  "How to change the pricing model?"                     │
│  → Implement new IPricingStrategy                       │
│  → Inject at runtime (Strategy Pattern)                 │
│                                                         │
│  "How to handle concurrent access?"                     │
│  → Already using std::shared_mutex                      │
│  → Lock at spot level, not lot level (granular locking) │
│                                                         │
│  "How to add notifications?"                            │
│  → Add Observer pattern for display boards              │
│  → ParkingLot::notify() on enter/exit                   │
│                                                         │
│  "How to persist data?"                                 │
│  → Add IRepository interface                            │
│  → Implement InMemoryRepo, SQLRepo, etc.                │
│  → Dependency Injection (Dependency Inversion Principle)│
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 Interviewer Scoring Signals (LLD)

```
┌─────────────────────────────────────────────────────────────┐
│               WHAT INTERVIEWERS ACTUALLY SCORE               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ STRONG SIGNALS                                         │
│  • Used interfaces/abstract classes (not concrete deps)     │
│  • Applied design patterns and explained WHY                │
│  • Code compiles and handles edge cases                     │
│  • Used enums instead of magic strings/ints                 │
│  • Considered thread safety without being asked             │
│  • Showed extensibility ("to add X, just implement Y")      │
│  • Named SOLID principles in context                        │
│                                                             │
│  ❌ WEAK SIGNALS                                            │
│  • One giant "God class" that does everything               │
│  • No interfaces, everything concrete                       │
│  • Named patterns but can't explain when/why                │
│  • No error handling or edge cases                          │
│  • public data members, no encapsulation                    │
│  • Used inheritance where composition fits better           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 Design Patterns Quick Reference (C++)

| Pattern | When to Use | C++ Idiom |
|---------|-------------|-----------|
| **Strategy** | Swap algorithms at runtime | `unique_ptr<IStrategy>` member |
| **Observer** | Notify multiple listeners | `vector<function<void()>>` callbacks |
| **State** | Object behavior changes with state | State classes with `unique_ptr<IState>` |
| **Factory** | Create objects without knowing type | `static unique_ptr<Base> create(Type)` |
| **Singleton** | One global instance | Meyer's Singleton (static local) |
| **Builder** | Complex object construction | Fluent API with method chaining |
| **Decorator** | Add behavior dynamically | Wrapping with same interface |
| **Command** | Encapsulate action as object | `class ICommand { virtual void execute(); }` |
| **Composite** | Tree structure | `class Node : has vector<unique_ptr<Node>>` |
| **Chain of Resp.** | Pass request through handlers | Linked list of handlers |

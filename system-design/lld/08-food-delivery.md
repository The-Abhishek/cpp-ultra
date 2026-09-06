# Design a Food Delivery System (Swiggy/Zomato)

## Step 1: Requirements Clarification
*   **Actors:** Customer, Restaurant, Delivery Partner
*   **Use Cases:** Browse menu, place order, track order, assign delivery, payment
*   **Characteristics:** High read [QPS (Queries Per Second)] for menu browsing, transactional for ordering. 

> **Interview Tip:** Highlight the geographical proximity aspect for delivery assignment. A delivery partner must be physically close to the restaurant to be assigned.

## Step 2: Object Identification
*   **Entities:** `Order`, `Restaurant`, `MenuItem`, `DeliveryPartner`, `Customer`
*   **Enums:** `OrderStatus` (PLACED, ACCEPTED, PREPARING, PICKED_UP, DELIVERED)

## Step 3: Class Diagram (ASCII)
```text
+---------------+       1..* +---------------+
|   Restaurant  |----------->|   MenuItem    |
+---------------+            +---------------+
| -id           |            | -name         |
| -location     |            | -price        |
+---------------+            +---------------+
       | 1..*
       v
+---------------+            +---------------+
|     Order     |<-----------|   Customer    |
+---------------+            +---------------+
| -orderId      |            | -id           |
| -status       |            | -location     |
| -totalAmount  |            +---------------+
+---------------+
       | 1..1
       v
+---------------+
|DeliveryPartner|
+---------------+
| -id           |
| -location     |
| -isAvailable  |
+---------------+
```

## Step 4: Core APIs / Interfaces
*   `placeOrder(Customer customer, Cart cart)`
*   `assignDeliveryPartner(Order order)`
*   `updateOrderStatus(Order order, OrderStatus status)`

## Step 5: Design Patterns
*   **State Pattern:** Order transitions (PLACED → ACCEPTED → PREPARING → PICKED_UP → DELIVERED). *Why?* State logic is complex and order-dependent.
*   **Strategy Pattern:** Delivery assignment (Nearest first vs. Highest rated first). *Why?* Pluggable algorithms for dispatching.
*   **Observer Pattern:** Order status updates to Customer. *Why?* Asynchronous updates to the frontend without polling.

## Step 6: Code Implementation (C++)

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <mutex>

enum class OrderStatus { PLACED, ACCEPTED, PREPARING, PICKED_UP, DELIVERED };

// Forward declarations
class Customer;
class DeliveryPartner;

class Observer {
public:
    virtual void update(OrderStatus status) = 0;
    virtual ~Observer() = default;
};

class Order {
private:
    std::string orderId;
    OrderStatus status;
    std::vector<std::shared_ptr<Observer>> observers;
    std::mutex orderMutex;

public:
    Order(std::string id) : orderId(id), status(OrderStatus::PLACED) {}

    void attach(std::shared_ptr<Observer> obs) {
        observers.push_back(obs);
    }

    void notifyObservers() {
        for (auto& obs : observers) {
            obs->update(status);
        }
    }

    void setStatus(OrderStatus newStatus) {
        std::lock_guard<std::mutex> lock(orderMutex);
        status = newStatus;
        notifyObservers();
    }

    std::string getId() const { return orderId; }
};

class Customer : public Observer {
private:
    std::string name;
public:
    Customer(std::string n) : name(n) {}
    void update(OrderStatus status) override {
        std::cout << "[Customer " << name << "] Order status updated to: " << static_cast<int>(status) << "\n";
    }
};

class DeliveryPartner {
public:
    std::string id;
    bool isAvailable;
    DeliveryPartner(std::string id) : id(id), isAvailable(true) {}
};

class DeliveryStrategy {
public:
    virtual std::shared_ptr<DeliveryPartner> assignPartner(std::vector<std::shared_ptr<DeliveryPartner>>& partners) = 0;
    virtual ~DeliveryStrategy() = default;
};

class NearestPartnerStrategy : public DeliveryStrategy {
public:
    std::shared_ptr<DeliveryPartner> assignPartner(std::vector<std::shared_ptr<DeliveryPartner>>& partners) override {
        // [Logic] Greedy assignment: Pick first available (Simplified for example)
        for (auto& partner : partners) {
            if (partner->isAvailable) {
                partner->isAvailable = false;
                return partner;
            }
        }
        return nullptr;
    }
};

int main() {
    auto customer = std::make_shared<Customer>("Alice");
    Order order("ORD-123");
    order.attach(customer);

    std::cout << "Placing order...\n";
    order.setStatus(OrderStatus::ACCEPTED);

    // Setup delivery partners
    std::vector<std::shared_ptr<DeliveryPartner>> partners;
    partners.push_back(std::make_shared<DeliveryPartner>("DP1"));
    
    // Assign partner
    NearestPartnerStrategy strategy;
    auto partner = strategy.assignPartner(partners);
    
    if (partner) {
        std::cout << "Assigned Delivery Partner: " << partner->id << "\n";
        order.setStatus(OrderStatus::PREPARING);
        order.setStatus(OrderStatus::PICKED_UP);
        order.setStatus(OrderStatus::DELIVERED);
    }

    return 0;
}
```

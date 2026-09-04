# Design a Vending Machine

## Step 1: Requirements Clarification
*   **Actors:** Customer, Operator
*   **Use Cases:** Select product, insert money, dispense product, return change, refill
*   **Core Logic:** Handling state transitions correctly (e.g., cannot dispense if no money inserted).

> **Interview Tip:** This is THE classic State Pattern question. Do not attempt to solve it with a giant `switch-case` statement; interviewers specifically look for the State Pattern.

## Step 2: Object Identification
*   **Entities:** `VendingMachine`, `Product`, `Inventory`, `Coin`
*   **States:** `IDLE`, `HAS_MONEY`, `DISPENSING`

## Step 3: State Transition Diagram (ASCII)
```text
          insertMoney()            selectProduct()
[ IDLE ] ---------------> [ HAS_MONEY ] ---------------> [ DISPENSING ]
   ^                          |    |                           |
   |     cancel/refund        |    |                           | dispense()
   +--------------------------+    +---------------------------+
                                            change returned
```

## Step 4: Core APIs / Interfaces
*   `insertMoney(int amount)`
*   `selectProduct(String productId)`
*   `dispense()`
*   `refund()`

## Step 5: Design Patterns
*   **State Machine Pattern:** *Why?* Eliminates conditional logic (if-else/switch) for state behaviors. Each state implements an interface and handles transitions to the next state.

## Step 6: Code Implementation (C++)

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <unordered_map>
#include <stdexcept>

// Forward declarations
class VendingMachine;

// --- State Interface ---
class State {
public:
    virtual void insertMoney(VendingMachine* machine, int amount) = 0;
    virtual void selectProduct(VendingMachine* machine, std::string productId) = 0;
    virtual void dispense(VendingMachine* machine) = 0;
    virtual void refund(VendingMachine* machine) = 0;
    virtual ~State() = default;
};

// --- Vending Machine Context ---
class VendingMachine {
private:
    std::shared_ptr<State> currentState;
    int currentBalance = 0;

public:
    std::shared_ptr<State> idleState;
    std::shared_ptr<State> hasMoneyState;
    std::shared_ptr<State> dispensingState;

    VendingMachine(); // Implementation deferred

    void setState(std::shared_ptr<State> state) { currentState = state; }
    void addBalance(int amount) { currentBalance += amount; }
    int getBalance() const { return currentBalance; }
    void clearBalance() { currentBalance = 0; }

    void insertMoney(int amount) { currentState->insertMoney(this, amount); }
    void selectProduct(std::string productId) { currentState->selectProduct(this, productId); }
    void dispense() { currentState->dispense(this); }
    void refund() { currentState->refund(this); }
};

// --- Concrete States ---
class IdleState : public State {
public:
    void insertMoney(VendingMachine* machine, int amount) override;
    void selectProduct(VendingMachine* machine, std::string productId) override {
        std::cout << "Please insert money first.\n";
    }
    void dispense(VendingMachine* machine) override {
        std::cout << "Cannot dispense. No product selected.\n";
    }
    void refund(VendingMachine* machine) override {
        std::cout << "No money to refund.\n";
    }
};

class HasMoneyState : public State {
public:
    void insertMoney(VendingMachine* machine, int amount) override {
        machine->addBalance(amount);
        std::cout << "Inserted: " << amount << ". Total: " << machine->getBalance() << "\n";
    }
    void selectProduct(VendingMachine* machine, std::string productId) override;
    void dispense(VendingMachine* machine) override {
        std::cout << "Please select a product first.\n";
    }
    void refund(VendingMachine* machine) override {
        std::cout << "Refunding " << machine->getBalance() << "\n";
        machine->clearBalance();
        machine->setState(machine->idleState);
    }
};

class DispensingState : public State {
public:
    void insertMoney(VendingMachine* machine, int amount) override {
        std::cout << "Please wait, dispensing product.\n";
    }
    void selectProduct(VendingMachine* machine, std::string productId) override {
        std::cout << "Already dispensing.\n";
    }
    void dispense(VendingMachine* machine) override {
        std::cout << "Dispensing product...\n";
        int balance = machine->getBalance();
        // Assuming product costs 10 for simplicity
        if (balance > 10) {
            std::cout << "Returning change: " << (balance - 10) << "\n";
        }
        machine->clearBalance();
        machine->setState(machine->idleState);
    }
    void refund(VendingMachine* machine) override {
        std::cout << "Cannot refund, already dispensing.\n";
    }
};

// --- Resolving Circular Dependencies ---
VendingMachine::VendingMachine() {
    idleState = std::make_shared<IdleState>();
    hasMoneyState = std::make_shared<HasMoneyState>();
    dispensingState = std::make_shared<DispensingState>();
    currentState = idleState;
}

void IdleState::insertMoney(VendingMachine* machine, int amount) {
    machine->addBalance(amount);
    std::cout << "Inserted: " << amount << "\n";
    machine->setState(machine->hasMoneyState);
}

void HasMoneyState::selectProduct(VendingMachine* machine, std::string productId) {
    std::cout << "Product " << productId << " selected.\n";
    // In a real system, check inventory and price here
    machine->setState(machine->dispensingState);
    machine->dispense(); // Auto transition
}

int main() {
    VendingMachine vm;

    std::cout << "--- Scenario 1: Buy Product ---\n";
    vm.insertMoney(5);
    vm.insertMoney(10);
    vm.selectProduct("COKE");

    std::cout << "\n--- Scenario 2: Refund ---\n";
    vm.insertMoney(10);
    vm.refund();

    return 0;
}
```

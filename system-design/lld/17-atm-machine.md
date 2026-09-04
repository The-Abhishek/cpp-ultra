# Design ATM Machine

## 1. Requirements Clarification
**Functional Requirements:**
- Authenticate user via PIN.
- Check balance, withdraw cash, deposit cash, transfer funds.
- Dispense cash using available denominations.

**Non-Functional Requirements:**
- ACID [Atomicity, Consistency, Isolation, Durability (database transaction properties)] guarantees for transactions.
- Security and audit logging.
- Extensibility for new bill denominations.

> **Interview Tip**: State transitions are critical here. The interviewer wants to see how you prevent cash withdrawal when the ATM is in the `IDLE` state. The State Pattern is mandatory.

## 2. Actors & Use Cases
- **Customer**: Inserts card, enters PIN, selects transaction, collects cash.
- **Bank Server / Core Banking**: Validates PIN, approves balances, commits transactions.
- **Operator**: Refills cash.

## 3. Core Entities & Patterns
- **ATM**: Context class holding the state.
- **ATMState**: Interface for `Idle`, `CardInserted`, `Authenticated`, `Transaction`.
- **CashDispenser**: Chain of Responsibility to handle bills.
- **Design Patterns:**
  - **State Pattern**: *Why?* ATM behavior completely changes based on its current state (e.g., you cannot withdraw if a card isn't inserted). It prevents huge `switch-case` blocks.
  - **Chain of Responsibility**: *Why?* Dispensing cash requires splitting the amount into multiple denominations ($100, $50, $20, $10). Each handler checks if it can dispense its bill size, then passes the remainder to the next handler.

## 4. Class Diagram & Architecture

```text
+-------------------+      +-------------------------+
|       ATM         |----->| <<interface>> ATMState  |
|-------------------|      |-------------------------|
| - state: ATMState |      | + insertCard()          |
| - dispenser       |      | + authenticate(pin)     |
| + setState()      |      | + withdraw(amount)      |
+-------------------+      +-------------------------+
                                    ^
                                    |
          +-------------------------+-------------------------+
          |                         |                         |
+-------------------+     +-------------------+     +-------------------+
|     IdleState     |     |   HasCardState    |     | AuthenticatedState|
+-------------------+     +-------------------+     +-------------------+

+-------------------+      +-------------------+
|  CashDispenser    |----->|  CashDispenser    | (Next in chain)
|-------------------|      +-------------------+
| - billValue       |
| + dispense(amt)   |
+-------------------+
```

## 5. Deep Dive: Cash Dispensing Algorithm
**Greedy Approach:**
Withdrawal of `$270`.
1. **$100 Handler**: Takes `270 / 100 = 2` bills. Remaining amount: `$70`. Passes to $50 Handler.
2. **$50 Handler**: Takes `70 / 50 = 1` bill. Remaining amount: `$20`. Passes to $20 Handler.
3. **$20 Handler**: Takes `20 / 20 = 1` bill. Remaining amount: `$0`. Passes to $10 Handler.
4. **$10 Handler**: Sees `$0`, does nothing. Success.

| Scenario | Handling Mechanism |
| :--- | :--- |
| **Exact amount not possible** (e.g., $25 with only $10s) | The chain reaches the end and the remainder > 0. Throw exception / revert transaction. |
| **ATM runs out of a denomination** | Handler checks internal count. Dispenses what it can, passes the rest down the chain. |

## 6. Full C++ Implementation

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <stdexcept>

// --- CHAIN OF RESPONSIBILITY for Cash Dispenser ---
class CashDispenser {
protected:
    std::shared_ptr<CashDispenser> nextHandler;
    int denomination;
    int availableNotes;

public:
    CashDispenser(int val, int count) : denomination(val), availableNotes(count) {}
    
    void setNext(std::shared_ptr<CashDispenser> next) {
        nextHandler = next;
    }

    virtual void dispense(int& amount) {
        if (amount >= denomination && availableNotes > 0) {
            int notesRequired = amount / denomination;
            int notesToDispense = std::min(notesRequired, availableNotes);
            
            amount -= notesToDispense * denomination;
            availableNotes -= notesToDispense;
            
            std::cout << "Dispensing " << notesToDispense << " notes of $" << denomination << "\n";
        }
        
        if (amount > 0 && nextHandler) {
            nextHandler->dispense(amount);
        } else if (amount > 0 && !nextHandler) {
            throw std::runtime_error("Cannot dispense exact amount with available denominations.");
        }
    }
};

// --- STATE PATTERN for ATM ---
class ATM;

class ATMState {
public:
    virtual ~ATMState() = default;
    virtual void insertCard(ATM* atm) = 0;
    virtual void authenticate(ATM* atm, int pin) = 0;
    virtual void withdraw(ATM* atm, int amount) = 0;
};

class ATM {
private:
    std::shared_ptr<ATMState> currentState;
    std::shared_ptr<CashDispenser> dispenserChain;

public:
    ATM(std::shared_ptr<ATMState> initialState, std::shared_ptr<CashDispenser> chain) 
        : currentState(initialState), dispenserChain(chain) {}

    void setState(std::shared_ptr<ATMState> state) {
        currentState = state;
    }

    std::shared_ptr<CashDispenser> getDispenser() {
        return dispenserChain;
    }

    void insertCard() { currentState->insertCard(this); }
    void authenticate(int pin) { currentState->authenticate(this, pin); }
    void withdraw(int amount) { currentState->withdraw(this, amount); }
};

// Concrete States
class AuthenticatedState : public ATMState {
public:
    void insertCard(ATM* atm) override { std::cout << "Card already inserted.\n"; }
    void authenticate(ATM* atm, int pin) override { std::cout << "Already authenticated.\n"; }
    void withdraw(ATM* atm, int amount) override {
        std::cout << "Processing withdrawal of $" << amount << "...\n";
        try {
            atm->getDispenser()->dispense(amount);
            std::cout << "Withdrawal successful. Transitioning to Idle.\n";
            // atm->setState(std::make_shared<IdleState>()); // Assume IdleState exists
        } catch (const std::exception& e) {
            std::cout << "Error: " << e.what() << "\n";
        }
    }
};

class HasCardState : public ATMState {
public:
    void insertCard(ATM* atm) override { std::cout << "Card already inserted.\n"; }
    void authenticate(ATM* atm, int pin) override {
        if (pin == 1234) {
            std::cout << "Authentication successful.\n";
            atm->setState(std::make_shared<AuthenticatedState>());
        } else {
            std::cout << "Invalid PIN.\n";
        }
    }
    void withdraw(ATM* atm, int amount) override { std::cout << "Authenticate first.\n"; }
};

class IdleState : public ATMState {
public:
    void insertCard(ATM* atm) override {
        std::cout << "Card inserted.\n";
        atm->setState(std::make_shared<HasCardState>());
    }
    void authenticate(ATM* atm, int pin) override { std::cout << "Insert card first.\n"; }
    void withdraw(ATM* atm, int amount) override { std::cout << "Insert card first.\n"; }
};

int main() {
    // Setup Chain of Responsibility
    auto disp100 = std::make_shared<CashDispenser>(100, 2);
    auto disp50 = std::make_shared<CashDispenser>(50, 5);
    auto disp20 = std::make_shared<CashDispenser>(20, 10);
    
    disp100->setNext(disp50);
    disp50->setNext(disp20);

    ATM atm(std::make_shared<IdleState>(), disp100);

    atm.insertCard();
    atm.authenticate(1234);
    atm.withdraw(270);

    return 0;
}
```

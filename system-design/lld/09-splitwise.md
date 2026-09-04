# Design Splitwise (Expense Sharing)

## Step 1: Requirements Clarification
*   **Actors:** Users
*   **Use Cases:** Add expense, split (equal/exact/percentage), view balances, settle up, simplify debts
*   **Complexity:** The debt simplification algorithm is the core algorithmic challenge here.

> **Interview Tip:** Start by clarifying the split types. Make sure you separate the `Expense` entity from the `Split` strategy. Mention that [WAL (Write-Ahead Log)] could be used at the DB level to ensure no financial data is lost during server crashes.

## Step 2: Object Identification
*   **Entities:** `User`, `Group`, `Expense`, `Split`
*   **Enums:** `SplitType` (EQUAL, EXACT, PERCENT)

## Step 3: Class Diagram (ASCII)
```text
+---------------+       1..* +---------------+       1..* +---------------+
|     User      |----------->|   Expense     |----------->|     Split     |
+---------------+            +---------------+            +---------------+
| -id           |            | -amount       |            | -user         |
| -name         |            | -paidBy       |            | -amount       |
+---------------+            | -splits       |            +---------------+
                             +---------------+                 ^
                                                               |
                                                  +------------+-------------+
                                                  |            |             |
                                            +---------+  +---------+  +----------+
                                            |  Equal  |  |  Exact  |  | Percent  |
                                            +---------+  +---------+  +----------+
```

## Step 4: Core APIs / Interfaces
*   `addExpense(Expense expense)`
*   `showBalances(User user)`
*   `simplifyDebts(Group group)`

## Step 5: Design Patterns
*   **Strategy Pattern:** Different splitting logic (Equal, Exact, Percentage). *Why?* Easy to add new split types (e.g., Shares) without modifying core Expense class.
*   **Observer Pattern:** Balance change notifications.

## Step 6: Deep Dive: Debt Simplification Algorithm
*   **Concept:** Model debts as a directed graph. 
*   **Simplify:** Find net balance for each person. Match creditors (positive balance) with debtors (negative balance).
*   **Greedy approach:** Sort by balance, match largest creditor with largest debtor recursively.

## Step 7: Code Implementation (C++)

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <algorithm>
#include <memory>
#include <cmath>

class User {
public:
    std::string id;
    std::string name;
    User(std::string id, std::string name) : id(id), name(name) {}
};

// --- Debt Simplification ---
struct Transaction {
    std::string from;
    std::string to;
    double amount;
};

class DebtSimplifier {
public:
    static std::vector<Transaction> simplifyDebts(const std::unordered_map<std::string, double>& balances) {
        std::vector<std::pair<std::string, double>> positive, negative;

        for (const auto& [user, amount] : balances) {
            if (amount > 0.01) positive.push_back({user, amount});
            else if (amount < -0.01) negative.push_back({user, -amount});
        }

        // Sort by amount descending
        auto comp = [](const auto& a, const auto& b) { return a.second > b.second; };
        std::sort(positive.begin(), positive.end(), comp);
        std::sort(negative.begin(), negative.end(), comp);

        std::vector<Transaction> result;
        int i = 0, j = 0;

        while (i < positive.size() && j < negative.size()) {
            double settleAmount = std::min(positive[i].second, negative[j].second);
            
            result.push_back({negative[j].first, positive[i].first, settleAmount});
            
            positive[i].second -= settleAmount;
            negative[j].second -= settleAmount;

            if (positive[i].second < 0.01) i++;
            if (negative[j].second < 0.01) j++;
        }
        return result;
    }
};

// --- Expense Management ---
class Split {
public:
    std::shared_ptr<User> user;
    double amount;
    Split(std::shared_ptr<User> u, double amt) : user(u), amount(amt) {}
    virtual ~Split() = default;
};

class ExpenseManager {
private:
    std::unordered_map<std::string, double> balances; // Net balance per user

public:
    void addExpense(std::shared_ptr<User> paidBy, double totalAmount, const std::vector<std::shared_ptr<Split>>& splits) {
        // [SOLID] Single Responsibility: Manager handles balances, Splits handle their own amounts
        balances[paidBy->id] += totalAmount;
        for (const auto& split : splits) {
            balances[split->user->id] -= split->amount;
        }
    }

    void showBalances() {
        for (const auto& [user, amount] : balances) {
            std::cout << user << " net balance: " << amount << "\n";
        }
    }

    void simplify() {
        std::cout << "\n--- Simplified Debts ---\n";
        auto transactions = DebtSimplifier::simplifyDebts(balances);
        for (const auto& tx : transactions) {
            std::cout << tx.from << " pays " << tx.to << " $" << tx.amount << "\n";
        }
    }
};

int main() {
    auto u1 = std::make_shared<User>("U1", "Alice");
    auto u2 = std::make_shared<User>("U2", "Bob");
    auto u3 = std::make_shared<User>("U3", "Charlie");

    ExpenseManager manager;

    // Alice paid $300, split equally among Alice, Bob, Charlie ($100 each)
    std::vector<std::shared_ptr<Split>> splits = {
        std::make_shared<Split>(u1, 100),
        std::make_shared<Split>(u2, 100),
        std::make_shared<Split>(u3, 100)
    };
    
    manager.addExpense(u1, 300, splits);
    manager.showBalances();
    manager.simplify();

    return 0;
}
```

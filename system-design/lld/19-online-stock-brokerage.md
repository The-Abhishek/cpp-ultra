# Design Online Stock Brokerage

## 1. Requirements Clarification
**Functional Requirements:**
- Place orders: Market, Limit, Stop-Loss.
- Order execution and matching (Price-Time Priority).
- View portfolios and real-time prices.

**Non-Functional Requirements:**
- Ultra-low latency for order matching.
- ACID properties for trades and ledger updates.
- Concurrency for thousands of orders per second (QPS [Queries Per Second (measure of traffic)]).

> **Interview Tip**: Focus heavily on the **Order Matching Engine**. Using simple lists for orders will fail an interview. Use Priority Queues or Hash Maps combined with Queues for O(1) matching and O(log N) insertion.

## 2. Actors & Use Cases
- **Trader**: Submits Buy/Sell orders, cancels orders.
- **Matching Engine**: Core backend service that crosses Bids and Asks.

## 3. Core Entities & Patterns
- **Order**: Contains type (Buy/Sell), limit price, quantity, timestamp.
- **OrderBook**: Manages Bids (buys) and Asks (sells) for a specific stock ticker.
- **Design Patterns:**
  - **Command Pattern**: *Why?* Placing an order acts as a command that can be queued, logged, or reversed (cancelled).
  - **Strategy Pattern**: *Why?* Order execution behaviors differ heavily between Market Orders (match any best price) and Limit Orders (match exact or better price).

## 4. Class Diagram & Architecture

```text
+-------------------+      +-------------------+
|   MatchingEngine  |<>--->|    OrderBook      |
|-------------------|      |-------------------|
| + placeOrder()    |      | - bids: std::map  |
+-------------------+      | - asks: std::map  |
                           | + match()         |
                           +-------------------+
                                     |
                                     v
                           +-------------------+
                           |      Order        |
                           |-------------------|
                           | - price, qty      |
                           | - side (BUY/SELL) |
                           +-------------------+
```

## 5. Deep Dive: Order Matching (Price-Time Priority)
The heart of an exchange is the Limit Order Book.
- **Bids (Buy Orders)**: We want to match the *highest* bid first. If prices are equal, match the *oldest* order (FIFO).
  - Data Structure: `std::map<double, std::queue<Order>, std::greater<double>>`
- **Asks (Sell Orders)**: We want to match the *lowest* ask first. If prices are equal, FIFO.
  - Data Structure: `std::map<double, std::queue<Order>>`

**Matching Logic:**
`if (best_bid.price >= best_ask.price)` -> Match! Deduct quantities. If a queue empties, remove it from the map.

## 6. Full C++ Implementation

```cpp
#include <iostream>
#include <string>
#include <queue>
#include <map>
#include <vector>

enum class Side { BUY, SELL };

struct Order {
    std::string orderId;
    Side side;
    double price;
    int quantity;
    long timestamp;

    Order(std::string id, Side s, double p, int q, long t)
        : orderId(id), side(s), price(p), quantity(q), timestamp(t) {}
};

class OrderBook {
private:
    std::string ticker;
    // Bids: Highest price first -> std::greater
    std::map<double, std::queue<Order>, std::greater<double>> bids;
    // Asks: Lowest price first -> default std::less
    std::map<double, std::queue<Order>> asks;

public:
    OrderBook(std::string t) : ticker(t) {}

    void addOrder(const Order& order) {
        if (order.side == Side::BUY) {
            bids[order.price].push(order);
        } else {
            asks[order.price].push(order);
        }
        match();
    }

private:
    void match() {
        while (!bids.empty() && !asks.empty()) {
            auto bestBidIt = bids.begin();
            auto bestAskIt = asks.begin();

            if (bestBidIt->first >= bestAskIt->first) {
                // Match possible!
                Order& bidOrder = bestBidIt->second.front();
                Order& askOrder = bestAskIt->second.front();

                int tradeQty = std::min(bidOrder.quantity, askOrder.quantity);
                double matchPrice = bestAskIt->first; // Trade executes at resting order's price

                std::cout << "TRADE EXECUTED: " << tradeQty << " shares of " << ticker 
                          << " @ $" << matchPrice << "\n";

                bidOrder.quantity -= tradeQty;
                askOrder.quantity -= tradeQty;

                if (bidOrder.quantity == 0) bestBidIt->second.pop();
                if (askOrder.quantity == 0) bestAskIt->second.pop();

                // Cleanup empty queues
                if (bestBidIt->second.empty()) bids.erase(bestBidIt);
                if (bestAskIt->second.empty()) asks.erase(bestAskIt);
            } else {
                break; // No match possible
            }
        }
    }
};

int main() {
    OrderBook aaplBook("AAPL");

    // Add some asks (sellers)
    aaplBook.addOrder(Order("S1", Side::SELL, 150.00, 100, 1));
    aaplBook.addOrder(Order("S2", Side::SELL, 150.50, 200, 2));

    // Add a bid that matches (buyer)
    std::cout << "Incoming Buy Order @ $150.00 for 50 shares\n";
    aaplBook.addOrder(Order("B1", Side::BUY, 150.00, 50, 3)); 
    // Trades 50 @ 150.00, S1 remains with 50

    std::cout << "Incoming Buy Order @ $151.00 for 100 shares\n";
    aaplBook.addOrder(Order("B2", Side::BUY, 151.00, 100, 4));
    // Trades 50 @ 150.00 (finishes S1), Trades 50 @ 150.50 (eats into S2)

    return 0;
}
```

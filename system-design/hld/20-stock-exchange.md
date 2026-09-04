# System Design: Stock Exchange / Trading Platform

## 1. Requirements

### Functional Requirements (FR)
- Place limit orders and market orders.
- Order matching based on Price-Time priority (FIFO).
- Maintain Order Book (bids and asks).
- Trade execution reporting.

### Non-Functional Requirements (NFR)
- Ultra-low latency: <1ms (nanoseconds for HFT systems).
- High throughput: Millions of orders/day.
- High availability.
- Strict fairness (deterministic ordering).

## 2. Terminology & Concepts
- **Bid**: Buy order.
- **Ask**: Sell order.
- **Order Book**: Record of all active buy/sell orders.
- **Limit Order**: Buy/sell at a specific price or better.
- **Market Order**: Buy/sell immediately at the best available price.
- **Price-Time Priority**: Orders are sorted first by price (best price wins), then by time (earlier order wins).
- **LMAX Disruptor**: A high-performance lock-free ring buffer used for passing messages between threads.

## 3. Back-of-the-Envelope Estimation
- Orders per day: 10M.
- Peak QPS: 10,000 orders/sec.
- Latency budget: < 100 microseconds for matching.
- Memory: Order book fits easily in RAM (few GBs per symbol).

## 4. Architecture Diagram

```mermaid
flowchart TD
    Client --> API[FIX / REST Gateway]
    API --> Sequencer[Sequencer / Timestamping]
    Sequencer --> ME[Matching Engine (Single Thread)]
    ME --> Book[(In-Memory Order Book)]
    ME --> Trades[Trade Reporter / Persister]
    Trades --> DB[(Trade DB)]
    Trades --> MarketData[Market Data Feed]
    MarketData --> Client
```

### Why Single-Threaded?
The core matching engine is often pinned to a single CPU core. No locks, no context switching, deterministic execution. Horizontal scaling is done by partitioning by stock symbol (e.g., AAPL on Server 1, MSFT on Server 2).

## 5. Core Components & Deep Dive

### Order Book Data Structure
Need fast inserts, fast deletes (cancels), and fast matching (getting min/max).
- **Bids**: Sorted descending by price.
- **Asks**: Sorted ascending by price.
- Implementation: Usually a `std::map` (Red-Black tree) grouped by price level, pointing to a linked list (FIFO queue) of orders at that price. Or flat arrays for extreme low latency.

## 6. Core Logic / Pseudocode

### Order Book & Matching (C++)
```cpp
#include <iostream>
#include <map>
#include <list>
#include <string>

struct Order {
    uint64_t id;
    bool is_buy;
    double price;
    int quantity;
};

class OrderBook {
private:
    // Price -> List of orders (FIFO)
    // std::greater for Bids (highest price first)
    std::map<double, std::list<Order>, std::greater<double>> bids;
    
    // std::less for Asks (lowest price first)
    std::map<double, std::list<Order>> asks;

public:
    void process_limit_order(Order incoming) {
        if (incoming.is_buy) {
            match(incoming, asks, bids);
        } else {
            match(incoming, bids, asks);
        }
    }

private:
    template<typename T_Opposite, typename T_Same>
    void match(Order& incoming, T_Opposite& opposite_side, T_Same& same_side) {
        auto it = opposite_side.begin();
        
        while (it != opposite_side.end() && incoming.quantity > 0) {
            double best_price = it->first;
            
            // Check if prices cross
            if ((incoming.is_buy && incoming.price < best_price) || 
                (!incoming.is_buy && incoming.price > best_price)) {
                break;
            }
            
            auto& order_queue = it->second;
            while (!order_queue.empty() && incoming.quantity > 0) {
                Order& resting_order = order_queue.front();
                int trade_qty = std::min(incoming.quantity, resting_order.quantity);
                
                std::cout << "Trade Executed: " << trade_qty << " @ " << best_price << "\n";
                
                incoming.quantity -= trade_qty;
                resting_order.quantity -= trade_qty;
                
                if (resting_order.quantity == 0) {
                    order_queue.pop_front();
                }
            }
            
            if (order_queue.empty()) {
                it = opposite_side.erase(it);
            } else {
                break; // Incoming order filled
            }
        }
        
        // Add remaining to the order book
        if (incoming.quantity > 0) {
            same_side[incoming.price].push_back(incoming);
        }
    }
};
```

## 7. Failure Scenarios & Scaling
- **Engine Crash**: State is replicated using event sourcing. All incoming orders hit an append-only journal (sequencer) before the matching engine. On crash, replay the journal to reconstruct the in-memory order book.
- **Scaling**: Shard the matching engines by ticker symbol. AAPL trades are entirely independent of TSLA trades.

## 8. Interview Tips
- **Crucial**: Mention that matching engines are single-threaded to avoid lock contention.
- Mention bypassing the OS kernel (kernel bypass, e.g., DPDK) and using UDP/multicast for market data feeds.
- Emphasize Event Sourcing for crash recovery instead of traditional database transactions.

# Design Rate Limiter (Code Level)

## 1. Requirements & Use Cases
- **Actors:** System, User
- **Use Cases:** Limit API calls per user, configurable limits, support multiple algorithms.

## 2. Terminology & Entities
- **QPS [Queries Per Second]:** The rate at which requests arrive.
- **Throttling [Intentionally limiting rate of requests]:** What a rate limiter enforces.
- **Entities:** IRateLimiter, TokenBucketLimiter, SlidingWindowLimiter, FixedWindowLimiter.

## 3. Design Patterns & Principles
- **Strategy Pattern:** Algorithm selection (Token Bucket, Sliding Window). Why? Encapsulates the algorithm details from the user.
- **Factory Pattern:** Create the appropriate limiter based on config.
- **SOLID Principles:** Interface Segregation - all limiters implement a simple `allowRequest()` interface.

## 4. ASCII Class Diagram
```text
+-------------------+
|   IRateLimiter    |
+-------------------+
| + allowRequest()  |
+-------------------+
          ^
          |-----------------------------------+
          |                                   |
+--------------------+              +-------------------+
| TokenBucketLimiter |              |FixedWindowLimiter |
+--------------------+              +-------------------+
```

## 5. Deep Dive: Algorithms
- **Token Bucket:** Tokens refill at rate `r`, burst capacity `b`. Best for allowing bursts.
- **Fixed Window Counter:** Count requests in current time window. Flaw: spikes at window edges.
- **Sliding Window Counter:** Weighted combination of current and previous window. Smooths out edges.

## 6. Full Working C++ Implementation
```cpp
#include <iostream>
#include <chrono>
#include <mutex>
#include <thread>
#include <algorithm>

class IRateLimiter {
public:
    virtual bool allowRequest() = 0;
    virtual ~IRateLimiter() = default;
};

// 1. Token Bucket Limiter
class TokenBucketLimiter : public IRateLimiter {
    long long capacity;
    double tokens;
    double refillRatePerSec;
    std::chrono::time_point<std::chrono::steady_clock> lastRefillTime;
    std::mutex mtx;

public:
    TokenBucketLimiter(long long cap, double rate) 
        : capacity(cap), tokens(cap), refillRatePerSec(rate), 
          lastRefillTime(std::chrono::steady_clock::now()) {}

    bool allowRequest() override {
        std::lock_guard<std::mutex> lock(mtx);
        auto now = std::chrono::steady_clock::now();
        std::chrono::duration<double> diff = now - lastRefillTime;
        
        tokens += diff.count() * refillRatePerSec;
        if (tokens > capacity) tokens = capacity;
        lastRefillTime = now;

        if (tokens >= 1.0) {
            tokens -= 1.0;
            return true;
        }
        return false;
    }
};

// Simplified Testing
void testLimiter(IRateLimiter* limiter, const std::string& name) {
    std::cout << "Testing " << name << ":\n";
    for (int i = 0; i < 5; ++i) {
        if (limiter->allowRequest()) {
            std::cout << "Request " << i+1 << " allowed.\n";
        } else {
            std::cout << "Request " << i+1 << " blocked.\n";
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
}

int main() {
    // Capacity 3, Refill 2 tokens/sec
    TokenBucketLimiter tb(3, 2.0); 
    testLimiter(&tb, "Token Bucket");
    return 0;
}
```

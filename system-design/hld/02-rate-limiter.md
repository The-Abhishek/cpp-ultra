# Rate Limiter High Level Design

## Step 1: Requirements Clarification
**Functional Requirements (FR):**
- Limit requests based on IP, User ID, or API Key.
- Highly configurable rules (e.g., 5 req/sec for Free tier, 100 req/sec for Premium).
- Inform clients when they are throttled (HTTP 429 Too Many Requests).

**Non-Functional Requirements (NFR):**
- **Low Latency**: <1ms overhead (cannot block core API logic).
- **High Availability**: 99.99% uptime. If the rate limiter fails, it should fail open (allow requests).
- **Distributed**: Work correctly across multiple servers.

## Step 2: Architecture Placement

Where should the rate limiter go?
1. **Client-side**: Unreliable, can be tampered with.
2. **Server-side**: Tightly coupled with business logic.
3. **API Gateway / Middleware (Chosen)**: Centralized, language-agnostic, handles load before it reaches app servers.

```ascii
    +--------+       +-------------------+       +-------------+
    |        |       |                   |       |             |
    | Client +------>+    API Gateway    +------>+ App Servers |
    |        |       | (w/ Rate Limiter) |       |             |
    +--------+       +---------+---------+       +-------------+
                               |
                         +-----v-----+
                         |           |
                         |   Redis   |
                         |           |
                         +-----------+
```

## Step 3: Deep Dive into Algorithms

### 1. Token Bucket
A bucket holds tokens. Tokens are added at a constant rate. Requests consume tokens.
- **Pros**: Easy to implement, memory efficient, allows brief bursts.
- **Cons**: Tuning bucket size and refill rate can be tricky.

**Pseudocode:**
```cpp
bool allowRequest(string userId) {
    long currentTime = getTime();
    auto bucket = redis.get(userId);
    
    // Refill logic
    long tokensToAdd = (currentTime - bucket.lastRefillTime) * REFILL_RATE;
    bucket.tokens = min(BUCKET_CAPACITY, bucket.tokens + tokensToAdd);
    bucket.lastRefillTime = currentTime;
    
    if (bucket.tokens > 0) {
        bucket.tokens--;
        redis.set(userId, bucket);
        return true; // Allow
    }
    return false; // Reject
}
```

### 2. Leaky Bucket
Requests enter a queue (bucket). Processed at a fixed rate.
- **Pros**: Smooths out traffic (traffic shaping).
- **Cons**: Bursts fill the queue, blocking subsequent requests; not ideal for real-time APIs.

### 3. Fixed Window Counter
Count requests in fixed time windows (e.g., 10:00:00 to 10:01:00).
- **Pros**: Simple, fast.
- **Cons**: Spike at edges of windows can allow 2x the rate limit.

### 4. Sliding Window Log
Keep timestamps of all requests in a sorted set. Remove timestamps older than the window.
- **Pros**: Perfectly accurate.
- **Cons**: High memory footprint (storing every timestamp).

### 5. Sliding Window Counter (Hybrid/Chosen)
Combines fixed window and sliding window log. Calculates a weighted sum of the previous window and current window.
- **Pros**: Smooths out edge spikes, low memory footprint.

## Step 4: Step-by-Step Data Flow

1. Request arrives at API Gateway with a UserID.
2. API Gateway evaluates rate limit rules (e.g., 10 req / minute).
3. Executes a Lua script in Redis for the specific UserID (Lua scripts ensure atomicity).
4. **If limit exceeded:** Return HTTP 429 Too Many Requests.
5. **If allowed:** Forward request to App Servers.

## Step 5: Distributed Environment Challenges

**1. Race Conditions**
In a distributed setup, two API gateways might read/write to Redis simultaneously.
- **Solution**: Use Redis Lua scripts. Lua scripts are executed atomically in Redis, preventing concurrent updates from overwriting each other. Alternatively, use Redis `MULTI/EXEC`.

**2. Synchronization**
If using local memory on API gateways, state diverges.
- **Solution**: Use a centralized datastore like Redis.

## Step 6: HTTP Headers

When returning responses, include metadata:
- `X-Ratelimit-Limit`: The allowed number of requests per window.
- `X-Ratelimit-Remaining`: Remaining requests in current window.
- `X-Ratelimit-Reset`: Unix timestamp when the limit resets.
- `Retry-After`: (On 429 response) Seconds to wait before retrying.

> **Interview Tip**: Mention "Fail Open" vs "Fail Closed". If Redis goes down, we should "Fail Open" (let traffic through) to not break the core product, relying on auto-scaling to absorb the load.

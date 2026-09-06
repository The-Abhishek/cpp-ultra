# 04. Caching Strategies

Caching improves read/write performance by storing frequently accessed data in fast, volatile memory (RAM).

## Reading Strategies

### 1. Cache-Aside (Lazy Loading)
The application is responsible for reading and writing from storage.
- **Flow:** App asks Cache -> (Miss) -> App asks DB -> App writes to Cache -> Returns data.
- **Pros:** Cache only contains requested data. Resilient to cache failure (DB takes over).
- **Cons:** Cache miss penalty is high (3 trips). Data can become stale.

### 2. Read-Through
The cache sits between the app and the DB. The cache itself is responsible for fetching from the DB on a miss.
- **Flow:** App asks Cache -> (Miss) -> Cache fetches from DB -> Cache returns data.
- **Pros:** Transparent to the app. Simplifies app code.
- **Cons:** Similar to Cache-Aside, miss penalty is high.

## Writing Strategies

### 3. Write-Through
Data is written to the cache and the DB at the same time.
- **Flow:** App writes to Cache -> Cache writes to DB -> Ack to App.
- **Pros:** Data in cache is never stale. Fast reads.
- **Cons:** Write latency is high (must wait for both writes).

### 4. Write-Behind (Write-Back)
Data is written to the cache, and asynchronously to the DB.
- **Flow:** App writes to Cache -> Ack to App -> (Async) Cache writes to DB.
- **Pros:** Extremely fast writes and reads. Good for write-heavy workloads.
- **Cons:** Risk of data loss if the cache crashes before the async DB write.

### 5. Write-Around
Data is written directly to the DB, bypassing the cache.
- **Flow:** App writes to DB -> Ack to App. (Read is handled by Cache-Aside).
- **Pros:** Good for data that is written once and rarely read. Prevents cache churn.

## Cache Invalidation & Eviction
How do we keep the cache fresh and bounded in size?

### Eviction Policies (When cache is full)
- **LRU (Least Recently Used):** Discard the least recently accessed items first. (Most common).
- **LFU (Least Frequently Used):** Discard items with the lowest access count.
- **FIFO (First In First Out):** Discard oldest items regardless of usage.

### Invalidation (When data changes)
- **TTL (Time To Live):** Keys expire after a set time.
- **Explicit Invalidation:** App deletes the cache key when updating the DB.

## Thundering Herd / Cache Stampede
- **Problem:** When a highly-accessed cache key expires, thousands of concurrent requests miss the cache and hit the DB simultaneously, bringing it down.
- **Mitigation:**
  - **Mutex/Locks:** Only allow one request to hit the DB; others wait for the cache to be populated.
  - **Probabilistic Early Expiration (PERF):** Recompute the cache slightly before the TTL expires.

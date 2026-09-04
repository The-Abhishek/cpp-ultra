# Design a Distributed Cache (Redis / Memcached)

## 1. Clarify Requirements

### Functional Requirements (FR)
- **Basic Operations**: `GET`, `SET`, `DELETE` operations.
- **Eviction**: Automatic removal of keys when memory is full using policies (LRU, LFU, TTL).
- **TTL (Time to Live)**: Support expiry on keys.
- **Cluster Mode**: Distributed architecture across multiple nodes with high availability.

### Non-Functional Requirements (NFR)
- **Latency**: Sub-millisecond [P99 (99th percentile latency) - latency experienced by the top 1% of the slowest requests].
- **Throughput**: High [QPS (Queries Per Second) - number of requests handled per second], e.g., 10M QPS.
- **Availability**: 99.99% availability (about 52 minutes of downtime per year).
- **Consistency**: Eventual consistency is usually acceptable, though strong consistency might be required in specific use cases [CAP Theorem - Consistency, Availability, Partition Tolerance].

> **Interview Tip**: Always establish whether the cache needs to survive crashes. Redis offers persistence via RDB/AOF [WAL (Write-Ahead Log)], whereas Memcached is purely in-memory.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation / Estimate |
|--------|------------------------|
| **QPS** | 10M requests / sec (9M reads, 1M writes) |
| **Data Size** | 1B objects * 1KB avg size = 1TB memory required |
| **Network Bandwidth** | 10M * 1KB = 10 GB/s bandwidth |
| **Nodes Required** | Assuming 64GB RAM per node: 1TB / 64GB ≈ 16 nodes |

---

## 3. High-Level Architecture

```text
       +----------+         +----------+
       | Client 1 |         | Client 2 |
       +----+-----+         +----+-----+
            |                    |
            v                    v
      +-----+--------------------+-----+
      |        Load Balancer           |
      +-----+--------------------+-----+
            |                    |
            v                    v
      +-----+-----+        +-----+-----+
      | Router /  |        | Router /  |  <-- Computes Consistent Hash
      | Proxy Node|        | Proxy Node|
      +-----+-----+        +-----+-----+
            |                    |
            +---------+----------+
                      |
        +-------------+-------------+
        |             |             |
  +-----v-----+ +-----v-----+ +-----v-----+
  |Cache Node1| |Cache Node2| |Cache Node3| <-- Distributed Cache Ring
  | (Shard A) | | (Shard B) | | (Shard C) |
  +-----------+ +-----------+ +-----------+
        |             |             |
        +-------------+-------------+
                      | Cache Miss / Fallback
                +-----v-----+
                | Database  |
                +-----------+
```

---

## 4. API Design

```text
- get(key: string) -> bytes
- set(key: string, value: bytes, ttl: int) -> bool
- delete(key: string) -> bool
```

---

## 5. Detailed Component Design

### 5.1 Consistent Hashing

A standard hash modulo `N` (e.g., `hash(key) % N`) causes massive cache misses when a node is added or removed. **Consistent Hashing** maps both keys and nodes to a circular hash space (e.g., 0 to 2^32 - 1).

```text
    Hash Ring with Virtual Nodes

           NodeA (v1) 
          /          \
  NodeB (v2)        NodeC (v1)
        |              |
  NodeC (v2)        NodeA (v2)
          \          /
           NodeB (v1)
```

**Pseudocode for Consistent Hashing:**

```cpp
class ConsistentHash {
private:
    std::map<int, string> ring;
    int virtual_nodes = 100; // V-nodes for even distribution
public:
    void addNode(string node) {
        for(int i = 0; i < virtual_nodes; i++) {
            int hash = hashFunction(node + std::to_string(i));
            ring[hash] = node;
        }
    }
    
    string getNode(string key) {
        if(ring.empty()) return "";
        int hash = hashFunction(key);
        auto it = ring.lower_bound(hash);
        if(it == ring.end()) {
            it = ring.begin(); // Wrap around
        }
        return it->second;
    }
};
```

### 5.2 Eviction Policies (LRU)

When cache is full, we must remove old items. **LRU (Least Recently Used)** is standard. Implemented via a Hash Map (for O(1) lookups) + Doubly Linked List (DLL) (for O(1) removals/inserts).

```cpp
class LRUCache {
    struct Node {
        string key;
        string value;
        Node* prev; Node* next;
    };
    int capacity;
    std::unordered_map<string, Node*> cache;
    Node* head; Node* tail;

    void moveToHead(Node* node) {
        // Unlink and insert at head
    }
    void removeTail() {
        // Remove node at tail (LRU item)
    }
public:
    string get(string key) {
        if(cache.find(key) == cache.end()) return "";
        Node* node = cache[key];
        moveToHead(node);
        return node->value;
    }
    void set(string key, string value) {
        if(cache.find(key) != cache.end()) {
            cache[key]->value = value;
            moveToHead(cache[key]);
        } else {
            if(cache.size() == capacity) removeTail();
            Node* newNode = new Node{key, value};
            // Add to head and cache
        }
    }
};
```

### 5.3 Cache Strategies

1.  **Cache-Aside**: Application asks Cache -> Miss -> Asks DB -> Updates Cache. (Most common).
2.  **Write-Through**: Application writes to Cache -> Cache synchronously writes to DB.
3.  **Write-Behind (Write-Back)**: Application writes to Cache -> Cache async writes to DB (faster, but risks data loss).

---

## 6. Addressing Bottlenecks

### Cache Stampede / Thundering Herd
When a highly popular key (e.g., celebrity tweet) expires, thousands of threads simultaneously query the DB to rebuild it.
- **Solution 1**: **Mutex/Locking** - Only one thread builds the cache, others wait.
- **Solution 2**: **Probabilistic Early Expiry** - Recompute cache *before* it actually expires in the background.

---

## 7. Scaling and Resilience
- **Replication**: Primary-Replica pattern. Primary takes writes, replicas take reads. 
- **Failure Detection**: Nodes use a **Gossip Protocol** to ping each other. If Node A goes down, other nodes mark it as dead and route keys to the next node on the consistent hash ring.

## 8. Summary & Wrap Up
- A robust distributed cache hinges on memory efficiency, fast networking, and intelligent routing (consistent hashing).
- Understanding how to prevent cache stampedes demonstrates deep operational experience.

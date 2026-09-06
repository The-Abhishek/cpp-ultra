# Design a Distributed Message Queue (Kafka)

## 1. Clarify Requirements

### Functional Requirements (FR)
- **Publish/Subscribe**: Producers send messages, consumers read them.
- **Consumer Groups**: Multiple consumers can collaborate to read messages from a topic.
- **Message Ordering**: Strict ordering required within a partition.
- **Delivery Guarantee**: At-least-once delivery semantics.

### Non-Functional Requirements (NFR)
- **Throughput**: 1M messages per second [QPS (Queries Per Second)].
- **Durability**: Messages must be saved to disk without data loss [WAL (Write-Ahead Log)].
- **Scalability**: Horizontal scaling of both producers/consumers and broker nodes.

> **Interview Tip**: Real Kafka uses disk for everything (append-only logs), leveraging Sequential I/O which is often faster than Random RAM I/O.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation / Estimate |
|--------|------------------------|
| **Write Rate** | 1M msgs/sec * 1KB avg size = 1 GB/s |
| **Storage (1 week)** | 1 GB/s * 86400s * 7 days = ~600 TB |
| **Nodes Required** | Assuming 10TB per node: ~60 nodes |

---

## 3. High-Level Architecture

```text
  +------------+   publish(topic, msg)    +-------------------+
  | Producer A +------------------------->|   Broker Node 1   |
  +------------+                          |  (Partition 0)    |
                                          +---------+---------+
  +------------+                          | Storage: Log File |
  | Producer B +-----+                    +---------+---------+
  +------------+     |
                     |                    +-------------------+
                     +------------------->|   Broker Node 2   |
                                          |  (Partition 1)    |
                                          +---------+---------+
                                          | Storage: Log File |
                                          +---------+---------+
                                                ^      ^
                           consume(topic)       |      |
                    +---------------------------+      |
                    |                                  |
  +-----------------+--+            +------------------+--+
  | Consumer 1 (Grp A) |            | Consumer 2 (Grp A)  |
  +--------------------+            +---------------------+
```

---

## 4. API Design

```text
- publish(topic: string, partition_key: string, message: bytes) -> bool
- subscribe(topic: string, consumer_group: string) -> Subscription
- consume(subscription_id: string, batch_size: int) -> List<Message>
- commit_offset(subscription_id: string, offset: int) -> bool
```

---

## 5. Detailed Component Design

### 5.1 Partitioning & Storage
A **Topic** is split into multiple **Partitions**. Each partition is an **append-only log** (a series of segment files on disk).
- New messages are appended to the end.
- Each message gets a sequential ID called an **Offset**.

```text
Partition Log File:
+---+---+---+---+---+---+
| 0 | 1 | 2 | 3 | 4 | 5 |  <-- Offsets
+---+---+---+---+---+---+
                 ^
                 | Consumer Offset (Current Read Position)
```

**Partition Assignment Algorithm:**
```cpp
int getPartition(string topic, string key, int num_partitions) {
    if (key.empty()) {
        return round_robin_counter++ % num_partitions; // Round robin if no key
    }
    return hashFunction(key) % num_partitions; // Hash key to maintain order
}
```

### 5.2 Consumer Groups
A Consumer Group is a logical group of consumers. 
- Rule: **One partition can be read by AT MOST ONE consumer in a group.** 
- This guarantees message ordering within the partition.

### 5.3 At-Least-Once & Exactly-Once
- **At-Least-Once**: Consumer reads message -> processes -> commits offset. If it crashes before commit, the new consumer reads it again (duplicate).
- **Exactly-Once**: Achieved via **Idempotent Producers** (giving each message a sequence number) and **Transactional Writes** to ensure atomic commits of offsets and state.

---

## 6. Addressing Bottlenecks

### Log Compaction
To save space, background threads merge segment files and keep only the latest value for a specific key (useful for state changes, e.g., user profile updates).

### Page Cache & Zero-Copy
Kafka relies heavily on the OS Page Cache. Data read from disk is sent directly to the network socket using the `sendfile()` system call, bypassing application space (**Zero-Copy**).

---

## 7. Scaling and Resilience
- **Replication**: Each partition has 1 Leader and multiple Followers. Followers pull data from Leader.
- **ISR (In-Sync Replicas)**: Only replicas that are caught up with the leader can become the new leader if the current one dies.
- **Leader Election**: Handled by ZooKeeper (or KRaft in newer Kafka versions).

## 8. Summary
- Disk-based sequential append logs provide extreme write throughput.
- Partitioning is the key to scalability.
- Managing consumer offsets correctly is crucial for delivery guarantees.

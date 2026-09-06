# System Design: Distributed Key-Value Store (DynamoDB/Cassandra)

## 1. Requirements

### Functional Requirements (FR)
- `put(key, value)`
- `get(key)`
- Data replication for fault tolerance.
- Tunable consistency levels (Eventual vs Strong).

### Non-Functional Requirements (NFR)
- Sub-millisecond latency.
- High availability (99.99%).
- Scalability (add/remove nodes dynamically).

## 2. Terminology & Concepts
- **Consistent Hashing**: Distributing data across nodes in a ring. Minimizes data movement when nodes join/leave.
- **Virtual Nodes (vnodes)**: Mapping a physical server to multiple points on the hash ring to balance load.
- **Quorum**: W (write nodes) + R (read nodes) > N (replication factor). Ensures strong consistency.
- **Vector Clocks**: Versioning mechanism to detect and resolve conflicts during replication.
- **Gossip Protocol**: Decentralized p2p protocol for cluster state dissemination.
- **LSM Tree (Log-Structured Merge Tree)**: Write-optimized storage engine. (Memtable -> SSTable -> Compaction).

## 3. Back-of-the-Envelope Estimation
- Assuming 100M Daily Active Users.
- 10 Reads/day, 2 Writes/day per user.
- Read QPS: ~11.5K, Write QPS: ~2.3K.
- Capacity planning depends on replication factor (usually N=3).

## 4. Architecture Diagram

```mermaid
flowchart TD
    Client --> |Get/Put| Coordinator[Coordinator Node]
    Coordinator --> |Hash(key)| Ring((Hash Ring))
    
    Coordinator --> N1[Node 1]
    Coordinator --> N2[Node 2]
    Coordinator --> N3[Node 3]
    
    N1 --> LSM1[(LSM Tree)]
    N2 --> LSM2[(LSM Tree)]
    N3 --> LSM3[(LSM Tree)]
```

### Storage Engine (LSM Tree)
```text
Write -> Write-Ahead Log (WAL) -> Memtable (In-Memory) -> Flush to SSTable (Disk) -> Background Compaction
```

## 5. Core Components & Deep Dive

### Partitioning & Replication
Data is partitioned using consistent hashing. A key is hashed, placed on the ring, and stored on the first `N` unique physical nodes walking clockwise.

### Tunable Consistency (Quorum)
- `N = 3` (replicas).
- Fast read/write: `W=1, R=1` (Eventual consistency).
- Strong consistency: `W=2, R=2` (Since W+R > N, at least one node has the latest data).

### Anti-Entropy (Merkle Trees)
Nodes compare data to detect inconsistencies using Merkle trees. Only branches with different hashes are synchronized, saving bandwidth.

## 6. Core Logic / Pseudocode

### Quorum Calculation & Coordinator Logic
```cpp
struct Response {
    string value;
    int version;
};

// Simplified read coordinator
string read_data(string key, int R) {
    vector<Node> replicas = get_preference_list(key); // top N nodes
    vector<Response> responses;
    
    // Async fetch from all replicas
    for (auto& node : replicas) {
        responses.push_back(node.read(key));
    }
    
    // Wait for 'R' successful responses
    wait_for_responses(responses, R);
    
    // Conflict resolution: return highest version (or use vector clocks)
    Response best = get_highest_version(responses);
    
    // Read repair: asynchronously update stale replicas
    async_read_repair(replicas, best);
    
    return best.value;
}
```

### LSM Tree Write Path
```cpp
void put(string key, string value) {
    // 1. Append to Write-Ahead Log for durability
    wal.append(key, value);
    
    // 2. Insert into in-memory skip list/red-black tree
    memtable.insert(key, value);
    
    // 3. If memtable is full, flush to disk as SSTable
    if (memtable.size() > THRESHOLD) {
        flush_to_sstable(memtable);
        memtable.clear();
    }
}
```

## 7. Failure Scenarios & Scaling
- **Node failure**: Handled by replication. Reads go to available replicas. Hinted handoff temporarily stores writes for downed nodes.
- **Network Partition**: CAP theorem applies. We favor Availability and Partition tolerance (AP system like Dynamo) or Consistency (CP system like etcd).

## 8. Interview Tips
- Mention the CAP theorem explicitly.
- Know the difference between B-Trees (read-optimized) and LSM-Trees (write-optimized).
- Discuss the tradeoff between Hinted Handoff (fast write recovery) and Merkle Trees (deep anti-entropy).

# 10. Consensus Algorithms

## The Problem
In a distributed system, how do multiple independent nodes agree on a single source of truth (state) when networks can fail, messages can be delayed, and nodes can crash?

## Paxos
- The theoretical foundation of distributed consensus.
- Extremely difficult to understand and implement correctly in the real world.
- Uses a multi-phase approach (Prepare, Promise, Accept, Accepted) to reach agreement.

## Raft (Understandable Consensus)
Raft was created as a more understandable alternative to Paxos. It decomposes the problem into three subproblems:

### 1. Leader Election
- Nodes are either **Leader**, **Follower**, or **Candidate**.
- If a Follower receives no heartbeats from a Leader within a randomized timeout, it becomes a Candidate and requests votes.
- If it receives a majority of votes, it becomes the new Leader.

### 2. Log Replication
- The Leader accepts all client requests (writes).
- It appends the command to its log and sends an `AppendEntries` RPC to Followers.
- Once a majority of Followers acknowledge the write, the Leader commits the entry and applies it to its state machine.

### ASCII Diagram: Raft Log Replication
```text
Client -> [ Write X=10 ] -> Leader
                              |
                     +--------+--------+
                     |                 |
                   (Append)         (Append)
                     v                 v
                Follower 1         Follower 2
                     |                 |
                   (Ack)             (Ack)
                     +--------+--------+
                              |
                    [ Commit X=10 on Leader ] -> Ack to Client
                              |
                   [ Notify Followers to Commit ]
```

## ZAB (ZooKeeper Atomic Broadcast)
- Custom protocol used specifically by Apache ZooKeeper.
- Very similar to Raft but optimized for high-throughput primary-backup systems.

## Use Cases for Consensus
You don't use Raft for everything. Consensus is slow because it requires a majority network quorum. Use it only for critical metadata:
- **Leader Election:** Deciding which node acts as the master.
- **Distributed Configuration:** Storing critical cluster metadata (e.g., where are the shards located).
- **Distributed Locks:** Ensuring only one process holds a resource.

## Popular Implementations
- **etcd:** Key-value store that uses Raft. The brain of Kubernetes.
- **ZooKeeper:** Uses ZAB. The brain of Hadoop, Kafka (historically).
- **Consul:** Uses Raft for service discovery and configuration.

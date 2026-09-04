# 01. CAP Theorem & PACELC

The CAP Theorem is a foundational concept in distributed systems. It states that a distributed data store can only guarantee two out of the following three properties simultaneously:

## Core Concepts
- **[C] Consistency**: Every read receives the most recent write or an error. All nodes see the same data at the same time.
- **[A] Availability**: Every request receives a (non-error) response, without the guarantee that it contains the most recent write.
- **[P] Partition Tolerance**: The system continues to operate despite an arbitrary number of messages being dropped (or delayed) by the network between nodes.

> **Interview Tip:** In a distributed system over a network, network partitions (P) are unavoidable. You **must** choose P. Therefore, the real choice is between **CP** and **AP**.

## ASCII Diagram: CAP Triangle
```text
           Consistency (C)
             /      \
            /        \
           /          \
   CP     /            \     AP
         /              \
        /                \
       /                  \
 Partition(P)----------Availability (A)
```
*(Note: CA is only possible in single-node systems, not distributed networks)*

## Trade-offs

### CP Systems (Consistency + Partition Tolerance)
When a network partition occurs, the system will return an error or timeout rather than returning stale data.
- **When to choose:** Financial systems, billing, inventory where accuracy is paramount.
- **Examples:** HBase, MongoDB (with primary reads), ZooKeeper

### AP Systems (Availability + Partition Tolerance)
When a network partition occurs, the system will return the most recent available version of the data, which might be stale.
- **When to choose:** Social media feeds, product reviews, logs where availability matters more than strict consistency.
- **Examples:** Cassandra, DynamoDB (with eventual consistency), CouchDB

## PACELC Theorem
CAP only applies when a partition occurs. PACELC extends CAP to address normal operations:
- **P**artition ? **A**vailability or **C**onsistency
- **E**lse ? **L**atency or **C**onsistency

**Examples based on PACELC:**
- **DynamoDB/Cassandra (PA/EL):** If Partition -> Availability. Else -> Latency.
- **MongoDB (PA/EC):** If Partition -> Availability (elects new leader). Else -> Consistency.
- **HBase (PC/EC):** If Partition -> Consistency. Else -> Consistency.

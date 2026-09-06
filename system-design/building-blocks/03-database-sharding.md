# 03. Database Sharding

## What is Sharding?
Sharding (also called Horizontal Partitioning) is the process of splitting a large database table across multiple database servers to distribute the data and traffic load.

## Sharding Strategies

### 1. Hash-based Sharding (Algorithmic)
- **How it works:** `hash(Partition_Key) % NUM_SHARDS`
- **Pros:** Even distribution of data and traffic.
- **Cons:** Hard to add/remove shards (requires resharding, though Consistent Hashing helps). Range queries are inefficient as they scatter across all shards.
- **Use Case:** High-throughput writes where lookup by exact ID is common (e.g., User Profiles).

### 2. Range-based Sharding
- **How it works:** Divide data based on ranges of a key (e.g., User IDs 1-1000 -> Shard 1, 1001-2000 -> Shard 2).
- **Pros:** Excellent for range queries (e.g., finding all users in a specific ZIP code or date range). Easy to add new shards.
- **Cons:** High risk of data hotspots. (e.g., If sharding by date, all current traffic hits the "today" shard).

### 3. Directory-based Sharding (Lookup Table)
- **How it works:** A separate lookup service maintains a mapping of `Key -> Shard ID`.
- **Pros:** Highly flexible. Easy to move data around and update the mapping dynamically.
- **Cons:** The lookup table becomes a single point of failure (SPOF) and a potential bottleneck. Requires caching.

### 4. Geo-based Sharding
- **How it works:** Shard based on user location (e.g., EU users in EU shard, US users in US shard).
- **Pros:** Reduces latency, helps with data compliance (GDPR).
- **Cons:** Uneven load (US might be much busier than other regions).

## Challenges of Sharding

### 1. Resharding (Data Migration)
When a shard gets too large, it must be split. This is complex without downtime.
- **Solution:** Pre-shard (start with a large number of logical shards distributed across a few physical nodes, then move logical shards to new nodes as you scale).

### 2. Cross-Shard Queries (Distributed Joins)
Joins across shards are extremely expensive over a network.
- **Solution:** Denormalization (duplicate data to avoid joins). Or perform joins in the application code.

### 3. Hotspot Problem (Celebrity Problem)
Some keys get exponentially more traffic.
- **Solution:** Add random suffixes to hot keys to distribute them (`bieber_1`, `bieber_2`), or handle them purely in an aggressive caching layer.

> **Interview Tip:** Always start by suggesting scaling up (Vertical Scaling) or Read Replicas before jumping to Sharding. Sharding introduces massive operational complexity and should be a last resort.

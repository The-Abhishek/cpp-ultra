# 02. Consistent Hashing

## Problem Statement
In distributed caching or databases, we often use `hash(key) % N` to map data to servers (where N is the number of servers).
- **The Issue:** If we add or remove a server, N changes. This changes the mapping for almost all keys, requiring massive data movement (rebalancing).

## Solution: Consistent Hashing
Consistent Hashing maps both data keys and servers to the same conceptual circle (Hash Ring), typically sized `[0, 2^32 - 1]`.

### ASCII Diagram: Hash Ring
```text
          [0 / 2^32]
             |
       K1    |
      *      |       * S1
             |
             |
             |
* S3         |                 * K2
             |
             |
             |
        *    |       *
       K3    |      S2
             |
          [2^31]
```
- **S1, S2, S3** are Servers.
- **K1, K2, K3** are Keys.
- **Rule:** A key is assigned to the first server found by moving clockwise from the key's position on the ring.
  - K2 -> S2
  - K3 -> S3
  - K1 -> S1

### Impact of Scaling
- **Adding a Server:** Only affects the keys that fall between the new server and the previous server.
- **Removing a Server:** Only its immediate keys move to the next server.
- **Result:** Instead of moving `O(K)` keys, only `O(K/N)` keys are moved.

## Virtual Nodes (V-Nodes)
- **Problem:** Real servers might not be distributed evenly on the ring, leading to unbalanced loads (hotspots).
- **Solution:** Introduce Virtual Nodes. Map each physical server to multiple points on the ring.
- **Benefit:** Better load distribution and easier scaling for heterogeneous servers (bigger servers get more V-Nodes).

## Core Algorithm / Pseudocode
```cpp
class ConsistentHash {
private:
    int replicas;
    std::map<size_t, std::string> ring; // map hash -> server_name
    std::hash<std::string> hasher;

public:
    ConsistentHash(int virtual_nodes) : replicas(virtual_nodes) {}

    void addServer(const std::string& server) {
        for (int i = 0; i < replicas; i++) {
            size_t hash_val = hasher(server + std::to_string(i));
            ring[hash_val] = server;
        }
    }

    std::string getServer(const std::string& key) {
        if (ring.empty()) return "";
        size_t hash_val = hasher(key);
        
        // Find the first server hash >= key hash (Clockwise traversal)
        auto it = ring.lower_bound(hash_val);
        if (it == ring.end()) {
            it = ring.begin(); // Wrap around the ring
        }
        return it->second;
    }
};
```

> **Interview Tip:** Mention Consistent Hashing whenever asked about scaling distributed caches (Memcached, Redis clusters), NoSQL DBs (Cassandra, DynamoDB), or CDNs routing requests.

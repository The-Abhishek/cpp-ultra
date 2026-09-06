# System Design: Autocomplete / Typeahead Suggestion

## 1. Requirement Clarification

### Functional Requirements (FR)
- As a user types, show top 5-10 suggestions.
- Suggestions should be based on search volume (trending/popular).
- Updates to trending queries should reflect quickly.

### Non-Functional Requirements (NFR)
- **Extreme Low Latency**: < 100ms response time (must feel instant).
- **High Throughput**: 10M QPS (users type multiple characters, each keystroke is a request).
- **High Availability**: If it fails, fallback to no suggestions (graceful degradation).

## 2. Back-of-the-Envelope Estimation

| Metric | Estimation | Result |
| :--- | :--- | :--- |
| **DAU [Daily Active Users]** | 100 Million | 100M |
| **Searches per User** | 10 searches/day | 1B searches/day |
| **Keystrokes per Search** | ~10 keystrokes | 10B API calls/day |
| **QPS [Queries Per Second]** | 10B / 100K seconds | ~100,000 QPS |
| **Data Size (Trie)** | 10M unique words * 30 bytes | ~300 MB (fits in RAM easily) |

> **Interview Tip**: The defining characteristic of this problem is the enormous read QPS. The entire data structure must live in memory (RAM). Database queries per keystroke will absolutely fail.

## 3. System Interface Definition

- `GET /v1/suggestions?prefix={query}&limit=5`
  - Response: `["apple", "amazon", "api design"]`

## 4. High-Level Design (Architecture)

We split the system into two parts:
1. **Data Gathering Service** (Writes): Collects user searches, aggregates frequencies.
2. **Query Service** (Reads): Serves the autocomplete results using an in-memory Trie.

```mermaid
graph TD
    Client((Client)) --> LB[Load Balancer]
    
    %% Read Path
    LB --> QS[Query Service]
    QS --> Cache[(Redis Cache)]
    QS --> Trie[Trie Servers (RAM)]
    
    %% Write Path
    LB --> WS[Search API]
    WS --> Log[(Event Logs)]
    Log --> Agg[Spark / MapReduce]
    Agg --> TrieBuilder[Trie Builder]
    TrieBuilder --> ZK[ZooKeeper: Update Trie]
    ZK -.-> Trie
```

### ASCII Architecture Flow
```text
[Keystrokes] -> [Trie Servers (In-Memory)] <--- [Trie Builder] <--- [MapReduce] <--- [Search Logs]
      |                 ^                                                                  |
      v                 |                                                                  v
[Redis Cache] ----------+                                                          (Daily/Hourly Batch)
```

## 5. Detailed Design (Deep Dives)

### 5.1 Data Structure: The Trie
A Trie (Prefix Tree) is the optimal structure for prefix matching.
- **Standard Trie**: To find top 5, you find the prefix node, then traverse ALL child trees to find leaf frequencies, sort them, and return top 5. *Too slow.*
- **Optimized Trie (Cached Top-K)**: Store the top-K most frequent searches *at every single node*.
  - Node `a` stores top 5 words starting with `a`.
  - Node `ap` stores top 5 words starting with `ap`.
  - Time complexity drops from $O(prefix + \text{children})$ to $O(prefix)$.

### 5.2 Core Logic: Trie Search & Insertion Pseudocode

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        # Stores Top K suggestions at this prefix level
        self.top_suggestions = [] # List of tuples: (word, frequency)

class Trie:
    def search(self, prefix):
        node = self.root
        for char in prefix:
            if char not in node.children:
                return []
            node = node.children[char]
        # O(1) retrieval once node is found!
        return node.top_suggestions

    def build_trie(self, word_freq_list):
        # Done offline by MapReduce
        for word, freq in word_freq_list:
            self.insert(word, freq)
```

### 5.3 Data Collection & Aggregation
We cannot update the Trie on every single search.
- **Sampling**: Log only 1 in 1000 requests.
- **Aggregation**: Use Apache Spark or Hadoop MapReduce to aggregate search frequencies hourly or daily.
- **Trie Update**: The Trie Builder creates a *new* Trie in memory, then hot-swaps it with the old Trie on the servers to avoid read-locks.

### 5.4 Real-time Trending
MapReduce is too slow for viral news (e.g., celebrity news).
- Use a **Sliding Window Counter** in Redis (or Apache Flink) for recent 15-minute windows.
- The Query Service fetches from both the static Trie and the Real-time Redis cache, merging the results before returning.

## 6. Bottlenecks & Fault Tolerance

- **Trie Too Large**: Shard the Trie based on prefix. 
  - e.g., Server 1 handles `a-m`, Server 2 handles `n-z`.
  - Use ZooKeeper to manage the mapping of prefixes to servers.
- **High Latency**: 
  - Add a CDN or browser-level caching. `Cache-Control: max-age=3600`.
  - Cache results of popular prefixes (like `a`, `the`) in Redis.

## 7. Summary & Interview Tips

- **Tip 1**: Storing the top-K results inside the Trie nodes is the "Aha!" moment of this interview. Don't traverse leaves during a read request.
- **Tip 2**: Emphasize decoupling reads and writes. Typeahead is a heavily read-skewed system.
- **Tip 3**: Mention WebSocket or Debouncing on the client side to reduce QPS (e.g., wait 50ms after the last keystroke before sending the API request).

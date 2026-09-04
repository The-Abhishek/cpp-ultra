# Design Google Search / Web Search Engine

## 1. Understand the Goal & Scope
**Problem:** Design a web search engine capable of crawling the internet, indexing documents, and serving highly relevant search results based on user queries at scale.

### Functional Requirements (FR)
- Web Crawler to fetch pages.
- Indexing pages to make them searchable.
- Search by keywords.
- Ranked results based on relevance.
- Autocomplete / Typeahead (Optional if time permits).
- Spell Check (Optional).

### Non-Functional Requirements (NFR)
- **Scale:** 5 Billion queries/day. Crawl 1B+ pages.
- **Latency:** Query response < 200ms.
- **Availability:** 99.99% uptime.
- **Freshness:** Popular pages updated frequently, others eventually.

---

## 2. Terminology & Core Concepts
- **[Inverted Index]**: A data structure mapping keywords to the documents containing them.
- **[TF-IDF]**: Term Frequency-Inverse Document Frequency. A statistical measure used to evaluate how important a word is to a document in a corpus.
- **[PageRank]**: An algorithm that measures the importance of website pages based on the quantity and quality of links to them.
- **[URL Frontier]**: A priority queue of URLs waiting to be crawled.
- **[Politeness Policy]**: Crawler rules to avoid DDoSing a website (e.g., delaying requests to the same domain).
- **[Document Sharding]**: Partitioning the index based on Document IDs.

---

## 3. Back-of-the-Envelope Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| **Search QPS** | 5 Billion / 86400 | ~60,000 queries/sec |
| **Indexed Pages** | 10 Billion pages | 10B pages |
| **Page Size** | Average 20KB HTML text per page | 20KB |
| **Storage (Pages)** | 10B * 20KB | ~200 TB (compressed) |
| **Index Size** | ~10-20% of text size | ~20-40 TB |

> **Interview Tip:** Emphasize that the index must fit entirely in RAM (across a distributed cluster) to achieve the < 200ms latency requirement. Disk I/O is too slow for real-time search.

---

## 4. System Interface Design (APIs)

```rest
GET /v1/search?q={query}&page={page_num}
Response: {
  total_results: 1500000,
  latency_ms: 125,
  results: [
    { title: "...", url: "...", snippet: "..." }
  ]
}
```

---

## 5. High-Level Design (Architecture Diagram)

```ascii
[ Crawler & Indexing Pipeline (Offline/Async) ]
+-------------+      +---------------+      +-------------+
| URL Frontier| ---> |  Web Crawler  | ---> | Page Store  |
+-------------+      +-------+-------+      | (Blob/HDFS) |
                             |              +-------------+
                             v
                     +---------------+      +-------------+
                     |    Indexer    | ---> |  Inverted   |
                     | (MapReduce)   |      |   Index     |
                     +---------------+      +-------------+

[ Search Pipeline (Online/Real-time) ]
+-------------+      +---------------+      +-------------+
|    User     | ---> |  API Gateway  | ---> | Query Node  |
+-------------+      +-------+-------+      +------+------+
                             |                     |
                             v                     v
                     +---------------+      +------+------+
                     | Result Cache  |      |  Ranker /   |
                     |   (Redis)     |      | Index Nodes |
                     +---------------+      +-------------+
```

---

## 6. Deep Dive

### 6.1 The Web Crawler

- **URL Frontier:** Uses a prioritized queue system (Kafka or custom DB). Prioritizes URLs based on PageRank and update frequency. Implements Politeness (groups by domain, adds delays).
- **Crawler Nodes:** Fetch HTML, extract links, put new links in URL Frontier, and save HTML to the Page Store (BigTable/Cassandra).
- **Deduplication:** Use Bloom Filters or compute hashes (e.g., Simhash) of page content to prevent indexing duplicate pages.

### 6.2 Inverted Index

This is the core data structure. It maps words to lists of Document IDs.

**Structure Diagram:**
```text
Word     -> [DocID1, DocID2, DocID3, ...]
"apple"  -> [D5(freq:2), D12(freq:1), D99(freq:8)]
"banana" -> [D12(freq:3), D105(freq:1)]
```

**Index Construction (Pseudocode via MapReduce):**
```python
# Mapper: Emit (word, doc_id) for each word in document
def mapper(doc_id, text):
    words = tokenize_and_stem(text)
    for word in words:
        emit(word, doc_id)

# Reducer: Combine doc_ids for the same word
def reducer(word, doc_id_list):
    sorted_unique_docs = sort_and_dedup(doc_id_list)
    write_to_index(word, sorted_unique_docs)
```

### 6.3 Ranking: TF-IDF and PageRank

When a user searches for "apple pie":
1. Intersect the inverted index lists for "apple" and "pie" to find documents containing both.
2. Score these documents.

**Scoring combines:**
- **Relevance (TF-IDF):** Does the page have the words frequently, and are the words rare overall?
- **Authority (PageRank):** Is this a highly linked-to page?

**Simplified PageRank Iteration:**
```cpp
// Pseudocode: PageRank calculation (Offline)
const double d = 0.85; // Damping factor
void computePageRank(Graph webGraph, int iterations) {
    for (int i = 0; i < iterations; i++) {
        for (Page p : webGraph.pages) {
            double rank = (1.0 - d);
            for (Page inlink : p.incomingLinks) {
                rank += d * (inlink.currentRank / inlink.outgoingLinks.size());
            }
            p.nextRank = rank;
        }
        // Update ranks for next iteration
    }
}
```

### 6.4 Serving & Sharding the Index

Since the index is 40TB, it won't fit on one machine.
- **Term-based Sharding:** "A-M" words go to Node 1, "N-Z" to Node 2.
  - *Pros:* Only query specific nodes.
  - *Cons:* Hotspots (e.g., "the" is on one node).
- **Document-based Sharding (Used in practice):** Node 1 gets Doc IDs 1-1M, Node 2 gets 1M-2M.
  - *Pros:* Perfectly balanced.
  - *Cons:* Every search query must be broadcast to *all* nodes (Scatter-Gather pattern). Node responses are aggregated and sorted globally.

---

## 7. Fault Tolerance and Scalability

- **Scatter-Gather Latency:** If you broadcast a query to 1000 nodes, the latency is bound by the *slowest* node (tail latency). Solution: Issue duplicate requests to replicas, or just timeout slow nodes and return partial results.
- **Caching:** Cache the entire SERP (Search Engine Results Page) for popular queries (e.g., "facebook", "weather") in a distributed Redis cluster. This absorbs a huge percentage of traffic.

---

## 8. Summary & Interview Tips

> **Interview Tip:** Clearly separate the **Offline pipeline** (Crawling, Index building, PageRank) from the **Online pipeline** (Querying, Ranking). They have completely different performance profiles.
> Mention the **Scatter-Gather** pattern for querying a document-sharded index. It's a critical concept for search architectures.

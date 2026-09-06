# System Design: Web Crawler

## 1. Requirement Clarification

### Functional Requirements (FR)
- Crawl the web starting from seed URLs.
- Extract HTML, parse content, and extract outgoing links.
- Store content for search engine indexing.
- Respect `robots.txt`.

### Non-Functional Requirements (NFR)
- **Scale**: Crawl 1 Billion pages per month.
- **Politeness**: Do not overload target servers.
- **Deduplication**: Do not crawl the same URL twice; detect duplicate content.
- **Fault Tolerance**: Crawlers crash. Must be able to resume seamlessly.

## 2. Back-of-the-Envelope Estimation

| Metric | Estimation | Result |
| :--- | :--- | :--- |
| **Pages/Month** | 1 Billion | 1B |
| **QPS [Queries Per Second]** | 1B / (30 * 24 * 3600) | ~400 QPS |
| **Peak QPS** | 2x Avg | ~800 QPS |
| **Average Page Size** | 500 KB (HTML) | - |
| **Storage (Monthly)** | 1B * 500 KB | ~500 TB / month |

> **Interview Tip**: 400 QPS is very low for standard APIs, but downloading 500KB of HTML over slow external networks means high I/O wait times. The system is heavily network I/O bound.

## 3. System Interface Definition
No public APIs. Internal components communicate via RPC/Message Queues.

## 4. High-Level Design (Architecture)

```mermaid
graph TD
    Seed[Seed URLs] --> Frontier[URL Frontier (Queues)]
    Frontier --> Fetcher[HTML Fetcher]
    Fetcher <--> DNS[DNS Resolver Cache]
    Fetcher --> Parser[Content Parser]
    
    Parser --> ContentDedup[Content Dedup (SimHash)]
    ContentDedup --> Storage[(Blob Storage: HTML)]
    
    Parser --> Extractor[Link Extractor]
    Extractor --> URLDedup[URL Dedup (Bloom Filter)]
    URLDedup --> Frontier
```

### ASCII Architecture Flow
```text
[Seed] -> [URL Frontier] ---> [DNS Cache]
               ^                 |
               |                 v
          [URL Dedup]       [Fetcher] ---> [External Web]
               ^                 |
               |                 v
        [Link Extractor] <- [Parser] ----> [Content Dedup] -> [Storage]
```

## 5. Detailed Design (Deep Dives)

### 5.1 URL Frontier: Politeness & Priority
The URL frontier is not a simple queue. It manages **Politeness** (no concurrent requests to the same host) and **Priority** (PageRank, freshness).

- **Implementation**: 
  - One FIFO queue per domain (e.g., `queue_nytimes`, `queue_wikipedia`).
  - Worker threads map 1:1 to a queue. A worker sleeps between requests to the same domain (e.g., delay of 2 seconds).

### 5.2 Deduplication: URLs and Content

**URL Deduplication (Bloom Filter)**
Checking if we've seen a URL out of 1 Billion URLs requires too much memory for a Hash Set.
- Use a **Bloom Filter**: A probabilistic data structure (bit array + multiple hash functions).
- False positives are possible (we might skip a new URL), but false negatives are impossible (we never crawl a URL twice).

**Content Deduplication (SimHash / MinHash)**
Websites often have mirrored content or slightly modified pages (date changes).
- Standard hashes (SHA-256) change completely if 1 byte changes.
- **SimHash** (Locality-Sensitive Hashing): Similar documents produce similar hashes. We compare hashes by calculating the Hamming distance (number of differing bits).

### 5.3 Core Logic: BFS Crawl Loop Pseudocode

```python
def web_crawler_worker(domain_queue):
    while True:
        url = domain_queue.pop()
        
        # 1. Check Robots.txt
        if not is_allowed_by_robots(url):
            continue
            
        # 2. Fetch HTML
        ip_address = dns_resolve(url)
        html = fetch_html(ip_address, url)
        
        # 3. Content Dedup
        doc_hash = generate_simhash(html)
        if db.exists_similar_hash(doc_hash, hamming_threshold=3):
            continue # Skip duplicate content
            
        # 4. Save Content
        save_to_blob_storage(url, html)
        
        # 5. Extract Links
        links = extract_links(html)
        for link in links:
            # URL Dedup via Bloom Filter
            if not bloom_filter.contains(link):
                bloom_filter.add(link)
                # Route to appropriate domain queue based on Priority
                push_to_frontier(link)
                
        # 6. Politeness delay
        time.sleep(POLITENESS_DELAY_SECONDS)
```

## 6. Bottlenecks & Fault Tolerance

- **DNS Resolution**: DNS lookup can take 10ms - 100ms. **Solution**: Custom DNS caching server locally on crawler nodes.
- **Crawler Node Crash**: The URL Frontier is backed by Kafka or Redis. If a worker dies, the URL goes back to the queue (acknowledgement mechanism).
- **Spider Traps**: Infinite loops (e.g., `site.com/a/a/a/...`). **Solution**: Max depth limit per URL.

## 7. Summary & Interview Tips

- **Tip 1**: Distinguish between URL deduplication (Bloom Filter) and Content deduplication (SimHash). This is a strong hire signal.
- **Tip 2**: Explicitly mention **Politeness**. Interviewers deduct heavy points if your crawler effectively DDOS'es a target website.
- **Tip 3**: Mention that BFS [Breadth-First Search] is preferred over DFS [Depth-First Search] because DFS might get stuck deep inside one domain, violating politeness and priority.

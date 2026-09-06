# 🏗️ System Design — Interview Mastery

> **23 HLD** • **20 LLD** • **10 Building Blocks** • **Whiteboard Diagrams** • **Core Logic & Code**

---

## 📖 Terminology Quick Reference

| Term | Meaning |
|------|---------|
| **HLD** | High-Level Design — distributed architecture, scaling, databases |
| **LLD** | Low-Level Design — OOP, classes, design patterns, working code |
| **DAU** | Daily Active Users |
| **MAU** | Monthly Active Users |
| **QPS** | Queries Per Second |
| **TPS** | Transactions Per Second |
| **P99** | 99th Percentile Latency (99% of requests finish under this time) |
| **SLA** | Service Level Agreement (uptime guarantee, e.g., 99.99%) |
| **CAP** | CAP Theorem — Consistency, Availability, Partition-tolerance (pick 2) |
| **ACID** | Atomicity, Consistency, Isolation, Durability (DB transaction guarantees) |
| **BASE** | Basically Available, Soft state, Eventually consistent (NoSQL model) |
| **CDN** | Content Delivery Network (edge caches close to users) |
| **LB** | Load Balancer (distributes traffic across servers) |
| **WAL** | Write-Ahead Log (crash recovery — write log before data) |
| **SST** | Sorted String Table (on-disk sorted key-value file, used in LSM trees) |
| **LSM** | Log-Structured Merge Tree (write-optimized DB engine) |
| **CQRS** | Command Query Responsibility Segregation (separate read/write models) |
| **CDC** | Change Data Capture (stream DB changes to other systems) |
| **RPC** | Remote Procedure Call (inter-service communication) |
| **gRPC** | Google's high-performance RPC framework (uses Protocol Buffers) |
| **SPOF** | Single Point of Failure |
| **TTL** | Time To Live (cache/data expiry duration) |
| **ELB/ALB** | Elastic/Application Load Balancer (AWS) |
| **OLTP** | Online Transaction Processing (real-time reads/writes) |
| **OLAP** | Online Analytical Processing (batch analytics, data warehouse) |

---

## 🎯 HLD — High-Level Design (Distributed Systems)

| # | Question | Difficulty | Key Concepts |
|---|----------|:----------:|--------------|
| [01](hld/01-url-shortener.md) | Design TinyURL | 🟢 | Hashing, Base62, Read-heavy, Caching |
| [02](hld/02-rate-limiter.md) | Design Rate Limiter | 🟡 | Token Bucket, Sliding Window, Redis |
| [03](hld/03-chat-system.md) | Design WhatsApp | 🔴 | WebSocket, Message Queue, Fanout |
| [04](hld/04-news-feed.md) | Design Twitter Feed | 🔴 | Fanout-on-write vs read, Ranking |
| [05](hld/05-uber.md) | Design Uber | 🔴 | Geohashing, Matching, Real-time |
| [06](hld/06-netflix.md) | Design Netflix | 🔴 | CDN, Adaptive Streaming, Recommendations |
| [07](hld/07-search-engine.md) | Design Google Search | 🔴 | Inverted Index, PageRank, Crawling |
| [08](hld/08-notification-system.md) | Design Notification Service | 🟡 | Push/Pull, APNs/FCM, Priority Queue |
| [09](hld/09-distributed-cache.md) | Design Redis | 🔴 | Consistent Hashing, Eviction, Replication |
| [10](hld/10-message-queue.md) | Design Kafka | 🔴 | Partitioning, Consumer Groups, Ordering |
| [11](hld/11-instagram.md) | Design Instagram | 🟡 | Image Storage, Feed Generation, CDN |
| [12](hld/12-dropbox.md) | Design Dropbox | 🔴 | Chunking, Dedup, Sync, Conflict Resolution |
| [13](hld/13-ticketmaster.md) | Design Ticketmaster | 🔴 | Seat Locking, Distributed Transactions |
| [14](hld/14-payment-system.md) | Design Payment Gateway | 🔴 | Idempotency, Exactly-once, Reconciliation |
| [15](hld/15-web-crawler.md) | Design Web Crawler | 🟡 | BFS, Politeness, URL Frontier, Dedup |
| [16](hld/16-typeahead.md) | Design Autocomplete | 🟡 | Trie, Prefix Matching, Caching |
| [17](hld/17-metrics-monitoring.md) | Design Monitoring System | 🟡 | Time-series DB, Aggregation, Alerting |
| [18](hld/18-distributed-id-generator.md) | Design Unique ID Generator | 🟡 | Snowflake, UUID, Clock Skew |
| [19](hld/19-key-value-store.md) | Design KV Store | 🔴 | LSM Tree, Replication, Quorum |
| [20](hld/20-stock-exchange.md) | Design Stock Exchange | 🔴 | Order Matching, FIFO Queue, Low Latency |
| [21](hld/21-google-maps.md) | Design Google Maps | 🔴 | QuadTree, Dijkstra, Map Tiles, ETA |
| [22](hld/22-hotel-booking.md) | Design Airbnb | 🟡 | Search, Availability, Double-Booking |
| [23](hld/23-ad-click-aggregator.md) | Design Ad Click Aggregator | 🟡 | Stream Processing, MapReduce, Lambda |

---

## 🔧 LLD — Low-Level Design (OOP + Code)

| # | Question | Difficulty | Key Patterns |
|---|----------|:----------:|--------------|
| [01](lld/01-parking-lot.md) | Parking Lot | 🟢 | Strategy, Factory, Observer |
| [02](lld/02-elevator-system.md) | Elevator/Lift System | 🟡 | State, Strategy, Observer |
| [03](lld/03-lru-cache.md) | LRU Cache | 🟢 | HashMap + Doubly Linked List |
| [04](lld/04-snake-and-ladder.md) | Snake & Ladder Game | 🟢 | Strategy, Composite |
| [05](lld/05-chess-game.md) | Chess Game | 🔴 | Strategy, Command, Observer |
| [06](lld/06-hotel-management.md) | Hotel Management | 🟡 | State, Observer, Singleton |
| [07](lld/07-library-management.md) | Library System | 🟢 | Repository, Observer, Strategy |
| [08](lld/08-food-delivery.md) | Food Delivery (Swiggy) | 🟡 | Strategy, Observer, State |
| [09](lld/09-splitwise.md) | Splitwise | 🟡 | Graph (Debt Simplification) |
| [10](lld/10-vending-machine.md) | Vending Machine | 🟡 | State Machine, Strategy |
| [11](lld/11-movie-ticket-booking.md) | Movie Ticket Booking | 🟡 | Strategy, Observer, Locking |
| [12](lld/12-traffic-signal.md) | Traffic Signal Controller | 🟡 | State Machine, Observer |
| [13](lld/13-logger-framework.md) | Logging Framework | 🟢 | Singleton, Chain of Responsibility |
| [14](lld/14-rate-limiter-lld.md) | Rate Limiter (Code) | 🟡 | Strategy, Sliding Window |
| [15](lld/15-task-scheduler.md) | Task Scheduler | 🟡 | Priority Queue, Strategy, Observer |
| [16](lld/16-file-system.md) | In-Memory File System | 🟡 | Composite, Iterator |
| [17](lld/17-atm-machine.md) | ATM Machine | 🟡 | State, Chain of Responsibility |
| [18](lld/18-tic-tac-toe.md) | Tic-Tac-Toe | 🟢 | Strategy, Observer |
| [19](lld/19-online-stock-brokerage.md) | Online Stock Brokerage | 🔴 | Observer, Strategy, Command |
| [20](lld/20-pub-sub-system.md) | Pub-Sub Messaging | 🟡 | Observer, Mediator |

---

## 🧱 Building Blocks (Referenced in HLD/LLD)

| # | Topic | When to Use |
|---|-------|-------------|
| [01](building-blocks/01-cap-theorem.md) | CAP Theorem | Every distributed system trade-off |
| [02](building-blocks/02-consistent-hashing.md) | Consistent Hashing | Sharding, distributed caching |
| [03](building-blocks/03-database-sharding.md) | Database Sharding | Horizontal scaling of databases |
| [04](building-blocks/04-caching-strategies.md) | Caching Strategies | Read-heavy workloads |
| [05](building-blocks/05-load-balancing.md) | Load Balancing | Traffic distribution |
| [06](building-blocks/06-message-queues.md) | Message Queues | Async processing, decoupling |
| [07](building-blocks/07-cdn.md) | CDN | Static content, media streaming |
| [08](building-blocks/08-sql-vs-nosql.md) | SQL vs NoSQL | Database selection decisions |
| [09](building-blocks/09-api-gateway.md) | API Gateway | Routing, auth, rate limiting |
| [10](building-blocks/10-consensus-algorithms.md) | Consensus Algorithms | Leader election, distributed agreement |

---

## 🗺️ Study Path

```
Week 1-2: Building Blocks (01-10) → Foundation
Week 3-4: HLD Easy/Medium (01, 02, 08, 11, 15, 16, 17, 18) → Core patterns
Week 5-6: HLD Hard (03, 04, 05, 06, 09, 10, 12, 13, 14, 19, 20) → Advanced
Week 7:   LLD Easy (01, 03, 04, 07, 13, 18) → OOP foundations
Week 8:   LLD Medium/Hard (02, 05, 06, 08-12, 14-17, 19, 20) → Full mastery
```

## 🎯 Interview Frequency

| 🔴 Always Asked | 🟡 Frequently Asked | 🟢 Sometimes Asked |
|-----------------|--------------------|--------------------|
| URL Shortener | Notification System | Web Crawler |
| Rate Limiter | Instagram | Google Maps |
| Chat System | Dropbox | Stock Exchange |
| News Feed | Ticketmaster | Ad Click Aggregator |
| Uber/Lyft | Payment System | Hotel Booking |
| Netflix/YouTube | Monitoring System | |
| Distributed Cache | ID Generator | |
| Message Queue | KV Store | |

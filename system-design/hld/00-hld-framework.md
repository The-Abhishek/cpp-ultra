# HLD (High-Level Design) — Universal Framework

> Use this 8-step framework for **every** HLD interview. Internalize the *process*, not individual answers.
> A 45-minute interview should roughly follow this time allocation.

---

## The 8-Step HLD Framework

```
┌──────────────────────────────────────────────────────────────────────┐
│                     SYSTEM DESIGN INTERVIEW (45 min)                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Step 1: Requirements & Scope ──────────────────── [2-3 min]  🎯    │
│      ↓                                                               │
│  Step 2: Back-of-Envelope Estimation ───────────── [2-3 min]  📊    │
│      ↓                                                               │
│  Step 3: API Design ────────────────────────────── [3-5 min]  🔌    │
│      ↓                                                               │
│  Step 4: Data Model & Storage ──────────────────── [5 min]    💾    │
│      ↓                                                               │
│  Step 5: High-Level Architecture ───────────────── [5-7 min]  🏗️    │
│      ↓                                                               │
│  Step 6: Deep Dive (Core Algorithm) ────────────── [10 min]   🔬    │
│      ↓                                                               │
│  Step 7: Scalability & Reliability ─────────────── [5 min]    📈    │
│      ↓                                                               │
│  Step 8: Trade-offs & Extensions ───────────────── [2-3 min]  ⚖️    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step 1 — Clarify Requirements & Scope (2-3 min)

**WHY:** Shows the interviewer you think before coding. Prevents designing the wrong system.

### What to ask:
```
┌─────────────────────────────────────────────────────────┐
│  REQUIREMENTS GATHERING CHECKLIST                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Functional Requirements (FR):                          │
│  ✓ What are the core features?                         │
│  ✓ Who are the users/actors?                           │
│  ✓ What are the main use cases?                        │
│                                                         │
│  Non-Functional Requirements (NFR):                     │
│  ✓ Scale: How many DAU (Daily Active Users)?           │
│  ✓ Latency: Real-time? Near-real-time? Batch?          │
│  ✓ Availability: 99.9%? 99.99%? (SLA target)          │
│  ✓ Consistency: Strong? Eventual?                      │
│  ✓ Read-heavy or Write-heavy?                          │
│                                                         │
│  Out of Scope:                                          │
│  ✓ What do we NOT need to design?                      │
│  ✓ Explicitly state boundaries                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> **💡 Interview Tip:** Start with "Let me clarify the requirements before jumping in." 
> This single sentence sets you apart from 80% of candidates who start drawing boxes immediately.

---

## Step 2 — Back-of-Envelope Estimation (2-3 min)

**WHY:** Proves you can think at scale. Drives storage, bandwidth, and infrastructure decisions.

### Estimation Cheat Sheet

| Category | Useful Numbers |
|----------|---------------|
| **Seconds in a day** | ~86,400 ≈ ~100K (use 10^5 for easy math) |
| **Seconds in a month** | ~2.5M ≈ ~2.5 × 10^6 |
| **1 char** | 1 byte (ASCII) / 2 bytes (UTF-16) |
| **1 image (compressed)** | ~200 KB |
| **1 short video (1 min)** | ~5 MB |
| **1 million requests/day** | ~12 QPS (Queries Per Second) |
| **SSD random read** | ~100 μs |
| **Network round-trip (same DC)** | ~0.5 ms |
| **Network round-trip (cross-continent)** | ~150 ms |
| **Redis GET** | ~0.1-0.5 ms |
| **MySQL simple query** | ~1-5 ms |
| **1 server handles** | ~10K-50K concurrent connections |

### Template:
```
DAU (Daily Active Users):           ____
Actions per user per day:            ____
Total requests/day:                  ____ = DAU × actions
QPS (Queries Per Second):           ____ = total / 86400
Peak QPS:                            ____ = QPS × 2-5
Storage per record:                  ____ bytes
Storage/day:                         ____ = records × size
Storage (5 years):                   ____ = daily × 365 × 5
Bandwidth:                           ____ = QPS × record_size
```

> **💡 Interview Tip:** Round aggressively. Use powers of 10. Say "Let's call it 100K QPS"
> not "Exactly 97,532 QPS." The point is showing the thought process.

---

## Step 3 — API Design (3-5 min)

**WHY:** Defines the contract between client and server. Shows you think in interfaces.

### Template:
```
POST   /api/v1/resource           → Create
GET    /api/v1/resource/{id}      → Read
PUT    /api/v1/resource/{id}      → Update
DELETE /api/v1/resource/{id}      → Delete
GET    /api/v1/resource?query=X   → Search/List

Headers: Authorization: Bearer <JWT token>
Query:   ?page=1&limit=20&sort=created_at
```

### What to mention:
- **Authentication:** JWT (JSON Web Token), OAuth 2.0, API keys
- **Pagination:** Cursor-based (better for real-time) vs Offset-based
- **Rate Limiting:** Mentioned in headers (X-RateLimit-Remaining)
- **Versioning:** /v1/ in URL path

---

## Step 4 — Data Model & Storage (5 min)

**WHY:** The database is the heart of the system. Wrong choice = wrong system.

### Decision Framework:
```
┌────────────────────────────────────────────────────────────┐
│           DATABASE SELECTION DECISION TREE                  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Need ACID transactions? ──→ YES ──→ SQL (PostgreSQL)     │
│         │                                                  │
│         NO                                                 │
│         │                                                  │
│  Need flexible schema? ───→ YES ──→ Document DB (MongoDB) │
│         │                                                  │
│         NO                                                 │
│         │                                                  │
│  Write-heavy + time-series? → YES → Wide-Column (Cassandra)│
│         │                                                  │
│         NO                                                 │
│         │                                                  │
│  Need graph relationships? → YES → Graph DB (Neo4j)       │
│         │                                                  │
│         NO                                                 │
│         │                                                  │
│  Need full-text search? ──→ YES → Elasticsearch           │
│         │                                                  │
│         NO                                                 │
│         │                                                  │
│  Key-value lookups only? ─→ YES → Redis / DynamoDB        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Step 5 — High-Level Architecture (5-7 min)

**WHY:** The whiteboard diagram. This is the core deliverable.

### Universal Architecture Pattern:
```
                            ┌─────────┐
                            │   CDN   │ (static assets, images, video)
                            └────┬────┘
                                 │
┌──────────┐    HTTPS     ┌──────┴──────┐     ┌─────────────────┐
│  Client  │ ──────────→  │ API Gateway │────→│  Auth Service   │
│ (Mobile/ │              │   + LB      │     │  (JWT/OAuth)    │
│  Web)    │              └──────┬──────┘     └─────────────────┘
└──────────┘                     │
                    ┌────────────┼────────────┐
                    │            │            │
              ┌─────┴─────┐ ┌───┴────┐ ┌────┴─────┐
              │ Service A │ │Service B│ │Service C │
              └─────┬─────┘ └───┬────┘ └────┬─────┘
                    │           │            │
              ┌─────┴─────┐    │      ┌─────┴──────┐
              │  Cache     │    │      │  Message   │
              │  (Redis)   │    │      │  Queue     │
              └─────┬─────┘    │      │ (Kafka)    │
                    │           │      └─────┬──────┘
              ┌─────┴──────────┴─────┐       │
              │     Database(s)      │  ┌────┴─────┐
              │  (SQL / NoSQL)       │  │ Workers  │
              └──────────────────────┘  └──────────┘
```

### Component Checklist:
| Component | Purpose | When to Include |
|-----------|---------|----------------|
| **CDN (Content Delivery Network)** | Serve static files from edge | Images, video, JS/CSS |
| **Load Balancer** | Distribute traffic | Always (multiple servers) |
| **API Gateway** | Routing, auth, rate limit | Microservices architecture |
| **Cache (Redis/Memcached)** | Reduce DB load | Read-heavy systems |
| **Message Queue (Kafka/SQS)** | Async processing | Write-heavy, decoupling |
| **Workers** | Background processing | Emails, notifications, ETL |
| **Blob Storage (S3)** | Large files | Images, videos, documents |
| **Search Engine (Elasticsearch)** | Full-text search | Search features |

---

## Step 6 — Deep Dive (10 min)

**WHY:** This is where you show depth. The interviewer picks 1-2 components to go deep on.

### What to deep-dive:
- The **most interesting algorithm** (matching, ranking, routing)
- The **hardest scaling problem** (hot keys, thundering herd)
- **Data flow** for the most critical path (write path or read path)

### Deep Dive Template:
```
Data Flow for [Critical Operation]:

  Step 1: Client sends request
      ↓
  Step 2: API Gateway validates + rate limits
      ↓
  Step 3: Service processes (describe algorithm)
      ↓
  Step 4: Write to DB / Cache
      ↓
  Step 5: Publish event to Message Queue
      ↓
  Step 6: Consumers process async tasks
      ↓
  Step 7: Response returned to client
```

---

## Step 7 — Scalability & Reliability (5 min)

### Scaling Playbook:
```
┌────────────────────────────────────────────────────────────┐
│                 SCALING DECISION MATRIX                     │
├──────────────────┬─────────────────────────────────────────┤
│  Problem         │  Solution                               │
├──────────────────┼─────────────────────────────────────────┤
│  Too many reads  │  Add cache (Redis), read replicas, CDN  │
│  Too many writes │  Message queue, sharding, async writes  │
│  Single DB limit │  Shard (horizontal partition by key)    │
│  Hot partition   │  Consistent hashing, salt keys          │
│  SPOF (Single    │  Replication, failover, multi-AZ        │
│   Point of       │  (Availability Zone) deployment         │
│   Failure)       │                                         │
│  Slow queries    │  Indexing, denormalization, CQRS        │
│  Data loss       │  WAL, replication factor 3, backups     │
│  Thundering herd │  Cache stampede lock, jitter, pre-warm  │
│  Cross-region    │  Multi-DC, eventual consistency,        │
│   latency        │  geo-routing                            │
└──────────────────┴─────────────────────────────────────────┘
```

### Reliability Patterns:
- **Circuit Breaker**: Stop calling a failing service, fail fast
- **Retry with Exponential Backoff**: 1s → 2s → 4s → 8s + jitter
- **Bulkhead**: Isolate failures (separate thread pools per service)
- **Health Checks**: Liveness (/healthz) + Readiness (/readyz)

---

## Step 8 — Trade-offs & Extensions (2-3 min)

**WHY:** Shows maturity. No system is perfect — acknowledge trade-offs.

### Common Trade-offs:
| Decision | Option A | Option B |
|----------|----------|----------|
| Consistency model | Strong consistency (all reads see latest write) | Eventual consistency (faster, more available) |
| Push vs Pull | Fanout-on-write (fast reads, expensive writes) | Fanout-on-read (cheap writes, slow reads) |
| SQL vs NoSQL | ACID guarantees, complex queries | Horizontal scaling, flexible schema |
| Monolith vs Microservices | Simple, fast development | Independent scaling, team autonomy |
| Cache-aside vs Write-through | Simple, eventual stale data | Always consistent, higher write latency |

> **💡 Interview Tip:** End with "If I had more time, I'd also look into..." 
> This shows awareness of what you *didn't* cover and leaves a strong final impression.

---

## 🎯 Interviewer Scoring Signals

```
┌─────────────────────────────────────────────────────────────┐
│               WHAT INTERVIEWERS ACTUALLY SCORE               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✅ STRONG SIGNALS (Do these)                              │
│  ─────────────────────────────                              │
│  • Clarified requirements before designing                  │
│  • Showed capacity estimation (even if rough)               │
│  • Justified EVERY technology choice ("I chose X because")  │
│  • Identified bottlenecks PROACTIVELY                       │
│  • Discussed trade-offs honestly                            │
│  • Drew clear diagrams with data flow arrows                │
│  • Drove the conversation (didn't wait for hints)           │
│                                                             │
│  ❌ WEAK SIGNALS (Avoid these)                              │
│  ────────────────────────────                               │
│  • Jumped straight to drawing boxes                         │
│  • Named technologies without explaining WHY                │
│  • Designed for "infinite scale" without estimation          │
│  • Said "we can just add more servers" without details      │
│  • Couldn't discuss failure scenarios                       │
│  • Only presented happy path                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

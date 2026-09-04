# 08. SQL vs NoSQL Databases

Choosing the right database is often the most critical decision in a system design interview.

## SQL (Relational Databases)
- **Examples:** PostgreSQL, MySQL, Oracle.
- **Characteristics:** Structured schema, table-based, supports complex joins.
- **Guarantees:** Strict ACID properties (Atomicity, Consistency, Isolation, Durability).
- **Scaling:** Primarily scales vertically (bigger machines). Sharding is possible but complex.
- **When to use:** Financial transactions, relationships matter more than scale, structured predictable data.

## NoSQL (Non-Relational Databases)

NoSQL is not one thing; it's a category. They generally trade strict ACID compliance for high scalability and availability (BASE properties: Basically Available, Soft state, Eventual consistency).

### 1. Document Stores
- **Examples:** MongoDB, CouchDB.
- **Data Model:** JSON/BSON documents. Schema-less or flexible schema.
- **When to use:** Rapid prototyping, e-commerce product catalogs, content management.

### 2. Key-Value Stores
- **Examples:** Redis, Memcached, DynamoDB.
- **Data Model:** Simple dictionary (Hash table). Fast `O(1)` lookups by key.
- **When to use:** Caching, user session management, leaderboards.

### 3. Wide-Column Stores
- **Examples:** Cassandra, HBase.
- **Data Model:** Tables with rows and dynamic columns, optimized for fast writes.
- **When to use:** High write throughput, time-series data, logging, IoT metrics.

### 4. Graph Databases
- **Examples:** Neo4j, Amazon Neptune.
- **Data Model:** Nodes (entities) and Edges (relationships).
- **When to use:** Social networks, recommendation engines, fraud detection.

## Decision Matrix

| Requirement | Choice | Reason |
|-------------|--------|--------|
| Complex transactions & Joins | SQL (PostgreSQL) | ACID guarantees |
| Low latency caching | KV (Redis) | In-memory speed |
| Insane write volume (Logs, IoT) | Wide-Column (Cassandra) | Append-only architecture |
| Flexible JSON schema | Document (MongoDB) | No migrations needed |

> **Interview Tip:** Real-world systems use **Polyglot Persistence** — meaning they use multiple databases! A system might use PostgreSQL for billing, Redis for caching, Cassandra for logs, and Neo4j for social graphs. Don't be afraid to mix them in an interview.

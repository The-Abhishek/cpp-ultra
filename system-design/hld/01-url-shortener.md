# URL Shortener (TinyURL) High Level Design

## Step 1: Requirements Clarification
**Functional Requirements (FR):**
- **Shorten URL**: Generate a short alias for a given long URL.
- **Redirect**: Forward users to the original long URL when the short alias is accessed.
- **Custom Alias**: Users can specify a custom short alias.
- **Analytics**: Track click counts for each shortened URL.
- **Expiration**: URLs can be set to expire after a certain duration.

**Non-Functional Requirements (NFR):**
- **Scale**: Handle 100M DAU [Daily Active Users], store 1B [Billion] new URLs per month.
- **Latency**: <10ms [milliseconds] redirection time.
- **Availability**: 99.99% uptime.
- **Durability**: Shortened links must never be lost.

> **Interview Tip**: Always establish the read-to-write ratio early. URL shorteners are heavily read-heavy.

## Step 2: Back-of-the-Envelope Estimations

| Metric | Calculation | Result |
| :--- | :--- | :--- |
| **Write QPS** [Queries Per Second] | 1B URLs / month / (30 days * 24h * 3600s) | ~400 QPS |
| **Read QPS** | Read/Write ratio of 100:1 -> 400 * 100 | ~40K QPS |
| **Storage (5 years)** | 1B URLs/month * 60 months * 500 bytes/URL | ~30 TB [Terabytes] |
| **Bandwidth (Writes)** | 400 writes/sec * 500 bytes | ~200 KB/s [Kilobytes/second] |
| **Bandwidth (Reads)** | 40K reads/sec * 500 bytes | ~20 MB/s [Megabytes/second] |
| **Cache (20% of daily reads)**| 40K * 3600 * 24 * 500 bytes * 0.2 | ~345 GB [Gigabytes] |

## Step 3: API Design

**1. Create Short URL**
`POST /api/v1/urls`
- **Request:** `{ "long_url": "https://...", "custom_alias": "my-link", "expires_in": 3600 }`
- **Response:** `{ "short_url": "http://tiny.url/my-link" }`

**2. Redirect**
`GET /{short_alias}`
- **Response:** HTTP 301 or 302 Redirect to `long_url`

> **Interview Tip**: Discuss 301 vs 302 redirects.
> - **301 (Permanent)**: Browser caches redirect. Reduces server load, but analytics are lost for subsequent clicks.
> - **302 (Temporary)**: Browser always hits server first. Good for tracking analytics.

## Step 4: High-Level Architecture Diagram

```ascii
                      +------------------+
                      |                  |
      Write Request   |   Rate Limiter   |
    +---------------->|                  |
    |                 +--------+---------+
    |                          |
+---+---+                +-----v-----+      +------------+      +---------------+
|       |                |           |      |            |      |               |
| User  |                |    API    +----->+  ID Gen    +----->+ Zookeeper     |
|       |                |  Gateway  |      |  Service   |      |               |
+---+---+                |           |      |            |      +---------------+
    |                    +-----+-----+      +------------+
    | Read Request             |
    +------------------------->|
                               |
                      +--------v---------+
                      |                  |
                      |   App Servers    |
                      |                  |
                      +-+-------+------+-+
                        |       |      |
                 +------+       |      +-------+
                 |              |              |
           +-----v----+   +-----v----+   +-----v----+
           |          |   |          |   |          |
           |  Redis   |   | NoSQL DB |   |   Kafka  |
           | (Cache)  |   | (Storage)|   |(Analytics|
           +----------+   +----------+   +----------+
```

## Step 5: Deep Dive: Shortening Algorithm

We need to map a long URL to a short string (e.g., 7 characters).
Characters allowed: [0-9, a-z, A-Z] = 10 + 26 + 26 = 62 characters (Base62).
Length 7 in Base62 = 62^7 = ~3.5 Trillion combinations (plenty for 30TB storage).

### Approach 1: MD5 Hash
Hash the long URL using MD5 (128-bit) and take the first 7 characters.
**Pros**: Distributed easily.
**Cons**: Hash collisions [Multiple inputs yielding same output]. Needs collision resolution (append sequence and retry).

### Approach 2: Counter-based Base62 (Chosen)
Use a centralized, distributed ID generator (e.g., using Zookeeper or Twitter Snowflake) to get a unique integer ID. Convert the ID to Base62.

**Base62 Conversion Pseudocode:**

```cpp
string base62_encode(long long id) {
    string chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    string short_url = "";
    
    if (id == 0) return "0";
    
    while (id > 0) {
        short_url = chars[id % 62] + short_url; // Prepend
        id /= 62;
    }
    
    return short_url;
}
```

## Step 6: Step-by-Step Data Flow

**Write Path (URL Creation):**
1. User sends long URL to API Gateway.
2. API Gateway forwards to Rate Limiter. If allowed, forwards to App Server.
3. App Server requests a new unique ID from the ID Gen Service (which uses Zookeeper to hand out ranges/blocks of IDs to avoid contention).
4. App Server converts the ID to Base62 to get the `short_alias`.
5. App Server saves `(short_alias, long_url)` in the NoSQL Database.
6. Returns `short_url` to the user.

**Read Path (Redirection):**
1. User accesses `http://tiny.url/xyz123`.
2. Request hits App Server.
3. App Server checks Redis Cache for `xyz123`.
4. If cache hit, redirect immediately (HTTP 302).
5. If cache miss, query NoSQL Database.
6. Store result in Redis (Cache-Aside pattern) and redirect.
7. Send a message to Kafka queue asynchronously for analytics tracking.

## Step 7: Database Choices and Trade-offs

| Database Type | Examples | Pros | Cons | Decision |
| :--- | :--- | :--- | :--- | :--- |
| **RDBMS** | MySQL, Postgres | ACID properties, joins | Harder to scale horizontally, schema changes | No, we don't need complex relations. |
| **NoSQL (Key-Value)** | DynamoDB, Cassandra | High write/read throughput, horizontal scaling | Eventual consistency, no joins | **Yes**. Perfect for point lookups `(short_alias -> long_url)`. |

## Step 8: Refinements and Edge Cases

- **Rate Limiting**: Prevent abuse (e.g., a single user generating millions of URLs). Apply rate limiting by IP and API key.
- **Cache Eviction**: Use LRU [Least Recently Used] policy. 80/20 rule: 20% of links generate 80% of traffic.
- **Custom Aliases**: For custom aliases, we bypass ID generation and try to insert directly. If a unique constraint violation occurs in the DB, return an error to the user.
- **URL Expiry**: Run a background cleanup cron job (or use TTL [Time To Live] in NoSQL databases like DynamoDB) to purge expired links.

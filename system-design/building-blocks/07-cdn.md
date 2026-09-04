# 07. Content Delivery Network (CDN)

A CDN is a geographically distributed network of proxy servers and data centers. Its primary goal is to provide high availability and high performance by serving content closer to end-users.

## How it Works
- **Origin Server:** The main server that holds the original source of truth.
- **Edge Servers / PoPs (Points of Presence):** CDN servers located across the globe.
- When a user requests an asset (e.g., an image), the DNS routes the request to the nearest Edge Server. If cached, it serves it immediately. If not, the Edge fetches it from the Origin.

## Push vs Pull CDN

### Pull CDN (Most Common)
- **Mechanism:** The CDN pulls content from the Origin only when a user requests it (cache miss).
- **Pros:** Low maintenance. Only actively requested data is stored on the CDN.
- **Cons:** First request is slow (cache miss penalty).
- **Use Case:** High traffic websites with frequently changing or vast amounts of content.

### Push CDN
- **Mechanism:** The application proactively pushes content to the CDN whenever it is created or updated.
- **Pros:** No cache miss penalty.
- **Cons:** High storage costs. Unused content takes up space.
- **Use Case:** Small sites with static content or specific high-priority assets (e.g., a new game patch release).

## Cache Invalidation
How to handle updated content?
1. **Time-To-Live (TTL):** Assets naturally expire after a set time.
2. **Object Versioning:** Change the filename (e.g., `app-v1.js` -> `app-v2.js`). This is the most reliable method.
3. **Purge API:** Manually force the CDN to invalidate a specific URL.

## Advanced Patterns

### CDN Failover / Multi-CDN
Using multiple CDN providers simultaneously (e.g., Cloudflare + Fastly) to prevent a single point of failure if one CDN goes down.

### Origin Shield
A caching layer sitting directly in front of the Origin Server. If multiple Edge servers miss the cache simultaneously, they hit the Origin Shield instead of overwhelming the Origin DB.

> **Interview Tip:** CDNs aren't just for images and videos! Modern CDNs (like Cloudflare) can cache API responses, handle TLS termination, mitigate DDoS attacks, and even run serverless edge functions.

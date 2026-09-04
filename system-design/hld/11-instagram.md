# Design Instagram (Photo Sharing App)

## 1. Clarify Requirements

### Functional Requirements (FR)
- **Upload**: Users can upload photos.
- **Feed**: Users see a chronological or algorithmically sorted news feed of people they follow.
- **Social Graph**: Users can follow/unfollow other users.
- **Engagement**: Likes, comments, and stories.

### Non-Functional Requirements (NFR)
- **Scale**: 500M [DAU (Daily Active Users)], highly read-heavy (100:1 Read-to-Write ratio).
- **Latency**: Feed generation in < 1s.
- **Reliability**: Uploaded images must not be lost.

> **Interview Tip**: In social networks, the "Feed Generation" and "Social Graph" (Fanout) are the most complex parts. Focus heavily on these.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation / Estimate |
|--------|------------------------|
| **Write/Upload QPS** | 50M uploads/day ≈ 600 QPS |
| **Read/Feed QPS** | 600 QPS * 100 = 60,000 QPS |
| **Storage (1 year)** | 50M/day * 365 * 1MB (avg image) = ~18 Petabytes/year |

---

## 3. High-Level Architecture

```text
                     +---------------+
                     |  Load Balancer|
                     +-------+-------+
                             |
         +-------------------+-------------------+
         |                                       |
 +-------v-------+                       +-------v-------+
 | Image Upload  |                       | Feed Service  |
 |   Service     |                       |               |
 +-------+-------+                       +-------+-------+
         |                                       |
 +-------v-------+                       +-------v-------+
 | Object Storage|                       | Feed Cache    |
 |    (S3)       |                       | (Redis)       |
 +-------+-------+                       +---------------+
         |                                       ^
 +-------v-------+                       +-------+-------+
 |      CDN      |<--- serve images ---- | Feed Worker   |
 +---------------+                       | (Fanout)      |
                                         +---------------+
```

---

## 4. API Design

```text
- upload_image(user_id: int, image_data: bytes, metadata: dict) -> string (image_url)
- get_feed(user_id: int, next_cursor: string) -> List<Post>
- follow_user(follower_id: int, followee_id: int) -> bool
```

---

## 5. Detailed Component Design

### 5.1 Image Processing & Storage
Images shouldn't be served at full size. 
- **Pipeline**: Upload -> Message Queue -> Workers resize, filter, and compress (generating thumbnails).
- **Storage**: Raw and processed images go to Object Storage (e.g., S3).
- **Delivery**: Images are served via CDN (Content Delivery Network) for low latency.

### 5.2 Social Graph & DB Choice
Follower relationships:
- **DB**: Graph DB (Neo4j) or RDBMS (PostgreSQL) `User_Follow(follower_id, followee_id)`.
- **Media Metadata**: NoSQL (Cassandra/DynamoDB) is great for `User_Posts` due to high write volumes.

### 5.3 Feed Generation (The Core Logic)
**Fanout Strategy:**
1.  **Fanout-on-Write (Push)**: When a user posts, push the post ID into the Redis feed list of *all* their followers. Fast reads, but terrible if Justin Bieber posts (millions of followers = long delay).
2.  **Fanout-on-Read (Pull)**: When a user loads the app, fetch the latest posts of all followees and merge them. Good for celebrities, but slow reads for normal users.
3.  **Hybrid Fanout**: 
    - Normal users: Push model (Fanout-on-write).
    - Celebrities: Pull model (Fanout-on-read). When a user requests their feed, merge their pre-computed push feed with live pulls from the celebrities they follow.

**Feed Merging Algorithm (K-Way Merge):**
```cpp
// Pseudocode for pulling celebrity feeds and merging
List<Post> generateFeed(int user_id) {
    List<Post> feed = getCachedFeed(user_id); // From Push model
    
    List<int> celebrity_followees = getCelebrities(user_id);
    for(int celeb_id : celebrity_followees) {
        List<Post> celeb_posts = fetchRecentPosts(celeb_id);
        feed = mergeSortedByTimestamp(feed, celeb_posts);
    }
    return feed.subList(0, 50); // Return top 50
}
```

---

## 6. Addressing Bottlenecks

### Caching
- **Redis** is used heavily to cache User feeds, Social Graphs (who follows whom), and User Sessions.
- LRU eviction policies for the cache.

### Explore / Discovery
- Relies on **Machine Learning (Collaborative Filtering)** running in offline Hadoop/Spark clusters. It calculates user similarity based on likes and tags, pre-computing "Explore" feeds into a key-value store.

---

## 7. Scaling and Resilience
- Data is sharded by `user_id` (e.g., Hash Sharding).
- Handle hot-spots (celebrities) by heavily caching their recent posts in a global replicated cache.

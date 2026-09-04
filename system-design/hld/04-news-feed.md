# News Feed System High Level Design (Twitter/Facebook)

## Step 1: Requirements Clarification
**Functional Requirements (FR):**
- **Publish**: Users can publish posts (text/images).
- **News Feed**: Users can view an aggregated feed of posts from people they follow, chronologically or ranked.
- **Follow/Unfollow**: Users can subscribe to others.

**Non-Functional Requirements (NFR):**
- **Latency**: Feed generation < 500ms.
- **Availability**: Highly available, eventual consistency is acceptable.
- **Scale**: 300M DAU [Daily Active Users].

## Step 2: Architecture Diagram

```ascii
   +---------+        +-------------+         +---------------+
   |         | Write  |             |         |               |
   | User A  +------->+ Post Service+-------->+ Post Database |
   |         |        |             |         |               |
   +---------+        +------+------+         +-------+-------+
                             |                        |
                             v                        v
                      +------+------+         +-------+-------+
                      |             |         |               |
                      | Fanout Svc  +-------->+ Feed Cache    |
                      |             |         | (Redis)       |
                      +------+------+         +-------+-------+
                             |                        ^
   +---------+        +------v------+                 |
   |         | Read   |             |                 |
   | User B  +------->+ Feed Service+-----------------+
   |         |        |             |
   +---------+        +-------------+
```

## Step 3: Deep Dive: Fanout Architectures

"Fanout" is the process of delivering a post to all followers.

### 1. Fanout-on-Write (Push Model)
When User A posts, the system pre-computes the feed for all followers and pushes the post ID to their Feed Caches.
- **Pros**: Read is O(1) - extremely fast feed generation.
- **Cons**: "Celebrity Problem" (Justin Bieber has 100M followers). Pushing to 100M caches takes minutes and thrashes the cache.

### 2. Fanout-on-Read (Pull Model)
When User B requests their feed, the system fetches all users they follow, pulls their recent posts, and merges them on the fly.
- **Pros**: No massive write spikes for celebrities. Less wasted storage.
- **Cons**: Feed generation is slow (high latency on read).

### 3. Hybrid Approach (Chosen)
- For **Normal Users**: Use Fanout-on-Write.
- For **Celebrities** (>10k followers): Do NOT push. Instead, use Fanout-on-Read.
- When User B loads their feed, we take their pre-computed push feed and merge it dynamically with recent posts from celebrities they follow.

**Decision Logic Pseudocode:**
```cpp
void publishPost(string userId, Post post) {
    savePostToDB(post);
    
    int followerCount = getFollowerCount(userId);
    if (followerCount < 10000) {
        // Fanout-on-write
        vector<string> followers = getFollowers(userId);
        for (string followerId : followers) {
            pushToRedisFeedCache(followerId, post.id);
        }
    } else {
        // Celebrity: Do nothing. Followers will pull on read.
        notifyFollowersAsync(userId, post.id); // Optional: just metadata
    }
}
```

## Step 4: Feed Generation Algorithm (Read Path)

1. Client requests Feed.
2. Feed Service fetches User B's `Feed Cache` (List of Post IDs).
3. Feed Service gets the list of Celebrities User B follows.
4. Feed Service fetches recent posts from those Celebrities.
5. Merge the standard feed and celebrity posts, sort them (chronologically or by ranking score).
6. "Hydrate" the Post IDs into full Post Objects (text, media) from the Post Cache/DB.
7. Return JSON to client.

## Step 5: Caching Strategy

The Feed Cache is critical. We use **Redis Sorted Sets (ZSET)**.
- **Key**: `feed:{user_id}`
- **Score**: Unix Timestamp (or Ranking Score)
- **Value**: `post_id`

We limit the cache to 500-1000 posts per user. Older posts are fetched from the database if the user scrolls deeply (pagination).

## Step 6: Ranking Formula (Briefly)

If not chronological, feeds use EdgeRank or ML models.
`Score = Affinity(User, Creator) * Weight(Post_Type) * Time_Decay`

> **Interview Tip**: Always discuss pagination for feeds. Use cursor-based pagination (passing a `next_cursor` token) rather than offset-based (`LIMIT 10 OFFSET 50`), because new posts arriving can shift the offsets and cause duplicate posts to appear on the client.

## Step 7: Database Choices

| Component | Database | Justification |
| :--- | :--- | :--- |
| Users & Relationships | Graph DB (Neo4j) or RDBMS | Highly connected data (Followers/Following). |
| Posts Data | Cassandra / DynamoDB | Massive write volume, schema-less scale. |
| Caching | Redis | In-memory, Sorted Sets for fast feed merging. |

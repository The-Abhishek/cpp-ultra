# Chat System High Level Design (WhatsApp/Messenger)

## Step 1: Requirements Clarification
**Functional Requirements (FR):**
- **1-to-1 Chat**: Real-time message delivery.
- **Group Chat**: Support up to 500 members per group.
- **Online Status**: Display when users are online/offline.
- **Read Receipts**: Sent, Delivered, Read indicators.
- **Media Sharing**: Image/Video attachments.

**Non-Functional Requirements (NFR):**
- **Latency**: <100ms [milliseconds] message delivery.
- **Scale**: 50M DAU [Daily Active Users], handling billions of messages daily.
- **Security**: End-to-End (E2E) Encryption.
- **Reliability**: No lost messages, strict ordering.

## Step 2: Back-of-the-Envelope Estimations

| Metric | Calculation | Result |
| :--- | :--- | :--- |
| **QPS (Messages)** | 50M DAU * 40 msgs/day / 86400 | ~23,000 QPS |
| **Storage / Day** | 50M * 40 msgs * 100 Bytes | ~200 GB/day |
| **Connections** | 50M DAU (assume 10% concurrent) | ~5 Million WebSockets |

## Step 3: High-Level Architecture Diagram

```ascii
                                +-------------------+
                                |                   |
                          +---->+ Presence Service  |
                          |     |                   |
                          |     +-------------------+
    +--------+      +-----+------+                    +---------------+
    |        |      |            |                    |               |
    | User A +----->+ Chat Server+------------------->+ Message Queue |
    |        | (WS) |            |                    |   (Kafka)     |
    +---+----+      +-----+------+                    +-------+-------+
        ^                 |                                   |
        |                 v                                   |
        |           +-----+------+                            |
        |           |            |                            |
        +-----------+ Redis Pub/Sub<--------------------------+
      Push Notif    | (Routing)  |
    (If Offline)    +------------+
```

## Step 4: Connection Management (WebSockets)

HTTP is stateless and client-initiated. For real-time chat, servers must push data to clients.
- We use **WebSockets (WS)** for persistent, bi-directional communication.
- A **Connection Manager / Chat Server** holds millions of open WS connections. It stores a mapping: `UserID -> ServerID` in a central Redis cache.

## Step 5: Step-by-Step Message Data Flow

**Scenario: User A sends a message to User B**
1. User A sends message over WebSocket to Chat Server 1.
2. Chat Server 1 generates a Message ID (using Snowflake for orderability) and stores it in the Database (or Message Queue).
3. Chat Server 1 queries Redis to find which server User B is connected to.
4. **If User B is Online:**
   - Redis says User B is on Chat Server 2.
   - Chat Server 1 publishes the message to Chat Server 2 via Redis Pub/Sub (or RPC).
   - Chat Server 2 pushes the message to User B via WebSocket.
5. **If User B is Offline:**
   - Push Notification Service is triggered (APNS for iOS, FCM for Android).
   - Message remains in the DB. When User B reconnects, they pull unread messages.

## Step 6: Deep Dive: Group Chat Fanout

Handling groups of 500 users:
- **Fanout on Write (Push)**: When A sends a group message, the server duplicates the message 499 times and pushes to each user's message queue.
- Fast delivery, but high write amplification. Given the max group size is 500, this is acceptable.

## Step 7: Presence Service (Online/Offline)

How do we know if a user is online?
- **Heartbeats**: Client sends a heartbeat ping every 5 seconds.
- **Pseudocode Logic**:
```cpp
void updatePresence(string userId) {
    long currentTime = getTime();
    redis.set("presence:" + userId, currentTime, EXPIRE=30s);
    publishStatusChange(userId, "ONLINE");
}

// Background worker
void checkOfflineUsers() {
    // If a heartbeat isn't received in 30s, the key expires.
    // Redis Keyspace Notifications can trigger an OFFLINE event.
}
```

## Step 8: Database and Storage Choices

| Data Type | DB Choice | Rationale |
| :--- | :--- | :--- |
| User Data | PostgreSQL | Relational, strict consistency, rarely changes. |
| Chat History | Cassandra / HBase | Key-Value/Wide-column. Optimised for heavy writes and sequential reads by timestamp. |
| Media Files | S3 / Blob Storage | Cost-effective binary storage. CDNs cache media globally. |

> **Interview Tip**: Always mention End-to-End Encryption (E2EE) for modern chat apps. E2EE means the server *cannot* read the message. The server merely routes encrypted byte payloads. Keys are generated and exchanged by clients using the Signal Protocol.

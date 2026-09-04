# Design Notification System (Push, SMS, Email)

## 1. Understand the Goal & Scope
**Problem:** Design a robust, scalable notification system capable of delivering messages to users via multiple channels (Push notifications, SMS, Email).

### Functional Requirements (FR)
- Send notifications via Push (iOS APNs, Android FCM), SMS (Twilio), and Email (SendGrid/SES).
- Template rendering (plug in user names, data into message templates).
- User preferences (opt-in/opt-out per channel, quiet hours).
- Deduplication (prevent sending the same notification multiple times).

### Non-Functional Requirements (NFR)
- **Scale:** 100M notifications/day.
- **Performance:** <30s delivery time for high-priority messages.
- **Reliability:** At-least-once delivery guarantee, graceful degradation if 3rd parties fail.
- **Spam Prevention:** Rate limiting per user.

---

## 2. Terminology & Core Concepts
- **[APNs]**: Apple Push Notification service.
- **[FCM]**: Firebase Cloud Messaging (Android).
- **[Idempotency Key]**: A unique identifier used to recognize retries of the same request, preventing duplicate processing.
- **[Exponential Backoff]**: A standard error-handling strategy for network applications where retries are spaced out by increasingly longer intervals.
- **[Worker Pool]**: A group of background threads/processes consuming tasks from a queue.

---

## 3. Back-of-the-Envelope Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| **Total Volume** | 100M / day | 100M/day |
| **Average QPS** | 100M / 86400 | ~1,150 requests/sec |
| **Peak QPS** | Assume 5x peak (e.g., breaking news) | ~5,750 requests/sec |
| **Database Storage** | 100M * 500 bytes (metadata) * 365 | ~18 TB/year |

> **Interview Tip:** While QPS isn't astronomically high, the complexity lies in managing 3rd-party integrations, retries, and ensuring reliability. Queues are mandatory here.

---

## 4. System Interface Design (APIs)

```rest
POST /v1/notifications
Request: { 
  user_id: "12345", 
  type: "PROMO_SALE", 
  channel: ["PUSH", "EMAIL"], 
  payload: { "discount": "20%" },
  idempotency_key: "abc-123"
}
Response: 202 Accepted { notification_id: "..." }
```
*(Note: 202 Accepted is used because processing is asynchronous).*

---

## 5. High-Level Design (Architecture Diagram)

```ascii
                      +-------------------+
                      | Internal Services | (Billing, Marketing, etc.)
                      +---------+---------+
                                |
                                v
                      +---------+---------+
                      |   API Gateway &   |
                      |   Rate Limiter    |
                      +---------+---------+
                                |
                                v
                      +---------+---------+      +----------------+
                      |   Notification    | ---> | User Profile / |
                      |   Dispatcher      |      | Preferences DB |
                      +----+---------+----+      +----------------+
                           |         |
          +----------------+         +----------------+
          |                                           |
          v                                           v
+---------+---------+                       +---------+---------+
| Message Queue     |                       | Message Queue     |
| (High Priority)   |                       | (Low Priority)    |
+---------+---------+                       +---------+---------+
          |                                           |
          v                                           v
+---------+---------+                       +---------+---------+
| Worker Pool (SMS) |                       | Worker Pool (Email|
+---------+---------+                       +---------+---------+
          |                                           |
          v                                           v
    [ Twilio API ]                             [ Amazon SES ]
```

---

## 6. Deep Dive

### 6.1 Notification Delivery Pipeline

1. **Validation & Preferences:** Dispatcher fetches user contact info (device tokens, phone numbers) and checks preferences. If user opted out of promos, drop the message.
2. **Template Rendering:** Merge payload (e.g., `discount: 20%`) with the template string.
3. **Queuing:** Push to channel-specific, priority-based queues (e.g., Kafka topics or RabbitMQ).
4. **Workers:** Channel-specific workers consume from queues and call 3rd-party APIs.

### 6.2 Retry Logic & Exponential Backoff

Third-party APIs (APNs, SES) fail. We need robust retry logic.

```cpp
// Pseudocode: Worker processing a message with retries
void processNotification(Message msg) {
    int maxRetries = 5;
    int attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            Response res = thirdPartyApi.send(msg);
            if (res.status == 200) {
                logSuccess(msg);
                return;
            } else if (isRetryableError(res.status)) { // e.g., 429 Too Many Requests, 503
                throw new RetryableException();
            } else { // 400 Bad Request
                logFailure(msg, "Permanent Error");
                return;
            }
        } catch (RetryableException e) {
            attempt++;
            int sleepMs = pow(2, attempt) * 1000; // Exponential backoff: 2s, 4s, 8s...
            sleep(sleepMs);
        }
    }
    
    // Failed after max retries, move to Dead Letter Queue (DLQ)
    moveToDLQ(msg); 
}
```

### 6.3 Deduplication (Idempotency)

Network timeouts can cause the caller to retry the `POST /v1/notifications` request. We use an Idempotency Key in Redis.

```cpp
bool isDuplicate(string idempotencyKey) {
    // SETNX (Set if Not eXists) in Redis with a TTL of 24 hours
    bool isNew = redis.setnx("idemp:" + idempotencyKey, "1", 24_HOURS);
    return !isNew;
}
```

### 6.4 Rate Limiting per User (Spam Prevention)

To avoid annoying users (e.g., max 3 promo push notifications per day).
- Use a Redis sorted set or token bucket per user.
- Key: `rate_limit:user123:promo_push`

---

## 7. Fault Tolerance and Scalability

- **Message Loss Prevention:** Use durable queues (like Kafka or RabbitMQ with persistence) so messages survive broker restarts.
- **Worker Scaling:** If the queue backs up, we can dynamically auto-scale the worker instances based on queue depth metrics.
- **Database Scalability:** The User Preferences DB is highly read-heavy (read on every notification). Use a distributed cache (Redis/Memcached) in front of the DB.

---

## 8. Summary & Interview Tips

> **Interview Tip:** Interviewers look for how you handle **3rd party failures**. Mentioning Dead Letter Queues (DLQ), Exponential Backoff, and Circuit Breakers shows seniority.
>
> Differentiating between **High Priority** (e.g., OTP codes, fraud alerts) and **Low Priority** (e.g., marketing blasts) queues is a key optimization signal.

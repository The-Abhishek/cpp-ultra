# System Design: Hotel Booking System (Airbnb / Booking.com)

## 1. Requirement Gathering

### Functional Requirements [FR]
- **Search & Filter**: Users can search for hotels by location, date, price, and amenities.
- **View Details**: View hotel/room specifics, availability calendar, and photos.
- **Book & Pay**: Users can reserve a room for specific dates and process payment.
- **Host Management**: Hosts can add/edit listings and block out dates.
- **Double-booking Prevention**: A room must never be double-booked for the same dates.

### Non-Functional Requirements [NFR]
- **Scale**: 10M Daily Active Users [DAU].
- **Latency**: Search and filtering must be < 1s.
- **Consistency**: Strong consistency for bookings (ACID properties). No double bookings allowed.
- **High Availability**: Search system must be highly available.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation | Estimate |
|--------|-------------|----------|
| **DAU** | - | 10 Million |
| **Search QPS** | 10M * 10 searches / 86400 | ~1,200 QPS |
| **Booking QPS** | 10M * 0.1 bookings / 86400 | ~12 QPS |
| **Listings Count** | - | ~5 Million |
| **Storage (Listings/Images)** | 5M * 5MB per listing | ~25 TB |

> **Interview Tip:** Notice the extreme read-to-write skew. Searching is read-heavy and requires low latency, while booking is write-heavy and requires strict consistency.

---

## 3. System Interface (APIs)

```rest
GET /v1/search?location={string}&checkin={date}&checkout={date}&guests={int}
Response: [{ "hotel_id": 123, "name": "Hilton", "price": 150 }]

POST /v1/bookings
Payload: { "user_id": 456, "room_id": 789, "checkin": "2023-12-01", "checkout": "2023-12-05" }
Response: { "booking_id": "abc", "status": "PENDING_PAYMENT" }
```

---

## 4. Data Model

- **Search Index (Elasticsearch)**: For fast spatial (GeoJSON) and text search.
- **Relational DB (PostgreSQL / MySQL)**: For transactional data (Bookings, Payments, Inventory).
- **Cache (Redis)**: To cache popular search results and hotel metadata.

**Core Tables (PostgreSQL)**
1. `rooms (id, hotel_id, room_type, price, version)`
2. `reservations (id, room_id, user_id, start_date, end_date, status)`

---

## 5. High-Level Design (Architecture)

```text
                               +------------------+
                               |                  |
                               |      Client      |
                               |                  |
                               +--------+---------+
                                        |
                                        v
                               +------------------+
                               |                  |
                               |   API Gateway    |
                               |                  |
                               +---+----+-----+---+
                                   |    |     |
          +------------------------+    |     +-------------------------+
          |                             |                               |
          v                             v                               v
+------------------+          +------------------+             +------------------+
|                  |          |                  |             |                  |
|  Search Service  |          | Booking Service  |             |  Host Service    |
|                  |          |                  |             |                  |
+--------+---------+          +--------+---------+             +--------+---------+
         |                             |                                |
         v                             v                                v
+------------------+          +------------------+             +------------------+
|                  |          |                  |             |                  |
|  Elasticsearch   |          |    PostgreSQL    | ----------> |   Kafka (Events) |
| (Geo + Filters)  |          |  (Transactions)  |             |                  |
|                  |          |                  |             +--------+---------+
+------------------+          +--------+---------+                      |
         ^                               |                              v
         |                               v                     +------------------+
         |                    +------------------+             |                  |
         +------------------- |   Data Sync      |             | Notification Svc |
          (Async Update)      |   (Debezium)     |             |                  |
                              +------------------+             +------------------+
```

---

## 6. Detailed Design (Deep Dive)

### Availability Check
When searching, we must return only rooms available for the requested dates.
**Date Range Overlap Logic**: Two date ranges `[A_start, A_end]` and `[B_start, B_end]` overlap if:
`A_start < B_end` AND `A_end > B_start`.

**SQL to check availability:**
```sql
SELECT room_id FROM rooms 
WHERE room_id NOT IN (
    SELECT room_id FROM reservations
    WHERE status IN ('CONFIRMED', 'PENDING')
    AND start_date < 'REQUESTED_CHECKOUT'
    AND end_date > 'REQUESTED_CHECKIN'
);
```

### Double-Booking Prevention (Concurrency Control)
Multiple users might try to book the last available room simultaneously. We need to prevent anomalies.

**Approach 1: Pessimistic Locking (SELECT FOR UPDATE)**
Locks the row until the transaction commits. Can cause bottlenecks.

**Approach 2: Optimistic Locking (Versioning) [Preferred]**
Add a `version` column to the inventory/room record.
**Core Logic: Booking Pseudocode**
```cpp
bool bookRoom(int roomId, Date checkin, Date checkout) {
    // 1. Check availability
    if (!isAvailable(roomId, checkin, checkout)) return false;

    // 2. Read current version
    int currentVersion = db.query("SELECT version FROM rooms WHERE id = ?", roomId);

    // 3. Attempt to insert reservation AND update version atomically
    db.beginTransaction();
    
    // The trick: Update only if the version hasn't changed
    int rowsAffected = db.execute(
        "UPDATE rooms SET version = version + 1 WHERE id = ? AND version = ?", 
        roomId, currentVersion
    );

    if (rowsAffected == 0) {
        db.rollback();
        return false; // Someone else modified the room (e.g., booked it)
    }

    db.execute(
        "INSERT INTO reservations (room_id, start_date, end_date) VALUES (?, ?, ?)", 
        roomId, checkin, checkout
    );
    
    db.commit();
    return true;
}
```
*Note: In reality, since bookings represent a date range, inventory is often modeled as `room_daily_availability` (room_id, date, available_count) to apply the lock per date.*

### Search & Filtering with Elasticsearch
- Sync PostgreSQL changes to Elasticsearch using **CDC (Change Data Capture)** like Debezium.
- ES handles `geo_distance` queries to find hotels within a radius and applies filters (price, WiFi).

---

## 7. Bottlenecks & Trade-offs

- **Search vs Booking Consistency**: Elasticsearch might lag behind PostgreSQL by a few milliseconds. A user might see a room as available in search, but fail to book it (Optimistic lock failure). This is an acceptable UX trade-off for high-performance search.
- **Payment Failures**: When a user clicks "Book", the room status is set to `PENDING` and locked for 10 minutes (TTL). If payment fails or times out, a background worker (or Redis TTL event) releases the lock.

---

## 8. Summary
- **Search**: Elasticsearch for fast geo-spatial queries and filtering.
- **Booking**: PostgreSQL for ACID transactions.
- **Concurrency**: Optimistic locking (versioning) to prevent double-booking without severe performance hits.
- **Data Sync**: CDC (Kafka/Debezium) keeps the search index updated when reservations are made.

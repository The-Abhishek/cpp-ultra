# System Design: Ad Click Event Aggregator

## 1. Requirement Gathering

### Functional Requirements [FR]
- **Ingestion**: Track ad impressions and clicks from various platforms.
- **Aggregation**: Aggregate metrics (click count) per Ad ID over time windows (e.g., 1-minute, 1-hour).
- **Dashboard**: Provide a real-time dashboard for advertisers to view metrics.
- **Reconciliation**: Ensure absolute accuracy (billing depends on this).

### Non-Functional Requirements [NFR]
- **Scale**: 10 Billion events per day.
- **Latency**: Real-time dashboard delay < 1 minute.
- **Accuracy**: Exactly-once processing semantics for counting.
- **High Throughput**: Must handle massive traffic spikes.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation | Estimate |
|--------|-------------|----------|
| **Daily Events** | - | 10 Billion |
| **QPS (Average)** | 10B / 86400 seconds | ~115,000 QPS |
| **Peak QPS** | 115k * 3 (spike factor) | ~350,000 QPS |
| **Event Size** | AdID (8B), UserID (8B), TS (8B) | ~100 Bytes |
| **Daily Storage** | 10B * 100B | ~1 TB / day |

> **Interview Tip:** For ad/billing systems, emphasize **exactly-once processing** and **idempotency**. Dropping events loses money; double-counting overcharges clients.

---

## 3. System Interface (APIs)

```rest
POST /v1/events/click
Payload: { "ad_id": "ad_123", "user_id": "u_456", "timestamp": "2023-10-10T10:00:00Z", "ip": "..." }

GET /v1/analytics/clicks?ad_id=ad_123&start=...&end=...&window=1h
Response: [ { "time": "10:00", "clicks": 5000 }, { "time": "11:00", "clicks": 6200 } ]
```

---

## 4. Data Model

- **Raw Events (Message Queue)**: Kafka topics partitioned by `ad_id`.
- **Aggregated Data (OLAP DB)**: Apache Pinot, Apache Druid, or ClickHouse for fast analytical queries.
  `Table: ad_stats (ad_id, time_bucket, click_count)`

---

## 5. High-Level Design (Architecture)

We will use the **Lambda Architecture** concept (Speed Layer for real-time + Batch Layer for accurate reconciliation).

```text
                                 +----------------+
                                 |                |
                                 |    Ad Client   |
                                 |                |
                                 +-------+--------+
                                         |
                                         v
                                 +----------------+
                                 |                |
                                 |  API Gateway   |
                                 |                |
                                 +-------+--------+
                                         |
                                         v
                                 +----------------+
                                 |                |
                                 |  Apache Kafka  | (Raw Event Buffer)
                                 |                |
                                 +---+--------+---+
                                     |        |
         +---------------------------+        +---------------------------+
         | (Speed Layer)                                                  | (Batch Layer)
         v                                                                v
+------------------+                                            +------------------+
|                  |                                            |                  |
| Stream Processor | (Apache Flink)                             |    Object Store  | (HDFS / S3)
| (Aggregates 1m)  |                                            |    (Raw Logs)    |
|                  |                                            |                  |
+--------+---------+                                            +--------+---------+
         |                                                               |
         |                                                               | MapReduce / Spark Batch
         v                                                               v (Runs Nightly)
+------------------+                                            +------------------+
|                  |                                            |                  |
|     OLAP DB      | <----------------------------------------- |  Reconciliation  |
|  (ClickHouse)    | (Overwrite with accurate batch data)       |       Job        |
|                  |                                            |                  |
+--------+---------+                                            +------------------+
         |
         v
+------------------+
|                  |
|    Dashboard     |
|                  |
+------------------+
```

---

## 6. Detailed Design (Deep Dive)

### Deduplication
Clients might retry sending the same click due to network issues.
- Generate a unique `event_id` on the client.
- **Bloom Filter**: In the Stream Processor, use a Bloom Filter (e.g., Guava or Redis) keyed by `event_id` to quickly reject duplicates.
- **Fallback**: Maintain a bounded cache (or RocksDB) of recent `event_id`s for deterministic checks.

### MapReduce Aggregation (Time Windows)
Stream processors use windows to group events.
- **Tumbling Window**: Fixed, non-overlapping windows (e.g., 00:00-00:01, 00:01-00:02). Good for simple counts.
- **Sliding Window**: Overlapping windows (e.g., last 5 mins, updated every 1 min).

**Core Logic: MapReduce Aggregation Pseudocode (Flink Style)**
```cpp
// Map: Extract (AdID, Window, Count=1)
pair<Key, int> mapEvent(Event e) {
    long windowStart = floor(e.timestamp / 60000) * 60000; // 1-min bucket
    Key k = {e.ad_id, windowStart};
    return {k, 1};
}

// Reduce: Sum the counts for the key
int reduce(Key k, vector<int> counts) {
    int total = 0;
    for (int c : counts) total += c;
    return total;
}

// Flink processes streams in stateful operators
void processStream(Stream<Event> stream) {
    stream
        .filter(e -> !isDuplicate(e.id))
        .map(mapEvent)
        .keyBy(pair.Key)
        .window(TumblingEventTimeWindows.of(Time.minutes(1)))
        .reduce(reduce)
        .sink(ClickHouseDB);
}
```

### Watermarking and Late Events
Network delays cause events to arrive out-of-order.
- **Watermark**: A threshold indicating that all events up to a certain timestamp have been received.
- If an event arrives *after* the watermark has passed its window, it is considered a "late event".
- Strategy: Keep the window state open for a "grace period" (e.g., 5 minutes). Update the OLAP DB if late events arrive.

### Exactly-Once Semantics
- Kafka + Flink achieve exactly-once using **Distributed Checkpointing** (Chandy-Lamport algorithm).
- State (current aggregations, deduplication cache) is periodically saved to durable storage. Upon failure, Flink restarts from the last checkpoint, and Kafka offsets are reset.

---

## 7. Bottlenecks & Trade-offs

- **Hot Partitions**: A very popular ad (e.g., Superbowl ad) will cause all events for that `ad_id` to hit a single Kafka partition/Flink node.
  - **Mitigation (Salted Key)**: Append a random number to the key (`ad_id_1`, `ad_id_2`) during the `Map` phase to distribute the load, then do a second `Reduce` phase to sum them up.
- **Speed vs Accuracy**: Stream processing is fast but can suffer from state loss or extreme latencies. The Batch Layer (Spark on S3) runs nightly over immutable raw logs to overwrite the real-time aggregations, providing 100% accuracy for billing.

---

## 8. Summary
- **Ingestion**: Kafka acts as a durable, high-throughput buffer.
- **Real-time**: Flink processes data in tumbling windows, handles late events via watermarks, and guarantees exactly-once via checkpoints.
- **Storage**: ClickHouse/Druid provides sub-second query latency for the advertiser dashboard.
- **Reconciliation**: Lambda architecture ensures billing accuracy by running batch jobs to correct any real-time anomalies.

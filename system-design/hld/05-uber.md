# Design Uber/Lyft Ride-Sharing

## 1. Understand the Goal & Scope
**Problem:** Design a ride-sharing service like Uber or Lyft where riders can request rides, and drivers can accept them and track each other in real-time.

### Functional Requirements (FR)
- Request a ride (rider provides pickup and drop-off locations).
- Match driver (system matches a rider with a nearby driver).
- Real-time tracking (rider sees driver approaching).
- Dynamic pricing (Surge pricing based on supply/demand).
- Payment processing.
- Rating system for drivers and riders.

### Non-Functional Requirements (NFR)
- **Scale:** 20M [DAU] (Daily Active Users).
- **Latency:** Driver matching < 5s, Location updates < 1s latency.
- **Availability:** 99.99% uptime.
- **Consistency:** Eventual consistency for location, Strong consistency for payments and rides.

---

## 2. Terminology & Core Concepts
- **[DAU]**: Daily Active Users.
- **[QPS]**: Queries Per Second.
- **[Geohash]**: A spatial data structure that subdivides space into buckets of grid shape (using Base32 string representation).
- **[QuadTree]**: A tree data structure where each internal node has exactly four children, used to partition a two-dimensional space.
- **[S2 Cells]**: A mathematical curve (Hilbert curve) wrapping a sphere, used by Google Maps for spatial indexing.
- **[ETA]**: Estimated Time of Arrival.
- **[Pub/Sub]**: Publish-Subscribe messaging pattern.
- **[Surge Pricing]**: Dynamic pricing multiplier applied when demand > supply in a specific area.

---

## 3. Back-of-the-Envelope Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| **Active Drivers** | Assumption: ~3M active at peak times. | 3M drivers |
| **Location Updates QPS** | 3M drivers * 1 update / 4s | ~750,000 updates/sec |
| **Ride Requests** | 10M rides/day -> 10M / 86400 | ~115 requests/sec (Peak ~1000) |
| **Bandwidth (Location)** | 750K * 100 bytes/update | ~75 MB/s |
| **Storage (Trips/Year)** | 10M rides/day * 1KB * 365 | ~3.6 TB/year |

> **Interview Tip:** Emphasize that location updates are incredibly high-throughput and require a highly scalable ingest pipeline, typically using Kafka or Redis.

---

## 4. System Interface Design (APIs)

```rest
POST /v1/rides/request
Request: { rider_id, pickup_lat, pickup_long, dropoff_lat, dropoff_long, ride_type }
Response: { trip_id, status: "SEARCHING", estimated_price }

POST /v1/locations/driver
Request: { driver_id, lat, long, status }
Response: 200 OK

GET /v1/rides/{trip_id}/status
Response: { status, driver_id, driver_location: {lat, long}, eta }
```

---

## 5. High-Level Design (Architecture Diagram)

```ascii
                      +-------------------+
                      |   Mobile Client   |
                      |  (Rider/Driver)   |
                      +---------+---------+
                                |
                                v
                      +---------+---------+
                      |   Load Balancer   |
                      |    / API Gateway  |
                      +---------+---------+
                                |
        +-----------------------+-----------------------+
        |                       |                       |
        v                       v                       v
+---------------+       +---------------+       +---------------+
|   Location    |       |   Matching    |       |    Trip /     |
|   Service     |       |   Service     |       |   Pricing     |
+-------+-------+       +-------+-------+       +-------+-------+
        |                       |                       |
        v                       v                       v
+-------+-------+       +-------+-------+       +---------------+
|     Redis     |       |     Kafka     |       |   PostgreSQL  |
|  (Geospatial) |       | (Event Stream)|       | (Trips/Users) |
+---------------+       +---------------+       +---------------+
        |                                               |
        v                                               v
+---------------+                               +---------------+
|  Cassandra    |                               |    Payment    |
| (Loc History) |                               |    Service    |
+---------------+                               +---------------+
```

---

## 6. Deep Dive

### 6.1 Geospatial Indexing: Geohash vs QuadTree vs S2 Cells

- **Geohash**: Encodes 2D coordinates into a 1D string. Longer string = more precision. (e.g., `9q8yy`). Finding nearby drivers means querying drivers with matching Geohash prefixes. Easy to implement in Redis.
- **QuadTree**: In-memory tree. Good for dynamic density (e.g., dense in NYC, sparse in Wyoming). Harder to maintain in a distributed environment because modifying the tree requires locking.
- **S2 Cells**: Maps a sphere to a 1D index using Hilbert Curves. Highly accurate for distances and areas. Used by Uber (H3 is their specific hexagonal version).

**Design Decision:** We will use a highly optimized **Redis with Geospatial indexes (Geohash under the hood)** or an H3 grid system.

#### Geohash Proximity Search Algorithm

```cpp
// Pseudocode: Finding nearby drivers using Geohash
vector<Driver> findNearbyDrivers(double lat, double lon, int radius) {
    // 1. Calculate Geohash for the given location at required precision (e.g., length 5)
    string baseHash = encodeGeohash(lat, lon, 5); 
    
    // 2. Get the 8 neighboring geohashes
    vector<string> neighbors = getGeohashNeighbors(baseHash);
    neighbors.push_back(baseHash);
    
    // 3. Query Redis for drivers in these geohashes
    vector<Driver> nearbyDrivers;
    for (string hash : neighbors) {
        vector<Driver> driversInHash = redis.geoRadius(hash, lat, lon, radius);
        nearbyDrivers.insert(nearbyDrivers.end(), driversInHash.begin(), driversInHash.end());
    }
    
    return nearbyDrivers;
}
```

### 6.2 Driver Matching Algorithm

The matching service scores nearby drivers based on multiple factors, not just raw distance.

```cpp
// Pseudocode: Driver Scoring
double calculateDriverScore(Driver d, Rider r) {
    double distanceScore = 1.0 / calculateRouteDistance(d.location, r.pickup);
    double ratingScore = d.rating / 5.0;
    double acceptanceScore = d.acceptance_rate;
    
    // Weights determined by ML models
    return (W1 * distanceScore) + (W2 * ratingScore) + (W3 * acceptanceScore);
}

void matchRider(Rider r) {
    auto drivers = findNearbyDrivers(r.pickup.lat, r.pickup.lon, 5_km);
    sort(drivers.begin(), drivers.end(), [&r](Driver a, Driver b) {
        return calculateDriverScore(a, r) > calculateDriverScore(b, r);
    });
    
    for (Driver d : drivers) {
        bool accepted = sendRideRequestToDriver(d, r, timeout=10s);
        if (accepted) {
            confirmMatch(r, d);
            return;
        }
    }
    notifyRiderNoDrivers(r);
}
```

### 6.3 Surge Pricing

Surge pricing is calculated per Geohash zone.
`Surge Multiplier = max(1.0, Demand (requests in last 5m) / Supply (active drivers in zone))`
This is computed asynchronously by a streaming engine like Flink processing the Kafka events and stored in Redis for fast lookup by the Pricing Service.

### 6.4 Real-time Tracking

**Architecture:** WebSockets are used for bidirectional, low-latency communication.
1. Driver sends location updates to Location Service (HTTP or WebSocket).
2. Location Service publishes update to a Pub/Sub topic (e.g., Redis Pub/Sub) specific to `trip_id`.
3. Rider's active WebSocket connection is subscribed to this `trip_id` topic.
4. Rider receives updates in real-time.

---

## 7. Fault Tolerance and Scalability

- **High Location Ingest:** 750K updates/s is massive. We buffer updates in memory (or lightweight Kafka topic) and batch write to Redis/Cassandra.
- **Node Failure in Location Store:** If a Redis node holding a geohash fails, a replica takes over. Transient driver locations can be re-populated within 4 seconds by the next driver ping.
- **Idempotency:** Ride requests and payments must use an `idempotency_key` to prevent double charging on retries.

---

## 8. Summary & Interview Tips

> **Interview Tip:** Emphasize that location data is ephemeral. Redis is great for the current location (fast retrieval), but historical data for analytics/disputes should go to an append-only store like Cassandra or HDFS via a Kafka stream.
>
> Don't spend too much time on User/Auth services. Focus intensely on the **Geospatial indexing** and **Matching** logic.

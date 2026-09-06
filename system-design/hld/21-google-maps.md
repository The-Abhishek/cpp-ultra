# System Design: Google Maps

## 1. Requirement Gathering

### Functional Requirements [FR]
- **Search places**: Users can search for specific places or Points of Interest [POI] (e.g., restaurants, gas stations).
- **Navigation & Directions**: Users can get driving, walking, or biking directions.
- **ETA Calculation**: Provide Estimated Time of Arrival [ETA] based on current traffic and historical data.
- **Real-time Traffic**: Show traffic overlays on the map.
- **Map Rendering**: Smooth rendering of map tiles at various zoom levels.

### Non-Functional Requirements [NFR]
- **Scale**: 1 Billion+ Daily Active Users [DAU].
- **Latency**: Route calculation should take < 2 seconds.
- **Real-time**: Traffic updates should reflect in near real-time.
- **High Availability**: The system must remain available for navigation even under heavy load.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation | Estimate |
|--------|-------------|----------|
| **DAU** | - | 1 Billion |
| **QPS [Queries Per Second]** | 1B users * 5 queries / 100,000 secs | ~50,000 QPS |
| **Peak QPS** | 50,000 * 5 | ~250,000 QPS |
| **Tile Storage** | 20 zoom levels, exponential tiles | ~100s of TBs |
| **Bandwidth** | 50,000 QPS * 100KB per response | ~5 GB/s |

> **Interview Tip:** Emphasize that map data (tiles) is largely static and highly cacheable on CDNs, while traffic and ETA are dynamic and require real-time processing.

---

## 3. System Interface (APIs)

```rest
GET /v1/maps/route?origin={lat,long}&destination={lat,long}&mode={driving|walking}
Response: 
{
  "eta_seconds": 1200,
  "distance_meters": 15000,
  "route_geometry": "polyline_string",
  "steps": [...]
}

GET /v1/places/search?query={string}&location={lat,long}&radius={meters}
Response: [{ "place_id": "xyz", "name": "Starbucks", "lat": 40.71, "long": -74.00 }]

GET /v1/traffic/updates?tile_id={string}
Response: { "segments": [{"segment_id": "s1", "speed_kmh": 20}] }
```

---

## 4. Data Model

We need specialized databases for different access patterns.

- **Map Tiles (Object Storage / CDN)**: S3 for raw tiles, CDN for edge caching.
- **Road Graph (Graph DB)**: Nodes are intersections, edges are road segments. Stored in Neo4j or customized spatial DB.
- **POIs (Spatial Index)**: QuadTree or Geohash indexed in Elasticsearch or PostgreSQL with PostGIS.
- **Traffic Data (Time-series / KV Store)**: Redis or Cassandra for fast read/write of current speeds.

---

## 5. High-Level Design (Architecture)

```text
                                +-------------------+
                                |                   |
                                |       Client      |
                                | (Mobile/Web App)  |
                                |                   |
                                +---------+---------+
                                          |
                                          v
                                +-------------------+
                                |                   |
                                |    API Gateway    |
                                |  (Load Balancer)  |
                                |                   |
                                +---+----+----+-----+
                                    |    |    |
          +-------------------------+    |    +--------------------------+
          |                              |                               |
          v                              v                               v
+-------------------+          +-------------------+           +-------------------+
|                   |          |                   |           |                   |
|  Search Service   |          |  Routing Service  |           |  Map Tile Server  |
|                   |          |                   |           |                   |
+---------+---------+          +---------+---------+           +---------+---------+
          |                              |                               |
          v                              v                               v
+-------------------+          +-------------------+           +-------------------+
|                   |          |                   |           |                   |
|   Elasticsearch   |          |    Graph DB       |           |   CDN / S3        |
|    (QuadTree)     |          | (Road Segments)   |           | (Pre-rendered)    |
|                   |          |                   |           |                   |
+-------------------+          +---------+---------+           +-------------------+
                                         |
                                         v
                               +-------------------+
                               |                   |
                               | Traffic Service   | <----- Real-time location 
                               |   (ETA Engine)    |        pings from users
                               +---------+---------+
                                         |
                                         v
                               +-------------------+
                               |                   |
                               |  Cassandra/Redis  |
                               |                   |
                               +-------------------+
```

---

## 6. Detailed Design (Deep Dive)

### Map Tiling & QuadTree
Maps are broken into smaller tiles at different zoom levels.
- **QuadTree**: A tree data structure where each node has exactly four children. Useful for spatial indexing.
- At Zoom level 0, the world is 1 tile.
- Zoom level 1: 4 tiles.
- Zoom level N: $4^N$ tiles.
- The client requests tiles based on its viewport bounding box and zoom level.

### Routing & A* Algorithm
Finding the shortest path on a road network (graph).
- **Dijkstra's Algorithm**: Explores equally in all directions. Slow for global maps.
- **A* Algorithm [A-star]**: Uses a heuristic (straight-line distance to destination) to guide the search towards the goal.

**Core Logic: A* Routing Pseudocode**
```cpp
// Nodes represent intersections, edges represent road segments.
// gScore[n]: Cost from start to n.
// fScore[n]: gScore[n] + heuristic(n, goal).

vector<Node> calculateRoute(Node start, Node goal) {
    priority_queue<pair<double, Node>, vector<pair<double, Node>>, greater<>> openSet;
    unordered_map<Node, Node> cameFrom;
    unordered_map<Node, double> gScore, fScore;

    gScore[start] = 0;
    fScore[start] = heuristic(start, goal); // Straight-line distance
    openSet.push({fScore[start], start});

    while (!openSet.empty()) {
        Node current = openSet.top().second;
        if (current == goal) return reconstructPath(cameFrom, current);
        openSet.pop();

        for (Edge edge : current.neighbors) {
            Node neighbor = edge.destination;
            // Get dynamic weight (e.g., travel time based on traffic)
            double weight = getTrafficAdjustedWeight(edge);
            double tentative_gScore = gScore[current] + weight;

            if (tentative_gScore < gScore[neighbor]) {
                cameFrom[neighbor] = current;
                gScore[neighbor] = tentative_gScore;
                fScore[neighbor] = gScore[neighbor] + heuristic(neighbor, goal);
                openSet.push({fScore[neighbor], neighbor});
            }
        }
    }
    return {}; // No path found
}
```

### ETA Prediction
- **ETA Engine**: Computes the sum of travel times for all segments in the route.
- `Segment Travel Time = Segment Length / Average Speed`.
- Average speed is derived from:
  1. Historical patterns (e.g., Friday 5 PM is slow).
  2. Real-time traffic (pings from current users on that road).

### Traffic Ingestion Pipeline
Millions of users constantly send location pings (lat, long, speed, timestamp).
- **Kafka**: Acts as a buffer for incoming pings.
- **Flink/Spark Streaming**: Aggregates pings by road segment every minute to calculate average speed.
- Updates are written to Redis for fast reads by the Routing Service.

---

## 7. Bottlenecks & Trade-offs

- **Compute Heavy Routing**: Running A* on massive graphs for every request is expensive.
  - **Optimization**: Use **Contraction Hierarchies (CH)**. Pre-compute shortcuts between important nodes to drastically reduce search space for long routes.
- **Real-time Traffic Accuracy**: If few users are on a road, traffic data is noisy.
  - **Trade-off**: Fall back to historical data for sparse roads.
- **CAP Theorem**: For location pings, Availability is prioritized over Consistency. Losing a few location pings is acceptable.

---

## 8. Summary
- **Map Tiling**: Pre-rendered, CDN-cached for fast map loading.
- **Routing**: A* / Contraction Hierarchies with dynamically weighted edges for traffic.
- **Search**: QuadTree / Geohash for efficient POI spatial queries.
- **Traffic**: Streaming pipeline (Kafka + Flink) to ingest user location pings and update segment speeds.

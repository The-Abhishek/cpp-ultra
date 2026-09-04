# Design Netflix/YouTube Video Streaming

## 1. Understand the Goal & Scope
**Problem:** Design a global video streaming platform capable of uploading, storing, and serving high-quality video content smoothly to millions of concurrent users across varying network conditions.

### Functional Requirements (FR)
- Upload video (for content creators/Netflix ingestion).
- Stream video (smooth playback, no buffering).
- Search and Discover content.
- Video recommendations.
- Subtitle support.

### Non-Functional Requirements (NFR)
- **Scale:** 200M [DAU] (Daily Active Users), 1M+ videos.
- **Performance:** Buffer-free streaming, low latency start time.
- **Availability:** 99.99% uptime.
- **Global Reach:** Fast access from anywhere in the world.

---

## 2. Terminology & Core Concepts
- **[ABR]**: Adaptive Bitrate Streaming. Adjusts video quality dynamically based on client bandwidth.
- **[CDN]**: Content Delivery Network. Geographically distributed servers caching video chunks.
- **[DAG]**: Directed Acyclic Graph. Used to model the video transcoding workflow.
- **[HLS / DASH]**: HTTP Live Streaming / Dynamic Adaptive Streaming over HTTP. Standard protocols for chunked video delivery.
- **[Transcoding]**: Converting a video file from one format/codec/resolution to multiple others.
- **[Collaborative Filtering]**: Recommendation algorithm based on user interactions (e.g., "Users who watched X also watched Y").

---

## 3. Back-of-the-Envelope Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| **Daily Active Users** | Given | 200M |
| **Concurrent Streams** | Assume 10% peak concurrency | 20M concurrent |
| **Streaming Bandwidth** | 20M streams * 3 Mbps (average HD) | ~60 Tbps (handled by CDNs) |
| **Storage per Video** | 1 hr video * 5 resolutions * 2 codecs = ~10 GB | 10 GB |
| **Total Storage (1M vids)**| 1,000,000 * 10 GB | ~10 Petabytes (PB) |

> **Interview Tip:** High streaming bandwidth implies you CANNOT serve video from your central application servers. A multi-tiered CDN is absolutely mandatory.

---

## 4. System Interface Design (APIs)

```rest
POST /v1/videos/upload
Request: { video_file, metadata: {title, description, tags} }
Response: { video_id, status: "PROCESSING" }

GET /v1/videos/{video_id}/manifest
Response: URL to the HLS/DASH manifest (.m3u8 or .mpd)

GET /v1/search?q={query}
Response: [ {video_id, title, thumbnail_url} ]
```

---

## 5. High-Level Design (Architecture Diagram)

```ascii
                                +-------------------+
                                |   Mobile/Web TV   |
                                +---------+---------+
                                          |
                        +-----------------+-----------------+
                        |                 |                 |
                        v                 v                 v
            +-------------------+ +---------------+ +---------------+
            |  Upload Service   | |  API Gateway  | |      CDN      |
            +---------+---------+ +-------+-------+ +-------+-------+
                      |                   |                 ^
                      v                   v                 |
            +---------+---------+ +---------------+ +-------+-------+
            |  Transcoding DAG  | |   Metadata    | |   Blob Store  |
            |     Workers       | |   Service     | |   (S3 / GCS)  |
            +---------+---------+ +-------+-------+ +---------------+
                      |                   |
                      v                   v
            +---------+---------+ +---------------+
            |   Message Queue   | |  DB (SQL/NoSQL)|
            |  (Kafka/RabbitMQ) | |   Metadata    |
            +-------------------+ +---------------+
```

---

## 6. Deep Dive

### 6.1 Video Transcoding Pipeline (DAG Workflow)

When a video is uploaded, it must be converted into multiple formats (e.g., 1080p, 720p, 480p) and split into short chunks (e.g., 4-second segments).

- **Architecture:** We use a DAG-based workflow engine (like Netflix's Conductor or Apache Airflow).
- **Steps:**
  1. Inspect video metadata (resolution, frame rate).
  2. Split video into chunks.
  3. Parallel workers transcode chunks into multiple resolutions (1080p, 720p, etc.).
  4. Parallel workers extract audio and transcode.
  5. Generate manifests (HLS/DASH).

```ascii
Upload -> [Splitter] --> [Worker: 1080p H.264] --> [Assembler] -> Blob Store
                     --> [Worker: 720p H.264]  -->
                     --> [Worker: Audio AAC]   -->
```

### 6.2 Adaptive Bitrate Streaming (ABR)

**Concept:** The client player downloads a "manifest" file containing URLs for different quality levels of the video chunks. The client monitors network bandwidth and CPU load and dynamically switches to a higher or lower quality chunk for the *next* segment to prevent buffering.

```cpp
// Pseudocode: Client-side ABR Logic
void playVideo(string manifestUrl) {
    Manifest manifest = download(manifestUrl);
    int currentBitrate = estimateBandwidth(); 
    
    for (int i = 0; i < manifest.totalChunks; i++) {
        // Pick the highest quality stream that fits in our current bandwidth
        Stream quality = manifest.getBestStreamForBandwidth(currentBitrate);
        Chunk c = download(quality.getChunkUrl(i));
        
        player.play(c);
        
        // Continuously update bandwidth estimation
        currentBitrate = calculateRecentDownloadSpeed(); 
    }
}
```

### 6.3 Content Delivery Network (CDN) Architecture

Netflix uses Open Connect Appliances (OCAs) deployed directly inside ISPs (Internet Service Providers) worldwide.
- **Multi-tier caching:**
  - **Tier 1 (ISP level):** Caches the most popular content locally. Zero transit costs.
  - **Tier 2 (Regional):** Caches less popular, long-tail content.
  - **Tier 3 (Origin / Cloud S3):** Source of truth.

When a user requests a video, the backend determines the optimal CDN node based on geolocation and ISP peering arrangements.

### 6.4 Recommendation Engine

Uses **Collaborative Filtering** combined with Deep Learning.
- Offline pipeline: Analyzes watch history logs (Hadoop/Spark) to compute item-item similarity matrices.
- Online serving: When a user logs in, the service fetches their profile, looks up similar items in the pre-computed matrix (stored in Redis or Cassandra), and serves recommendations in < 100ms.

---

## 7. Fault Tolerance and Scalability

- **Transcoding Failures:** If a worker node crashes mid-transcode, the message remains in the message queue (Kafka/RabbitMQ) and is picked up by another worker.
- **CDN Failures:** If an edge server goes down, the client player automatically fails over to the next optimal CDN node URL provided in the routing response.
- **Database:** Metadata (titles, users) is read-heavy. Use PostgreSQL with Read Replicas, and cache aggressively with Redis.

---

## 8. Summary & Interview Tips

> **Interview Tip:** A common pitfall is trying to stream video directly from an application server or database. *Always* use Blob Storage -> CDN -> Client for video blocks.
>
> Emphasize **chunking**. Videos aren't streamed as one massive file; they are broken into 4-6 second chunks. This makes ABR possible and CDN caching efficient.

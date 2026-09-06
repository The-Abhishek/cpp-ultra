# Design Dropbox / Google Drive (File Sync)

## 1. Clarify Requirements

### Functional Requirements (FR)
- **Upload/Download**: Users can store and retrieve files.
- **Sync**: Files automatically sync across multiple devices (Laptop, Phone, Web).
- **Versioning**: Track file history.
- **Offline Support**: Edits made offline sync when reconnected.

### Non-Functional Requirements (NFR)
- **Scale**: 500M users, heavily storage-bound.
- **File Size**: Large files supported (up to 50GB).
- **Efficiency**: Instant sync, minimizing network bandwidth.
- **ACID**: High data consistency required (No silent file corruption).

> **Interview Tip**: The magic of Dropbox is *File Chunking* and *Deduplication*. You cannot send a 10GB file over the wire every time 1 byte changes.

---

## 2. Back-of-the-Envelope Estimation

| Metric | Calculation / Estimate |
|--------|------------------------|
| **Storage** | 500M users * 50GB per user = 25 Exabytes (Massive!) |
| **Deduplication Savings** | Usually saves 40-50% of storage |
| **QPS** | 500M * 0.1 ops/day = ~500 QPS (Storage bandwidth is the bottleneck, not request QPS) |

---

## 3. High-Level Architecture

```text
  +------------------+                   +--------------------+
  |  Client Device   |                   |  Metadata Service  |
  |  (Watcher,       +---- gRPC / API--->|  (DB, Hash Maps)   |
  |   Chunker)       |                   +---------+----------+
  +--------+---------+                             |
           |                                       |
    (Upload Chunks)                           (Sync Notif)
           |                                       |
  +--------v---------+                   +---------v----------+
  |   Block Servers  |                   |Notification Service|
  |  (Hash = File)   +<--Long Polling----+ (WebSocket/PubSub) |
  +--------+---------+                   +--------------------+
           |
  +--------v---------+
  |  Object Storage  |
  |      (S3)        |
  +------------------+
```

---

## 4. API Design

```text
- upload_chunk(file_id: string, chunk_hash: string, data: bytes) -> bool
- update_metadata(file_id: string, new_hashes: List<string>) -> bool
- download_chunk(chunk_hash: string) -> bytes
```

---

## 5. Detailed Component Design

### 5.1 File Chunking & Rolling Hash
To save bandwidth, files are split into chunks (e.g., 4MB). 
If a user inserts a word at the beginning of a file, standard fixed-size chunking shifts all bytes, changing *every* chunk.
**Solution**: **Content-Defined Chunking (CDC)** using a **Rolling Hash (Rabin Fingerprint)**.
The algorithm slides a window over the file and creates a chunk boundary whenever the hash matches a specific pattern (e.g., bottom 13 bits are 0).

**Pseudocode for CDC:**
```cpp
List<Chunk> chunkFile(File file) {
    List<Chunk> chunks;
    int chunk_start = 0;
    RollingHash hash;
    
    for (int i = 0; i < file.size(); i++) {
        hash.update(file[i]); // O(1) sliding window hash
        
        // Condition to cut a chunk
        if (hash.getValue() % 4096 == 0 || (i - chunk_start) >= MAX_CHUNK_SIZE) {
            chunks.push_back(createChunk(file, chunk_start, i));
            chunk_start = i + 1;
            hash.reset();
        }
    }
    return chunks;
}
```

### 5.2 Deduplication
Chunks are passed through SHA-256 to generate a unique hash.
- **Storage**: The backend stores chunks in a Content-Addressable Storage (CAS) where the Key is the `SHA-256 hash`.
- **Upload Flow**: 
  1. Client sends a list of hashes to Metadata server.
  2. Server says "I already have hashes A and B, just send me C."
  3. Client uploads only chunk C.

### 5.3 Sync Protocol
- Client monitors local File System events.
- On change, chunks file -> calculates hashes -> syncs metadata -> uploads missing chunks.
- **Notification**: Server tells other connected devices of the same user via **Long Polling** or **WebSockets** that a file changed. They pull the new metadata and download missing chunks.

---

## 6. Addressing Bottlenecks

### Conflict Resolution
If User A and User B edit offline and sync simultaneously:
- **Last-Writer-Wins**: Easiest, but loses data.
- **Branching (Dropbox way)**: Save both copies, renaming one to "Conflicted Copy (timestamp)". Leave it to the user to merge.

---

## 7. Scaling and Resilience
- **Metadata DB**: Relational DB (ACID properties are strict here to ensure a file's chunk list is perfectly maintained). Sharded by `user_id`.
- **Cold Storage**: Old versions of files or deleted files are moved to Glacier/Cold storage to save massive costs.

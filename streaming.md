# 📡 Case Study #10: Content Streaming Systems
## Heavy on Bandwidth + Scalability

> **Target Audience:** Students preparing for system design interviews | Curious learners wanting to understand how real systems work  
> **Difficulty:** ⭐⭐⭐⭐ Advanced  
> **Topics Covered:** Video Streaming · Music Streaming · Live Sports Streaming  

---

## 📖 Table of Contents

1. [Overview & Why Streaming is Hard](#overview)
2. [Core Concepts You Must Know](#core-concepts)
3. [Part A: Video Streaming — YouTube](#part-a-youtube)
4. [Part B: Music Streaming — Spotify](#part-b-spotify)
5. [Part C: Live Sports Streaming — JioStar / Hotstar](#part-c-jiostar)
6. [Comparison Table: All Three Systems](#comparison)
7. [Algorithms & Complexity Cheat Sheet](#algorithms)
8. [Common Trade-offs & Design Decisions](#tradeoffs)
9. [Interview Tips & Sample Questions](#interview-tips)
10. [Key Takeaways](#key-takeaways)

---

## 1. Overview & Why Streaming is Hard {#overview}

Streaming is deceptively simple from a user's point of view — you press Play and content just... plays. Behind that single tap is one of the most complex engineering challenges in modern computing.

### What makes streaming hard?

| Challenge | Why It's Difficult |
|---|---|
| **Scale** | Millions of concurrent users, each consuming MBs per second |
| **Bandwidth** | Video streaming alone accounts for ~82% of global internet traffic |
| **Latency** | Users expect playback to start in < 2 seconds |
| **Heterogeneity** | Different devices, screen sizes, network speeds (2G to 5G) |
| **Reliability** | Buffering = users leave. Zero tolerance for outages |
| **Cost** | Storing and delivering petabytes of content costs billions |
| **Global reach** | Content must be delivered fast to users in Tokyo, Mumbai, and São Paulo equally |

### The Three Flavors of Streaming

```
┌─────────────────────────────────────────────────────────────────┐
│                    STREAMING SYSTEMS                            │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │   VOD VIDEO  │  │    AUDIO     │  │    LIVE SPORTS      │   │
│  │  (YouTube)   │  │  (Spotify)   │  │  (JioStar/Hotstar)  │   │
│  │              │  │              │  │                     │   │
│  │ • Pre-stored │  │ • Small file │  │ • Real-time only    │   │
│  │ • Seekable   │  │ • Fast seek  │  │ • Cannot pause      │   │
│  │ • Async      │  │ • Offline OK │  │ • Massive spikes    │   │
│  │   encoding   │  │ • ML-heavy   │  │ • Ultra low latency │   │
│  └──────────────┘  └──────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Concepts You Must Know {#core-concepts}

Before diving into each system, master these foundational concepts. They appear everywhere.

### 2.1 Adaptive Bitrate Streaming (ABR)

The single most important concept in streaming.

**Problem:** Network speeds vary. If you serve 4K video to someone on slow 2G, they'll buffer endlessly. If you serve 240p to someone on fibre, the experience is terrible.

**Solution:** Break the video into small segments (2–10 seconds each). Encode each segment at multiple quality levels. The client player monitors bandwidth in real-time and automatically fetches the best quality it can handle.

```
Video File
    │
    ▼
┌─────────────────────────────────────────────┐
│              ENCODER / TRANSCODER           │
│                                             │
│  Original ──► 4K   (15–68 Mbps)            │
│           ──► 1080p (8 Mbps)               │
│           ──► 720p  (5 Mbps)               │
│           ──► 480p  (2.5 Mbps)             │
│           ──► 360p  (1 Mbps)               │
│           ──► 240p  (0.4 Mbps)             │
└─────────────────────────────────────────────┘
    │
    ▼
 Segments (chunks) stored in object storage + CDN
 
Player Logic:
  bandwidth_good?  → upgrade quality
  bandwidth_drop?  → downgrade quality
  buffer < 10s?    → pre-fetch next chunk urgently
```

**Two dominant protocols:**
- **HLS (HTTP Live Streaming)** — Created by Apple. Uses `.m3u8` manifest + `.ts` segments. Best support on iOS.
- **DASH (Dynamic Adaptive Streaming over HTTP)** — Open MPEG standard. Uses `.mpd` manifest. Preferred on Android, Smart TVs, browsers.

### 2.2 Content Delivery Networks (CDNs)

**Problem:** Origin servers in one location can't serve the world with low latency.

**Solution:** A CDN is a globally distributed network of edge servers (called PoPs — Points of Presence). Content is cached close to users.

```
Without CDN:
  User in Mumbai ──────────────────────► Server in USA
                  (200ms+ latency, high load)

With CDN:
  User in Mumbai ──► CDN Edge Server in Mumbai
                     (5–10ms latency, cached content)
                     │ (cache miss)
                     └──► Origin Server in USA
```

**Cache strategy:**
- Hot content (trending videos): Cached everywhere
- Warm content (moderately popular): Cached regionally  
- Cold content (rare old videos): Served from origin on demand

### 2.3 Video Transcoding Pipeline

When a video is uploaded, it cannot be served immediately. It goes through a processing pipeline:

```
Upload (.mov, .avi, .mp4, .mkv, etc.)
    │
    ▼
[1] Validation → Is file valid? Not corrupt?
    │
    ▼
[2] Demuxing  → Separate audio track + video track
    │
    ▼
[3] Encoding  → Re-encode into standard codec (H.264 / H.265 / AV1)
    │
    ▼
[4] Segmentation → Break into 2-10 second chunks
    │
    ▼
[5] Multi-resolution Packaging → Generate all quality levels
    │
    ▼
[6] Manifest Generation → Create .m3u8 / .mpd index file
    │
    ▼
[7] Upload to Object Storage → S3, GCS, etc.
    │
    ▼
[8] CDN Propagation → Push to edge nodes
    │
    ▼
[9] Available for Streaming ✓
```

> ⚠️ **Interview Tip:** Never design transcoding as a synchronous call. A 4GB video takes 10–30 minutes to transcode — always offload to an async job queue (e.g., Kafka + worker fleet).

### 2.4 Data Storage Tiers

Streaming platforms use tiered storage to balance cost vs. performance:

| Tier | Type | Access Speed | Cost | Used For |
|---|---|---|---|---|
| **Hot** | SSD / In-Memory | Milliseconds | Very High | Trending, recently uploaded |
| **Warm** | HDD / NVMe | Seconds | Medium | Moderately popular content |
| **Cold** | Tape / Glacier | Minutes–Hours | Very Low | Archive, rarely accessed |

---

## 3. Part A: Video Streaming — YouTube {#part-a-youtube}

### 3.1 System Overview

YouTube is the world's largest video platform, with over **500 hours of video uploaded every minute** and **billions of daily views**. It serves as the canonical example of a Video-on-Demand (VOD) streaming system.

### 3.2 Scale Estimations (Back-of-Envelope)

Let's calculate what YouTube actually needs to handle:

```
Daily Active Users:        800 million
Videos watched per user:   5/day
Total daily views:         800M × 5 = 4 billion views/day

Concurrent viewers (10%):  400 million
Avg download speed/user:   3 MB/s

Required throughput:       400M × 3 MB/s = 1,200 TB/s = 1.2 Exabytes/second

Single server limit:       ~1-2 GB/s (SSD)

Servers needed (approx):   1.2 × 10^18 / 1 × 10^9 ≈ 1.2 billion server-seconds!

→ This is WHY distributed systems, CDNs, and chunked delivery exist.
  No single server can handle this. Not even close.
```

**Storage estimation:**
```
New uploads per day:       500 hrs/min × 60 × 24 = 720,000 hrs of video/day
Storage per hour (all res):~1.5 GB/hr
Daily storage needed:      720,000 × 1.5 GB ≈ 1 Petabyte/day
```

### 3.3 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        YOUTUBE ARCHITECTURE                  │
│                                                              │
│  ┌──────────┐     ┌──────────────┐     ┌─────────────────┐  │
│  │  CLIENT  │────►│ Load Balancer│────►│   API Gateway   │  │
│  │(Web/App) │     │  (Anycast)   │     │  (Auth, Rate    │  │
│  └──────────┘     └──────────────┘     │   Limiting)     │  │
│                                        └────────┬────────┘  │
│                                                 │            │
│         ┌───────────────┬──────────────┬────────┘            │
│         ▼               ▼              ▼                     │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                │
│   │  Upload  │   │  Watch   │   │  Search  │  ← Microservices│
│   │ Service  │   │ Service  │   │ Service  │                 │
│   └────┬─────┘   └────┬─────┘   └──────────┘                │
│        │              │                                      │
│        ▼              ▼                                      │
│   ┌──────────┐   ┌──────────────────────────────────────┐    │
│   │  Kafka   │   │           CDN Layer                  │    │
│   │(Message  │   │  (Google CDN / Akamai Edge Servers)  │    │
│   │  Queue)  │   │  🌏 Tokyo  🌍 London  🌎 São Paulo   │    │
│   └────┬─────┘   └──────────────────────────────────────┘    │
│        │                        ▲                            │
│        ▼                        │ cache miss                 │
│   ┌──────────┐             ┌────┴──────────────────────┐     │
│   │Transcode │             │      Object Storage       │     │
│   │ Workers  │────────────►│  (Google Cloud Storage)   │     │
│   │(FFmpeg + │             │  Bigtable / Spanner (meta)│     │
│   │ Custom   │             └───────────────────────────┘     │
│   │  Chips)  │                                               │
│   └──────────┘                                               │
└──────────────────────────────────────────────────────────────┘
```

### 3.4 Deep Dive: Upload & Processing Flow

**Step 1: Upload**
- Client sends video via HTTP multipart upload to Upload Service
- For large files (>100MB), YouTube uses **resumable uploads**: file is split into chunks client-side, each chunk uploaded separately
- If connection drops, upload resumes from last successful chunk — not from beginning
- Upload Service immediately returns a `job_id` (async — doesn't wait for processing)

**Step 2: Ingestion to Message Queue**
- Upload Service publishes an event to **Kafka**: `{video_id, storage_path, uploader_id}`
- Video is stored in raw form in Google Cloud Storage (cold, immediately)

**Step 3: Transcoding (The Heavy Lifting)**
- Transcoding Worker Fleet (powered by FFmpeg + Google's custom encoding ASICs) picks up the job
- Parallel encoding: multiple workers encode different resolutions simultaneously
- Google uses **TPUs and custom chips** to accelerate encoding — far faster than CPU-only
- Generates: 144p, 240p, 360p, 480p, 720p, 1080p, 1440p, 4K, HDR variants
- Each resolution is **segmented** into 10-second `.ts` chunks
- `.m3u8` manifest file is generated linking all chunks

**Step 4: Storage & CDN Propagation**
- All video chunks stored in Google Cloud Storage
- Manifest + popular video chunks pushed to CDN edge nodes worldwide
- Metadata (title, description, tags, uploader, timestamps) stored in **Bigtable** (fast reads) and **Spanner** (global consistency)

**Step 5: Video Available**
- Uploader notified. Video is publicly available.
- Search index updated (Elasticsearch-like) for discoverability

### 3.5 Deep Dive: Playback Flow

```
User clicks "Play"
      │
      ▼
[1] GET /watch/{video_id}  → API Gateway
      │
      ▼
[2] Watch Service fetches metadata from Bigtable (cached in Redis)
      │
      ▼
[3] Returns manifest URL + signed access token (expires in ~2 hours)
      │
      ▼
[4] Client fetches .m3u8 manifest from CDN edge server
      │
      ▼
[5] Client detects bandwidth → selects initial quality
      │
      ▼
[6] Client requests first chunk → served from CDN
      │
      ▼
[7] Playback begins (< 2 seconds after click)
      │
      ▼
[8] Background: pre-fetch next 3 chunks, monitor bandwidth,
    adjust quality dynamically (ABR algorithm)
```

**CDN caching logic:**
```
Request for chunk arrives at CDN edge:
  ├── Cache HIT  → return immediately (< 5ms)
  └── Cache MISS → fetch from origin, cache, return (50-200ms)

CDN uses LRU (Least Recently Used) eviction:
  - Hot chunks (trending videos) stay in cache
  - Cold chunks evicted when storage is needed
```

### 3.6 Data Models

**Video Metadata Table (Bigtable)**
```
video_id     | title        | uploader_id | duration | views | status      | created_at
─────────────────────────────────────────────────────────────────────────────────────
abc123       | "My Vlog"    | user_456    | 600s     | 0     | PROCESSING  | 2024-01-01
def789       | "Tech Review"| user_789    | 1200s    | 5M    | AVAILABLE   | 2023-06-15
```

**Video Chunks Table (Object Storage metadata)**
```
chunk_id     | video_id | resolution | segment_num | duration | storage_path      | cdn_url
──────────────────────────────────────────────────────────────────────────────────────────────
c_001        | abc123   | 720p       | 1           | 10s      | gcs://bucket/...  | cdn.yt.com/...
c_002        | abc123   | 720p       | 2           | 10s      | gcs://bucket/...  | cdn.yt.com/...
```

### 3.7 Key Design Decisions & Justifications

| Decision | Rationale |
|---|---|
| Async transcoding via Kafka | HTTP times out on large videos. Decouples upload from processing |
| Multi-resolution encoding | Supports all devices and network conditions (ABR) |
| Google's custom CDN | Deep GCP integration, lower cost at YouTube's scale than using Akamai |
| Bigtable for video metadata | Horizontally scalable, fast reads, handles YouTube's billions of videos |
| Chunked (segmented) storage | Enables parallel CDN caching, random seeks, and partial downloads |
| Spanner for user/auth data | Strong consistency for financial/auth operations; multi-region |
| Redis for hot metadata cache | Sub-millisecond response for trending video metadata |

### 3.8 Scalability Patterns Used

**Horizontal Sharding for hot videos:**
```
Problem: A single viral video gets 10M concurrent viewers.
         One server handles ~1GB/s → can't handle 10M × 3MB/s = 30TB/s

Solution: Vertical sharding of chunks
  - Chunk #1 stored on Server Group A (with N replicas)
  - Chunk #2 stored on Server Group B (with N replicas)
  - Load spreads across ALL server groups
  - Each chunk server can be independently scaled
```

**The 20/80 Rule (Pareto Caching):**
```
20% of videos get 80% of views.

Cache top 20% → 40 GB for metadata cache
               → Massive CDN bandwidth savings
```

---

## 4. Part B: Music Streaming — Spotify {#part-b-spotify}

### 4.1 System Overview

Spotify serves over **600 million monthly active users** with a catalog of **100+ million songs**. Unlike video, audio files are small (3–8 MB per song), but the real complexity lies in **real-time personalization, multi-device sync, and recommendation algorithms**.

### 4.2 Scale Estimations

```
Monthly Active Users:     600 million
Daily Active Users:       100 million
Songs streamed daily:     100M × 10 songs = 1 billion streams/day
Data transfer daily:      1B × 5 MB = 5 Petabytes/day
Data transfer per second: 5 PB / 86,400s ≈ 58 GB/second

Total song storage:       100M songs × 5 MB = 500 Terabytes
Metadata storage:         100M songs × 2 KB = 200 GB
User data:                500M users × 10 KB = 5 TB

Top 20% songs cached:     20M × 2 KB = 40 GB metadata cache
```

### 4.3 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SPOTIFY ARCHITECTURE                         │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────────┐    │
│  │  iOS App │  │ Android  │  │   Web    │  │  Desktop/Smart  │    │
│  │          │  │   App    │  │ Player   │  │   TV / Car      │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬─────────┘    │
│       └─────────────┴──────────────┴────────────────┘              │
│                              │                                      │
│                              ▼                                      │
│                    ┌──────────────────┐                             │
│                    │   API Gateway    │                             │
│                    │ (Auth / Rate     │                             │
│                    │  Limiting / SSL) │                             │
│                    └────────┬─────────┘                             │
│                             │                                       │
│         ┌──────────┬────────┴──────────┬────────────┐              │
│         ▼          ▼                   ▼            ▼              │
│   ┌──────────┐ ┌──────────┐     ┌──────────┐  ┌──────────┐        │
│   │Streaming │ │ Search   │     │ Playlist │  │Recommend │        │
│   │ Service  │ │ Service  │     │ Service  │  │  Engine  │        │
│   └────┬─────┘ └──────────┘     └──────────┘  └────┬─────┘        │
│        │                                           │               │
│        ▼                                           ▼               │
│   ┌──────────────────────────┐        ┌──────────────────────┐     │
│   │         CDN Layer        │        │   Apache Kafka        │    │
│   │  (CloudFront / Akamai)   │        │  (Event streaming)    │    │
│   │  Audio chunks at edge    │        │  plays, skips, likes  │    │
│   └──────────────────────────┘        └──────────┬───────────┘     │
│                                                  │                 │
│                              ┌───────────────────┘                 │
│                              ▼                                      │
│                   ┌────────────────────────┐                       │
│                   │    Data Warehouse       │                       │
│                   │  Apache Spark (batch)   │                       │
│                   │  Flink (real-time)      │                       │
│                   │  → Discover Weekly      │                       │
│                   │  → Spotify Wrapped      │                       │
│                   └────────────────────────┘                       │
│                                                                     │
│  Storage Layer:                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │   AWS S3     │  │  Cassandra   │  │    Redis     │             │
│  │ (Audio files)│  │  (User data, │  │  (Session,   │             │
│  │              │  │   playlists) │  │   cache)     │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.4 Deep Dive: Audio Streaming

Audio streaming is simpler than video in file size, but harder in experience requirements:

**Chunked Progressive Streaming:**
```
Song file: "Blinding Lights.mp3" (5 MB, 3:22 long)
    │
    ▼
Split into ~10-second chunks:
  chunk_001.ts  (320kbps, high quality)
  chunk_002.ts
  ...
  chunk_020.ts
    │
    ▼
Stored in S3 → Cached on Akamai CDN edge servers

Client behavior:
  1. Request chunk_001 → playback starts immediately
  2. While playing chunk_001 → pre-fetch chunk_002, chunk_003 (buffer ahead)
  3. Smart buffering: increase buffer if bandwidth drops, decrease if memory is low
  4. Adaptive bitrate: switch between 96kbps / 160kbps / 320kbps based on connection
```

**The Multi-CDN / Multi-Tier caching:**
```
User requests song:
  └─► CDN Edge Cache (city level)
         └─► CDN Regional Cache (country level)
                └─► AWS S3 Origin (global)

Cache Eviction: LRU with popularity weighting
  → Top 20% songs (by streams) never evicted from edge
  → New releases pre-warmed at edges before release date
```

### 4.5 Deep Dive: The Recommendation Engine

Spotify's recommendation engine is a **hybrid system** combining three approaches:

```
┌──────────────────────────────────────────────────────────────┐
│              SPOTIFY RECOMMENDATION PIPELINE                  │
│                                                               │
│  Input Signals:                                               │
│  • Plays, skips, saves, playlist adds                         │
│  • Time of day, mood context                                  │
│  • Social signals (friends' activity)                         │
│  • Search queries                                             │
│                │                                              │
│                ▼                                              │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  1. COLLABORATIVE FILTERING                          │     │
│  │     "Users who listen to X also listen to Y"         │     │
│  │     Matrix factorization on user × song matrix       │     │
│  │     Complexity: O(users × songs) for training        │     │
│  │                                                      │     │
│  │  2. CONTENT-BASED FILTERING                          │     │
│  │     Analyze audio features: tempo, key, danceability │     │
│  │     Neural audio embeddings (NLP on audio waves)     │     │
│  │     Cold-start solution for new songs                │     │
│  │                                                      │     │
│  │  3. NATURAL LANGUAGE PROCESSING (NLP)                │     │
│  │     Analyze blog posts, reviews, social media        │     │
│  │     about artists to infer "vibe" / genre context    │     │
│  └──────────────────────────────────────────────────────┘     │
│                │                                              │
│                ▼                                              │
│         Ranking Model (learned weights)                       │
│         → Balances: user taste + discovery + business needs   │
│                │                                              │
│                ▼                                              │
│  Output: Discover Weekly, Daily Mix, Radio, Autoplay          │
└──────────────────────────────────────────────────────────────┘
```

**Discover Weekly Algorithm (simplified):**
1. Build a user-song interaction matrix (500M users × 100M songs — sparse)
2. Apply **Approximate Nearest Neighbors (ANN)** using embeddings
3. Find users with 80%+ overlapping taste profile
4. Recommend songs those users love that you haven't heard
5. Filter out songs you've already played or skipped multiple times
6. Batch-process every Monday morning → your new Discover Weekly playlist

> **Complexity note:** Full matrix factorization is O(users × factors × iterations). At 500M users, even a single pass is expensive. Spotify uses approximate methods (LSH, FAISS) for sub-linear lookup in production.

### 4.6 Multi-Device Synchronization

One of Spotify's most technically impressive features:

```
Challenge: You're listening on your phone → hand off to laptop mid-song

Solution: Persistent WebSocket connections

Each active client maintains a WebSocket connection to a "Player State" service.

State object:
{
  user_id: "user123",
  active_device: "iPhone_14",
  current_track: "track_789",
  position_ms: 45230,       ← millisecond position in song
  is_playing: true,
  volume: 0.8,
  queue: ["track_790", "track_791", ...]
}

When you switch device:
  1. New device (laptop) sends "transfer_playback" request
  2. Player State service updates active_device
  3. Pushes state to ALL connected devices via WebSocket
  4. Laptop player seeks to exact position_ms and resumes
  5. Phone player pauses

Sync latency target: < 300ms
Conflict resolution: Last-write-wins + client-side version numbers
```

### 4.7 Spotify's Database Evolution

A fascinating real-world example of scaling decisions:

```
Early Spotify (2006-2013):
  ├── PostgreSQL (relational) for everything
  └── Problem: Transatlantic cable cut → replication broke → DOWNTIME

Migration to Cassandra (2013):
  ├── Apache Cassandra: NoSQL, distributed, fault-tolerant
  ├── Migration strategy: "Dark Loading" 
  │   → Copy data silently to Cassandra while still serving from Postgres
  │   → Zero downtime. Users never knew.
  └── Why Cassandra?
      → Masterless architecture (no single point of failure)
      → Tunable consistency (can trade consistency for availability)
      → Linear horizontal scalability

Today:
  ├── Cassandra → User data, playlists, listening history (write-heavy)
  ├── AWS S3    → Audio files (object storage)
  ├── Redis     → Sessions, caching, pub/sub
  └── PostgreSQL → Billing, payments (needs ACID compliance)
```

### 4.8 Search Architecture

```
Inverted Index for "Blinding Lights":
  "blinding" → [track_789, track_1203, ...]
  "lights"   → [track_789, track_456, ...]
  
Fuzzy matching with Trie structure:
  "Blyndng Lites" → Levenshtein distance ≤ 2 → matches "Blinding Lights"

Search ranking factors:
  1. Exact match score
  2. Popularity (stream count)
  3. Freshness (new releases boosted)
  4. Personalization (your taste profile)
  5. Geographic relevance (local artists boosted)
```

---

## 5. Part C: Live Sports Streaming — JioStar / Hotstar {#part-c-jiostar}

### 5.1 System Overview

Live sports streaming is fundamentally **different** from VOD (YouTube) or on-demand audio (Spotify). There is **no second chance**. If the stream drops during a World Cup final, millions of people see it immediately. You cannot replay a missed moment from the server's perspective — it's happening right now.

**JioStar (merger of JioCinema + Disney+ Hotstar, February 2025)** is the world's most tested live streaming platform:

- **65.2 million concurrent viewers** — India vs England T20 World Cup 2026 semi-final (highest ever recorded on any streaming platform globally)
- **55 million concurrent viewers** — IPL 2025
- **840 billion minutes** of total watch time for IPL 2025 season
- **800+ microservices** running on AWS EKS
- **500 million registered users**, 300 million paid subscribers
- Content in **19 Indian languages**

### 5.2 Why Live Streaming is Harder Than VOD

| Property | VOD (YouTube) | Live Sports (JioStar) |
|---|---|---|
| **Content available before delivery?** | Yes — pre-processed | No — encoding happens in real time |
| **Latency tolerance** | 2-5s startup OK | Must be < 10-30s or spoilers ruin it |
| **Traffic pattern** | Gradual, predictable | **Explosive spikes** in seconds |
| **Caching** | Works great for hot videos | Much harder — live stream is unique each second |
| **Failure recovery** | Re-fetch chunk | Viewer misses moment forever |
| **Scale planning** | Can be estimated | India scores a boundary → +5M viewers in 10 seconds |

### 5.3 The Live Streaming Pipeline

```
On-Ground (Stadium)
┌──────────────────────────────────────────────────┐
│  OB Truck (Outside Broadcast)                    │
│  4K Cameras → Switcher → Satellite Uplink        │
└──────────────────────────┬───────────────────────┘
                           │ Raw feed (uncompressed or lightly compressed)
                           ▼
┌──────────────────────────────────────────────────┐
│  Ingest Layer                                    │
│  Protocol: RTMP (Real-Time Messaging Protocol)   │
│  OR SRT (Secure Reliable Transport) for low lat  │
│  Ingest servers: multiple redundant receivers    │
└──────────────────────────┬───────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────┐
│  Encoding Farm (AWS Elemental MediaLive)          │
│  7 simultaneous renditions in PARALLEL:           │
│  ├── 4K HDR  (15+ Mbps)                          │
│  ├── 1080p   (8 Mbps)                            │
│  ├── 720p    (5 Mbps)                            │
│  ├── 480p    (2.5 Mbps)                          │
│  ├── 360p    (1 Mbps)                            │
│  ├── 240p    (400 Kbps)                          │
│  └── 144p    (150 Kbps) ← for 2G users           │
│                                                   │
│  Output format: HLS (.m3u8 + .ts segments)       │
│  Segment size: 2-4 seconds (shorter = lower lat) │
└──────────────────────────┬───────────────────────┘
                           │ ~10 second end-to-end latency target
                           ▼
┌──────────────────────────────────────────────────┐
│  Packaging & DRM                                 │
│  • Encrypt segments with AES-128 (Widevine/FPS)  │
│  • Package into HLS + DASH variants              │
│  • Generate live manifest (sliding window)        │
│    [chunk_n-10, chunk_n-9, ..., chunk_n] ← latest│
└──────────────────────────┬───────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────┐
│  Multi-CDN Distribution                          │
│  Primary: Akamai (global)                        │
│  Secondary: AWS CloudFront                       │
│  Tertiary: Jio MEC (Mobile Edge Computing)       │
│            ← Physical servers INSIDE Jio's 5G   │
│              network. Unbeatable latency for     │
│              Jio subscribers in India.           │
│                                                  │
│  In-house CDN load optimizer:                    │
│  → Monitors CDN health in real-time             │
│  → Dynamically routes viewers to least congested │
│  → 90% of IPL content served from CDN cache     │
└──────────────────────────┬───────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────┐
│  Viewer's Device                                 │
│  ABR Player monitors bandwidth every 2 seconds  │
│  Adjusts quality: 4K ↔ 1080p ↔ 720p ↔ 480p...  │
│  Buffer: 3-4 chunks ahead (6-16 seconds buffer) │
└──────────────────────────────────────────────────┘
```

### 5.4 The Autoscaling Challenge

This is the most unique and hardest engineering problem in live sports streaming.

```
Traffic during a cricket match:

100M
 │                           ┌────────────────────┐
 │                   ┌───────┘  India wins!        │
 │           ┌───────┘  BOUNDARY! +5M viewers     │
 │   ┌───────┘          in 10 seconds              │
 │ ──┘ Match starts                                │
 │                                                  └────
 │                                              Match ends
 └─────────────────────────────────────────────────────► time

Problem: Traditional autoscaling takes 3-5 MINUTES to spin up new servers.
         5 million new viewers arrive in 10 seconds.

JioStar's solution:

1. PRE-WARMING:
   → Before match starts, spin up 70% of expected peak capacity
   → Use ML models to predict viewership based on:
      • Teams playing (India match = 5× more viewers)
      • Day/time (weekend evening = higher)
      • Tournament stage (finals = maximum)
      • Historical data from similar matches

2. CUSTOM AUTOSCALER on AWS EKS:
   → Reacts in < 30 seconds (vs. standard 3-5 minutes)
   → Monitors: requests/sec, CDN origin hit rate, CPU utilization
   → Spins up pods in parallel, not sequentially
   → Peak: 4,000+ Kubernetes worker nodes
             800+ microservices
             16 TB RAM, 8,000 CPU cores, 32 Gbps bandwidth

3. SPOT INSTANCE OPTIMIZATION:
   → Mix of on-demand (reliable) + spot instances (cheap, interruptible)
   → ML models predict when spot instances will be interrupted
   → Shift load to on-demand BEFORE interruption happens
   → Saves ~60-70% on compute costs vs. all on-demand
```

### 5.5 Server-Side Ad Insertion (SSAI)

JioStar's business model depends on serving ads at massive scale. A key technical challenge:

```
Client-Side Ad Insertion (Old way):
  → Browser/app fetches video stream
  → Separately fetches ad
  → Plays ad, then resumes stream
  Problem: Ad blockers! Users block the ad request.
           Also: buffering during ad-video transition.

Server-Side Ad Insertion (JioStar since 2019):
  → Ad is "stitched" into the video stream at the server side
  → Client receives ONE seamless stream (video + ads merged)
  → Ad blockers can't distinguish ad segments from video segments
  → No buffering during ad break

Technical implementation:
  ┌────────────────────────────────────────────────────┐
  │  Decision Engine (real-time, per viewer):          │
  │  User profile + match moment → ad selection        │
  │  Decision must happen in < 50ms                    │
  │                                                    │
  │  At 50M concurrent viewers with 1 ad break/min:   │
  │  = 50M ad decisions per minute = 833,000/second   │
  │                                                    │
  │  Solution: Pre-bid auction + local caching         │
  └────────────────────────────────────────────────────┘
```

### 5.6 Anti-Piracy Architecture

During major matches, piracy is a serious threat (and significant revenue loss):

```
Threats:
  1. Stream ripping: Automated bots download live stream chunks
  2. Re-streaming: Pirates re-broadcast your stream elsewhere
  3. Credential sharing: Multiple users on one account
  4. Screen capture: Not preventable at software level

JioStar's defenses:

1. DRM (Digital Rights Management):
   • AES-128 encryption of every video segment
   • Unique decryption key per session (short-lived, 2-hour TTL)
   • Keys delivered via Widevine (Android/Chrome) or FairPlay (iOS/Safari)
   → Even if a pirate downloads encrypted .ts files, they can't play them

2. Token Authentication:
   • Every manifest and chunk URL contains a signed JWT token
   • Token encodes: user_id, expiry, allowed_geo, device_fingerprint
   • CDN validates token before serving any content

3. Geo-restriction:
   • IP geolocation + GPS (mobile) check
   • License for IPL only covers India → block international IPs

4. Anomaly Detection:
   • Flag accounts streaming from 5+ different IPs/locations
   • Real-time traffic analysis for bot patterns
   • Rate limiting on manifest requests

5. Watermarking:
   • Forensic watermark embedded invisibly in video
   • If pirated stream found online → trace it back to original account
```

### 5.7 Real Numbers: JioStar Infrastructure

| Component | Specification |
|---|---|
| Compute | 500+ C4 AWS instances (C4.4xlarge: 30GB RAM, C4.8xlarge: 60GB RAM) |
| Total RAM | 16 TB |
| Total CPU cores | 8,000 |
| Peak bandwidth | 32 Gbps |
| Microservices | 800+ on AWS EKS |
| CDN partners | Akamai (primary) + AWS CloudFront |
| Peak concurrent viewers | 65.2 million (T20 World Cup 2026) |
| Uptime during IPL 2025 | 99.995% |
| Autoscaler reaction time | < 30 seconds |
| End-to-end latency | < 10 seconds from stadium to viewer |

---

## 6. Comparison Table: All Three Systems {#comparison}

| Property | YouTube (VOD Video) | Spotify (Audio) | JioStar (Live Sports) |
|---|---|---|---|
| **Content type** | Long-form video | Audio tracks | Live video stream |
| **Pre-processing** | Yes (async transcoding) | Yes (pre-encoded) | No (real-time encoding) |
| **Avg file size** | Huge (GBs) | Small (3-8 MB) | N/A (streaming only) |
| **Latency requirement** | Low (2-5s startup OK) | Very Low (< 1s) | Ultra-low (< 10s) |
| **Traffic pattern** | Predictable, smooth | Smooth | **Explosive spikes** |
| **Caching effectiveness** | Very High (VOD) | High | Medium (live is unique) |
| **Seek capability** | Yes (random access) | Yes (instant) | Limited (DVR window only) |
| **Main DB** | Bigtable + Spanner | Cassandra | DynamoDB / Aurora |
| **Message queue** | Kafka | Kafka | Kafka + Kinesis |
| **CDN strategy** | Google CDN | Akamai + CloudFront | Multi-CDN + Jio MEC |
| **Recommendation** | Neural nets (Watch history) | Collaborative + Content-based | Less relevant |
| **Unique challenge** | Transcoding at scale | Multi-device sync | Traffic spikes in seconds |
| **Biggest cost** | Storage + CDN egress | CDN + Recommendation compute | Compute (no pre-caching) |
| **Scale** | 2B+ users | 600M+ users | 500M+ users, 65M concurrent peak |

---

## 7. Algorithms & Complexity Cheat Sheet {#algorithms}

### 7.1 Adaptive Bitrate (ABR) Algorithm

```
At each chunk boundary (every 2-10 seconds):

Algorithm BOLA (Buffer Occupancy based Lyapunov Algorithm):
  Input:  current_buffer_level, available_bandwidths[], quality_levels[]
  Output: next_chunk_quality

Simplified logic:
  if buffer_level > TARGET_BUFFER (e.g., 30s):
    → upgrade quality (bandwidth is more than enough)
  elif buffer_level < CRITICAL_BUFFER (e.g., 10s):
    → downgrade quality (prevent rebuffering)
  else:
    → estimate_bandwidth() → pick highest quality ≤ estimated_bandwidth × 0.8

Time Complexity: O(1) per decision (constant time)
Space Complexity: O(k) where k = number of quality levels (constant, ~6-8 levels)
```

### 7.2 Collaborative Filtering (Spotify)

```
Matrix Factorization (ALS - Alternating Least Squares):

User-Song matrix R (500M × 100M) → decompose into:
  R ≈ U × V^T
  where U = User embedding matrix (500M × k)
        V = Song embedding matrix (100M × k)
        k = number of latent factors (~128-512)

Training complexity:   O(|R| × k) per iteration where |R| = non-zero entries
Inference complexity:  O(k) per recommendation (dot product of two k-dim vectors)
Space:                 O((users + songs) × k)

In practice: Uses approximate nearest neighbor (ANN) search
  → FAISS (Facebook AI Similarity Search)
  → Complexity: O(log n) for k-nearest neighbor lookup
  → vs. O(n) for exact search (n = 100M songs)

This is why Spotify can recommend songs in milliseconds.
```

### 7.3 Search with Inverted Index + Trie

```
Inverted Index build: O(N × L) where N = songs, L = avg title length
Inverted Index lookup: O(q + k) where q = query terms, k = result count

Trie-based autocomplete:
  - Build: O(total characters across all terms)
  - Search: O(p + n) where p = prefix length, n = results
  - Space: O(ALPHABET_SIZE × max_depth × nodes) = O(26 × max_depth × terms)

Fuzzy matching (Levenshtein):
  - Naive: O(m × n) per pair where m, n = string lengths
  - With Trie pruning: O(m × n × relevant_nodes) — much faster in practice
```

### 7.4 CDN Cache Eviction (LRU with Popularity Weighting)

```
Standard LRU: O(1) get/put using HashMap + Doubly Linked List

Weighted LRU (used in CDNs):
  Score = (recency_score × α) + (popularity_score × β)
  
  Evict lowest score when cache is full.
  
  YouTube variant: Never evict top-1000 trending videos regardless of recency.

LFU (Least Frequently Used) alternative:
  - Better for stable popular content
  - O(1) operations using HashMap + Min-Heap or Frequency List
```

### 7.5 Consistent Hashing (for Distributed Caching)

```
Problem: How do you distribute 10M video chunks across 1000 servers
         such that when you add/remove a server, minimal data moves?

Naive approach: server = chunk_id % num_servers
  → Adding/removing server → rehash EVERYTHING → massive data movement

Consistent Hashing:
  - Place servers on a hash ring (0 to 2^32)
  - Each chunk maps to closest server clockwise
  - Adding a server: only ~1/N of data moves (where N = server count)
  - Removing a server: only that server's data moves to next
  
Time: O(log N) lookup using binary search on sorted ring
Space: O(N + K) where N = servers, K = virtual nodes per server
```

---

## 8. Common Trade-offs & Design Decisions {#tradeoffs}

### 8.1 Consistency vs. Availability (CAP Theorem)

| Scenario | Choice | Reason |
|---|---|---|
| View count on YouTube | Eventual Consistency | A video showing 1,234,567 vs 1,234,568 views doesn't matter. Availability matters more. |
| User account / password | Strong Consistency | Must be correct everywhere immediately. |
| Spotify playlist sync | Eventual Consistency + Versioning | Brief inconsistency tolerable; use version numbers to resolve conflicts. |
| Payment / billing | Strong Consistency (ACID) | Money must never be lost or double-charged. |
| Live stream position | Strong Consistency | All devices must agree on current playback position. |

### 8.2 Push vs. Pull for CDN Content

| Approach | How it works | Best for |
|---|---|---|
| **Push CDN** | Origin pushes content to all edge nodes proactively | Predictable, popular content (YouTube trending videos) |
| **Pull CDN** | Edge fetches from origin on first request, then caches | Long-tail content (most of YouTube's 800M+ videos that rarely get watched) |

YouTube uses both: **push** for trending/new releases, **pull** for long-tail.

### 8.3 HLS vs. DASH

| | HLS | DASH |
|---|---|---|
| **Creator** | Apple | MPEG (open standard) |
| **Manifest format** | `.m3u8` | `.mpd` |
| **Best supported on** | iOS, Safari, Apple TV | Android, Chrome, Smart TVs |
| **Latency** | Higher (6-30s typically) | Lower (2-8s typical) |
| **DRM** | FairPlay (Apple) | Widevine (Google) |
| **Usage** | YouTube (both), Spotify, Netflix | YouTube (both), JioStar |

Modern platforms support **both** via adaptive switching.

### 8.4 Monolith vs. Microservices

| | Monolith | Microservices |
|---|---|---|
| **Deployment** | One deployable unit | Hundreds of independent services |
| **Scaling** | Scale everything together | Scale only what needs it |
| **Failure isolation** | One bug can crash all | One service fails, others keep running |
| **Development speed** | Fast initially | Fast at scale with many teams |
| **Complexity** | Low | Very High (service discovery, distributed tracing) |
| **Who uses it** | Early-stage startups | YouTube, Spotify, JioStar (all use microservices) |

---

## 9. Interview Tips & Sample Questions {#interview-tips}

### 9.1 How to Approach a Streaming System Design Interview

```
Step 1: Clarify Requirements (5 min)
  → "Is this live or on-demand?"
  → "What's the scale? 1M users or 1B users?"
  → "Any special requirements: offline, DRM, multi-device?"
  → "What's more important: latency or cost?"

Step 2: High-Level Architecture (10 min)
  → Draw the main boxes: Client, CDN, API Gateway, Services, Storage
  → Explain data flow: upload path + read/playback path separately

Step 3: Deep Dive Components (15 min)
  → Pick 2-3 components to go deep on
  → Always discuss: the naive approach → why it fails → the real solution

Step 4: Address Bottlenecks (5 min)
  → Where does the system break under load?
  → How do you handle hot videos / sudden traffic spikes?
  → What happens if CDN goes down?

Step 5: Trade-offs & Alternatives (5 min)
  → "I chose Kafka over RabbitMQ because..."
  → "I chose Cassandra over MySQL because..."
  → Show you understand the WHY, not just the WHAT
```

### 9.2 Sample Interview Questions

**Easy / Medium:**
1. Design a basic video upload and playback system.
2. How would you implement video seeking? (Why can't you just `seek()` on a single file?)
3. Why is transcoding done asynchronously? What problems does sync transcoding cause?
4. What is ABR and why does it matter for user experience?
5. How does a CDN work? What happens on a cache miss?

**Hard:**
6. Design YouTube's transcoding pipeline for 500 hours of uploads per minute.
7. How would you handle a video that goes viral (10M concurrent viewers within 1 hour)?
8. Design Spotify's "Discover Weekly" recommendation system.
9. How does JioStar handle 5 million viewers joining in 10 seconds during a cricket match?
10. Design a system for multi-device playback synchronization (Spotify Connect).

**Advanced:**
11. How would you implement server-side ad insertion at 50M concurrent streams?
12. Design a content watermarking system that can trace a pirated stream back to its source.
13. How would you design CDN cache eviction for a platform with both viral short videos and long-tail archive content?
14. Spotify used "dark loading" to migrate from PostgreSQL to Cassandra. How does that pattern work?

### 9.3 Common Mistakes to Avoid

❌ **Don't design transcoding as a synchronous HTTP call.**  
✅ Always use async job queues (Kafka, SQS).

❌ **Don't serve videos from a single origin server.**  
✅ Always mention CDN with edge caching.

❌ **Don't use a single database for everything.**  
✅ Different data has different access patterns — use the right DB for each.

❌ **Don't forget about encoding formats and codecs.**  
✅ Mention H.264 (compatibility) vs H.265/AV1 (efficiency) trade-off.

❌ **Don't ignore bandwidth estimation.**  
✅ Do back-of-envelope: users × quality × concurrency → shows you understand scale.

❌ **Don't say "add more servers" without explaining HOW (autoscaling, consistent hashing, sharding).**

---

## 10. Key Takeaways {#key-takeaways}

### The Big Ideas

1. **Break it up** — Streaming works by breaking content into small chunks (2-10s), not streaming one giant file. This enables ABR, CDN caching, parallel delivery, and seekability.

2. **Push content to the edge** — CDNs are non-negotiable at scale. The user should get bytes from 5ms away, not 200ms away.

3. **Async everything heavy** — Transcoding, recommendation updates, analytics — never do these synchronously. Use message queues (Kafka).

4. **Design for the spike, not the average** — Live streaming systems are designed for the 65M-viewer moment, not the 5M-viewer baseline.

5. **Microservices enable independent scaling** — You need to scale your transcoding service 100×, not your authentication service. Microservices make this possible.

6. **The right database for the right job** — Bigtable for metadata, Cassandra for user events, Redis for caching, PostgreSQL for billing. One size does not fit all.

7. **Algorithms matter at scale** — Approximate nearest neighbors instead of exact search. Consistent hashing instead of modulo. LRU with popularity weighting instead of pure LRU. Small algorithmic improvements = billions of dollars in infrastructure savings.

### Mental Model

```
Think of a streaming platform as a pipeline:

PRODUCE → PROCESS → STORE → DISTRIBUTE → CONSUME
(Upload) (Transcode)(Object  (CDN/Edge)  (Player)
                   Storage)

Every box in this pipeline can (and will) become a bottleneck.
Great system design means knowing WHICH box to optimize and WHEN.
```

---

## Further Reading & References

- [YouTube Engineering Blog](https://blog.youtube/inside-youtube/)
- [Spotify Engineering Blog](https://engineering.atspotify.com/)
- [Hotstar Engineering Blog](https://blog.hotstar.com/)
- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) ← Essential reading
- [System Design Interview Vol. 1 & 2 — Alex Xu](https://www.amazon.com/System-Design-Interview-Insiders-Guide/dp/1736049119)
- [The FAANG System Design Interview Handbook](https://www.systemdesignhandbook.com/)

---

*Case Study #10 — Content Streaming Systems | Part of a System Design Case Study Series*  
*For interview prep and learning. Systems described are based on publicly available engineering resources.*

# Design YouTube

> **Difficulty:** Advanced · **Time:** 50–60 minutes

## Problem

Design a video-sharing platform. Users can:

- Upload videos (any resolution, any container).
- Watch videos from any device on any network.
- Search videos by title, description, channel, hashtag.
- Get personalized recommendations.

Scale targets:

- **1B users**
- **500 hours of new video uploaded every minute** (~720k hours/day)
- **1B hours of video watched per day**

## Real-world intuition

YouTube's scale is brutal. A single uploaded 4K video can be 5 GB. Multiply
by 720,000 hours/day and the *upload* side alone moves petabytes daily.
But the watch side is even larger — 1B hours watched at, say, 3 Mbps
average bitrate is roughly **10 exabytes of egress per day**. No origin
infrastructure on Earth can serve that directly.

This means three things drive the entire design:

1. **The CDN *is* the system.** 99%+ of bytes go from edge caches, not from
   YouTube's origin. The interesting problem is which video chunks live on
   which edge.
2. **Encoding is a pipeline, not a step.** A 5 GB upload must be transcoded
   into ~20 variants (240p, 360p, 480p, 720p, 1080p, 1440p, 4K, plus
   different codecs and bitrates for HLS/DASH). Done synchronously, this
   would take hours. Done as a distributed pipeline of GPU/CPU workers, it
   takes minutes for the lower resolutions and is incrementally available.
3. **Watch implies streaming, not file downloads.** Players use adaptive
   bitrate streaming (HLS/DASH) — they download 2–6 second chunks and switch
   quality as bandwidth changes. The architecture must serve **chunks**, not
   files.

Recommendations and view-count are the second layer of interesting problems
on top of that.

## Approach

### Capacity estimation

- **Upload rate:** 500 hours/min × ~1 GB/hour (raw, varying) = **~500 GB/min**
  = **720 TB/day** of raw uploads, **~260 PB/year**.
- **Encoded storage:** ~3× original (multiple resolutions/codecs) ≈
  **~800 PB/year**.
- **Watch egress:** 1B hours/day × 3 Mbps ≈ **~10 EB/day**.
- **Read QPS:** chunk requests at ~6 sec/chunk → roughly **~50M concurrent
  viewers × 1 req / 6 s ≈ 8M chunk req/sec globally** at peak.
- **Search QPS:** millions of queries/sec, served from Elasticsearch
  cluster.

These numbers tell you immediately: object storage + CDN, not a database,
and an asynchronous encoding fleet sized to upload Hz, not to user Hz.

### Data model

```
-- Sharded by video_id
videos(
    video_id PK,        -- snowflake or random
    uploader_id,
    title, description, tags[],
    duration_sec,
    status,             -- UPLOADING, ENCODING, READY, BLOCKED
    created_at,
    s3_original_key,
    available_renditions  -- ["240p","360p",...,"4K"]
)

-- One row per (video_id, rendition); s3_key points to manifest
video_renditions(video_id, rendition, s3_manifest_key, bitrate, codec)

-- Sharded by user_id; partitioned by date in Cassandra
watch_history(user_id, video_id, watched_at, position_sec)

view_counts(video_id, count)   -- aggregated; updated by stream pipeline
```

### Upload + encoding pipeline

```
[Client]                                                       [CDN edges]
   │                                                                ▲
   │ multipart upload                                                │ pull on demand
   ▼                                                                 │
[API GW] ─▶ [Upload Service] ─▶ [S3 (original)]                      │
                  │                                                  │
                  │ emit "video.uploaded"                            │
                  ▼                                                  │
              [Kafka]                                                │
                  │                                                  │
        ┌─────────┴───────────────────────────────────────┐         │
        ▼                              ▼                  ▼          │
[Transcode Workers]            [Thumbnail Workers]  [Content ID]    │
(K8s GPU pool, sharded                                              │
 by video_id)                                                       │
        │                                                            │
        ▼                                                            │
[S3: per-rendition chunks + HLS/DASH manifests]  ───────────────────┘
        │
        │ "video.ready"
        ▼
[Search Indexer] ─▶ [Elasticsearch]
[Recommendation Trainer] (batch ingest)
```

Key points:

- **Chunked transcode.** The original is split into 2-second segments, each
  transcoded independently into all required renditions. Workers can
  parallelize across segments, so a 1-hour upload becomes available in
  minutes, not hours.
- **Incremental availability.** As soon as 240p/360p chunks exist, the video
  is playable. 4K can finish later. This is why "Processing higher
  resolutions" is a UI state, not an outage.
- **Distributed encoding** is K8s jobs or AWS MediaConvert in real systems.

### Streaming + CDN

Adaptive bitrate streaming (HLS or DASH):

```
[Player] ─▶ GET manifest.m3u8 (lists available renditions)
[Player] ─▶ GET chunk_360p_001.ts
        ▶ measures throughput
[Player] ─▶ GET chunk_720p_002.ts   (upgraded)
[Player] ─▶ GET chunk_480p_003.ts   (downgraded after buffer dip)
```

The CDN caches chunks by URL. The interesting twist is **predictive edge
caching**: the recommendation system predicts which videos a region is
likely to want (e.g., a new Mr Beast video in North America) and pre-pushes
the popular renditions to edge POPs before peak.

!!! note "Why this matters"
    A cold cache miss for a viral video at the edge is catastrophic — every
    miss fetches from origin, and 10M concurrent viewers can saturate origin
    egress in seconds. Predictive caching turns the first-watch latency
    spike into a non-event.

### View counts: stream aggregation

A naive `UPDATE video SET views = views + 1` on every play would melt the
DB. Instead:

```
[Player] heartbeat ─▶ [Kafka topic "video.plays"]
                              │
                              ▼
                  [Flink job: dedupe by (video, user, session)]
                              │
                              ▼
                  [Aggregate per 1-min window]
                              │
                              ▼
                  [Write to Cassandra: view_counts]
                              │
                              ▼
                  [Periodically rolled to MySQL aggregates]
```

This also handles fraud (a bot looping a video shouldn't add 1M views) via
deduping on user/session/IP signals.

### Search & recommendations

- **Search:** Elasticsearch shard per language, with metadata + transcribed
  captions indexed. Click-through-rate feeds back as a relevance signal.
- **Recommendations:** Two pipelines.
  - **Batch (Spark):** trains a candidate-generation model on watch history
    overnight, materializes per-user candidate lists into Cassandra.
  - **Real-time (Kafka Streams):** updates the user's "recent" features
    (last 10 watched) and re-ranks the precomputed candidates at request
    time.

### Architecture diagram

```
                   ┌──────────────────────────────────────┐
                   │           CDN (global)               │
                   │  edge POPs hold popular chunks       │
                   └──────┬───────────────────────────────┘
                          │ miss
       ┌──────────────────▼────────────────────┐
       │   S3 / blob storage (renditions)      │
       └───────────────────────────────────────┘

  [Client]──▶ LB ──▶ ┌──────────────┐   ┌─────────────┐  ┌──────────────┐
                     │Video Service │   │ Search      │  │ Recommendation│
                     │(metadata,    │   │ Service     │  │ Service       │
                     │ playback URL)│   │ (ES)        │  │ (Cassandra)   │
                     └──────┬───────┘   └─────────────┘  └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   MySQL      │
                     │ (video meta) │
                     └──────────────┘

  [Upload]──▶ Upload Service ──▶ S3 ──▶ Kafka ──▶ Transcode fleet ──▶ S3 (chunks)
                                              │
                                              ├──▶ Thumbnail workers
                                              ├──▶ Content-ID scan
                                              └──▶ Search indexer

  [Play heartbeats] ──▶ Kafka ──▶ Flink ──▶ Cassandra view_counts
```

## Trade-offs

- **HLS vs. DASH.** HLS is Apple-blessed and universal on iOS; DASH is the
  open standard with finer-grained control. Most platforms ship both with a
  shared chunk encoder.
- **Pre-encode all renditions vs. on-demand.** Pre-encoding wastes work on
  videos that never get views (90%+ of uploads). Some platforms encode only
  240p/720p eagerly, the rest lazily on first request. Cost vs. first-watch
  latency.
- **Push to edge vs. let CDN fill.** Predictive push saves catastrophic
  origin spikes for viral content, but wastes bandwidth on guesses. Effective
  recommender → effective edge cache.
- **Strong vs. eventual view counts.** Eventually consistent. A "12,345
  views" that lags by 30 seconds is invisible; a synchronous DB write per
  play is impossible.
- **Storing originals.** Keeping the original 4K for years costs a fortune.
  Some systems delete originals after N days once all renditions are
  verified, accepting that re-encoding (e.g., to a new codec) requires
  re-upload.

## Edge cases & gotchas

- **Upload resumability.** Multi-GB uploads on flaky connections must
  resume. Use S3 multipart upload with the client tracking part numbers.
- **Live streaming** uses a different pipeline (low-latency HLS / WebRTC).
  Treat as a separate sub-system; do not retrofit into the VOD encoder.
- **Content-ID / copyright scanning.** Run async on upload; videos with
  matches get monetization split or block based on rules.
- **Geographic blocking.** Per-country availability is enforced at the CDN
  layer via signed URLs and edge logic.
- **Resume playback.** Watch position must persist across devices.
  Cassandra `watch_history` partitioned by `user_id` makes this a single
  read.
- **Thumbnail generation.** Storyboard thumbnails (the preview when you
  scrub) are generated by sampling frames during transcode. Cache as JPEG
  sprites.
- **Hot videos.** Surge after a celebrity tweet. CDN absorbs most of it; if
  origin breaks anyway, fall back to lower-resolution renditions.
- **Codec churn.** AV1 / VP9 / H.264 — keeping multiple codec families
  doubles storage. Pick based on device share.

!!! warning
    Almost every interview pitfall in YouTube comes from underestimating
    encoding cost. If you treat transcode as a synchronous step, the entire
    timeline of the system collapses. Always answer "uploads are pushed to
    a queue and transcoded in parallel by a worker fleet."

## Linked concepts

- [CDN](../fundamentals/cdn.md) — the load-bearing piece for video delivery
- [Caching strategies](../fundamentals/caching.md) — predictive edge caching
- [Message queues](../fundamentals/message-queues.md) — Kafka for encoding + view events
- [Data processing](../fundamentals/data-processing.md) — Flink for views, Spark for recs
- [NoSQL (Cassandra)](../fundamentals/databases/nosql.md) — watch history, view counts
- [Search & indexing](../fundamentals/search-indexing.md) — Elasticsearch for video search
- [Microservices](../fundamentals/microservices.md) — upload, transcode, playback split
- [Database scaling](../fundamentals/databases/scaling.md) — sharding video metadata

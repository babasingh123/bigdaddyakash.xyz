# Design Instagram

> **Difficulty:** Intermediate · **Time:** 40–50 minutes

## Problem

Design a photo-sharing social network. Users can:

- Upload photos with captions.
- Follow other users.
- Open a "home feed" of recent photos from people they follow
  (chronological or ranked).
- Like and comment on photos.

Scale targets:

- **500M daily active users**
- **100M photos uploaded per day**
- **Average photo size 2 MB**

## Real-world intuition

Three numbers shape every architectural decision:

1. **100M photos × 2 MB = 200 TB of new image bytes every day.**
   You do not put that in a relational database. It goes in S3 (or
   equivalent), and the database stores only pointers. The CDN handles the
   delivery so origin bandwidth stays sane.
2. **The home feed is the read.** A user opens the app and expects ~30 fresh
   photos in <200 ms. Computing that feed on demand by joining
   `follows × photos ORDER BY created_at` for a user with 500 followees is a
   query you cannot afford at 500M DAU.
3. **The follower count distribution is power-law.** A typical user has
   ~200 followers. A celebrity has 100M. If you "fan out" every post to
   every follower's feed, a celebrity posting once means 100M database
   writes — impossible in real time, and also a giant waste because most of
   those followers will never open the app today.

The hybrid fan-out architecture (push for the long tail, pull for celebrities)
exists precisely because of this distribution. It's the canonical reason
"celebrity problem" is a phrase in system design.

## Approach

We design the upload path, the feed path, and the celebrity escape hatch
separately.

### Capacity estimation

- **Photo writes:** 100M/day ÷ 86,400 ≈ **1,200 writes/sec** (avg), **~5,000/sec peak**.
- **Photo storage:** 100M × 2 MB = **200 TB/day** → **~75 PB/year** of raw originals.
  Multiple encoded sizes (thumb, medium, full) ≈ 1.5× → ~110 PB/year.
- **Feed reads:** 500M DAU × ~5 feed refreshes = **2.5B feed reads/day** ≈
  **30k feed reads/sec**.
- **Fan-out work:** if average follower count is 200, each post produces 200
  feed inserts ⇒ ~240k feed inserts/sec on a pure push model. A celebrity
  blows this up by orders of magnitude.

### Data model

```sql
-- Sharded by user_id
users(user_id PK, username UNIQUE, profile_pic_s3, follower_count, ...)

-- Sharded by photo_id
photos(photo_id PK, user_id, s3_keys (thumb,med,full), caption, created_at)

-- Sharded by follower_id (we read "who do I follow" much more than "who follows me")
follows(follower_id, followee_id, created_at, PRIMARY KEY(follower_id, followee_id))

-- Counters; often stored in Redis with periodic flush to DB
likes(photo_id, user_id, created_at, PRIMARY KEY(photo_id, user_id))
comments(comment_id PK, photo_id, user_id, body, created_at)

-- Feed table (Cassandra)
-- Partition key: user_id, clustering: created_at DESC
feed(user_id, created_at, photo_id, author_id)
```

Cassandra is a natural fit for `feed`: writes are append-only, queries are
always "give me the top N rows for a partition key in time order," and
partitions stay reasonably bounded by feed-size trimming.

### Upload pipeline

```
[Client]
   │ POST /upload (multipart)
   ▼
[API Gateway] ──▶ [Photo Service] ──▶ [S3 (original)]
                       │
                       │  emit "photo_uploaded" event
                       ▼
                   [Kafka]
                       │
              ┌────────┴───────────────────────────────┐
              ▼                ▼                       ▼
       [Resize Workers]  [Feed Fan-out]        [Search Indexer]
              │                │                       │
              ▼                ▼                       ▼
        [S3 thumb/med]  [Cassandra feed]       [Elasticsearch]
              │
              ▼
           [CDN]
```

The response to the client returns as soon as the original is in S3 and the
DB row is written. Resizing and fan-out happen asynchronously — the user
sees their own photo via "read-your-writes" by reading from a dedicated
"my photos" path, not the home feed.

!!! tip "Why async resize?"
    Synchronous resize means a slow ImageMagick worker stalls the user's
    upload response. Async lets the API return in ~200 ms; the worker fleet
    can autoscale on Kafka lag.

### Feed generation: push vs. pull

#### Push (fan-out on write)

When user A posts photo P, for every follower F we write a row into F's
Cassandra feed partition. Reads are then a single partition scan.

- **Pros:** O(1) read; feeds are pre-materialized.
- **Cons:** O(followers) writes per post; celebrities are catastrophic.

#### Pull (fan-out on read)

Store only `photos` and `follows`. When F opens the feed, fetch followees,
then top-K photos from each, then merge.

- **Pros:** No write amplification.
- **Cons:** Every feed view is a fan-out read; users with 500 followees
  trigger 500 lookups.

#### Hybrid (the real answer)

- **Push** for normal users (<10k followers).
- **Pull** for celebrities. Their posts are not pre-materialized; the feed
  service merges them in at read time.
- A user's home feed = `(pre-materialized push entries) ∪ (just-in-time pull
  from any celebrities they follow)`, merged by timestamp.

The celebrity threshold (10k? 100k?) is a tuning knob — set it where the
push cost per post equals the marginal pull cost for their followers'
feed views.

!!! note "Why this matters"
    A 100M-follower celebrity posting 5 times a day on pure push = 500M
    extra writes/day, all bunched around their posting times. On hybrid,
    those are zero writes and a handful of extra reads merged into each
    follower's feed.

### Architecture diagram

```
                         ┌───────────────────────┐
                         │         CDN           │
                         │  (thumbs, full images)│
                         └───────────┬───────────┘
                                     │
        ┌──────────────┐      ┌──────▼──────┐    ┌─────────────┐
        │   Clients    │◀────▶│Load Balancer│    │   S3        │
        └──────────────┘      └──────┬──────┘    │ (originals  │
                                     │           │  + resizes) │
                       ┌─────────────┼───────────┴─────────────┘
                       ▼             ▼                  ▲
                ┌────────────┐ ┌────────────┐   uploads │
                │ Feed       │ │ Photo      │───────────┘
                │ Service    │ │ Service    │
                └────┬───────┘ └────┬───────┘
                     │              │ emits
                     │              ▼
                     │         ┌──────────┐
                     │         │  Kafka   │
                     │         └────┬─────┘
                     │              │
                     │   ┌──────────┴────────────┐
                     ▼   ▼                       ▼
              ┌──────────────┐          ┌───────────────┐
              │  Cassandra   │          │  Resize       │
              │  feed table  │◀─────────│  Workers      │
              │ (push fanout)│          └───────────────┘
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │ Redis cache  │
              │ (top of feed)│
              └──────────────┘
```

## Trade-offs

- **Push vs. pull vs. hybrid.** Push optimizes reads; pull optimizes writes;
  hybrid optimizes both at the cost of more code paths. At Instagram scale,
  hybrid is non-negotiable.
- **Strong vs. eventual consistency.** Like counts, follower counts, and
  feed inclusion are eventually consistent. A like that takes 2 seconds to
  reflect on another user's screen is fine. This unlocks Cassandra/Redis
  counters instead of synchronous SQL increments.
- **Chronological vs. ranked feed.** Pure chronological is cheaper (just
  sort by timestamp). Ranked needs a scoring pipeline (ML features, recency,
  affinity) and roughly doubles read latency. Most platforms moved to ranked
  for engagement reasons, accepting the cost.
- **Pre-resize all sizes vs. on-demand.** Pre-resizing wastes storage on
  photos no one views. On-demand resizing adds first-view latency. Hybrid:
  pre-resize thumb + medium (cheap), generate larger sizes on first request
  and cache.

## Edge cases & gotchas

- **The celebrity problem.** Already covered — hybrid fan-out is the fix.
- **Inactive followers.** If 80% of a user's followers are inactive, pushing
  to them is wasted work. Some designs push only to *recently active*
  followers and lazily reconstruct the feed for the rest.
- **Hot photo / hot user.** Cache the photo metadata and likes count
  aggressively. A single Beyoncé post will dominate Redis traffic.
- **Replication lag.** "I just posted but my feed doesn't show it" — fix
  by reading the user's own posts from the primary or by injecting them
  client-side on submit.
- **Image moderation.** Async, after upload. Block on CDN propagation if a
  match is found. Keep a separate "shadow" S3 bucket for content removed
  by moderation.
- **Photo deletion / GDPR.** Hard to undo a push. Either tombstone the
  photo_id in feed rows and filter on read, or accept that deleted photos
  may briefly appear in stale caches.
- **Bot abuse.** A new account that follows 5,000 people in an hour
  triggers a fan-out storm into 5,000 home feeds. Rate-limit follows per
  account.

## Linked concepts

- [Database scaling](../fundamentals/databases/scaling.md) — sharding users, photos, feeds
- [NoSQL (Cassandra)](../fundamentals/databases/nosql.md) — column-family timelines
- [Caching strategies](../fundamentals/caching.md) — hot feeds, hot photos
- [CDN](../fundamentals/cdn.md) — global image delivery
- [Message queues](../fundamentals/message-queues.md) — Kafka for fan-out and resize
- [Distributed systems](../fundamentals/distributed-systems.md) — CAP, eventual consistency
- [Search & indexing](../fundamentals/search-indexing.md) — Elasticsearch for hashtag search

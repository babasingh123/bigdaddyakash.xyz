# Design Twitter

> **Difficulty:** Intermediate · **Time:** 40–50 minutes

## Problem

Design a microblogging platform. Users can:

- Post short tweets (≤ 280 characters).
- Follow other users.
- View a chronological home timeline of people they follow.
- Like, retweet, and reply.
- See trending topics and search tweets.

Scale targets:

- **200M daily active users**
- **400M tweets per day**

## Real-world intuition

Twitter is Instagram's sibling, with two extra problems:

1. **Tweets are tiny but numerous.** 400M × ~300 bytes ≈ 120 GB/day — small.
   But tweets-per-second peaks (Super Bowl, election night) hit 100k+
   inserts/sec in narrow windows. The system has to absorb bursts cleanly.
2. **Trending and search are first-class.** Unlike Instagram, the home
   timeline is not the only consumption surface — millions of users land on
   `#WorldCupFinal` or search "Apple earnings" and expect a fresh, ranked
   list within seconds of an event.

So the architecture has three big movers: a fan-out system (like Instagram),
a real-time stream-processing pipeline (for trending), and a full-text
search index (for the search box). The "celebrity problem" is even more
extreme here — Cristiano Ronaldo has ~600M followers.

Every key decision in this system is downstream of two truths: **reads
massively outnumber writes**, and **the follower distribution is brutally
power-law**.

## Approach

### Capacity estimation

- **Tweet writes:** 400M/day ≈ **4,600 writes/sec** avg, **150k+ writes/sec peak**.
- **Timeline reads:** 200M DAU × ~20 refreshes = **4B reads/day** ≈ **45k reads/sec** avg.
- **Tweet storage (text + metadata):** 400M × 1 KB = **400 GB/day** ≈ **150 TB/year**.
- **Hot tweets** (last 7 days, where 90%+ of reads land) ≈ **3 TB** — fits
  in a Redis/Cassandra hot tier with replicas.
- **Fan-out write amplification:** average ~700 followers → ~3M timeline
  inserts/sec avg, with celebrity spikes that we explicitly avoid via hybrid.

### Data model

```
-- Sharded by user_id
users(user_id PK, handle UNIQUE, follower_count, ...)

-- Sharded by tweet_id (snowflake; sortable by time)
tweets(tweet_id PK, user_id, body, created_at, reply_to, retweet_of, lang)

-- Sharded by follower_id
follows(follower_id, followee_id, created_at)

-- Cassandra: partition by user_id, clustering by created_at DESC
home_timeline(user_id, created_at, tweet_id, author_id)

-- Elasticsearch
tweet_search_index(tweet_id, body, hashtags[], user_id, lang, created_at)
```

`tweet_id` should be a Snowflake ID — 64 bits encoding timestamp + machine +
sequence. This gives globally unique, roughly-sortable IDs without a central
counter.

### Tweet storage: hot/cold split

```
[Tweet write]
   │
   ▼
[Tweet Service] ──▶ [Cassandra hot tier (last 30 days)]
   │
   └─▶ async ──▶ [HDFS / cold storage (older tweets)]
                 [Elasticsearch (search)]
                 [Kafka topic "tweets" (downstream consumers)]
```

- **Hot tier (Cassandra/Redis):** last 30 days, served from RAM/SSD.
- **Cold tier (HDFS/S3):** everything older, fetched on demand. Most user
  reads never touch it.

### Home timeline: hybrid fan-out

Same pattern as Instagram, more aggressively tuned:

- **Push to followers' timelines** for users with < ~20k followers.
- **Pull at read time** for celebrities. The user's timeline-read path does:

```
home = (push entries from cassandra)
     ∪ (just-in-time pull from each celebrity I follow)
     sorted by created_at DESC
     limit 50
```

The "celebrities I follow" set is small (every user follows maybe 5–20
celebrities), so the pull cost is bounded.

!!! note "Why hybrid is non-optional at this scale"
    Pure push: a Ronaldo tweet = 600M fanout writes. Even spread across a
    Cassandra cluster, that takes minutes and blows your write budget. Pure
    pull: every timeline open does 700 lookups. Hybrid is the only design
    that lives.

### Trending topics: stream processing

```
[Kafka "tweets" topic]
        │
        ▼
[Flink / Kafka Streams job]
   - tokenize, extract hashtags + entities
   - count per (region, hashtag) in sliding 5-min window
   - dedupe per-user (prevent gaming)
        │
        ▼
[Redis sorted set: trending:{region} → ZSET of (score, hashtag)]
        │
        ▼
[Trending API] reads top-K from sorted set
```

The sliding window is the key insight: a hashtag is "trending" because its
*rate* spikes, not its total count. A 5-minute window of counts compared
against a baseline catches breaking news within seconds.

### Search

Tweets are indexed in Elasticsearch within a few seconds of publication.
Indexing is async (Kafka → indexer workers) so write latency is unaffected.
Search queries hit ES with relevance ranking (BM25) plus engagement signals.

### Architecture diagram

```
                        ┌──────────────────────────┐
                        │       CDN / Edge         │
                        └────────────┬─────────────┘
                                     │
                              ┌──────▼──────┐
                              │     LB      │
                              └──────┬──────┘
                                     │
            ┌────────────┬───────────┼───────────┬─────────────┐
            ▼            ▼           ▼           ▼             ▼
       ┌────────┐  ┌──────────┐ ┌────────┐ ┌─────────┐  ┌────────────┐
       │Tweet   │  │Timeline  │ │Search  │ │Trending │  │User/Follow │
       │Service │  │Service   │ │Service │ │Service  │  │Service     │
       └───┬────┘  └────┬─────┘ └───┬────┘ └────┬────┘  └─────┬──────┘
           │            │           │           │              │
           │ emits      │           │           │              │
           ▼            ▼           ▼           ▼              ▼
       ┌────────┐  ┌────────────────────────────────────┐ ┌────────┐
       │ Kafka  │  │   Cassandra (tweets + timelines)   │ │ MySQL  │
       │tweets  │  └────────────────────────────────────┘ │(users) │
       └───┬────┘                                         └────────┘
           │
   ┌───────┴────────┬─────────────┬──────────────┐
   ▼                ▼             ▼              ▼
┌───────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐
│ Fan-out   │  │  Search  │  │ Trending │  │ Analytics  │
│ Workers   │  │ Indexer  │  │  (Flink) │  │ / ML       │
└─────┬─────┘  └────┬─────┘  └────┬─────┘  └────────────┘
      │             │             │
      ▼             ▼             ▼
 [home_timeline] [Elasticsearch] [Redis ZSET]
```

## Trade-offs

- **Hybrid fan-out vs. pure pull.** Pure pull is simpler but adds latency
  proportional to followee count. Hybrid is more code but bounds read
  latency.
- **Chronological vs. ranked timeline.** Chronological is cheap and
  predictable; ranked needs an ML pipeline and roughly doubles read latency
  but improves engagement. Twitter ships both.
- **Cassandra vs. relational for timelines.** Cassandra wins on write
  throughput and partition-key access patterns, loses on ad-hoc queries
  (you cannot join). The home-timeline pattern doesn't need joins, so
  Cassandra is the right shape.
- **Real-time vs. batch trending.** Real-time (Flink) catches news fast but
  is noisy (bots, brigading). Batch (hourly Spark job) is more accurate.
  Most platforms combine: stream for the moment, batch for the daily summary.
- **At-least-once vs. exactly-once Kafka delivery.** At-least-once is the
  norm and forces consumers to be idempotent (use `tweet_id` as a natural
  dedup key in fan-out workers).

## Edge cases & gotchas

- **Celebrity tweets.** Solved via hybrid. Also: rate-limit the read-time
  pull so a celebrity-heavy user doesn't generate a thundering herd against
  the celebrity's tweet shard.
- **Retweets multiply fan-out.** A retweet by a celebrity is just as bad as
  an original. Treat retweets identically in the hybrid policy.
- **Replies aren't timeline items by default.** They appear in conversation
  view, but only in followers' timelines if both parties are followed.
  Filtering at fan-out time is cheaper than filtering at read time.
- **Trending abuse.** Bot networks pump fake trends. Mitigations: per-user
  dedupe, IP/device clustering, ML-based anomaly detection on participation
  graphs.
- **Eventually-consistent counters.** Like/retweet counts are written to
  Redis HLL/counters and flushed periodically. Showing "12.4K likes" is
  fine even if it lags by a few seconds.
- **Tombstoning deletes.** A deleted tweet may already be in millions of
  timeline rows. Tombstone the `tweet_id`, filter on read, and let Cassandra
  compaction reclaim space.
- **Locality / sharding by region.** Trending lists are region-scoped, so
  the trending Flink job partitions by region; tweet storage stays globally
  sharded by `tweet_id`.

!!! tip "Interview pacing"
    Twitter is best answered as "Instagram + trending + search." Spend the
    first 5 min on capacity, 10 min on hybrid fan-out (referencing the
    celebrity problem by name), then split the remaining time between the
    trending pipeline and search. Don't try to cover everything.

## Linked concepts

- [Database scaling](../fundamentals/databases/scaling.md) — sharding users, tweets, timelines
- [NoSQL (Cassandra)](../fundamentals/databases/nosql.md) — column-family for timelines
- [Caching strategies](../fundamentals/caching.md) — hot tweets, hot timelines
- [Message queues](../fundamentals/message-queues.md) — Kafka for fan-out, indexing, trending
- [Data processing](../fundamentals/data-processing.md) — Flink stream processing for trending
- [Search & indexing](../fundamentals/search-indexing.md) — Elasticsearch full-text search
- [Distributed systems](../fundamentals/distributed-systems.md) — eventual consistency, CAP

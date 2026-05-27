# Design a URL Shortener

> **Difficulty:** Beginner · **Time:** 30–40 minutes

## Problem

Design a service like bit.ly that turns a long URL such as
`https://example.com/very/long/path?with=params` into a compact 7-character
short URL like `bit.ly/aZ3kP9q`. The service must:

- Generate a unique short code per URL (7 characters from a 62-symbol alphabet).
- Redirect the short URL back to the original long URL with low latency.
- Handle **100M new URLs per month** (~40 writes/sec).
- Handle reads that are **~100× writes** (~4,000 reads/sec, with peaks much higher).
- Treat URLs as permanent (no expiration in the base design).

## Real-world intuition

Short links are the plumbing of the internet — every tweet, marketing email,
QR code, and Slack message is full of them. From the outside this looks like a
toy problem: "just store a row in a database." The interesting design questions
all come from two facts:

1. **Reads dwarf writes.** Every link is created once and clicked thousands of
   times. A naive design that hits MySQL on every redirect will saturate the
   database long before it saturates network or CPU.
2. **The short code is a primary key.** Two simultaneous requests must never
   collide on the same 7-character code, and the code must stay stable forever.

Real services (bit.ly, t.co, goo.gl in its time) live or die by p99 redirect
latency. A 300ms redirect breaks the illusion that the short link "just works,"
so the entire architecture is bent toward making the read path a single cache
hop plus an HTTP 301.

## Approach

We walk through the design read-path-first, because that is where the load is.

### 1. Capacity estimation

Run the numbers before picking technologies.

- **Writes:** 100M URLs/month ÷ (30 × 86400) ≈ **40 writes/sec** average,
  ~120/sec at peak.
- **Reads:** 100× writes → **~4,000 reads/sec** average, ~12k/sec at peak.
- **Storage:** 100M × ~500 B/row (URL + metadata) ≈ **50 GB/month**, **600 GB/year**.
  Five-year horizon: ~3 TB — fits in a sharded MySQL/Postgres cluster comfortably.
- **Cache working set:** If the top 20% of links serve 80% of clicks
  (the usual Pareto), we only need to keep ~20M entries hot →
  20M × 600 B ≈ **12 GB** in Redis. One node can hold this; two nodes
  for HA.
- **Keyspace:** $62^7 \approx 3.5 \times 10^{12}$ — more than enough headroom
  even if we burn codes for 50 years at this rate.

### 2. Generating the short code

Three approaches, each with a different failure mode:

| Strategy                                 | Pros                          | Cons                                                          |
| ---------------------------------------- | ----------------------------- | ------------------------------------------------------------- |
| **Hash long URL (MD5/SHA), take 7 chars** | Idempotent, no coordination   | Collisions; same URL always maps to same code (privacy leak) |
| **Base62 encode auto-increment ID**       | No collisions, dense keyspace | Predictable codes (scraping); single counter is a bottleneck |
| **Random 7-char string + collision check** | Unguessable, simple to reason about | Wasted DB lookups as the keyspace fills              |

The pragmatic answer at this scale is a **pre-generated key pool**: a separate
"key-generation service" picks random or counter-derived 7-char codes in
batches of, say, 1M, writes them to a `keys_unused` table, and the URL service
just `SELECT … LIMIT 1 FOR UPDATE SKIP LOCKED` to grab a fresh code. This
removes collision checks from the hot path.

!!! note "Why this matters"
    At 120 writes/sec, even a 1% collision rate (1 extra DB query per
    collision) means 1.2 wasted writes/sec — fine. At 12,000 writes/sec
    that becomes 120 wasted writes/sec, which doubles your write traffic for
    no reason. The pre-generated pool moves the variability off the hot path.

### 3. Data model

```sql
-- Primary table (sharded by short_code hash)
CREATE TABLE urls (
    short_code   VARCHAR(7)  PRIMARY KEY,
    long_url     TEXT        NOT NULL,
    user_id      BIGINT,
    created_at   TIMESTAMP   NOT NULL DEFAULT NOW()
);

-- Pre-generated key pool (single shard, written by key-gen service)
CREATE TABLE keys_unused (
    short_code   VARCHAR(7)  PRIMARY KEY
);
```

The primary key *is* the short code, so the natural index works for both
inserts and the redirect lookup. No secondary index is needed on the read
path.

### 4. Read path: cache first, DB second

```
GET /aZ3kP9q
        │
        ▼
[Cache lookup (Redis)] ── HIT ──▶ HTTP 301 → long_url   ◀── ~1 ms
        │
        MISS
        ▼
[urls table by PK]    ── HIT ──▶ write to cache, then HTTP 301
        │
        MISS
        ▼
HTTP 404
```

We use **cache-aside** (lazy loading): the application reads from Redis first,
falls back to MySQL on miss, then populates Redis with a 24-hour TTL. Hot
codes naturally stay resident. We do **not** use write-through here — most
short codes are clicked zero times, and caching them on creation would pollute
Redis with cold entries.

### 5. Architecture diagram

```
                ┌────────────────────────────┐
                │       CDN (optional)       │
                │  Caches 301 responses for  │
                │   very-popular short codes │
                └─────────────┬──────────────┘
                              │
                       ┌──────▼──────┐
                       │   Layer-7   │
                       │ Load Balancer│
                       └──────┬──────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
       ┌─────────┐       ┌─────────┐       ┌─────────┐
       │  API 1  │       │  API 2  │  ...  │  API N  │
       └────┬────┘       └────┬────┘       └────┬────┘
            │                 │                 │
            └────────┬────────┴────────┬────────┘
                     ▼                 ▼
              ┌────────────┐    ┌────────────┐
              │   Redis    │    │ Key-Gen    │
              │  (hot URLs)│    │  Service   │
              └─────┬──────┘    └─────┬──────┘
                    │ miss            │ inserts
                    ▼                 ▼
              ┌─────────────────────────────┐
              │  MySQL/Postgres (sharded)   │
              │  shard key = hash(short)    │
              └─────────────────────────────┘
```

Sharding by `hash(short_code)` distributes both reads and writes evenly.
Range sharding would create hotspots if codes are sequential.

## Trade-offs

- **Strong vs. eventual consistency.** Redirects only need eventual consistency
  — if a brand-new link takes 100 ms to propagate to a read replica, no human
  notices. We use a primary for writes and read replicas for the cache-miss
  path, accepting milliseconds of replication lag.
- **Cache TTL vs. freshness.** A 24-hour TTL is a compromise. Shorter TTLs
  thrash the cache; longer TTLs make link-deletion appear delayed. Since we
  don't allow deletion in the base design, longer is better.
- **Pre-generated keys vs. on-demand.** Pre-generation needs a separate
  service and table, but eliminates collision races on the hot path. For
  smaller scale (under ~50 writes/sec), random + retry is simpler.
- **Predictable vs. unpredictable codes.** Base62-of-counter is predictable
  (anyone can enumerate your URLs). Random codes are not, at the cost of a
  collision-check round-trip. Most public shorteners use random codes for
  exactly this reason.

## Edge cases & gotchas

- **Custom aliases.** If users can pick their own short code, you need a
  unique-check on write and a way to reserve names. Don't let `keys_unused`
  hand out a code that someone is about to claim manually.
- **Idempotent creation.** A client retrying a `POST /shorten` should not
  create two short codes for the same long URL. Either dedupe by
  `hash(long_url)` (extra index) or accept that retries create duplicates.
- **Malicious URLs.** Shorteners are spam magnets. Add a Safe-Browsing /
  URLhaus check on creation and a 410 Gone status for revoked links.
- **Hot keys.** A celebrity's link can take 100k QPS by itself. Redis can
  handle this, but put the response on a CDN with a 60-second TTL for
  extreme cases.
- **Sharding pain.** Re-sharding MySQL is expensive. Start with more shards
  than you need (e.g., 64 logical shards mapped to 4 physical instances) so
  growth is just moving shards, not splitting them.
- **Analytics fan-out.** Counting clicks should not block the redirect. Emit
  a Kafka event ("click {short_code, ts, ip, ua}") and aggregate
  asynchronously.

!!! tip
    For interview pacing: spend ~5 min on capacity, ~10 min on key-gen
    strategy, ~10 min on read path & caching, then trade-offs. The
    interviewer almost always wants to dig into "what if a link goes
    viral?" — so keep the hot-key answer ready.

## Linked concepts

- [SQL databases](../fundamentals/databases/sql.md) — primary store and indexing
- [Database scaling](../fundamentals/databases/scaling.md) — sharding by short code, read replicas
- [Caching strategies](../fundamentals/caching.md) — cache-aside, TTLs, hot keys
- [Load balancing](../fundamentals/load-balancing.md) — distributing redirects across API hosts
- [Rate limiting](../fundamentals/rate-limiting.md) — protecting `POST /shorten` from abuse
- [API design](../fundamentals/api-design.md) — REST endpoints, status codes (301 vs 302)
- [CDN](../fundamentals/cdn.md) — pushing 301 responses to the edge for viral links

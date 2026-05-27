# Caching

A cache is a small, fast store placed in front of a slow store, holding the data we're most likely to need next. Caching is the single highest-leverage technique in HLD: a well-placed cache routinely turns a database that would have collapsed into one that hums along at a fraction of its potential load.

## Why it exists

Databases are designed to be **correct and durable**, not fast. They persist to disk, enforce constraints, replicate writes, hold locks. Even a cheap query — "fetch user by id" — costs the database a network hop, parsing, planning, an index seek, and a serialization back across the wire. Microseconds, but at a million QPS those microseconds add up to a herd of database servers you don't want to pay for.

A cache replaces that round trip with an in-memory lookup. Redis can answer a key lookup in tens of microseconds. A CPU L1 cache answers in nanoseconds. The closer you cache to the consumer, the faster — and the less load reaches the origin.

The economic argument: real workloads follow a heavy power law. 80% of requests typically go to 20% of the data, and often 1% of the data takes 50% of the traffic. Caching that hot slice in RAM is much cheaper than scaling the database to absorb the same traffic at disk speed.

## How it works

There are four canonical patterns. They differ in *who* writes to the cache and *when*.

### Cache-Aside (Lazy Loading)

```
1. App checks cache for key
2. If hit, return value
3. If miss, app reads from DB
4. App writes value to cache
5. Return data
```

The application owns the cache. The cache only holds data that's actually been requested.

- **Pros**: Only caches what's needed. Simple to reason about. Works with any cache and any DB.
- **Cons**: The first request for any key always pays the DB cost. Stale data possible if you don't invalidate on writes.
- **Use**: The most common pattern by a wide margin. Default choice unless you have a specific reason to do something else.

### Write-Through

```
1. App writes to cache
2. Cache writes synchronously to DB
3. Return success
```

Every write goes to the cache, and the cache writes through to the DB before acknowledging. Reads then always hit a populated cache.

- **Pros**: Cache and DB are always in sync. Reads after writes always hit cache.
- **Cons**: Every write pays two round trips. The cache must be smart enough to write to the DB (or you build that logic).
- **Use**: When consistency between cache and DB matters and writes aren't the bottleneck.

### Write-Back (Write-Behind)

```
1. App writes to cache
2. Cache returns success immediately
3. Cache asynchronously persists to DB later
```

Writes are absorbed by the cache and batched/flushed to the DB in the background.

- **Pros**: Very fast writes. The DB sees a smoothed write rate.
- **Cons**: If the cache fails before flushing, you lose data. Adds operational complexity (durable buffer, replay).
- **Use**: High write throughput where some data loss risk is tolerable (metrics, view counts, telemetry).

### Write-Around

```
1. App writes only to DB
2. Cache populated on first read (cache-aside)
```

Writes skip the cache entirely. The cache only learns about a key when someone reads it.

- **Pros**: Avoids polluting the cache with write-only data (e.g. logs that won't be read again).
- **Cons**: First read after a write is always a miss.
- **Use**: Write-heavy workloads where most written data isn't read soon (audit logs, telemetry sinks).

### Eviction policies

A cache is finite. When it fills up, you have to evict something. The policy decides who.

| Policy | Rule | Best for |
|--------|------|----------|
| **LRU** (Least Recently Used) | Evict the item that hasn't been accessed for the longest time. | General-purpose default. Recent activity is the best predictor of future activity. |
| **LFU** (Least Frequently Used) | Evict the item with the lowest access frequency. | When access frequency matters more than recency (e.g. trending content). |
| **FIFO** (First In, First Out) | Evict the oldest inserted item, regardless of access. | Simple, predictable. Rarely the best choice. |
| **TTL** (Time To Live) | Items expire after a fixed time, evicted lazily or proactively. | Time-bounded data: sessions, tokens, search suggestions. |

Real caches usually combine policies — Redis defaults to `allkeys-lru` with optional TTL on individual keys, for instance.

### Cache layers

Caching is a layered defense. A request can hit a cache at any of these layers before reaching the database:

| Layer | Where it lives | Typical hit cost | Stores |
|-------|----------------|------------------|--------|
| **Browser cache** | User's device | ~0 (no network) | HTML, JS, CSS, images |
| **CDN** | Edge POPs near user | 10–50 ms | Static assets, video, cacheable API responses |
| **API gateway / reverse proxy cache** | In front of services | 1–5 ms | Cacheable HTTP responses |
| **Application cache** | In-process (LRU) or Redis/Memcached | < 1 ms | DB query results, computed objects, sessions |
| **Database cache** | Buffer pool inside the DB | < 1 ms | Hot rows and index pages |

A multi-layer request flow looks like:

```
Request → Browser Cache → CDN → App Cache → DB Cache → Database
```

Each layer that handles the request saves the layers below from doing work.

### Caching tools

- **Redis** — in-memory store with rich data types (strings, lists, sets, sorted sets, hashes, streams), persistence options, replication, pub/sub, transactions, Lua scripting. The Swiss-army knife of caches.
- **Memcached** — pure key-value, simpler and slightly faster than Redis for plain string caching. No persistence, no data structures.
- **CDN** — CloudFlare, Akamai, CloudFront, Fastly. See [CDN](cdn.md).
- **In-process caches** — Caffeine (Java), Guava Cache (Java), `lru_cache` (Python). Sub-microsecond, but per-process — duplicated work across your fleet.

| Use case | Recommended tool |
|----------|------------------|
| Session storage, hot rows | Redis |
| Leaderboards, ranked lists | Redis (sorted sets) |
| Rate limiting counters | Redis (`INCR` + `EXPIRE`) |
| Pub/sub between services | Redis or Kafka, depending on durability needs |
| Static content delivery | CDN |
| Plain LRU on a fleet | Memcached |
| Per-request memoization | In-process cache |

### Cache problems and solutions

Three classic failure modes, all worth knowing for interviews.

#### Cache stampede (thundering herd)

**Problem**: A popular key expires. The next thousand requests all miss the cache and stampede the database. The DB falls over, every request times out, the cache stays empty, the herd keeps stampeding.

**Solutions**:

- **Lock / single-flight**: only one request refreshes the key; the others wait for that single fetch.
- **Probabilistic early expiration**: start refreshing the key *before* it actually expires, with a probability that grows as expiration approaches. (See "XFetch" algorithm.)
- **Async refresh**: a background job pre-warms hot keys before they expire.

#### Cache penetration

**Problem**: Repeated requests for keys that don't exist in the DB. Every one bypasses the cache and hits the DB. An attacker can weaponize this to overwhelm your database.

**Solutions**:

- **Cache negative results**: store a sentinel value ("does not exist") with a short TTL.
- **Bloom filter**: a small probabilistic structure that knows whether a key *might* exist. See [Misc Patterns](misc-patterns.md). If the bloom filter says "no", skip the cache *and* the DB.

#### Cache avalanche

**Problem**: A large batch of keys expires at the same instant (e.g. you warmed the cache at startup with the same TTL on everything), and the DB gets a synchronized flood of misses.

**Solutions**:

- **Randomize TTLs**: add jitter to expirations (`ttl = base + random(0, jitter)`).
- **Cache warming**: pre-populate the cache before traffic arrives, with staggered TTLs.
- **Tiered caches**: a longer-TTL secondary layer (or "stale-while-revalidate" semantics) so requests don't fall through to the DB during the refresh window.

!!! warning "Caches are not a correctness layer"
    The cache can — and will — disagree with the database briefly. Design your application so this is invisible to users. Never use a cache where a stale read would cause a correctness bug (e.g. authorization decisions, money balances at the moment of charge).

## When to use

- **Reads >> writes.** The cache pays its rent on the read side. If writes dominate, the cache adds latency without saving DB load.
- **Power-law access patterns.** A small fraction of data takes most of the traffic. Caching the hot set hides the long tail.
- **Tolerable staleness.** A few seconds (or minutes) of staleness is acceptable — or you're disciplined about invalidating on writes.
- **Expensive computations** that can be reused — rendered HTML, search result pages, ML predictions.
- **Rate limiting / quotas.** Redis is the standard tool for distributed counters with atomic increment + TTL.

Skip caching when: writes dominate and reads don't repeat; data must be perfectly fresh per read; or the data set is so cold there's no hot subset to cache.

## Trade-offs

- **Caches add a new source of bugs**: invalidation, race conditions, version skew between layers. ("There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton.)
- **Memory is bounded** — eviction policy is a *guess* about the future. Wrong guess = lots of misses.
- **Operational surface grows**: another cluster to monitor, fail over, scale, secure.
- **Consistency vs. freshness**: every cache decision is a knob between "always correct" and "always fast".

Alternatives to consider before reaching for a cache:

- Better indexes / denormalization in the DB.
- Read replicas (cheap horizontal read scale, see [Database Scaling](databases/scaling.md)).
- Pre-computing the answer asynchronously and storing it as a regular row.

## Linked problems

Caching shows up in every realistic HLD. A few where it's central:

- [URL Shortener](../problems/url-shortener.md) — cache-aside on Redis for hot short codes, 24h TTL, 80/20 rule.
- [Pastebin](../problems/pastebin.md) — CDN + Redis for hot pastes; cache-aside.
- [Instagram](../problems/instagram.md) / [Twitter](../problems/twitter.md) — feed caching in Redis, CDN for images.
- [Netflix](../problems/netflix.md) / [YouTube](../problems/youtube.md) — CDN as the primary cache layer (videos pre-encoded and pushed to edge).
- [Search Autocomplete](../problems/search-autocomplete.md) — precomputed top-N suggestions cached in Redis and CDN.
- [Rate Limiter](../problems/rate-limiter.md) — Redis as a distributed counter store with atomic `INCR`.
- [Distributed Cache](../problems/distributed-cache.md) — the canonical "build Memcached/Redis" problem.

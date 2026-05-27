# Design Search Autocomplete

> **Difficulty:** Intermediate · **Time:** 30–40 minutes

## Problem

Design a search-autocomplete service like Google's "type-ahead suggestions."
As the user types, return the **top 10 query suggestions** for the current
prefix, ranked by popularity (with recency / trending boosts). The system must:

- Be **fast** — p99 latency under 50 ms; ideally <20 ms perceived.
- Reflect **trending** queries within minutes.
- Handle **10M queries/day** of autocomplete traffic.
- Be **easy to extend** with personalization (per-user history),
  spell-correction, and i18n.

## Real-world intuition

Autocomplete looks trivial ("find strings starting with prefix X") and
becomes nontrivial only because **users type faster than the network round
trip**. You have ~30 ms total budget end-to-end including TLS, request
parsing, response, and rendering — meaning the backend has perhaps 10 ms
to do real work.

That budget forces three architectural choices:

1. **Precompute the answer.** You do not search a database on each
   keystroke; you look up a *prebuilt* list of top suggestions for that
   prefix.
2. **Cache aggressively** — the same first-letter prefixes (`"a"`, `"the"`,
   `"how"`) absorb the majority of the load and can be served from RAM or
   even the CDN.
3. **Update via stream + batch.** Trending events (a new movie release)
   must show up within minutes via a streaming aggregator. Long-tail
   popularity comes from a daily batch job. Lambda architecture in
   miniature.

The fundamental data structure is a **trie** — but the deployed
representation is a flat key-value store of `prefix → suggestions[]`
because lookups are O(1) and replicate well across cache nodes.

## Approach

### Capacity estimation

- **QPS:** 10M/day → ~120/sec avg, but bursty (every keystroke triggers a
  call; UI debounces partially). Plan for ~5,000/sec peak.
- **Per-key latency budget:** 10 ms backend.
- **Storage:**
  - Tracked queries: ~100M unique searches / language.
  - Prefix index: roughly **#queries × avg prefix length × top-K
    suggestions × bytes**. For English ≈ 100M × 15 × 10 × 16 bytes ≈
    **240 GB**. Fits in a Redis cluster.
- **Cache hit ratio for top prefixes:** >95%, because the
  prefix-frequency distribution is extreme power-law.

### Data structure

#### Trie (in-memory; or its serialized form)

Each node stores the top-K suggestions for the path-prefix ending at that
node. Inserting "facebook" into a trie at f→a→c→e→b→o→o→k pre-cooks the
top-K suggestions at every prefix node.

```
root
 ├── f
 │    └── a (top10: "facebook","facetime","fast food",...)
 │         └── c
 │              └── e (top10: "facebook","facetime",...)
 │                   └── ...
```

Lookup is O(prefix length) → effectively constant.

#### Deployed form: Redis hashes

Tries are fragile to update in a distributed setting. Serialize to a flat
KV store:

```
key:   ac:{lang}:{prefix}
value: ["facebook","facetime","face id","fast food",...]
TTL:   1 hour (refreshed by background job)
```

The autocomplete API: `GET ac:en:fac` → list. One round-trip, ~0.5 ms.

### Popularity scoring

```
score(query, t_now) = sum over each search event s:
                       decay_factor(t_now - s.ts)
```

`decay_factor` is exponential (`e^(-λ·age)`); λ tuned so a query has
half-life of, say, 7 days. Recent searches matter more than old ones.

Streaming layer (Flink) updates the rolling per-query score; a batch
job recomputes the trie nightly to settle long-tail rankings.

### Build pipeline

```
[Search events]
       │
       ▼
   [Kafka]
       │
       ├──── Stream (Flink): per-query rolling score → Redis hot
       │
       └──── Batch (Spark, daily): aggregate all queries, build trie,
              flatten to KV pairs (top-K per prefix), bulk-load Redis cold
```

The serving layer reads from Redis. A nightly job overwrites the entire
prefix-KV namespace from the freshly built trie; the streaming layer
patches changes that happen mid-day (especially for short, common prefixes
where rankings shift fastest).

### Serving path

```
[Browser]         ── 30ms RTT ──
     │ "fac" keystroke
     ▼
[CDN / edge]    ← cache hit for very popular prefixes
     │ miss
     ▼
[Autocomplete API]
     │
     ▼
[Redis cluster]  → top-10 suggestions for "fac" + lang
     │
     ▼
[client renders]
```

For very-popular prefixes (a single letter "a", "t", "h"), pin to the CDN
with 5-min TTL.

### Architecture diagram

```
                    ┌─────────────────────┐
                    │       Client        │
                    └──────────┬──────────┘
                               │ debounced keystrokes
                               ▼
                    ┌─────────────────────┐
                    │       CDN / edge    │
                    │  (1-letter prefixes,│
                    │   trending lists)   │
                    └──────────┬──────────┘
                               │ miss
                               ▼
                    ┌─────────────────────┐
                    │  Autocomplete API   │
                    │  - lang detect      │
                    │  - personalize hook │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
       ┌────────────┐  ┌────────────┐  ┌──────────────┐
       │ Redis hot  │  │ Redis cold │  │ Personal pref│
       │ (streaming │  │ (nightly   │  │ (user-level  │
       │ updates)   │  │ batch)     │  │ index)       │
       └─────┬──────┘  └─────┬──────┘  └──────┬───────┘
             ▲               ▲                 ▲
             │ patches       │ bulk load       │
             │               │                 │
     ┌───────┴──────┐  ┌─────┴────┐
     │  Flink       │  │  Spark   │
     │  (5-min      │  │ (nightly │
     │   windows)   │  │  batch)  │
     └──────┬───────┘  └────┬─────┘
            ▲               ▲
            └───────┬───────┘
                    │
              ┌─────┴─────┐
              │   Kafka   │ ← search events from /search endpoint
              └───────────┘
```

## Trade-offs

- **Trie in memory vs. flat KV.** Trie wins on storage (shared prefixes
  reuse nodes); flat KV wins on serving simplicity, sharding, and cache
  semantics. In practice you keep the trie offline and serve from a flat
  KV.
- **Top-K per node vs. compute at read.** Pre-storing top-K is O(1) read
  but more storage; computing at read is more flexible (e.g., per-user
  personalized) but slower. Production splits: top-K global precomputed,
  per-user re-rank on top.
- **Batch + stream (lambda) vs. stream-only (kappa).** Lambda gives clean
  separation between accurate-but-slow and fresh-but-noisy. Kappa is
  simpler ops but you must trust the streaming layer for long-term
  accuracy. Most autocomplete systems are lambda.
- **Personalize vs. share.** Personalized suggestions need a per-user
  feature lookup → adds latency. Hybrid: serve global top-K, then a tiny
  per-user re-rank.
- **Per-language vs. global.** Per-language indexes simplify scoring and
  filtering (no Mandarin suggestions when typing in English) at the cost
  of N× storage.

## Edge cases & gotchas

- **Misspellings / typos.** Index queries by their *edit-distance-1*
  neighbors (or use a phonetic key, Metaphone) so "facbook" still finds
  "facebook." Costs extra prefix storage.
- **Profanity / safe search.** Block-list filter on outputs; do not index
  blocked terms even if popular.
- **NSFW + young users.** Per-user safe-search flag picks the
  filter-on/filter-off namespace.
- **New-event burst.** A breaking-news term goes from "0 searches yesterday"
  to "100k/min." Stream layer catches it within a couple of minutes; the
  next batch promotes it to long-term rankings.
- **Adversarial pumping.** Bots searching the same query 10k times to
  promote it. Dedup by user/IP/session before scoring.
- **Multibyte / RTL languages.** Indexing prefix bytes is wrong for
  Unicode — split by codepoints (or grapheme clusters) instead.
- **Click feedback.** Did the user actually click a suggestion? Click rate
  is the best signal of suggestion quality and should feed back into the
  ranker.
- **Stale TTL.** A 1-hour TTL means a freshly promoted suggestion can take
  up to an hour to appear. For trending events, the streaming layer
  pushes invalidations.
- **Memory pressure.** 240 GB of prefix data won't all be hot. Use a
  layered cache: hot prefixes in RAM, cold prefixes in a SSD-backed
  store, pay the extra ms on tail queries.

!!! tip
    A great interview answer hits three notes: "trie for the structure,
    flat KV for serving, lambda pipeline for freshness." Anything else is
    polish on top.

## Linked concepts

- [Search & indexing](../fundamentals/search-indexing.md) — trie, inverted index, prefix matching
- [Caching strategies](../fundamentals/caching.md) — Redis hot keys, CDN edge cache
- [Data processing](../fundamentals/data-processing.md) — lambda architecture, Flink + Spark
- [Message queues](../fundamentals/message-queues.md) — Kafka for search events
- [CDN](../fundamentals/cdn.md) — edge caching of common prefixes
- [Distributed systems](../fundamentals/distributed-systems.md) — eventual consistency on rankings
- [API design](../fundamentals/api-design.md) — low-latency endpoint, debouncing UX

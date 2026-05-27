# Design a Recommendation System

> **Difficulty:** Architect · **Time:** 60+ minutes

## Problem

Design a recommendation system in the style of Amazon "Customers who
bought this also bought…" or Netflix's "Top picks for you." The system
must:

- Recommend **N items per user** (typically 10–50 in a ranked list).
- Personalize based on **user behavior** (views, clicks, purchases,
  watches, ratings).
- Combine **batch** (historical) and **real-time** (session) signals.
- Scale to **billions of users × hundreds of millions of items**.
- Handle the **cold-start** problem for new users and new items.
- Support **A/B testing** of recommendation strategies.

## Real-world intuition

A recommendation system is shaped by the same realization that defines
every modern personalization product: **you cannot compute personalized
results on every request from scratch**.

A naive "for each user, score every item" is `users × items` =
billions × hundreds of millions = an astronomical work item. The
canonical architecture splits the problem into two stages:

1. **Candidate generation (offline, broad)** — for each user, narrow the
   item space from hundreds of millions to a few thousand "plausible"
   candidates using cheap algorithms (collaborative filtering,
   embedding-similarity, popular-in-segment).
2. **Ranking (online, narrow, fresh)** — for the few thousand candidates,
   apply a more expensive ranker that incorporates recent behavior,
   context (device, time of day), and ML scoring.

This **two-tower / candidate-and-rank** pattern shows up in every
production recommender — YouTube, TikTok, Netflix, Amazon, Spotify. It is
*the* canonical recommender pattern.

Then layer on cold-start handling (no history → fall back to popularity
or content-based), A/B testing infrastructure, and feedback loops, and
you have a system that's as much MLOps as it is "ML."

## Approach

### Capacity estimation

- **Users:** 500M; ~50M daily active.
- **Items:** 100M.
- **Behavior events:** 5B/day (views, clicks, plays, purchases).
- **Recommendation requests:** 50M DAU × ~20 reqs/day = **1B reqs/day**
  → ~12k QPS avg.
- **Per-user offline candidate storage:** ~2,000 candidates × ~12 bytes =
  ~24 KB × 500M users = **12 TB** in Cassandra.

### Pipeline

```
┌────────────────────────────────────────────────────────────┐
│                       BATCH LAYER                          │
│                                                            │
│  Behavior events ─▶ HDFS / Lake                            │
│         │                                                  │
│         ▼                                                  │
│   Spark / Airflow:                                         │
│   - clean, dedup                                           │
│   - train collaborative filter (matrix factorization)      │
│   - train item embeddings (Word2Vec on co-watch sequences) │
│   - compute per-user candidate set (top 2,000 items)       │
│   - bulk-load → Cassandra: rec_candidates[user_id]         │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                       STREAM LAYER                         │
│                                                            │
│  Live events ─▶ Kafka                                      │
│         │                                                  │
│         ▼                                                  │
│  Flink / Kafka Streams:                                    │
│   - update user features (last-10 watched, last category)  │
│   - update item features (CTR last hour, freshness)        │
│   - write to feature store (online KV: Redis / Cassandra)  │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                       SERVING LAYER                        │
│                                                            │
│  GET /recommendations?user=u1                              │
│         │                                                  │
│         ▼                                                  │
│  1. fetch candidates[u1] from Cassandra (2k items)         │
│  2. fetch real-time features for u1 + 2k items from store  │
│  3. score with ranking model (XGBoost / DNN, served via    │
│     TensorFlow Serving or in-process)                      │
│  4. apply business rules (diversity, freshness, blocklist) │
│  5. return top 10                                          │
└────────────────────────────────────────────────────────────┘
```

### Candidate generation algorithms

#### Collaborative filtering (the classic)

- **User-based:** find users similar to u, recommend what they liked.
  Memory-bound at scale (users × users similarity matrix).
- **Item-based:** for each item, precompute "items similar to this
  item." Much smaller (items × items, fixed size). Recommend by
  aggregating similarity scores across the user's past items. **More
  scalable; preferred in production.**
- **Matrix factorization:** factor the user-item interaction matrix into
  low-dimensional latent factors. `prediction(u, i) ≈ user_emb[u] ·
  item_emb[i]`. Compact and queryable via nearest-neighbor search
  (Faiss, ScaNN).

#### Content-based

Each item has features (genre, actors, brand, price). Build a profile
"this user likes X kind of features" and score items by feature overlap.
Critical for cold-start items (we have features even with no
interactions).

#### Hybrid (the real answer)

Combine collaborative + content + popularity. Each technique generates
a candidate pool; merge & dedup → unified candidate set passed to the
ranker.

!!! note "Why multiple candidate sources?"
    Each technique is blind to part of the truth. Collaborative ignores
    content. Content ignores social signal. Popularity ignores
    personalization. Merging diversifies the candidate pool and reduces
    failure modes.

### Ranking

A learned ranker (LightGBM, DNN, or two-tower neural net) takes:

- **User features:** demographics, long-term affinities (genres),
  recent actions (last 10 items), session context (device, time).
- **Item features:** category, freshness, recent CTR, embedding.
- **User-item cross features:** has the user viewed this item before?
  affinity score, embedding dot product.

Output: per-candidate score → sort → top K.

Latency budget: ~50 ms total request → ~20 ms in the model.

### Cold-start

- **New user:** show **popular-in-segment** (top items by country, age
  range, device). After 3–5 interactions, switch to personalized.
- **New item:** content-based recommendation (features only) until it
  accrues click data; "explore" injection — show new items to a small
  random sample of users to gather initial signal.

### A/B testing

```
[Request] ─▶ Experiment service ─▶ "user u1 is in variant V2 of exp e3"
                  │
                  ▼
           Pick model variant V2 (e.g., new ranker)
                  │
                  ▼
           Serve recs; emit impressions
                  │
                  ▼
           [User clicks?] ─▶ Kafka ─▶ Spark aggregates per-variant CTR
                                         │
                                         ▼
                                  Dashboard: V2 +3.2% CTR significance p<0.01
```

A/B is non-optional — recommender quality is judged by behavior, not
opinions.

### Real-time personalization

A user just clicked "Stranger Things." Within seconds, their next
recommendation row should reflect this. The streaming layer updates
their feature store row; the next request reads the fresh features and
the ranker re-orders candidates accordingly. No full recompute needed —
the candidate set is reasonably stable; only the order changes.

### Architecture diagram

```
                  ┌──────────────────────────────┐
                  │    Behavior events (clicks,  │
                  │ views, watches, purchases)   │
                  └──────────────┬───────────────┘
                                 │
                  ┌──────────────▼───────────────┐
                  │            Kafka              │
                  └─────┬─────────────────┬───────┘
                        │                 │
                ┌───────▼────────┐ ┌──────▼─────────┐
                │  Flink stream  │ │  HDFS/Lake     │
                │  - real-time   │ │  (raw events)  │
                │    features    │ └──────┬─────────┘
                └───────┬────────┘        │
                        │                 ▼
                        ▼          ┌────────────────┐
                ┌──────────────┐   │ Spark jobs:    │
                │ Feature Store│   │  - train CF    │
                │ (Redis/      │   │  - train rank  │
                │  Cassandra)  │   │  - candidates  │
                └──────┬───────┘   └────────┬───────┘
                       │                    │
                       │           ┌────────▼─────────┐
                       │           │  Cassandra:      │
                       │           │  rec_candidates  │
                       │           │  per user        │
                       │           └────────┬─────────┘
                       │                    │
                       └────────┬───────────┘
                                ▼
                       ┌──────────────────┐
                       │ Recommendation    │
                       │  Serving API      │
                       │ - candidate fetch│
                       │ - features fetch │
                       │ - rank w/ model  │
                       │ - business rules │
                       └────────┬─────────┘
                                │
                                ▼
                          [Client UI]
                          (also emits
                           impression events → Kafka)
```

## Trade-offs

- **Candidate + rank vs. score-everything.** Candidate-and-rank is the
  only scalable shape. The trade-off is recall — items not in the
  candidate set cannot be recommended, no matter how good the ranker is.
- **Collaborative vs. content vs. hybrid.** Collaborative captures
  social/popularity signal but fails on cold-start. Content survives
  cold-start but misses non-obvious cross-domain affinities. Hybrid is
  more code but materially better.
- **Online vs. offline ranking.** Offline (precompute all top-K per
  user) is fast but stale. Online (rank at request time) is fresh but
  costs serving infrastructure. Most systems compute candidates offline,
  rank online.
- **Personalization vs. exploration.** Pure personalization creates filter
  bubbles. Inject ε% random or "explore" recommendations to discover new
  affinities and keep the model from collapsing into top-K-of-everything.
- **Strong vs. eventual consistency on features.** Eventual is fine —
  features lagging by a few seconds is invisible. Strong consistency
  would require synchronous writes per event, melting the pipeline.

## Edge cases & gotchas

- **Feedback loops.** The model recommends what users click; users click
  what the model recommends. Over time the system overfits to a
  popularity loop. Counter: explore injection, diversification
  constraints in the ranker, fresh-item boost.
- **Diversity / "more of the same."** Without diversity constraints, a
  user who watches one cooking video gets 10 cooking recommendations.
  Apply Maximum Marginal Relevance (MMR) or category caps in the
  business-rules layer.
- **Position bias.** Items shown at position 1 get more clicks than
  position 10 regardless of relevance. Train the model with inverse-
  propensity weights or position features.
- **Recency bias.** New items always get a popularity bump because no
  one has had time to ignore them. Counterbalance with a freshness
  decay.
- **Time-of-day / session context.** A user's mood at 7 AM differs from
  10 PM. Encode hour-of-day, day-of-week.
- **Blocked / muted items.** "Don't show me this category" or "I bought
  it, stop offering it." Maintain per-user negative signals.
- **Stale candidates.** Nightly batch means a user who joins a new
  community at noon sees no related candidates until tomorrow.
  Real-time candidate injection (just-in-time embedding lookup) fixes
  this.
- **Catalog churn.** Items go out of stock / unavailable in region.
  Filter at serve time, not at candidate generation, so a 10-item ask
  doesn't return 7.
- **Privacy and consent.** Some users opt out of personalization. Serve
  popular-in-segment in that case.
- **A/B sample bias.** Random assignment by user, not by request, or
  the user sees a confusing mix of variants.

!!! warning
    "Generate recommendations" is *not* a single model. Get used to
    splitting candidate generation from ranking explicitly. Conflating
    them is the most common interview mistake in this problem.

## Linked concepts

- [Data processing](../fundamentals/data-processing.md) — Spark batch, Flink stream
- [NoSQL (Cassandra)](../fundamentals/databases/nosql.md) — per-user candidates, watch history
- [Caching strategies](../fundamentals/caching.md) — feature store, model output cache
- [Message queues](../fundamentals/message-queues.md) — Kafka for event ingestion
- [Microservices](../fundamentals/microservices.md) — candidate, ranker, A/B service split
- [Search & indexing](../fundamentals/search-indexing.md) — nearest-neighbor (Faiss, ScaNN)
- [Distributed systems](../fundamentals/distributed-systems.md) — eventual consistency on features
- [Database scaling](../fundamentals/databases/scaling.md) — sharding candidate tables by user
- [Monitoring](../fundamentals/monitoring.md) — CTR, conversion, model drift dashboards

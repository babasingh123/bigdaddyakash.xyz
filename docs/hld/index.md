# High-Level Design

High-Level Design (HLD) is the discipline of sketching the moving parts of a software system — its data stores, services, queues, caches, and how requests flow between them — *before* anyone writes production code. This section is organized as a study reference: a collection of **fundamentals** you can dip into when you forget how sharding interacts with consistency, and a set of **worked problems** that show how to combine those fundamentals into a real system.

!!! tip "How to use this section"
    Skim the fundamentals once to get the vocabulary, then jump straight to a problem (e.g. URL Shortener) and follow the links *back* to fundamentals when something is unfamiliar. Reading the fundamentals top-to-bottom is fine, but most concepts only "click" when you see them anchored to a real design.

## Structure

```
hld/
├── index.md              ← you are here
├── fundamentals/         ← the building blocks
│   ├── databases/        ← SQL, NoSQL, scaling, transactions
│   ├── caching.md
│   ├── load-balancing.md
│   ├── cdn.md
│   ├── message-queues.md
│   ├── api-design.md
│   ├── microservices.md
│   ├── rate-limiting.md
│   ├── networking.md
│   ├── distributed-systems.md
│   ├── data-processing.md
│   ├── search-indexing.md
│   ├── security.md
│   ├── monitoring.md
│   └── misc-patterns.md  ← consistent hashing, bloom filter, idempotency, etc.
└── problems/             ← worked architect-level designs
    ├── url-shortener.md
    ├── instagram.md
    ├── uber.md
    └── ...
```

## Fundamentals

The fundamentals are grouped roughly the way they layer in a real architecture — storage at the bottom, then caching and load distribution, then the application protocols and the cross-cutting concerns.

### Storage and data

- [SQL databases (RDBMS)](fundamentals/databases/sql.md) — ACID, indexing, normalization, joins, locking. The default starting point for almost every design.
- [NoSQL databases](fundamentals/databases/nosql.md) — document, key-value, column-family, and graph stores. When to leave SQL behind.
- [Database scaling patterns](fundamentals/databases/scaling.md) — replication, sharding, partitioning. How a single Postgres box becomes a global fleet.
- [Database transactions & consistency](fundamentals/databases/transactions.md) — 2PC, saga, eventual consistency. The hard problem of "did it really happen?"

### Performance and distribution

- [Caching](fundamentals/caching.md) — cache-aside, write-through, eviction policies, stampede and avalanche problems.
- [Load balancing](fundamentals/load-balancing.md) — Layer 4 vs Layer 7, round robin vs least connections, health checks.
- [CDN](fundamentals/cdn.md) — edge caching for static assets and video.
- [Message queues & async processing](fundamentals/message-queues.md) — Kafka, RabbitMQ, SQS, delivery guarantees, dead-letter queues.

### Application and architecture

- [API design](fundamentals/api-design.md) — REST, GraphQL, gRPC, WebSockets, API gateway.
- [Microservices architecture](fundamentals/microservices.md) — service discovery, circuit breakers, saga, sync vs async communication.
- [Rate limiting & throttling](fundamentals/rate-limiting.md) — token bucket, leaky bucket, sliding window.
- [Networking & protocols](fundamentals/networking.md) — HTTP/HTTPS, TCP vs UDP, DNS, SSE vs long polling.

### Distributed systems theory and ops

- [Distributed systems concepts](fundamentals/distributed-systems.md) — CAP, consistency models, consensus (Paxos/Raft), clocks.
- [Data processing](fundamentals/data-processing.md) — batch vs stream, Lambda and Kappa architectures.
- [Search & indexing](fundamentals/search-indexing.md) — inverted indexes, full-text search, geospatial indexing.
- [Security](fundamentals/security.md) — auth (sessions, JWT, OAuth), authorization (RBAC/ABAC), encryption, common threats.
- [Monitoring & observability](fundamentals/monitoring.md) — metrics, structured logging, distributed tracing.
- [Miscellaneous patterns](fundamentals/misc-patterns.md) — consistent hashing, bloom filter, idempotency, heartbeat, backpressure.

## Worked problems

Each problem page applies several fundamentals to a concrete product. They're roughly ordered by difficulty.

A few starting points:

- [URL Shortener](problems/url-shortener.md) — the classic warm-up. Touches SQL, base62 encoding, caching, rate limiting.
- [Instagram](problems/instagram.md) — sharding, CDN, fan-out on write vs read, Cassandra timelines, image processing pipelines.
- [Uber](problems/uber.md) — geospatial indexing, real-time location via WebSockets, ETA, surge pricing, payment sagas.
- [Dropbox](problems/dropbox.md) — file chunking, sync algorithms, conflict resolution, deduplication.

The full list lives in `problems/` and covers Pastebin, Twitter, YouTube, Netflix, WhatsApp, Rate Limiter, Web Crawler, Notification System, Search Autocomplete, Distributed Cache, and Recommendation System.

!!! note "Reading order"
    If you're prepping for interviews, the highest leverage path is: read the fundamentals once for vocabulary, then for each problem write your *own* design first, then read the problem page and diff against your answer. The mistakes you make in the diff are where the learning happens.

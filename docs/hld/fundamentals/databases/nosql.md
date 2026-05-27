# NoSQL Databases

"NoSQL" is an umbrella term for non-relational data stores that trade SQL's rigid schemas and ACID guarantees for some combination of horizontal scale, schema flexibility, or specialized access patterns. The four common categories are **document**, **key-value**, **column-family**, and **graph** stores — each optimized for a different shape of data and query.

## Why it exists

By the late 2000s, web-scale companies (Google, Amazon, Facebook) hit walls that single-node relational databases couldn't easily clear: terabytes of user-generated content, millions of writes per second, geographically distributed users, and access patterns where joins and ad-hoc SQL added latency the product couldn't afford. Two influential papers — Google's Bigtable (2006) and Amazon's Dynamo (2007) — described how to give up some SQL features (joins, multi-row transactions, strong consistency) in exchange for **linear horizontal scalability** and **always-on availability**.

NoSQL exists because, for a meaningful fraction of workloads, the constraints relational databases enforce are *not the problem you're solving*. If you're storing user sessions, you don't need joins. If you're storing time-series sensor data, you don't need a flexible query language — you need millions of writes per second. NoSQL lets you pick the right tool for the access pattern.

## How it works

There isn't one "NoSQL" — there are four families with very different internals. Pick the family based on how the data is shaped *and* how it's queried.

### Document stores (MongoDB, CouchDB)

Each record is a self-describing JSON-like document. Schemas are flexible: two documents in the same collection can have different fields.

```json
{
  "_id": "user_123",
  "name": "Alice",
  "email": "alice@example.com",
  "addresses": [
    { "type": "home", "city": "NYC" },
    { "type": "work", "city": "Boston" }
  ]
}
```

Key design decision: **embed vs. reference**.

- **Embed** child data when it's always read with the parent and doesn't grow unboundedly (an order with its line items).
- **Reference** (store a foreign ID) when the child is shared across many parents (a product referenced by many orders) or when it can grow large (a user's comments).

Watch out for **document size limits** (16 MB in MongoDB) and the cost of indexing fields inside large nested arrays.

**Use cases**: content management, user profiles, product catalogs, event logging.

### Key-value stores (Redis, DynamoDB, Memcached)

The simplest model: a hash map. `GET key` returns a value, `SET key value` stores one. O(1) lookups, no joins, no rich queries.

Redis goes further than a plain key-value store by giving each value a *type* — string, list, set, sorted set, hash, stream, hyperloglog — each with its own atomic operations. That makes Redis a Swiss-army knife for things like leaderboards (sorted sets), session storage (hashes), or rate limiters (`INCR` with a TTL).

```
SET session:abc123 "{user_id: 42, expires: ...}"  EX 3600
INCR rate_limit:user_42
ZADD leaderboard 9000 "player_7"
ZREVRANGE leaderboard 0 9 WITHSCORES   # top 10
```

**Use cases**: session storage, caching, rate limiting, real-time analytics, leaderboards, distributed locks.

### Column-family stores (Cassandra, HBase, ScyllaDB)

Data is stored by *column family* rather than by row. Each row has a **partition key** (which determines the node) and one or more **clustering columns** (which determine the sort order *within* a partition). Designed for very high write throughput and time-series workloads.

```
CREATE TABLE messages (
    conversation_id UUID,
    sent_at         TIMESTAMP,
    sender_id       UUID,
    content         TEXT,
    PRIMARY KEY ((conversation_id), sent_at)
) WITH CLUSTERING ORDER BY (sent_at DESC);
```

Here `conversation_id` is the partition key (all messages for a conversation live on the same node, in the same SSTable region), and `sent_at` is the clustering column (within the partition, messages are sorted newest-first — perfect for "load the last 50 messages"). You **design the schema around the queries**: there's no ad-hoc SQL planner that will rescue a poorly chosen partition key.

**Use cases**: time-series, IoT data, write-heavy logs, messaging timelines, analytics events.

### Graph databases (Neo4j, Amazon Neptune, JanusGraph)

Data is nodes and edges. Both nodes and edges can carry properties. Designed for queries that traverse relationships many hops deep — exactly the queries that turn into nightmare self-joins in SQL.

```cypher
MATCH (a:User {name: "Alice"})-[:FRIEND*2..3]-(suggestion:User)
WHERE NOT (a)-[:FRIEND]-(suggestion) AND a <> suggestion
RETURN suggestion, count(*) AS mutual_friends
ORDER BY mutual_friends DESC LIMIT 10;
```

That query — "people 2 to 3 hops away in my friend graph, ranked by mutual connections" — is trivial in Cypher and brutal in SQL.

**Use cases**: social networks, recommendation engines, fraud detection, knowledge graphs, network/IT topology.

### Cross-cutting concepts

#### BASE

The NoSQL counterpart to ACID is **BASE**:

- **B**asically **A**vailable — the system stays available for reads/writes even under failure.
- **S**oft state — the state of the system may change over time even without input (eventual consistency does background convergence).
- **E**ventual consistency — given no new writes, all replicas eventually converge to the same value.

BASE is a *consequence* of choosing AP under CAP (see below) for the sake of availability and scale.

#### CAP theorem

A distributed system can guarantee at most two of:

- **C**onsistency — every read returns the most recent write.
- **A**vailability — every request gets a non-error response.
- **P**artition tolerance — the system keeps working when the network drops messages between nodes.

In a real network, partitions *will* happen, so P is non-negotiable. That leaves the real choice:

- **CP** systems — sacrifice availability during partitions to preserve consistency. Examples: MongoDB (in strict mode), HBase, ZooKeeper, etcd, Spanner. Use for banking, inventory, configuration.
- **AP** systems — sacrifice consistency during partitions to stay available. Examples: Cassandra, DynamoDB, Riak, CouchDB. Use for social feeds, analytics, recommendations.

See [Distributed Systems Concepts](../distributed-systems.md) for a deeper treatment of CAP and consistency models.

## When to use

Pick NoSQL when one or more of these is true:

- **Schema-flexible data** — you're storing heterogeneous documents and a strict relational schema would be a constant migration burden → document store.
- **Sub-millisecond key lookups at huge scale** — session data, cache layer, rate limits → key-value store.
- **Massive write throughput** with a known, narrow set of read patterns — IoT, telemetry, timelines, time-series → column-family store.
- **Relationship-centric queries** that go many hops deep — social graphs, recommendations, fraud rings → graph database.
- **Linear horizontal scale** without operational heroics — your data outgrew a single primary and you can co-design the schema around partitioning.

Pick SQL instead when you need ACID across multiple entities, ad-hoc analytical queries, or you genuinely have relational data and no scale requirement that demands otherwise.

## Trade-offs

What NoSQL gives up:

- **Joins** — most NoSQL stores don't have them, or only support them within a single partition. You denormalize or join in the application.
- **Multi-document/multi-row transactions** — limited or absent. Some stores (MongoDB 4+, DynamoDB) added transactional APIs, but with caveats.
- **Ad-hoc queries** — you must design the schema around the queries you'll run. Adding a new query pattern can mean adding a new table or a secondary index.
- **Strong consistency by default** — most AP systems give you eventual consistency; you opt into stronger guarantees per-operation (quorum reads/writes, light transactions) at a latency cost.
- **Maturity of tooling** — SQL has 50 years of BI, ORM, and reporting tools. NoSQL ecosystems are improving but uneven.

!!! warning "NoSQL is not "schemaless""
    You still have a schema — it's just encoded in your application code instead of the database. That schema still drifts, still needs migration plans, and still needs documentation. The flexibility is real, but it's flexibility for *evolution*, not an excuse to skip data modeling.

## Linked problems

NoSQL stores show up across most architect-level designs:

- [Instagram](../../problems/instagram.md) — Cassandra for timeline storage, sharded by `user_id`; user metadata in SQL.
- [Twitter](../../problems/twitter.md) — Cassandra/Redis for hot tweets and home timeline; Elasticsearch for search.
- [Uber](../../problems/uber.md) — Redis for real-time driver location, with geohash keys for spatial queries.
- [WhatsApp](../../problems/whatsapp.md) — Cassandra for message storage (time-series), sharded by `user_id`.
- [Netflix](../../problems/netflix.md) — Cassandra for watch history (`{user_id, video_id, timestamp, progress}`).
- [Dropbox](../../problems/dropbox.md) — chunk metadata in SQL, chunks themselves in object storage (S3).
- [URL Shortener](../../problems/url-shortener.md) — common alternative design uses DynamoDB / Redis instead of SQL.
- [Distributed Cache](../../problems/distributed-cache.md) — the canonical key-value store, built from scratch.
- [Recommendation System](../../problems/recommendation-system.md) — graph DBs for collaborative filtering, key-value for serving.

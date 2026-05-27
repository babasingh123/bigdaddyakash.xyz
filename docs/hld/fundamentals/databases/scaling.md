# Database Scaling Patterns

Database scaling is the toolkit for taking a single-node database beyond what one machine can serve. The two big levers are **replication** (copy the data onto more nodes to handle more reads or survive failures) and **sharding** (split the data across more nodes to handle more total data and writes). Most real systems use both.

## Why it exists

A single Postgres or MySQL server can take you a long way — terabytes of data and tens of thousands of QPS on modern hardware. But three forces push you beyond a single node:

1. **Read traffic exceeds one machine.** A product goes viral, a marketing campaign succeeds, and reads outstrip what the primary can serve.
2. **Total data outgrows one machine.** No single SSD can hold 100 TB of orders, even if your QPS is fine.
3. **Geography matters.** Users in Tokyo shouldn't pay 150 ms of RTT to your `us-east-1` primary.

Replication addresses (1) and (3) cheaply. Sharding addresses (2) and (3) more aggressively, at significant operational cost.

!!! tip "Don't shard until you have to"
    Sharding is one of the highest-cost architectural decisions in HLD. Cross-shard joins, resharding, and hotspots are all genuinely hard. Exhaust vertical scaling, read replicas, and caching first.

## How it works

### Replication

Replication keeps multiple copies of the same data, typically a **primary** that accepts writes and one or more **replicas** that mirror it.

#### Master-Slave (Primary-Replica)

```
        writes
          ↓
    ┌─────────┐
    │ Primary │
    └────┬────┘
         │ async replication
    ┌────┴────┬─────────┐
    ▼         ▼         ▼
 Replica   Replica   Replica
    ↑         ↑         ↑
        reads
```

- The primary handles all writes.
- Replicas asynchronously copy the primary's WAL/binlog and serve reads.
- Replicas lag the primary by milliseconds to seconds — this is **replication lag**.
- If the primary fails, you **promote** a replica to be the new primary (failover).

The big gotcha is **read-after-write inconsistency**: a user posts a comment (write goes to primary), reloads the page (read goes to a lagging replica), and the comment isn't there. Mitigations: route the user's reads to the primary for a short window after a write, or use "read-your-writes" routing by user_id.

#### Master-Master (Multi-Master)

```
    writes              writes
      ↓                   ↓
  ┌────────┐         ┌────────┐
  │ Node A │ ←─────→ │ Node B │
  └────────┘  sync   └────────┘
```

- Multiple nodes accept writes.
- Each node replicates changes to the others.
- Increased write availability — but you must resolve **write conflicts**: what if both nodes accept conflicting updates to the same row?

Conflict resolution strategies: last-write-wins (clock-based, lossy), version vectors (Dynamo-style), CRDTs (mergeable types), or application-level reconciliation. Most teams find single-primary with fast failover simpler than true multi-master.

#### Read replicas

Read replicas are the practical instance of master-slave: a fleet of read-only copies that scale read traffic horizontally. Use them for:

- **Reporting and analytics** — heavy queries that would otherwise contend with OLTP traffic.
- **Geographic reads** — replicas in EU and APAC for low-latency reads, writes still go to a primary.
- **Read-heavy services** — when reads dominate (90%+) and lag of a few seconds is acceptable.

For things where lag *isn't* acceptable, prefer caching (see [Caching](../caching.md)) over more replicas.

### Sharding (horizontal partitioning)

Sharding splits *rows* across multiple databases. Each shard is an independent database holding a slice of the total data. To find data, you need a **shard key** — the column whose value determines which shard a row lives on.

#### Sharding strategies

- **Hash-based sharding** — apply a hash function to the shard key, modulo the number of shards. `shard = hash(user_id) % N`. Distributes data evenly. The downside: changing `N` (adding or removing shards) reshuffles almost everything. Use **consistent hashing** (see [Misc Patterns](../misc-patterns.md)) to mitigate this.
- **Range-based sharding** — assign contiguous ranges of the key to each shard (`A–M` on shard 1, `N–Z` on shard 2; or timestamp ranges). Easy to add a new shard for the next range. **Hotspots** are the risk: if your key is monotonically increasing (timestamps, sequential IDs), the newest shard takes all the load.
- **Directory-based sharding** — a lookup service maps each key (or key range) to a shard. Maximum flexibility, but the directory becomes a hot dependency and another scaling problem.
- **Geographic sharding** — shard by region. US users live in US shards, EU users in EU shards. Reduces latency, helps with data residency laws (GDPR), but cross-region queries are expensive.

#### Challenges

| Challenge | What goes wrong | Mitigation |
|-----------|----------------|------------|
| **Cross-shard queries** | A query that needs data from multiple shards must fan out, gather, and merge — slow and complex. | Co-locate related data (use the same shard key for `users` and `orders`). Denormalize. Avoid these queries by design. |
| **Resharding** | Doubling the number of shards rewrites half the data. While in flight, the system is in a fragile dual-write state. | Use consistent hashing to bound movement. Plan capacity well ahead. Use tools (Vitess, Citus) that automate it. |
| **Hotspot shards** | Uneven key distribution — one celebrity user has 100× the data of average. | Add salt to the key, sub-shard hot ranges, or move hot tenants to dedicated shards. |
| **Distributed transactions** | ACID across shards needs 2PC or sagas (see [Transactions](transactions.md)). | Design so transactions stay within one shard. Use saga pattern for cross-shard work. |

### Partitioning vs. sharding

The terms overlap, but the standard distinction is:

- **Vertical partitioning** — split *columns* into separate tables. The `users` table becomes `users` (hot columns: id, name, email) and `user_profiles` (cold columns: bio, avatar URL, preferences). Often a step in normalization.
- **Horizontal partitioning (sharding)** — split *rows* across servers. The `users` table is sliced into `users_shard_0`, `users_shard_1`, etc.

Some databases also support **declarative partitioning** within a single server (Postgres `PARTITION BY RANGE`, MySQL `PARTITION BY HASH`) — that's horizontal partitioning *within* one machine, useful for managing very large tables and time-based archival. It's not sharding (no horizontal scale), but it shares the design vocabulary.

### Putting it together

A common production setup looks like:

```
                       ┌──────────────────────┐
                       │   App / API layer    │
                       └──────────┬───────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
       ┌─────────┐           ┌─────────┐           ┌─────────┐
       │ Shard 0 │           │ Shard 1 │           │ Shard 2 │
       │ Primary │           │ Primary │           │ Primary │
       └────┬────┘           └────┬────┘           └────┬────┘
            │                     │                     │
       ┌────┴────┐           ┌────┴────┐           ┌────┴────┐
       │ Replica │           │ Replica │           │ Replica │
       └─────────┘           └─────────┘           └─────────┘
```

Shard by `user_id` (hash), each shard has a primary and one or more read replicas, and the application layer routes reads/writes appropriately.

### Tools

- **Vitess** — MySQL sharding, originally from YouTube. Production-grade resharding.
- **Citus** — Postgres sharding extension (Microsoft).
- **MongoDB** — built-in sharding, configurable shard key.
- **Cassandra** — sharding is the model; partition key required.
- **CockroachDB / YugabyteDB / Spanner** — NewSQL: SQL interface, automatic sharding, strong consistency via consensus.

## When to use

Apply in this order — each step has compounding cost and complexity:

1. **Tune the schema and indexes.** A missing index can multiply load by 100×.
2. **Add caching** (Redis, in-process). Often the cheapest way to absorb read traffic.
3. **Scale vertically** — bigger machine, more RAM, faster disks. Boring but effective.
4. **Add read replicas** when reads dominate and stale reads are tolerable.
5. **Shard** when total data or write throughput exceeds a single primary, and only then.

## Trade-offs

- **Replication trades freshness for throughput.** Replicas serve more reads but they lag — your application must tolerate (or detect and route around) stale data.
- **Multi-master trades simplicity for write availability.** You gain write resilience but inherit conflict resolution as a permanent problem.
- **Sharding trades flexibility for scale.** You scale linearly but lose easy cross-shard queries, ad-hoc analytical SQL, and simple schema evolution.
- **Sharding by the wrong key is very expensive to undo.** A bad shard key choice in year one can dominate engineering work in year three.

!!! warning "Pick the shard key first"
    The single most important decision when sharding is the shard key. It dictates which queries are fast, which are slow, and where hotspots form. Choose to match your dominant query pattern (usually `user_id` for user-facing apps).

## Linked problems

- [Instagram](../../problems/instagram.md) — shards `users` by `user_id`, `photos` by `photo_id`, `follows` by `follower_id`; read replicas for feed reads.
- [Twitter](../../problems/twitter.md) — same shape: timeline data in Cassandra (partitioned by `user_id`), hot tweet cache in Redis.
- [WhatsApp](../../problems/whatsapp.md) — messages sharded by `user_id` in Cassandra; cross-user lookups stay within partition.
- [Dropbox](../../problems/dropbox.md) — metadata DB sharded by `user_id`; cross-shard joins avoided by design.
- [Uber](../../problems/uber.md) — sharding combined with geographic partitioning; payment data demands strict per-shard transactions.
- [URL Shortener](../../problems/url-shortener.md) — common interview discussion: shard by hash of the short URL.
- [Distributed Cache](../../problems/distributed-cache.md) — consistent hashing applied to a cache fleet (same shape, different store).

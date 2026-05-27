# Design a Distributed Cache

> **Difficulty:** Advanced · **Time:** 40–50 minutes

## Problem

Design a distributed in-memory cache like Memcached or Redis. The system
must support:

- `GET`, `SET`, `DELETE` on string keys.
- **Horizontal scalability** — add or remove nodes without massive
  rebalancing.
- **High availability** — surviving node failures.
- An **eviction policy** (LRU) when memory fills.
- Low latency (<1 ms p99 on local-LAN reads).

## Real-world intuition

Caching as a *concept* is simple: a key-value store with a TTL and an
eviction policy. The reason "design a distributed cache" is a real
interview is everything that happens when you scale that store across
machines:

1. **Sharding without thrashing.** If you grow from 5 nodes to 6, modular
   hashing (`hash(key) % N`) re-maps almost every key. That's catastrophic
   — every entry becomes a cache miss simultaneously. **Consistent
   hashing** is the canonical fix and the headline data-structures
   problem of this design.
2. **Failure detection without false positives.** A node went down? Or is
   it just GC-paused for 200 ms? Wrong call → either you lose data or you
   thrash the cluster. Heartbeats, gossip, and quorum reads are the tools.
3. **Replication for HA without write coordination overhead.** Strong
   consistency across replicas requires consensus (slow). Eventual
   consistency is fast but stale. For a cache (where staleness is by
   definition tolerated), eventual + last-write-wins is usually correct.
4. **In-process correctness.** Each node implements LRU + concurrency
   control. The data-structure question ("LRU in O(1)") matters as much
   as the distributed question.

## Approach

### Capacity estimation

- **Cluster:** start with 10 nodes × 64 GB = **640 GB** total cache
  capacity.
- **Throughput target:** 1M QPS aggregate (GET-dominated), p99 < 1 ms
  inside the DC.
- **Memory per key:** 1 KB average value + 200 B overhead → ~600M keys
  cluster-wide.
- **Network:** 1M QPS × ~1 KB = ~10 Gbps east-west.

### Sharding: consistent hashing

```
Hash ring (size 2^32):

      ┌────A─────────────────────────┐
      │                              B
      │                              │
      │   key1 →                     │
      │   nearest clockwise = B      │
      │                              │
      │                              C
      └────D─────────────────────────┘
```

Each node owns the "arc" from itself counter-clockwise to the previous
node. Adding a node E only steals from its clockwise neighbor — only ~1/N
of keys move.

**Virtual nodes (vnodes):** each physical node is mapped to ~150 random
positions on the ring. This smooths the per-node load distribution
(otherwise a single bad random draw could give one node a 4× share).

#### Why not modular sharding?

```
5 nodes, modular:
   hash(k) % 5 = 2  → node 2

Add a node:
   hash(k) % 6 = 4  → node 4    (key moved!)

Probability of move = (N-1)/N → ~83% of keys move on a 6-node grow.
```

For a cache, that means 83% of your entries become misses → origin DB
gets hammered → outage. Consistent hashing makes this ~1/N (~17% in this
example).

### Client-side vs. server-side routing

- **Client-side hashing:** clients know the ring topology and pick the
  node directly. Saves a hop. Used by Memcached's standard clients.
  Downside: clients must agree on topology (Zookeeper / config service).
- **Server-side proxy:** a thin proxy fronts the cluster, hides topology.
  Easier client UX, extra hop. Used by Twemproxy, mcrouter.

Both are valid; client-side is faster for read-heavy paths.

### Replication and HA

For each key, store **N copies** on N consecutive nodes around the ring.
On a `SET`:

```
1. Coordinator hashes key, finds primary + N-1 replicas.
2. Writes to all N (async or sync depending on consistency level).
3. Acks client when W writes complete.
```

Tunable consistency (Dynamo-style):

- **W writes acknowledged + R reads queried** with **W + R > N** = strong
  consistency.
- For a cache, **W=1, R=1, N=2** is typically fine — eventual is
  acceptable, and we trade a tiny stale-read window for huge speed.

On node failure, replicas keep serving. When the node returns, **hinted
handoff** replays missed writes.

### Failure detection (gossip + heartbeat)

```
Every node pings 3 random peers every second.
If 3 peers can't reach node N in 5s → mark N suspect.
After 30s of suspicion → mark N dead, redistribute its arcs.
```

Gossip distributes membership changes across the cluster without a
single coordinator. Variations: SWIM (Hashicorp Consul), phi-accrual
detector (Cassandra).

### Eviction: LRU in O(1)

In each node, an LRU is a **doubly-linked list + hash map**:

```
HashMap: key → list_node
List:    most_recent ←→ ... ←→ least_recent

GET key:
  - HashMap lookup → node
  - Move node to head of list
  - Return value

SET key:
  - If exists: update value, move to head
  - Else: create node at head; HashMap[key] = node
  - If size > capacity: remove tail, evict from HashMap
```

All operations O(1). Memcached uses a slab allocator on top to avoid
heap fragmentation; Redis uses jemalloc with explicit LRU sampling
(approximate LRU is cheaper at high QPS).

### Architecture diagram

```
              ┌──────────────────────────────┐
              │          Clients              │
              └──────────────┬───────────────┘
                             │  hash(key) → node
       ┌─────────────────────┼─────────────────────┐
       ▼                     ▼                     ▼
  ┌─────────┐           ┌─────────┐           ┌─────────┐
  │ Node A  │           │ Node B  │           │ Node C  │
  │ vnodes  │◀─gossip──▶│ vnodes  │◀─gossip──▶│ vnodes  │
  │ +LRU    │           │ +LRU    │           │ +LRU    │
  └────┬────┘           └────┬────┘           └────┬────┘
       │  replication        │                     │
       └─────────────┬───────┴─────────────────────┘
                     ▼
              ┌───────────────┐
              │ Membership /  │
              │ topology store│
              │  (etcd/ZK)    │
              └───────────────┘
```

## Trade-offs

- **Consistent hashing vs. modular.** Consistent hashing is essential at
  scale; the only reason to consider modular is a fixed, never-resized
  small cluster.
- **Synchronous vs. asynchronous replication.** Sync gives durability and
  consistency at the cost of write latency. For a cache, async is
  usually fine — losing a few recent writes on node death is acceptable.
- **Tunable consistency (W, R).** Lower W = faster writes, more stale
  reads. Higher W = slower writes, fresher reads. Caches default to
  loose values.
- **Client-aware routing vs. proxy.** Client-aware = fewer hops, more
  complexity. Proxy = simpler clients, +1 hop, easier topology changes.
- **Approximate vs. exact LRU.** Exact LRU has lock contention on the
  list at high QPS. Sampled LRU (Redis style) — pick 5 random entries,
  evict the oldest — is ~95% as good and dramatically faster.
- **Memcached vs. Redis style.** Memcached is pure cache (no
  persistence, no data structures). Redis adds persistence, pub/sub,
  rich types. Pick based on use case.

## Edge cases & gotchas

- **Hot keys.** One viral key on one shard saturates that node. Mitigate
  with **client-side caching** (TTL of 1s) or **key replication** (store
  hot keys on multiple shards explicitly).
- **Cache stampede / thundering herd.** Key expires, 10k clients miss
  simultaneously, all hit origin DB. Solutions: probabilistic early
  expiration (refresh slightly before TTL), single-flight (only first
  request fetches, others wait).
- **Cache penetration.** A query for a key that's never been set hits
  the origin every time. Cache a "negative" placeholder with short TTL.
- **Cache avalanche.** Many keys expire at exactly the same time → DB
  pile-up. Randomize TTLs (`TTL = base + jitter`).
- **Resharding under load.** Adding a node during peak is risky. Migrate
  arcs slowly with double-reads (check old + new owner during transition).
- **Node failure during write.** With W=1, the data may be only on the
  primary; if it dies, the write is lost. Acceptable for cache, not for
  data store.
- **Network partition (split brain).** Two halves of the cluster believe
  they own the same arcs. Gossip + quorum-aware writes minimize damage;
  many caches accept temporary inconsistency.
- **Large values.** A 10 MB cache value blows the memory bound when
  multiplied by replicas. Cap value size (Memcached default: 1 MB).
- **Eviction storm.** When memory fills, naive LRU evicts continuously
  during writes → latency spikes. Pre-evict at 90% capacity in
  background.

!!! tip
    The single most "interview-bait" moment is *why* consistent hashing
    matters. Draw the ring, draw a key, add a node, point at the few
    keys that move. Then mention virtual nodes for distribution. That's
    the core idea.

## Linked concepts

- [Caching strategies](../fundamentals/caching.md) — eviction, TTL, stampede protection
- [Misc patterns](../fundamentals/misc-patterns.md) — consistent hashing, virtual nodes, heartbeats
- [Distributed systems](../fundamentals/distributed-systems.md) — CAP, quorum, gossip protocols
- [Database scaling](../fundamentals/databases/scaling.md) — sharding parallels
- [NoSQL](../fundamentals/databases/nosql.md) — Dynamo-style consistency tuning
- [Load balancing](../fundamentals/load-balancing.md) — request routing parallels
- [API design](../fundamentals/api-design.md) — GET/SET/DELETE protocol shape
- [Monitoring](../fundamentals/monitoring.md) — hit-rate, latency, evictions metrics

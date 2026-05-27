# Miscellaneous Patterns

A grab-bag of small patterns that recur across HLD problems: **consistent hashing**, **bloom filters**, **idempotency**, **heartbeats / health checks**, and **backpressure**. Each is the answer to a specific recurring problem — and each is something interviewers expect you to know by name and apply on demand.

## Why it exists

These patterns are pulled out into one place because each is small enough that a dedicated page would be overkill, and large enough that you'll meet them in almost every realistic design. They're the verbs of the HLD vocabulary: once you recognize the problem each one solves, you'll use them on autopilot.

## Consistent Hashing

### Problem

Distribute keys across `N` servers so each server holds roughly `1/N` of the data. Easy with `hash(key) % N` — until `N` changes. Add or remove a server, and `hash(key) % (N±1)` rebalances *almost everything*. Most keys move.

### Solution

Map both keys *and* servers to positions on a **circular hash ring** (a `2^32`-sized circle). To find a key's server, walk clockwise from the key's position to the first server you hit.

```
                ┌──── server A ────┐
        ┌───────│                  │───────┐
        │       │                  │       │
        │   ◯  k1                  │ k3 ◯  │
        │                          │       │
   server D                       server B
        │                          │       │
        │   k4 ◯                ◯ k2       │
        │       │                  │       │
        └───────│                  │───────┘
                └──── server C ────┘
```

Each key maps to the first server clockwise from its position. Adding a new server only steals keys from a single neighbor. Removing a server only sends its keys to the next clockwise neighbor.

### Virtual nodes

A single position per server gives uneven distribution if you have few servers or unlucky hashes. **Virtual nodes** fix this: each physical server is placed at `K` (e.g. 100–200) random positions on the ring. With many virtual positions per server, the average chunk per server is smooth.

```
hash("serverA#0"), hash("serverA#1"), ..., hash("serverA#K-1") → 100 positions
hash("serverB#0"), hash("serverB#1"), ..., hash("serverB#K-1") → 100 positions
```

### Properties

- Adding or removing one server reshuffles only **1/N** of the keys on average.
- The remapping is bounded and predictable, not catastrophic.
- A staple of distributed caches, sharded databases, and request routing.

### Use cases

- **Distributed cache** (Memcached client-side hashing, Redis Cluster).
- **Database sharding** that needs cheap scale-out (DynamoDB, Cassandra).
- **Load balancing** with session affinity (consistent hash on user ID).
- **CDN edge routing**.

!!! tip "Use library implementations"
    Don't hand-roll consistent hashing in production. Libraries (Ketama, libring, dynomite) get the edge cases right (virtual node distribution, hash function choice) and are widely battle-tested.

## Bloom Filter

### What it is

A probabilistic data structure that tells you whether an element is in a set, with one notable property:

- **No false negatives.** If the bloom filter says "not in the set", it is *definitely* not in the set.
- **Possible false positives.** If it says "probably in the set", the element might not actually be there.

You trade a tunable error rate for an enormous space saving. A bloom filter can represent a set of millions of items in megabytes of memory.

### How it works

- A bit array of size `m`, initially all zeros.
- `k` independent hash functions.
- To insert `x`: compute `k` hashes of `x`, set those `k` bits in the array.
- To check `x`: compute the same `k` hashes; if *all* bits are set, "probably in the set"; if any bit is zero, "definitely not".

```
insert "alice": hash1=3, hash2=7, hash3=12 → set bits 3, 7, 12
insert "bob":   hash1=2, hash2=7, hash3=15 → set bits 2, 7, 15

check "alice": bits 3, 7, 12 — all set → "probably in"
check "carol": hash1=5 — bit 5 unset → "definitely not in"
```

The false positive rate depends on `m`, `k`, and the number of inserted items `n`. The optimum:

```
k = (m/n) * ln(2)
false_positive_rate ≈ (1 - e^(-kn/m))^k
```

You pick the error rate you can tolerate (e.g. 1%), and that fixes the size.

### Use cases

- **Avoid hitting the DB / disk for items that don't exist.** Check the bloom filter first; if "not in set", skip the expensive lookup. Cassandra uses bloom filters per SSTable for this.
- **Web crawler URL deduplication** — bloom filter in memory; full check in DB only on positive.
- **Cache filtering** — only put items in cache if the bloom filter says they exist (prevents cache penetration).
- **Username availability checks** — cheap "probably taken" check before a DB lookup.
- **Malware / phishing URL detection** — fast probable-match against huge blocklist.

### Limitations

- Can't remove elements (resetting bits would create false negatives). Variants like **counting bloom filters** support removal, at the cost of more memory.
- False positive rate degrades as the filter fills past its design capacity. Plan capacity.
- Doesn't tell you *which* element matched, just "something might match".

## Idempotency

### Concept

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

```
Idempotent:     SET balance = 100         (call it 5 times, balance is still 100)
Not idempotent: balance += 100            (call it 5 times, balance is +500)
```

### Why it matters

Networks fail. A client sends `POST /charge` for $100. The server processes it and writes the charge to the DB. The response is lost in transit. The client retries → the server charges *again*. Now the customer has been charged $200 and is opening a support ticket.

In a distributed system, **every cross-network operation is potentially a duplicate**:

- Client retries on timeout.
- Load balancer retries failed requests.
- Message queues with at-least-once delivery.
- Webhook handlers that get called twice when the receiver acks slowly.

Idempotency is the discipline that makes duplicates safe.

### Implementation patterns

#### Idempotency keys

The client generates a unique ID per logical operation and sends it with the request:

```
POST /charge
Idempotency-Key: 7f3e2c-a4b1-..

{ "amount": 100, "user_id": "42" }
```

The server:

1. Checks if the idempotency key already has a recorded response.
2. If yes — return the stored response. **Do not re-execute.**
3. If no — execute the operation, store the response under the key, return it.

Storage: a key-value store (Redis, DynamoDB) keyed by `(client_id, idempotency_key)` with a TTL of 24 hours or so. Stripe popularized this pattern and it's now standard for payment APIs.

#### Naturally idempotent operations

- **PUT** is idempotent by HTTP convention (same body, same result).
- **DELETE** is idempotent (deleting an already-deleted thing is a no-op).
- **Upsert** (`INSERT ... ON CONFLICT DO UPDATE`) — same result regardless of how many times it runs.

#### Side-effect free with deduplication

- Generate a deterministic ID from the request payload; insert with `UNIQUE` constraint; catch duplicate-key errors as "already done".

### Use cases

- **Payment processing** — charge once, even if the call retries.
- **Order creation** — `Idempotency-Key` prevents accidental double orders.
- **Webhook handlers** — providers (Stripe, GitHub) send each event with an ID; deduplicate.
- **Message queue consumers** — at-least-once delivery means duplicates; consumer idempotency handles them.
- **Distributed transactions** — saga retries depend on each step being idempotent.

!!! warning "Idempotency keys must outlive retries"
    A common bug: idempotency key TTL is 5 minutes, client retry interval is 10 minutes. Second attempt looks like a new request, gets executed again. Make TTLs comfortably longer than any expected retry window.

## Heartbeat / Health Check

### Concept

A periodic message ("I'm alive") sent by a service to a monitor (or by a monitor probing a service). If the monitor stops seeing heartbeats, it marks the service unhealthy.

### Why it matters

In a distributed system you can't tell, by silence alone, whether a node is:

- Alive but slow.
- Crashed.
- On the other side of a network partition.

A heartbeat protocol gives you a *bounded* time to detect each failure mode. Without it, you discover a dead node only when something tries to use it.

### Implementations

#### Active health checks (most common)

The load balancer (or orchestrator) periodically hits a `/health` endpoint:

```
GET /health
→ 200 OK         ← keep in rotation
→ 5xx / timeout  ← remove from rotation after N consecutive failures
```

- Configure interval (e.g. 5s), timeout (e.g. 2s), unhealthy threshold (e.g. 3 consecutive failures), healthy threshold (e.g. 2 consecutive successes).
- **Shallow check** — process is up.
- **Deep check** — also verifies DB connectivity, cache, downstream services. Don't go too deep, or one downstream blip cascades into your service being marked unhealthy too.

#### Push-based heartbeats

Each service writes a heartbeat to a central store every N seconds. If the timestamp is older than 2N, monitoring marks it stale.

```
SET service:user-svc:host-7 timestamp=NOW EX 30
```

Used when there's no LB doing active checks (e.g. workers consuming from a queue).

#### Gossip-based heartbeats

Each node tracks the heartbeats of its neighbors and gossips that knowledge. Used by Cassandra, Consul, Akka clusters. Resilient to central monitor failure.

### Use cases

- **Load balancer health checks** — remove unhealthy backends from rotation. See [Load Balancing](load-balancing.md).
- **Service discovery** — register on heartbeat, deregister on timeout (Consul, etcd).
- **Cluster membership** — Cassandra, Kafka use heartbeats to know who's in the cluster.
- **Worker liveness** — orphaned workers detected and their tasks reassigned.

### Failure detection trade-offs

Tighter intervals mean faster failure detection but more network overhead and more false positives (a momentary GC pause looks like a death). Standard pattern: 5s interval, 15s declare dead. Tune per workload.

## Backpressure

### Problem

A fast producer overwhelms a slow consumer. Without intervention, work piles up — in memory, in a queue, on disk — until something fails. The classic symptoms: ballooning queue depth, growing latency, eventual OOM, cascading failure.

### Solutions

Three families of responses, from least to most graceful.

#### Drop requests

Reject incoming work when capacity is exceeded.

```
HTTP 503 Service Unavailable
Retry-After: 30
```

- Cheap, predictable.
- Pushes the problem to the client (which had better handle it).
- Common at the API gateway.

#### Buffer

Hold incoming work in a queue (in-memory or persistent) until the consumer can catch up.

- **Bounded buffer**: drop or reject when full. Bounded = capacity-controlled.
- **Unbounded buffer**: never reject, but risk OOM. Almost always wrong.

Buffering smooths spikes but doesn't help sustained overload — it just delays the failure.

#### Throttle the producer

Tell the producer to slow down. Implementation styles:

- **Reactive Streams** semantics — consumer signals "I'm ready for N more items". Producer respects the signal. Used by gRPC streaming, RSocket, modern reactive libraries.
- **TCP congestion control** — built into the network stack. If your consumer stops `ACK`ing, the OS slows the sender.
- **Rate limiting upstream** — return `429` to producers exceeding limits.
- **Closed-loop systems** — control loops that observe queue depth and adjust producer rate (e.g. AIMD).

### Where backpressure shows up

- **Streaming systems** (Kafka, Flink) — consumer lag is the backpressure signal.
- **Message queues** — bounded queues drop or block; DLQs absorb persistent failure (see [Message Queues](message-queues.md)).
- **HTTP services** — `503` + `Retry-After`, circuit breakers on the caller.
- **Reactive frameworks** — RxJS, Project Reactor, Akka Streams have explicit backpressure primitives.

!!! warning "Watch for queue accumulation"
    A growing queue is a warning sign even before anything breaks. Alert on **consumer lag** or queue depth trends, not just queue full / consumer crashed. By the time something breaks, you've usually had 10+ minutes of growing lag.

## Linked problems

These patterns are scattered across every problem; some particularly central instances:

- **Consistent hashing**: [Distributed Cache](../problems/distributed-cache.md), [Web Crawler](../problems/web-crawler.md), [Instagram](../problems/instagram.md) (sharding).
- **Bloom filter**: [Web Crawler](../problems/web-crawler.md) (URL dedup), [URL Shortener](../problems/url-shortener.md) (collision detection on key generation), cache penetration prevention.
- **Idempotency**: [Notification System](../problems/notification-system.md), [Uber](../problems/uber.md) (payments), [URL Shortener](../problems/url-shortener.md) (`Idempotency-Key` on `POST /shorten`).
- **Heartbeat / health check**: [Distributed Cache](../problems/distributed-cache.md) (node failure detection), [Uber](../problems/uber.md) (driver online status), [WhatsApp](../problems/whatsapp.md) (presence).
- **Backpressure**: [Notification System](../problems/notification-system.md) (third-party rate limits), [Web Crawler](../problems/web-crawler.md) (per-domain politeness), [YouTube](../problems/youtube.md) (encoding pipeline).

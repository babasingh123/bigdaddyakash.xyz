# Distributed Systems Concepts

A distributed system is any system whose components run on more than one networked machine. The moment that's true, you inherit a thorny set of theoretical problems: nodes fail independently, clocks disagree, messages drop, and "the data" is no longer a single thing — it's many copies that must somehow stay in sync. This page covers the concepts you have to fluently reach for in any non-trivial HLD: CAP, consistency models, consensus, distributed transactions, and clocks.

## Why it exists

The single-node mental model — `BEGIN; UPDATE; COMMIT;` — gets you very far. But every system that needs more capacity than one machine, more reliability than one machine, or low latency to globally distributed users hits the same wall: **the network is part of the system**. Messages get lost. Replicas get out of sync. Nodes fail at different times in different ways.

The theory on this page exists because every distributed system pays for these realities. CAP and consistency models give you the vocabulary to describe trade-offs you must make. Consensus algorithms tell you how to safely agree on anything. The clock section explains why "happened before" is harder than it looks.

## How it works

### CAP theorem

A distributed data store can guarantee at most two of:

- **C**onsistency — every read returns the most recent write.
- **A**vailability — every request gets a non-error response.
- **P**artition tolerance — the system continues to operate despite arbitrary network partitions.

**The trade-off**: you can only have two of three.

**In reality**: network partitions *will* happen — switches die, links flap, regions get cut off. So `P` is non-negotiable for any real distributed system. The real choice is between:

- **CP systems** — sacrifice availability during partitions to keep the data consistent. If a node can't talk to its peers, it refuses requests rather than risk diverging.
- **AP systems** — sacrifice consistency during partitions to stay up. Each side of the partition keeps serving with its local data; reconciliation happens after.

#### Examples

| Type | Examples | Use case |
|------|----------|----------|
| **CP** | MongoDB (strict), HBase, ZooKeeper, etcd, Spanner | Banking, inventory, configuration, locks |
| **AP** | Cassandra, DynamoDB, Riak, CouchDB | Social feeds, analytics, telemetry, shopping cart |

!!! note "PACELC: the full picture"
    CAP is what happens *during* a partition. PACELC adds: **else** (when there's no partition), the system still chooses between **L**atency and **C**onsistency. Spanner is CP/CC (strict). DynamoDB is AP/EL (eventual + low latency by default).

CAP is a *coarse* model — most systems offer dials, not a single pick. DynamoDB defaults to AP but supports strongly-consistent reads at higher latency. MongoDB has tunable write/read concerns. Treat CAP as the framing, not the answer.

### Consistency models

"Consistency" actually spans a spectrum. From strongest to weakest:

#### Linearizability (strong / external consistency)

- Reads always see the latest committed write.
- Operations behave as if they executed one at a time in real-time order.
- Most expensive — requires coordination on every write (usually consensus).
- **Examples**: Spanner with TrueTime, CockroachDB serializable.

#### Sequential consistency

- All clients agree on the same order of operations.
- That order may not match real time, but it's globally consistent.
- Slightly weaker than linearizability.

#### Causal consistency

- If operation A happened-before operation B, every client sees A before B.
- Unrelated operations can be observed in different orders by different clients.
- **Example**: a comment must appear after the post it's commenting on; two unrelated posts can be seen in different orders.

#### Read-your-writes (session) consistency

- A given client always sees its own writes immediately on subsequent reads, even if other clients see stale data.
- Practical compromise — feels right to the user, doesn't require global coordination.
- **Example**: editing your profile and seeing the change reflected; other users may catch up moments later.

#### Eventual consistency

- Given no new writes, all replicas converge to the same value eventually.
- Two clients may briefly disagree.
- Fastest, cheapest, weakest.
- **Example**: like counts on social media.

```
linearizable > sequential > causal > read-your-writes > eventual
   strong, slow                            weak, fast
```

Pick the weakest consistency that doesn't cause product bugs. Reaching for "strong consistency by default" without justification is a common architectural sin — you pay coordination costs you don't need.

See [Database Transactions](databases/transactions.md) for how these map to practical patterns (2PC, sagas, etc.).

### Consensus algorithms

**The problem**: distributed nodes need to agree on a single value (who's the leader, what's the next log entry, whether to commit a transaction) in the face of network failures and node crashes.

#### Algorithms

| Algorithm | Notable for |
|-----------|------------|
| **Paxos** | The original. Proven correct, famously hard to understand and implement. |
| **Multi-Paxos** | Practical extension of Paxos for log replication. |
| **Raft** | Designed for understandability. Same guarantees as Paxos, much simpler. The de facto choice for new systems. |
| **ZAB** | ZooKeeper Atomic Broadcast. Powers ZooKeeper's coordination. |
| **PBFT** | Byzantine fault tolerant — handles malicious nodes. Used in blockchains. |

#### How Raft works (in a sentence)

A cluster of N nodes elects a **leader** (majority vote). The leader accepts writes, replicates them to followers via a log, and acknowledges to the client only after a majority of followers have persisted the entry. If the leader dies, the surviving majority elects a new one. As long as a majority is alive, the system makes progress.

The math: you need `2f + 1` nodes to tolerate `f` failures. With 3 nodes you survive 1 failure; with 5 you survive 2.

#### Use cases

- **Leader election** in a cluster.
- **Distributed locks** ("who holds the lock?").
- **Configuration management** ("what's the current schema?").
- **Replicated state machines** — replicating an arbitrary state machine consistently.
- **Distributed transactions** in NewSQL (Spanner, CockroachDB).

**Tools**: ZooKeeper, etcd, Consul, HashiCorp Raft, openraft.

!!! tip "You probably don't implement consensus yourself"
    99% of the time, you *use* a system that implements consensus (etcd, Consul, ZooKeeper) rather than implement it. Knowing how it works helps you reason about its failure modes — split brain, election timeouts, the cost of cross-region clusters.

### Distributed transactions

Three approaches, from strictest to most pragmatic.

#### Two-Phase Commit (2PC)

```
Phase 1 (Prepare):
    Coordinator → all nodes: "Can you commit?"
    Each node responds: "Yes" or "No"

Phase 2 (Commit):
    If all "Yes": Coordinator tells all to commit
    If any "No":  Coordinator tells all to abort
```

- **Atomicity** guaranteed across nodes.
- **Blocking** — a coordinator crash leaves participants stuck holding locks.
- Used by XA, some NewSQL internals. Avoided in greenfield designs.

#### Three-Phase Commit (3PC)

- Adds a pre-commit phase. Non-blocking in the absence of network partitions.
- Rarely used in practice — extra round-trip cost, doesn't solve partition cases cleanly.

#### Saga pattern

- Sequence of local transactions, each with a compensating action that undoes it.
- No global locks, no blocking — the dominant pattern for cross-service workflows.
- Trades isolation for liveness.

Deeper treatment in [Database Transactions](databases/transactions.md).

### Clocks in distributed systems

#### Why clocks are hard

Nodes have independent physical clocks. They drift (microseconds to seconds per day) and they don't agree with each other. "What happened first" is meaningless without a shared notion of time.

#### NTP (Network Time Protocol)

- Synchronizes clocks across the network against time sources.
- Typical accuracy: tens of milliseconds within a data center, hundreds across the internet.
- **Not good enough** for tight distributed ordering. Two nodes that disagree by 50 ms can interleave their events arbitrarily.

#### Logical clocks

Forget physical time. Just track *causality*.

##### Lamport timestamps

Each event in the system has a counter. The rule:

- On a local event: `clock++`.
- On sending a message: include `clock` in the message.
- On receiving a message with timestamp `t`: `clock = max(clock, t) + 1`.

If event A happened-before event B, then `A.clock < B.clock`. **But** two unrelated events can have the same or arbitrary order — Lamport gives you a *total order consistent with causality*, not real time.

##### Vector clocks

Each node maintains a vector `[c_1, c_2, ..., c_N]` — one counter per node.

- Local event: increment own counter.
- On message send: include the vector.
- On message receive: take elementwise max, then increment own counter.

Vector clocks let you detect *concurrent* events: A `→` B iff every component of A's vector is `<=` B's. If neither A `→` B nor B `→` A, they're concurrent — useful for conflict detection.

**Use case**: Dynamo and Riak use vector clocks to detect concurrent writes to the same key and surface conflicts to the application.

#### Spanner's TrueTime

A different approach: invest heavily in tightly-synchronized atomic clocks + GPS to give every node a bounded-uncertainty timestamp `[earliest, latest]`. Spanner waits out the uncertainty window before committing, achieving external consistency across the globe. Expensive infrastructure, but it lets a relational database span continents.

### Why all this matters

In every interview, you'll need to answer questions that boil down to:

- "What happens during a network partition?" → CAP, consistency model.
- "How does the system stay correct under failure?" → Consensus, replication.
- "How do you handle cross-service workflows?" → Sagas vs. 2PC.
- "How do you order events across nodes?" → Logical clocks vs. physical clocks.

The page above is the toolkit. The problem pages apply it.

## When to use

| Concept | Reach for it when... |
|---------|---------------------|
| CAP framing | Discussing trade-offs of multi-node data stores. |
| Strong consistency | Money, inventory, identity, anything where a stale read is a bug. |
| Eventual consistency | Counters, feeds, analytics, anything where staleness is invisible. |
| Consensus (Raft/Paxos) | Leader election, distributed locks, configuration, replicated state. |
| 2PC | Almost never in greenfield — but mention it to show you know it. |
| Saga | Cross-service business workflows. |
| Vector clocks | Conflict detection in AP systems. |
| TrueTime / hybrid logical clocks | Global strongly-consistent databases. |

## Trade-offs

- **Strong consistency costs latency.** Coordination requires round trips. Globally distributed strong consistency requires either heroics (TrueTime) or accepting tens of ms per write.
- **Availability vs. consistency** is the central dial. Pick deliberately per data set, not globally.
- **Consensus is expensive.** Every write touches a majority of nodes. Designed-in for things that need it; never put on the hot path of high-QPS reads.
- **Logical clocks add complexity** to your protocol but enable real correctness across nodes.
- **Distributed transactions are still the boss fight** of HLD. The pragmatic answer is usually "don't have them" — design the system so transactions stay local.

!!! warning "There's no such thing as "exactly once" without effort"
    Any time someone says "exactly-once delivery", the truthful version is "at-least-once delivery + idempotent consumer". Build idempotency into your design (see [Misc Patterns: Idempotency](misc-patterns.md)) and stop chasing the impossible.

## Linked problems

- [Distributed Cache](../problems/distributed-cache.md) — consistent hashing, replication, failure detection, eventual consistency.
- [Uber](../problems/uber.md) — strong consistency for payments (saga + ACID); eventual for surge pricing.
- [Instagram](../problems/instagram.md) / [Twitter](../problems/twitter.md) — eventual consistency on feeds, likes, counters.
- [WhatsApp](../problems/whatsapp.md) — vector-clock-ish reasoning for message ordering across devices.
- [Dropbox](../problems/dropbox.md) — conflict resolution between concurrent edits.
- [Rate Limiter](../problems/rate-limiter.md) — atomic counters in Redis; consensus when you need exact distributed limits.
- [Recommendation System](../problems/recommendation-system.md) — batch + stream architecture relies on eventual consistency between layers.

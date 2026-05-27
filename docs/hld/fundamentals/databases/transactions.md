# Database Transactions & Consistency

A transaction groups multiple data operations into a single all-or-nothing unit. In a single-node database, transactions are a solved problem (ACID, see [SQL](sql.md)). In a **distributed** system — multiple databases, multiple services, multiple regions — preserving the same guarantee is one of the hardest problems in software. This page covers the patterns and their trade-offs.

## Why it exists

The motivating example is always the same: transfer $100 from account A to account B.

```
debit_account(A, 100)
credit_account(B, 100)
```

If the process crashes between those two lines, $100 vanishes. If the network drops the second call, $100 vanishes. If both succeed but the response is lost and the client retries, $100 might be moved twice. Any production money mover, any inventory system, any order pipeline has this exact problem in some form.

On a single database, `BEGIN ... COMMIT` solves it. But the moment those two accounts live in two different services (or two different shards), you cross a *distributed transaction* boundary — and now you need a protocol. This page is about those protocols and the alternatives.

## How it works

There are roughly three approaches, from strictest and slowest to most pragmatic and most popular:

1. **Two-Phase Commit (2PC)** — guarantee atomicity with a coordinator. Strict, blocking, painful.
2. **Saga pattern** — chain of local transactions with compensating actions. Pragmatic, the microservices default.
3. **Eventual consistency** — accept that the system is temporarily inconsistent and design for convergence. Cheapest, but only valid where staleness is tolerable.

### Two-Phase Commit (2PC)

A **coordinator** orchestrates the commit across all participating nodes.

```
Phase 1 (Prepare):
    Coordinator → all nodes: "Can you commit?"
    Each node:
        - writes the change to its WAL but does NOT commit
        - locks the affected rows
        - replies "Yes" (vote-commit) or "No" (vote-abort)

Phase 2 (Commit / Abort):
    If all "Yes":  Coordinator → all nodes: "COMMIT"
    If any "No":   Coordinator → all nodes: "ABORT"
    Each node releases locks after acknowledging.
```

#### What 2PC guarantees

- **Atomicity** across nodes — either every node commits or none do.
- All participants either see the new state or none of it.

#### Why it hurts

- **Blocking.** A node that voted "Yes" must wait for the coordinator's decision before releasing its locks. If the coordinator crashes between Phase 1 and Phase 2, every participant is stuck holding locks. They can't safely abort (some peer may have committed) or commit (some peer may have voted no).
- **Coordinator is a single point of failure.** Recovery requires durable coordinator state and complex recovery protocols.
- **Latency.** Every transaction takes at least 2 round trips, and locks are held the entire time. Throughput craters.

2PC still has its place — XA transactions, traditional enterprise systems, some NewSQL databases use it internally — but for greenfield distributed designs, the saga pattern usually wins.

!!! note "Three-Phase Commit (3PC)"
    3PC adds a pre-commit phase that makes the protocol non-blocking under most failure scenarios. It's rarely used in practice — the failure modes it solves are rare enough that the extra round trip isn't worth it, and it still doesn't handle network partitions cleanly.

### Saga pattern

A saga is a sequence of **local transactions**, each of which has a **compensating transaction** that semantically undoes it. Instead of holding locks across the whole flow, each step commits immediately. If a later step fails, you run the compensations in reverse.

Example: placing an order across four services.

```
Order placed
   │
   ├─→ T1: Create Order              (compensate: Cancel Order)
   │
   ├─→ T2: Charge Payment            (compensate: Refund Payment)
   │
   ├─→ T3: Reserve Inventory         (compensate: Restore Inventory)
   │
   └─→ T4: Schedule Shipping         (compensate: Cancel Shipment)

If T4 fails:
   → run T3.compensate, T2.compensate, T1.compensate
```

#### Two coordination styles

- **Choreography** — each service emits events when it finishes, and other services subscribe. No central coordinator. Simple for small flows; spaghetti for large ones.
- **Orchestration** — a central saga coordinator (often a state machine) issues commands to each service and reacts to results. Easier to reason about and observe; the coordinator itself must be highly available.

#### What sagas guarantee

- **Eventual atomicity** — eventually, either the whole flow succeeds or all completed steps are compensated.
- Local steps remain ACID; no global locks.

#### What sagas don't give you

- **Isolation.** Other transactions can see intermediate states. A reader could observe an order with a charged payment but no inventory reserved. You need application-level mitigations: pending statuses, "semantic locks", or compensating reads.
- **Easy compensations.** Some operations don't truly compensate — sending an email is hard to "un-send". You design around this (e.g. only send the confirmation email after the entire saga commits).

Saga is the default for microservices because it preserves service autonomy, doesn't hold locks across the network, and tolerates partial failure gracefully. See [Microservices](../microservices.md) for the broader pattern context.

### Eventual consistency

For a large class of problems, you don't need atomicity *or* isolation across services — you just need the system to eventually agree.

- **Updates propagate over time.** A "like" recorded on shard 3 is replicated to other shards and aggregated into counters within seconds.
- **Reads may briefly disagree.** Two users refreshing the same post might see different like counts for a few hundred milliseconds.
- **Convergence is guaranteed.** Given no new writes, all replicas eventually reach the same state.

When this is acceptable:

| Acceptable | Not acceptable |
|------------|---------------|
| Social media likes, view counts | Bank account balances |
| News feed ordering | Inventory at checkout |
| Recommendations | Payment confirmation |
| Analytics dashboards | Order placement |
| Search index freshness | Authentication state |

The rule of thumb: if a temporary inconsistency would *only* annoy a user (counter ticks up a moment late), eventual consistency is fine. If it would *deceive* them (charged twice, double-booked seat, double-spent token), you need stronger guarantees.

!!! tip "Eventual ≠ "whenever""
    "Eventual" in practice usually means tens of milliseconds to a few seconds, *not* "we'll get to it next Tuesday". If your replication lag is minutes, you have an operational problem regardless of your consistency model.

## When to use

| You need... | Use... |
|-------------|-------|
| Cross-shard atomicity within one database | Database-native distributed transactions (Spanner, CockroachDB, Vitess) |
| Atomic commit across heterogeneous resources | 2PC / XA — only when you have to |
| Atomic-ish behavior across microservices | Saga pattern (orchestrated for complex flows) |
| Read-your-own-writes guarantee | Sticky reads to primary, or session-level consistency token |
| Plain replication catching up over time | Eventual consistency (default for AP NoSQL) |

Decision shortcut for an interview:

- **One DB, multiple rows** → SQL transactions.
- **One DB, sharded** → choose a shard key that keeps the transaction local. If you can't, look at NewSQL (Spanner-style).
- **Multiple services, business workflow** → saga.
- **Multiple services, "is it ok if this is briefly wrong?"** → eventual consistency.

## Trade-offs

- **Strong consistency costs latency.** Every coordinated commit involves at least one round trip — often several — across nodes. At global scale this can mean hundreds of milliseconds.
- **2PC trades availability for atomicity.** A single slow or down participant blocks every transaction touching it.
- **Sagas trade isolation for liveness.** No locks, but partial states are visible to other readers. You handle that in product design.
- **Eventual consistency trades freshness for availability.** The system stays up under partition, but two clients can see different "truths" for a window.

!!! warning "Compensating actions are not magic"
    A common subtle bug: writing a "compensating transaction" that *can fail* and not handling that failure. Sagas need each compensating step to be **retryable until success**, and you need monitoring for stuck sagas. Otherwise you've replaced a transactional bug with a silent data integrity bug.

## Linked problems

- [Uber](../../problems/uber.md) — explicitly uses the saga pattern for charge rider → pay driver across services, with compensating refunds.
- [Notification System](../../problems/notification-system.md) — at-least-once delivery with idempotency keys; eventual consistency on user preferences.
- [Instagram](../../problems/instagram.md) / [Twitter](../../problems/twitter.md) — eventual consistency on like/comment counters and feed propagation.
- [Dropbox](../../problems/dropbox.md) — conflict resolution when two clients edit the same file offline; "create conflict copy" instead of merging.
- [WhatsApp](../../problems/whatsapp.md) — read receipts (sent → delivered → read) propagated eventually across devices.
- [URL Shortener](../../problems/url-shortener.md) — write to primary, eventual read consistency from replicas/cache.
- [Distributed Cache](../../problems/distributed-cache.md) — replication with last-write-wins (eventual) or quorum (stronger).

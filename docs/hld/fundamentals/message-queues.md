# Message Queues & Async Processing

A message queue is a durable buffer between **producers** (services that emit work) and **consumers** (services that perform it). Instead of producer A calling consumer B directly over HTTP, A drops a message into a queue and B reads it when ready. That tiny piece of indirection unlocks an enormous amount of architectural flexibility.

## Why it exists

Synchronous calls couple services together in three painful ways:

1. **Availability coupling** — if B is down, A's request fails. Cascading failures.
2. **Throughput coupling** — A can only call B as fast as B can serve. A traffic spike on A becomes a spike on B even if A's work was just to enqueue.
3. **Latency coupling** — A waits for B to finish before responding. A 50 ms task downstream becomes a 50 ms tax on every user-facing request.

A queue breaks all three:

1. **Decoupling** — producer and consumer don't need to know about each other; they only know the queue.
2. **Asynchronous** — the producer enqueues and returns immediately; consumers process at their own pace.
3. **Load leveling** — the queue absorbs traffic spikes. A 10× burst becomes a longer queue, not a backend collapse.
4. **Reliability** — durable queues survive consumer crashes. Failed messages can be retried automatically.
5. **Scalability** — add more consumers to drain the queue faster. The queue is the load balancer.

A real example: when a user signs up, you want to (a) create their account in the DB, (b) send a welcome email, (c) provision a workspace, (d) post to your analytics pipeline, (e) trigger a cron job. Doing all five synchronously means the signup HTTP call is dog-slow and any one failure breaks signup. Doing them async — write to DB, enqueue an "UserCreated" event, return — keeps the user flow snappy and lets each downstream consumer fail and retry on its own.

## How it works

### Two core patterns

#### Point-to-Point (Queue)

```
Producer → [Queue] → Consumer
```

- One message is delivered to **one** consumer (whoever grabs it first).
- Multiple consumers can compete for messages — that's how you scale processing.
- **Use cases**: job queues, background tasks (image resizing, email sending), work distribution.

#### Publish-Subscribe (Topic)

```
                    ┌──→ Consumer A (inventory)
Producer → [Topic] ─┼──→ Consumer B (shipping)
                    └──→ Consumer C (analytics)
```

- One message is delivered to **every subscriber**.
- Subscribers are independent — they each get their own copy, with their own offset/cursor.
- **Use cases**: event broadcasting, fan-out, decoupled services that react to the same event.

Kafka collapses the distinction by using **consumer groups**: subscribers in the same group share work (point-to-point); subscribers in different groups each get a full copy (pub/sub).

### Delivery guarantees

Every queue makes a trade-off between speed and certainty. The three canonical levels:

| Guarantee | Meaning | When to use |
|-----------|---------|-------------|
| **At-most-once** | Message delivered 0 or 1 times. May be lost. | Telemetry, non-critical events where speed matters more than correctness. |
| **At-least-once** | Message delivered 1 or more times. May duplicate. | The common default. Consumer must be **idempotent**. |
| **Exactly-once** | Message delivered exactly once. Hardest, slowest. | Payments, billing, anywhere duplicates are unacceptable. |

In practice, true exactly-once is rare and expensive. The pragmatic pattern is **at-least-once + idempotent consumer**: the producer can retry, the consumer deduplicates by idempotency key. See [Misc Patterns: Idempotency](misc-patterns.md).

!!! tip "Always make consumers idempotent"
    Even if your queue promises exactly-once, network blips and consumer crashes mean you'll get duplicates. The cheapest insurance is to process each message in a way that's safe to run twice (look up by a unique key, upsert instead of insert, etc.).

### Comparison of tools

#### Kafka

- **Model**: distributed, replicated, partitioned log.
- **Throughput**: millions of messages per second per cluster.
- **Durability**: messages persisted to disk; replicated across brokers.
- **Ordering**: guaranteed *per partition*; globally unordered.
- **Consumers**: consumer groups for parallel work; offsets stored in the broker.
- **Use**: event streaming, log aggregation, real-time analytics, change data capture, the backbone of stream processing.

#### RabbitMQ

- **Model**: traditional broker, AMQP protocol.
- **Throughput**: tens of thousands of messages/sec.
- **Durability**: messages can be persisted; ack/nack model.
- **Ordering**: per queue.
- **Flexibility**: exchanges (direct, topic, fanout, header) → bindings → queues. Very flexible routing.
- **Use**: task queues, RPC, work distribution, complex routing logic.

#### AWS SQS / SNS

- **Model**: managed queue (SQS) and pub/sub (SNS).
- **Throughput**: nearly unlimited (managed).
- **Durability**: messages stored across multiple AZs.
- **Ordering**: standard queues are best-effort; FIFO queues preserve order with exactly-once semantics at lower throughput.
- **Use**: AWS-native workloads, serverless triggers, fan-out to many consumers.

#### Redis Pub/Sub & Streams

- **Pub/Sub**: simple, fire-and-forget, no persistence — if no one's subscribed, the message is gone.
- **Streams**: a persistent log with consumer groups, more durable but less battle-tested than Kafka.
- **Use**: real-time notifications, lightweight events when persistence isn't critical.

#### Comparison table

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| Throughput | Very High | Medium | Medium–High |
| Durability | High | Medium | High |
| Ordering | Per partition | Per queue | FIFO queues |
| Replay | Yes (offsets) | No (consume = remove) | No |
| Routing flexibility | Low | High | Low |
| Operational cost | High (self-hosted) | Medium | None (managed) |
| Use Case | Streaming, events | Task queues, RPC | AWS serverless |

### Dead Letter Queues (DLQ)

What happens when a message just won't process? Maybe the payload is malformed, maybe a downstream call fails permanently. You don't want to retry forever and block the queue.

**Pattern**: after N failed delivery attempts, move the message to a **dead letter queue**. The original queue keeps moving; ops can inspect the DLQ later.

```
Producer → Main Queue → Consumer
                ↓ (after N failed retries)
              DLQ → manual inspection / alert / replay
```

- Configure max retries per message.
- Alert on non-empty DLQ; it's a signal something is broken.
- Build a "replay" tool to push messages back from DLQ once the bug is fixed.

### Retry strategies

Retries should be **bounded and exponential**:

```
attempt 1 → delay 1s
attempt 2 → delay 2s
attempt 3 → delay 4s
attempt 4 → delay 8s
...
attempt N → move to DLQ
```

Add jitter to the delay to avoid thundering herds on a downstream service.

### Backpressure

If producers consistently outpace consumers, the queue grows forever — eventually exhausting memory or disk. **Backpressure** signals the producer to slow down (or drop) when the queue is full. See [Misc Patterns: Backpressure](misc-patterns.md).

## When to use

Reach for a message queue when:

- You want to **decouple** services so one can deploy/fail independently of another.
- The work is **fan-out** — one event triggers many consumers (order placed → inventory + shipping + email + analytics).
- The work is **slow** and shouldn't block the user-facing request (image processing, video encoding, sending notifications).
- You have **traffic spikes** that exceed your steady-state capacity and you can absorb the spike in the queue.
- You need **reliable delivery** with retries and dead-letter handling.
- You're doing **stream processing**: aggregations, joins, windowing over events (Kafka + Flink/Kafka Streams).
- You want **event sourcing / CDC**: the queue is the source of truth, services derive state from it.

Don't reach for a queue when:

- The producer needs an immediate response (use synchronous RPC).
- The work is truly transactional with the producer's commit (use a DB transaction; see [Outbox pattern](databases/transactions.md) for the bridge).
- You only have one consumer and one producer with no spike concerns — the queue is overhead.

## Trade-offs

- **Eventual consistency** by default. A user posts a like → the like is enqueued → the counter updates a moment later. Usually fine; sometimes a UX bug.
- **Operational surface** grows: monitoring queue depth, consumer lag, DLQ depth, broker health.
- **Debugging is harder.** Distributed traces help, but causality across async boundaries needs deliberate instrumentation (trace IDs in message headers).
- **Ordering is limited.** Per-partition / per-queue at best. Global ordering across a high-throughput stream is impossible — design so you don't need it.
- **Duplicates are real.** At-least-once is the practical default; idempotency must be in your design.

!!! warning "Watch consumer lag, not just queue size"
    Queue depth alone is misleading. The metric that matters is **consumer lag**: how far behind real time the consumers are. Lag spikes are early warnings of capacity problems long before anyone notices missing emails.

## Linked problems

Queues appear in almost every realistic system; here are problems where the queue *is* the architecture:

- [Notification System](../problems/notification-system.md) — separate queues per channel (email/SMS/push), workers, retries, DLQ.
- [Web Crawler](../problems/web-crawler.md) — URL frontier is a distributed priority queue (often Kafka).
- [Instagram](../problems/instagram.md) — image processing pipeline triggered by Kafka after upload.
- [YouTube](../problems/youtube.md) — video encoding pipeline: upload → Kafka → encoder workers → S3 → CDN.
- [Twitter](../problems/twitter.md) — fan-out on write goes through a queue; trending topics computed by stream processing.
- [WhatsApp](../problems/whatsapp.md) — offline messages queued for delivery when recipient comes online.
- [Uber](../problems/uber.md) — events stream from rides for surge pricing computation; payment workflows use sagas over queues.
- [Recommendation System](../problems/recommendation-system.md) — real-time updates from event stream, batch retraining triggered offline.

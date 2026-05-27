# Microservices Architecture

Microservices is the practice of building a system as a set of small, independently deployable services, each owning a single business capability and its own data. The opposite is a **monolith**: one codebase, one deployable, one shared database. Both are legitimate architectures; the right choice depends on team size, scale, and how fast you need independent parts to evolve.

## Why it exists

A monolith is the right place for almost every system to start. It's fast to build, easy to test locally, and trivially correct (one DB, one transaction boundary). Real codebases stay monolithic for a long time before any of the microservices arguments kick in.

The pressure to split arises when:

- **Team size grows.** Twenty engineers stepping on each other in one codebase ship slower than four teams of five, each owning a service.
- **Components have very different scaling profiles.** The search service needs 100 nodes; the admin panel needs one. Decoupling lets you scale each independently.
- **A single failure cascades.** A memory leak in the image processor takes down checkout. Independent deploys + isolation help.
- **Technology pluralism is needed.** ML team wants Python, payments team wants Java, real-time team wants Go. Microservices let each pick.
- **Deploy frequency** needs to vary. Search wants to deploy twenty times a day; the payment service wants to deploy carefully on a weekly cadence.

Microservices solve **organizational** problems as much as technical ones (Conway's Law: your architecture mirrors your communication structure). The cost is real and distributed — network failures, eventual consistency, distributed tracing — so the move makes sense only when the pain of the monolith exceeds the pain of distribution.

## How it works

### Monolith vs. microservices

#### Monolith

```
       ┌────────────────────────────┐
       │     Web / Mobile clients   │
       └──────────────┬─────────────┘
                      ▼
          ┌───────────────────────┐
          │   Single Application  │
          │  ┌─────┬─────┬─────┐  │
          │  │User │Order│Cart │  │
          │  │ mod │ mod │ mod │  │
          │  └─────┴─────┴─────┘  │
          └──────────┬────────────┘
                     ▼
                ┌─────────┐
                │   DB    │
                └─────────┘
```

- **Pros**: simple to develop, test, deploy. One transaction boundary. Easy to reason about.
- **Cons**: hard to scale parts independently; one bug can crash the whole app; large codebases slow teams.

#### Microservices

```
       ┌────────────────────────────┐
       │     Web / Mobile clients   │
       └──────────────┬─────────────┘
                      ▼
                ┌──────────┐
                │ Gateway  │
                └─┬────┬──┬┘
        ┌─────────┘    │  └──────────┐
        ▼              ▼             ▼
   ┌─────────┐    ┌─────────┐   ┌─────────┐
   │ Users   │    │ Orders  │   │  Cart   │
   │ + DB    │    │  + DB   │   │  + DB   │
   └─────────┘    └─────────┘   └─────────┘
```

- **Pros**: independent scaling, independent deploys, fault isolation, technology choice.
- **Cons**: network is now part of the design; cross-service consistency is hard; observability and on-call get more complex.

A practical pattern: split *only when the pain demands it.* Many systems run as a "modular monolith" — modules with well-defined interfaces inside one deployable — until specific capabilities need to split off.

### Key patterns

#### Service discovery

**Problem**: how does service A find an instance of service B? IPs change as containers come and go.

**Solution**: a service registry. Services register themselves on startup, deregister on shutdown, and look others up by name.

- **Tools**: Consul, etcd, ZooKeeper, Kubernetes DNS (the de facto default).
- **Approaches**:
    - *Client-side discovery*: client queries registry, picks an instance, calls directly.
    - *Server-side discovery*: client calls a load balancer, which queries the registry.
    - *Kubernetes DNS*: `users-service.default.svc.cluster.local` resolves to a virtual IP backed by a service.

#### Circuit breaker

**Problem**: service B is failing. Service A keeps calling, every call times out, A's threads exhaust waiting, and A fails too. **Cascading failure.**

**Solution**: wrap calls to B in a circuit breaker.

```
States:

   Closed (normal) ──[N consecutive failures]──→ Open (fail fast)
        ▲                                            │
        │                                            │
        └───[probe success]─── Half-Open ←──[timeout elapsed]
```

- **Closed**: normal operation; all calls go through.
- **Open**: after a threshold of failures, the breaker opens — calls fail immediately without touching B. Lets A stay responsive.
- **Half-Open**: after a timeout, allow one probe request. If it succeeds, close the breaker; if it fails, open again.

**Tools**: Hystrix (deprecated, but seminal), Resilience4j, Envoy/Istio circuit breaking, application-level libraries.

!!! tip "Pair circuit breakers with sensible fallbacks"
    A circuit breaker without a fallback turns one error into many faster errors. With a fallback (a cached value, a default response, a graceful degradation), it turns a downstream outage into a slightly worse — but still functional — experience.

#### Saga pattern

**Problem**: a business workflow spans services, but services don't share a database. You can't `BEGIN TRANSACTION` across the network.

**Solution**: a saga — a sequence of local transactions, each with a **compensating transaction** that semantically undoes it.

```
Order placed →
  T1: Create Order   ┐
  T2: Charge Payment │
  T3: Reserve Stock  │ → on failure at step T_k,
  T4: Ship Order     ┘   run compensations T_{k-1}, ..., T_1
```

Coordination can be:

- **Choreography** — each service emits an event when done; the next service listens. Decentralized, can become spaghetti.
- **Orchestration** — a central saga coordinator drives the flow. Easier to reason about; coordinator must be HA.

Deeper coverage in [Transactions](databases/transactions.md).

#### API Composition (Backend for Frontend)

**Problem**: a UI screen needs data from five services. Five round trips per screen.

**Solution**: an API gateway (or a "Backend for Frontend") aggregates from multiple services into a single response.

```
   Mobile App
       ↓
   ┌────────────────┐
   │ Mobile BFF     │  ── aggregates 5 service calls into 1 response
   └─┬─┬─┬─┬─┬──────┘
     ↓ ↓ ↓ ↓ ↓
    User Order Cart Inventory Shipping
```

- The BFF can shape the data exactly for the consumer.
- Reduces chattiness on slow mobile networks.
- Becomes its own service to maintain.

#### Event-driven architecture

Services communicate via **events** instead of direct API calls. A service emits an event ("OrderPlaced") and any number of other services react.

```
Order Service ── publishes "OrderPlaced" ──→ Event Bus
                                                ├─→ Inventory Service (reserves stock)
                                                ├─→ Shipping Service  (schedules shipment)
                                                ├─→ Email Service     (sends confirmation)
                                                └─→ Analytics Service (records event)
```

- Maximal decoupling: producers don't know consumers.
- Adding a new reactive consumer is zero change to the producer.
- Observability is harder — causality is implicit, you need correlation IDs.

See [Message Queues](message-queues.md) for the underlying transport.

### Service communication

Synchronous vs. asynchronous is the most consequential per-call decision.

| | Synchronous (REST/gRPC) | Asynchronous (queue/events) |
|---|------------------------|-----------------------------|
| Pattern | Request–response | Fire-and-forget / publish |
| Coupling | Caller waits for callee | Decoupled in time |
| Availability | Both must be up | Sender can publish even if receivers are down |
| Latency | Caller pays full downstream latency | Sender returns immediately |
| Complexity | Simple to reason about | Eventual consistency, retries, idempotency |
| Use case | User-facing reads/writes that need immediate result | Background work, cross-service events |

A rule of thumb: **synchronous for the read path, asynchronous for everything else.** A user clicks "place order" → the order service writes the order synchronously (so the response says "order #123 confirmed") → publishes an event → inventory, shipping, email, analytics all react asynchronously.

### Per-service data

A core microservices principle: **each service owns its data.** No other service reads its tables directly. Cross-service data needs go through the owning service's API.

This is harder than it sounds. People reach for the shared database constantly, because joins are easy. The discipline that makes microservices work is keeping data ownership clear, even when it costs joins.

When you need data from multiple services in one query, the options are:

- **API composition** — call each service, join in the application.
- **CQRS read model** — a separate denormalized projection populated by events from the source services.
- **Materialize a view** in the consuming service from events emitted by the source services.

## When to use

Use microservices when:

- The team is **large enough to warrant separation** (typically 30+ engineers, multiple teams).
- **Independent deploys** are required for velocity (the search team can't be blocked behind the billing team's release).
- Components have **vastly different scale or technology needs**.
- The system has **clear, durable domain boundaries** — you understand the business well enough to draw the lines.

Stay with a monolith (or "modular monolith") when:

- Team is small. A small team running ten services pays massive ops tax.
- Boundaries aren't clear yet. Bad service boundaries are worse than no service boundaries.
- Cross-service consistency requirements are dense. Splitting tightly-coupled functionality across services makes everything harder.

!!! warning "Don't microservice prematurely"
    The mistake almost every company makes once is breaking into microservices too early. You get all the cost (distributed debugging, network failures, ops surface) and none of the benefit (the team is still small enough to be agile in a monolith). Wait until the monolith is the *demonstrable* bottleneck.

## Trade-offs

- **Network is part of the design.** Every cross-service call can fail, time out, or be slow. You have to handle that.
- **Distributed transactions are hard.** You give up easy ACID across services and inherit sagas, idempotency, and eventual consistency.
- **Operational surface explodes.** Every service is its own deploy, its own monitoring, its own on-call. Service meshes (Istio, Linkerd) and platforms (Kubernetes) help but cost too.
- **Distributed debugging is hard.** A single user action traverses 12 services. You need distributed tracing (see [Monitoring](monitoring.md)).
- **Team coordination changes shape.** Less code coupling, more contract coupling. API versioning and backward compatibility become first-class concerns.

## Linked problems

- [Uber](../problems/uber.md) — many microservices coordinated by sagas: matching, pricing, payment, notifications.
- [Netflix](../problems/netflix.md) — the canonical microservices case study; everything described here was forged there.
- [Instagram](../problems/instagram.md) / [Twitter](../problems/twitter.md) — separate services for feed, social graph, media, search.
- [WhatsApp](../problems/whatsapp.md) — connection servers, message storage, group fan-out as separate services.
- [Notification System](../problems/notification-system.md) — independent workers per channel, all driven by a shared queue.
- [Recommendation System](../problems/recommendation-system.md) — model training, feature store, serving layer are separate services.
- [Distributed Cache](../problems/distributed-cache.md) — used by every other service in a microservices stack.

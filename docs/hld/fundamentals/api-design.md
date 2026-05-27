# API Design

The API is the contract between services and their clients. Designing it well means choosing the right *style* for the workload (REST, GraphQL, gRPC, WebSockets), and then applying a fistful of conventions — versioning, pagination, status codes, error formats — so consumers can integrate quickly and evolve without rewrites.

## Why it exists

Without a clear API contract:

- Every client invents its own way of talking to the service. Some send JSON, some send form-encoded, some use weird HTTP verbs. Every new client is a custom integration.
- Versioning breaks consumers silently. You add a field, a strict parser explodes.
- Authentication, rate limits, errors, and pagination are bolted on inconsistently per endpoint.
- The team's productivity collapses under the weight of integration ping-pong.

A well-designed API solves a sociotechnical problem: it lets dozens of clients integrate with hundreds of endpoints without anyone having to talk to anyone. The right API style for the job, plus a small set of cross-cutting conventions, multiplies the productivity of every team that touches it.

## How it works

### REST

REST (Representational State Transfer) models the system as a set of **resources** addressed by URLs, manipulated via standard HTTP verbs.

#### Principles

- **Resource-based URLs** — nouns, not verbs: `/users`, `/users/42`, `/users/42/orders`.
- **HTTP verbs** carry the action:

| Verb | Use |
|------|-----|
| `GET` | Read; safe, idempotent |
| `POST` | Create; not idempotent (unless you add an idempotency key) |
| `PUT` | Replace; idempotent |
| `PATCH` | Partial update; idempotent if you design it that way |
| `DELETE` | Remove; idempotent |

- **Stateless** — each request carries everything needed to process it. No server-side session state required between requests.
- **Status codes carry meaning**:

| Range | Meaning | Examples |
|-------|---------|----------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 304 Not Modified |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 429 Too Many Requests |
| 5xx | Server error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

#### Best practices

- **Versioning**: `/v1/users`, `/v2/users`. Some teams use a header (`Accept: application/vnd.example.v1+json`); the URL approach is simpler for clients to spot and route.
- **Pagination**: `?page=2&limit=20` for offset-based, or `?cursor=abc123` for cursor-based. Cursor-based scales better for deep pagination and changing data.
- **Filtering / sorting**: `?status=active&sort=-created_at`.
- **Rate limiting**: return `429 Too Many Requests` with a `Retry-After` header.
- **Errors**: consistent JSON shape, e.g. `{ "error": { "code": "INSUFFICIENT_FUNDS", "message": "..." } }`.
- **HATEOAS** — including hypermedia links in responses (`{"next": "/users?page=3"}`). Aspirational; few APIs do it fully. Often skipped for pragmatism.

#### A small REST example

```http
POST /v1/orders
Content-Type: application/json
Idempotency-Key: abc-123

{ "items": [{"sku": "X", "qty": 2}] }

→ 201 Created
  Location: /v1/orders/9001
  { "id": 9001, "status": "pending", ... }
```

### GraphQL

GraphQL exposes a **single endpoint** (typically `/graphql`) and a typed schema. The client sends a query specifying exactly the fields it wants — and only those fields are returned. One request can traverse multiple related entities in a single round trip.

```graphql
query {
  user(id: 123) {
    name
    email
    posts {
      title
      comments(last: 5) {
        author { name }
        text
      }
    }
  }
}
```

#### Pros

- **Flexible**: the client picks the shape of the response. No over-fetching, no under-fetching.
- **Single request** for nested, related data — no chatty round trips.
- **Strongly typed schema** — clients can generate types, tooling is excellent (introspection, playgrounds).
- **Aggregates many backends** — the GraphQL layer can be a thin facade over REST/gRPC services.

#### Cons

- **Caching is hard.** HTTP caching is keyed on URL; one URL means CDN/proxy caches don't work out of the box.
- **Expensive queries.** A malicious or naive client can request deeply nested data and crush your backend. You need query depth limits, complexity analysis, persisted queries.
- **Backend complexity** — N+1 query problems are easy to introduce; you need DataLoader-style batching.

#### When to use

- Mobile apps where bandwidth matters and screens have very different data needs.
- Multiple clients with different needs against the same data graph.
- Complex, deeply nested data that REST would require many round trips to assemble.

### gRPC

gRPC is an RPC framework over **HTTP/2** with **Protocol Buffers** (binary) as the serialization format. You define the service contract in a `.proto` file; clients and servers are code-generated in many languages.

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc StreamUsers(StreamUsersRequest) returns (stream User);
}

message GetUserRequest { string id = 1; }
message User {
  string id    = 1;
  string name  = 2;
  string email = 3;
}
```

#### Why gRPC

- **Fast** — binary protocol on HTTP/2 with multiplexing. 5–10× the throughput of equivalent JSON-over-HTTP/1.1.
- **Strongly typed** — the `.proto` is the contract, and codegen makes the wire format invisible.
- **Streaming** — unary, server-streaming, client-streaming, and bidirectional streaming, all first-class.
- **Polyglot** — codegen across all major languages.

#### When to use

- **Microservice-to-microservice communication.** Internal traffic where latency and throughput matter.
- **High-throughput APIs.**
- **Streaming use cases.**
- **Polyglot environments** — Go service calling a Python service calling a Java service.

#### When *not* to use

- Public-facing APIs where you can't assume HTTP/2 + Protobuf tooling on the client. (gRPC-Web exists but is more work than REST.)
- Browser-direct calls (use REST or GraphQL there; bridge to gRPC at an API gateway).

### WebSockets

A persistent, **bidirectional** TCP connection initiated over HTTP. After an HTTP `Upgrade` handshake, the connection stays open and either side can push messages at any time.

#### Use cases

- **Chat applications** (WhatsApp, Slack).
- **Live notifications** (real-time updates to a UI).
- **Collaborative editing** (Google Docs, Figma).
- **Live dashboards / multiplayer games.**

#### Trade-offs

- The server holds one connection per client → resource cost grows with concurrent users.
- Load balancing is harder (sticky connections, least-connections).
- Reconnect logic, heartbeats, backpressure all need explicit handling.
- For purely server → client updates without bidirectional needs, Server-Sent Events (SSE) is simpler. See [Networking](networking.md).

### API Gateway pattern

An **API gateway** is the single entry point for all client requests in a microservices architecture. It sits between clients and your backend services, handling cross-cutting concerns so every service doesn't have to.

```
                ┌─────────────────┐
                │   API Gateway   │
                │  - routing      │
   Clients ───→ │  - auth         │
                │  - rate limit   │
                │  - aggregation  │
                │  - SSL terminate│
                └────┬────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   User Svc     Order Svc    Payment Svc
```

#### Responsibilities

- **Routing**: `/users/*` → user service, `/orders/*` → order service.
- **Authentication & authorization**: validate JWTs once, pass user identity downstream.
- **Rate limiting** and quota enforcement.
- **Request/response transformation** — version skew, header normalization.
- **TLS termination.**
- **Caching** of cacheable responses.
- **Monitoring & logging** — central observability.
- **API composition** — aggregate multiple service calls into one response (the "Backend for Frontend" pattern).

**Tools**: Kong, AWS API Gateway, Apigee, Azure API Management, Envoy + Istio, Traefik.

!!! warning "Don't let the gateway become a monolith"
    A common anti-pattern is stuffing business logic into the API gateway. Keep it focused on cross-cutting concerns. Business decisions belong in services.

## When to use

| Need | Use |
|------|-----|
| Public web API consumed by random clients | REST + JSON |
| Mobile app with screens of different shapes | GraphQL |
| Microservice-to-microservice, low latency | gRPC |
| Real-time bidirectional updates | WebSockets |
| Server-to-client push only | SSE (simpler than WebSockets) |
| One entry point for many services | API Gateway |

A common production stack: REST or GraphQL at the public edge → API gateway handles auth/routing/rate-limit → internal services talk over gRPC → realtime channels over WebSockets where needed.

## Trade-offs

- **REST** is universal and well-tooled but can be chatty for nested data.
- **GraphQL** removes round trips but trades for query complexity and caching headaches.
- **gRPC** is fast and typed but harder to debug (binary), harder for browsers, and demands schema discipline.
- **WebSockets** are flexible but operationally heavy and stateful.
- **An API gateway** centralizes useful concerns but becomes a deployment chokepoint and SPOF — invest in its HA.

!!! tip "Pick one as the default, allow others where they fit"
    Teams that fight constant style-debates spend less time building. Pick a default (e.g. REST externally, gRPC internally), document when it's okay to deviate (e.g. WebSocket for chat), and let the boring choice win most arguments.

## Linked problems

- [URL Shortener](../problems/url-shortener.md) — small, clean REST API: `POST /shorten`, `GET /:code`.
- [Twitter](../problems/twitter.md) / [Instagram](../problems/instagram.md) — REST endpoints + GraphQL exploration for feed flexibility.
- [Uber](../problems/uber.md) — REST for trip lifecycle, WebSockets for live driver/rider location, gRPC for internal microservices.
- [WhatsApp](../problems/whatsapp.md) — WebSockets are the core API; REST for media uploads and account.
- [YouTube](../problems/youtube.md) — REST for metadata, HLS/DASH HTTP for video streaming.
- [Notification System](../problems/notification-system.md) — REST API to enqueue notifications; downstream via queues.
- [Rate Limiter](../problems/rate-limiter.md) — almost always deployed at the API gateway layer.
- [Search Autocomplete](../problems/search-autocomplete.md) — GET endpoint with cached prefix responses at the CDN.

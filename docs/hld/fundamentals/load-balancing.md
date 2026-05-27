# Load Balancing

A load balancer is the traffic cop in front of a fleet of servers. It receives client requests, picks a backend to handle each one, and forwards the request along. The goals are simple: spread load evenly, route around failures, and survive traffic spikes without anyone behind it noticing.

## Why it exists

A single web server can handle some number of requests per second. The moment your traffic exceeds that, you need a *fleet* of identical servers behind a single public address — and something that decides which server handles each incoming request. That something is the load balancer.

But load balancing isn't just about *capacity*. Three other concerns make it essential:

1. **Fault tolerance.** Servers crash, deploys fail, kernels panic. The load balancer detects unhealthy servers and stops sending them traffic, so a single bad node doesn't take the service down.
2. **Elastic scaling.** Add servers under load, remove them when load drops — the load balancer transparently incorporates the new nodes.
3. **Operational flexibility.** Rolling deploys (drain one node at a time), blue/green deployments, canaries — all rely on a load balancer to direct traffic.

A system without a load balancer either has a single point of failure or punts the routing problem to the client (DNS round-robin, custom client logic), which is much harder to evolve.

## How it works

### Algorithms

The load balancer's main job is picking which backend gets each request. The choice depends on how uniform the backends are, what kind of state they hold, and how variable individual requests are.

- **Round Robin** — cycle through backends in order. Simple and works well when servers are identical and requests have similar cost. The default for most setups.
- **Weighted Round Robin** — backends with higher weights receive proportionally more traffic. Use when capacity is heterogeneous (a 16-core box gets twice the weight of an 8-core).
- **Least Connections** — route the next request to whichever backend currently has the fewest open connections. Best for long-lived connections (WebSockets, database connections) where some sessions are much heavier than others.
- **Least Response Time** — route to the backend with the lowest current measured response time. Useful when latency varies (e.g. backends with cold/warm caches).
- **IP Hash (Sticky Sessions)** — hash the client IP and always route a given client to the same backend. Useful when the backend keeps in-memory session state. Caveats: bad NAT distribution can create hotspots; one big proxy IP can pin many clients to a single backend.
- **Random** — pick uniformly at random. Surprisingly effective at high scale because randomness smooths out micro-imbalances; pairs well with the "power of two random choices" (pick two at random, send to the less loaded one).

| Algorithm | Best when | Watch out for |
|-----------|-----------|---------------|
| Round Robin | Servers identical, requests uniform | Heterogeneous request cost causes uneven load |
| Weighted RR | Heterogeneous capacity | Weights drift out of sync with actual hardware |
| Least Connections | Long-lived connections | Slow backends accumulate connections, get more traffic |
| Least Response Time | Variable latency | Newly-added warm-but-empty backends look fast and get flooded |
| IP Hash | Need session affinity | Skew from NAT'd clients; rebalancing hurts during scaling |
| Random | High scale, stateless | None significant; combine with power-of-two-choices |

!!! tip "Prefer stateless backends"
    Sticky sessions are a sign your backend is holding state it shouldn't. Push state into a shared store (Redis, DB) and any backend can serve any request. You gain flexibility and lose a class of bugs.

### Layer 4 vs Layer 7

Load balancers operate at different layers of the OSI model, with very different capabilities.

#### Layer 4 (Transport Layer)

Routes based on IP address and port. The balancer sees a TCP/UDP stream — bytes in, bytes out — and never inspects the payload.

- Very fast (no parsing, no buffering).
- Works for any protocol over TCP/UDP, not just HTTP.
- Cannot route based on URL, header, or cookie.
- **Tools**: HAProxy (TCP mode), AWS NLB, IPVS.

#### Layer 7 (Application Layer)

Understands HTTP, gRPC, or other application protocols. Can route on URL path, host header, cookie, query string — anything in the request.

- Can route `/api/users/*` to the user service and `/api/orders/*` to the order service.
- Can terminate TLS, rewrite headers, retry failed requests, transform responses.
- Slower than L4 (parsing, buffering, more state per connection).
- **Tools**: Nginx, HAProxy (HTTP mode), AWS ALB, Envoy, Traefik.

A common production layout uses both: an L4 balancer at the very edge for raw throughput and DDoS absorption, with L7 balancers (or service meshes like Envoy) inside doing intelligent routing.

```
       internet
          ↓
   ┌─────────────┐
   │  L4 (NLB)   │  ← raw TCP, very fast
   └──────┬──────┘
          ↓
   ┌─────────────┐
   │  L7 (ALB)   │  ← URL routing, TLS termination
   └──┬──────┬───┘
      ↓      ↓
 ┌────────┐ ┌────────┐
 │ Users  │ │ Orders │
 │ Svc    │ │ Svc    │
 └────────┘ └────────┘
```

### Patterns

#### DNS load balancing

Return multiple A records for the same domain. Clients pick one — usually the first in the list, sometimes randomly. Most resolvers cache the records, so it's a coarse, slow lever.

- **Pros**: zero infrastructure, free.
- **Cons**: caching defeats fine-grained control; can't react quickly to failures; uneven distribution because of resolver caching.
- **Use**: as the *first* layer in front of regional load balancers — not as your primary balancer.

#### Global Server Load Balancing (GSLB)

DNS-based routing with geographic awareness. Resolve a domain to the *nearest* datacenter based on the client's IP or GeoDNS rules. Often combined with health checks: if a datacenter is unhealthy, GSLB stops routing traffic there.

- **Tools**: Route 53 (latency-based routing), Cloudflare, NS1, Akamai.
- **Use**: multi-region services where you want users to hit the closest region.

#### Health checks

The load balancer pings each backend periodically (usually a `/health` endpoint) and removes unhealthy backends from rotation. Always mention this in interview designs.

```
GET /health
→ 200 OK              ← keep in rotation
→ 500 / timeout / 0×N ← remove from rotation
```

Two flavors:

- **Active health checks** — the balancer initiates requests. Standard.
- **Passive health checks** — the balancer marks a backend unhealthy after it observes a streak of failed real requests. Cheaper, but slower to detect.

A good `/health` endpoint checks the dependencies the service actually needs (DB connectivity, cache, downstream services) — not just "the process is running". A "deep" vs "shallow" health check is a common interview discussion.

### High availability of the load balancer itself

The load balancer is now a single point of failure. Three common patterns:

- **Active-passive pair** with a floating virtual IP. If the active node dies, the passive takes over the IP (keepalived, VRRP).
- **Active-active** behind DNS or anycast.
- **Cloud-managed**: AWS ALB/NLB, GCP LB — these are themselves redundant fleets you don't manage.

## When to use

- **More than one backend**, full stop. Even two-server services should have a load balancer.
- **Multi-region traffic** — GSLB at the edge, regional LB inside each region.
- **Microservices** — typically a service mesh (Envoy) handles internal LB transparently.
- **Long-lived connections** (WebSockets, gRPC streams) — use least-connections or a connection-aware policy.
- **Mixed-cost endpoints** — separate L7 routing rules per endpoint, so a slow `/report` doesn't crowd out `/login`.

## Trade-offs

- **More moving parts.** Every layer is one more thing that can fail, drop packets, or be misconfigured.
- **L7 features cost CPU and latency.** TLS termination, regex routing, retries, observability — all worth it, but not free.
- **Sticky sessions are a tax** on horizontal scale. They make rebalancing painful and add coupling.
- **Health checks have a detection window.** Between a backend going bad and the LB noticing, real users see errors. Tune intervals down with care (too aggressive = false positives and flapping).
- **The LB becomes a deployment bottleneck.** Config changes, certificates, routing rules all funnel through it — make config changes versioned and revertible.

!!! warning "Beware misconfigured timeouts"
    A classic outage pattern: LB timeout is shorter than backend timeout. The backend keeps working on the request, the LB gives up and retries, the backend now has duplicate work. Always make sure timeouts increase as you go deeper (client < LB < service < DB), and that retries are bounded.

## Linked problems

Every multi-server design has a load balancer; here are problems where the LB design is non-trivial:

- [URL Shortener](../problems/url-shortener.md) — L7 LB in front of app servers; read replicas for redirect traffic.
- [Instagram](../problems/instagram.md) / [Twitter](../problems/twitter.md) — L4 at edge, L7 internally; CDN absorbs static traffic.
- [Uber](../problems/uber.md) — WebSocket-aware LB (least connections); GSLB for regional driver matching.
- [WhatsApp](../problems/whatsapp.md) — connection servers behind least-connections LB; persistent WebSocket affinity.
- [YouTube](../problems/youtube.md) / [Netflix](../problems/netflix.md) — GSLB plus CDN-level routing to the nearest edge.
- [Distributed Cache](../problems/distributed-cache.md) — client-side load balancing via consistent hashing (a different shape of the same idea).
- [Rate Limiter](../problems/rate-limiter.md) — often *deployed at* the LB / API gateway layer.

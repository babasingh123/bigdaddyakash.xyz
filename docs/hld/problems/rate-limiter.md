# Design a Rate Limiter

> **Difficulty:** Intermediate · **Time:** 30–40 minutes

## Problem

Design a distributed rate limiter that protects an API from abuse. The
system must:

- Allow N requests per minute (or per second) per user / API key / IP.
- Be **distributed** — the limit applies across many API servers, not
  per-host.
- Be **fast** — adding rate-limit checks should add <1 ms to every
  request.
- Be **accurate** — must not over-limit (legitimate users blocked) or
  under-limit (limit blown past by 10x).
- Return a clear response (`429 Too Many Requests` with `Retry-After`
  header).

## Real-world intuition

Rate limiting feels like a small feature ("just count the requests"),
but in any system at scale it sits on the **hottest** path in
infrastructure. Every API call goes through it. Every millisecond it
adds is paid by every request. Every storage trip it makes is
multiplied by total QPS.

So the design problem isn't "how do I count" — it's "how do I count
*correctly, atomically, and globally* while adding zero observable
latency." That answer almost always involves Redis + atomic ops or Lua
scripts, with careful attention to the race conditions you'd otherwise
hit at distributed scale.

Underneath that, the choice of algorithm (fixed window, sliding window,
token bucket, leaky bucket) shapes burst behavior, fairness, and memory
cost. Picking the right one is a real architectural decision, not a
detail.

## Approach

### Capacity estimation

- **Request volume:** assume 1M requests/sec across the API.
- **Storage per key:** a counter + TTL ≈ 50 bytes.
- **Active keys at any moment:** 10M (10M unique IPs / users in the
  window) × 50 B = **500 MB** in Redis. Fits one node; cluster for HA.
- **Latency budget:** sub-ms median, single-digit ms p99. One Redis
  round-trip = 0.5 ms typical.

### Choosing an algorithm

| Algorithm        | Burst behavior        | Memory       | Implementation     |
| ---------------- | --------------------- | ------------ | ------------------ |
| **Fixed window** | Boundary burst (2× N) | O(1) per key | INCR + EXPIRE      |
| **Sliding window log** | Smooth                | O(N) per key | Sorted set         |
| **Sliding window counter** | Approx smooth         | O(1) per key | Two counters       |
| **Token bucket** | Allows bursts up to B | O(1) per key | Atomic Lua script  |
| **Leaky bucket** | Smooths to steady rate | O(1) per key | Background drain   |

For most APIs, **token bucket** is the right default: it lets short bursts
through (good UX) but enforces an average rate (good protection). Sliding
window counter is the most efficient "no burst" option.

#### Why fixed window is dangerous

```
limit = 100 / minute
00:00:59 → 100 requests pass (window 00:00)
00:01:01 → 100 more pass (window 00:01)
=> 200 requests in 2 seconds
```

That 2× burst at boundaries is enough to take down systems whose capacity
was sized to "100/min." Sliding window or token bucket fix this.

### Token bucket with Redis (Lua)

The bucket stores `(tokens, last_refill_ts)`. Every request:

1. Read current tokens and last refill.
2. Add `(now - last_refill) * rate` tokens (capped at burst size).
3. If `tokens >= 1`, decrement and allow. Else reject.

Steps 1–3 must be atomic across all API servers. A naive `GET`/`SET`
sequence has a race window. Redis Lua executes server-side atomically:

```lua
-- KEYS[1] = "rl:user:42"
-- ARGV: rate, capacity, now_ms, requested
local data = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or tonumber(ARGV[2])
local ts = tonumber(data[2]) or tonumber(ARGV[3])
local rate, cap, now, req = tonumber(ARGV[1]), tonumber(ARGV[2]),
                            tonumber(ARGV[3]), tonumber(ARGV[4])

-- refill
tokens = math.min(cap, tokens + (now - ts) / 1000.0 * rate)
if tokens >= req then
    tokens = tokens - req
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
    redis.call('PEXPIRE', KEYS[1], 60000)
    return 1
else
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
    redis.call('PEXPIRE', KEYS[1], 60000)
    return 0
end
```

One round-trip, atomic, ~0.3 ms.

!!! note "Why Lua, not MULTI/EXEC?"
    MULTI/EXEC queues commands but doesn't run them as a single logical
    operation against current state — `WATCH/OPTIMISTIC` patterns are
    needed and they retry under contention. Lua scripts truly execute
    atomically inside Redis's single-threaded loop. At 1M QPS that
    difference matters.

### Where to apply: API gateway is canonical

```
[Client]
   │
   ▼
[API Gateway / LB / Edge]      ◀── rate-limit check here
   │
   ▼
[Service A]   [Service B]      ◀── per-service limits if needed
```

Putting the check at the gateway protects the whole backend from one
hop. A second, finer-grained limit inside the service catches abuse that
slipped past (e.g., expensive operations).

### Sliding window counter (cheap, accurate)

Approximate sliding window with **two fixed-window counters**:

```
limit = 100/min
window_current  = INCR rl:user:42:00:01     # current minute
window_previous = GET  rl:user:42:00:00     # previous minute

weight = (60 - seconds_into_current_minute) / 60
allowed = (window_previous * weight) + window_current <= 100
```

This is O(1) memory, smooths the boundary, and is the algorithm used in
Cloudflare's rate limiter.

### Distributed concerns

- **Where Redis lives.** Either a single shared Redis cluster (sharded by
  key) or a Redis instance co-located with each API region. Cross-region
  rate limiting (one global limit) is harder — usually accept eventual
  consistency.
- **Hot keys.** A famous IP getting hammered concentrates load on one
  Redis shard. Mitigate with client-side pre-checks ("if my local counter
  says I'm above, don't even ask Redis") or rate-limit at the CDN edge.
- **Redis failure.** Fall back to **fail-open** (allow but log) for most
  APIs — better to accept some over-limit than to reject everyone. For
  abuse-prone endpoints, **fail-closed**.

### Architecture diagram

```
                ┌───────────────────┐
                │    Clients         │
                └─────────┬──────────┘
                          │
                  ┌───────▼────────┐
                  │  API Gateway   │
                  │ ┌────────────┐ │
                  │ │ Rate limit │ │──┐
                  │ │  module    │ │  │ Lua eval
                  │ └────────────┘ │  │
                  └───────┬────────┘  │
                          │ allowed   ▼
                          │     ┌──────────────────┐
                          │     │   Redis Cluster  │
                          │     │  (sharded by key)│
                          ▼     └──────────────────┘
                  ┌────────────┐
                  │  Services  │
                  └────────────┘
```

The gateway includes a thin SDK that calls the Lua script. Reject ⇒
`429`. Allow ⇒ forward request and emit a metric.

### HTTP semantics

On rejection, return a useful response:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716817200
```

Clients with good backoff (and your SDKs) use these headers to behave;
rude clients don't, but at least they're protected by the same response.

## Trade-offs

- **Single Redis vs. cluster.** Single is simpler, sharded handles more
  QPS. Most rate limiters fit in one cluster; per-key locality is fine.
- **Strict vs. approximate counts.** Strict (sliding window log) is
  accurate but expensive. Approximate (sliding counter) loses ~1%
  accuracy for huge memory savings. Pick approximate unless you're
  selling billing-grade enforcement.
- **Fail-open vs. fail-closed.** Open prioritizes availability; closed
  prioritizes protection. Most public APIs go open with a separate
  WAF/circuit-breaker for true overload.
- **Per-user vs. per-IP vs. per-key.** Mobile users behind NAT share IPs
  — IP-only is unfair. Most systems combine: limit per-key (authenticated)
  *and* per-IP (unauthenticated) *and* a global circuit breaker.

## Edge cases & gotchas

- **Clock skew across servers.** If your `now_ms` comes from each API
  server, NTP drift creates inconsistencies. Use Redis `TIME` command as
  the source of truth.
- **Race on first request.** Two concurrent first requests both find no
  key, both `INCR` to 1. Lua atomicity fixes this; naive code doesn't.
- **TTL expiry exactly at boundary.** A request landing exactly when the
  key expires triggers a fresh window. Acceptable for fixed window;
  invisible for token bucket.
- **Bursty legitimate traffic.** A mobile app retrying after a server
  blip can synchronize millions of clients to retry the same second.
  Add jitter to client retry, and use token-bucket so a moderate burst
  is allowed.
- **Distributed cache stampede.** When a popular user crosses the limit
  for the first time, every API server hits Redis on the same key for
  the next second. Lua + single round-trip absorbs it.
- **Different limits for different endpoints.** Use composite keys:
  `rl:{user_id}:{endpoint}` so `/search` and `/checkout` have separate
  buckets.
- **DDoS at L4.** Rate limiter protects L7 capacity, not raw bandwidth.
  Combine with CDN/WAF + DDoS scrubbing.
- **Testing.** Rate limiters are deceptively hard to test — make them
  configurable (rate, capacity) at runtime so you can dial down for
  perf tests.

!!! tip
    For interviews, draw the Lua atomicity diagram. Explaining the
    race condition without it and then showing how the script removes
    it is what separates a "good" answer from a great one.

## Linked concepts

- [Rate limiting](../fundamentals/rate-limiting.md) — algorithms in depth
- [Caching strategies](../fundamentals/caching.md) — Redis as the state store
- [API design](../fundamentals/api-design.md) — 429 responses, Retry-After headers
- [Distributed systems](../fundamentals/distributed-systems.md) — consistency, atomicity
- [Load balancing](../fundamentals/load-balancing.md) — putting the limiter at the edge
- [Misc patterns](../fundamentals/misc-patterns.md) — token bucket as a general pattern

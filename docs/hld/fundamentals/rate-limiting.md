# Rate Limiting & Throttling

Rate limiting is the discipline of capping how often a client can call your service over a time window. Throttling is the act of slowing down or rejecting requests that exceed that cap. Both protect your system from being overwhelmed, prevent abuse, and enforce fair use among customers.

## Why it exists

Without rate limiting:

- **A buggy client in a tight retry loop can take down your service.** No malice required.
- **Bot traffic, credential stuffing, scrapers** consume capacity meant for paying users.
- **A single tenant's spike** degrades every other tenant's experience.
- **Free-tier abuse** drives up your cloud bill.
- **DDoS attacks** sail through with no friction.

Rate limiting solves all of these from one place. It's the cheapest defense per dollar in your security and reliability budget.

A subtler reason: rate limits make your system's behavior **predictable** to clients. They can build retry logic, surface a useful error message, and design their integration around clear quotas — instead of being surprised by a vague "service unavailable" at unpredictable times.

## How it works

### Algorithms

The four canonical algorithms differ in how they smooth or burst traffic and how much state they need.

#### Token Bucket

```
- Bucket holds N tokens (capacity)
- Tokens refill at rate R per second
- Each request consumes 1 token
- If no tokens, reject (or queue) the request
```

A request can grab a token instantly if the bucket has one, but tokens accumulate only up to capacity `N`. So a client can burst up to `N` requests if they've been quiet, then is throttled to the refill rate `R`.

- **Pros**: Allows controlled bursts (good UX for occasional spikes). Smooths over short variance.
- **Cons**: A bit of state per client (current token count + last refill time).
- **Use**: The flexible default. AWS, Stripe, and most public APIs use a variant.

#### Leaky Bucket

```
- Requests enter a fixed-size bucket
- Bucket drains at constant rate R
- If bucket full, reject new requests
```

Conceptually a queue with a fixed drain rate. Bursts go in but emerge at the rate `R`, smoothing traffic.

- **Pros**: Strictly smooths traffic to a constant rate.
- **Cons**: No bursts allowed; the user notices the smoothing as added latency.
- **Use**: When you must protect a downstream that can't tolerate bursts (e.g. a fragile third-party API).

#### Fixed Window

```
- Counter per (user, window) — e.g. (user_42, minute_10:30)
- Allow N requests per window
- Counter resets at the start of each new window
```

- **Pros**: Trivial to implement. One counter, one TTL.
- **Cons**: **Boundary problem** — a user can burst 2× the limit by sending N at 10:30:59 and N at 10:31:00.
- **Use**: Quick-and-dirty when accuracy doesn't matter much.

#### Sliding Window

A more accurate version that smooths the boundary problem. Two common variants:

- **Sliding window log** — store a timestamp for each request; on each new request, drop timestamps older than the window and count the rest. Accurate but memory-heavy (one entry per request).
- **Sliding window counter** — approximate by interpolating between the current and previous fixed window: `count ≈ current_window_count + previous_window_count × overlap_fraction`. Cheap and accurate enough.

- **Pros**: No boundary spike. Accurate.
- **Cons**: More state (log) or more logic (counter).
- **Use**: When boundary spikes matter (financial APIs, abuse-sensitive endpoints).

### Visual comparison

| Algorithm | Allows bursts? | Smooths traffic? | State per key | Implementation |
|-----------|----------------|------------------|---------------|----------------|
| Token bucket | Yes (up to N) | Partially | `(tokens, last_refill)` | Medium |
| Leaky bucket | No | Yes | `(level, last_leak)` | Medium |
| Fixed window | At boundaries | No | `(count, window)` | Easy |
| Sliding window log | No | Yes | List of timestamps | Hard |
| Sliding window counter | Slightly | Yes | Two counters | Medium |

### Where to apply

Rate limits are not one-size-fits-all. You usually layer them by *who is asking*:

| Scope | Purpose |
|-------|---------|
| **Per user** | Fairness across tenants; abuse prevention. |
| **Per IP** | DDoS / scraping; rough abuse signal. Caveat: NAT and proxies share IPs. |
| **Per API key** | Tier enforcement (free vs. paid). |
| **Per endpoint** | Protect expensive operations (e.g. `/search` vs. `/health`). |
| **Global** | Hard cap protecting your backend regardless of who's calling. |

A real system applies several of these in combination. A free-tier user hitting `/expensive-endpoint` from a noisy IP might pass the per-user limit, fail the per-endpoint limit, and trigger the per-IP block — all in one request.

### Where to enforce

| Layer | Reason to put it here |
|-------|----------------------|
| **CDN / edge** (Cloudflare, etc.) | Stops bad traffic before it hits origin. Cheap. |
| **API gateway** (Kong, Envoy, AWS API Gateway) | Central, language-agnostic, easy to maintain. |
| **Reverse proxy** (Nginx) | Simple rules for legacy stacks. |
| **Application code** | Fine-grained business logic limits (per-org quotas, per-feature). |
| **WAF** | Pattern-based abuse detection (bots, credential stuffing). |

In practice, you stack them: CDN handles L3/L4 floods, API gateway handles per-user/per-API-key, application code handles business quotas.

### Implementation: Redis as the store

The standard distributed rate-limiter implementation uses Redis because of three properties:

- **Atomic operations** (`INCR`, `INCRBY`, `EVAL` Lua scripts) avoid race conditions across multiple app servers.
- **TTL** built in (`EXPIRE`) for window-based limits.
- **Single-digit-millisecond latency** so the limiter doesn't itself become a bottleneck.

A fixed-window Redis recipe:

```
key = "rl:user_42:minute_10:30"
count = INCR key
if count == 1:
    EXPIRE key 60
if count > LIMIT:
    return 429 Too Many Requests
else:
    allow
```

Two operations need atomicity together; use `MULTI`/`EXEC` or a Lua `EVAL` script to combine them.

A token-bucket Redis recipe is a Lua script: on each request, compute how many tokens to refill based on the elapsed time since `last_refill`, store updated state, decide whether to allow.

### Response and headers

When you reject, give the client what they need to behave better:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716800000
```

`Retry-After` is the only one strictly defined by spec; the others are de facto. Together they let well-behaved clients back off cleanly.

### Throttling vs. blocking

Two responses to "over limit":

- **Hard reject** (`429`) — fast, predictable. Use for public APIs with documented limits.
- **Soft throttle** (slow the response, queue, or reduce concurrency) — invisible to the client, no error UX. Use for trusted internal clients.

## When to use

Always. Every public endpoint should have *some* rate limit, even if generous. The specific question is *what* limits, applied *where*.

Most aggressive limits go on:

- Login and signup endpoints (credential stuffing, account creation abuse).
- Search and other expensive read endpoints.
- Write endpoints (anti-spam, anti-abuse).
- Any endpoint that hits external paid APIs.

Lighter limits on:

- Health checks and metrics.
- Internal-only endpoints (already protected by network policy).

## Trade-offs

- **False positives.** A shared corporate NAT can blow through per-IP limits with legitimate traffic. Combine signals.
- **Distributed accuracy is hard.** True global rate limits across regions and shards need either a centralized Redis (latency cost, SPOF) or sophisticated approximate algorithms (gossip, sketches).
- **Bursty traffic UX.** Strict smoothing (leaky bucket) adds latency; loose smoothing (token bucket with high capacity) allows bursts that hurt downstreams.
- **Configuration drift.** Limits set during architecture review rarely get revisited as the system evolves. Track them in code, review periodically.
- **Cache penetration.** A rejected request still costs you a Redis call. At extreme DDoS scale, the rate limiter itself becomes a target; put a CDN in front.

!!! warning "Always alert on 429 rates"
    A spike in `429`s could be a malicious actor, a buggy client, or your own internal service in a retry loop. All three are urgent in different ways. Make the dashboard exist before you need it.

## Linked problems

- [Rate Limiter](../problems/rate-limiter.md) — the canonical "design a distributed rate limiter" problem; covers algorithm choice and Redis implementation in depth.
- [URL Shortener](../problems/url-shortener.md) — per-user / per-IP limits on `POST /shorten`.
- [Notification System](../problems/notification-system.md) — outbound rate limits to respect third-party API quotas (Twilio, SendGrid, FCM).
- [Web Crawler](../problems/web-crawler.md) — politeness = per-domain rate limiting on outbound crawls.
- [Twitter](../problems/twitter.md) / [Instagram](../problems/instagram.md) — per-user and per-API-key limits across the API.
- [Search Autocomplete](../problems/search-autocomplete.md) — rate-limited high-frequency queries from typing UIs.

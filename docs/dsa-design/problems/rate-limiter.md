# Design Rate Limiter

!!! note "The disguise"
    Interviewer says: **"Design an API rate limiter."**
    They mean: **"Implement a sliding window with a Queue of timestamps per user."**

## Problem

Build a rate limiter that allows each user at most *N* requests per *W* seconds (a sliding window). API:

- `allow_request(user_id, timestamp) -> bool` — return `True` and record the request, or `False` (rate limited).

Variants worth knowing:

- **Fixed window** — bucket by floor(timestamp / W). Simple, but bursty at boundaries.
- **Sliding window log** — exact, but stores every timestamp.
- **Token bucket** — capacity + refill rate.
- **Leaky bucket** — see [Leaky Bucket](leaky-bucket.md).

This page focuses on the sliding window log, which is the standard coding-interview answer.

## What it really tests

- **Sliding window technique**, applied to *time* instead of array indices.
- **Queue (Deque) of timestamps** with O(1) push to tail and O(1) pop from head.
- **HashMap (user → Deque)** for per-user windows.
- **Naming the variants.** Most interviewers want you to mention fixed vs sliding.

## Approach

The data we need per user is the set of recent request timestamps. We need:

- Fast append (new request).
- Fast removal of stale entries (timestamp < now - W).
- Fast size check (against N).

A **Deque** gives all three: `append` is O(1), `popleft` is O(1), `len` is O(1).

**The algorithm.**

```
on allow_request(user, now):
    q = users[user]
    while q and q[0] <= now - W: q.popleft()    # evict expired
    if len(q) >= N: return False
    q.append(now); return True
```

That's the whole thing. Each timestamp is appended once and removed once → **amortized O(1)** per call.

**Why a queue and not, say, a set?** Order matters — the oldest entry is the first to expire, and a Deque naturally exposes head and tail in O(1). A set would force you to scan to find the oldest.

**Why per-user, not global?** The problem says "per user" — that's what the HashMap is for. If the requirement were global, one Deque would do.

**Variants to mention:**

- **Fixed window** uses a single counter that resets at every period boundary → cheap (O(1) counter increment) but allows 2N requests across a boundary.
- **Sliding window counter** approximates the sliding window log with weighted contributions from the previous bucket → constant memory per user (instead of N timestamps), tiny accuracy loss.
- **Token bucket** refills *r* tokens per second up to capacity *N* and lets requests burst → great for "smoothing" traffic.

The interview win is "I'd start with sliding window log for accuracy; in production I might switch to token bucket for memory."

## Code sketch

```python
from collections import defaultdict, deque

class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.N = max_requests
        self.W = window_seconds
        self.users: dict[str, deque[int]] = defaultdict(deque)

    def allow_request(self, user_id: str, now: int) -> bool:
        q = self.users[user_id]
        cutoff = now - self.W
        while q and q[0] <= cutoff:
            q.popleft()
        if len(q) >= self.N:
            return False
        q.append(now)
        return True
```

For **distributed** systems, you'd back this with Redis: a sorted set per user with timestamps as scores. `ZREMRANGEBYSCORE` evicts old entries; `ZCARD` checks count; `ZADD` records the new one. Same algorithm, persisted.

## Complexity

- `allow_request`: **amortized O(1)** per call. Each timestamp is inserted and removed once over its lifetime.
- Space: **O(active users × N)** in the worst case (each queue holds at most N).

## Edge cases

- **Timestamps not monotonically increasing.** If clients can send out-of-order timestamps, the sliding-window eviction breaks. Use `time.monotonic()` server-side.
- **Burst right at expiry.** Be precise about `<=` vs `<` in the cutoff comparison. `q[0] <= now - W` means a request from exactly W seconds ago has just expired.
- **Stale users keep their queues forever.** Add a periodic cleanup or use an LRU on the user map.
- **Multi-process / multi-server deployments.** Local rate limiter is per-process. For true global limits, externalize to Redis.

## Linked concepts

- [Rate Limiting & Counting category overview](../by-category/rate-limiting-counting.md)
- [Rate limiting (HLD)](../../hld/fundamentals/rate-limiting.md)
- [Hit Counter](hit-counter.md) — the global-counter version of this
- [Leaky Bucket](leaky-bucket.md) — alternative rate-shaping algorithm

# Design Leaky Bucket

!!! note "The disguise"
    Interviewer says: **"Design a traffic shaping algorithm."**
    They mean: **"Implement a fixed-capacity queue that drains at a constant rate."**

## Problem

Implement a leaky bucket with two parameters:

- `capacity` — maximum number of requests the bucket can hold.
- `leak_rate` — requests drained per second.

API:

- `allow(timestamp) -> bool` — try to admit a new request. Return `True` if the bucket has room, `False` if it's full.

The bucket "leaks" between calls: at any moment, the effective fill level is `last_fill - (now - last_time) * leak_rate`, floored at zero.

## What it really tests

- **Continuous-time accounting.** Unlike sliding-window log, you don't store every event — you store a single fill level and update it lazily.
- **Queue or counter abstraction.** Two natural implementations: a literal queue of items or just a single number.
- **Comparison with token bucket.** Leaky bucket smooths traffic; token bucket allows bursts.

## Approach

There are two equivalent ways to think about leaky bucket:

### View 1: Counter (recommended)

Track:

- `water` — current fill level (a float).
- `last_time` — timestamp of the last update.

On every `allow(ts)`:

1. Compute the leak since `last_time`: `leaked = (ts - last_time) * leak_rate`.
2. `water = max(0, water - leaked)`.
3. If `water < capacity`, add 1 (admit the request) and return `True`.
4. Otherwise, return `False`.
5. `last_time = ts`.

This is **O(1) per call**, **O(1) memory**.

### View 2: Literal queue

Keep a FIFO of admitted requests. A background process pops one every `1 / leak_rate` seconds. Reject if `len(queue) == capacity`.

This is the *operational* picture used in network shapers — it produces an output stream at a constant rate. For pure rate-limiting decisions you don't need the actual queue; the counter view captures the answer.

**Leaky bucket vs token bucket — the question interviewers ask:**

- **Leaky bucket** drains at a constant rate. Output is *smoothed* — bursts are not allowed.
- **Token bucket** refills tokens at a constant rate. A request consumes one token; if tokens are available, allow. Output *can be bursty up to the bucket capacity*.

The mental model: leaky bucket is rate *enforcement* (the network sees a uniform stream); token bucket is rate *budgeting* (the network sees bursts up to capacity).

## Code sketch

```python
class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.water = 0.0
        self.last_time: float | None = None

    def allow(self, ts: float) -> bool:
        if self.last_time is not None:
            leaked = (ts - self.last_time) * self.leak_rate
            self.water = max(0.0, self.water - leaked)
        self.last_time = ts
        if self.water < self.capacity:
            self.water += 1
            return True
        return False
```

The literal-queue version (useful when you want to deliver requests downstream at the leak rate):

```python
from collections import deque

class LeakyBucketQueue:
    def __init__(self, capacity: int, leak_interval: float):
        self.capacity = capacity
        self.interval = leak_interval
        self.q: deque[float] = deque()  # arrival timestamps
        self.next_leak: float | None = None

    def allow(self, ts: float) -> bool:
        if self.next_leak is None:
            self.next_leak = ts + self.interval
        while self.q and self.next_leak <= ts:
            self.q.popleft()
            self.next_leak += self.interval
        if len(self.q) < self.capacity:
            self.q.append(ts)
            return True
        return False
```

## Complexity

Counter version:

- `allow`: **O(1)**.
- Space: **O(1)**.

Queue version:

- `allow`: **amortized O(1)** (each item enqueued/dequeued once).
- Space: **O(capacity)**.

## Edge cases

- **First call.** `last_time` is `None` → no leak yet. Initialize on first call.
- **Multiple events at the same timestamp.** No leak between them; bucket fills until full.
- **Long gaps.** Bucket fully drains; `water` clamps at 0.
- **Fractional water.** Use float for accurate accounting. Comparing `water < capacity` is fine even with floats here (off-by-tiny-epsilon doesn't materially harm rate limiting).
- **Confusion with token bucket.** Be ready to explain the difference. Many interviewers will follow up: "what if I want to allow bursts?" Answer: switch to token bucket.

## Linked concepts

- [Rate Limiting & Counting category overview](../by-category/rate-limiting-counting.md)
- [Rate limiting (HLD)](../../hld/fundamentals/rate-limiting.md)
- [Rate Limiter](rate-limiter.md) — sliding-window log alternative

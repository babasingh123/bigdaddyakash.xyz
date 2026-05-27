# Design Hit Counter

!!! note "The disguise"
    Interviewer says: **"Design a hit counter that returns hits in the last 5 minutes."**
    They mean: **"Implement a Queue of timestamps with sliding-window eviction."**

## Problem

Build `HitCounter` with:

- `hit(timestamp)` — record one hit at this second.
- `getHits(timestamp)` — return the number of hits in the past 300 seconds (inclusive of `timestamp - 299` through `timestamp`).

`timestamp` is in seconds, monotonically non-decreasing across calls.

## What it really tests

- **Deque-based sliding window over time.**
- **Lazy eviction** — clean up old timestamps only when needed, not on a timer.
- **The bucket optimization** for high-throughput variants.

## Approach

### Naive: per-timestamp deque

Use a `deque[int]` of timestamps. `hit(ts)` appends; `getHits(ts)` evicts everything ≤ `ts - 300` from the head and returns the size.

```
deque: [t1, t2, ..., tk]   (sorted because timestamps are non-decreasing)
hit(ts): deque.append(ts)
getHits(ts):
    while deque and deque[0] <= ts - 300: deque.popleft()
    return len(deque)
```

This is correct and simple. Each timestamp is appended once and removed once → amortized O(1) per call. Space is O(hits in window).

### Bucketed: O(1) worst-case per call

If hits-per-second can be huge (millions), the deque grows to `300 × hits_per_second`. We don't need per-hit resolution — we just need per-second counts.

Use a **circular array of size 300**, indexed by `ts % 300`. Each cell stores `(timestamp, count)`:

- `hit(ts)`: cell `i = ts % 300`. If `cell.ts == ts`, increment count. Otherwise, reset to `(ts, 1)`.
- `getHits(ts)`: sum `cell.count` for cells whose `cell.ts > ts - 300`.

This is **O(1) space** (300 buckets) and **O(300) per call** = O(1).

**Why two solutions?** The interview signal: start with the deque (clear, correct), then propose the buckets as an optimization for high throughput. Showing both shows you understand the tradeoff.

## Code sketch

The deque version:

```python
from collections import deque

class HitCounter:
    def __init__(self):
        self.q: deque[int] = deque()

    def hit(self, timestamp: int) -> None:
        self.q.append(timestamp)

    def getHits(self, timestamp: int) -> int:
        cutoff = timestamp - 300
        while self.q and self.q[0] <= cutoff:
            self.q.popleft()
        return len(self.q)
```

The bucketed version:

```python
class HitCounterBuckets:
    def __init__(self):
        self.times = [0] * 300
        self.counts = [0] * 300

    def hit(self, timestamp: int) -> None:
        i = timestamp % 300
        if self.times[i] != timestamp:
            self.times[i] = timestamp
            self.counts[i] = 0
        self.counts[i] += 1

    def getHits(self, timestamp: int) -> int:
        total = 0
        for i in range(300):
            if timestamp - self.times[i] < 300:
                total += self.counts[i]
        return total
```

## Complexity

Deque version:

- `hit`: **O(1)**.
- `getHits`: **amortized O(1)** (each hit evicted at most once).
- Space: **O(hits in window)**.

Bucketed version:

- `hit`: **O(1)**.
- `getHits`: **O(300) = O(1)**.
- Space: **O(300) = O(1)**.

## Edge cases

- **Out-of-order timestamps.** The problem usually guarantees non-decreasing. If not, you need a more general structure (TreeMap of `ts → count`).
- **`getHits` called before any `hit`.** Returns 0.
- **A burst of hits at the same timestamp.** Both versions handle this — deque appends each one; bucket increments the count cell.
- **Window boundary precision.** Convention: hits at `ts - 299` through `ts` count (300 distinct seconds inclusive). Match the problem's wording — `<=` vs `<` in the cutoff is a classic off-by-one trap.

## Linked concepts

- [Rate Limiting & Counting category overview](../by-category/rate-limiting-counting.md)
- [Rate Limiter](rate-limiter.md) — same pattern, per user
- [Moving Average from Data Stream](moving-average.md) — fixed-count window instead of fixed-time

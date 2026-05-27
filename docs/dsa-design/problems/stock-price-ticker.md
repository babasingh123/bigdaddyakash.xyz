# Design Stock Price Ticker

!!! note "The disguise"
    Interviewer says: **"Design a system to track max/min stock price."**
    They mean: **"Implement a TreeMap by price (or a Monotonic Deque for windowed max/min)."**

## Problem

Build `StockPrice` (LeetCode 2034 style):

- `update(timestamp, price)` — corrections to past prices are allowed; the latest call per timestamp wins.
- `current() -> int` — the most recent price (by largest timestamp seen).
- `maximum() -> int` — the highest price across all valid timestamps.
- `minimum() -> int` — the lowest price across all valid timestamps.

## What it really tests

- **Two TreeMaps (or SortedDicts).** One by timestamp (for `current`), one by price (for max/min).
- **Correction handling.** When a timestamp's price is updated, the old price must be evicted from the price index.
- **Knowing when to use Monotonic Deque** instead, for windowed-max problems.

## Approach

The two queries that need to be fast are:

1. **`current()`** — value at the largest timestamp.
2. **`maximum()` / `minimum()`** — extremes over all valid timestamps.

**Single TreeMap by timestamp** handles `current` in O(log n) (last entry). It does not handle max/min — those would need an O(n) scan.

**Single TreeMap by price?** Handles max/min in O(log n) but loses the "which timestamp is latest" info.

So we need **two indexes**:

- `ts_to_price: dict[int, int]` — for correction lookups and `current`.
- `latest_ts: int` — the largest timestamp seen.
- `price_index: SortedDict[int, int]` (price → count) — for max/min.

**Update flow.**

```
update(ts, price):
    if ts in ts_to_price:
        old = ts_to_price[ts]
        decrement price_index[old]; if zero, remove the entry
    ts_to_price[ts] = price
    increment price_index[price]
    latest_ts = max(latest_ts, ts)
```

**Queries** are now O(log n):

- `current()`: `ts_to_price[latest_ts]`.
- `maximum()`: largest key in `price_index`.
- `minimum()`: smallest key in `price_index`.

**Why a multi-count map and not a heap?** Heaps can't decrement counts efficiently. With many corrections, a heap accumulates stale entries; you'd need lazy deletion *and* a side counter. A SortedDict with counts is cleaner.

**The Monotonic Deque variant.** If the problem instead said "max/min over the last *k* ticks," you'd use a [Monotonic Deque](sliding-window-max.md) — keys deque holds candidates in monotonically decreasing order; the front is the current max. That's an O(1)-per-element algorithm.

## Code sketch

```python
from sortedcontainers import SortedDict

class StockPrice:
    def __init__(self):
        self.ts_to_price: dict[int, int] = {}
        self.price_counts: SortedDict[int, int] = SortedDict()
        self.latest_ts: int = -1

    def update(self, timestamp: int, price: int) -> None:
        if timestamp in self.ts_to_price:
            old = self.ts_to_price[timestamp]
            self.price_counts[old] -= 1
            if self.price_counts[old] == 0:
                del self.price_counts[old]
        self.ts_to_price[timestamp] = price
        self.price_counts[price] = self.price_counts.get(price, 0) + 1
        if timestamp > self.latest_ts:
            self.latest_ts = timestamp

    def current(self) -> int:
        return self.ts_to_price[self.latest_ts]

    def maximum(self) -> int:
        return self.price_counts.peekitem(-1)[0]

    def minimum(self) -> int:
        return self.price_counts.peekitem(0)[0]
```

For the **windowed-max** variant (last *k* prices, in arrival order), use a monotonic deque:

```python
from collections import deque

class WindowedMax:
    def __init__(self, k: int):
        self.k = k
        self.window: deque[int] = deque()           # actual values
        self.monoq: deque[int] = deque()            # indices, prices monotonically decreasing

    def push(self, price: int) -> None:
        self.window.append(price)
        while self.monoq and self.window[self.monoq[-1]] < price:
            self.monoq.pop()
        self.monoq.append(len(self.window) - 1)
        if self.monoq[0] <= len(self.window) - 1 - self.k:
            self.monoq.popleft()

    def max(self) -> int:
        return self.window[self.monoq[0]]
```

## Complexity

For the timestamp-corrections variant:

- `update`: **O(log n)** — SortedDict insert/delete.
- `current` / `maximum` / `minimum`: **O(log n)**.
- Space: **O(distinct timestamps)**.

For the windowed-max variant (Monotonic Deque):

- `push`: **amortized O(1)**.
- `max`: **O(1)**.
- Space: **O(k)**.

## Edge cases

- **No updates yet.** `current`, `max`, `min` are undefined; raise or return a sentinel.
- **Correction reduces unique prices.** Don't forget to delete the price-count entry when the count drops to 0, or `max/min` will return stale values.
- **Corrections at the latest timestamp.** Update `ts_to_price` in place; `latest_ts` doesn't change.
- **Many updates at the same timestamp.** Each overwrites the previous price; price-index counts must be maintained accurately.

## Linked concepts

- [Scheduling & Priority category overview](../by-category/scheduling-priority.md)
- [Sliding Window Maximum](sliding-window-max.md) — the deque-based windowed version
- [Median Finder](median-finder.md) — different ordered-stream pattern (two heaps)

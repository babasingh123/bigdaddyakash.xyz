# Design Time-Based Key-Value Store

!!! note "The disguise"
    Interviewer says: **"Design a key-value store that supports time-travel queries."**
    They mean: **"Implement a HashMap of TreeMaps and binary-search the timestamps."**

## Problem

Build `TimeMap` with:

- `set(key, value, timestamp)` — store `value` at this timestamp.
- `get(key, timestamp)` — return the value associated with the **largest timestamp ≤ the given one**. If none, return `""`.

Timestamps in `set` calls for a single key are **strictly increasing** (this is the standard variant). Multiple writes to the same key form a version history.

## What it really tests

- **Nested data structures.** A single `dict` doesn't cut it — each key has a *history*, not a single value.
- **Binary search on time.** Given a sorted list of `(timestamp, value)`, find the largest timestamp ≤ T. That's a floor query, classic binary search.
- **TreeMap.floorEntry / `bisect` knowledge.** Knowing which library call (or how to write the loop) is the test.

## Approach

What do we need?

- `set(key, val, ts)` — append a version. Since timestamps for the same key are strictly increasing, append-only is fine.
- `get(key, ts)` — find the largest stored timestamp ≤ `ts`.

For each key, we need a sorted collection. There are two reasonable choices:

- **TreeMap (sorted dict).** Java: `TreeMap.floorEntry(ts)`. Python: `sortedcontainers.SortedDict`. Both give O(log n) floor lookup.
- **Plain list, since inserts are monotone.** If timestamps come in increasing order, you can just `append` to a list and do `bisect_right` on the timestamps. Same O(log n) lookup, simpler code, no library.

Top-level structure: `dict[key, list[(ts, val)]]` or `dict[key, SortedDict]`. The HashMap dispatches in O(1) to the right history; the binary search handles the time dimension.

**Why binary search and not linear?** Histories can grow huge (logging, audit trails). Linear scan is O(n) per `get`, which is fine for toy cases but a fail for the interview signal. The interviewer wants O(log n) and the *name* "binary search" or "floor query."

**Why `bisect_right - 1`?** We want the largest index *i* such that `ts[i] <= target`. `bisect_right(ts, target)` returns the first index strictly greater than `target`. Subtract one. If the result is `-1`, no version exists at or before `target`.

## Code sketch

```python
from bisect import bisect_right
from collections import defaultdict

class TimeMap:
    def __init__(self):
        self.store: dict[str, list[tuple[int, str]]] = defaultdict(list)

    def set(self, key: str, value: str, timestamp: int) -> None:
        self.store[key].append((timestamp, value))

    def get(self, key: str, timestamp: int) -> str:
        if key not in self.store:
            return ""
        history = self.store[key]
        i = bisect_right(history, (timestamp, chr(0x10FFFF))) - 1
        if i < 0:
            return ""
        return history[i][1]
```

The `chr(0x10FFFF)` is a sentinel — it makes `(timestamp, sentinel)` compare *greater than* any real `(timestamp, value)` at that timestamp, so `bisect_right` returns the index *just past* all entries at or before `timestamp`. Subtract 1 to get the floor.

A cleaner alternative if you keep a parallel `keys` list:

```python
def get(self, key: str, timestamp: int) -> str:
    history = self.store.get(key, [])
    if not history:
        return ""
    timestamps = [t for t, _ in history]  # ideally store this separately
    i = bisect_right(timestamps, timestamp) - 1
    return history[i][1] if i >= 0 else ""
```

In production code, keep `(timestamps, values)` as two parallel lists to avoid the comparison-key hack.

## Complexity

- `set`: **O(1)** amortized (append to list). If using a TreeMap: O(log n).
- `get`: **O(log n)** where *n* is the number of versions for that key.
- Space: **O(total versions across all keys)**.

## Edge cases

- **Key never set.** Return `""`.
- **Timestamp before all stored versions.** `bisect_right - 1 = -1` → return `""`.
- **Timestamp exactly matches a stored version.** Use `bisect_right` (not `bisect_left`) and `- 1` so we pick that exact version.
- **Non-monotonic timestamps.** If the problem variant allows them, you must use a TreeMap or sort on read; the simple-append-only trick breaks.

## Linked concepts

- [Cache & Memory category overview](../by-category/cache-memory.md)
- [Snapshot Array](snapshot-array.md) — same technique applied per index
- [Caching strategies (HLD)](../../hld/fundamentals/caching.md)

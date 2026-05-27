# Design Logger with Timestamp Ordering

!!! note "The disguise"
    Interviewer says: **"Design a logger that processes logs in timestamp order even though they arrive out of order"**
    They mean: **"Use a sorted map (TreeMap / `SortedList`) keyed by timestamp"**

## Problem

- `log(ts, msg)` — record a log entry at time `ts` (may arrive out of order).
- `getInOrder()` — return all logs in increasing timestamp order.
- `flushUntil(ts)` — emit (and remove) every log with timestamp ≤ `ts`, in order.

## What it really tests

- Picking a structure with **sorted insertion** *and* **sorted iteration**: TreeMap in Java, `SortedList` from `sortedcontainers` in Python, `std::map` in C++. All give O(log n) insert and O(log n) range queries.
- Recognizing that a min-heap can't iterate in sorted order without `O(n log n)` total work (pop n times).
- A plain HashMap + sort-on-read is O(n log n) per query — too slow.

## Approach

Wrap `sortedcontainers.SortedList` of `(ts, msg)` tuples. Tuples compare lexicographically, so identical timestamps still order deterministically by message.

For `getInOrder()` iterate in sorted order — O(n).
For `flushUntil(ts)` use `bisect_right` to find the cut, slice, then delete that prefix — O(log n + k) where k is the flushed count.

Why not a min-heap? Heap pop is O(log n) and only gives you the minimum. To iterate everything in order you must pop n times → O(n log n). Sorted containers iterate in O(n).

## Code sketch

```python
from sortedcontainers import SortedList

class TimestampLogger:
    def __init__(self):
        self.logs = SortedList()  # (timestamp, msg)

    def log(self, ts: int, msg: str) -> None:
        self.logs.add((ts, msg))

    def getInOrder(self) -> list[tuple[int, str]]:
        return list(self.logs)

    def flushUntil(self, ts: int) -> list[tuple[int, str]]:
        cut = self.logs.bisect_right((ts, chr(0x10FFFF)))
        out = list(self.logs[:cut])
        del self.logs[:cut]
        return out
```

## Complexity

- `log`: O(log n).
- `getInOrder`: O(n).
- `flushUntil`: O(log n + k).
- Space: O(n).

## Edge cases

- Duplicate timestamps with different messages: tuple comparison breaks ties by message.
- `flushUntil(ts)` before any log → empty list.
- A late-arriving log with `ts` already flushed: still inserted; depending on spec, drop it instead.
- Very high write rate: consider partitioning by time bucket.

## Linked concepts

- [String & Pattern problems](../by-category/string-pattern.md)
- TreeMap variants also drive [Time-Based KV Store](./time-based-kv-store.md) and [Snapshot Array](./snapshot-array.md).

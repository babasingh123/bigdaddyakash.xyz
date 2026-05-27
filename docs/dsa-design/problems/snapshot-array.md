# Design Snapshot Array

!!! note "The disguise"
    Interviewer says: **"Design an array that supports snapshots in addition to set / get-at-snapshot"**
    They mean: **"For each index, store a sorted list of `(snap_id, value)` — `get` is a binary search"**

## Problem

- `SnapshotArray(length)`
- `set(index, val)` — write to the current array.
- `snap()` — increment the snap counter; return the previous id.
- `get(index, snap_id)` — return the value at `index` at the time of `snap_id`.

## What it really tests

- Recognizing the **naïve trap**: deep-copying the array on every snap is O(n) per snap and O(n · snaps) memory.
- The real insight: only `set` mutates. Per-index version history is tiny — store only diffs.
- Binary search: `bisect_right` over the per-index version list to find the latest write at-or-before `snap_id`.

## Approach

Per-index `list[(snap_id, val)]`, sorted by `snap_id` (always appending → already sorted).

- `set(i, v)`: append `(current_snap, v)`. If the last entry has the same snap, overwrite it so that each `(index, snap_id)` has at most one entry.
- `snap()`: increment counter, return the previous value.
- `get(i, sid)`: `bisect_right(history, (sid, +∞))` then back up by one. Default to 0 if no writes exist at or before `sid`.

Why per-index and not a global timeline? Reads are by `(index, snap_id)`; a per-index list is the most cache-friendly layout. A global timeline forces scanning all writes since the last snap.

## Code sketch

```python
from bisect import bisect_right

class SnapshotArray:
    def __init__(self, length: int):
        self.snap_id = 0
        self.history = [[(-1, 0)] for _ in range(length)]   # sentinel

    def set(self, index: int, val: int) -> None:
        last = self.history[index][-1]
        if last[0] == self.snap_id:
            self.history[index][-1] = (self.snap_id, val)
        else:
            self.history[index].append((self.snap_id, val))

    def snap(self) -> int:
        self.snap_id += 1
        return self.snap_id - 1

    def get(self, index: int, snap_id: int) -> int:
        h = self.history[index]
        i = bisect_right(h, (snap_id, float("inf"))) - 1
        return h[i][1]
```

## Complexity

- `set`: O(1) amortized.
- `snap`: O(1).
- `get`: O(log k) where k is the number of writes to that index.
- Space: O(total writes) — *not* O(length · snaps).

## Edge cases

- `get` before any `set` → returns 0 thanks to the sentinel.
- Multiple `set` calls at the same `snap_id` → only the last value matters.
- No `snap()` ever called → `snap_id == 0`; all reads use that.

## Linked concepts

- [String & Pattern problems](../by-category/string-pattern.md)
- Same versioning pattern as [Time-Based Key-Value Store](./time-based-kv-store.md).

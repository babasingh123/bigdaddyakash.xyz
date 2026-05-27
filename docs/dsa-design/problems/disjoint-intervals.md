# Design Data Stream as Disjoint Intervals

!!! note "The disguise"
    Interviewer says: **"As numbers stream in, return the disjoint intervals that summarize them so far"**
    They mean: **"Maintain a `SortedDict` of `start -> end` and merge with the neighbors on each insert"**

## Problem

- `addNum(val)` — add an integer to the stream.
- `getIntervals()` — return the current set of disjoint intervals in sorted order.

## What it really tests

- Picking a structure with **sorted iteration** *and* **O(log n) neighbor lookup** — TreeMap / `SortedDict`. A heap can't do `floorKey`, a HashMap can't iterate in order.
- Handling the "fan-in" case: a new value can simultaneously bridge the interval to its left and the one to its right — e.g., insert `3` into `{1..2, 4..5}` ⇒ `{1..5}`.

## Approach

Maintain `intervals: SortedDict[start, end]`. On `addNum(v)`:

1. Find `lo` = the largest key ≤ `v` (`floorKey`).
2. Find `hi` = the smallest key ≥ `v` (`ceilingKey`).
3. Cases:
    - `lo` exists and `intervals[lo] >= v` → `v` is already covered, do nothing.
    - `lo` is left-adjacent (`intervals[lo] == v - 1`) **and** `hi` is right-adjacent (`hi == v + 1`) → merge both: replace `lo`'s end with `intervals[hi]`, delete `hi`.
    - Only left-adjacent → extend `lo`'s end to `v`.
    - Only right-adjacent → delete `hi`, insert `[v, intervals[hi]]`.
    - Otherwise → insert `[v, v]`.

This keeps both `addNum` and `getIntervals` operating on the *compact* representation rather than the raw stream.

## Code sketch

```python
from sortedcontainers import SortedDict

class SummaryRanges:
    def __init__(self):
        self.tree = SortedDict()   # start -> end

    def addNum(self, v: int) -> None:
        keys = self.tree.keys()
        lo_idx = self.tree.bisect_right(v) - 1
        hi_idx = self.tree.bisect_left(v)

        lo = keys[lo_idx] if lo_idx >= 0 else None
        hi = keys[hi_idx] if hi_idx < len(keys) else None

        if lo is not None and self.tree[lo] >= v:
            return                                   # already covered

        left_adj  = lo is not None and self.tree[lo] == v - 1
        right_adj = hi is not None and hi == v + 1

        if left_adj and right_adj:
            self.tree[lo] = self.tree[hi]
            del self.tree[hi]
        elif left_adj:
            self.tree[lo] = v
        elif right_adj:
            end = self.tree.pop(hi)
            self.tree[v] = end
        else:
            self.tree[v] = v

    def getIntervals(self) -> list[list[int]]:
        return [[s, e] for s, e in self.tree.items()]
```

## Complexity

- `addNum`: O(log n).
- `getIntervals`: O(k) where k is the number of disjoint intervals (worst case n/2).
- Space: O(k).

## Edge cases

- Duplicate insertion → already-covered branch handles it.
- A single integer → `[[v, v]]`.
- Long contiguous streams collapse to a single interval, keeping memory low.

## Linked concepts

- [Data Stream problems](../by-category/data-stream.md)
- The same TreeMap-of-intervals trick powers [Meeting Scheduler](./meeting-scheduler.md).

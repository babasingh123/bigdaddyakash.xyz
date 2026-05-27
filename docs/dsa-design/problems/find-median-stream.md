# Design Find Median from Data Stream

!!! note "The disguise"
    Interviewer says: **"Find the median of an unbounded stream of integers"**
    They mean: **"Two heaps — a max-heap for the lower half and a min-heap for the upper half — kept balanced in size"**

## Problem

- `addNum(num)` — O(log n).
- `findMedian()` — O(1). Returns the median (the middle value if odd-sized, average of the two middles if even-sized).

## What it really tests

- Why naïve fails:
    - A sorted list: insert is O(n) (linear shift); too slow for streams.
    - A single heap: median needs the **middle**, but a heap exposes only one extreme.
- The two-heap insight: split the data at the median.
    - `lo` = max-heap of the smaller half.
    - `hi` = min-heap of the larger half.
- After every insert, the heaps differ in size by at most 1, so the median is at one of the tops.

## Approach

Invariants:

1. Every value in `lo` ≤ every value in `hi`.
2. `len(lo) - len(hi) ∈ {0, 1}` (we let the lower side hold the extra element when the count is odd).

`addNum(x)`:

- Push to `lo` first (as a max-heap by negating values).
- Move `lo`'s top to `hi` to keep invariant 1.
- If `hi` is now larger than `lo`, move one back.

`findMedian`:

- Odd total → top of `lo` (negate it).
- Even total → average the two tops.

## Code sketch

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.lo = []   # max-heap (store negatives)
        self.hi = []   # min-heap

    def addNum(self, num: int) -> None:
        heapq.heappush(self.lo, -num)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))

    def findMedian(self) -> float:
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2
```

## Complexity

- `addNum`: O(log n) — three heap operations.
- `findMedian`: O(1).
- Space: O(n).

## Edge cases

- Stream of length 0 → `findMedian` is undefined; problems usually guarantee at least one `addNum` first.
- All identical numbers → heaps stay balanced.
- Streaming floats: works unchanged.
- Removing values (a follow-up question): switch to two `SortedList`s, or two heaps + lazy deletion.

## Linked concepts

- [Data Stream problems](../by-category/data-stream.md)
- The two-heap pattern also handles "sliding window median" with lazy deletion.

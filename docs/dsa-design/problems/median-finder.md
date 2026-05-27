# Design Median Finder

!!! note "The disguise"
    Interviewer says: **"Design a data structure to find the median from a stream."**
    They mean: **"Implement Two Heaps — a Max Heap for the lower half, a Min Heap for the upper half."**

## Problem

Build `MedianFinder` with:

- `addNum(num)` — incorporate a number from the stream.
- `findMedian() -> float` — return the median of all numbers seen so far.

The classic interview problem (LeetCode 295). This is also exactly the same as [Find Median from Data Stream](find-median-stream.md) — it gets framed both ways.

## What it really tests

- **Two Heaps** — *the* signature stream pattern.
- **Heap balancing** — keep `|lower| - |upper| ≤ 1`.
- **O(log n) insert, O(1) query.**

## Approach

We need:

- **O(1) query** for the median.
- Fast inserts.
- No requirement to access non-median elements.

**Single sorted array?** O(n) insert (shift), O(1) query. Insert is too slow.

**Single Heap?** Gives you min or max, not middle.

**Idea: split the data at the median.**

- **`lower` (Max Heap)** — holds the smaller half of seen numbers. The top is the largest of the smaller half.
- **`upper` (Min Heap)** — holds the larger half. The top is the smallest of the larger half.

If we keep `|lower| - |upper| ∈ {0, 1}` after every insert:

- If sizes are equal, median = `(lower.top + upper.top) / 2`.
- If `lower` has one more, median = `lower.top`.

That's O(1) per query.

**Insert balancing.** For each new number:

1. Push to one heap (e.g., always to `lower` first, after comparing with its top).
2. If `lower.top > upper.top`, the boundary is misaligned — pop from `lower`, push to `upper`.
3. If `|lower| - |upper| > 1`, move one from `lower` to `upper`.
4. If `|upper| > |lower|`, move one from `upper` to `lower`.

The result: after every insert, both heaps are non-empty (or `upper` is empty when we've seen exactly one number), and the invariants hold.

**Why does this work?** The median is the value(s) at position(s) ⌊n/2⌋ (and ⌊(n-1)/2⌋ for even n). Splitting the data exactly at the median means the two heap tops are precisely those values.

**Python heapq trick.** It only does min-heap. For the max-heap, push negated values: `heappush(lower, -num)`; the "max" is `-lower[0]`.

## Code sketch

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.lower: list[int] = []   # max-heap (negated)
        self.upper: list[int] = []   # min-heap

    def addNum(self, num: int) -> None:
        # decide which side to push to
        if not self.lower or num <= -self.lower[0]:
            heapq.heappush(self.lower, -num)
        else:
            heapq.heappush(self.upper, num)
        # rebalance
        if len(self.lower) > len(self.upper) + 1:
            heapq.heappush(self.upper, -heapq.heappop(self.lower))
        elif len(self.upper) > len(self.lower):
            heapq.heappush(self.lower, -heapq.heappop(self.upper))

    def findMedian(self) -> float:
        if len(self.lower) > len(self.upper):
            return float(-self.lower[0])
        return (-self.lower[0] + self.upper[0]) / 2
```

## Complexity

- `addNum`: **O(log n)** — up to two heap push/pop pairs.
- `findMedian`: **O(1)**.
- Space: **O(n)** total across both heaps.

## Edge cases

- **First number.** `lower` gets it; `findMedian` returns it.
- **All identical numbers.** Heaps work fine; tops are equal, median is correct.
- **Very large stream.** Two heaps still O(log n) per insert; no degradation.
- **`findMedian` before any `addNum`.** Undefined — raise or return NaN; clarify with the interviewer.
- **Removing elements** (follow-up question). Two heaps don't naturally support removal. Use lazy deletion with a "to-remove" counter, or switch to a SortedList / SortedDict.

## Linked concepts

- [Ranking & Leaderboard category overview](../by-category/ranking-leaderboard.md)
- [Data Stream category overview](../by-category/data-stream.md)
- [Find Median from Data Stream](find-median-stream.md) — alias for this problem in the stream category
- [Kth Largest in Stream](kth-largest-stream.md) — sibling streaming problem

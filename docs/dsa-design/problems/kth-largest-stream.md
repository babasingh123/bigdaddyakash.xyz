# Design Kth Largest in a Stream

!!! note "The disguise"
    Interviewer says: **"After each `add(v)`, return the kth largest value seen so far"**
    They mean: **"Keep a min-heap of size exactly k — its root *is* the kth largest"**

## Problem

- `KthLargest(k, nums)`
- `add(v) -> int` — insert `v`, return the kth largest element among all values seen so far.

## What it really tests

- The trick: a min-heap of size k contains exactly the top-k largest values, and the smallest of them (the root) is the kth largest overall.
- Why not a max-heap? You'd have to pop k − 1 elements just to peek at the kth.
- A sorted list also works but insert is O(n) — too slow.
- Maintenance: after pushing `v`, if the size exceeds k, pop the smallest. That preserves the "top-k by size, kth at root" invariant.

## Approach

1. Initialize a min-heap from `nums`.
2. While `len(heap) > k`, pop the smallest.
3. On `add(v)`:
    - If `len(heap) < k`: push.
    - Else if `v > heap[0]`: `heappushpop` (atomic push + pop of the smallest).
    - Else: ignore (`v` can't be in the top-k).
4. Return `heap[0]`.

Why is `heap[0]` the kth largest? The heap stores the k biggest values seen so far; the smallest among them is the kth largest globally.

## Code sketch

```python
import heapq

class KthLargest:
    def __init__(self, k: int, nums: list[int]):
        self.k = k
        self.heap = nums[:]
        heapq.heapify(self.heap)
        while len(self.heap) > k:
            heapq.heappop(self.heap)

    def add(self, val: int) -> int:
        if len(self.heap) < self.k:
            heapq.heappush(self.heap, val)
        elif val > self.heap[0]:
            heapq.heappushpop(self.heap, val)
        return self.heap[0]
```

## Complexity

- Constructor: O(n log k) — heapify and trim.
- `add`: O(log k).
- Space: O(k).

## Edge cases

- Initial `nums` has fewer than k elements; the first few `add` calls fill the heap.
- Duplicates: counted with multiplicity — `[5, 5, 5, 5]` with k = 2 returns 5 each time.
- Streaming negative numbers: works unchanged.
- `add` before reaching k elements: the problem says it won't happen, but in real code guard with `heap[0] if len(heap) >= k else None`.

## Linked concepts

- [Data Stream problems](../by-category/data-stream.md)
- The same heap pattern drives [Top K Frequent](./top-k-frequent.md).

# Design Top K Frequent Elements

!!! note "The disguise"
    Interviewer says: **"Design a system to track the top K frequent items."**
    They mean: **"Use a HashMap for counts plus a Min Heap of size K — or Bucket Sort for O(n)."**

## Problem

Given a stream (or array) of items and an integer *K*, return the *K* most frequent items.

For the streaming "design" variant: support `add(item)` and `top_k()` queries.

## What it really tests

- **HashMap for frequency counting.**
- **Min Heap of size K** — the canonical streaming top-K pattern.
- **Bucket Sort** as an O(n) batch alternative when frequencies are bounded.
- **Resisting the "sort everything" reflex.**

## Approach

The "obvious" solution — count, sort by count, take top K — is O(n log n). We can do better.

### Min Heap of size K

Keep a Min Heap of size *exactly K*, keyed by `(count, item)`:

- New item arrives → update its count in the HashMap.
- If heap size < K, push.
- Else if new count > heap top, pop and push.

The heap's top is always the *smallest* of the K largest counts. New items only displace the top if they beat it.

For the batch (array) version:

```
freq = Counter(items)
heap = []
for item, count in freq.items():
    heapq.heappush(heap, (count, item))
    if len(heap) > K:
        heapq.heappop(heap)
return [item for _, item in heap]
```

This is **O(n log K)** — far better than O(n log n) when K is small.

**Streaming complication.** When an item already in the heap has its count incremented, the heap entry becomes stale (the count in the heap is lower than the true count). You have three options:

1. Rebuild the heap on every `top_k` query → O(distinct items × log K).
2. Lazy invalidation: store `(version, count, item)`, skip stale entries on pop.
3. **Use a `TreeMap` instead of a heap**, with O(log n) increments and `top K` by iterating from the largest. This is essentially the [Leaderboard](leaderboard.md) design.

For the LeetCode batch problem, the heap is fine because frequencies are computed once.

### Bucket Sort (batch only)

If you know item counts are bounded (which they are: ≤ n), bucket-sort by count:

- `buckets[i]` = list of items with frequency `i`.
- Iterate from the highest bucket down, collecting items until you have K.

**O(n)** — linear time, no heap, no sort. The textbook answer for batch top-K when an interviewer asks "can you do better than O(n log K)?"

## Code sketch

Heap-of-K (batch):

```python
import heapq
from collections import Counter

def top_k_frequent(items: list[int], k: int) -> list[int]:
    freq = Counter(items)
    heap: list[tuple[int, int]] = []
    for item, c in freq.items():
        heapq.heappush(heap, (c, item))
        if len(heap) > k:
            heapq.heappop(heap)
    return [item for _, item in heap]
```

Bucket sort (batch):

```python
def top_k_frequent_bucket(items: list[int], k: int) -> list[int]:
    freq = Counter(items)
    buckets: list[list[int]] = [[] for _ in range(len(items) + 1)]
    for item, c in freq.items():
        buckets[c].append(item)
    result: list[int] = []
    for c in range(len(buckets) - 1, 0, -1):
        for item in buckets[c]:
            result.append(item)
            if len(result) == k:
                return result
    return result
```

Streaming with TreeMap:

```python
from sortedcontainers import SortedDict

class TopKStreaming:
    def __init__(self, k: int):
        self.k = k
        self.freq: dict[int, int] = {}
        self.by_count: SortedDict[int, set[int]] = SortedDict()

    def add(self, item: int) -> None:
        old = self.freq.get(item, 0)
        if old > 0:
            self.by_count[old].discard(item)
            if not self.by_count[old]:
                del self.by_count[old]
        new = old + 1
        self.freq[item] = new
        if new not in self.by_count:
            self.by_count[new] = set()
        self.by_count[new].add(item)

    def top_k(self) -> list[int]:
        out: list[int] = []
        for c in reversed(self.by_count):
            for it in self.by_count[c]:
                out.append(it)
                if len(out) == self.k:
                    return out
        return out
```

## Complexity

Heap-of-K (batch):

- Build counts: **O(n)**.
- Heap operations: **O(d log K)** where *d* = distinct items.
- Total: **O(n + d log K)** ≈ **O(n log K)**.

Bucket sort (batch):

- **O(n)** total.

Streaming with TreeMap:

- `add`: **O(log d)**.
- `top_k`: **O(log d + K)**.

## Edge cases

- **K ≥ number of distinct items.** Return all items.
- **All items have the same frequency.** Heap-of-K returns an arbitrary K of them; bucket sort returns the first K encountered.
- **Empty input.** Return `[]`.
- **Ties in count.** Problem usually allows any valid answer. If it specifies "lexicographically smallest among ties," add that as the secondary heap key.

## Linked concepts

- [Ranking & Leaderboard category overview](../by-category/ranking-leaderboard.md)
- [Leaderboard](leaderboard.md) — adapted for player scores
- [Kth Largest in Stream](kth-largest-stream.md) — same heap-of-K trick

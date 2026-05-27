# Design Range Sum Query (Mutable)

!!! note "The disguise"
    Interviewer says: **"Design a structure for range sum queries with updates."**
    They mean: **"Implement a Segment Tree or Fenwick (Binary Indexed) Tree."**

## Problem

Build `NumArray(nums)`:

- `update(index, val)` — set `nums[index] = val`.
- `sumRange(left, right) -> int` — return `nums[left] + ... + nums[right]`.

Both must be **O(log n)**.

LeetCode 307. The "design" framing emphasizes that calls come in any order, indefinitely.

## What it really tests

- **Segment Tree** or **Fenwick Tree (BIT)** — two ways to support range queries with point updates in O(log n).
- **Knowing the prefix-sum array is *not enough*** when updates are allowed.
- **Cleanly handling 1-indexed BIT arithmetic.**

## Approach

Why is this hard? Either operation alone is easy:

- **Only `sumRange`?** Precompute prefix sums. O(1) per query. But updates are O(n).
- **Only `update`?** Just write the array. O(1). But queries are O(n).

We need *both* in O(log n). Two structures give that.

### Option 1: Fenwick Tree (Binary Indexed Tree)

A clever array `bit[1..n]` where each cell holds the sum of a specific power-of-two range. Operations:

- `update(i, delta)` — add `delta` to `nums[i]`. Walk *up*: `i += i & -i`, applying delta to each visited index.
- `prefix_sum(i)` — sum of `nums[1..i]`. Walk *down*: `i -= i & -i`, summing.
- `sumRange(l, r) = prefix_sum(r) - prefix_sum(l - 1)`.

The bit-twiddling `i & -i` isolates the lowest set bit, which encodes how big a range each cell covers. It's magical the first time you see it; it works because every prefix decomposes uniquely into power-of-two chunks.

For *point updates + range sums*, BIT is the smaller-code solution: ~15 lines.

### Option 2: Segment Tree

A binary tree where each node represents a contiguous range. Leaves hold individual array values; internal nodes hold the sum of their children.

- `update(i, val)` — find the leaf, update, propagate sum changes up the tree.
- `sumRange(l, r)` — recursively combine partial sums from nodes whose ranges are fully contained in `[l, r]`.

Segment trees are more flexible (work for min, max, gcd, lazy propagation for range updates) but require ~50-80 lines.

### Which to pick

- **Only point updates + range sums:** BIT. Shorter, faster constant factor.
- **Range updates, non-sum aggregates, lazy propagation:** Segment Tree.
- **Read-only:** Plain prefix sums (no need for either).

In an interview, **state both options**, then implement the BIT if "sum" is the only aggregate. If the interviewer follows up with "what if I want range min?" → switch to segment tree.

## Code sketch

Fenwick Tree (BIT):

```python
class NumArray:
    def __init__(self, nums: list[int]):
        self.n = len(nums)
        self.nums = [0] * self.n
        self.bit = [0] * (self.n + 1)
        for i, v in enumerate(nums):
            self.update(i, v)

    def update(self, index: int, val: int) -> None:
        delta = val - self.nums[index]
        self.nums[index] = val
        i = index + 1
        while i <= self.n:
            self.bit[i] += delta
            i += i & -i

    def _prefix(self, i: int) -> int:
        s = 0
        while i > 0:
            s += self.bit[i]
            i -= i & -i
        return s

    def sumRange(self, left: int, right: int) -> int:
        return self._prefix(right + 1) - self._prefix(left)
```

Segment Tree (array-backed, iterative):

```python
class NumArraySeg:
    def __init__(self, nums: list[int]):
        self.n = len(nums)
        self.tree = [0] * (2 * self.n)
        for i, v in enumerate(nums):
            self.tree[self.n + i] = v
        for i in range(self.n - 1, 0, -1):
            self.tree[i] = self.tree[2 * i] + self.tree[2 * i + 1]

    def update(self, index: int, val: int) -> None:
        i = index + self.n
        self.tree[i] = val
        i //= 2
        while i:
            self.tree[i] = self.tree[2 * i] + self.tree[2 * i + 1]
            i //= 2

    def sumRange(self, left: int, right: int) -> int:
        l, r = left + self.n, right + self.n
        s = 0
        while l <= r:
            if l % 2 == 1:
                s += self.tree[l]; l += 1
            if r % 2 == 0:
                s += self.tree[r]; r -= 1
            l //= 2; r //= 2
        return s
```

## Complexity

- `update`: **O(log n)**.
- `sumRange`: **O(log n)**.
- Construction: **O(n)** (BIT initialized by point updates is O(n log n), but a smarter linear init exists).
- Space: **O(n)**.

## Edge cases

- **Empty array.** Allowed; no queries possible. Handle the n=0 case.
- **`update` with same value.** `delta = 0`, the walk is wasted but harmless.
- **`sumRange(i, i)`** — single element. Both implementations handle correctly.
- **Indexing conventions.** BIT is naturally 1-indexed (so `0 & -0 = 0` would never advance the walk). Add 1 when going from external index to internal.
- **Integer overflow** (Java/C++). Sum can exceed 32-bit; use `long`.

## Linked concepts

- [Ranking & Leaderboard category overview](../by-category/ranking-leaderboard.md)
- [Snapshot Array](snapshot-array.md) — different problem, similar "support arbitrary queries efficiently" theme
- [Sliding Window Maximum](sliding-window-max.md) — sibling structured-stream problem (no updates, fixed window)

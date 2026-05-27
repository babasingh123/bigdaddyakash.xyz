# Design Sliding Window Maximum

!!! note "The disguise"
    Interviewer says: **"Track the maximum in a sliding window."**
    They mean: **"Implement a Monotonic Deque (strictly decreasing) for O(n) total."**

## Problem

Given an array `nums` and a window size `k`, return an array where the *i*-th element is the maximum of `nums[i - k + 1 .. i]` for all `i >= k - 1`.

For the streaming "design" framing: support `add(num)` returning the current window's max.

## What it really tests

- **Monotonic Deque** — the most beautiful "free heap" trick in DSA.
- **Why a heap is too slow.** O(n log k) is the obvious answer; the deque gets O(n).
- **Index tracking** to drop elements that fall out of the window.

## Approach

Naive: for each window, scan k elements → O(n × k). Too slow.

Heap: maintain a max-heap of (value, index); pop stale entries lazily → **O(n log k)**. Acceptable but not best.

**Monotonic Deque: O(n) total.**

Keep a deque of *indices* such that the values at those indices are **strictly decreasing** from front to back. The front of the deque is always the index of the current window's max.

**Two invariants on every step:**

1. **Window invariant.** When the front index falls out of the window (`front <= i - k`), pop it from the front.
2. **Monotonicity invariant.** Before pushing index `i`, pop from the back while `nums[back] <= nums[i]`. These back values can never be max again — `i` is more recent and at least as large.

After processing index `i`, the front of the deque is the index of the max in `nums[i - k + 1 .. i]`.

**Why is this O(n)?** Each index is pushed once and popped at most once. Total work is 2n.

**Why does the monotonicity hold?** When `nums[i]` arrives, every prior index *j* with `nums[j] <= nums[i]` is dominated for the rest of *j*'s window — `i` is both later and at least as good. Popping them is safe. What remains is a deque of indices whose values are strictly decreasing.

**Why a deque and not a stack?** Because we pop from *both ends* — the back for monotonicity, the front for window expiry. A deque is exactly the right shape.

## Code sketch

```python
from collections import deque

def max_sliding_window(nums: list[int], k: int) -> list[int]:
    dq: deque[int] = deque()  # indices, nums[dq[0]] >= nums[dq[1]] >= ...
    out: list[int] = []
    for i, v in enumerate(nums):
        # 1) drop indices out of window
        if dq and dq[0] <= i - k:
            dq.popleft()
        # 2) maintain monotonic decrease
        while dq and nums[dq[-1]] <= v:
            dq.pop()
        dq.append(i)
        # 3) record current window max once we've seen k elements
        if i >= k - 1:
            out.append(nums[dq[0]])
    return out
```

Streaming class form:

```python
class SlidingMax:
    def __init__(self, k: int):
        self.k = k
        self.dq: deque[tuple[int, int]] = deque()  # (index, value)
        self.i = -1

    def add(self, val: int) -> int:
        self.i += 1
        if self.dq and self.dq[0][0] <= self.i - self.k:
            self.dq.popleft()
        while self.dq and self.dq[-1][1] <= val:
            self.dq.pop()
        self.dq.append((self.i, val))
        return self.dq[0][1]
```

## Complexity

- Per element: **amortized O(1)** — each index is pushed and popped at most once.
- Total: **O(n)**.
- Space: **O(k)** for the deque.

## Edge cases

- **`k = 1`.** The deque always holds one element; output is just `nums`.
- **`k >= n`.** A single window; output is `[max(nums)]`.
- **All equal elements.** Each new equal value pops the previous one off the back (using `<=`, not `<`). Behavior is still correct.
- **Empty array.** Return `[]`.
- **For sliding minimum,** swap the comparison: pop from the back while `nums[back] >= v`, keep monotonically increasing.

## Linked concepts

- [Ranking & Leaderboard category overview](../by-category/ranking-leaderboard.md)
- [Stock Price Ticker](stock-price-ticker.md) — uses the same deque trick for windowed max
- [Moving Average from Data Stream](moving-average.md) — same window shape, different aggregate

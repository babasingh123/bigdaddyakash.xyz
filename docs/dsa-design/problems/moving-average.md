# Design Moving Average from Data Stream

!!! note "The disguise"
    Interviewer says: **"Design a class to calculate the moving average."**
    They mean: **"Implement a fixed-size queue (circular buffer) with a running sum."**

## Problem

Build `MovingAverage(size)`:

- `next(val) -> float` — incorporate `val` into the stream and return the moving average of the last `size` numbers (or all numbers if fewer than `size` have been seen).

## What it really tests

- **Fixed-size queue / circular buffer.**
- **Running-sum trick** — don't recompute the sum every call.
- **O(1) per-call complexity.**

## Approach

The naive solution: store all values in a list, sum the last `size` per call. That's O(size) per `next`, and O(n) memory after *n* calls — both bad.

**Key insight:** when a new value arrives, you can update the average by:

1. Adding the new value to a `running_sum`.
2. If the buffer is full, subtracting the value that just fell off the window.
3. Dividing `running_sum` by the current buffer size.

That's **O(1) per call** and **O(size) memory**.

**Data structures:**

- A **Deque** of size up to `size`. Append on the right, `popleft` when full. Or:
- A **circular buffer**: a fixed array of length `size`, plus an `index` and a `count`.

Deque is simpler. Circular buffer wins on memory locality in tight loops.

**Why a queue, not just a sum and a count?** Because when a value leaves the window, you need its exact value to subtract. The queue stores the values for exactly that purpose.

## Code sketch

```python
from collections import deque

class MovingAverage:
    def __init__(self, size: int):
        self.size = size
        self.q: deque[int] = deque()
        self.s = 0

    def next(self, val: int) -> float:
        if len(self.q) == self.size:
            self.s -= self.q.popleft()
        self.q.append(val)
        self.s += val
        return self.s / len(self.q)
```

The circular-buffer version (slightly faster, fixed memory layout):

```python
class MovingAverageCircular:
    def __init__(self, size: int):
        self.size = size
        self.buf = [0] * size
        self.i = 0
        self.count = 0
        self.s = 0

    def next(self, val: int) -> float:
        self.s -= self.buf[self.i]
        self.buf[self.i] = val
        self.s += val
        self.i = (self.i + 1) % self.size
        self.count = min(self.count + 1, self.size)
        return self.s / self.count
```

## Complexity

- `next`: **O(1)**.
- Space: **O(size)**.

## Edge cases

- **Fewer than `size` values seen.** Divide by the actual count, not by `size`. Both implementations handle this with `len(q)` or `count`.
- **`size = 1`.** Reduces to "return the latest value." Code still works.
- **Negative or zero values.** No special handling needed — arithmetic is the same.
- **Floating-point drift.** With many updates over a long period, the running sum can accumulate tiny errors. For interview purposes ignore; for real streams, consider Kahan summation.

## Linked concepts

- [Rate Limiting & Counting category overview](../by-category/rate-limiting-counting.md)
- [Hit Counter](hit-counter.md) — time-windowed instead of count-windowed
- [Sliding Window Maximum](sliding-window-max.md) — same window, different aggregate

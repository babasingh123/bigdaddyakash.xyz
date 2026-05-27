# Design Card Shuffle

!!! note "The disguise"
    Interviewer says: **"Design a deck of cards that can shuffle and reset"**
    They mean: **"Implement the Fisher–Yates shuffle"**

## Problem

- `Solution(nums)` — initialize with an array.
- `reset()` — return the original array.
- `shuffle()` — return a uniformly random permutation.

(LeetCode 384.)

## What it really tests

- Why naïve approaches fail:
    - Pick a random index, append to output, delete from input → O(n²) due to deletes, and easy to bias.
    - Assign each element a random key and sort → O(n log n); correct, but slower.
- Fisher–Yates does it in O(n) with a clean uniformity proof: every permutation has probability `1 / n!`.

## Approach

In-place Fisher–Yates:

```
for i in 0..n-1:
    j = random integer in [i, n-1]
    swap a[i], a[j]
```

Why uniform? Position 0 receives one of n elements with probability `1/n`. Conditional on that, position 1 receives one of the remaining `n - 1` with probability `1/(n - 1)`, and so on. The product is `1 / n!`.

For `reset` keep the original array untouched and return a copy.

## Code sketch

```python
import random

class Solution:
    def __init__(self, nums: list[int]):
        self.original = nums[:]
        self.arr = nums[:]

    def reset(self) -> list[int]:
        self.arr = self.original[:]
        return self.arr

    def shuffle(self) -> list[int]:
        a = self.arr
        for i in range(len(a)):
            j = random.randint(i, len(a) - 1)
            a[i], a[j] = a[j], a[i]
        return a
```

## Complexity

- `reset`: O(n) (copy).
- `shuffle`: O(n).
- Space: O(n) for the original snapshot.

## Edge cases

- Empty array → both methods are no-ops.
- Single-element array → only one permutation exists.
- `random.randint(i, n - 1)` must be inclusive of both ends — off-by-one here is the classic Fisher–Yates bug that biases the distribution.
- A reservoir-sampling variant uses the same swap idiom on a stream.

## Linked concepts

- [Game & Simulation problems](../by-category/game-simulation.md)
- The same algorithm underlies the `random.shuffle` standard-library implementation.

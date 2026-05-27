# Design Number Allocator (Phone Directory)

!!! note "The disguise"
    Interviewer says: **"Allocate and release phone numbers from a fixed range"**
    They mean: **"Same free-list trick as a seat manager — track which numbers are available"**

## Problem

(LeetCode 379.)

- `PhoneDirectory(maxNumbers)` — numbers `0..maxNumbers - 1` initially free.
- `get()` — allocate any free number (return `-1` if exhausted).
- `check(number)` — `True` iff the number is still free.
- `release(number)` — mark as free again.

## What it really tests

- Knowing you need **two views**: "is this specific number free?" (O(1) → HashSet) and "give me any free one" (O(1) → deque or stack).
- Keeping the two structures synchronized on every mutation.
- Subtle bug: `release` of an already-free number should be a no-op.

## Approach

State:

- `free_set: set[int]` — fast `check`.
- `free_list: deque[int]` — fast `get`.

`get()`:

- Pop from `free_list`. If empty → `-1`. Also remove from `free_set`.

`check(n)`:

- `n in free_set`.

`release(n)`:

- If `n` not already in `free_set`, add it to both structures.

We *don't* maintain order — any free number satisfies the spec. If "smallest free number" were required, swap the deque for a min-heap and lazily skip stale entries on pop.

## Code sketch

```python
from collections import deque

class PhoneDirectory:
    def __init__(self, maxNumbers: int):
        self.free_set = set(range(maxNumbers))
        self.free_list = deque(range(maxNumbers))

    def get(self) -> int:
        if not self.free_list:
            return -1
        n = self.free_list.popleft()
        self.free_set.remove(n)
        return n

    def check(self, number: int) -> bool:
        return number in self.free_set

    def release(self, number: int) -> None:
        if number not in self.free_set:
            self.free_set.add(number)
            self.free_list.append(number)
```

## Complexity

- All three operations: O(1) average.
- Space: O(maxNumbers).

## Edge cases

- Releasing an already-free number → no-op (the guard prevents duplicates in the deque).
- Allocating beyond capacity → `-1`.
- Very large `maxNumbers` with sparse usage: replace the eager set construction with a `next_seat` pointer (the trick from [Seat Reservation](./seat-reservation.md)).

## Linked concepts

- [Allocation & Resource problems](../by-category/allocation-resource.md)
- See [Seat Reservation](./seat-reservation.md) for the "smallest first" variant.

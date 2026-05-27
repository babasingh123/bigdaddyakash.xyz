# Design Seat Reservation System

!!! note "The disguise"
    Interviewer says: **"Allocate the lowest-numbered free seat; users may also un-reserve seats"**
    They mean: **"Use a min-heap (or TreeSet) of free seats"**

## Problem

- `SeatManager(n)` — seats `1..n` start free.
- `reserve()` — return the lowest-numbered free seat and mark it taken.
- `unreserve(seat)` — release a seat.

## What it really tests

- "Lowest free seat" + "freed seats become reusable" = a structure with both *insert* and *extract-min* in O(log n).
- Options:
    - **Min-heap**: O(log n) extract and insert. Allows duplicates — guard against double-unreserve.
    - **TreeSet** (`SortedList` in Python): O(log n) for both, plus natural dedup.
- Pre-populating the structure with all n seats up front is wasteful for large n; instead, track "next never-used seat" + a free-list heap.

## Approach

Two pieces of state:

- `next_seat` — counter starting at 1 for seats never yet touched.
- `freed` — min-heap of seats released by `unreserve`.

`reserve()`:

- If the heap is non-empty, pop and return the smallest freed seat.
- Otherwise return `next_seat` and increment it.

`unreserve(seat)`:

- Push it onto the heap.

Why this hybrid? Allocating an array of n booleans is O(n) memory up front; this scheme uses only O(freed seats) extra space.

## Code sketch

```python
import heapq

class SeatManager:
    def __init__(self, n: int):
        self.n = n
        self.next_seat = 1
        self.freed: list[int] = []

    def reserve(self) -> int:
        if self.freed:
            return heapq.heappop(self.freed)
        seat = self.next_seat
        self.next_seat += 1
        return seat

    def unreserve(self, seat: int) -> None:
        heapq.heappush(self.freed, seat)
```

## Complexity

- `reserve`: O(log n).
- `unreserve`: O(log n).
- Space: O(freed seats).

## Edge cases

- All seats reserved, then one freed → the next `reserve` returns the freed seat (not n + 1).
- Double-unreserve of the same seat → in a strict spec, raise; otherwise the heap can return the same seat twice.
- `n == 0` → `reserve` should fail; add a guard.
- Asking for top-K seats: pop K times from the heap.

## Linked concepts

- [Allocation & Resource problems](../by-category/allocation-resource.md)
- Cousin of [Number Allocator](./number-allocator.md), which uses the same free-list idea.

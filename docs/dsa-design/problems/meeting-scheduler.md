# Design Meeting Scheduler

!!! note "The disguise"
    Interviewer says: **"Design a calendar booking system."**
    They mean: **"Implement a TreeMap of intervals with overlap detection using `floorKey` / `ceilingKey`."**

## Problem

Build a `MyCalendar` supporting:

- `book(start, end) -> bool` — return `True` if the half-open interval `[start, end)` doesn't overlap any existing booking; record it and return `True`. Else, return `False`.

Variants:

- **Double booking allowed up to k=2** (MyCalendarTwo): allow if at most one overlap exists.
- **k overlapping bookings** (MyCalendarThree): return the max number of concurrent bookings after each booking.

## What it really tests

- **TreeMap / SortedDict** for sorted-by-start intervals.
- **`floorEntry(start)` and `ceilingEntry(start)`** to check the two candidate neighbors for overlap in O(log n).
- **Half-open interval semantics** — `[s1, e1)` and `[s2, e2)` overlap iff `s1 < e2 and s2 < e1`.

## Approach

Brute force: keep a list of intervals, check every existing one for overlap. O(n) per booking — fails for large *n*.

**TreeMap solution.** Maintain a TreeMap keyed by `start`, value is `end`. To check if `[s, e)` overlaps anything:

1. Look at the interval *just before* `s` — the floor entry. If it exists and its `end > s`, there's overlap.
2. Look at the interval *just after* `s` — the ceiling entry. If it exists and its `start < e`, there's overlap.

Only two checks. Both are O(log n).

If no overlap, insert `(s, e)` into the TreeMap. Otherwise, return `False`.

**Why exactly two neighbors?** Because the existing intervals are non-overlapping among themselves, any new interval can only overlap with the one *containing or just before* its start, or the one *just at or after* its start. There's no third case.

**Why `floorKey` and `ceilingKey`?** They're the TreeMap operations that return the largest key ≤ target and the smallest key ≥ target. Without them, you'd need to binary-search a sorted list manually — same complexity, more code.

**In Python:** `sortedcontainers.SortedDict` gives you the same operations. `bisect` on a sorted list of starts also works (and is often clearer in code-only environments).

## Code sketch

```python
from sortedcontainers import SortedDict

class MyCalendar:
    def __init__(self):
        self.book_map: SortedDict[int, int] = SortedDict()  # start -> end

    def book(self, start: int, end: int) -> bool:
        # ceiling: smallest key >= start
        i = self.book_map.bisect_left(start)
        keys = self.book_map.keys()
        # check neighbor BEFORE
        if i > 0:
            prev_start = keys[i - 1]
            prev_end = self.book_map[prev_start]
            if prev_end > start:
                return False
        # check neighbor AT/AFTER
        if i < len(keys):
            nxt_start = keys[i]
            if nxt_start < end:
                return False
        self.book_map[start] = end
        return True
```

A library-free version with `bisect` on parallel arrays is just as good:

```python
from bisect import bisect_left, insort

class MyCalendarBisect:
    def __init__(self):
        self.starts: list[int] = []
        self.ends: list[int] = []

    def book(self, start: int, end: int) -> bool:
        i = bisect_left(self.starts, start)
        if i > 0 and self.ends[i - 1] > start:
            return False
        if i < len(self.starts) and self.starts[i] < end:
            return False
        self.starts.insert(i, start)
        self.ends.insert(i, end)
        return True
```

`list.insert` is O(n), so this is O(n) per booking — fine for moderate sizes but the SortedDict version is O(log n).

## Complexity

- `book`: **O(log n)** with TreeMap/SortedDict; **O(n)** with sorted lists due to insertion shifting.
- Space: **O(n)**.

## Edge cases

- **Touching intervals.** `[1, 5)` and `[5, 10)` do *not* overlap (half-open). Verify with the interviewer.
- **Same start as an existing booking.** Forced overlap → return `False`.
- **Booking inside an existing one.** Floor entry's `end` exceeds `start` → overlap.
- **End <= start.** Invalid input; reject up front.

## Linked concepts

- [Scheduling & Priority category overview](../by-category/scheduling-priority.md)
- [Meeting Rooms II](meeting-rooms-ii.md) — overlapping intervals problem with heaps
- [Snapshot Array](snapshot-array.md) — also uses sorted-map operations

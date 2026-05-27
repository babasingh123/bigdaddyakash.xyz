# Design Meeting Rooms II

!!! note "The disguise"
    Interviewer says: **"Find the minimum number of conference rooms needed."**
    They mean: **"Sort by start time, sweep, and use a Min Heap of end times."**

## Problem

Given an array of meetings `[(start, end), ...]`, find the **minimum number of conference rooms** required so that no two overlapping meetings share a room.

LeetCode 253. The "design" framing usually adds: "now support streaming meetings and return current room count."

## What it really tests

- **Line sweep** — process events in chronological order.
- **Min Heap of end times.** Tells you "what's the earliest a room becomes free?"
- **Greedy reuse** — whenever the next meeting starts after the earliest-finishing meeting ends, reuse that room.

## Approach

The intuition: think of each meeting as needing a "room." At any moment, the number of rooms in use equals the number of meetings currently in progress. We want the **max simultaneous meetings**.

### Heap-based sweep

1. Sort meetings by start time.
2. Use a Min Heap of end times. The top of the heap is the room whose meeting finishes earliest.
3. For each meeting in sorted order:
    - If `heap` is non-empty and `heap[0] <= meeting.start`, that room is free — pop it (reuse the room).
    - Push the current meeting's end onto the heap (the room is now booked until that time).
4. Answer = heap size at the end (or `max(heap_size_during_loop)` if you want to track concurrent peak).

**Why does this work?** Sort guarantees we process meetings in arrival order. The heap's top is "next available room." If that room is free by the time the current meeting starts, we don't need a new one — we just push its end time and the heap size stays the same. If not, we push without popping, and the heap grows. Heap size at any point = rooms in use right now.

**Why min-heap and not "scan all rooms"?** We only care about the room that frees up *first*. Scanning all rooms is O(rooms) per meeting; the min-heap reduces this to O(log rooms).

### Event-based alternative (line sweep with +1/-1)

Create a list of `(time, +1)` for each start and `(time, -1)` for each end. Sort. Sweep left to right, maintaining a running counter; track the max. The trick: at the same time, process `-1` (end) before `+1` (start) so back-to-back meetings can share a room. Both approaches give the same answer.

## Code sketch

Heap version:

```python
import heapq

def min_meeting_rooms(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0
    intervals.sort(key=lambda x: x[0])
    rooms: list[int] = []  # min-heap of end times
    for start, end in intervals:
        if rooms and rooms[0] <= start:
            heapq.heappop(rooms)
        heapq.heappush(rooms, end)
    return len(rooms)
```

Event-sweep version:

```python
def min_meeting_rooms_events(intervals: list[list[int]]) -> int:
    events: list[tuple[int, int]] = []
    for s, e in intervals:
        events.append((s, 1))
        events.append((e, -1))
    events.sort()
    cur = best = 0
    for _, delta in events:
        cur += delta
        best = max(best, cur)
    return best
```

A streaming version (rooms-currently-in-use over a stream of meetings):

```python
class RoomTracker:
    def __init__(self):
        self.heap: list[int] = []

    def add(self, start: int, end: int) -> int:
        while self.heap and self.heap[0] <= start:
            heapq.heappop(self.heap)
        heapq.heappush(self.heap, end)
        return len(self.heap)
```

## Complexity

- Sort: **O(n log n)**.
- Heap operations: **O(n log n)** total.
- Total: **O(n log n)**.
- Space: **O(n)** for the heap.

## Edge cases

- **Empty intervals.** Return 0.
- **All meetings at the same time.** Heap grows to size `n`; never pops. Answer = `n`.
- **Back-to-back meetings (`end == next.start`).** Should share a room (half-open intervals). `heap[0] <= start` (with `<=`) handles this; using `<` would mis-count.
- **Single meeting.** Heap goes to size 1; returns 1.

## Linked concepts

- [Scheduling & Priority category overview](../by-category/scheduling-priority.md)
- [Meeting Scheduler](meeting-scheduler.md) — overlap detection with TreeMap
- [Task Scheduler](task-scheduler.md) — different scheduling, same heap idea

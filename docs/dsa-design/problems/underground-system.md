# Design Underground System

!!! note "The disguise"
    Interviewer says: **"Design a metro card system that returns the average travel time between stations"**
    They mean: **"Implement two nested HashMaps — one for in-flight check-ins, one for accumulated route stats"**

## Problem

Implement:

- `checkIn(id, station, t)` — passenger taps in.
- `checkOut(id, station, t)` — passenger taps out.
- `getAverageTime(start, end)` — mean travel time across all completed trips between two stations.

## What it really tests

- Two-level HashMap design: one keyed by passenger (live state), one keyed by route (running aggregate).
- Knowing **not to store every trip** when you only need an average — keep a running `(sum, count)` instead.
- Cleaning up state on checkout so memory doesn't leak.

## Approach

Two dictionaries:

1. `checkins: dict[id, (station, time)]` — the passenger's currently open trip.
2. `routes: dict[(start, end), (sum_time, count)]` — running totals per route.

Why not `dict[(start, end), list[int]]` and average on the fly? It works, but `getAverageTime` becomes O(n) and memory is O(total trips). Storing `(sum, count)` is O(1) per query and O(distinct routes) memory.

Why a tuple key `(start, end)` and not nested dicts `routes[start][end]`? Both are fine — tuples are slightly cleaner in Python.

On `checkOut`, `pop` from `checkins` so an abandoned check-in doesn't sit forever.

## Code sketch

```python
class UndergroundSystem:
    def __init__(self):
        self.checkins = {}   # id -> (station, time)
        self.routes = {}     # (start, end) -> [sum_time, count]

    def checkIn(self, id: int, stationName: str, t: int) -> None:
        self.checkins[id] = (stationName, t)

    def checkOut(self, id: int, stationName: str, t: int) -> None:
        start, t0 = self.checkins.pop(id)
        key = (start, stationName)
        if key not in self.routes:
            self.routes[key] = [0, 0]
        self.routes[key][0] += t - t0
        self.routes[key][1] += 1

    def getAverageTime(self, startStation: str, endStation: str) -> float:
        s, c = self.routes[(startStation, endStation)]
        return s / c
```

## Complexity

All three operations: O(1) average. Space O(active passengers + distinct routes).

## Edge cases

- Double check-in for the same id: the second overwrites the first (matches LeetCode 1396).
- `getAverageTime` for an unseen route → KeyError. The problem guarantees at least one prior completed trip.
- Floating point: divide only at query time; keep `sum_time` as an integer.

## Linked concepts

- [Graph & Network problems](../by-category/graph-network.md) — stations form a route graph even though we never traverse it.
- The same nested-HashMap idea backs [Time-Based Key-Value Store](./time-based-kv-store.md) and [Snapshot Array](./snapshot-array.md).

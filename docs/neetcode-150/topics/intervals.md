# Intervals

> **6 problems** · no explicit prerequisites — pulls from sorting, heaps, and greedy

## What this topic teaches

Interval problems are about **sorting + sweep**: once you sort the intervals (almost always by start time, occasionally by end time), the answer comes from a single linear scan with a tiny amount of state — the "current merged interval," a heap of active endpoints, or a counter. The unlock is realising that interval problems on `N` items look quadratic if you compare every pair, but sorting gives you a canonical order that lets you decide each interval's fate in O(1).

The two main shapes are **merge / cover** (sweep with a "current" interval, extend or emit-and-restart on the next one — *Merge Intervals*, *Insert Interval*) and **scheduling / resource counting** (use a heap of end-times or a sorted-events trick to count maximum concurrency — *Meeting Rooms II*). The greedy *Non Overlapping Intervals* problem cheats slightly by sorting on **end** time instead of start: keeping the interval that ends earliest leaves maximum room for the rest.

## Prerequisites

*This topic doesn't have a dedicated prerequisite in the NeetCode course tree — it draws on sorting, greedy (Kadane / interval-greedy), and heaps.*

## Core patterns

### Pattern 1: Sort by start + merge sweep

Sort intervals by start. Initialise `merged = [intervals[0]]`. For each subsequent `[s, e]`: if `s <= merged[-1].end`, **extend**: `merged[-1].end = max(merged[-1].end, e)`. Otherwise **append**. *Merge Intervals* is exactly this; *Insert Interval* does the same thing on the already-sorted list, inserting the new interval in three phases (all-before, overlapping, all-after).

### Pattern 2: Greedy by end-time (earliest-finish-first)

For *Non Overlapping Intervals* (remove the minimum number to make the rest disjoint), sort by **end** time. Greedily keep intervals whose start is `>= last_kept_end`; count the ones you skip. The proof: any optimal solution can be transformed without loss into the "always pick the earliest-ending compatible interval" solution. This is the classic activity-selection greedy.

### Pattern 3: Heap of end-times for max concurrency

*Meeting Rooms II* (minimum rooms needed). Sort meetings by start time. Maintain a min-heap of end-times of currently-occupied rooms. For each meeting: if the heap's top end-time `<= start`, pop (that room is free now); push the new end-time. Answer is the maximum heap size during the sweep.

### Pattern 4: All-or-nothing overlap check

*Meeting Rooms* (can a single person attend all?). Sort by start; return `False` if any `intervals[i].start < intervals[i-1].end`. Single line of logic post-sort.

### Pattern 5: Offline queries with heap (sort + lazy-eviction)

*Minimum Interval to Include Each Query*: sort intervals by start, sort queries by value (keeping original indices). Use a min-heap keyed by interval *length*, storing `(length, end_time)`. For each query `q`: push every interval with `start <= q`; lazily pop entries with `end_time < q`; the heap's top is the smallest covering interval. O((N + Q) log N).

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Insert Interval | Medium |
| 2 | Merge Intervals | Medium |
| 3 | Non Overlapping Intervals | Medium |
| 4 | Meeting Rooms | Easy |
| 5 | Meeting Rooms II | Medium |
| 6 | Minimum Interval to Include Each Query | Hard |

## Tips

- **Pick the sort key consciously.** Sort by start for merge / overlap-detection / concurrency. Sort by end for activity-selection-style greedy.
- **Two-pointer "events" trick for Meeting Rooms II.** Build two sorted arrays — all starts, all ends. Walk them together: increment a counter on a start, decrement on an end. The max counter value is the answer. Often shorter than the heap version.
- **For *Insert Interval*, the three-phase split is cleaner than a general merge.** Don't try to call merge as a subroutine — write the explicit "before / overlap / after" loop.
- **Lazy eviction beats eager eviction in heap-based interval problems.** Don't try to remove specific items from a heap; just leave stale entries and pop them when they surface.

## Linked concepts

- [Meeting Scheduler (DSA Design)](../../dsa-design/problems/meeting-scheduler.md) — find the earliest common free slot between two calendars: two-pointer over sorted intervals.
- [Meeting Rooms II (DSA Design)](../../dsa-design/problems/meeting-rooms-ii.md) — production version of the heap-of-end-times pattern.
- [Disjoint Intervals (DSA Design)](../../dsa-design/problems/disjoint-intervals.md) — maintain a sorted set of disjoint intervals under add/query operations.
- [Range Sum Query (DSA Design)](../../dsa-design/problems/range-sum-query.md) — interval queries answered with prefix sums / segment trees.

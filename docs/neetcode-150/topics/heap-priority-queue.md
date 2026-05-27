# Heap / Priority Queue

> **7 problems** · prerequisites below

## What this topic teaches

A heap is what you reach for when you keep asking the question "what's the smallest (or largest) thing I have right now?" repeatedly while the set of things changes. Push and pop are both O(log n); peek at the extreme is O(1). That trade — slightly slower inserts in exchange for fast access to the extreme — wins whenever the problem doesn't need the full sort, just the top item over and over.

The patterns split into three buckets: **top-k** (keep a heap of size `k`, throw away anything that can't be top-k), **two heaps** (a max-heap of the smaller half and a min-heap of the larger half — together they keep the median at the boundary), and **simulated scheduling** (always pop the highest-priority task; sometimes maintain a parallel cooldown / availability structure). Once you map a problem onto one of those, the code is mostly heap boilerplate.

## Prerequisites

- Heap Properties (Data Structures & Algorithms for Beginners)
- Push and Pop (Data Structures & Algorithms for Beginners)
- Heapify (Data Structures & Algorithms for Beginners)
- Two Heaps (Advanced Algorithms)

## Core patterns

### Pattern 1: Bounded-size top-k heap

Maintain a heap of size exactly `k`: push the new item; if the size exceeds `k`, pop the smallest (using a min-heap to track the k *largest*) or the largest (max-heap for k smallest). *Kth Largest Element In a Stream* and *K Closest Points to Origin* are textbook applications. O(n log k) — much better than O(n log n) when `k ≪ n`.

### Pattern 2: Two heaps for running median

A max-heap (`small`) holds the lower half; a min-heap (`large`) holds the upper half. Invariants: `len(small) >= len(large)`, and `small`'s top ≤ `large`'s top. On `add(num)`: push to `small`, then move `small`'s top to `large`, then rebalance sizes. The median is `small[0]` if odd-sized, else the average of the two tops. *Find Median From Data Stream* is exactly this.

### Pattern 3: Greedy pop-highest-priority simulation

Used for scheduling and load-balancing problems. *Last Stone Weight*: repeatedly pop the two heaviest stones, push back their difference. *Kth Largest Element In An Array*: heapify and pop `k` times (O(n + k log n)), or quickselect O(n) average. *Task Scheduler*: max-heap of frequencies, on each tick pop the most-frequent available task, schedule it, decrement, push it back after its cooldown expires.

### Pattern 4: Heap as a composite priority

When the "priority" is a tuple, just push tuples — Python's heap compares lexicographically. *Design Twitter* pushes `(-timestamp, tweet_id, user_id, next_pointer)` and pops the most-recent tweet across all followees in O((U + F) log F) per `getNewsFeed`.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Kth Largest Element In a Stream | Easy |
| 2 | Last Stone Weight | Easy |
| 3 | K Closest Points to Origin | Medium |
| 4 | Kth Largest Element In An Array | Medium |
| 5 | Task Scheduler | Medium |
| 6 | Design Twitter | Medium |
| 7 | Find Median From Data Stream | Hard |

## Tips

- **Python's `heapq` is a min-heap only.** For a max-heap, push the negated value (or negate one tuple field). Forgetting this is the #1 heap bug in Python interview code.
- **Size-k heaps run forever.** As long as you pop after every push that would exceed `k`, the heap can't grow unbounded — perfect for streaming top-k.
- **For *Task Scheduler*, two structures.** A max-heap of frequencies for "ready to schedule," and a queue of `(available_time, frequency_remaining)` for "in cooldown." On each tick, push expired cooldowns back to the heap.
- **For *Kth Largest Element In An Array*, mention quickselect.** Heaps are O(n log k); quickselect is O(n) average and the canonical follow-up question.
- **For two-heaps median**, decide ownership of the middle element upfront (`small` holds it when odd). Doing it consistently makes the rebalance one-line.

## Linked concepts

- [Median Finder (DSA Design)](../../dsa-design/problems/median-finder.md) — two-heaps median, with API and edge cases.
- [Kth Largest Stream (DSA Design)](../../dsa-design/problems/kth-largest-stream.md) — production size-k min-heap.
- [Top K Frequent (DSA Design)](../../dsa-design/problems/top-k-frequent.md) — heap and bucket-sort alternatives.
- [Task Scheduler (DSA Design)](../../dsa-design/problems/task-scheduler.md) — heap + cooldown queue at scale.
- [CPU Task Scheduler (DSA Design)](../../dsa-design/problems/cpu-task-scheduler.md) — priority-queue scheduling with deadlines.
- [Stock Price Ticker (DSA Design)](../../dsa-design/problems/stock-price-ticker.md) — heap with stale-entry lazy deletion.

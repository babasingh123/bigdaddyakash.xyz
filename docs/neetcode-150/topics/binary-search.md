# Binary Search

> **7 problems** · prerequisites below

## What this topic teaches

Binary search is what you reach for whenever a problem has a **monotone predicate** — a yes/no test where the "yes" answers form a contiguous prefix or suffix of some search space. The literal "find a value in a sorted array" is just the simplest instance; the deeper skill is recognising that you can binary-search on *any* ordered domain, including the answer itself ("smallest eating speed that finishes in time", "smallest divisor", "min element in a rotated sorted array").

The two big ideas are: (1) shrink the search space by half each step using `mid = (lo + hi) // 2`, and (2) be religious about the loop invariant — what does "the answer lies in `[lo, hi]`" mean, inclusive/exclusive of which end, and what's true after the loop terminates? Most binary-search bugs are off-by-one issues, not algorithmic ones.

## Prerequisites

- Search Array (Data Structures & Algorithms for Beginners)
- Search Range (Data Structures & Algorithms for Beginners)

## Core patterns

### Pattern 1: Classic search in sorted array

`lo, hi = 0, n - 1`; loop while `lo <= hi`; on match return, otherwise narrow. Generalises to 2-D matrices that are row-major sorted (*Search a 2D Matrix*) by treating the matrix as a flat sorted array of length `m * n` and indexing as `(idx // n, idx % n)`.

### Pattern 2: Binary search on the answer

The search space isn't the input — it's the set of possible answers. *Koko Eating Bananas*: binary-search the eating speed `k` in `[1, max(piles)]`; the predicate `canFinish(k)` is monotone (faster speeds always work). Same pattern in *Median of Two Sorted Arrays* (search the partition position) and many scheduling/capacity problems. The rule of thumb: if you can write a fast `bool canDo(x)` and "doable" is monotone in `x`, binary-search `x`.

### Pattern 3: Pivoted / rotated arrays

A rotated sorted array still has one sorted half at every step — compare `arr[mid]` to `arr[lo]` (or `arr[hi]`) to figure out *which* half. Then check if the target lies in the sorted half (regular binary search there) or in the unsorted half (recurse). *Find Minimum In Rotated Sorted Array* and *Search In Rotated Sorted Array* share this template; only the post-decision differs.

### Pattern 4: Snapshot / timestamp search

*Time Based Key Value Store* keeps a sorted list of `(timestamp, value)` per key and binary-searches for the largest timestamp `<= target`. The "find largest index where predicate is true" template is `lo, hi = 0, n - 1; while lo < hi: mid = (lo + hi + 1) // 2 ...` — the `+ 1` in the midpoint biases rounding *up* so the loop terminates.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Binary Search | Easy |
| 2 | Search a 2D Matrix | Medium |
| 3 | Koko Eating Bananas | Medium |
| 4 | Find Minimum In Rotated Sorted Array | Medium |
| 5 | Search In Rotated Sorted Array | Medium |
| 6 | Time Based Key Value Store | Medium |
| 7 | Median of Two Sorted Arrays | Hard |

## Tips

- **Pick one template and stick with it.** Either `while lo <= hi` (search for exact match, return on hit) or `while lo < hi` (search for boundary, no match check inside loop). Mixing them is the #1 source of binary-search bugs.
- **Use `mid = lo + (hi - lo) // 2`** instead of `(lo + hi) // 2`. Python doesn't overflow but the habit transfers to Java/C++ and reads as deliberate.
- **For "binary search on the answer," draft the `canDo(x)` predicate first**, prove it's monotone, *then* wire up the search. The search is boilerplate; the predicate is the problem.
- **For rotated arrays, when `arr[mid] == arr[lo]` (duplicates), shrink linearly: `lo += 1`.** The standard logarithmic bound degrades but correctness is preserved.

## Linked concepts

- [Time Based KV Store (DSA Design)](../../dsa-design/problems/time-based-kv-store.md) — production version of the timestamp-search pattern.
- [Snapshot Array (DSA Design)](../../dsa-design/problems/snapshot-array.md) — sorted log per index, binary-search by snapshot id.
- [Find Peak Element (DSA Design)](../../dsa-design/problems/find-peak-element.md) — monotone-segment binary search on an unsorted but locally-decidable array.

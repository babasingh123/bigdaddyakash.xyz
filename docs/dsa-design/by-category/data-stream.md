# Data Stream & Real-Time

Stream problems share a single constraint: **you only get to see each element once**. You can't sort. You can't index back. You have to maintain just enough summary state to answer a query at any moment.

## The pattern

The recurring move: choose a structure that supports **O(log n) insert + O(1) or O(log n) query**, and that fits the *exact* query you're being asked.

- **Running median** → Two Heaps (Max Heap of lower half, Min Heap of upper half), rebalance after each insert.
- **Kth largest** → Min Heap of size K. New element > top? Pop, push.
- **Disjoint intervals from a stream** → TreeMap (sorted by start), check `floorKey` and `ceilingKey` to merge with neighbors.
- **Stream of characters / pattern detection** → Trie built from patterns (often in reverse — the Aho-Corasick trick).
- **Peak in a stream / array** → Binary search if the array is fully available; for true streams, you'd need a different formulation.

The category headline: **don't store the stream; store the summary**. Two heaps for median is a perfect example — you've kept O(n) elements but you've kept them in *exactly* the configuration that makes the query trivial.

## Problems

- [Stream of Characters](../problems/stream-of-characters.md) — Trie + sliding window on stream
- [Data Stream as Disjoint Intervals](../problems/disjoint-intervals.md) — TreeMap + interval merge
- [Find Median from Data Stream](../problems/find-median-stream.md) — Two Heaps
- [Kth Largest in Stream](../problems/kth-largest-stream.md) — Min Heap of size K
- [Find Peak Element](../problems/find-peak-element.md) — Binary search variant

## Why interviewers love this category

Stream problems are the cleanest test of "do you actually understand the data structure?" Sorting-based solutions are off the table — you can't sort an infinite stream — so the candidate has to reach for a heap, a tree, or a trie.

Signals interviewers grade on:

1. **Two-heap mastery.** Median Finder is *the* signature stream problem. Knowing it cold (and being able to walk through rebalancing) is table stakes for L4+ DSA interviews.
2. **Heap-size discipline for Kth largest.** A new candidate might keep all elements and sort. A better candidate keeps a Min Heap of *exactly K* elements. The size cap is the whole optimization.
3. **TreeMap for intervals.** Disjoint Intervals is where a `SortedDict` / `TreeMap` becomes obviously right. The `floorKey` / `ceilingKey` operations don't have a clean substitute in a HashMap.
4. **Binary search on unsorted.** Find Peak Element is the trick question — most candidates think binary search needs sorted input. The peak problem shows it just needs a *monotonic invariant* you can extract per step.

Stream problems are also where you start to feel the difference between "I know the data structure" and "I can deploy it under real-time constraints."

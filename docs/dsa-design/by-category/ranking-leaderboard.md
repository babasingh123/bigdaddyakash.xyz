# Ranking & Leaderboard

These problems share a single goal: *keep things in some kind of order so you can answer "what's at the top?" fast.* The answer is either a **heap** (when you want one extreme), a **TreeMap** (when you want the full sorted view), or **two heaps** (when you want the middle).

## The pattern

The unifying question is *how much order do I need?*

- **Single extreme** (top, max, min, kth) → one Heap.
- **Full sorted view with mutations** → TreeMap (`SortedDict`, red-black tree).
- **Middle / median** → Two Heaps that meet in the middle.
- **Window-bounded extreme** → Monotonic Deque.
- **Range queries with updates** → Segment Tree or Binary Indexed Tree (Fenwick).

The data structure choice is a function of *which slice* of the order you need to read fast and *how often you mutate*. A leaderboard with frequent score updates and "give me top 3" queries is the textbook case for `HashMap<player, score> + TreeMap<score, set<player>>` — one map for O(1) update, the other for O(log n) ranked retrieval.

## Problems

- [Leaderboard](../problems/leaderboard.md) — HashMap + TreeMap combination
- [Top K Frequent Elements](../problems/top-k-frequent.md) — HashMap + Min Heap of size K (or Bucket Sort)
- [Median Finder](../problems/median-finder.md) — Two Heaps (Max Heap + Min Heap)
- [Sliding Window Maximum](../problems/sliding-window-max.md) — Monotonic Deque
- [Range Sum Query (Mutable)](../problems/range-sum-query.md) — Segment Tree or Fenwick Tree

## Why interviewers love this category

This is where senior candidates pull away. The basic moves (sort, then take top K) are O(n log n) and pass the easy interview question. The advanced moves drop that to O(log n) per operation and require *naming* the right structure.

Interviewers are looking for:

1. **Heap vs sort tradeoff.** "Top K from a stream of n" should never trigger a sort. It's a Min Heap of size K, O(n log k). The candidate who says "sort and take last K" has revealed they think in arrays, not streams.
2. **Two-heap reflex for medians.** Median from a stream is *the* canonical two-heap problem. If you've seen it once, you should never miss it again — and interviewers know that.
3. **Knowing the Segment Tree / BIT exists.** Range-sum-with-updates is impossible to solve well without one of these. The candidate who proposes a prefix-sum array (and recomputes on update) is showing they only know the static version. Naming "Fenwick Tree" or "Segment Tree" is itself a signal.
4. **Monotonic deque as a "free heap."** Sliding Window Max in O(n) instead of O(n log k) is a beautiful trick — and explaining *why* the deque stays monotonic is a strong technical answer.

The category boundary is precisely where "sort and look at the top" stops being good enough.

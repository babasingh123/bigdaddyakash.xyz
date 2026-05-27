# Sliding Window

> **6 problems** · prerequisites below

## What this topic teaches

The sliding-window pattern turns "for every subarray of length up to `n`, compute something" — which is O(n²) or worse — into a single O(n) sweep by maintaining a *window* `[left, right]` and amortising work across moves. The key insight: when you advance `right`, you usually only need to add the new element; when you advance `left`, you usually only need to remove the old one. As long as each step is O(1) (or O(log k) with a tree/heap), the whole walk is linear.

There are two distinct shapes you must keep separate: **fixed-size windows** (size `k` is given, `left` and `right` move in lockstep — *Sliding Window Maximum*, anagram checks) and **variable-size windows** (`right` grows greedily, `left` shrinks while a condition is violated — *Longest Substring*, *Minimum Window Substring*, character-replacement). The fixed version is a sweep; the variable version is a two-pointer dance with a "shrink-while-bad" inner loop.

## Prerequisites

- Sliding Window Fixed Size (Advanced Algorithms)
- Sliding Window Variable Size (Advanced Algorithms)

## Core patterns

### Pattern 1: Variable window — shrink while invariant is violated

`left = 0`; for `right in range(n)`: add `arr[right]` to the window state; **while the window is "bad," shrink from the left** by removing `arr[left]` and incrementing `left`; record the answer. This is *Longest Substring Without Repeating Characters* (bad = a character repeats), *Longest Repeating Character Replacement* (bad = `window_size - max_count > k`), and *Minimum Window Substring* (bad = some required char is missing — record on "good"). Master this template; it's 90% of variable-window problems.

### Pattern 2: Fixed window — slide in lockstep

Advance `right`, then if `right - left + 1 > k`, advance `left`. The window stays exactly size `k`. *Permutation In String* compares the window's frequency vector to the pattern's after every slide; equality (often tracked by a `matches` counter that updates incrementally) means an anagram is at this position.

### Pattern 3: Best stock — past-min, current-price

*Best Time to Buy And Sell Stock* is the degenerate sliding-window case: keep a single pointer for the minimum-so-far on the left and a single pointer scanning on the right. Generalises to "maximum gain between two indices" anywhere.

### Pattern 4: Monotonic deque for sliding max/min

When the window-state query is "max / min in the window," a hash map of counts isn't enough — you need a structure that pops stale extrema cheaply. A **monotonic deque** stores indices whose values are strictly decreasing (for max); the front is always the window's max, and you pop the front when it falls out of `[left, right]`. *Sliding Window Maximum* is the canonical use; O(n) amortised.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Best Time to Buy And Sell Stock | Easy |
| 2 | Longest Substring Without Repeating Characters | Medium |
| 3 | Longest Repeating Character Replacement | Medium |
| 4 | Permutation In String | Medium |
| 5 | Minimum Window Substring | Hard |
| 6 | Sliding Window Maximum | Hard |

## Tips

- **Identify fixed vs variable first.** If the problem fixes `k`, you're in the lockstep template; if it asks for "longest / shortest / smallest" subarray with property P, you're in the shrink-while-bad template.
- **Track window state with `Counter` or a single running stat, not by re-scanning.** Re-scanning the window inside the loop quietly takes you back to O(n·k). Update by `+1` on the new element, `-1` on the removed one.
- **For *Minimum Window Substring*, maintain a `formed` counter** (how many distinct required chars currently have enough copies) instead of recomparing dictionaries — O(1) per move.
- **For *Sliding Window Maximum*, push indices onto the deque, not values.** You need positions to evict stale entries when they fall outside the window.

## Linked concepts

- [Rate Limiter (DSA Design)](../../dsa-design/problems/rate-limiter.md) — sliding-window rate limiting over timestamps.
- [Hit Counter (DSA Design)](../../dsa-design/problems/hit-counter.md) — count of events in the last N seconds, classic fixed-window.
- [Moving Average (DSA Design)](../../dsa-design/problems/moving-average.md) — fixed-window running average with a queue.
- [Sliding Window Max (DSA Design)](../../dsa-design/problems/sliding-window-max.md) — monotonic deque productionised as a streaming API.

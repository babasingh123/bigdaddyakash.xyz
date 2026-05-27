# Greedy

> **8 problems** · prerequisites below

## What this topic teaches

Greedy algorithms make the locally-optimal choice at every step and *hope* that this gives a globally-optimal answer. The catch — and the whole intellectual content of this topic — is that "hope" must be replaced with a **proof** (or at least an exchange-argument sketch): you have to convince yourself that no future step could ever benefit from swapping in a different choice you skipped. When the greedy structure works, the algorithm is one pass and the code is tiny; when it doesn't, you fall back to DP.

The problems in this set teach you to **see the greedy invariant**. *Jump Game*: track the farthest reachable index — if at any point the current index exceeds it, return false. *Gas Station*: if total gas ≥ total cost, a solution exists, and the starting index is the position right after the lowest cumulative gas-minus-cost. *Hand of Straights*: always start the next group at the smallest remaining card. *Partition Labels*: the rightmost occurrence of any character in the current window dictates the partition end. These look unrelated until you notice they all share one shape: *commit to the locally-forced choice and prove no future regret*.

## Prerequisites

- Kadane's Algorithm (Advanced Algorithms)

## Core patterns

### Pattern 1: Kadane's running-max

*Maximum Subarray*: `cur = max(nums[i], cur + nums[i]); best = max(best, cur)`. Either you extend the current subarray or you start fresh at `i` — locally cheap, globally optimal. The same shape with `min(...)` solves "minimum subarray," and combined with two passes handles circular variants.

### Pattern 2: Farthest-reachable sweep

*Jump Game*: maintain `farthest = max(farthest, i + nums[i])` as you walk; return false if `i > farthest`. *Jump Game II* extends this — also track the current-jump's end; when `i` reaches it, increment jump count and set the new end to `farthest`. BFS-on-the-array, basically.

### Pattern 3: Total-balance + lowest-point

*Gas Station*: if `sum(gas) < sum(cost)`, no solution. Otherwise, walk once: when running `tank` goes negative, **the next index** is the candidate start; reset `tank = 0` and continue. The proof: any station before the candidate would also fail before reaching it.

### Pattern 4: Sort + match smallest available

*Hand of Straights*: use a `Counter` keyed by card value, walk values in sorted order, and for each value with positive count, "spend" `groupSize` cards starting from it. If any required card is missing, return false. The locally-forced choice is "the smallest remaining card must start *some* group."

### Pattern 5: Position constraint propagation

*Partition Labels*: precompute `last[c]` = last occurrence of each character. Walk the string maintaining `end = max(end, last[s[i]])`; when `i == end`, emit a partition. Greedy because once you've seen any character of class `c`, the partition can't end before `last[c]`.

### Pattern 6: "Just match the constraints" (triplet merge)

*Merge Triplets to Form Target Triplet*: ignore any triplet that has a coordinate > target's coordinate (it would over-shoot and never come back). For the rest, check whether the max over each coordinate equals the target.

### Pattern 7: Wildcard parenthesis range

*Valid Parenthesis String*: maintain `lo` and `hi` = the min and max possible count of open parens given the wildcards. On `(`: both +1. On `)`: both −1. On `*`: `lo -= 1, hi += 1`. Clamp `lo` to 0 (negative would mean already invalid via the `*` choices). If `hi < 0` anywhere, return false. At the end, `lo == 0` means a valid choice exists.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Maximum Subarray | Medium |
| 2 | Jump Game | Medium |
| 3 | Jump Game II | Medium |
| 4 | Gas Station | Medium |
| 5 | Hand of Straights | Medium |
| 6 | Merge Triplets to Form Target Triplet | Medium |
| 7 | Partition Labels | Medium |
| 8 | Valid Parenthesis String | Medium |

## Tips

- **Greedy works → prove it works.** Before committing, sketch an exchange argument: "if I picked X here and the optimum picked Y, swapping Y for X doesn't hurt." If the swap might hurt, you need DP, not greedy.
- **Default to DP if uncertain.** Greedy that's wrong is silently wrong; DP that's slower is at least correct. In an interview, mention both and explain why greedy suffices.
- **Kadane is a one-line interview test.** Don't write a full table — `cur = max(x, cur + x)` is the answer.
- **For *Jump Game II*, the "BFS layers" framing is the cleanest mental model**: each "layer" is the set of indices reachable in `k` jumps, and the layer boundaries are exactly when `i == current_end`.
- **For *Hand of Straights*, use a `SortedDict` / sorted Counter or a min-heap** to keep "smallest remaining" cheap. A linear scan over a sorted-keys list is also fine when input is small.

## Linked concepts

Greedy is a strategy, not a structure — direct DSA-design parallels are rare. The greedy *sweep* mindset does show up in:

- [Disjoint Intervals (DSA Design)](../../dsa-design/problems/disjoint-intervals.md) — locally extend the current merged interval, or start a new one.
- [Meeting Rooms II (DSA Design)](../../dsa-design/problems/meeting-rooms-ii.md) — earliest-finish-first room reuse.
- [Task Scheduler (DSA Design)](../../dsa-design/problems/task-scheduler.md) — always schedule the most-frequent ready task.

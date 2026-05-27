# 1-D Dynamic Programming

> **12 problems** · prerequisites below

## What this topic teaches

Dynamic programming is the art of recognising that a problem has **overlapping subproblems** and an **optimal-substructure** property — so you can store each subproblem's answer once and reuse it instead of recomputing. The "1-D" in this topic name means the state is captured by a single integer index `i`: `dp[i]` is the answer to the problem restricted to the prefix (or suffix) ending at `i`, and the recurrence `dp[i] = f(dp[i-1], dp[i-2], …)` only looks back a constant number of steps.

This is the most important interview topic to drill, because every DP problem is the same workflow: (1) define the state in one sentence, (2) write the recurrence, (3) decide the base cases, (4) iterate (bottom-up) or memoise (top-down). The recurrence is the algorithm; if you can write it on paper, the code is mechanical. The trap most people fall into is jumping to code before pinning down what `dp[i]` *means* — and ending up with subtly off-by-one transitions.

## Prerequisites

- 1-Dimension DP (Data Structures & Algorithms for Beginners)
- Palindromes (Advanced Algorithms)

## Core patterns

### Pattern 1: Fibonacci-shape — `dp[i] = dp[i-1] + dp[i-2]`

The transition only looks at one or two predecessors. *Climbing Stairs* (`dp[i] = dp[i-1] + dp[i-2]`), *Min Cost Climbing Stairs* (`dp[i] = cost[i] + min(dp[i-1], dp[i-2])`), *House Robber* (`dp[i] = max(dp[i-1], dp[i-2] + nums[i])`). Constant-space variant: keep only the last two values in two scalars.

### Pattern 2: Circular variant — solve linear twice

*House Robber II* (houses arranged in a circle): can't rob both first and last. Solve linearly on `nums[:-1]` and on `nums[1:]`, take the max. The trick — "circular constraint → solve two linear subproblems" — recurs throughout DP.

### Pattern 3: Decision-per-character — `dp[i]` depends on a window of valid choices

*Decode Ways*: `dp[i]` = number of decodings of `s[:i]`; transition considers `s[i-1]` alone (1 digit) and `s[i-2:i]` (2 digits if in `[10..26]`). *Word Break*: `dp[i]` is "is `s[:i]` decomposable?"; transition: `dp[i] = any(dp[j] and s[j:i] in dict for j in range(i))`.

### Pattern 4: Expand-around-center palindromes

*Longest Palindromic Substring* and *Palindromic Substrings*: not "true" DP, but the same spirit. For each centre (n character-centres + n−1 gap-centres), expand outwards while `s[l] == s[r]`. O(n²) time, O(1) space, beats the textbook 2-D DP version on both axes.

### Pattern 5: Unbounded coin change (1-D over amount)

*Coin Change*: `dp[a]` = min coins for amount `a`. Transition: `dp[a] = 1 + min(dp[a - c] for c in coins if a >= c)`. Iterate amount outermost when minimising coin count; iterate coins outermost (and amount inwards from `c` to `target`) when *counting* combinations (*Coin Change II*).

### Pattern 6: Track two scalars — running max / min product

*Maximum Product Subarray*: a negative number can flip the sign, so keep both `cur_max` and `cur_min` ending at `i`. Update: `cur_max, cur_min = max(x, x*cur_max, x*cur_min), min(x, x*cur_max, x*cur_min)`; answer is the running max of `cur_max`.

### Pattern 7: Longest Increasing Subsequence

`dp[i]` = LIS ending at `i`. Transition: `dp[i] = 1 + max(dp[j] for j < i if nums[j] < nums[i], default=0)`. O(n²). Mention the patience-sorting / binary-search O(n log n) version as the optimisation: maintain a `tails` array, replace via `bisect_left`, length of `tails` is the answer.

### Pattern 8: Subset-sum / partition (boolean DP)

*Partition Equal Subset Sum*: target = `total / 2` (must be int). `dp` is a boolean array of size `target + 1`; `dp[0] = True`; for each `num`, iterate `s` from `target` down to `num` and set `dp[s] |= dp[s - num]`. The reverse iteration is essential — it prevents reusing the same number twice.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Climbing Stairs | Easy |
| 2 | Min Cost Climbing Stairs | Easy |
| 3 | House Robber | Medium |
| 4 | House Robber II | Medium |
| 5 | Longest Palindromic Substring | Medium |
| 6 | Palindromic Substrings | Medium |
| 7 | Decode Ways | Medium |
| 8 | Coin Change | Medium |
| 9 | Maximum Product Subarray | Medium |
| 10 | Word Break | Medium |
| 11 | Longest Increasing Subsequence | Medium |
| 12 | Partition Equal Subset Sum | Medium |

## Tips

- **Define `dp[i]` in one English sentence before writing code.** "`dp[i]` = max money robbable from houses `0..i`." If you can't say it cleanly, the recurrence won't be right either.
- **Bottom-up after top-down.** Solve recursively with memoisation first (it's easier to think in), then convert to a `for i in range(n):` table if you need constant-space optimisation.
- **Constant-space optimisation is a follow-up.** Don't lead with `dp_prev, dp_curr = 0, 0`. Get the O(n)-space version working, then optimise if asked.
- **For Coin Change, watch the iteration order:** for *fewest coins*, iterate amount outermost; for *counting combinations*, iterate coins outermost (otherwise you count `(1,2)` and `(2,1)` separately).
- **Don't confuse subsequence with substring.** Subsequence preserves relative order but not contiguity; substring is contiguous. The DP shapes are different — substring usually does *expand-around-center* or 2-D DP, subsequence does the standard `O(n²)` table.

## Linked concepts

DP problems don't map cleanly onto DSA-design APIs — they're algorithmic. But the **state-transition** mindset shows up in:

- [Logger Rate Limiter (DSA Design)](../../dsa-design/problems/logger-rate-limiter.md) — "do I emit?" decision based on prior state.
- [Stock Price Ticker (DSA Design)](../../dsa-design/problems/stock-price-ticker.md) — accumulated state evolving with each event.

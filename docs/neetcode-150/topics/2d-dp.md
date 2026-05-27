# 2-D Dynamic Programming

> **11 problems** · prerequisites below

## What this topic teaches

2-D DP is the next rung above 1-D DP: now `dp[i][j]` captures the answer for a subproblem indexed by **two** dimensions — often "first `i` characters of one string and first `j` of another" (*LCS*, *Edit Distance*, *Distinct Subsequences*, *Interleaving String*), or "row `i`, column `j`" in a grid (*Unique Paths*, *Longest Increasing Path*), or "amount-so-far" + "coin-index-so-far" (*Coin Change II*, *Target Sum*). The recurrence shape generalises from "look back one position" to "look back one position in each of two axes," and the table fills in row-major or with explicit memoisation.

Two big families to recognise: **string-pair DP** (the classical edit-distance / LCS family — fill an `(n+1) × (m+1)` table where the `+1` row/column models the empty-prefix base case) and **knapsack-style DP** (one axis is item index, the other is the resource budget). The hardest problems in this topic — *Burst Balloons*, *Regular Expression Matching* — break that mould: *Burst Balloons* uses interval DP where the state is "subproblem on range `[i..j]`," and *Regex Matching* requires three transitions per cell because of the `*` operator. But once you've nailed *LCS* and *Coin Change II*, the rest is variations.

## Prerequisites

- 2-Dimension DP (Data Structures & Algorithms for Beginners)
- 0/1 Knapsack (Advanced Algorithms)
- Unbounded Knapsack (Advanced Algorithms)
- LCS (Advanced Algorithms)

## Core patterns

### Pattern 1: Grid path-counting / path-min

*Unique Paths*: `dp[i][j] = dp[i-1][j] + dp[i][j-1]`, base case `dp[0][*] = dp[*][0] = 1`. Constant-space: a single row, updated left-to-right. Trivially extends to obstacle grids and minimum-path-sum variants.

### Pattern 2: LCS-shape — `(n+1) × (m+1)` table over two strings

`dp[i][j]` = LCS of `s1[:i]` and `s2[:j]`. If `s1[i-1] == s2[j-1]`, `dp[i][j] = dp[i-1][j-1] + 1`. Else `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`. *Longest Common Subsequence* is the template. *Edit Distance* differs only in the recurrence: insert / delete / replace correspond to `dp[i][j-1]`, `dp[i-1][j]`, `dp[i-1][j-1]`. *Distinct Subsequences* counts ways (so it sums) instead of taking a max.

### Pattern 3: Knapsack — items × budget

*Coin Change II* (count combinations): outer loop over coins, inner over amounts, `dp[a] += dp[a - c]` — this iteration order avoids counting `(1,2)` and `(2,1)` as distinct. *Target Sum*: reduce to subset-sum by computing `target = (sum + S) // 2`, then standard 0/1 knapsack count.

### Pattern 4: State-machine DP (stocks with cooldown)

*Best Time to Buy And Sell Stock With Cooldown*: state is "(day, holding-stock?)" with three transitions — buy, sell, rest. Two parallel 1-D arrays (`hold[i]`, `sold[i]`) handle this; the recurrence is the state-transition diagram written out.

### Pattern 5: Interleaving / mixed-string DP

*Interleaving String*: `dp[i][j]` = "can `s3[:i+j]` be formed by interleaving `s1[:i]` and `s2[:j]`?" Transition: `dp[i][j] = (dp[i-1][j] and s1[i-1] == s3[i+j-1]) or (dp[i][j-1] and s2[j-1] == s3[i+j-1])`. Boolean DP, O(n·m).

### Pattern 6: Memoised DFS on a grid (top-down DP)

*Longest Increasing Path In a Matrix*: from each cell, the answer is `1 + max(dfs(nbr) for nbr with grid[nbr] > grid[cell])`. Memoise on `(r, c)`. No risk of revisits because the DAG is induced by the strict-greater constraint.

### Pattern 7: Interval / "last operation" DP

*Burst Balloons*: `dp[i][j]` = max coins from bursting all balloons in `(i, j)` exclusive. The trick is to think of which balloon is the **last** to burst (not first): for that balloon `k`, the coins are `nums[i] * nums[k] * nums[j] + dp[i][k] + dp[k][j]`. Iterate by interval length, O(n³).

### Pattern 8: Regex with `*` — three-way branch

*Regular Expression Matching*: at `dp[i][j]`, if `p[j-1]` is `*`, two choices: skip the `x*` pair (`dp[i][j-2]`) or consume one char if it matches (`dp[i-1][j]` provided `s[i-1]` matches `p[j-2]`). Else, just compare `s[i-1]` against `p[j-1]` (allowing `.`).

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Unique Paths | Medium |
| 2 | Longest Common Subsequence | Medium |
| 3 | Best Time to Buy And Sell Stock With Cooldown | Medium |
| 4 | Coin Change II | Medium |
| 5 | Target Sum | Medium |
| 6 | Interleaving String | Medium |
| 7 | Longest Increasing Path In a Matrix | Hard |
| 8 | Distinct Subsequences | Hard |
| 9 | Edit Distance | Hard |
| 10 | Burst Balloons | Hard |
| 11 | Regular Expression Matching | Hard |

## Tips

- **Pad with a +1 row and column for string DP.** The 0-th row/column represents "match against the empty prefix," which usually has a clean base case (0, 1, or False) and removes a ton of boundary checks.
- **Top-down memoisation first.** Writing a recursive function with `@lru_cache(None)` is almost always easier to reason about for 2-D DP than wiring up the iteration order; convert to bottom-up only if you need O(1) row constant-space.
- **For knapsack-count problems, iteration order picks the semantics.** Coins outermost = combinations (order-independent); amount outermost = permutations (order matters).
- **Constant-space tricks: roll the table.** Most LCS-shape problems only need the previous row + current row, so two 1-D arrays (or even one with reverse iteration) suffice.
- **For interval DP (*Burst Balloons*), think about the *last* operation, not the first.** Doing it "last" gives you cleanly independent left/right subproblems; doing it "first" creates dependencies that block memoisation.

## Linked concepts

- [Range Sum Query (DSA Design)](../../dsa-design/problems/range-sum-query.md) — range queries on intervals, the data-structure cousin of interval DP.
- [Snapshot Array (DSA Design)](../../dsa-design/problems/snapshot-array.md) — stateful 2-D indexing (snapshot × index), similar in shape to the 2-D table.

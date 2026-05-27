# Backtracking

> **10 problems** · prerequisites below

## What this topic teaches

Backtracking is a structured way to enumerate every valid combination/permutation/configuration by building a partial solution incrementally, **trying** an extension, **recursing**, and **undoing** the extension before trying the next one. The mantra is **choose → explore → unchoose**. It's just DFS over an implicit tree where each node is a partial solution and each edge is "add one more piece."

The reason this topic has its own section (rather than living under "recursion") is the discipline of **pruning**: a backtracking algorithm without early termination quickly becomes exponential. The art is recognising *which constraint lets you abort a branch the instant it's hit* — duplicate-skipping for *Subsets II*, "remaining sum < 0" for *Combination Sum*, the column / diagonal sets for *N Queens*. Backtracking with good pruning is often **fast enough** even though the worst case is 2ⁿ or n!.

## Prerequisites

- Tree Maze (Data Structures & Algorithms for Beginners)
- Subsets (Advanced Algorithms)
- Combinations (Advanced Algorithms)
- Permutations (Advanced Algorithms)

## Core patterns

### Pattern 1: Subsets — include-or-exclude (or for-loop expansion)

Two equivalent shapes. **Include-or-exclude**: at index `i`, recurse twice — once with `nums[i]` added to `path`, once without — then unchoose. **For-loop expansion**: at each call, loop `j` from `start` to `n-1`, append `nums[j]`, recurse with `start = j+1`, pop. Both produce 2ⁿ subsets. *Subsets* uses either; *Subsets II* uses the for-loop version with `if j > start and nums[j] == nums[j-1]: continue` after sorting to skip duplicates.

### Pattern 2: Combinations with reuse and target

*Combination Sum*: at each step, try every candidate `c >= last_used` (so combinations aren't double-counted); recurse with `remaining - c`; prune when `remaining < 0`. Allowing reuse means you pass `start = i` (not `i+1`). *Combination Sum II* disallows reuse and may contain duplicates, so sort + dedup-skip + `start = i+1`.

### Pattern 3: Permutations — used-set

For permutations, **order matters**, so you loop over *all* candidates and skip ones already on the path. Maintain a `used` boolean array (or remove/restore from a list). For *Permutations* with duplicates, sort and apply: "skip `nums[i]` if `nums[i] == nums[i-1]` and `not used[i-1]`" — the canonical permutation-dedup rule.

### Pattern 4: Grid backtracking with mutation+restore

*Word Search* and similar grid problems: at each cell, mark the cell visited (overwrite with a sentinel), DFS to its 4 neighbours with the next character, restore the cell on return. The implicit trie-like pruning here is "if `grid[r][c] != word[i]`, abort immediately."

### Pattern 5: Partition / split with palindrome check

*Palindrome Partitioning*: at each step, for every end position `j >= start`, if `s[start:j+1]` is a palindrome, take that prefix and recurse from `j+1`. The is-palindrome check can be O(n²) precomputed if you want; the basic version is O(n) per check, fast enough.

### Pattern 6: Constraint-set N-Queens

*N Queens*: place one queen per row; maintain sets `cols`, `pos_diag` (`r + c`), `neg_diag` (`r - c`). On placement, add to sets; on backtrack, remove. The three sets give O(1) "is this square attacked?" checks — without them, the algorithm is needlessly quadratic.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Subsets | Medium |
| 2 | Combination Sum | Medium |
| 3 | Combination Sum II | Medium |
| 4 | Permutations | Medium |
| 5 | Subsets II | Medium |
| 6 | Generate Parentheses | Medium |
| 7 | Word Search | Medium |
| 8 | Palindrome Partitioning | Medium |
| 9 | Letter Combinations of a Phone Number | Medium |
| 10 | N Queens | Hard |

## Tips

- **Memorise the choose / explore / unchoose skeleton.** Every problem in this topic is a riff on it; if your code doesn't have a matching `path.append(...)` and `path.pop()` (or equivalent), something is wrong.
- **Sort the input before deduping.** All duplicate-skip tricks rely on equal elements being adjacent.
- **Append a *copy* of `path` when recording an answer.** `result.append(path[:])` — without the slice, every entry in `result` points at the same (later-emptied) list.
- **For *Generate Parentheses*, track `(open_used, close_used)` and prune when `close > open` or either exceeds `n`.** Avoids generating invalid strings entirely.
- **Pruning is where the speed lives.** Before writing the recursion, list every constraint that would let you abort early — `remaining < 0`, `len(path) > k`, attacked diagonal — and check it at the *top* of the recursive call.

## Linked concepts

Backtracking doesn't have direct DSA-design analogues (it's an algorithmic shape, not a data structure), but the **recursive DFS-with-state** idea recurs in:

- [Add Search Words (DSA Design)](../../dsa-design/problems/add-search-words.md) — recursive search with wildcard branching, structurally identical to a small backtracker.
- [Word Dictionary (DSA Design)](../../dsa-design/problems/word-dictionary.md) — same shape with a different API surface.
- [Minesweeper (DSA Design)](../../dsa-design/problems/minesweeper.md) — DFS flood-fill, the non-undo cousin of grid backtracking.

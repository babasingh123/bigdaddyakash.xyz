# Two Pointers

> **5 problems** · prerequisites below

## What this topic teaches

Two pointers is the smallest, sharpest tool in the kit: walk two indices over the same array (or two arrays) so that each step makes provable progress toward the answer, turning what looks like an O(n²) search into an O(n) sweep. The unlock is realizing that on a **sorted** or **monotone** structure, the value at `(left, right)` is enough to decide *which pointer should move* — you never need to consider the pairs you skipped.

The pattern shows up in two flavours: **opposite-ends** (start `left=0, right=n-1` and converge — palindromes, sorted Two Sum, *Container With Most Water*, *Trapping Rain Water*) and **same-direction / fast-slow** (linked-list cycles, partitioning, *Remove Duplicates*). Both share the invariant: at every step, the answer (if it exists) lies inside the still-unscanned window.

## Prerequisites

- Two Pointers (Advanced Algorithms)

## Core patterns

### Pattern 1: Opposite-ends convergence

`left = 0, right = n - 1`; at each step compute `f(arr[left], arr[right])` and move the pointer that *cannot* be part of an improved answer. In *Container With Most Water*, the shorter line is the limiting factor, so move it inward. In sorted *Two Sum*, if `sum < target` move `left` up; if `sum > target` move `right` down. The correctness proof is always the same: the pointer you don't move can never pair with anything better than what's left.

### Pattern 2: Fix one, two-pointer the rest

When the problem is "find a triple / quadruple summing to X," sort the array, fix the outermost index in a `for` loop, and two-pointer the remaining range. This is *3Sum*: sort, then for each `i`, run a two-pointer pass over `i+1..n-1`. Deduplicate by skipping equal neighbors at every level — that's the entire trick.

### Pattern 3: Precompute then two-pointer (Trapping Rain Water)

When the answer at each index depends on a max/min of its left and right sides, precompute `leftMax[i]` and `rightMax[i]` arrays — *or* maintain them as running scalars and let two pointers carry them. *Trapping Rain Water* is the canonical example: water at position `i` is `min(leftMax, rightMax) - height[i]`, and two pointers let you compute it in O(1) extra space.

### Pattern 4: Palindrome check / in-place partition

Walk `left` and `right` inward, skipping non-alphanumerics or comparing case-insensitively (*Valid Palindrome*). The same shape — `while left < right: …; left += 1; right -= 1` — also implements in-place quicksort partitioning, *Sort Colors*, and *Move Zeroes*.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Valid Palindrome | Easy |
| 2 | Two Sum II Input Array Is Sorted | Medium |
| 3 | 3Sum | Medium |
| 4 | Container With Most Water | Medium |
| 5 | Trapping Rain Water | Hard |

## Tips

- **Sorted input is a giant signal.** If the array is sorted (or you can afford to sort it), two pointers is almost always the right tool.
- **Always justify which pointer moves.** Write a one-line invariant: "we move `left` because no `j > right` can improve the sum." If you can't state it, your loop is probably wrong.
- **Dedup at every level for k-Sum.** In *3Sum*, skip equal values both in the outer `i` loop *and* after a successful `(left, right)` match — forgetting either produces duplicate triples.
- **For *Trapping Rain Water*, prefer the two-pointer solution over the precomputed-arrays one in interviews.** It demonstrates the deeper invariant ("whichever side has the smaller max is bounded and contributes water").

## Linked concepts

- [Sliding Window Max (DSA Design)](../../dsa-design/problems/sliding-window-max.md) — fast/slow pointer variant: window endpoints advancing in sync.
- [Disjoint Intervals (DSA Design)](../../dsa-design/problems/disjoint-intervals.md) — sweeping two indices over sorted starts/ends mirrors the same-direction two-pointer pattern.
- [Find Peak Element (DSA Design)](../../dsa-design/problems/find-peak-element.md) — directional decision-by-comparison, the same logic as opposite-ends convergence.

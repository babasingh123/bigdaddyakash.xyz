# Math & Geometry

> **8 problems** · no explicit prerequisites — leans on matrix indexing, modular arithmetic, and careful case-work

## What this topic teaches

Math & Geometry is the catch-all bucket for problems where the trick is **getting the indexing right** rather than choosing a clever data structure. *Rotate Image*, *Spiral Matrix*, and *Set Matrix Zeroes* are pure index-juggling — the algorithm is one paragraph of English ("transpose then reverse rows") and the implementation is ten lines once you've drawn the matrix on paper. *Pow(x, n)* and *Multiply Strings* exercise standard numerical tricks (fast exponentiation, schoolbook multiplication). *Happy Number* is Floyd's cycle detection wearing a math-problem hat. *Detect Squares* is a hashing-on-coordinates problem in disguise.

The skill this topic builds is **precise case-work**. There is no general technique to learn — instead, each problem has a small set of geometric or arithmetic identities you exploit (matrix rotation = transpose + horizontal flip, spiral = four bounds shrinking inward, fast pow = `x^n = (x^(n/2))^2` with parity branch). Get them wrong and you'll spend the interview fighting off-by-ones; get them right and the code is a one-shot.

## Prerequisites

*No NeetCode prerequisite course — relies on careful matrix indexing, modular arithmetic, and a tiny dose of cycle-detection (covered in Linked List).*

## Core patterns

### Pattern 1: Matrix rotation by transpose + reverse

*Rotate Image* (90° clockwise): **transpose** the matrix in place (swap `M[i][j]` and `M[j][i]` for `j > i`), then **reverse each row**. For 90° counter-clockwise: transpose, then reverse each column (equivalently, reverse the row order, then transpose). For 180°: reverse rows top-to-bottom, then reverse each row. Memorise these three.

### Pattern 2: Spiral traversal with four bounds

*Spiral Matrix*: maintain `top, bottom, left, right`. Walk along the top row left→right (then `top += 1`), right column top→bottom (then `right -= 1`), bottom row right→left if `top <= bottom` (then `bottom -= 1`), left column bottom→top if `left <= right` (then `left += 1`). The two `if` guards prevent re-traversing the centre row/column in single-strip matrices.

### Pattern 3: Use first row / column as in-place markers

*Set Matrix Zeroes*: instead of an extra `O(m+n)` boolean array, **use the matrix's own first row and first column as the marker buffer**. Two passes: first, scan the matrix; if `M[i][j] == 0`, set `M[i][0]` and `M[0][j]` to 0. Second, walk the inner cells; zero them if their row-marker or column-marker is 0. Handle the first row and first column separately with two scalar flags.

### Pattern 4: Fast exponentiation (binary)

*Pow(x, n)*: `x^n = (x^(n/2))^2` if `n` even, `x * (x^(n//2))^2` if odd. Recursion is fine; iterative with bit-walking is fancier. Handle `n < 0` by computing `pow(1/x, -n)`. O(log n) multiplications.

### Pattern 5: Floyd's cycle detection on a non-list

*Happy Number*: define `f(x) = sum_of_squared_digits(x)`. The sequence `x, f(x), f(f(x)), …` either reaches 1 (happy) or cycles. Apply Floyd's tortoise/hare: `slow = f(slow); fast = f(f(fast))`; if they meet at 1, return true; if they meet at anything else, return false.

### Pattern 6: Carry-propagation arrays

*Plus One*: iterate the digits right-to-left, add 1 plus carry, take mod 10. If the carry survives past index 0, prepend a 1. The same shape generalises to *Add to Array Form* and *Add Strings*.

### Pattern 7: Schoolbook multiplication of big numbers

*Multiply Strings*: result of `m × n` digits has at most `m + n` digits. Initialise `res = [0] * (m + n)`. For each `i` in `num1` (right-to-left) and `j` in `num2` (right-to-left), add `(d_i * d_j)` into position `i + j + 1` and let the carry flow into `i + j`. Strip leading zeros at the end. No `int()` cheating.

### Pattern 8: Geometry by coordinate hashing

*Detect Squares*: store points in a `Counter[(x, y)]`. To count squares with one corner at query `(qx, qy)`, iterate over all stored `(x, y)` that share the diagonal (`abs(x - qx) == abs(y - qy)` and both nonzero); the other two corners are then determined, and the contribution is the product of their counts.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Rotate Image | Medium |
| 2 | Spiral Matrix | Medium |
| 3 | Set Matrix Zeroes | Medium |
| 4 | Happy Number | Easy |
| 5 | Plus One | Easy |
| 6 | Pow(x, n) | Medium |
| 7 | Multiply Strings | Medium |
| 8 | Detect Squares | Medium |

## Tips

- **Draw a tiny example before coding.** For *Rotate Image*, a `3×3` matrix on paper makes the transpose-then-reverse insight obvious; for *Spiral Matrix*, a `3×4` matrix surfaces the "remaining row/column" edge cases.
- **Watch the negative-power edge case in *Pow(x, n)*.** `-INT_MIN` overflows in C/Java; in Python it's safe, but mention the issue. The clean fix is `if n < 0: x, n = 1/x, -n`.
- **For *Set Matrix Zeroes*, decide first-row / first-column flags first, then proceed.** Otherwise you'll overwrite your own markers before reading them.
- **For *Multiply Strings*, position `i + j + 1` for the units digit of `d_i * d_j` and `i + j` for the tens digit.** Off-by-one here is the entire failure mode.
- **For *Happy Number*, the hash-set alternative is shorter than Floyd's** — just record every number you've seen, return false on a revisit. Floyd's is the O(1)-space version, mention it as the follow-up.

## Linked concepts

Math & Geometry problems mostly stand alone — they aren't tied to a specific data structure on the design side. The closest matches on this site are:

- [Tic Tac Toe (DSA Design)](../../dsa-design/problems/tic-tac-toe.md) — grid-state tracking with row / col / diagonal counters, the same indexing discipline as *Rotate Image*.
- [Minesweeper (DSA Design)](../../dsa-design/problems/minesweeper.md) — grid DFS with neighbour-counting, another matrix-indexing exercise.
- [Snake Game (DSA Design)](../../dsa-design/problems/snake-game.md) — coordinate-tracking on a grid with hash-set occupancy, conceptually similar to *Detect Squares*.

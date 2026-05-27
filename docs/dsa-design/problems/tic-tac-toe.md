# Design Tic-Tac-Toe

!!! note "The disguise"
    Interviewer says: **"Design Tic-Tac-Toe with an efficient `move(row, col, player)` win check"**
    They mean: **"Replace the O(n) board scan with O(1) per-line counters"**

## Problem

On an `n × n` board, support `move(row, col, player) -> 0 | 1 | 2`. Return the winner if this move ends the game, else `0`.

## What it really tests

- The obvious solution rescans rows, columns, and diagonals after every move → O(n) per move.
- Insight: a player wins a line iff *all n* of its cells are theirs. Equivalently, the **net count** along that line is `±n` (`+1` per move by player 1, `-1` per move by player 2).
- Maintain four counters: per row, per column, the main diagonal, and the anti-diagonal. Each move updates at most four counters — O(1).

## Approach

- `rows[n]`, `cols[n]`, `diag`, `anti` all start at 0.
- On `move(r, c, p)`, add `delta = +1 if p == 1 else -1` to:
    - `rows[r]`, `cols[c]`.
    - `diag` if `r == c`.
    - `anti` if `r + c == n - 1`.
- After the update, if any of the four affected counters has absolute value `n`, return `p`.

Why does this work? A line has exactly n cells; the only way the running sum hits `±n` is if all n cells belong to the same player and the other never played there. Because the rules forbid overwriting cells, a counter cannot be inflated by repeated moves on the same square.

## Code sketch

```python
class TicTacToe:
    def __init__(self, n: int):
        self.n = n
        self.rows = [0] * n
        self.cols = [0] * n
        self.diag = 0
        self.anti = 0

    def move(self, row: int, col: int, player: int) -> int:
        d = 1 if player == 1 else -1
        self.rows[row] += d
        self.cols[col] += d
        if row == col:
            self.diag += d
        if row + col == self.n - 1:
            self.anti += d

        if (abs(self.rows[row]) == self.n
                or abs(self.cols[col]) == self.n
                or abs(self.diag) == self.n
                or abs(self.anti) == self.n):
            return player
        return 0
```

## Complexity

- `move`: O(1).
- Space: O(n).

## Edge cases

- `n == 1`: any move wins.
- Illegal move (same cell twice) — the problem usually rules this out; otherwise add a `seen` set.
- Draw (no winner after `n²` moves) — returns 0; the caller tracks the move count.

## Linked concepts

- [Game & Simulation problems](../by-category/game-simulation.md)
- The counter pattern shows up again in Sudoku validators and constraint solvers.

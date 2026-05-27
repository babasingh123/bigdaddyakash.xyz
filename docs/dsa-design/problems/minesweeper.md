# Design Minesweeper

!!! note "The disguise"
    Interviewer says: **"Implement Minesweeper's reveal-on-click logic"**
    They mean: **"Flood-fill (BFS or DFS) on an 8-connected grid, stopping at numbered cells"**

## Problem

Given a board where `'M'` is a mine, `'E'` is an unrevealed empty cell, `'B'` is a revealed blank, digits show adjacent-mine counts, and `'X'` is exploded, implement `reveal(board, click)`:

- Clicking a mine → reveal it as `'X'`.
- Clicking an empty cell with no neighboring mines → reveal as `'B'` and recursively reveal all 8 neighbors.
- Clicking an empty cell with `k ≥ 1` neighboring mines → reveal as the digit `str(k)` and stop.

## What it really tests

- Recognizing the recursive reveal as **flood fill** with a special stopping rule (numbered cells terminate the cascade).
- 8-directional traversal — the standard `dx, dy ∈ {-1, 0, 1}` minus `(0, 0)`.
- Boundary and visited handling so the algorithm doesn't blow the stack on a large empty board.

## Approach

DFS or BFS — both work; BFS uses an explicit queue and avoids stack-overflow risk on big boards.

For each popped cell:

1. If already revealed, skip.
2. Count neighboring mines.
3. If count > 0 → write the digit. Done with this cell.
4. If count == 0 → write `'B'` and enqueue all 8 neighbors that are still `'E'`.

The "stop at numbered cells" rule keeps the flood from leaking through a thin border of digits, which would otherwise reveal the entire board.

## Code sketch

```python
from collections import deque

DIRS = [(-1, -1), (-1, 0), (-1, 1),
        (0, -1),           (0, 1),
        (1, -1),  (1, 0),  (1, 1)]

def reveal(board: list[list[str]], click: list[int]) -> list[list[str]]:
    R, C = len(board), len(board[0])
    r, c = click
    if board[r][c] == "M":
        board[r][c] = "X"
        return board

    q = deque([(r, c)])
    while q:
        i, j = q.popleft()
        if board[i][j] != "E":
            continue
        mines = 0
        neighbors = []
        for di, dj in DIRS:
            ni, nj = i + di, j + dj
            if 0 <= ni < R and 0 <= nj < C:
                if board[ni][nj] == "M":
                    mines += 1
                elif board[ni][nj] == "E":
                    neighbors.append((ni, nj))
        if mines > 0:
            board[i][j] = str(mines)
        else:
            board[i][j] = "B"
            q.extend(neighbors)
    return board
```

## Complexity

- Time: O(R · C) — each cell processed at most once.
- Space: O(R · C) for the queue in the worst case (an all-empty board).

## Edge cases

- First click is a mine → immediate `'X'`, no cascade.
- Click on an already revealed cell → no-op (the early `continue` handles it).
- 1×1 board with no mines → cell becomes `'B'`.
- Recursion depth: prefer BFS on large boards to avoid Python's recursion limit.

## Linked concepts

- [Game & Simulation problems](../by-category/game-simulation.md)
- Flood-fill is the same engine behind [Number of Islands](https://leetcode.com/problems/number-of-islands/).

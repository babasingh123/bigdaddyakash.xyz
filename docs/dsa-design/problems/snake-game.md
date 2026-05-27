# Design Snake Game

!!! note "The disguise"
    Interviewer says: **"Design the classic Snake game"**
    They mean: **"Use a deque for the body and a HashSet for O(1) collision detection"**

## Problem

Build a Snake game on a `width × height` board:

- Snake starts at `(0, 0)`, length 1.
- A queue of food positions appears one at a time.
- `move(direction)` — move the snake one cell. Eat food (grow + advance food queue), else move (advance head, drop tail). Return the score, or `-1` on death (wall or self-collision).

## What it really tests

- Why a plain list isn't enough: appending the head and popping the tail are fine, but `in body` is O(n) — slow for self-collision checks.
- Combining structures:
    - **Deque** for the ordered body — O(1) head push, O(1) tail pop.
    - **Set** of occupied cells — O(1) collision check.
- Keeping the two views in sync.

## Approach

State:

- `body: deque[(r, c)]` — ordering.
- `occupied: set[(r, c)]` — membership.
- `food: list[(r, c)]` with an index pointer.
- Score = food eaten so far.

Each `move`:

1. Compute the new head from direction deltas.
2. Wall check: out of bounds → death.
3. Did we eat? If `(head == food[idx])`, advance the food pointer; do *not* drop the tail.
4. Otherwise, drop the tail from both `body` and `occupied`. (Crucial: drop the tail *before* the self-collision check, so the snake can move into the spot it's vacating.)
5. Self-collision: if `new_head in occupied`, death.
6. Otherwise add the new head to both structures and return the score.

## Code sketch

```python
from collections import deque

class SnakeGame:
    DIRS = {"U": (-1, 0), "D": (1, 0), "L": (0, -1), "R": (0, 1)}

    def __init__(self, width: int, height: int, food: list[list[int]]):
        self.W, self.H = width, height
        self.food = food
        self.fi = 0
        self.body = deque([(0, 0)])
        self.occupied = {(0, 0)}
        self.score = 0

    def move(self, direction: str) -> int:
        dr, dc = self.DIRS[direction]
        r, c = self.body[0]
        nr, nc = r + dr, c + dc

        if not (0 <= nr < self.H and 0 <= nc < self.W):
            return -1

        if self.fi < len(self.food) and [nr, nc] == self.food[self.fi]:
            self.fi += 1
            self.score += 1
        else:
            tail = self.body.pop()
            self.occupied.discard(tail)

        if (nr, nc) in self.occupied:
            return -1

        self.body.appendleft((nr, nc))
        self.occupied.add((nr, nc))
        return self.score
```

## Complexity

- `move`: O(1).
- Space: O(snake length).

## Edge cases

- Snake moves into the cell its own tail just vacated → legal (tail removed first).
- Food at the same cell as the head at game start → eaten on the first move into that direction.
- No more food in the queue → keep moving without growing.
- 1×1 board → the only "legal" move is into a wall ⇒ instant death.

## Linked concepts

- [Game & Simulation problems](../by-category/game-simulation.md)
- The deque-plus-set duo is reused for any "ordered + fast membership" requirement — LRU cache is the classic.

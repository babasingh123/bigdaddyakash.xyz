# Game & Simulation

"Design game X" is almost never about the game. It's about **picking the right structure to simulate state cheaply**. Most often: a 2D array, a deque, a hash set, or a clever counter.

## The pattern

Every simulation problem has the same shape:

1. Some state on a grid or sequence.
2. A `move(action)` function that mutates the state.
3. A query (is the game over? did anyone win? what's the score?).

The pattern is **pick the structure that makes `move` and `query` O(1) or O(grid_perimeter)** rather than O(grid_area).

- **Snake** → Deque for the body (head = newest, tail = oldest) + HashSet for O(1) collision check. Don't scan the body each tick.
- **Tic-Tac-Toe** → Per-row, per-col, and two diagonal counters. Each move updates 4 counters; a win is detected when any counter hits `n`. Beats O(n²) scan.
- **2048** → 2D array + sliding-merge per row. Rotate the grid for the other directions.
- **Minesweeper** → DFS/BFS flood fill from the click, expanding through zeros.
- **Card Shuffle** → Fisher-Yates in place, O(n).

The lesson: the *obvious* representation (a full grid you scan every tick) is the wrong one. The right one buys O(1) per operation with a small auxiliary structure.

## Problems

- [Snake Game](../problems/snake-game.md) — Deque + HashSet
- [Tic-Tac-Toe](../problems/tic-tac-toe.md) — Row/col/diag counters (O(1) win check)
- [2048 Game](../problems/2048-game.md) — 2D array + slide/merge + rotation
- [Minesweeper](../problems/minesweeper.md) — DFS/BFS flood fill on 2D grid
- [Card Shuffle](../problems/card-shuffle.md) — Fisher-Yates algorithm

## Why interviewers love this category

Game problems are an excellent test of **representation skill**. They're not algorithmically deep (no Dijkstra, no DP), but they punish lazy data-structure choices brutally.

Interviewers look for:

1. **Refusal to brute-force.** The candidate who scans the whole grid every move for Tic-Tac-Toe has shown they're optimizing for *writing time*, not *running time*. A counter array is barely more code, and it's O(1) instead of O(n²).
2. **Knowing Fisher-Yates.** Card shuffle is a gimme — but only if you know the algorithm. The candidate who tries `random.sample` is fine; the one who *explains why* Fisher-Yates is uniformly random has done the homework.
3. **Deque-for-snake instinct.** Snake game tests whether you recognize "head + tail with O(1) access at both ends" → Deque. A list with `pop(0)` is technically correct and O(n) per move.
4. **Boundary handling.** Minesweeper's 8-directional traversal is where bugs hide. The clean version uses a `directions` array and a single bounds check; the messy version unrolls 8 if-statements.

These problems are deliberately concrete and visual — they're easy to picture, but the *implementation has to be right*. That's what interviewers grade.

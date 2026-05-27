# Design a Chess Game

> **Time:** 60+ minutes

## Problem

Design the engine for a two-player chess game. The system must:

- Maintain an **8×8 board** with the standard initial position.
- Support all **six piece types**: King, Queen, Rook, Bishop, Knight, Pawn.
- **Validate moves** according to each piece's movement rules.
- Detect **check** and **checkmate**.
- Support special moves: **castling**, **en passant**, **pawn promotion**.
- Track **game state**: `IN_PROGRESS`, `CHECK`, `CHECKMATE`, `STALEMATE`, `DRAW`.
- Maintain **move history** (enables undo/redo and PGN export).

## Real-world intuition

Chess is the classic "lots of types with shared structure" LLD problem. There are six pieces with different movement rules — that screams polymorphism. There are special moves that depend on board *history* (en passant) or piece *history* (castling requires neither piece to have moved) — that demands a thoughtful `Move` representation. And the game cycles through clearly defined states — natural fit for state-driven logic.

## Key classes

- `ChessGame` — top-level orchestrator; owns board, players, move history, current state.
- `Board` — 8×8 grid of `Spot`s; provides `getPiece(row, col)`, `applyMove`, `clone()` for hypothetical positions.
- `Spot` — one square: coordinates and the (possibly null) piece on it.
- `Piece` (abstract) — color, hasMoved flag, `isValidMove(Board, Move)` polymorphic method.
    - `King`, `Queen`, `Rook`, `Bishop`, `Knight`, `Pawn` — concrete subclasses.
- `Move` — start `Spot`, end `Spot`, piece moved, optional captured piece, flags (`isCastling`, `isEnPassant`, `promotionPiece`).
- `Player` — color, name, captured pieces.
- `GameStatus` (enum) — drives end-of-game detection.
- `MoveValidator` — answers "would this move leave my own king in check?" by playing the move on a board clone.

## Design patterns used

- **Strategy (per-piece move rules via polymorphism)** — each piece has its own valid-move algorithm. Putting them on the `Piece` subclasses is textbook polymorphism, and the cleanest implementation of Strategy here.
- **Command (Move)** — every move is an object with `execute` and `undo`. This gives you free undo/redo, replay, and the ability to test "what if?" positions cheaply.
- **State (GameStatus)** — `IN_PROGRESS` accepts moves normally, `CHECK` requires the next move to escape check, `CHECKMATE`/`STALEMATE`/`DRAW` reject all moves. Clean state-driven dispatch.
- **Factory (PieceFactory)** — pawn promotion and game setup both create pieces. A factory centralizes the mapping from `(type, color)` to a fresh `Piece`.

## Class diagram (sketch)

```text
              ┌─────────────┐
              │ ChessGame   │
              ├─────────────┤
              │ -board      │
              │ -players[2] │
              │ -history    │
              │ -status     │
              │ +makeMove(m)│
              └──────┬──────┘
                     ◆
              ┌──────▼──────┐
              │   Board     │
              ├─────────────┤
              │ -spots[8][8]│
              └──────┬──────┘
                     ◆
              ┌──────▼─────┐
              │   Spot     │
              ├────────────┤
              │ -row, -col │
              │ -piece     │
              └────────────┘
                     │
                     ▼ may contain
              ┌─────────────┐
              │   Piece     │ «abstract»
              ├─────────────┤
              │ -color      │
              │ -hasMoved   │
              │ +isValidMove│
              └──────△──────┘
       ┌─────┬──────┼──────┬──────┬──────┐
      King  Queen  Rook  Bishop Knight  Pawn

      ChessGame ──◇── Move «command»  ◇── history (Deque)
```

## Code sketch

```java
enum Color { WHITE, BLACK }
enum GameStatus { IN_PROGRESS, CHECK, CHECKMATE, STALEMATE, DRAW }

abstract class Piece {
    protected final Color color;
    protected boolean hasMoved = false;

    protected Piece(Color c) { this.color = c; }

    public abstract boolean canMove(Board board, Spot from, Spot to);

    public Color color() { return color; }
}

class Knight extends Piece {
    public Knight(Color c) { super(c); }

    public boolean canMove(Board board, Spot from, Spot to) {
        // Knights ignore blockers — pure L-shape check.
        int dr = Math.abs(from.row() - to.row());
        int dc = Math.abs(from.col() - to.col());
        boolean lShape = (dr == 2 && dc == 1) || (dr == 1 && dc == 2);
        if (!lShape) return false;
        Piece target = to.piece();
        return target == null || target.color() != this.color;
    }
}

class Pawn extends Piece {
    public Pawn(Color c) { super(c); }
    public boolean canMove(Board b, Spot from, Spot to) {
        // Direction, single-step, double-step from start, diagonal capture, en passant.
        return false;
    }
}

class Move {                                              // Command
    private final Spot from, to;
    private final Piece moved;
    private Piece captured;
    private boolean isCastling, isEnPassant;
    private Piece promotion;

    public void execute(Board b) {
        captured = to.piece();
        to.setPiece(moved);
        from.setPiece(null);
        moved.hasMoved = true;
        // handle castling/en passant/promotion as flagged
    }

    public void undo(Board b) {
        from.setPiece(moved);
        to.setPiece(captured);
        // reverse special-move side effects, reset hasMoved if first move
    }
}

class ChessGame {
    private final Board board = new Board();
    private final Player[] players = { new Player(Color.WHITE), new Player(Color.BLACK) };
    private final Deque<Move> history = new ArrayDeque<>();
    private GameStatus status = GameStatus.IN_PROGRESS;
    private int turn = 0;

    public boolean makeMove(Move m) {
        if (status == GameStatus.CHECKMATE || status == GameStatus.STALEMATE) return false;

        Piece p = m.from().piece();
        if (p == null || p.color() != players[turn].color()) return false;
        if (!p.canMove(board, m.from(), m.to())) return false;

        m.execute(board);
        if (leavesOwnKingInCheck(p.color())) {        // illegal: must not expose your own king
            m.undo(board);
            return false;
        }

        history.push(m);
        turn = 1 - turn;
        recomputeStatus();
        return true;
    }

    public void undo() {
        if (history.isEmpty()) return;
        history.pop().undo(board);
        turn = 1 - turn;
        recomputeStatus();
    }

    private boolean leavesOwnKingInCheck(Color c) { /* ... */ return false; }
    private void recomputeStatus() { /* check, checkmate, stalemate, draw */ }
}
```

## Key design decisions

1. **Move rules live on `Piece` subclasses** — polymorphism replaces a giant `switch (pieceType)`. Adding a fairy chess piece is a new class, nothing else changes.
2. **`Move` is a Command, not a tuple** — encapsulating execute/undo on a Move gives you free undo, makes "would this move leave me in check?" trivially implementable (apply, test, undo), and enables PGN export.
3. **Board clone for hypotheticals** — checkmate detection needs to ask *"can the current player make any move that escapes check?"* The cleanest way is to clone the board, try each candidate move, see if the king is still attacked. Make `Board` and `Piece` cheaply cloneable.
4. **Status as enum, transitions in `recomputeStatus`** — status is *derived* from the board after every move. Keeping it derived (rather than mutated incrementally) eliminates a whole class of bugs where state and reality drift apart.
5. **`hasMoved` on pieces** — needed for castling (neither piece has moved) and pawn double-step (first move only). Sits naturally on the piece; the `Move.undo` must restore it accurately if it was that piece's first move.
6. **Special moves as flags on `Move`** — `isCastling`, `isEnPassant`, `promotionPiece`. Keeps the `Move` representation uniform; execute/undo dispatch on the flags. Avoids subclassing `Move` for each special case.

## Class hierarchy: pieces

```text
              Piece (abstract)
                  △
   ┌────┬─────┬──┴───┬──────┬──────┐
  King Queen Rook  Bishop  Knight  Pawn
```

Each defines `canMove`. `Queen` is essentially `Rook | Bishop` — many implementations compose it from those two checks (a small composition-over-inheritance win).

## Extensibility

**Add a new piece (e.g., fairy chess "Archbishop" = Bishop + Knight):**

1. Create `class Archbishop extends Piece` with a `canMove` that ORs Bishop's and Knight's rules.
2. Update `PieceFactory` to know the new piece code.
3. Nothing else changes.

**Add a clock / time control:**

1. Add a `Clock` per player; consult it in `makeMove` before validation.
2. `ChessGame.status` gains a `TIMEOUT` value.

**Support PGN import/export:**

1. Add `PgnSerializer` that walks `history`.
2. Each `Move` carries the data needed (start, end, captured, special flags) — already there.

**Add AI opponent:**

1. AI is a `Player` subclass (or a separate `Engine` that picks moves and calls `game.makeMove`).
2. Engine cheaply tries moves via `Move.execute` / `undo` on the real board (or clones), evaluating each.

## Linked concepts

- [OOP](../fundamentals/oop.md) — abstract `Piece` with six subclasses is the canonical polymorphism example.
- [SOLID](../fundamentals/solid.md) — OCP (new piece types), SRP (`Move` vs `Board` vs `ChessGame`).
- [Design Patterns](../fundamentals/design-patterns.md) — Command (Move), State (GameStatus), Strategy (piece rules), Factory (piece creation).

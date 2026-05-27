# Design a Snake and Ladder Game

> **Time:** 30–40 minutes

## Problem

Design the game engine for Snakes and Ladders. The system must:

- Use a **100-cell board** numbered 1 to 100.
- Place **snakes** (head > tail, slide down on landing on head) and **ladders** (bottom < top, climb up on landing on bottom).
- Support **multiple players**, each starting at position 0.
- Roll a **dice** (1–6) on each turn.
- Move the player; apply snake/ladder if applicable.
- Determine the **winner** (first to reach 100). Optionally: require exact landing on 100.

## Real-world intuition

Snake and Ladder is one of the simplest LLD problems on the list. There's no clever pattern needed — the whole point is to demonstrate clean **object-oriented modeling** of a small domain. The interviewer wants to see: do you reach for a `Map<Integer, Integer> snakes` or do you model `Snake` and `Ladder` as proper objects? Are your classes cohesive? Do your responsibilities make sense?

## Key classes

- `Game` — orchestrates the turn cycle; owns board, players, dice; tracks current player and winner.
- `Board` — knows the cells, the snakes, and the ladders; computes "where do I end up if I land on cell X?"
- `Cell` — a board position (optional — could be implicit by integer).
- `Snake` — head position, tail position. `Snake.apply(playerPos)` returns the tail if `playerPos == head`.
- `Ladder` — bottom position, top position.
- `Jump` (optional abstraction) — common parent for `Snake` and `Ladder`, since both are "if you land on X, go to Y".
- `Player` — name, current position, score.
- `Dice` — rolls 1–6; supports 1-die or N-dice variants.

## Design patterns used

This problem is intentionally pattern-light. You can solve it cleanly without invoking any heavyweight pattern. That said, two are worth noting:

- **Strategy (DiceStrategy)** — if you want to support 1 die, 2 dice, biased dice for AI opponents — that's Strategy. Otherwise a plain `Dice` class is fine.
- **Polymorphism over `Snake` and `Ladder` via a common `Jump`** — collapses board lookup into a single `Map<Integer, Jump>` instead of two maps. Light-touch use of OOP.

## Class diagram (sketch)

```text
   ┌─────────┐
   │  Game   │
   ├─────────┤
   │ -board  │◇──▶ Board
   │ -players│◇──▶ Player (queue)
   │ -dice   │◇──▶ Dice
   │ -winner │
   │ +play() │
   └─────────┘

   ┌──────────┐
   │  Board   │
   ├──────────┤
   │ -size    │
   │ -jumps   │  Map<Integer, Jump>
   │ +next(p,roll) │
   └──────────┘

   ┌──────────┐  «abstract»
   │  Jump    │   start, end
   └────△─────┘
        │
   ┌────┴──────┬───────────┐
   │           │           │
 Snake       Ladder      (extension)
```

## Code sketch

```java
abstract class Jump {
    protected final int start;
    protected final int end;
    protected Jump(int start, int end) { this.start = start; this.end = end; }
    public int start() { return start; }
    public int end()   { return end; }
}

class Snake extends Jump {
    public Snake(int head, int tail) {
        super(head, tail);
        if (tail >= head) throw new IllegalArgumentException("Snake must go down");
    }
}

class Ladder extends Jump {
    public Ladder(int bottom, int top) {
        super(bottom, top);
        if (top <= bottom) throw new IllegalArgumentException("Ladder must go up");
    }
}

class Board {
    private final int size;
    private final Map<Integer, Jump> jumps = new HashMap<>();

    public Board(int size) { this.size = size; }

    public void addJump(Jump j) {
        if (jumps.containsKey(j.start())) throw new IllegalArgumentException("Conflicting jump at " + j.start());
        jumps.put(j.start(), j);
    }

    public int nextPosition(int from, int roll, boolean exactLanding) {
        int dest = from + roll;
        if (dest > size) {
            return exactLanding ? from : size;   // bounce-back rule optional
        }
        Jump j = jumps.get(dest);
        return j != null ? j.end() : dest;
    }
}

class Player {
    private final String name;
    private int position = 0;
    public void moveTo(int pos) { this.position = pos; }
}

class Dice {
    private final Random rng = new Random();
    private final int faces;
    public Dice(int faces) { this.faces = faces; }
    public int roll() { return 1 + rng.nextInt(faces); }
}

class Game {
    private final Board board;
    private final Deque<Player> players = new ArrayDeque<>();
    private final Dice dice;
    private Player winner;

    public Game(Board board, List<Player> players, Dice dice) {
        this.board = board;
        this.players.addAll(players);
        this.dice = dice;
    }

    public Player play() {
        while (winner == null) {
            Player current = players.poll();
            int roll = dice.roll();
            int next = board.nextPosition(current.position(), roll, false);
            current.moveTo(next);
            if (next == board.size()) {
                winner = current;
            } else {
                players.offer(current);
            }
        }
        return winner;
    }
}
```

## Key design decisions

1. **`Snake` and `Ladder` as objects, not raw map entries** — the temptation is `Map<Integer, Integer> snakes`. Modeling them as classes gives you (a) validation at construction (a "Snake" must go down), (b) extensibility (e.g., a `MagicJump` later), and (c) a place to attach metadata (was this jump a snake or ladder? — needed for UI).
2. **Common `Jump` parent** — board lookup becomes one `Map<Integer, Jump>`. Without it, the board would maintain two maps and check both.
3. **`Board.nextPosition` is the pure function at the heart of the system** — given a starting position and a roll, returns the destination. Everything else (game loop, players, UI) is plumbing around this. Keeping it pure makes it trivially testable.
4. **Board size is parameterized** — the standard board is 100, but 10×10 isn't the only valid version. A `size` parameter makes 4×4 test boards possible.
5. **Player queue, not array index** — using a `Deque` and rotating on each turn handles arbitrary player counts naturally. No `currentPlayerIdx = (idx + 1) % n` arithmetic.
6. **No exact landing by default; toggle via flag** — the rules vary. A flag exposes the variation without two `Board` classes.

## Extensibility

**Add a different board (e.g., 10×10 with custom jumps):**

1. Construct a `Board(100)`, call `addJump` for each snake and ladder.
2. No code changes.

**Add biased dice:**

1. `Dice` becomes an interface or has a strategy.
2. `LoadedDice implements Dice` with a custom probability distribution.

**Add power-ups (cell types that grant extra turn / shield from snake):**

1. Introduce a `Cell` class with optional behavior on landing.
2. The board returns the affected cell; the game loop applies its effect.

**Add an exact-landing rule:**

1. Already supported via the `exactLanding` flag — flip it on game construction.

**Add multiple dice:**

1. `MultiDice implements Dice` returning the sum of N rolls.

## Linked concepts

- [OOP](../fundamentals/oop.md) — the whole point of this problem is to demonstrate clean OOP modeling on a tiny domain.
- [SOLID](../fundamentals/solid.md) — SRP across `Game`, `Board`, `Player`, `Dice`.
- [Design Patterns](../fundamentals/design-patterns.md) — minimal here; light Strategy for `Dice` is the most you'd reach for.

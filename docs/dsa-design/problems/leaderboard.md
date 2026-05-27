# Design Leaderboard

!!! note "The disguise"
    Interviewer says: **"Design a gaming leaderboard with top K players."**
    They mean: **"Implement a HashMap (player → score) plus a TreeMap (score → set of players) for O(log n) updates and ordered retrieval."**

## Problem

Build `Leaderboard` with:

- `addScore(playerId, score)` — add `score` to the player's existing total (create if new).
- `top(K)` — return the sum of the top K scores.
- `reset(playerId)` — set the player's score to 0 (effectively remove from rankings).

## What it really tests

- **Two-structure composition** — HashMap for O(1) "what's this player's score?" + TreeMap for O(log n) "what's the highest score?".
- **Score → set of players** mapping, because multiple players can tie.
- **Top-K traversal** — iterate descending over the TreeMap, accumulating K players.

## Approach

Naive: `dict[player, score]` + sort on each `top` call → O(n log n) per query.

That's fine for small leaderboards. The interview test is whether you can do better.

**Two indexes:**

- `score_of: dict[player, int]` — O(1) lookup of a player's current score.
- `players_at_score: SortedDict[int, set[player]]` — sorted by score, with player sets for ties.

**`addScore(player, delta)`** flow:

1. Read `old = score_of.get(player, 0)`. Remove player from `players_at_score[old]` (if `old > 0`); if the set becomes empty, delete the entry.
2. Compute `new = old + delta`. Add player to `players_at_score[new]` (creating the entry if needed).
3. Update `score_of[player] = new`.

**`top(K)`** flow:

Iterate from the largest score downward in the TreeMap. For each score, count players in the set; if you can take all (running count + set.size ≤ K), accumulate `score × set.size`; otherwise, take the remaining K and `break`.

**`reset(player)`** is just `addScore(player, -score_of[player])` followed by `del score_of[player]`.

**Why this works:** every operation is bounded by the TreeMap's height (O(log n)) plus a small amount of constant work per score level.

**Alternative for huge leaderboards (millions of players, top 100 only):** keep the data in Redis sorted sets — `ZADD`, `ZINCRBY`, `ZREVRANGE` give you the same semantics with persistence. The algorithmic structure is identical.

## Code sketch

```python
from sortedcontainers import SortedDict
from collections import defaultdict

class Leaderboard:
    def __init__(self):
        self.score_of: dict[int, int] = {}
        self.players_at: SortedDict[int, set[int]] = SortedDict()

    def addScore(self, player_id: int, score: int) -> None:
        old = self.score_of.get(player_id, 0)
        if old in self.players_at and player_id in self.players_at[old]:
            self.players_at[old].discard(player_id)
            if not self.players_at[old]:
                del self.players_at[old]
        new = old + score
        self.score_of[player_id] = new
        if new not in self.players_at:
            self.players_at[new] = set()
        self.players_at[new].add(player_id)

    def top(self, K: int) -> int:
        total = 0
        remaining = K
        for score in reversed(self.players_at):
            count = len(self.players_at[score])
            take = min(count, remaining)
            total += score * take
            remaining -= take
            if remaining == 0:
                break
        return total

    def reset(self, player_id: int) -> None:
        if player_id not in self.score_of:
            return
        old = self.score_of.pop(player_id)
        self.players_at[old].discard(player_id)
        if not self.players_at[old]:
            del self.players_at[old]
```

## Complexity

Let *n* = active players, *s* = distinct score values.

- `addScore`: **O(log s)** — TreeMap insert/delete.
- `top(K)`: **O(log s + K)** in the worst case (one set per score level, K small). Typically dominated by K when most players have distinct scores.
- `reset`: **O(log s)**.
- Space: **O(n)**.

## Edge cases

- **Player not yet on the board.** `addScore` treats their `old` as 0; this is correct.
- **All players tied.** `players_at` has one entry with set of all players; `top(K)` correctly takes K players from that set, even though all contribute the same score.
- **K larger than number of players.** Return the sum of all scores.
- **Reset on missing player.** No-op.
- **Negative scores.** Allowed by the structure; ensure the TreeMap doesn't have a floor at 0.

## Linked concepts

- [Ranking & Leaderboard category overview](../by-category/ranking-leaderboard.md)
- [Top K Frequent Elements](top-k-frequent.md) — same heap-of-K technique
- [Snapshot Array](snapshot-array.md) — similar HashMap + sorted-dict composition

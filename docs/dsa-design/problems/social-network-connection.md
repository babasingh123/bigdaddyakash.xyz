# Design Social Network Connection

!!! note "The disguise"
    Interviewer says: **"Design a friend recommendation system"**
    They mean: **"Implement BFS on an undirected graph and find nodes at distance 2"**

## Problem

Model a friend graph and support:

- `addFriendship(u, v)` and `removeFriendship(u, v)`.
- `degreesOfSeparation(u, v)` — shortest-path length between two users, or `-1` if none.
- `recommendFriends(u)` — users who are friends-of-friends of `u` but not already direct friends.

## What it really tests

- Modeling an undirected graph as an **adjacency set** (vs list) so that `add`, `remove`, and "is friend?" are all O(1) average.
- BFS layered traversal — distance is implicit in BFS layer number. You do not need Dijkstra here because every edge has weight 1.
- Recognizing that "friends of friends" is just **nodes at BFS depth 2**, excluding depth-0 (self) and depth-1 (already friends).

## Approach

The graph is sparse, undirected, and demands cheap membership checks → `dict[user] -> set[user]`.

Why a set and not a list? `addFriendship` is fine with either, but `removeFriendship` would be O(deg) with a list. A set keeps every operation O(1) average and dedupes accidental double-adds.

For shortest path, BFS is correct because every edge costs 1. Dijkstra would be overkill (an extra `log V`).

For friend recommendations: BFS one layer past the user, collecting visited nodes at depth 2. Don't recommend the user themselves or their existing friends.

Bonus heuristic: count the number of mutual friends as a ranking score for recommendations — this is `len(friends_of_u ∩ friends_of_candidate)`.

## Code sketch

```python
from collections import defaultdict, deque, Counter

class SocialNetwork:
    def __init__(self):
        self.adj = defaultdict(set)

    def addFriendship(self, u: int, v: int) -> None:
        self.adj[u].add(v)
        self.adj[v].add(u)

    def removeFriendship(self, u: int, v: int) -> None:
        self.adj[u].discard(v)
        self.adj[v].discard(u)

    def degreesOfSeparation(self, u: int, v: int) -> int:
        if u == v:
            return 0
        visited = {u}
        q = deque([(u, 0)])
        while q:
            node, d = q.popleft()
            for nbr in self.adj[node]:
                if nbr == v:
                    return d + 1
                if nbr not in visited:
                    visited.add(nbr)
                    q.append((nbr, d + 1))
        return -1

    def recommendFriends(self, u: int, top_k: int = 5) -> list[int]:
        direct = self.adj[u]
        score = Counter()
        for friend in direct:
            for candidate in self.adj[friend]:
                if candidate == u or candidate in direct:
                    continue
                score[candidate] += 1   # one more mutual friend
        return [c for c, _ in score.most_common(top_k)]
```

## Complexity

- `addFriendship` / `removeFriendship`: O(1) average.
- `degreesOfSeparation`: O(V + E) worst case.
- `recommendFriends`: O(deg(u) · avg_deg) — cheap for typical social graphs.

## Edge cases

- Self-friendship: skip it.
- Adding the same friendship twice: the set dedupes, no double counting.
- Isolated user → empty recommendation list.
- Very high-degree "celebrity" nodes can blow up BFS; in production, cap traversal depth.

## Linked concepts

- [Graph & Network problems](../by-category/graph-network.md)
- BFS on an unweighted graph is also the spine of [Word Ladder](./word-ladder.md).
- For partitioning a friend graph across machines see [Distributed Systems fundamentals](../../hld/fundamentals/distributed-systems.md).

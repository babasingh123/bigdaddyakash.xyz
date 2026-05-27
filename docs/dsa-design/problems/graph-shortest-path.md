# Design Graph Shortest Path

!!! note "The disguise"
    Interviewer says: **"Design a graph that can find the shortest path between two nodes"**
    They mean: **"Implement Dijkstra's algorithm (or BFS for unweighted graphs)"**

## Problem

Build a directed graph that supports:

- `addEdge(u, v, w)` — add an edge from `u` to `v` with non-negative weight `w`.
- `shortestPath(src, dst)` — length of the shortest path from `src` to `dst`, or `-1` if unreachable.

Nothing here is about distributed nodes or partitioning. It is the textbook shortest-path problem.

## What it really tests

- Picking a sensible graph representation. For sparse graphs — i.e., most real ones — an **adjacency list** beats an adjacency matrix on both memory and traversal cost.
- Knowing which algorithm fits the weights:
    - All weights equal? **BFS** is O(V + E).
    - Non-negative weights? **Dijkstra** with a min-heap is O((V + E) log V).
    - Negative weights? **Bellman-Ford** in O(V · E).
- Implementing Dijkstra correctly. Lazy deletion (skip stale heap entries) is the usual gotcha.

## Approach

A `dict[u] -> list[(v, w)]` adjacency list is the right structure: O(1) insertion, O(deg(u)) iteration, no wasted space on absent edges. Use a matrix only when graphs are dense (E ≈ V²) or you need O(1) edge-weight lookups. For an interviewer's "design" prompt, assume sparse.

For shortest paths we use Dijkstra:

1. Maintain a `dist[node]` table seeded with `dist[src] = 0`.
2. Push `(0, src)` onto a min-heap keyed by current distance.
3. Pop the smallest. If it's stale (heap distance > current best), skip it. Otherwise relax each outgoing edge.

Why a heap and not a sorted list? Heap push/pop is O(log V); maintaining a sorted list is O(V).

Why "lazy deletion" instead of decrease-key? Python's `heapq` has no decrease-key — pushing a duplicate and skipping stale entries when popped is the idiomatic pattern and has the same asymptotic complexity.

## Code sketch

```python
import heapq
from collections import defaultdict

class Graph:
    def __init__(self):
        self.adj = defaultdict(list)  # u -> [(v, w), ...]

    def addEdge(self, u: int, v: int, w: int) -> None:
        self.adj[u].append((v, w))

    def shortestPath(self, src: int, dst: int) -> int:
        dist = {src: 0}
        pq = [(0, src)]  # (distance, node)
        while pq:
            d, u = heapq.heappop(pq)
            if u == dst:
                return d
            if d > dist[u]:           # stale entry
                continue
            for v, w in self.adj[u]:
                nd = d + w
                if nd < dist.get(v, float("inf")):
                    dist[v] = nd
                    heapq.heappush(pq, (nd, v))
        return -1
```

For the unweighted variant swap the heap for a `deque` and you get BFS in O(V + E).

## Complexity

- `addEdge`: O(1).
- `shortestPath`: O((V + E) log V) with a binary heap. Space O(V + E).

## Edge cases

- `src == dst` → return 0.
- Disconnected nodes → return -1.
- Self-loops with weight 0 are harmless; with positive weight Dijkstra correctly ignores them.
- Parallel edges: keep them all; Dijkstra naturally picks the cheapest.
- Negative weights: Dijkstra is wrong — switch to Bellman-Ford.

## Linked concepts

- [Graph & Network problems](../by-category/graph-network.md)
- Same engine as [Network Delay Time](./network-delay-time.md) (single-source over the whole graph) and [Word Ladder](./word-ladder.md) (BFS on an implicit graph).
- For partitioning a real graph across machines, see [Distributed Systems fundamentals](../../hld/fundamentals/distributed-systems.md).

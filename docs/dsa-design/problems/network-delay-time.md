# Design Network Delay Time

!!! note "The disguise"
    Interviewer says: **"Find the time for a signal sent from node K to reach all N nodes"**
    They mean: **"Run Dijkstra from K and return the maximum shortest-path distance"**

## Problem

Given a list of directed, weighted edges `times[i] = (u, v, w)` over `n` nodes labelled `1..n`. A signal starts at node `K`. Return the time it takes for the signal to reach **every** node, or `-1` if some node is unreachable.

## What it really tests

- Classic single-source shortest paths with non-negative weights → **Dijkstra**.
- The reduction: "time to reach all nodes" = `max(dist[i] for i in 1..n)` after running Dijkstra from K.
- Handling unreachable nodes (return `-1`).

## Approach

Why Dijkstra and not BFS? Edges have weights, so BFS gives wrong answers. Why not Bellman-Ford? Weights are non-negative; Dijkstra is faster (O((V + E) log V) vs O(V · E)).

Algorithm:

1. Build an adjacency list `adj[u] = [(v, w), ...]`.
2. `dist = {K: 0}`. Push `(0, K)` onto a min-heap.
3. Pop the smallest. If stale (`d > dist[u]`), skip. Otherwise relax each neighbor.
4. After the loop, `ans = max(dist.values())` if `len(dist) == n`, else `-1`.

### Example trace

```
n = 4, K = 2
edges = [(2,1,1), (2,3,1), (3,4,1)]

Init   dist = {2:0},  pq = [(0,2)]
Pop (0,2): relax → dist = {2:0, 1:1, 3:1}, pq = [(1,1), (1,3)]
Pop (1,1): no outgoing edges.
Pop (1,3): relax → dist = {..., 4:2}, pq = [(2,4)]
Pop (2,4): done. max(dist.values()) = 2.
```

## Code sketch

```python
import heapq
from collections import defaultdict

def networkDelayTime(times: list[list[int]], n: int, k: int) -> int:
    adj = defaultdict(list)
    for u, v, w in times:
        adj[u].append((v, w))

    dist = {k: 0}
    pq = [(0, k)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue
        for v, w in adj[u]:
            nd = d + w
            if nd < dist.get(v, float("inf")):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))

    return max(dist.values()) if len(dist) == n else -1
```

## Complexity

- Time: O((V + E) log V).
- Space: O(V + E).

## Edge cases

- A node unreachable from K → return `-1`.
- `n == 1` → answer is 0 (signal already at the only node).
- Disconnected components.
- Multiple edges between the same pair: take the cheapest one implicitly during relaxation.

## Linked concepts

- [Graph & Network problems](../by-category/graph-network.md)
- Shares its core engine with [Graph Shortest Path](./graph-shortest-path.md).

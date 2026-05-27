# Advanced Graphs

> **6 problems** · prerequisites below

## What this topic teaches

Advanced Graphs is where the four "named algorithms" you should be able to recite from memory live: **Dijkstra** (shortest path in a non-negative-weighted graph), **Prim** and **Kruskal** (minimum spanning tree), and **topological sort with constraints** (used in *Alien Dictionary*). The skill is recognising — under interview pressure — which of those four to reach for, and then writing the textbook implementation cleanly. The implementations themselves are short; the hard part is the *mapping* from problem statement to algorithm.

The Bellman-Ford / modified-BFS variants also show up: *Cheapest Flights Within K Stops* is "Dijkstra with a step-count constraint," which actually wants Bellman-Ford (or a Dijkstra variant where the state is `(node, stops_used)`). *Reconstruct Itinerary* is Hierholzer's algorithm for **Eulerian paths**. Each of these problems is an excuse to introduce one more graph-algorithm primitive; once you know them, future graph problems become "spot the algorithm" exercises.

## Prerequisites

- Dijkstra's (Advanced Algorithms)
- Prim's (Advanced Algorithms)
- Kruskal's (Advanced Algorithms)
- Topological Sort (Advanced Algorithms)

## Core patterns

### Pattern 1: Dijkstra's shortest path (heap-based)

Min-heap of `(distance_so_far, node)`. Pop the closest unsettled node; relax its outgoing edges (push `(d + w, neighbour)` if it beats the recorded distance). The "lazy" version doesn't bother decrease-key — just push duplicates and skip when `distance_so_far > recorded[node]`. *Network Delay Time* is canonical Dijkstra: return `max(dist)` over all nodes (or `-1` if unreachable).

### Pattern 2: Dijkstra on a modified state space

*Swim In Rising Water*: the cost of a path is the **max** elevation on it (not sum); Dijkstra still works because "max-along-path" is monotone — relax with `max(current_cost, grid[r][c])` instead of `current_cost + edge_weight`. The lesson: Dijkstra's correctness depends only on monotonicity, not on additivity.

### Pattern 3: Minimum Spanning Tree (Prim's)

Min-heap of `(weight, node)`. Start from any vertex; repeatedly pop the cheapest edge to an unvisited node; add it to the MST. *Min Cost to Connect All Points* uses Prim's directly on the complete graph of Manhattan distances. O(N² log N) with a heap is fine for the typical N ≤ 1000 constraint.

### Pattern 4: Topological sort with custom ordering

*Alien Dictionary*: build a directed graph where edge `a → b` means "letter `a` comes before letter `b`," derived from comparing consecutive words. Topologically sort. Edge cases that catch people: (a) shared prefixes that are themselves ordered (`["abc", "ab"]` is invalid because `"abc"` should come *after* `"ab"`), (b) characters that appear in no comparison still need to be in the output, (c) cycles → return `""`.

### Pattern 5: Bellman-Ford for constrained shortest path

*Cheapest Flights Within K Stops*: pure Dijkstra over-prunes paths that take detours to use cheaper later edges. Use Bellman-Ford instead: do `K + 1` rounds of edge relaxation on a *copy* of the current distances (so updates in round `r` don't propagate to round `r`). O(K · E).

### Pattern 6: Hierholzer's Eulerian path

*Reconstruct Itinerary*: find an itinerary using every ticket exactly once, in lexicographic order. Build adjacency as a min-heap (so the smallest destination is popped first). DFS: at each node, exhaust outgoing edges by `heappop`, recursing on each. Post-order append the node to the result; reverse at the end. The post-order trick is what makes Eulerian paths work even when DFS hits a dead end before consuming all edges.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Network Delay Time | Medium |
| 2 | Reconstruct Itinerary | Hard |
| 3 | Min Cost to Connect All Points | Medium |
| 4 | Swim In Rising Water | Hard |
| 5 | Alien Dictionary | Hard |
| 6 | Cheapest Flights Within K Stops | Medium |

## Tips

- **Dijkstra doesn't work with negative edges.** If you see negative weights, that's a Bellman-Ford signal. If you see "at most K edges / hops," that's also Bellman-Ford or layered-Dijkstra (state = `(node, hops_used)`).
- **For Prim's, the heap can grow to O(E) with lazy decrease-key — that's fine.** Skip popped nodes already in the MST.
- **For *Alien Dictionary*, build edges only from the *first differing character* of each adjacent pair.** Comparing entire words gives you redundant edges and easy bugs.
- **Hierholzer's post-order is the trick.** Don't try to build the itinerary front-to-back via DFS — you'll hit dead ends. Build it back-to-front by appending after recursion.
- **Mention Bidirectional Dijkstra / A-star** as follow-ups for *Swim In Rising Water* in the discussion phase if you have time.

## Linked concepts

- [Network Delay Time (DSA Design)](../../dsa-design/problems/network-delay-time.md) — Dijkstra as a service.
- [Graph Shortest Path (DSA Design)](../../dsa-design/problems/graph-shortest-path.md) — both BFS (unweighted) and Dijkstra (weighted) variants.
- [Social Network Connection (DSA Design)](../../dsa-design/problems/social-network-connection.md) — Union-Find / BFS for connectivity at scale, the cousin of MST.

# Graphs

> **13 problems** · prerequisites below

## What this topic teaches

Graph problems are where pattern recognition pays off most: the problem statement is almost never "do a graph traversal" — it's "count islands," "see which cells can reach the ocean," "rot every reachable orange in N steps," "are these courses takeable?" Your job is to recognise the underlying graph (nodes = grid cells / nodes = courses / nodes = words), pick **DFS or BFS** depending on whether you need *reachability* or *shortest distance*, and execute the textbook traversal with the right bookkeeping.

The split between BFS and DFS is the single most important decision. **BFS** (queue, layer-by-layer) gives shortest path in unweighted graphs and is the right choice when the problem mentions "minimum time" or "fewest steps" (*Rotting Oranges*, *Walls and Gates*, *Word Ladder*). **DFS** (recursive or explicit stack) is for reachability, connected components, and cycle detection (*Number of Islands*, *Clone Graph*, *Course Schedule*). Union-Find is the third tool — use it for "are these connected?" queries over a stream of edges (*Graph Valid Tree*, *Number of Connected Components*, *Redundant Connection*).

## Prerequisites

- Intro to Graphs (Data Structures & Algorithms for Beginners)
- Matrix DFS (Data Structures & Algorithms for Beginners)
- Matrix BFS (Data Structures & Algorithms for Beginners)
- Adjacency List (Data Structures & Algorithms for Beginners)

## Core patterns

### Pattern 1: Grid DFS / BFS for connected components

For each cell, if it's land and unvisited, run DFS/BFS marking all reachable land cells, increment a counter (or accumulate area). *Number of Islands* (count), *Max Area of Island* (sum cells in the component), *Surrounded Regions* (mark border-connected `O`s first, then flip the rest). Direction array `dirs = [(0,1),(1,0),(0,-1),(-1,0)]` is universal.

### Pattern 2: Multi-source BFS

When multiple starting cells should expand simultaneously and you want the time-step (or shortest distance) at which each cell is reached, push *all* sources into the queue at the start with distance 0. *Rotting Oranges* (all initially-rotten cells), *Walls and Gates* (all gates, fill distances to empty rooms). One BFS instead of N — same correctness, n× faster.

### Pattern 3: Reverse-direction BFS for "reachable from"

*Pacific Atlantic Water Flow*: instead of "from each cell, can water flow to both oceans?" (which is N² BFS), invert it — start a BFS *from each ocean* moving to cells with **greater or equal** height, mark reachable cells; the answer is the intersection. Reversing direction frequently turns N source-traversals into 2.

### Pattern 4: Hash-map clone (DFS or BFS)

*Clone Graph*: walk the original, maintain `old → new` map. Before recursing on a neighbour, check if it's already in the map (return the clone) — this is what handles cycles. The same trick clones any graph-like structure.

### Pattern 5: Topological order / cycle detection (DFS)

*Course Schedule* (is the prereq graph a DAG?) and *Course Schedule II* (output a valid order). DFS with three-colour marking (white/gray/black or 0/1/2): on encountering a gray node, you've found a cycle. Post-order push to a list, then reverse, gives a topological order. Kahn's BFS version (queue of zero-in-degree nodes) is the BFS alternative — both are O(V + E).

### Pattern 6: Union-Find (Disjoint Set Union)

For "are these connected?" / "does adding this edge create a cycle?" over a stream of edges. `parent[]` array, path compression in `find`, union by rank/size. *Number of Connected Components* (count distinct roots after all unions), *Graph Valid Tree* (it's a tree iff `n - 1` edges and no cycle on union), *Redundant Connection* (the first edge whose two endpoints are already connected).

### Pattern 7: Word-ladder BFS over implicit graph

*Word Ladder*: the graph is implicit — two words are neighbours if they differ by one letter. Pre-build a map `pattern → words` (e.g. `h*t → [hot, hit, hut]`) by replacing each letter with `*`. BFS level-by-level until you hit `endWord`. Bidirectional BFS is the optimisation question to mention.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Number of Islands | Medium |
| 2 | Max Area of Island | Medium |
| 3 | Clone Graph | Medium |
| 4 | Walls And Gates | Medium |
| 5 | Rotting Oranges | Medium |
| 6 | Pacific Atlantic Water Flow | Medium |
| 7 | Surrounded Regions | Medium |
| 8 | Course Schedule | Medium |
| 9 | Course Schedule II | Medium |
| 10 | Graph Valid Tree | Medium |
| 11 | Number of Connected Components In An Undirected Graph | Medium |
| 12 | Redundant Connection | Medium |
| 13 | Word Ladder | Hard |

## Tips

- **Decide DFS vs BFS before you start coding.** "Shortest" / "minimum steps" / "fewest moves" → BFS. "Count components" / "is reachable" / "has a cycle" → DFS or Union-Find.
- **For DFS in big grids, prefer iterative DFS (explicit stack) over recursion** to avoid stack overflow at ~10⁴ cells deep.
- **Mark visited in-place when you can.** Overwrite `1` with `#` (or `0`) during the DFS — saves the visited-set memory and is one fewer thing to maintain.
- **For Union-Find, always implement path compression in `find`** (`parent[x] = find(parent[x])`). Without it, each `find` is O(log n); with it, the amortised cost is essentially O(1) (inverse Ackermann).
- **For *Word Ladder*, build the `pattern → words` map once.** Building the adjacency list pairwise is O(N² · L) and times out; the pattern map is O(N · L²) and fast.
- **Topological sort: cycle == no valid order.** Don't waste cycles trying to "fix" a cycle in *Course Schedule II* — return `[]`.

## Linked concepts

- [Graph Shortest Path (DSA Design)](../../dsa-design/problems/graph-shortest-path.md) — BFS on an unweighted graph as a streaming API.
- [Social Network Connection (DSA Design)](../../dsa-design/problems/social-network-connection.md) — Union-Find for friend groups with dynamic edges.
- [Word Ladder (DSA Design)](../../dsa-design/problems/word-ladder.md) — production version of the pattern-map BFS.
- [Network Delay Time (DSA Design)](../../dsa-design/problems/network-delay-time.md) — Dijkstra on a weighted graph (lives in *Advanced Graphs* but starts here).

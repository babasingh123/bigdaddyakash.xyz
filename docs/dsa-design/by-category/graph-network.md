# Graph & Network

Every "network," "social," "metro," or "shortest path" question is a thinly disguised graph problem. The two algorithms you need are **BFS** (unweighted shortest path) and **Dijkstra** (weighted shortest path). The data structure is always the same: an **adjacency list**.

## The pattern

A graph is just `Map<Node, List<Edge>>`. From that, three algorithms cover almost everything in this category:

- **BFS** — shortest path in an unweighted graph; level-by-level traversal.
- **Dijkstra** — shortest path with non-negative weights; BFS with a priority queue.
- **Bidirectional BFS** — search from both ends to halve the explored frontier.

The variations come from *what the nodes and edges represent*:

- **Explicit graph** (Network Delay Time) — nodes given, edges given.
- **Implicit graph** (Word Ladder) — nodes are words, edges are "one letter different." You construct the graph by *querying* adjacency rather than storing it.
- **Social graph** — friends are edges, mutual-friends-at-distance-2 is a BFS-up-to-depth-2 problem.
- **Time-on-routes problem** (Underground System) — barely a graph at all; just nested HashMaps tracking averages.

The hardest part is usually recognizing that there's a graph at all. Once you see it, the algorithm is rarely the surprise.

## Problems

- [Graph with Shortest Path](../problems/graph-shortest-path.md) — Adjacency List + Dijkstra/BFS
- [Social Network Connection](../problems/social-network-connection.md) — Adjacency List + BFS up to depth 2
- [Underground System](../problems/underground-system.md) — Nested HashMaps for averages
- [Word Ladder](../problems/word-ladder.md) — Implicit graph + BFS (or bidirectional BFS)
- [Network Delay Time](../problems/network-delay-time.md) — Dijkstra's algorithm

## Why interviewers love this category

Graph questions are where interviewers separate "I've memorized BFS" from "I can model a problem as a graph." The signal they want:

1. **Recognition.** Spotting that "friend recommendation" or "word transformation" is a graph problem is half the answer. The candidate who starts with "let me build an adjacency list" has already won.
2. **Picking BFS vs Dijkstra correctly.** Unweighted → BFS. Weighted with non-negative edges → Dijkstra. Negative edges → Bellman-Ford. Getting this wrong wastes the rest of the interview.
3. **Handling implicit graphs.** Word Ladder is the canonical test — there's no graph in the input, you have to *imagine* it. The candidate who builds an adjacency list by mutating each character and looking up the dictionary is doing graph theory under the hood.
4. **Dijkstra implementation cleanliness.** A clean Dijkstra has *one heap pop per node finalization*, lazy decrease-key (push duplicates and skip stale entries), and proper "visited" tracking. Most candidates butcher one of these three.

The mental model: **"what are my nodes? what are my edges? do edges have weights?"** Answer those three, and the algorithm picks itself.

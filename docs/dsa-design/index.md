# DSA Disguised as Design

> 50 "design" problems that are really pure data-structure and algorithm questions wearing a costume.

## The trick

Real **system design** (HLD) is about distributed systems, capacity planning, partitioning, replication, and trade-offs at scale. Real **object-oriented design** (LLD) is about classes, responsibilities, and SOLID principles.

But there's a third category interviewers love: questions that *sound* like design but really test whether you know which data structure to reach for. They say **"Design X"** instead of **"Implement X"** to see if you can spot the underlying pattern.

!!! tip "The pattern recognition test"
    - "Design LRU Cache" → "Implement a HashMap + Doubly Linked List"
    - "Design Search Autocomplete" → "Implement a Trie + Heap"
    - "Design Rate Limiter" → "Implement a Sliding Window + Queue"
    - "Design Median Finder" → "Implement Two Heaps"

The word "design" is camouflage. Underneath, you're being tested on whether you can:

1. **Recognize** which classical DSA pattern this problem maps to.
2. **Justify** why that data structure is the right tool (what does it give you that alternatives don't).
3. **Implement** it cleanly, hitting the expected time/space complexity.

Every page in this section calls out the disguise, then teaches the underlying data structure.

## The 10 categories

Each category groups problems that share an underlying DSA pattern.

- [Cache & Memory Management](by-category/cache-memory.md) — HashMap + Linked List combinations for O(1) eviction
- [Search & Autocomplete](by-category/search-autocomplete.md) — Tries for prefix matching
- [Rate Limiting & Counting](by-category/rate-limiting-counting.md) — Sliding window with queues and timestamps
- [Scheduling & Priority](by-category/scheduling-priority.md) — Heaps and TreeMaps for ordered work
- [Ranking & Leaderboard](by-category/ranking-leaderboard.md) — TreeMap, Heap, and two-heap patterns
- [Graph & Network](by-category/graph-network.md) — BFS, Dijkstra, and adjacency lists
- [String & Pattern](by-category/string-pattern.md) — Iterators, stacks, and snapshot tricks
- [Game & Simulation](by-category/game-simulation.md) — 2D arrays, deques, and clever optimizations
- [Data Stream & Real-Time](by-category/data-stream.md) — Two heaps, tries, and interval merging on streams
- [Allocation & Resource Management](by-category/allocation-resource.md) — TreeSets and bit tricks

## Quick reference: problem → data structure

| Need | Pattern |
|------|---------|
| O(1) operations with ordering | HashMap + (LinkedList or Array) |
| Sorted data, range queries | TreeMap / sorted dict |
| Prefix search | Trie |
| Sliding window over time | Queue + HashMap |
| Min / max repeatedly | Heap |
| Top K | Heap of size K |
| Undo / redo | Two Stacks |
| Interval merging | TreeMap + merge logic |
| Graph problems | Adjacency List + BFS/DFS |
| Running median | Two Heaps |

## How to use this section

Read it like a *pattern catalog*, not a problem set. Pick a category, read the **pattern** section first, then drill into a couple of problems. The reasoning for *why* each data structure is chosen matters far more than memorizing code — that reasoning is what you'll need to recreate in an interview.

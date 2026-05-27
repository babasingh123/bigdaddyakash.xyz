# Cache & Memory Management

When the interviewer says "design a cache," they're really asking: *can you combine two data structures to get O(1) access **and** O(1) eviction of the right element?* No single structure gives you both. The answer is almost always a HashMap glued to a Linked List.

## The pattern

The core trick across this whole category is **structure composition**:

- A **HashMap** gives you O(1) key lookup, but no notion of order.
- A **Linked List** (singly or doubly) gives you O(1) insert/remove at known positions, plus a clear notion of "oldest" or "newest."
- Wire them together: the HashMap's value is a *pointer into the linked list*.

That single idea unlocks LRU, LFU, time-travel KV stores, file systems (tree of HashMaps), and browser history (two stacks or a DLL with a cursor). The variations come from *what kind of order* you maintain:

- **Recency** → DLL where access moves a node to the head.
- **Frequency** → HashMap of frequency-buckets, each bucket is a DLL.
- **Time** → HashMap of TreeMaps, binary search to find a version.
- **Hierarchy** → Tree of HashMaps, paths walked node-by-node.
- **Navigation history** → Two stacks, or a DLL with a "current" pointer.

The reason interviewers love this category: it forces you to *justify* your composition. "Why DLL and not singly linked? Because I need O(1) removal from the middle when a key is accessed."

## Problems

- [LRU Cache](../problems/lru-cache.md) — HashMap + Doubly Linked List
- [LFU Cache](../problems/lfu-cache.md) — Multi-level HashMap + DLL per frequency
- [Time-Based Key-Value Store](../problems/time-based-kv-store.md) — HashMap + TreeMap (binary search on timestamps)
- [In-Memory File System](../problems/in-memory-file-system.md) — Tree of HashMaps (trie-like)
- [Browser History](../problems/browser-history.md) — Two Stacks (or DLL with cursor)

## Why interviewers love this category

These problems force you to demonstrate three things at once:

1. **You recognize that no single off-the-shelf structure suffices.** Pure HashMap loses order. Pure LinkedList loses lookup speed. The whole point is to compose.
2. **You can reason about pointers and edge cases under pressure.** Doubly linked list manipulation is a classic place to break — head/tail sentinels, removing the only node, moving a node to the front. These are exactly the bugs interviewers probe.
3. **You understand eviction policies.** LRU vs LFU vs TTL aren't just words — they map to different structures, and the choice cascades through your whole design.

When you walk into one of these problems, start by writing the desired complexity on the whiteboard: *"I want O(1) for both `get` and `put`."* Then ask: *"Which data structure can I add to a HashMap to make eviction O(1)?"* That sentence is the entire interview.

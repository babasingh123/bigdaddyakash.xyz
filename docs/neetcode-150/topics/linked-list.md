# Linked List

> **11 problems** · prerequisites below

## What this topic teaches

Linked-list problems are pointer surgery: you don't have indices, so every algorithm reduces to "what nodes do I currently hold references to, and which `.next` do I overwrite when?" Three primitive tools cover almost every problem in the set: **dummy head nodes** to avoid edge cases at the front, **fast/slow pointers** to find the middle or detect cycles in one pass, and **iterative reversal** to flip a sublist (or the whole list) in-place with three pointers (`prev`, `curr`, `next`).

The reason this topic shows up so often in interviews is that it forces you to reason about *aliasing*. When `a.next = b` and `b.next = c`, mutating one pointer can quietly leave the others dangling. Drawing the diagram **before** you write the swap — and updating it line-by-line as you go — is not optional; it's the algorithm.

## Prerequisites

- Singly Linked Lists (Data Structures & Algorithms for Beginners)
- Doubly Linked Lists (Data Structures & Algorithms for Beginners)
- Fast and Slow Pointers (Advanced Algorithms)

## Core patterns

### Pattern 1: Dummy head + tail pointer

`dummy = ListNode(0); tail = dummy`; build the result by `tail.next = something; tail = tail.next`; return `dummy.next`. This eliminates the "is this the first node?" branch in *Merge Two Sorted Lists*, *Add Two Numbers*, and *Remove Nth Node*. Whenever the output is "build a new list (or partial list) one node at a time," reach for the dummy.

### Pattern 2: Fast and slow pointers (Floyd's tortoise and hare)

`slow = head, fast = head`; in the loop, `slow = slow.next, fast = fast.next.next`. Two uses: (a) find the middle when `fast` falls off the end, `slow` is at the middle (*Reorder List* uses this to split); (b) detect a cycle when `fast == slow`. To find the cycle's *start*, reset one pointer to `head` and walk both one step at a time until they meet again. *Linked List Cycle* and *Find The Duplicate Number* (numbers as next pointers) both lean on this.

### Pattern 3: Iterative reverse-in-place

`prev = None; curr = head; while curr: nxt = curr.next; curr.next = prev; prev = curr; curr = nxt; return prev`. Five lines, memorise them. Used directly (*Reverse Linked List*), as a subroutine on a sublist (*Reverse Nodes In K Group* — reverse k nodes at a time, stitch the pieces), and as a building block (*Reorder List* — reverse second half, then interleave).

### Pattern 4: Hash map for "old → new" mapping

When you have to clone a list with side pointers (*Copy List With Random Pointer*), make two passes: first pass creates a hash map `old → new` of bare clones; second pass wires up `new.next = map[old.next]` and `new.random = map[old.random]`. The O(1)-space variant interleaves originals and clones in the same list — clever, but the hash-map version is what you write in an interview.

### Pattern 5: Heap-merge of k lists

*Merge K Sorted Lists* uses a min-heap of `(value, list_id, node)` tuples. Pop the smallest, attach it to the tail, push its `.next` if any. O(n log k) total. The alternative — pairwise merging in `log k` rounds — is the same complexity and a good answer if a heap isn't allowed.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Reverse Linked List | Easy |
| 2 | Merge Two Sorted Lists | Easy |
| 3 | Linked List Cycle | Easy |
| 4 | Reorder List | Medium |
| 5 | Remove Nth Node From End of List | Medium |
| 6 | Copy List With Random Pointer | Medium |
| 7 | Add Two Numbers | Medium |
| 8 | Find The Duplicate Number | Medium |
| 9 | LRU Cache | Medium |
| 10 | Merge K Sorted Lists | Hard |
| 11 | Reverse Nodes In K Group | Hard |

## Tips

- **Draw the diagram before you write the swap.** Five circles, four arrows, and a pencil beat any amount of reasoning in your head. Mark which pointer you're about to overwrite and what reference you need to save first.
- **Always use a dummy head when the answer involves a new list or the head might change.** Saves an entire branch of edge-case code.
- **For "Nth from end," use a two-pointer offset.** Move `fast` `n` steps ahead, then walk `slow` and `fast` together until `fast` is at the end. `slow` is now at the node *before* the one to remove (with a dummy in front, this is one expression: `slow.next = slow.next.next`).
- **LRU Cache = hash map + doubly linked list.** The hash map gives O(1) lookup; the doubly linked list gives O(1) move-to-front. Don't try to do it with just one structure.
- **Cycle-start derivation (Floyd's)**: after meet, distance from head to start = distance from meet to start. Don't memorise the proof, but know the procedure — reset one pointer to head, walk both at speed 1, they meet at the start.

## Linked concepts

- [LRU Cache (DSA Design)](../../dsa-design/problems/lru-cache.md) — the production-grade doubly-linked-list + hash map design.
- [LFU Cache (DSA Design)](../../dsa-design/problems/lfu-cache.md) — DLL per frequency bucket, hash map of buckets, evictions in O(1).
- [Browser History (DSA Design)](../../dsa-design/problems/browser-history.md) — doubly linked list with truncation on `visit`.

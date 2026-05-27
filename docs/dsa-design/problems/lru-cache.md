# Design LRU Cache

!!! note "The disguise"
    Interviewer says: **"Design a Least Recently Used cache."**
    They mean: **"Implement a HashMap + Doubly Linked List that gives O(1) get and put with O(1) eviction of the oldest entry."**

## Problem

Build a class `LRUCache(capacity)` with two operations:

- `get(key)` — return the value if present, else `-1`. Accessing a key marks it as most-recently-used.
- `put(key, value)` — insert or update. If the cache is full, evict the least-recently-used entry.

Both operations must run in **O(1) average time**.

## What it really tests

- **HashMap as a pointer index.** You need O(1) lookup from key to "where in my order structure does this live?" That's a HashMap whose value is a *node reference* (not the value itself directly).
- **Doubly Linked List for order.** You need to (a) know the oldest node in O(1) to evict it and (b) move any node to "newest" in O(1) when it's accessed. Only a doubly linked list supports both.
- **The composition.** Neither structure alone works. The interview is whether you can *glue them together* without bugs.
- **Cache eviction policies.** You should be able to compare LRU to LFU and TTL in one sentence each.

## Approach

Start with the desired API and work backwards from the complexity target.

We want O(1) for both `get` and `put`. Consider the candidates:

- **Pure HashMap?** O(1) lookup, but no ordering → can't find "oldest" without scanning. ✗
- **Pure Linked List?** Has ordering, but O(n) lookup → `get(key)` is too slow. ✗
- **Sorted structure (TreeMap)?** O(log n), not O(1). And ordering by "recency" doesn't fit a comparator naturally. ✗
- **Array + index map?** "Move to front" on an array is O(n) shift. ✗
- **HashMap + Doubly Linked List?** HashMap gives O(1) `get(key) → node`. DLL gives O(1) "remove this node" and "insert at head." ✓

That last combination is the only one that hits the target.

**Why doubly linked, not singly?** When `get` is called on a middle node, we need to *remove it from its current position* and *move it to the head*. To remove a node from a singly linked list you need the *previous* node, which costs O(n) to find. A DLL lets each node know its prev, so removal is O(1).

**The structure**:

- `head` and `tail` sentinel nodes — these are dummies so we never special-case "empty list." `head.next` is the most-recently-used; `tail.prev` is the least.
- `cache: dict[key, Node]` — maps a key to its node in the DLL.
- Each `Node` holds `(key, value, prev, next)`. We store the *key* on the node so that when we evict from the tail, we know which entry to delete from the HashMap.

**Operation flow**:

- `get(key)`: HashMap lookup → if missing, return -1. Otherwise, unlink node from its position, re-insert at head, return value.
- `put(key, val)`: if key exists, update value and move to head. If not, create a new node, insert at head, and add to HashMap. If `len > capacity`, remove the tail's predecessor (the LRU node) and delete it from the HashMap.

## Code sketch

```python
class Node:
    __slots__ = ("key", "val", "prev", "next")
    def __init__(self, key=0, val=0):
        self.key, self.val = key, val
        self.prev = self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.cache: dict[int, Node] = {}
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: Node) -> None:
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_front(self, node: Node) -> None:
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_front(node)
        return node.val

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.val = value
            self._remove(node)
            self._add_to_front(node)
            return
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_front(node)
        if len(self.cache) > self.cap:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

In Python you can cheat with `collections.OrderedDict` (its `move_to_end` is exactly the operation we need), but interviewers expect you to know the underlying mechanism.

## Complexity

- `get(key)`: **O(1)** — HashMap lookup + constant pointer surgery.
- `put(key, val)`: **O(1)** — same.
- Space: **O(capacity)** — one node per cached entry, plus two sentinels.

## Edge cases

- **Capacity 0 or 1.** With capacity 0, every `put` should be a no-op (or the question banned it). With capacity 1, every `put` overwrites.
- **Update of existing key.** Must move it to head *and* update the value. Easy to forget the move.
- **First `put` into empty cache.** Sentinels make this trivial; without them it's a bug magnet.
- **`get` on a missing key.** Return `-1`, don't mutate the list.

## Linked concepts

- [Cache & Memory category overview](../by-category/cache-memory.md)
- [Caching strategies (HLD)](../../hld/fundamentals/caching.md)
- [LFU Cache](lfu-cache.md) — the harder cousin
- [Browser History](browser-history.md) — DLL with a cursor instead of LRU order

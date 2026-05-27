# Design an LRU Cache

> **Time:** 30–45 minutes

## Problem

Design a cache with **Least Recently Used (LRU)** eviction. The system must:

- Support `get(key)` returning the cached value, or null/`-1` if not present.
- Support `put(key, value)` inserting or updating a value.
- Have a fixed **capacity**; when full, **evict the least recently used** entry on insert.
- Both operations must run in **O(1)** time.

## Real-world intuition

LRU caches are everywhere — CDNs, database buffer pools, browser caches, page replacement in operating systems. They balance two simple ideas: hot items should be cheap to access, and stale items should fall out automatically. The data-structure question (how do you achieve O(1) on both `get` and `put`) is the entire problem — this is more of a **data structure design** exercise than a typical class-modeling one.

## Key classes

- `LRUCache<K, V>` — the public API: `get`, `put`, `size`, `capacity`.
- `Node<K, V>` — internal doubly-linked-list node holding key, value, and prev/next pointers.
- `DoublyLinkedList<K, V>` (optional helper) — encapsulates list operations: `addFirst`, `remove`, `moveToFront`, `removeLast`.
- (For interview: usually inlined into `LRUCache` for simplicity.)

## Design patterns used

This problem is intentionally pattern-light. It is a **data structure design** exercise. The patterns worth noting:

- **Composition of HashMap + Doubly Linked List** — neither structure alone gives O(1) for both operations. The combination does. This is "use the right data structures" more than "apply a pattern".
- **Optional Strategy (EvictionPolicy)** — if you want to support LRU, LFU, FIFO interchangeably, abstract the eviction policy behind an interface. Often overkill; mention as an extension point.

## Class diagram (sketch)

```text
   ┌─────────────────────────┐
   │ LRUCache<K, V>          │
   ├─────────────────────────┤
   │ -capacity               │
   │ -map: HashMap<K, Node>  │
   │ -head: Node (MRU end)   │
   │ -tail: Node (LRU end)   │
   │ +get(k): V              │
   │ +put(k, v)              │
   └─────────────────────────┘

   ┌──────────────┐
   │   Node<K,V>  │  doubly-linked list node
   ├──────────────┤
   │ -key, -value │
   │ -prev, -next │
   └──────────────┘

   HashMap gives O(1) key lookup;
   Doubly Linked List gives O(1) move-to-front / remove-tail.
```

## Code sketch

```java
public class LRUCache<K, V> {

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;   // sentinel; head.next is the MRU
    private final Node<K, V> tail;   // sentinel; tail.prev is the LRU

    public LRUCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be positive");
        this.capacity = capacity;
        this.map = new HashMap<>(capacity * 4 / 3 + 1);
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> n = map.get(key);
        if (n == null) return null;
        moveToFront(n);                  // touch — now most recently used
        return n.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;
            moveToFront(existing);
            return;
        }
        if (map.size() == capacity) {
            Node<K, V> lru = tail.prev;  // LRU is just before tail sentinel
            remove(lru);
            map.remove(lru.key);
        }
        Node<K, V> node = new Node<>(key, value);
        addFirst(node);
        map.put(key, node);
    }

    public int size()    { return map.size(); }
    public int capacity(){ return capacity; }

    private void addFirst(Node<K, V> n) {
        n.next = head.next;
        n.prev = head;
        head.next.prev = n;
        head.next = n;
    }

    private void remove(Node<K, V> n) {
        n.prev.next = n.next;
        n.next.prev = n.prev;
        n.prev = n.next = null;          // help GC
    }

    private void moveToFront(Node<K, V> n) {
        remove(n);
        addFirst(n);
    }
}
```

## Why this design hits O(1)

- **`HashMap.get(key)`** → O(1) average. Gets us the `Node` reference instantly.
- **`Node.prev` / `Node.next`** → O(1) to splice out and splice in.
- **Head / tail sentinels** → O(1) to add to front and remove from back without null checks. (Sentinels avoid special-casing the empty list and the first/last element.)

Without the doubly-linked list, moving an arbitrary element to the front would require O(n) traversal. Without the hashmap, finding the node by key would require O(n) traversal. Both structures are necessary.

## Key design decisions

1. **HashMap + Doubly Linked List combination** — the canonical O(1) LRU. Memorize this.
2. **Sentinel head/tail nodes** — eliminate edge cases for the first and last elements. Code is uniform: every node has a `prev` and a `next`, never null.
3. **Most-recently-used at the head, least-recently-used at the tail** — convention is arbitrary but pick one and stick with it. The naming `head.next == MRU` and `tail.prev == LRU` is consistent.
4. **`get` is a mutation** — accessing a key updates the LRU order. This violates the "queries should be const" instinct; embrace it. Concurrent variants need a write lock for `get`, which is a design call (see extensions).
5. **`put` of an existing key just updates the value and moves to front** — does not count against capacity, doesn't evict anything new. Easy to get wrong on a whiteboard.
6. **No `LinkedHashMap` shortcut for the interview** — Java's `LinkedHashMap` can be configured as an LRU in one line, but the interviewer wants to see the underlying mechanism. Implement it from scratch.

## Alternative: `LinkedHashMap` (production code)

Outside of interviews, this is a one-liner:

```java
class LinkedLruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    public LinkedLruCache(int capacity) {
        super(capacity, 0.75f, /* accessOrder = */ true);
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

Mention this if asked. The from-scratch version is the *interview* answer; this is the *production* one.

## Extensibility

**Add TTL (time-based expiration in addition to LRU):**

1. Each `Node` gains an `expiresAt`.
2. `get` checks expiry; if expired, remove and return null.
3. Optionally a background sweeper purges expired nodes.

**Add LFU (Least Frequently Used) as an alternative:**

1. Abstract an `EvictionPolicy` interface.
2. `LRUPolicy` (current implementation) and `LFUPolicy` (frequency-counter + min-heap or doubly-linked frequency buckets) both implement it.
3. Cache delegates eviction decisions.

**Add thread safety:**

1. Wrap everything in a `synchronized` or use a `ReentrantLock`. `get` must hold a write lock because it mutates order.
2. For higher throughput, segment the cache (concurrent hash-based partitioning) and lock per segment.

**Add capacity in bytes, not entries:**

1. Each entry has a size estimate; track total bytes; evict from the tail until under capacity.

**Add a write-through to a backing store:**

1. `put` writes to the cache and the backing store.
2. `get` consults the cache first, falls back to the store and populates on hit.

## Linked concepts

- [OOP](../fundamentals/oop.md) — small, focused classes; clean public API.
- [SOLID](../fundamentals/solid.md) — SRP on `Node` (just data) vs `LRUCache` (orchestration); OCP via optional eviction-policy abstraction.
- [Design Patterns](../fundamentals/design-patterns.md) — minimal here; the lesson is "use the right data structures" rather than "apply a pattern".

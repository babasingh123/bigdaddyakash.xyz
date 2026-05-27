# Design LFU Cache

!!! note "The disguise"
    Interviewer says: **"Design a Least Frequently Used cache."**
    They mean: **"Implement a HashMap + (Frequency → Doubly Linked List) structure that evicts the least-frequent, breaking ties by recency, all in O(1)."**

## Problem

Build `LFUCache(capacity)` with:

- `get(key)` — return value if present (else `-1`), and increment the key's frequency.
- `put(key, value)` — insert/update, increment frequency, evict if over capacity.

When evicting, remove the entry with the **lowest frequency**. If multiple share the lowest frequency, remove the **least recently used** among them.

Both operations must be **O(1)**.

## What it really tests

- **Multi-level HashMaps.** You need two indexes — one by key, one by frequency.
- **Per-frequency DLLs.** Each frequency level is itself an LRU list (for tie-breaking by recency).
- **Tracking the minimum frequency.** You need O(1) access to the bucket holding the LFU items.
- **Strictly harder than LRU.** This is the "make me sweat" cache problem.

## Approach

Start with what changes from LRU. In LRU, "order" is purely recency, so one DLL suffices. In LFU, "order" is `(frequency_asc, recency_asc)`. That's two dimensions, so one DLL can't capture it.

**Idea:** group by frequency. Each frequency *f* gets its own DLL — all keys currently at frequency *f*, ordered by recency. Inside a single DLL we get O(1) LRU semantics for that bucket.

**Three structures:**

1. `key_to_node: dict[key, Node]` — O(1) lookup. Each node stores `(key, value, freq)`.
2. `freq_to_list: dict[freq, DLL]` — bucket of nodes that share that frequency.
3. `min_freq: int` — the smallest frequency that currently has any nodes. Crucial for O(1) eviction.

**On `get(key)`:**

- Look up node. Remove it from `freq_to_list[node.freq]`.
- Increment `node.freq`. Append it to `freq_to_list[node.freq + 1]`.
- If the old bucket is now empty *and* it was `min_freq`, increment `min_freq` by 1.

**On `put(key, value)`:**

- If key exists, update value and run the `get` flow.
- Otherwise, if at capacity, evict the head (LRU) of `freq_to_list[min_freq]`. Insert the new node at frequency 1, set `min_freq = 1`.

**Why does `min_freq` jump cleanly?** When you bump a key from *f* to *f+1*, if *f* was the min and the bucket is now empty, *f+1* is the new min (because the node we just moved is now there). On insertion of a brand-new node, frequency is 1, which is always ≤ any existing min, so set `min_freq = 1`. Those two cases cover all transitions.

**Why each bucket is itself a DLL:**

- Within a frequency, ties break by recency → we need LRU semantics in the bucket → DLL again.
- This is also why every operation stays O(1): we always remove a known node from a known bucket.

## Code sketch

```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.min_freq = 0
        self.key_to_val: dict[int, int] = {}
        self.key_to_freq: dict[int, int] = {}
        self.freq_to_keys: dict[int, OrderedDict] = defaultdict(OrderedDict)

    def _bump(self, key: int) -> None:
        f = self.key_to_freq[key]
        del self.freq_to_keys[f][key]
        if not self.freq_to_keys[f]:
            del self.freq_to_keys[f]
            if self.min_freq == f:
                self.min_freq += 1
        self.key_to_freq[key] = f + 1
        self.freq_to_keys[f + 1][key] = None

    def get(self, key: int) -> int:
        if key not in self.key_to_val:
            return -1
        self._bump(key)
        return self.key_to_val[key]

    def put(self, key: int, value: int) -> None:
        if self.cap == 0:
            return
        if key in self.key_to_val:
            self.key_to_val[key] = value
            self._bump(key)
            return
        if len(self.key_to_val) >= self.cap:
            evict_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            del self.key_to_val[evict_key]
            del self.key_to_freq[evict_key]
            if not self.freq_to_keys[self.min_freq]:
                del self.freq_to_keys[self.min_freq]
        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1
```

`OrderedDict` here is doing the job of a doubly linked list — `popitem(last=False)` is O(1) and pops the oldest, which is exactly the LRU tie-break we need.

## Complexity

- `get`: **O(1)** — one HashMap lookup + one bucket remove + one bucket insert.
- `put`: **O(1)** — same shape, plus an O(1) eviction.
- Space: **O(capacity)**.

## Edge cases

- **Capacity 0.** Every `put` is a no-op. Handle this up front; otherwise `min_freq` logic gets weird.
- **Update vs insert.** An update bumps frequency *and* updates value — don't forget the value part.
- **Empty bucket cleanup.** Always delete the bucket dict entry when its DLL/OrderedDict goes empty, or `min_freq` can land on a stale empty bucket.
- **`min_freq` after eviction.** If you evict the *last* member of `min_freq`'s bucket and then insert a brand-new key, the new key is at frequency 1 — reset `min_freq` to 1.

## Linked concepts

- [Cache & Memory category overview](../by-category/cache-memory.md)
- [Caching strategies (HLD)](../../hld/fundamentals/caching.md)
- [LRU Cache](lru-cache.md) — the simpler cousin

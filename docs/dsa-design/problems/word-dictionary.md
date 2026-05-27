# Design Word Dictionary

!!! note "The disguise"
    Interviewer says: **"Design a dictionary that can check if a word exists or has a prefix."**
    They mean: **"Implement a Trie with `insert`, `search`, and `startsWith`."**

## Problem

Build `Trie` with:

- `insert(word)` — store a word.
- `search(word)` — return `True` if the exact word was inserted.
- `startsWith(prefix)` — return `True` if any inserted word starts with `prefix`.

All operations should be **O(m)** where *m* is the length of the query.

## What it really tests

- **Basic trie construction.** Walk per character, create nodes on demand.
- **Distinguishing `search` from `startsWith`.** `search` checks `is_end`; `startsWith` doesn't.
- **Time complexity awareness.** Note that O(m) is independent of dictionary size — that's the *point* of a trie.

## Approach

The data structure: each node has `children: dict[char, Node]` and a `is_end: bool` flag.

**Why a Trie and not a HashSet?**

- HashSet gets `search` in O(m) (hashing the string), but cannot do `startsWith` in O(m). A HashSet of strings forces you to scan every string to find prefix matches → O(N × m).
- Trie supports `startsWith` natively. Walk *m* edges; if you don't fall off, the prefix exists.

**`insert` walk.** For each character, look up or create the child node. At the end, set `is_end = True`.

**`search` vs `startsWith`.** Both walk the same way:

```
walk down the trie character by character
if at any point the child is missing → return False
```

The only difference is the final check: `search` requires `is_end == True`; `startsWith` returns `True` either way.

**Implementation detail: children as `dict` vs `list[26]`.**

- `dict` (HashMap): cleaner, supports any character set, slightly higher constant factor.
- `list[26]` with index `ord(c) - ord('a')`: faster constant factor, locked to lowercase ASCII.

In an interview, `dict` is the safe answer. Mention the array trick as an optimization.

## Code sketch

```python
class TrieNode:
    __slots__ = ("children", "is_end")
    def __init__(self):
        self.children: dict[str, "TrieNode"] = {}
        self.is_end: bool = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def _walk(self, s: str) -> TrieNode | None:
        node = self.root
        for ch in s:
            node = node.children.get(ch)
            if node is None:
                return None
        return node

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_end

    def startsWith(self, prefix: str) -> bool:
        return self._walk(prefix) is not None
```

Factoring `_walk` keeps `search` and `startsWith` honest — same walking logic, different terminal check.

## Complexity

Let *m* = length of input string, *N* = total characters across all inserted words.

- `insert`: **O(m)**.
- `search`: **O(m)**.
- `startsWith`: **O(m)**.
- Space: **O(N)** in the worst case (no shared prefixes). Tries shine when words share prefixes.

## Edge cases

- **Empty word.** `insert("")` sets `root.is_end = True`. `search("")` returns `True`. `startsWith("")` returns `True`. Often the problem disallows empty strings; verify with the interviewer.
- **Inserting duplicates.** Already idempotent; same path, same `is_end`.
- **Non-ASCII characters.** With a `dict[char, Node]`, any character works. With an `int[26]` array, only `a-z`.
- **Querying a long string that diverges early.** Returns `False` quickly thanks to `is None` check.

## Linked concepts

- [Search & Autocomplete category overview](../by-category/search-autocomplete.md)
- [Add and Search Words](add-search-words.md) — same trie plus wildcard DFS
- [Search & indexing (HLD)](../../hld/fundamentals/search-indexing.md)

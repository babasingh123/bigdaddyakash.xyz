# Design Add and Search Words Data Structure

!!! note "The disguise"
    Interviewer says: **"Design a data structure that supports adding words and searching with wildcards."**
    They mean: **"Implement a Trie with DFS + backtracking on the `.` wildcard."**

## Problem

Support:

- `addWord(word)` — insert a word.
- `search(word)` — return `True` if any inserted word matches. The query word may contain the `.` character, which matches **any single letter**.

## What it really tests

- **Trie storage** for prefix-friendly search.
- **DFS with backtracking** at every `.` — try every child branch.
- **Pruning.** Once a branch can't possibly match, stop early.
- **Recursion vs iteration.** Iterative search works for plain letters but DFS is cleanest for wildcards.

## Approach

Without wildcards, this is a basic trie: walk the word character-by-character and check `isEnd` at the leaf.

The wildcard makes it interesting. At a `.`, we don't know which child to follow, so we must try *all of them*. That's a DFS + backtracking pattern.

**The recursive shape:**

```
def search_from(node, index):
    if index == len(word): return node.is_end
    ch = word[index]
    if ch == '.':
        for child in node.children.values():
            if search_from(child, index + 1): return True
        return False
    if ch not in node.children: return False
    return search_from(node.children[ch], index + 1)
```

That's the whole algorithm. Wildcards branch by **at most 26** (English alphabet); non-wildcards are O(1) descent.

**Why DFS and not BFS?** Either works correctness-wise, but DFS uses O(word length) stack instead of O(branching factor × word length) queue, and short-circuits naturally with `return True`.

**Pruning matters.** If a `.` is followed by characters that don't exist in the subtree, we can fail-fast on that branch. The base implementation already does this (returns `False` on missing key), but it's worth calling out: the trie *structurally prunes* the search space — every level eliminates non-matching letters.

**Worst-case branching.** A query of all dots `"..."` over a dense dictionary touches every node in the first *k* layers — that's O(26^k) in the absolute worst case. In practice the dictionary is sparse and this is far smaller.

## Code sketch

```python
class TrieNode:
    __slots__ = ("children", "is_end")
    def __init__(self):
        self.children: dict[str, "TrieNode"] = {}
        self.is_end: bool = False

class WordDictionary:
    def __init__(self):
        self.root = TrieNode()

    def addWord(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def search(self, word: str) -> bool:
        return self._dfs(self.root, word, 0)

    def _dfs(self, node: TrieNode, word: str, i: int) -> bool:
        if i == len(word):
            return node.is_end
        ch = word[i]
        if ch == ".":
            for child in node.children.values():
                if self._dfs(child, word, i + 1):
                    return True
            return False
        nxt = node.children.get(ch)
        return False if nxt is None else self._dfs(nxt, word, i + 1)
```

## Complexity

Let *L* = word length, *N* = number of words in dictionary, *Σ* = alphabet size (26 here).

- `addWord(word)`: **O(L)**.
- `search(word)` without wildcards: **O(L)**.
- `search(word)` with *k* wildcards: **O(L × Σ^k)** worst-case; typically much less because the trie is sparse.
- Space: **O(total characters across all inserted words)**.

## Edge cases

- **Empty word.** `addWord("")` marks the root as `is_end=True`; `search("")` returns `True`. Decide if you want to allow this.
- **Query of all dots, length L.** Worst case branching — defensible answer to the interviewer: "this could be expensive, but in practice the trie depth-bounds the explosion."
- **Duplicate adds.** Idempotent; the trie already has those nodes.
- **Query longer than any inserted word.** Returns `False` because the DFS runs out of children before reaching `i == len(word)` with `is_end`.

## Linked concepts

- [Search & Autocomplete category overview](../by-category/search-autocomplete.md)
- [Word Dictionary](word-dictionary.md) — same trie, no wildcards
- [Search & indexing (HLD)](../../hld/fundamentals/search-indexing.md)

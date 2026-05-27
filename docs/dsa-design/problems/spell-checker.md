# Design Spell Checker

!!! note "The disguise"
    Interviewer says: **"Design a spell checker with suggestions."**
    They mean: **"Implement a Trie plus edit-distance-bounded DFS (or BFS) to find similar words."**

## Problem

Given a dictionary of valid words, build a system that, given a query word, returns:

- `True / False` for "is this word spelled correctly?"
- A list of suggestions within edit distance *k* (typically 1 or 2). Edits are: insert, delete, substitute a letter.

## What it really tests

- **Trie for the dictionary** — O(prefix-length) descent and natural pruning.
- **Levenshtein (edit) distance** — the dynamic programming definition.
- **DFS that prunes by remaining edit budget.** Search the trie with a row of the DP table, pruning subtrees whose minimum row value exceeds the budget.
- **BFS over edit operations** as an alternative.

## Approach

The naive solution is "compute edit distance between query and every dictionary word." That's O(N × m × n). For a real dictionary (100K+ words) this is too slow.

The clever solution combines the trie with edit-distance DP:

**Idea.** Run the edit-distance DP one *row at a time* as you DFS the trie. Each trie node corresponds to a prefix; each row of the DP table corresponds to "edit distance from query to that prefix at each query position." If every value in the current row exceeds the budget *k*, prune — every word in this subtree must have edit distance > k.

**The row update.** Standard Levenshtein recurrence:

```
row[j] = min(
    row[j-1] + 1,          # insertion
    prev_row[j] + 1,       # deletion
    prev_row[j-1] + (query[j-1] != current_char)  # match/substitute
)
```

Compute the row for the trie node's character given the parent's row.

**The DFS.**

```
def dfs(node, char_into_node, prev_row, results):
    row = next_row(prev_row, char_into_node, query)
    if row[-1] <= k and node.is_end:
        results.append(current_word)
    if min(row) > k:
        return  # prune entire subtree
    for ch, child in node.children.items():
        dfs(child, ch, row, results)
```

This is the classic **trie + edit distance** algorithm. Total work is bounded by `O(|valid prefixes| × m)` where *m* is query length, which is far less than `O(N × m)` for dense dictionaries with shared prefixes.

**BFS variant.** For just `k=1`, you can enumerate all candidates by generating every single-edit variant of the query and checking each against the trie. That's `O(m × Σ)` candidates, each O(m) to check → O(m² × Σ). Simple but only works for small *k*.

**Why is this hard?** Most candidates jump to "compute edit distance N times" or to a too-clever fuzzy hash. The trie+DP solution is the canonical interview answer and requires you to remember the Levenshtein recurrence on the spot.

## Code sketch

```python
class TrieNode:
    __slots__ = ("children", "word")
    def __init__(self):
        self.children: dict[str, "TrieNode"] = {}
        self.word: str | None = None

class SpellChecker:
    def __init__(self, dictionary: list[str]):
        self.root = TrieNode()
        for w in dictionary:
            self._insert(w)

    def _insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.word = word

    def is_correct(self, query: str) -> bool:
        node = self.root
        for ch in query:
            node = node.children.get(ch)
            if node is None:
                return False
        return node.word is not None

    def suggest(self, query: str, k: int = 1) -> list[str]:
        m = len(query)
        # Initial row: edit distance from empty trie-prefix to query[:j]
        current_row = list(range(m + 1))
        results: list[str] = []
        for ch, child in self.root.children.items():
            self._dfs(child, ch, query, current_row, k, results)
        return results

    def _dfs(self, node, ch, query, prev_row, k, results):
        m = len(query)
        cur_row = [prev_row[0] + 1]
        for j in range(1, m + 1):
            cost = 0 if query[j - 1] == ch else 1
            cur_row.append(min(
                cur_row[j - 1] + 1,     # insertion
                prev_row[j] + 1,        # deletion
                prev_row[j - 1] + cost  # substitute / match
            ))
        if cur_row[-1] <= k and node.word is not None:
            results.append(node.word)
        if min(cur_row) <= k:
            for c, child in node.children.items():
                self._dfs(child, c, query, cur_row, k, results)
```

## Complexity

Let *N* = total dictionary characters, *m* = query length, *k* = edit-distance budget.

- `is_correct(query)`: **O(m)**.
- `suggest(query, k)`: hard to bound tightly. Worst case touches O(N × m), but in practice the pruning (`min(cur_row) > k`) cuts most subtrees aggressively, giving roughly **O(W × m)** where *W* is the count of trie prefixes within edit distance *k*.
- Space: **O(N)** for the trie, **O(depth × m)** for the recursion stack and DP rows.

## Edge cases

- **Query already in dictionary.** Should be included in suggestions (distance 0 ≤ k).
- **Empty query.** Suggestions are all words of length ≤ k.
- **Budget 0.** Equivalent to exact match — just walk the trie.
- **Case sensitivity.** Decide upfront whether `Hello` and `hello` are the same. Normalize on insert and on query.
- **Transpositions.** Plain Levenshtein doesn't count `ab → ba` as one edit (it's two). Use Damerau-Levenshtein if needed.

## Linked concepts

- [Search & Autocomplete category overview](../by-category/search-autocomplete.md)
- [Word Dictionary](word-dictionary.md) — trie basics
- [Search & indexing (HLD)](../../hld/fundamentals/search-indexing.md)

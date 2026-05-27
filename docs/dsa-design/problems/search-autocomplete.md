# Design Search Autocomplete System

!!! note "The disguise"
    Interviewer says: **"Design Google search suggestions."**
    They mean: **"Implement a Trie with DFS to collect candidates and a Heap to rank them."**

## Problem

Build `AutocompleteSystem(sentences, times)` with:

- Constructor: a list of historical sentences and their frequencies.
- `input(c: char) -> List[str]` — feed one character at a time. After each char (except `#`), return the **top 3 historical sentences** that have the current accumulated input as a prefix, ranked by frequency, breaking ties lexicographically.
- The `#` character ends the current sentence: record/update its frequency in the corpus and reset.

## What it really tests

- **Trie for prefix matching.** O(prefix length) navigation to the subtree of candidates.
- **DFS to collect matches in the subtree.**
- **Heap (size 3) to rank by `(−frequency, sentence)`.**
- **State across calls.** Each `input` keeps building the prefix.

## Approach

The data structure is a Trie where each *end-of-sentence* node stores the sentence's frequency.

**Why a Trie?** Each call to `input(c)` extends the current prefix by one character. With a Trie, we can keep a "current pointer" and move it down by one edge in O(1). Compare to a list-of-strings approach: every call would re-filter all sentences (O(N×L)).

**Top-K ranking.** From the prefix node, we DFS into the subtree and collect every `(sentence, freq)` pair. Then either:

- Sort all candidates and take the top 3 → O(C log C) where C = candidates in subtree.
- Use a Min Heap of size 3 → O(C log 3) = O(C).

For just top 3, a sort is fine; the heap-of-K becomes essential when K is larger.

**The sort key.** Higher frequency first, then alphabetical. In Python: `sorted(candidates, key=lambda x: (-x.freq, x.sentence))`.

**Handling `#`.** When the input char is `#`, end the sentence:

- Walk the trie (creating nodes as needed) for the accumulated sentence.
- Increment frequency on the end node.
- Reset state: `cur = root`, `prefix = ""`.

**State for invalid prefix.** If at some point `input(c)` reaches a non-existent child, we know there's nothing in the corpus for the current prefix — return `[]` and keep returning `[]` until `#` resets us. We track this with a `dead` flag so we don't have to recheck.

## Code sketch

```python
class TrieNode:
    __slots__ = ("children", "freq", "sentence")
    def __init__(self):
        self.children: dict[str, "TrieNode"] = {}
        self.freq: int = 0
        self.sentence: str | None = None

class AutocompleteSystem:
    def __init__(self, sentences: list[str], times: list[int]):
        self.root = TrieNode()
        for s, t in zip(sentences, times):
            self._insert(s, t)
        self.cur: TrieNode | None = self.root
        self.buf: list[str] = []

    def _insert(self, sentence: str, times: int) -> None:
        node = self.root
        for ch in sentence:
            node = node.children.setdefault(ch, TrieNode())
        node.freq += times
        node.sentence = sentence

    def _collect(self, node: TrieNode) -> list[tuple[int, str]]:
        results: list[tuple[int, str]] = []
        stack = [node]
        while stack:
            n = stack.pop()
            if n.sentence is not None:
                results.append((n.freq, n.sentence))
            stack.extend(n.children.values())
        return results

    def input(self, c: str) -> list[str]:
        if c == "#":
            sentence = "".join(self.buf)
            self._insert(sentence, 1)
            self.cur = self.root
            self.buf = []
            return []
        self.buf.append(c)
        if self.cur is not None:
            self.cur = self.cur.children.get(c)
        if self.cur is None:
            return []
        candidates = self._collect(self.cur)
        candidates.sort(key=lambda x: (-x[0], x[1]))
        return [s for _, s in candidates[:3]]
```

## Complexity

Let *P* = length of current prefix, *C* = sentences in the subtree under the prefix node, *L* = average sentence length.

- `input(c)`: **O(C × L + C log C)** for the DFS collect + sort. With a Min Heap of size K, **O(C log K + L per result)**.
- `input('#')` (sentence end): **O(L)** for the insertion.
- Space: **O(total characters across the corpus)**.

## Edge cases

- **Prefix not in trie.** Return `[]`. Don't crash on `None.children`.
- **Fewer than 3 candidates.** Return however many exist, in ranked order.
- **Same sentence inserted multiple times via `#`.** Accumulate frequency on the end node.
- **Empty sentence between two `#`s.** Most variants don't allow this; if they do, treat it as freq++ on the root with empty sentence (or skip).
- **Tie-breaking by sentence.** Strictly alphabetical (case-sensitive `<`), not by length.

## Linked concepts

- [Search & Autocomplete category overview](../by-category/search-autocomplete.md)
- [Search & indexing (HLD)](../../hld/fundamentals/search-indexing.md)
- [Word Dictionary](word-dictionary.md) — basic trie
- [Top K Frequent Elements](top-k-frequent.md) — the same heap-of-K technique

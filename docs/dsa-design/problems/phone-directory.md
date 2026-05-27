# Design Phone Directory

!!! note "The disguise"
    Interviewer says: **"Design a phone number autocomplete."**
    They mean: **"Implement a Trie where each node has 10 children (digits 0-9) instead of 26 letters."**

## Problem

Build a structure for storing phone numbers that supports:

- `addNumber(number)` — insert a number.
- `searchByPrefix(prefix)` — return all numbers in the directory that start with this digit prefix.

Optional extension: limit to top-K suggestions, often by call frequency.

## What it really tests

- **Trie variant for digits.** Same idea as a word trie, but the alphabet is `{0,...,9}` so children fit in a 10-element array.
- **DFS to collect all numbers under a prefix node.**
- **Realization that "phone autocomplete" is just "string autocomplete" with a smaller alphabet.**

## Approach

This is structurally identical to a [Word Dictionary](word-dictionary.md) trie — only the alphabet changes.

**Why use a list instead of a dict?** With only 10 possible children per node, an `int[10]` or `list[10]` of node references is denser and faster than a HashMap. Index by `int(ch)`. Many production phone systems use this exact trick because the constant factor wins matter.

**Insert walk.** Per digit, take child at index `int(digit)`; create if `None`. Mark `is_end` and optionally store the full number on the end node (handy for collection later).

**Prefix search → DFS-collect.**

- Walk the trie by the prefix digits. If any child is missing, prefix doesn't exist → return `[]`.
- From the prefix node, DFS into the subtree. Whenever you hit `is_end`, add the stored number to the results.
- Stop early if you've already collected K results (for top-K variants).

**Why store the full number on the end node?** Saves us from reconstructing the path. Costs O(L) extra space per number but makes the collection step trivial.

## Code sketch

```python
class DigitNode:
    __slots__ = ("children", "number")
    def __init__(self):
        self.children: list["DigitNode | None"] = [None] * 10
        self.number: str | None = None

class PhoneDirectory:
    def __init__(self):
        self.root = DigitNode()

    def addNumber(self, number: str) -> None:
        node = self.root
        for ch in number:
            i = int(ch)
            if node.children[i] is None:
                node.children[i] = DigitNode()
            node = node.children[i]
        node.number = number

    def searchByPrefix(self, prefix: str, limit: int = 10) -> list[str]:
        node = self.root
        for ch in prefix:
            i = int(ch)
            if node.children[i] is None:
                return []
            node = node.children[i]
        results: list[str] = []
        self._collect(node, results, limit)
        return results

    def _collect(self, node: DigitNode, out: list[str], limit: int) -> None:
        if len(out) >= limit:
            return
        if node.number is not None:
            out.append(node.number)
        for child in node.children:
            if child is not None:
                self._collect(child, out, limit)
                if len(out) >= limit:
                    return
```

## Complexity

Let *m* = prefix length, *N* = total digits stored, *C* = numbers in the subtree under the prefix.

- `addNumber(number)`: **O(|number|)**.
- `searchByPrefix(prefix, K)`: **O(m + min(C, K) × L)** where *L* is average number length. The DFS short-circuits once K results are collected.
- Space: **O(N)**.

## Edge cases

- **Non-digit input.** `int(ch)` will throw on `'+'` or `'-'`. Validate or strip first.
- **International prefixes.** Country codes vary in length; the trie handles this naturally — `1` and `91` just branch differently from the root.
- **Duplicate insertion.** Overwrites the `number` field on the same end node; harmless.
- **Empty prefix.** Returns the first K numbers in the directory (or all if K is unbounded). Useful for "show all."
- **Frequency-aware suggestions.** Replace the simple collection with a heap-of-K ordered by call count; same outer DFS.

## Linked concepts

- [Search & Autocomplete category overview](../by-category/search-autocomplete.md)
- [Word Dictionary](word-dictionary.md) — same trie, 26 children
- [Search Autocomplete System](search-autocomplete.md) — adds ranking by frequency

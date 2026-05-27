# Design Stream of Characters

!!! note "The disguise"
    Interviewer says: **"Given a dictionary of words, support `query(letter)` returning true if the last k letters in the stream form any dictionary word"**
    They mean: **"Build a *reversed* Trie of the dictionary and walk it as new characters arrive"**

## Problem

- `StreamChecker(words)`
- `query(letter)` — return `True` iff *any suffix* of the stream so far is a word in `words`.

## What it really tests

- Why naïve fails: after each char, checking every dictionary word against the suffix is O(N · maxLen) per query.
- The Trie insight: build a Trie of the words **reversed**. Then to check the current stream, walk the Trie using the *last* character, then the *second-to-last*, and so on. If you hit a node marked "end-of-word" at any point, return true.
- Why reversed? We always have the most recent character; we can extend a suffix to the left by reading older characters in reverse.

## Approach

- Insert each `word` into a Trie character-by-character in reverse.
- Maintain a list `stream` of characters seen so far.
- On `query(c)`, append `c` to `stream`, then walk the Trie node-by-node going right-to-left through `stream`. Stop early if there's no child for the current letter, or as soon as we land on `is_end`.

This is bounded by the length of the longest word — not the length of the stream — so each query is O(maxWordLen).

## Code sketch

```python
class TrieNode:
    __slots__ = ("children", "end")

    def __init__(self):
        self.children = {}
        self.end = False

class StreamChecker:
    def __init__(self, words: list[str]):
        self.root = TrieNode()
        self.max_len = 0
        for w in words:
            node = self.root
            for ch in reversed(w):
                node = node.children.setdefault(ch, TrieNode())
            node.end = True
            self.max_len = max(self.max_len, len(w))
        self.stream = []

    def query(self, letter: str) -> bool:
        self.stream.append(letter)
        node = self.root
        # walk newest -> oldest, up to longest word length
        start = len(self.stream) - 1
        stop = max(-1, start - self.max_len)
        for i in range(start, stop, -1):
            ch = self.stream[i]
            if ch not in node.children:
                return False
            node = node.children[ch]
            if node.end:
                return True
        return False
```

## Complexity

- `query`: O(min(len(stream), maxWordLen)).
- Inserting the dictionary: O(total chars).
- Space: O(total chars in dictionary).

## Edge cases

- Empty dictionary → every query returns `False`.
- Streaming non-letter characters: the Trie handles any character; the dictionary simply won't have a branch.
- Long-lived stream: cap the in-memory `stream` to `max_len` to keep memory bounded.
- Word longer than the stream so far → naturally handled; the loop terminates at the start of `stream`.

## Linked concepts

- [Data Stream problems](../by-category/data-stream.md)
- Tries also drive [Add and Search Words](./add-search-words.md) and [Search Autocomplete](./search-autocomplete.md). For the indexing big picture see [Search & Indexing fundamentals](../../hld/fundamentals/search-indexing.md).

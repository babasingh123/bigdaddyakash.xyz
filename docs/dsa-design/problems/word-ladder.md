# Design Word Ladder

!!! note "The disguise"
    Interviewer says: **"Find the shortest transformation sequence from `beginWord` to `endWord`"**
    They mean: **"Implement BFS on an implicit graph where words are nodes and one-letter-different words are neighbors"**

## Problem

Given `beginWord`, `endWord`, and `wordList`, find the length of the shortest transformation sequence such that:

- Only one letter changes per step.
- Each intermediate word is in `wordList`.

Return `0` if no path exists.

## What it really tests

- Recognizing this is **shortest path on an unweighted graph** ⇒ BFS.
- Building the adjacency on demand by generating "one-letter-off" neighbors using the alphabet, rather than precomputing all pairs (which would be O(N² · L)).
- Optional optimization: **bidirectional BFS** halves the search radius.

## Approach

Naïve approach: build the graph by comparing every pair of words → O(N² · L).
Better: for each word, generate the 25 · L candidates by swapping each character with every other letter and check membership in `set(wordList)` → O(N · 26 · L) per BFS layer.

BFS is the right algorithm because every edge cost is 1; the first time we reach `endWord` we know the path length is minimal.

### Example trace

```
beginWord = "hit", endWord = "cog"
wordList  = ["hot","dot","dog","lot","log","cog"]

Layer 0: hit
Layer 1: hot                       (hit -> hot)
Layer 2: dot, lot                  (hot -> dot, hot -> lot)
Layer 3: dog, log                  (dot -> dog, lot -> log)
Layer 4: cog                       (dog -> cog or log -> cog)

Length = 5 (5 words in the sequence).
```

### Bidirectional BFS

Run BFS from both ends, alternating to expand the smaller frontier. They meet in the middle in O(b^(d/2)) instead of O(b^d) — exponential speedup in practice.

## Code sketch

```python
from collections import deque

def ladderLength(beginWord: str, endWord: str, wordList: list[str]) -> int:
    words = set(wordList)
    if endWord not in words:
        return 0

    q = deque([(beginWord, 1)])
    visited = {beginWord}

    while q:
        word, steps = q.popleft()
        if word == endWord:
            return steps
        for i in range(len(word)):
            for c in "abcdefghijklmnopqrstuvwxyz":
                if c == word[i]:
                    continue
                nxt = word[:i] + c + word[i + 1:]
                if nxt in words and nxt not in visited:
                    visited.add(nxt)
                    q.append((nxt, steps + 1))
    return 0
```

## Complexity

- Time: O(N · 26 · L) where L = word length, N = words in list.
- Space: O(N · L) for the set + the queue.
- Bidirectional BFS shares the asymptotic worst case but typically runs much faster.

## Edge cases

- `endWord` not in dictionary → return 0.
- `beginWord == endWord` → return 1 in most variants; LeetCode treats "no transformation needed" as 0. Confirm with the interviewer.
- `wordList` with duplicates → the set dedupes.
- Very long words: the 26 · L generator dominates; precomputing patterns like `"h*t"` → list of words can be faster.

## Linked concepts

- [Graph & Network problems](../by-category/graph-network.md)
- Same BFS template as [Graph Shortest Path](./graph-shortest-path.md) (unweighted case) and [Social Network Connection](./social-network-connection.md).

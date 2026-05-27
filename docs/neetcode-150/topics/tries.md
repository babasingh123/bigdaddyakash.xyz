# Tries

> **3 problems** · prerequisites below

## What this topic teaches

A trie (prefix tree) is the answer to one specific question: "given a stream of queries, do any of them share a prefix with strings I already know about?" Each node stores a hash map (or fixed-size array, e.g. 26 slots for lowercase) of children plus an `is_end` flag. Insertion, lookup, and prefix-existence are all O(L) where `L` is the length of the query — *independent of the number of strings stored.* That property is what makes tries the right tool for autocomplete, spell-check, IP routing, and word-search-on-grids problems.

The conceptual leap is that a trie isn't just "a data structure for words" — it's the shape that lets you run **DFS / BFS over the dictionary itself**, pruning entire branches that can't possibly extend to a valid word. *Word Search II* is the canonical example: instead of running a fresh DFS for each word, you walk the grid once and consult the trie at every step, pruning whole subtrees when the current prefix has no children.

## Prerequisites

- Trie (Advanced Algorithms)

## Core patterns

### Pattern 1: Standard insert / search / startsWith

Each `TrieNode` has `children: dict[str, TrieNode]` and `is_end: bool`. `insert(word)` walks character-by-character, creating nodes when missing, then sets `is_end = True` on the last one. `search(word)` walks and checks `is_end` at the end; `startsWith(prefix)` walks and just returns `True` if we got there. *Implement Trie Prefix Tree* is exactly this.

### Pattern 2: Trie + wildcard DFS

When queries can contain wildcards (`.` matching any character in *Design Add And Search Words Data Structure*), `search` becomes a recursive DFS: at each step, if the character is `.`, recurse into all children; otherwise recurse into the matching child if present. Time is O(L) for pure queries and O(26ᴸ) worst-case for all-wildcard queries — still linear in practice because real queries have few wildcards.

### Pattern 3: Trie-guided grid DFS (prune dictionary, not grid)

For *Word Search II* (find all dictionary words in a grid), the naive approach is "DFS from every cell for every word" — too slow. Instead, **build a trie of the dictionary**, then DFS from every cell *once* while walking the trie in parallel. At each step you check `node.children[grid[r][c]]` — if it's missing, the entire branch is dead and you backtrack immediately. Mark visited cells with a sentinel and restore on return. Bonus: delete trie nodes after a word is found to avoid duplicate emissions and to allow more pruning.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Implement Trie Prefix Tree | Medium |
| 2 | Design Add And Search Words Data Structure | Medium |
| 3 | Word Search II | Hard |

## Tips

- **Use `dict` for `children`, not a fixed-size list of 26**, unless the input is guaranteed lowercase ASCII and you're optimising for tight performance. The dict version is shorter and handles Unicode.
- **For *Word Search II*, prune the trie itself.** After emitting a found word, optionally walk back up deleting now-childless trie nodes — this dramatically shrinks the search space for the remaining cells.
- **Store the *full word* on `is_end` nodes** in *Word Search II* instead of rebuilding it from the DFS path. Trades a tiny amount of memory for cleaner code.
- **Wildcards in tries are a recursive DFS, not a loop.** If you find yourself trying to implement `.` with iteration, you'll reinvent recursion badly.

## Linked concepts

- [Search Autocomplete (DSA Design)](../../dsa-design/problems/search-autocomplete.md) — trie of historical queries with frequency aggregation per node.
- [Add Search Words (DSA Design)](../../dsa-design/problems/add-search-words.md) — production version of the wildcard-DFS pattern.
- [Word Dictionary (DSA Design)](../../dsa-design/problems/word-dictionary.md) — same trie + wildcard structure, slightly different API.
- [Phone Directory (DSA Design)](../../dsa-design/problems/phone-directory.md) — trie of phone numbers / contact prefixes.
- [Stream of Characters (DSA Design)](../../dsa-design/problems/stream-of-characters.md) — **reversed** trie (insert words in reverse), match on a streaming suffix buffer.
- [Spell Checker (DSA Design)](../../dsa-design/problems/spell-checker.md) — trie with case-insensitive and vowel-error variants.

# Search & Autocomplete

Every "design a search box" problem is really a question about one data structure: the **Trie** (prefix tree). The interesting variations are about what you bolt onto the trie — a heap for ranking, DFS with backtracking for wildcards, or edit distance for fuzzy matching.

## The pattern

A Trie is a tree where each edge is a single character. Walking from the root spells out a prefix; nodes can be marked as "end of word." That gives you:

- **O(m)** insert and lookup, where *m* is the word length (independent of dictionary size).
- **O(m)** prefix matching — walk *m* edges, then collect all words in the subtree.
- A natural place to attach metadata per node (frequencies, child pointers, suggestions).

The variations across this category come from *what you do once you reach the prefix node*:

- **Autocomplete** → DFS the subtree, collect candidates, rank with a heap.
- **Wildcard search** (`.` matches any char) → DFS with backtracking; at each `.` try all children.
- **Plain dictionary** → just check `isEnd` after walking the word.
- **Numeric phone directory** → same trie, but each node has 10 children instead of 26.
- **Spell checker** → walk the trie with an *edit-distance budget*; allow skip/substitute moves.

The recurring move: the trie compresses the dictionary, and the algorithm on top decides how strictly to match.

## Problems

- [Search Autocomplete System](../problems/search-autocomplete.md) — Trie + DFS + Heap
- [Add and Search Words](../problems/add-search-words.md) — Trie + DFS with backtracking
- [Word Dictionary](../problems/word-dictionary.md) — Basic Trie (insert / search / startsWith)
- [Phone Directory](../problems/phone-directory.md) — Trie with 10 children per node
- [Spell Checker](../problems/spell-checker.md) — Trie + Edit distance + BFS

## Why interviewers love this category

A Trie is one of those data structures that's *just specialized enough* to be a real test. Plenty of candidates can describe it on a slide; far fewer can implement insertion and recursive search cleanly in 20 minutes.

Interviewers are watching for three things:

1. **Do you reach for a Trie at all?** A surprising number of candidates try to brute-force prefix matching with a HashSet of strings. That works for `contains` but is hopeless for `startsWith` over a large dictionary.
2. **Can you handle the recursion?** Wildcard search is a clean DFS-with-backtracking exercise. It separates people who *say* they know DFS from people who can actually write it under stress.
3. **Do you think about ranking?** "Autocomplete" without ranking is useless — Google doesn't show you alphabetical results. The moment you mention "top K suggestions," you're proving you understand the product, not just the data structure.

Strategy for any problem in this category: draw the trie for 3-4 sample words, then implement `insert` first, then the variant of `search` the problem asks for.

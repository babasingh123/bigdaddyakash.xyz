# Arrays & Hashing

> **9 problems** · prerequisites below

## What this topic teaches

Arrays and hash tables are the workhorse pairing of competitive coding. A hash set or hash map lets you trade O(n²) "for every element, scan all others" loops for a single O(n) pass: you store what you've seen, then ask "have I seen the complement?" instead of recomputing it. Most "find a pair / count duplicates / detect anagrams / group by signature" problems collapse to *choose a key, choose a value, walk the array once.*

The second idea this topic introduces is the **prefix sum / running aggregate** — precomputing cumulative information so each query is O(1) instead of O(n). Prefix sums (and their cousins: prefix products, prefix counts, prefix XOR) are the bridge between brute-force range queries and the more sophisticated sliding-window and segment-tree techniques that come later.

## Prerequisites

- Dynamic Arrays (Data Structures & Algorithms for Beginners)
- Hash Usage (Data Structures & Algorithms for Beginners)
- Hash Implementation (Data Structures & Algorithms for Beginners)
- Prefix Sums (Advanced Algorithms)

## Core patterns

### Pattern 1: Seen-set / complement lookup

One pass, one hash set. As you walk the array, ask "is `target - x` already in the set?" before adding `x`. This is *Two Sum*, *Contains Duplicate*, and the dedup half of *Longest Consecutive Sequence*. The mental cue is: "I want an O(1) yes/no on something I've already passed."

### Pattern 2: Bucket / signature grouping

Pick a canonical key per element — sorted characters for anagrams, frequency tuple for *Top K Frequent Elements*, `(row, col, box)` triples for *Valid Sudoku* — and group with `defaultdict(list)` or `defaultdict(int)`. Two elements collide iff they share the signature, so the hash map *is* the answer structure.

### Pattern 3: Prefix / suffix product (or sum)

When the answer at index `i` depends on "everything to the left" and "everything to the right" but you can't divide (e.g. zeros in *Product of Array Except Self*), compute two passes — once left-to-right, once right-to-left — and combine. The output array doubles as the prefix buffer to hit O(1) extra space.

### Pattern 4: Encode/decode and string-as-array

Many "string problems" are array problems in disguise once you commit to a serialization: length-prefixed encoding in *Encode and Decode Strings*, sorted-string keys in *Group Anagrams*. Choose your encoding so collisions are impossible and decoding is unambiguous.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Contains Duplicate | Easy |
| 2 | Valid Anagram | Easy |
| 3 | Two Sum | Easy |
| 4 | Group Anagrams | Medium |
| 5 | Top K Frequent Elements | Medium |
| 6 | Encode and Decode Strings | Medium |
| 7 | Product of Array Except Self | Medium |
| 8 | Valid Sudoku | Medium |
| 9 | Longest Consecutive Sequence | Medium |

## Tips

- **Default to a hash map before nested loops.** If your first instinct is two `for` loops, stop and ask whether one of them can become a hash lookup.
- **Pick keys carefully.** For anagrams, `tuple(sorted(s))` works but `tuple(Counter(s).items())` after sorting items is faster; for *Top K Frequent*, **bucket sort by frequency** beats a heap when `n` is large.
- **Longest Consecutive Sequence is the trap.** It looks like sorting (O(n log n)) but is O(n) via a hash set: only start counting at `x` when `x-1` is not in the set, so each run is walked once.
- **Prefix sums own their off-by-one.** Decide upfront whether `prefix[i]` means "sum of first `i`" or "sum up to and including `i`" — the rest of your code follows from that one choice.

## Linked concepts

- [Top K Frequent (DSA Design)](../../dsa-design/problems/top-k-frequent.md) — bucket-by-frequency, heap, and quickselect variants of the same idea.
- [Leaderboard (DSA Design)](../../dsa-design/problems/leaderboard.md) — hash map of player → score, with ordered structure layered on top for ranking queries.
- [Encode/Decode Strings (DSA Design)](../../dsa-design/problems/encode-decode-strings.md) — length-prefix serialization at production scale.
- [Range Sum Query (DSA Design)](../../dsa-design/problems/range-sum-query.md) — prefix sums as an API.

# Design Compressed String Iterator

!!! note "The disguise"
    Interviewer says: **"Design an iterator over a run-length-encoded string like `'a4b2c1'`"**
    They mean: **"Parse on demand and emit one character at a time without ever expanding the string"**

## Problem

Given a string of `<letter><count>` runs (counts can be multi-digit), implement:

- `next()` — return the next character, or `' '` if exhausted.
- `hasNext()` — `True` if more characters remain.

## What it really tests

- **Lazy** parsing — *don't* expand `"a1000000000b1"` into a billion-character string.
- Iterator state: cursor through the compressed string + remaining count of the current run.
- Handling multi-digit counts (`"a12"` is `a × 12`, not `a, 1, 2`).

## Approach

Keep:

- `s` — the compressed string.
- `i` — index of the *next letter* to process.
- `cur` — the last emitted letter (or sentinel).
- `count` — how many of `cur` are still owed.

`next()`:

1. If `count == 0`, advance `i`: read the letter at `s[i]`, then read consecutive digits as the new `count`.
2. Decrement `count`, return `cur`.

`hasNext()` is true iff `count > 0` or `i < len(s)`.

Why not pre-expand? Counts can be huge (the LeetCode constraints allow up to 10⁹). Lazy expansion is O(1) memory beyond the input.

Why store `cur` separately from advancing `i`? Because `hasNext()` must be cheap and side-effect-free.

## Code sketch

```python
class StringIterator:
    def __init__(self, compressedString: str):
        self.s = compressedString
        self.i = 0
        self.cur = " "
        self.count = 0

    def next(self) -> str:
        if not self.hasNext():
            return " "
        if self.count == 0:
            self.cur = self.s[self.i]
            self.i += 1
            n = 0
            while self.i < len(self.s) and self.s[self.i].isdigit():
                n = n * 10 + int(self.s[self.i])
                self.i += 1
            self.count = n
        self.count -= 1
        return self.cur

    def hasNext(self) -> bool:
        return self.count > 0 or self.i < len(self.s)
```

## Complexity

- `next` / `hasNext`: O(1) amortized (digit parsing happens once per run).
- Space: O(1) beyond the input string.

## Edge cases

- Empty input → both methods are correct: `hasNext` false, `next` returns `' '`.
- Count of zero (`"a0b3"`) — depending on spec, treat as "skip". The parser above moves past it on the next call.
- Very large counts → use Python's arbitrary-precision ints; in Java guard against overflow.

## Linked concepts

- [String & Pattern problems](../by-category/string-pattern.md)
- The same "expand on demand" pattern shows up in stream decompression and protocol parsers.

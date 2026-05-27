# String & Pattern

This category is the most varied — strings, iterators, undo/redo, ordered logs, versioned arrays. The thread that unifies them: **state management**. Each problem asks you to expose a clean API on top of a non-obvious internal representation.

## The pattern

There isn't one data structure here; there's a *design instinct*. The recurring move is:

- Pick an internal representation that makes the dominant operation cheap.
- Hide it behind a small, well-typed API.

Examples:

- **Encode/Decode** → Use a length-prefix scheme (`"5#hello"`), not a delimiter. The delimiter could appear in the data.
- **Compressed iterator** → Store the run-length pairs, not the expanded string. State = `(current_char, remaining_count, index_into_pairs)`.
- **Text editor undo/redo** → Two stacks. Every action pushes its inverse onto `undoStack`. Redo pops `redoStack` and re-applies.
- **Timestamp-ordered logger** → A TreeMap (sorted dict) by timestamp; out-of-order inserts still process in order.
- **Snapshot array** → Don't copy on every snap. Per index, store `(snap_id → value)` and binary-search.

The common thread is **lazy work**: don't materialize the whole expanded/copied state, store just enough to reconstruct what's asked for.

## Problems

- [Encode and Decode Strings](../problems/encode-decode-strings.md) — Length-prefix string processing
- [Compressed String Iterator](../problems/compressed-string-iterator.md) — Iterator + State management
- [Text Editor with Undo/Redo](../problems/text-editor-undo-redo.md) — Two Stacks
- [Logger with Timestamp Ordering](../problems/logger-timestamp-ordering.md) — TreeMap by timestamp
- [Snapshot Array](../problems/snapshot-array.md) — HashMap + TreeMap + Binary Search per index

## Why interviewers love this category

These questions test **API design under DSA constraints**. You're not just told *what* to compute; you're told *what methods to expose*, and you have to invent the internal state.

Interviewers are watching for:

1. **Lazy materialization.** The candidate who copies the whole array on `snap()` has missed the point. Snapshot Array is *the* lesson in "only store what changed."
2. **Iterator state.** Compressed String Iterator separates people who can write `next()` and `hasNext()` cleanly. The internal state has to satisfy a clear invariant after every call.
3. **Undo/redo as inverse operations.** Two stacks is easy; what's hard is realizing the redo stack must be *cleared* on a new action (otherwise redoing after editing creates inconsistency).
4. **Length-prefix encoding.** Encode/Decode is a deceptively simple problem — until you realize delimiters break on adversarial input. The correct answer (length + `#` + data) shows you've thought about edge cases.

The lesson: when an interview problem asks for a class with several methods, the *internal data structure choice* is the whole question.

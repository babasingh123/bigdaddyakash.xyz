# Design Text Editor with Undo/Redo

!!! note "The disguise"
    Interviewer says: **"Design the undo/redo feature of a text editor"**
    They mean: **"Use two stacks — one for past states, one for future states"**

## Problem

Support:

- `type(s)` / `delete(k)` — mutate the document.
- `undo()` — restore the previous state.
- `redo()` — re-apply the last undone action.

Any new edit invalidates the redo history.

## What it really tests

- The two-stack pattern is a textbook fit:
    - `undoStack`: snapshots (or inverse commands) of past states.
    - `redoStack`: snapshots that were popped off undo.
- LIFO ordering matches "undo most recent first" perfectly.
- Knowing **when to clear `redoStack`**: any fresh edit makes future history obsolete.

## Approach

Two design choices:

1. **Snapshot-based**: each operation pushes the full document state. Simple, but O(n) memory per edit.
2. **Command-based** (preferred): each operation pushes a struct describing what changed (insert vs delete + payload). Undo applies the inverse. O(k) memory per edit where k is the change size.

We use snapshot-based here for clarity; in production, prefer commands.

- `type` / `delete` push the *current* state onto `undoStack` before mutating, and **clear** `redoStack`.
- `undo` pops `undoStack` into `redoStack` and sets the current state to the popped value.
- `redo` does the reverse.

Why two stacks rather than a doubly-linked list with a cursor? Functionally equivalent — stacks are just simpler to reason about and code on a whiteboard.

## Code sketch

```python
class TextEditor:
    def __init__(self):
        self.buf = []            # current document as a char list
        self.undo_stack = []     # list[list[str]] snapshots
        self.redo_stack = []

    def _snapshot(self) -> None:
        self.undo_stack.append(list(self.buf))
        self.redo_stack.clear()

    def type(self, s: str) -> None:
        self._snapshot()
        self.buf.extend(s)

    def delete(self, k: int) -> None:
        self._snapshot()
        del self.buf[-k:]

    def undo(self) -> str:
        if not self.undo_stack:
            return "".join(self.buf)
        self.redo_stack.append(list(self.buf))
        self.buf = self.undo_stack.pop()
        return "".join(self.buf)

    def redo(self) -> str:
        if not self.redo_stack:
            return "".join(self.buf)
        self.undo_stack.append(list(self.buf))
        self.buf = self.redo_stack.pop()
        return "".join(self.buf)
```

## Complexity

- `type` / `delete`: O(n) for snapshot (the copy dominates); O(k) for command-based.
- `undo` / `redo`: O(n) snapshot, O(k) command-based.
- Space: O(n · history) snapshot, O(total edits) command-based.

## Edge cases

- `undo` with empty history → no-op (return current state).
- `redo` after a new edit → must return current state because the redo history was cleared.
- Deleting more characters than exist: cap to current length.
- Memory blow-up: cap the undo history (real editors keep e.g. 100 entries).

## Linked concepts

- [String & Pattern problems](../by-category/string-pattern.md)
- The two-stack template also drives [Browser History](./browser-history.md).

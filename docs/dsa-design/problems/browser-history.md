# Design Browser History

!!! note "The disguise"
    Interviewer says: **"Design a browser with back and forward buttons."**
    They mean: **"Implement two stacks (or a doubly linked list with a cursor)."**

## Problem

Build `BrowserHistory(homepage)` with:

- `visit(url)` — navigate to `url`. Clears any forward history.
- `back(steps)` — move back up to `steps` pages. Return the current URL.
- `forward(steps)` — move forward up to `steps` pages. Return the current URL.

The classic detail: visiting a new URL after some `back`s **erases the forward history** — just like a real browser.

## What it really tests

- **Two-stack pattern** (back stack + forward stack), or
- **Doubly linked list with a cursor** (current node).
- **Understanding that "new visit clears redo."** This is the bug interviewers probe for.
- **Stack operations** — push, pop, peek.

## Approach

Two viable solutions; both are O(1) per operation.

### Solution 1: Two stacks

- `back_stack` — pages behind the current one (top = most recent past page).
- `forward_stack` — pages ahead of the current one (top = next forward page).
- `current` — the currently displayed URL.

**Operations:**

- `visit(url)`: push `current` onto `back_stack`, set `current = url`, **clear `forward_stack`**.
- `back(steps)`: pop up to `steps` items from `back_stack`. Each popped page goes onto `forward_stack`. The last popped page becomes `current` (push the previous `current` to forward first).
- `forward(steps)`: mirror.

This is clean conceptually but slightly tricky to get the "current sits between the two stacks" invariant right.

### Solution 2: Array (or DLL) with a cursor — simpler

- `history: list[str]` — all pages visited so far on the *current timeline*.
- `idx: int` — the current page's index.
- `size: int` — logical length (we don't truncate the array; we just shrink `size` when `visit` clears forward history).

**Operations:**

- `visit(url)`: increment `idx`, write at `history[idx]`, set `size = idx + 1`. (Use append if `idx == len(history)`.) Anything past `size` is dead history.
- `back(steps)`: `idx = max(0, idx - steps)`. Return `history[idx]`.
- `forward(steps)`: `idx = min(size - 1, idx + steps)`. Return `history[idx]`.

This is the version I'd actually code in an interview — fewer moving parts, harder to mess up.

**Why two solutions?** Interviewers will sometimes ask "could you do this without a list, using only stacks?" — the answer is yes, with the two-stack version. Knowing both is a strong signal.

## Code sketch

The cursor approach:

```python
class BrowserHistory:
    def __init__(self, homepage: str):
        self.history: list[str] = [homepage]
        self.idx: int = 0
        self.size: int = 1

    def visit(self, url: str) -> None:
        self.idx += 1
        if self.idx < len(self.history):
            self.history[self.idx] = url
        else:
            self.history.append(url)
        self.size = self.idx + 1

    def back(self, steps: int) -> str:
        self.idx = max(0, self.idx - steps)
        return self.history[self.idx]

    def forward(self, steps: int) -> str:
        self.idx = min(self.size - 1, self.idx + steps)
        return self.history[self.idx]
```

The two-stack approach:

```python
class BrowserHistoryStacks:
    def __init__(self, homepage: str):
        self.current = homepage
        self.back_stack: list[str] = []
        self.forward_stack: list[str] = []

    def visit(self, url: str) -> None:
        self.back_stack.append(self.current)
        self.current = url
        self.forward_stack.clear()

    def back(self, steps: int) -> str:
        while steps > 0 and self.back_stack:
            self.forward_stack.append(self.current)
            self.current = self.back_stack.pop()
            steps -= 1
        return self.current

    def forward(self, steps: int) -> str:
        while steps > 0 and self.forward_stack:
            self.back_stack.append(self.current)
            self.current = self.forward_stack.pop()
            steps -= 1
        return self.current
```

## Complexity

For the cursor approach:

- `visit`: **O(1)** amortized (append).
- `back`: **O(1)**.
- `forward`: **O(1)**.

For the two-stack approach:

- `visit`: O(1) amortized, but **`forward_stack.clear()` is O(|forward_stack|)** in the worst case (amortized O(1) over the lifetime if you account for each pushed URL being cleared at most once).
- `back(k)`, `forward(k)`: **O(k)**.

Space: **O(total URLs visited on the current timeline)**.

## Edge cases

- **`back(huge)` when only a few pages exist.** Clamp at the homepage.
- **`forward(huge)` after no `back`.** Should be a no-op; cursor stays put.
- **`visit` immediately after a `back`.** This is the critical one — forward history must be wiped. The cursor approach handles this by setting `size = idx + 1`. The stack approach handles it by `forward_stack.clear()`.
- **Visiting the same URL twice.** Should still create a new entry (so back/forward make sense).

## Linked concepts

- [Cache & Memory category overview](../by-category/cache-memory.md)
- [Text Editor with Undo/Redo](text-editor-undo-redo.md) — same two-stack pattern with different semantics
- [LRU Cache](lru-cache.md) — also uses a DLL, but for recency instead of navigation

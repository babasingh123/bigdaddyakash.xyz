# Stack

> **6 problems** · prerequisites below

## What this topic teaches

A stack gives you O(1) "what did I see most recently?" which is exactly the question that nested / balanced / lookback problems keep asking. Three families of problems live here: **parsing/matching** (every push waits for its mirror pop — parentheses, RPN, decoders), **monotonic stacks** (you maintain an ordered stack to answer "next greater / next smaller" in amortized O(n)), and **stack-augmented data structures** (carrying an auxiliary stack of mins, maxes, or fleets alongside the main one to make queries O(1)).

The mental shift this topic forces is that a `for` loop with a stack is *not* the same as a `for` loop with an array — you commit to processing things in LIFO order, which is exactly when "the most recent unresolved thing" is the one that matters. Once that clicks, *Daily Temperatures*, *Car Fleet*, and *Largest Rectangle In Histogram* stop looking like three different problems.

## Prerequisites

- Stacks (Data Structures & Algorithms for Beginners)

## Core patterns

### Pattern 1: Bracket / parser stack

Push the opening symbol, pop and check on the closing one. Generalises to RPN evaluation (push operands, pop two on each operator) and *Min Stack* (push the running min alongside every value, so `pop` and `getMin` are both O(1)). The invariant is: *every item on the stack is waiting for a matching event*.

### Pattern 2: Monotonic stack

Maintain a stack whose values are strictly increasing (or decreasing) from bottom to top. When the new element violates the order, pop and process the popped elements — they've just found their "next greater" (or "next smaller") neighbour. *Daily Temperatures* (next warmer day) and *Largest Rectangle In Histogram* (each bar's first smaller neighbour on either side) are both monotonic-stack canonical examples. Amortised O(n) because each element is pushed and popped at most once.

### Pattern 3: Stack of "fleets" / merged states

*Car Fleet* uses a stack where each element represents an entire cluster of cars that have collapsed into one fleet. You iterate cars in order of decreasing starting position; if the next car's arrival time exceeds the fleet at the top of the stack, it forms a new fleet (push); otherwise it merges (don't push). The stack's height is the answer.

### Pattern 4: Two stacks for amortised O(1) operations

*Min Stack* keeps a parallel `minStack` so the running minimum after each push/pop is always at the top. The broader pattern — "augment the stack with whatever extra info would make the query O(1)" — also implements stack-based queues, max stacks, and undo histories.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Valid Parentheses | Easy |
| 2 | Min Stack | Medium |
| 3 | Evaluate Reverse Polish Notation | Medium |
| 4 | Daily Temperatures | Medium |
| 5 | Car Fleet | Medium |
| 6 | Largest Rectangle In Histogram | Hard |

## Tips

- **Push indices, not values, for monotonic-stack problems.** You almost always need the position to compute distances (*Daily Temperatures*) or widths (*Largest Rectangle*).
- **Decide increasing vs decreasing upfront.** "Next greater to the right" → strictly decreasing stack; "next smaller to the right" → strictly increasing stack. Write the rule down before you code the loop.
- **For *Largest Rectangle In Histogram*, append a sentinel `0` at the end.** It forces the stack to flush at the end so you don't need a separate cleanup loop.
- **For *Car Fleet*, sort by position descending and iterate** — that way the stack top is always the fleet "ahead," which is the only one a new car can catch.

## Linked concepts

- [Text Editor Undo/Redo (DSA Design)](../../dsa-design/problems/text-editor-undo-redo.md) — two stacks for forward/back history.
- [Browser History (DSA Design)](../../dsa-design/problems/browser-history.md) — back/forward stacks with truncation on visit.
- [Sliding Window Max (DSA Design)](../../dsa-design/problems/sliding-window-max.md) — monotonic deque, the double-ended cousin of the monotonic stack.

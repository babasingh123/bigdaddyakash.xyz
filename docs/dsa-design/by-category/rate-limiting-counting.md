# Rate Limiting & Counting

"Design a rate limiter" sounds like a distributed-systems question. In a coding interview, it almost never is — it's a **sliding window** problem with a queue of timestamps. Every problem in this category boils down to the same move: track events in time, drop events outside the window.

## The pattern

The shared mechanic is a *time-windowed structure*:

1. Each event carries a timestamp.
2. Use a **Queue** (FIFO) to hold events in the current window — head is oldest, tail is newest.
3. On every operation, **evict from the head** while `now - timestamp > windowSize`.
4. Answer queries by inspecting what's left (count, sum, average, etc.).

This is just the **sliding window** pattern from arrays, but the window is defined by *time* instead of index. Variations:

- **Rate limiter** → one queue per user (HashMap → Queue). Reject when queue size hits the limit.
- **Hit counter** → one shared queue; answer "how many in the last N seconds."
- **Logger rate limiter** → no queue needed; a HashMap of `message → lastSeenTimestamp` is enough.
- **Moving average** → fixed-size queue (count-based, not time-based) + a running sum.
- **Leaky bucket** → queue that drains at a constant rate; reject when full.

The trick is recognizing that almost all of these can be solved with a Deque (or `collections.deque` in Python) and constant-time amortized math. You rarely need anything fancier.

## Problems

- [Rate Limiter](../problems/rate-limiter.md) — Queue per user + Sliding Window
- [Hit Counter](../problems/hit-counter.md) — Queue with time-based eviction
- [Logger Rate Limiter](../problems/logger-rate-limiter.md) — HashMap of last-seen timestamps
- [Moving Average from Data Stream](../problems/moving-average.md) — Fixed-size queue + running sum
- [Leaky Bucket](../problems/leaky-bucket.md) — Queue with constant drain rate

## Why interviewers love this category

These problems are the canonical place to check whether you can reason about **time as a dimension** in a data structure, not just space. Most candidates have written sliding-window-on-array code; far fewer have written sliding-window-on-time-stream code.

The interviewer is testing:

1. **Do you pick a Queue/Deque, not a List?** A `list.pop(0)` is O(n). A `deque.popleft()` is O(1). The difference is the whole exercise.
2. **Do you evict lazily?** You don't need a background thread cleaning the queue. You evict at the start of every `record` and `query` call. This is the *amortized O(1)* insight.
3. **Do you know when you *don't* need a queue?** The Logger Rate Limiter is a trap — most candidates reach for a queue, when a single HashMap entry per message is plenty.

After this category, you should be able to convert any phrase like "rolling 5-minute window" into a deque-of-timestamps reflex.

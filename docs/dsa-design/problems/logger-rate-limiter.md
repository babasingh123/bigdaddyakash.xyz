# Design Logger Rate Limiter

!!! note "The disguise"
    Interviewer says: **"Design a logger that rate limits messages."**
    They mean: **"Implement a HashMap from message to its last-allowed timestamp."**

## Problem

Build `Logger` with:

- `shouldPrintMessage(timestamp, message) -> bool` — return `True` and record if this `message` hasn't been printed in the last 10 seconds. Otherwise return `False`.

Timestamps are non-decreasing.

## What it really tests

- **Recognizing when *not* to use a queue.** Many candidates over-engineer this with a sliding window. It doesn't need one.
- **HashMap with timestamp comparison.**
- **Optional cleanup** to keep memory bounded.

## Approach

The naive (and tempting) plan: maintain a deque of `(timestamp, message)` pairs, evict ones older than 10 seconds, and check if `message` is in the window. That's correct but expensive — O(window size) per call.

**The minimum sufficient structure: a HashMap.**

For each message, store the last timestamp it was *allowed* to print. On a new call:

- If `message` not in map, or `timestamp - map[message] >= 10`, allow it: update `map[message] = timestamp` and return `True`.
- Otherwise, return `False`.

That's O(1) per call. We never need to know what *other* messages are in the window — we only need to know about *this* message.

**Why does this work?** The question is per-message rate limiting. A different message and the current one don't interact. So per-message state is enough.

**Memory caveat.** The HashMap grows unbounded with distinct messages. Two pragmatic fixes:

1. **Periodic sweep.** Every so often, walk the map and delete entries where `now - timestamp >= 10`.
2. **Bucketed eviction.** Keep one HashMap per second over the last 10 seconds; rotate.

In the standard interview answer, just state the issue and note "in production, we'd add expiry." Don't over-engineer it during the coding portion.

## Code sketch

```python
class Logger:
    def __init__(self):
        self.last_printed: dict[str, int] = {}

    def shouldPrintMessage(self, timestamp: int, message: str) -> bool:
        prev = self.last_printed.get(message, float("-inf"))
        if timestamp - prev >= 10:
            self.last_printed[message] = timestamp
            return False if False else True  # explicit
        return False

    # Stable, simpler version:
    def shouldPrintMessage_clean(self, timestamp: int, message: str) -> bool:
        if message not in self.last_printed or timestamp - self.last_printed[message] >= 10:
            self.last_printed[message] = timestamp
            return True
        return False
```

For production-style hardening with bounded memory:

```python
from collections import OrderedDict

class LoggerBounded:
    def __init__(self, max_distinct: int = 100_000):
        self.last_printed: OrderedDict[str, int] = OrderedDict()
        self.cap = max_distinct

    def shouldPrintMessage(self, timestamp: int, message: str) -> bool:
        if message in self.last_printed:
            prev = self.last_printed[message]
            if timestamp - prev < 10:
                return False
        self.last_printed[message] = timestamp
        self.last_printed.move_to_end(message)
        if len(self.last_printed) > self.cap:
            self.last_printed.popitem(last=False)
        return True
```

## Complexity

- `shouldPrintMessage`: **O(1)** average (HashMap lookup + insert).
- Space: **O(distinct messages seen)** in the simple version; **O(cap)** in the bounded version.

## Edge cases

- **First time we see a message.** Always allowed. The `message not in map` branch covers this.
- **Exact boundary at 10 seconds.** Convention: `>= 10` means allowed (the message is now 10 seconds old). Match the problem's wording.
- **Same message at the same timestamp twice.** The first allows, updates the map; the second hits `delta = 0 < 10` → blocked.
- **Memory growth.** Worth mentioning to the interviewer even if you don't implement the bounded version.

## Linked concepts

- [Rate Limiting & Counting category overview](../by-category/rate-limiting-counting.md)
- [Rate Limiter](rate-limiter.md) — when you *do* need a sliding window
- [Rate limiting (HLD)](../../hld/fundamentals/rate-limiting.md)

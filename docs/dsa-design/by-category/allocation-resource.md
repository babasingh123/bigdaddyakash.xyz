# Allocation & Resource Management

"Design X allocation" is usually one of three things: an array of counters (easy), a TreeSet of free slots (medium), or bit manipulation for distributed IDs (hard).

## The pattern

This category groups problems where the operation is "give me an available resource" or "free this resource." The structures fall into three buckets:

- **Counters** → Plain array or HashMap. Use when slots are interchangeable.
- **Sorted free-set** → TreeSet (or Min Heap of free indices). Use when you need a *specific* slot (smallest available).
- **Bit manipulation** → Snowflake-style IDs: pack `timestamp | machine_id | sequence` into 64 bits. Use for distributed unique IDs.
- **Backtracking / enumeration** → Phone-number-to-letters: DFS over possible mappings.

The structure choice is dictated by *what "available" means*:

- "Any car of size large" → counter.
- "Smallest available seat number" → TreeSet `first()`.
- "A globally unique 64-bit ID right now" → bit-packed timestamp + machine ID.
- "All possible strings from these digit keys" → backtracking.

## Problems

- [Parking System](../problems/parking-system.md) — Array of counters
- [Seat Reservation System](../problems/seat-reservation.md) — TreeSet of available seats
- [Number Allocator](../problems/number-allocator.md) — TreeSet or Min Heap of freed numbers
- [Unique ID Generator](../problems/unique-id-generator.md) — Snowflake-style bit packing
- [Phone Number to Words](../problems/phone-number-to-words.md) — Backtracking / DFS

## Why interviewers love this category

This category quietly tests **matching structure to requirement**. The problems range from trivial (parking counter) to subtle (Snowflake IDs), and the signal is whether the candidate picks the *minimum sufficient* tool.

Interviewers want to see:

1. **No overengineering on Parking System.** It's three counters. A TreeMap is wrong (overkill). A class hierarchy is wrong (overengineered). Recognizing simplicity is a real skill.
2. **TreeSet for sorted-free-set problems.** Seat reservation needs the *smallest available* repeatedly — that's literally what `TreeSet.first()` is for. Candidates who scan a boolean array are O(n) per operation.
3. **Snowflake-style ID generation.** This is where interviewers test whether you've thought about distributed systems at all. Bit layout matters: 41 bits timestamp + 10 bits machine ID + 12 bits sequence is a classic. Locks/CAS on the sequence counter is the concurrency point.
4. **Backtracking template.** Phone Number → Letters is the canonical "I want to see your DFS template" problem. Clean code: a `dfs(index, current)` function with an explicit base case.

The thread across all five: **the smallest tool that gets the job done is usually the right answer.**

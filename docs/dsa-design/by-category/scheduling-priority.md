# Scheduling & Priority

"Design a scheduler" is just "use a priority queue." The interesting part is what you pair the heap with — a cooldown queue, a topological sort, a TreeMap of intervals, or a monotonic deque.

## The pattern

Scheduling problems are unified by one question: *what should I do next?* The answer is always "the highest-priority eligible thing." That's the definition of a **heap**.

The variations come from how you define *eligible*:

- **Pure priority** → Min/Max Heap, pop and process.
- **Priority + cooldown** → Heap + auxiliary queue that holds "cooling" items with their ready-time.
- **Priority + dependencies** → Topological sort first; feed ready tasks into the heap.
- **Interval-based** → TreeMap sorted by start time; check overlaps with `floorEntry` / `ceilingEntry`.
- **Sweep line** → sort events by time; use a heap to track currently-active items.
- **Sliding-window max** → Monotonic deque (related — it's a "lightweight heap" that only works because the window slides).

Two structures dominate this category: **PriorityQueue** when you need ordering by score, and **TreeMap** when you need ordering by *key* and range queries.

## Problems

- [Task Scheduler](../problems/task-scheduler.md) — Max Heap + Cooldown Queue
- [Meeting Scheduler](../problems/meeting-scheduler.md) — TreeMap of intervals + overlap detection
- [Meeting Rooms II](../problems/meeting-rooms-ii.md) — Sort + Min Heap of end times (line sweep)
- [CPU Task Scheduler](../problems/cpu-task-scheduler.md) — Topological Sort + Heap + Cooldown Queue
- [Stock Price Ticker](../problems/stock-price-ticker.md) — TreeMap or Monotonic Deque

## Why interviewers love this category

These problems separate candidates who *know about* heaps from candidates who *reach for them by reflex*. The signal interviewers want:

1. **Heap-first thinking.** The moment you hear "scheduling," "priority," "next deadline," or "minimum / maximum at each step," your default should be a heap. If you start with arrays-and-sort, you've already lost time.
2. **Composing heap + queue.** Cooldown problems are where this clicks. The heap holds *ready* tasks; a regular queue holds *cooling* tasks with their ready-time. You move things between them each tick. Two structures, both O(log n), clean code.
3. **Knowing the line-sweep pattern.** Meeting Rooms II is the cleanest example — sort all start/end events, sweep through, and a Min Heap of end times tells you how many rooms are live right now. This generalizes to *every* interval-overlap problem you'll see.

The mental model: a heap is "what to do next," and the *eligibility filter* on top of it is where the puzzle lives.

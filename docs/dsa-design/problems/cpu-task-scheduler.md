# Design CPU Task Scheduler

!!! note "The disguise"
    Interviewer says: **"Schedule tasks with dependencies and cooldowns."**
    They mean: **"Build a dependency graph (topological sort) and feed ready tasks into a Heap with a cooldown Queue."**

## Problem

Given tasks with:

- Pairwise dependencies (`a` must finish before `b`).
- A cooldown `n` — the same *task type* (or each task instance, depending on variant) cannot run within `n` ticks of itself.
- (Optional) a priority/duration per task.

Return the schedule (or just the total time) that respects both constraints.

## What it really tests

- **Topological sort** for dependencies.
- **Priority Queue** to pick the next ready task by some score (often higher priority first, or shorter duration first).
- **Cooldown Queue** to hold finished tasks until they're cool.
- **Composition of three classical patterns.**

## Approach

This problem layers three patterns. Build them up one at a time.

### 1. Resolve dependencies → topological order

- For each task `t`, count its `indegree` (number of unfinished prerequisites).
- A task is **runnable** when `indegree == 0`.
- When a task finishes, decrement the indegree of each of its successors. Any successor whose indegree drops to 0 becomes runnable.

This is Kahn's algorithm. Build an adjacency list and an indegree map up front.

### 2. Among runnable tasks, pick the best one → Heap

If runnable tasks have a priority (or duration, or some score), use a heap to always pick the best next one. In the LeetCode 1834 variant, the score is `(processingTime, taskIndex)` — shortest job first, tie-break by index.

If there's no priority and tasks just need *any* ordering, a regular queue (BFS) is enough.

### 3. Enforce cooldown → auxiliary FIFO queue

When a task starts (or finishes), it "cools" — put it on a cooldown queue with `ready_time = current_time + n`. Pop from the cooldown queue back into the ready heap once `current_time >= ready_time`.

The cooldown queue is FIFO because tasks added later have a later `ready_time` (assuming monotonic `current_time`).

### Putting it together

```
build graph + indegree
push all initially-runnable tasks onto the heap
time = 0
while heap or cooldown_q or unfinished:
    release any tasks from cooldown_q whose ready_time <= time
    if heap empty: advance time to next cooldown release; continue
    t = heap.pop()  # best runnable task
    execute t (advance time by t.duration)
    for each successor s: indegree[s] -= 1; if 0, push s onto heap (or cooldown_q if cooldown applies)
```

**Why heap + queue together?** The heap optimizes "which ready task to run next." The queue manages "when can a cooling task become ready again." They don't replace each other; they handle orthogonal constraints.

## Code sketch

A streamlined version that schedules by `(duration, index)` priority (LeetCode 1834 style; ignore dependencies for brevity, then layer them in):

```python
import heapq
from collections import defaultdict, deque

def schedule_with_deps_and_cooldown(
    tasks: list[tuple[int, int]],          # (duration, id)
    deps: list[tuple[int, int]],           # (prereq, dependent)
    cooldown: int,
) -> list[int]:
    n = len(tasks)
    successors: dict[int, list[int]] = defaultdict(list)
    indegree = [0] * n
    for p, d in deps:
        successors[p].append(d)
        indegree[d] += 1

    duration = {i: t[0] for i, t in enumerate(tasks)}
    heap: list[tuple[int, int]] = []     # (duration, task_id)
    cooldown_q: deque[tuple[int, int]] = deque()  # (ready_time, task_id)
    for i in range(n):
        if indegree[i] == 0:
            heapq.heappush(heap, (duration[i], i))

    time = 0
    order: list[int] = []
    finished = 0
    while finished < n:
        while cooldown_q and cooldown_q[0][0] <= time:
            _, tid = cooldown_q.popleft()
            heapq.heappush(heap, (duration[tid], tid))

        if not heap:
            time = cooldown_q[0][0]
            continue

        d, tid = heapq.heappop(heap)
        order.append(tid)
        time += d
        finished += 1
        for s in successors[tid]:
            indegree[s] -= 1
            if indegree[s] == 0:
                cooldown_q.append((time + cooldown, s))
    return order
```

## Complexity

Let *n* = number of tasks, *E* = number of dependency edges.

- Building graph: **O(n + E)**.
- Each task is pushed/popped on the heap once: **O(n log n)**.
- Cooldown queue operations: **O(n)** total.
- Total: **O((n + E) log n)**.
- Space: **O(n + E)**.

## Edge cases

- **Cycle in dependencies.** Topological sort impossible — return error / empty schedule. Detect by `finished < n` at the end with empty heap and queue.
- **No dependencies.** Becomes the pure [Task Scheduler](task-scheduler.md) problem.
- **No cooldown (`n = 0`).** Becomes a pure topological scheduling problem; cooldown queue stays empty.
- **Ties in priority.** Heap key should include a tiebreaker (often `id`) to ensure deterministic output.

## Linked concepts

- [Scheduling & Priority category overview](../by-category/scheduling-priority.md)
- [Task Scheduler](task-scheduler.md) — no dependencies, just cooldown
- [Meeting Rooms II](meeting-rooms-ii.md) — different scheduling problem, same heap technique

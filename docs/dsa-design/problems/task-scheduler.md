# Design Task Scheduler

!!! note "The disguise"
    Interviewer says: **"Design a task scheduler with cooldown periods."**
    They mean: **"Implement a Max Heap (by remaining count) plus a cooldown Queue."**

## Problem

Given a list of tasks (each is a character A-Z) and an integer `n`, return the minimum number of intervals needed to execute all tasks. The same task **cannot execute again within `n` intervals**; you can insert idle slots.

Example: `tasks = ["A","A","A","B","B","B"], n = 2` → `A B idle A B idle A B` → 8 intervals.

This is LeetCode 621 — frequently dressed up as "design a scheduler."

## What it really tests

- **Max Heap by frequency.** Always run the most-remaining task next (greedy choice).
- **Cooldown Queue.** Tasks just executed cool down for `n` intervals before becoming eligible.
- **Greedy reasoning.** Why does "always pick the most frequent eligible task" work?

## Approach

The greedy intuition: at every tick, run whichever runnable task has the highest remaining count. Tasks cool down for `n` intervals after running. Insert idle if nothing's runnable.

**Why is greedy correct?** If you fail to run the highest-count task now, you'll have to run it later, but every other task has fewer occurrences left, so they'll finish sooner and you'll be stuck with the high-count one running into a tail of idles. Always running the most-remaining task minimizes that tail.

**The two structures:**

- **Max Heap** keyed on remaining count. The top of the heap is whatever task is "most urgent."
- **Cooldown Queue** of `(ready_time, task_remaining_count)`. When you run a task, you don't immediately put it back in the heap — you put it in the cooldown queue with `ready_time = current_time + n + 1`.

**The main loop.**

```
time = 0
while heap or cooldown_q:
    time += 1
    # release any cooled-down tasks back to the heap
    while cooldown_q and cooldown_q[0].ready_time == time:
        count = cooldown_q.popleft().count
        heappush(heap, -count)
    if heap:
        c = -heappop(heap) - 1  # one execution
        if c > 0:
            cooldown_q.append(CooldownEntry(time + n, c))
    # else: idle tick
```

Python's `heapq` is a min-heap, so negate counts to simulate a max-heap.

**Why a queue and not another heap for cooldown?** Because cooldown entries are released in FIFO order (their `ready_time` is monotonically increasing). A queue with O(1) peek-and-pop is enough.

**The closed-form bound (sanity check):**

```
intervals = max(n_total, (max_freq - 1) * (n + 1) + count_of_max_freq)
```

It's worth deriving on the board if asked — but the heap + queue simulation always gives the right answer and is what most interviewers want to see.

## Code sketch

```python
import heapq
from collections import Counter, deque

def least_interval(tasks: list[str], n: int) -> int:
    counts = Counter(tasks)
    heap: list[int] = [-c for c in counts.values()]
    heapq.heapify(heap)
    cooldown: deque[tuple[int, int]] = deque()  # (ready_time, remaining_count)

    time = 0
    while heap or cooldown:
        time += 1
        if cooldown and cooldown[0][0] == time:
            _, c = cooldown.popleft()
            heapq.heappush(heap, -c)
        if heap:
            c = -heapq.heappop(heap) - 1
            if c > 0:
                cooldown.append((time + n, c))
    return time
```

For a *streaming* task-scheduler API (more "design"-ish), wrap this in a class and expose `add_task(t)` / `next_tick() -> task_or_idle`.

## Complexity

Let *T* = total tasks (sum of frequencies), *K* = number of distinct tasks (≤ 26 for A-Z).

- Per tick: **O(log K)** for heap push/pop.
- Total: **O(T log K)** = effectively **O(T)** since K is small.
- Space: **O(K)** for the heap, cooldown queue at most K entries.

## Edge cases

- **`n = 0`.** No cooldown → answer is just `T`. The heap can immediately reuse popped tasks.
- **One task type dominates.** The cooldown forces idles. The formula `(max_freq - 1) * (n + 1) + count_of_max_freq` becomes the binding bound.
- **All distinct tasks.** Heap is always non-empty, no idles needed; answer is `T`.
- **Empty input.** Return 0.

## Linked concepts

- [Scheduling & Priority category overview](../by-category/scheduling-priority.md)
- [CPU Task Scheduler](cpu-task-scheduler.md) — adds dependency graph
- [Meeting Rooms II](meeting-rooms-ii.md) — different scheduling problem, similar heap idea

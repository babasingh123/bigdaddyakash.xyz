# Design Unique ID Generator (Snowflake)

!!! note "The disguise"
    Interviewer says: **"Design a distributed unique ID generator"**
    They mean: **"Implement the Twitter Snowflake scheme: pack timestamp + machine ID + sequence into a 64-bit integer"**

## Problem

Generate 64-bit IDs that are:

- **Unique** across many machines without coordination.
- **Roughly time-ordered** (k-sortable) so they're DB-index friendly.
- **Compact** — fit in a 64-bit signed integer.

## What it really tests

- Awareness of why UUIDs alone aren't enough — they're 128 bits and not sortable.
- Bit-packing: layout, masks, shifts.
- Concurrency: a per-process sequence counter and a clock-going-backwards guard.

## Approach

The 64-bit layout (Twitter Snowflake):

```
 1 bit   |    41 bits         |    10 bits      |    12 bits
 sign    | ms since custom    | machine id      | sequence per ms
 (0)     |       epoch        | (1024 nodes)    | (4096 / ms / node)
```

- **41-bit timestamp** in milliseconds since a custom epoch → ~69 years of headroom.
- **10-bit machine ID** → up to 1024 distinct generators (often split 5 / 5 into datacenter / worker).
- **12-bit sequence** → 4096 IDs per machine per millisecond. If you hit 4096, busy-wait until the next ms.

Encoding:

```
id = (timestamp_ms << 22) | (machine_id << 12) | sequence
```

Concurrency: take a lock around the `(timestamp, sequence)` update so two threads on the same machine don't collide.

Clock skew: detect when the clock goes *backwards* and refuse to generate IDs (or wait), otherwise duplicates are possible.

## Code sketch

```python
import threading
import time

EPOCH_MS = 1577836800000   # 2020-01-01, customize per system

class Snowflake:
    def __init__(self, machine_id: int):
        assert 0 <= machine_id < 1024
        self.machine_id = machine_id
        self.last_ts = -1
        self.seq = 0
        self.lock = threading.Lock()

    def _now_ms(self) -> int:
        return int(time.time() * 1000) - EPOCH_MS

    def next_id(self) -> int:
        with self.lock:
            ts = self._now_ms()
            if ts < self.last_ts:
                # clock moved backwards — refuse rather than risk duplicates
                raise RuntimeError("clock moved backwards")
            if ts == self.last_ts:
                self.seq = (self.seq + 1) & 0xFFF      # 12-bit mask
                if self.seq == 0:                      # overflowed in this ms
                    while ts <= self.last_ts:
                        ts = self._now_ms()
            else:
                self.seq = 0
            self.last_ts = ts
            return (ts << 22) | (self.machine_id << 12) | self.seq
```

## Complexity

- `next_id`: O(1) amortized. Busy-wait only kicks in at >4096 requests/ms on a single machine.
- Space: O(1) per generator.

## Edge cases

- **Clock skew**: the most painful real-world bug. Some implementations wait; others raise; never silently allow duplicates.
- **Machine ID collisions**: enforced externally — Zookeeper, k8s pod ordinal, or a config service.
- **Decoding an ID**: `ts = id >> 22`, `mid = (id >> 12) & 0x3FF`, `seq = id & 0xFFF`.
- **64-bit signed in JS**: clients in JavaScript lose precision past 53 bits; emit IDs as strings for browser consumers.

## Linked concepts

- [Allocation & Resource problems](../by-category/allocation-resource.md)
- Big-picture context: [Distributed Systems fundamentals](../../hld/fundamentals/distributed-systems.md).

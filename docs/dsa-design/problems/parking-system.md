# Design Parking System

!!! note "The disguise"
    Interviewer says: **"Design a parking lot that hands out spots of various sizes"**
    They mean: **"Three integer counters. That's the whole problem."**

## Problem

`ParkingSystem(big, medium, small)` is constructed with capacities. `addCar(carType)` (1 = big, 2 = medium, 3 = small) decrements the matching counter and returns `True` if there was space, `False` otherwise.

## What it really tests

- Knowing **not to over-engineer**. No graph, no heap — just an array of three counters indexed by car type.
- Avoiding off-by-one in the decrement-on-success pattern.
- Recognizing that "design" doesn't always mean "fancy". This problem is a complexity sanity check more than an algorithm test.

## Approach

`slots = [0, big, medium, small]` (index 0 unused, so `carType` indexes directly). On `addCar(t)`:

```
if slots[t] > 0:
    slots[t] -= 1
    return True
return False
```

Why not a HashMap? An array of length 4 is the same big-O, less memory, and clearer.

## Code sketch

```python
class ParkingSystem:
    def __init__(self, big: int, medium: int, small: int):
        self.slots = [0, big, medium, small]   # 1-indexed by carType

    def addCar(self, carType: int) -> bool:
        if self.slots[carType] > 0:
            self.slots[carType] -= 1
            return True
        return False
```

## Complexity

- `addCar`: O(1).
- Space: O(1).

## Edge cases

- Zero capacity for some type → all `addCar` of that type return `False`.
- Invalid `carType` → out of bounds; in production validate the input.
- Concurrent calls in a real lot → wrap in a lock or use atomic decrements.

## Linked concepts

- [Allocation & Resource problems](../by-category/allocation-resource.md)
- The "counter array" trick recurs in any bounded-resource problem.

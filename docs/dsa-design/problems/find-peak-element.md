# Design Find Peak Element

!!! note "The disguise"
    Interviewer says: **"Find *any* peak (`a[i] > a[i-1]` and `a[i] > a[i+1]`) in O(log n)"**
    They mean: **"Binary search on slope — the unsorted array hides a usable invariant"**

## Problem

Return the index of any peak in `nums`. Boundaries `nums[-1]` and `nums[n]` are treated as `-∞`. Adjacent values are guaranteed distinct.

## What it really tests

- Spotting a **binary search opportunity in an unsorted array** — the "sorted-ness" we exploit is the *slope*, not the values.
- The insight: at `mid`, compare `nums[mid]` with `nums[mid + 1]`.
    - If `nums[mid] < nums[mid + 1]` → there is a peak strictly to the right. The array climbs from `mid`; the right boundary is `-∞`, so it must come back down somewhere.
    - Else → there is a peak at `mid` or to the left (symmetric argument).
- This halves the range each iteration → O(log n).

## Approach

```
lo, hi = 0, len(nums) - 1
while lo < hi:
    mid = (lo + hi) // 2
    if nums[mid] < nums[mid + 1]:
        lo = mid + 1
    else:
        hi = mid
return lo
```

The loop exits with `lo == hi` pointing at *a* peak.

### Example trace

```
nums = [1, 2, 1, 3, 5, 6, 4]

lo=0, hi=6, mid=3, nums[3]=3 < nums[4]=5  → lo = 4
lo=4, hi=6, mid=5, nums[5]=6 > nums[6]=4  → hi = 5
lo=4, hi=5, mid=4, nums[4]=5 < nums[5]=6  → lo = 5
lo == hi == 5. Peak at index 5 (value 6).
```

## Code sketch

```python
def findPeakElement(nums: list[int]) -> int:
    lo, hi = 0, len(nums) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if nums[mid] < nums[mid + 1]:
            lo = mid + 1
        else:
            hi = mid
    return lo
```

## Complexity

- Time: O(log n).
- Space: O(1).

## Edge cases

- Single element → it's a peak (both boundaries are `-∞`).
- Strictly increasing array → peak is the last index.
- Strictly decreasing array → peak is index 0.
- Plateaus (equal adjacent values) — the problem rules them out; if present, this algorithm can stall.

## Linked concepts

- [Data Stream problems](../by-category/data-stream.md) — categorized under stream / search algorithms in this guide.
- Same "binary search on an invariant rather than sorted order" idea: [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/).

# Design Phone Number to Words

!!! note "The disguise"
    Interviewer says: **"Given a phone number, return every letter combination it could spell on a T9 keypad"**
    They mean: **"Implement classic letter-combinations backtracking (DFS)"**

## Problem

Input: a digit string (digits 2–9 only). Each digit maps to 3–4 letters (`2 → abc`, `3 → def`, …, `7 → pqrs`, `8 → tuv`, `9 → wxyz`). Return all possible letter combinations.

(LeetCode 17.)

## What it really tests

- Recognizing **combinatorial generation** ⇒ backtracking.
- Building a recursion tree where each level fixes one digit's letter, and the leaves at depth `len(digits)` are the answers.
- Avoiding extra work: don't use Cartesian-product helpers if the interviewer asks for the DFS spelled out.

## Approach

DFS / backtracking template:

```
def backtrack(i, path):
    if i == len(digits):
        out.append("".join(path))
        return
    for ch in mapping[digits[i]]:
        path.append(ch)
        backtrack(i + 1, path)
        path.pop()
```

### Recursion tree for `"23"` (`2 → abc`, `3 → def`)

```
              ""
       /       |       \
      a        b        c
    / | \    / | \    / | \
   ad ae af bd be bf cd ce cf
```

Each leaf is a complete combination. The number of leaves equals the product of the per-digit counts (here 3 × 3 = 9).

Iterative alternative — BFS: start with `[""]`, then for each successive digit cross-product with that digit's letters. Same answer, no recursion stack.

## Code sketch

```python
def letterCombinations(digits: str) -> list[str]:
    if not digits:
        return []

    mapping = {
        "2": "abc", "3": "def", "4": "ghi", "5": "jkl",
        "6": "mno", "7": "pqrs", "8": "tuv", "9": "wxyz",
    }

    out: list[str] = []
    path: list[str] = []

    def backtrack(i: int) -> None:
        if i == len(digits):
            out.append("".join(path))
            return
        for ch in mapping[digits[i]]:
            path.append(ch)
            backtrack(i + 1)
            path.pop()

    backtrack(0)
    return out
```

## Complexity

- Time: O(4ⁿ · n) where n = number of digits. Each digit contributes ≤4 letters; copying each n-length result is O(n).
- Space: O(n) recursion depth + O(4ⁿ · n) output.

## Edge cases

- Empty input → return `[]`, *not* `[""]` (per the LeetCode spec).
- Input contains `0` or `1` → undefined; depending on spec, skip or treat as empty.
- Very long inputs: the answer set blows up exponentially. A common follow-up is "return only the k-th combination" — that becomes a base-conversion problem rather than backtracking.

## Linked concepts

- [Allocation & Resource problems](../by-category/allocation-resource.md) — this problem is grouped under allocation in the source guide, though the underlying algorithm is pure backtracking.
- Same backtracking template: Generate Parentheses, Permutations, N-Queens.

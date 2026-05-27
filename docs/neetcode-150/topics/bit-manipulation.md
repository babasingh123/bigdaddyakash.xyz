# Bit Manipulation

> **7 problems** · prerequisites below

## What this topic teaches

Bit manipulation is the smallest, most tactical topic on the list: each problem is essentially one or two **bit tricks** dressed up as a problem. The reward for learning these tricks is that several O(n)-via-hashing problems collapse to O(n) constant-extra-space via XOR or arithmetic on bits — interviewers love asking the "now do it without the hash set" follow-up, and the answer is almost always a bit trick.

The fundamental observations to internalise are: `x ^ x = 0`, `x ^ 0 = x`, and XOR is **commutative and associative** (so order doesn't matter — every pair cancels). That single property powers *Single Number*, *Missing Number*, and the XOR-encoding tricks in graph and array problems. The second cluster of tricks is around **counting / extracting set bits**: `x & (x - 1)` clears the lowest set bit, `x & -x` isolates it, and `bin(x).count('1')` is the Python shortcut you should know exists but be ready to implement by hand.

## Prerequisites

- Bit Operations (Data Structures & Algorithms for Beginners)

## Core patterns

### Pattern 1: XOR cancellation

XOR all elements together; anything that appears an even number of times cancels out, leaving the odd one. *Single Number* (every element appears twice except one) is the textbook example. *Missing Number* extends it: XOR all indices `0..n` together with all `nums`; the absent index is the only one not cancelling. Constant space, single pass.

### Pattern 2: Popcount (number of 1-bits)

*Number of 1 Bits*: loop while `n != 0`, do `n &= n - 1` and increment a counter. This clears the lowest set bit each iteration, so it runs in O(set-bits) instead of O(32). The naive shift-and-mask `n & 1` + `n >>= 1` version is fine too — both are interview-acceptable.

### Pattern 3: DP-on-bits (Counting Bits)

*Counting Bits* asks for `popcount(i)` for every `i` from `0` to `n`. Naive: call popcount n times → O(n log n). Better: `dp[i] = dp[i >> 1] + (i & 1)` — `i >> 1` drops the LSB, so `dp[i]` is "popcount of higher bits, plus the LSB." O(n). The other classic recurrence is `dp[i] = dp[i & (i - 1)] + 1`.

### Pattern 4: Bit-reverse in 32 bits

*Reverse Bits*: 32 iterations, each shifting the result left by 1 and OR-ing in the LSB of the input, then shifting the input right. Cleaner: swap halves, then quarters, then eighths… with masks like `0x55555555` (alternating bits) — the divide-and-conquer version that hardware does. The 32-loop version is fine in interviews.

### Pattern 5: Adder without `+`

*Sum of Two Integers*: `sum = a XOR b` (without carry), `carry = (a AND b) << 1`. Loop until `carry == 0`, repeatedly applying. Python complicates this because integers are unbounded — mask to 32 bits and handle the sign manually. C/Java versions are simpler.

### Pattern 6: Overflow-safe integer reversal

*Reverse Integer*: pop the last digit (`digit = x % 10; x //= 10`), push it onto the result (`res = res * 10 + digit`). Before each push, check if `res > INT_MAX // 10` (or equals it with a too-large incoming digit) — return `0` if so. Handle negative sign separately.

## Problems

| # | Problem | Difficulty |
|---|---------|------------|
| 1 | Single Number | Easy |
| 2 | Number of 1 Bits | Easy |
| 3 | Counting Bits | Easy |
| 4 | Reverse Bits | Easy |
| 5 | Missing Number | Easy |
| 6 | Sum of Two Integers | Medium |
| 7 | Reverse Integer | Medium |

## Tips

- **Memorise these identities:**
    - `x ^ x = 0`, `x ^ 0 = x` (XOR with self cancels, XOR with zero is identity)
    - `x & (x - 1)` clears the lowest set bit
    - `x & -x` isolates the lowest set bit
    - `(x >> i) & 1` reads the i-th bit
    - `x | (1 << i)` sets the i-th bit; `x & ~(1 << i)` clears it
- **Python's integers are arbitrary precision — mask to 32 bits manually** for problems like *Sum of Two Integers* and *Reverse Bits*. `x & 0xFFFFFFFF` is your friend.
- **For *Counting Bits*, the `dp[i] = dp[i >> 1] + (i & 1)` recurrence is the cleanest answer.** The "kill lowest bit" version (`dp[i] = dp[i & (i-1)] + 1`) is also fine and worth knowing.
- **For *Missing Number*, the XOR solution and the sum-of-arithmetic-series solution (`n*(n+1)/2 - sum(nums)`) are both O(1)-space O(n)-time.** XOR doesn't overflow; sum can. Mention both.

## Linked concepts

Bit manipulation rarely shows up directly in DSA-design problems, but bitmasking is sometimes used as an optimisation inside larger structures:

- [Range Sum Query (DSA Design)](../../dsa-design/problems/range-sum-query.md) — Binary Indexed Trees (Fenwick trees) use the `x & -x` lowest-set-bit trick to navigate.
- [Unique ID Generator (DSA Design)](../../dsa-design/problems/unique-id-generator.md) — bitfield-packed IDs (timestamp | worker | sequence) at production scale.

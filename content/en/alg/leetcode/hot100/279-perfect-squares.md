---
title: "LeetCode 279: Perfect Squares, Deriving Dynamic Programming from Remaining Sums"
date: 2026-10-08
draft: false
categories: ["LeetCode"]
tags: ["Hot100", "dynamic programming", "unbounded knapsack", "perfect squares", "LeetCode 279"]
description: "Start with a failed largest-square choice, then build recursion, memoization, and iterative dynamic programming to minimize the number of perfect squares."
keywords: ["LeetCode 279", "Perfect Squares", "dynamic programming", "unbounded knapsack", "minimum squares"]
---

## Problem Requirements

Given a positive integer `n`, return the minimum number of **positive perfect squares** whose sum is exactly `n`. A perfect square is the square of an integer, such as `1 = 1²`, `4 = 2²`, `9 = 3²`, or `16 = 4²`.

- The sum must equal `n` exactly.
- You may use the same perfect square any number of times.
- Return only the count, not the decomposition. The order of the terms does not affect the answer.
- Constraint: `1 <= n <= 10^4`.

```text
Input: n = 12
Output: 3
Explanation: 12 = 4 + 4 + 4 uses 3 perfect squares.
```

```text
Input: n = 13
Output: 2
Explanation: 13 = 9 + 4 uses 2 perfect squares.
```

This guide builds the solution in Python 3.9+. It assumes familiarity with functions, loops, and basic recursion, and develops the dynamic programming reasoning step by step.

## Step 1: First Answer the Zero Remainder

Does choosing the largest square first minimize the count? For `n = 12`, that choice produces:

```text
12 -> take 9 -> remaining sum 3 -> take 1 + 1 + 1
Total: 9 + 1 + 1 + 1, or 4 terms
```

But `4 + 4 + 4` uses only 3 terms. The current baseline is "always choose the largest square no greater than the remainder." It already fails on `12`: taking more in one choice does not guarantee using fewer terms overall.

We need to compare the remaining sums after different choices. Once one square is removed, what remains is the same kind of problem: how many squares are needed to make that smaller number? Name that question:

```text
best(x) = minimum number of perfect squares whose sum is x
```

First establish its stopping condition. If `x = 0`, the target is complete and no more terms are needed, so the answer is `0`. This zero is an internal state reached during a decomposition; the problem's input `n` is still at least `1`.

Implement only that completion condition for now, leaving an explicit gap for positive-sum candidates:

```python
def best(x: int) -> int:
    if x == 0:
        return 0

    raise NotImplementedError("Positive sums need a candidate transition")
```

Run these checks to distinguish the completed part from the missing part:

```python
assert best(0) == 0

try:
    best(1)
except NotImplementedError:
    pass
else:
    raise AssertionError("A positive sum should expose the missing transition")
```

We can now state what `best(x)` asks and execute the completion rule `best(0) = 0`. However, `best(12)` still raises an exception: we have not enumerated its square choices or compared their total counts. The next step fills that precise gap with a candidate transition.

## Step 2: Try Every Possible Final Square

For `12`, the final square could be `1`, `4`, or `9`. If it is `4`, the preceding terms must make `8`. Once we know `best(8)`, including that final `4` gives a candidate count of `best(8) + 1`.

The previous version has only the completion rule `best(0)` and raises an exception for every positive sum. Its missing piece is a transition that connects each possible final choice to a smaller remainder. Replace the `raise` with a candidate loop, and put `best` inside a function that accepts the original input `n`:

```python
def num_squares_recursive(n: int) -> int:
    def best(x: int) -> int:
        if x == 0:
            return 0

        minimum = x
        root = 1
        while root * root <= x:
            square = root * root
            minimum = min(minimum, best(x - square) + 1)
            root += 1

        return minimum

    return best(n)
```

`minimum = x` is the count of a genuinely feasible solution: `x` copies of `1` always sum to `x`. Every non-negative remainder is therefore reachable, so this problem needs neither `-1` nor an unreachable marker.

Starting `root` at `1` produces `1²`, `2²`, `3²`, and so on until the next square exceeds `x`. Every square is positive, so `x - square` is strictly smaller than `x`, and recursion can eventually reach `0`. Each smaller problem again considers all squares, allowing the same value to appear repeatedly.

The transition is:

```text
best(x) = min(best(x - square) + 1)
          over all perfect squares with 1 <= square <= x
```

The `+ 1` counts only the final square we just chose. Walk through the three candidates for `12`:

| Final square | Minimum count for the remainder | Include the final square | Candidate total |
| --- | --- | --- | ---: |
| `1` | `best(11) = 3`, from `9 + 1 + 1` | `3 + 1` | 4 |
| `4` | `best(8) = 2`, from `4 + 4` | `2 + 1` | 3 |
| `9` | `best(3) = 3`, from `1 + 1 + 1` | `3 + 1` | 4 |

The function compares all candidates and keeps `3`. This explains why following only the final-`9` branch is insufficient.

Run the small-input checks:

```python
assert num_squares_recursive(0) == 0
assert num_squares_recursive(1) == 1
assert num_squares_recursive(4) == 1
assert num_squares_recursive(12) == 3
assert num_squares_recursive(13) == 2
```

We now have the first complete recursive solution, which runs on small inputs and compares every possible final square. However, different paths repeatedly solve the same `best(x)`. The search tree grows quickly and recursive calls may become too deep, so this version is not yet suitable for the full constraint. The next step addresses the duplicated work first.

## Step 3: Cache Repeated Remaining Sums

Where does the same remainder appear twice? When the previous version evaluates `best(12)`, both paths below ask for `best(8)`:

```text
12 -> 8                      choose one 4
12 -> 11 -> 10 -> 9 -> 8      choose 1 each time
```

The paths reach `8` differently, but the question is identical: using all positive perfect squares, what is the minimum count for `8`? The path taken to reach this sum cannot change its answer.

The current recursive transition is already correct. Its problem is that it expands the same `best(8)` again. Caching completed results by the remainder `x` lets later calls reuse them. Add `@cache` to the inner function, leaving the candidate loop unchanged:

```python
from functools import cache


def num_squares_memo(n: int) -> int:
    @cache
    def best(x: int) -> int:
        if x == 0:
            return 0

        minimum = x
        root = 1
        while root * root <= x:
            square = root * root
            minimum = min(minimum, best(x - square) + 1)
            root += 1

        return minimum

    return best(n)
```

The first call for a particular `best(x)` runs the function and saves the result. Later calls with the same `x` read that result directly. Only `x` is needed as the key because the square choices are always the same. Each outer call creates a new inner function, so separate inputs do not share an old cache.

Run the earlier examples and compare both versions over a small input range:

```python
assert num_squares_memo(12) == 3
assert num_squares_memo(13) == 2

for n in range(31):
    assert num_squares_memo(n) == num_squares_recursive(n)
```

For a cache check, inspect the inner `best` function's `cache_info()` in a local test. One fresh call with `n = 12` produces:

```text
CacheInfo(hits=13, misses=13, maxsize=None, currsize=13)
```

The 13 misses mean that the states `0` through `12` are each actually solved once. The 13 hits mean that another 13 requests reused saved answers. The current size is the number of stored results. These counts check the mechanism without changing the function's integer-returning interface.

We now solve each remainder only once and store at most `n + 1` states. Recursively solving smaller problems while remembering their answers is called memoization. However, caching does not remove the initial call chain. The loop tries `1` first, so it still forms `best(n) -> best(n - 1) -> ... -> best(0)`. At `n = 10000`, that depth may exceed Python's default recursion limit. The next step keeps the transition while removing the recursive call stack.

## Step 4: Fill a Table from Smaller Sums

Caching removes duplicated work but does not remove the call stack. At `n = 10000`, even solving each state once can create roughly ten thousand nested calls along the path that tries square `1` first. The current baseline hides the states inside function calls; we need to place the answers for `0` through `n` directly in a table.

Convert the recursive function into a table function. `dp[x]` means "the minimum number of perfect squares needed to make exactly `x`." Initialize with `x` copies of `1`; this is a feasible upper bound for every state and carries forward the previous `minimum = x` meaning. When processing `x`, every `x - square` is smaller than `x` and has already been completed:

```python
def num_squares_table(n: int) -> int:
    dp = list(range(n + 1))
    dp[0] = 0

    for x in range(1, n + 1):
        root = 1
        while root * root <= x:
            square = root * root
            dp[x] = min(dp[x], dp[x - square] + 1)
            root += 1

    return dp[n]
```

The update is the same `best(x - square) + 1` from the previous version; the recursive call has become a read of the completed `dp[x - square]`. For example, `4` gets result `1`; at `8`, reading `dp[4]` gives `dp[8] = 2`; at `12`, reading `dp[8]` gives `dp[12] = 3`. The same `4` can therefore be used three times through `4 -> 8 -> 12`. This is how unlimited square reuse appears in the table update.

Treat each square as a denomination that can be used any number of times: that is the selection rule of unbounded knapsack. This version puts amounts in the outer loop and processes them in ascending order, so every smaller amount is ready and can be extended by the same square again.

The states from `0` through `13` should be:

```text
dp = [0, 1, 2, 3, 1, 2, 3, 4, 2, 1, 2, 3, 3, 2]
```

Run the table version and compare it with the memoized version:

```python
assert num_squares_table(12) == 3
assert num_squares_table(13) == 2
assert num_squares_table(9999) == 4
assert num_squares_table(10000) == 1

for n in range(101):
    assert num_squares_table(n) == num_squares_memo(n)
```

The final two assertions show that the iterative table handles the problem's upper bound without relying on the recursive call stack. It is not yet the `Solution.numSquares` method required by LeetCode, and it does not yet include the complete inductive correctness argument or complexity summary; those belong to the next step.

## Step 5: Freeze the Submission Interface

The previous section produced an iterative table that covers the problem's constraint. The remaining gap is delivery: LeetCode calls `Solution().numSquares(n)`, and we need to explain why each table entry is minimal and what resources the method uses.

Place the table logic from the previous section directly in the required method. The loops, initialization, and update rule stay the same:

```python
class Solution:
    def numSquares(self, n: int) -> int:
        dp = list(range(n + 1))
        dp[0] = 0

        for x in range(1, n + 1):
            root = 1
            while root * root <= x:
                square = root * root
                dp[x] = min(dp[x], dp[x - square] + 1)
                root += 1

        return dp[n]
```

Check it with this induction invariant: before processing `x`, every `dp[y]` for `0 <= y < x` is the minimum number of squares needed to make `y`. `dp[0] = 0` is the base case. For `x`, every possible final square satisfies `square <= x`, so `x - square < x` and `dp[x - square]` already has its minimum value. Taking `min(... + 1)` over all final choices therefore gives the minimum for `dp[x]`. After `n` is processed, the returned value is the required answer. Initializing with `x` copies of `1` supplies a feasible upper bound, so this problem has no unreachable branch.

For each `x`, the inner loop checks `floor(sqrt(x))` roots. The total time complexity is `O(n sqrt(n))`, and the table uses `O(n)` extra space. Squares are reused through the ascending state order; the code minimizes the count rather than counting distinct permutations.

Run the final boundary checks:

```python
solver = Solution()
assert solver.numSquares(1) == 1
assert solver.numSquares(2) == 2
assert solver.numSquares(3) == 3
assert solver.numSquares(4) == 1
assert solver.numSquares(12) == 3
assert solver.numSquares(13) == 2
assert solver.numSquares(43) == 3
assert solver.numSquares(9999) == 4
assert solver.numSquares(10000) == 1
```

An independent local checker that does not use this DP transition also compared every input from `1` through `200`; all results agreed.

Common mistakes are initializing `dp[0]` to `1`, omitting the `+1` for the final square, or making each square usable only once in an attempt to "deduplicate" choices. Each one changes the state meaning we have verified. As with [LeetCode 322: Coin Change]({{< relref "322-coin-change.md" >}}), we solve smaller amounts first and reuse their results for larger amounts; here the available denominations are perfect squares.

At this point the code, state meaning, correctness invariant, complexity, and submission interface form one complete checkpoint. Number-theory theorems, BFS as the submitted algorithm, and other language implementations are outside this tutorial's scope.

Problem statement: [LeetCode 279: Perfect Squares](https://leetcode.com/problems/perfect-squares/).

---
title: "LeetCode 152: Maximum Product Subarray, Deriving Two DP States from a Negative Counterexample"
date: 2026-10-08
draft: false
categories: ["LeetCode"]
tags: ["Hot100", "dynamic programming", "maximum product subarray", "LeetCode 152"]
description: "Start by enumerating nonempty contiguous ranges, then use a negative counterexample to derive maximum and minimum product states and a constant-space Python solution."
keywords: ["LeetCode 152", "Maximum Product Subarray", "dynamic programming", "maximum and minimum products"]
---

## Problem Requirements

Given an integer array `nums`, find a **nonempty contiguous subarray** with the greatest product and return that product.

Contiguous means the chosen elements must be adjacent: you cannot skip a number in between. Nonempty means you must choose at least one element. Return only the product, not the range or its length.

```text
Input: nums = [2, 3, -2, 4]
Output: 6
Explanation: The contiguous subarray [2, 3] has the greatest product, 2 * 3 = 6.
```

```text
Input: nums = [-2, 0, -1]
Output: 0
Explanation: You cannot skip 0 and treat -2 and -1 as a subarray. Choosing [0] gives 0.
```

The constraints are `1 <= len(nums) <= 20000` and `-10 <= nums[i] <= 10`. The problem guarantees that the product of any subarray fits in a signed 32-bit integer. The input is never empty, but it may contain negatives and zeros.

## Step 1: Enumerate Every Nonempty Contiguous Range

For `[2, 3, -2, 4]`, multiplying the entire array gives `-48`, whereas the answer is the product of the first two elements, `6`. The current "multiply all elements" approach is missing the choice the problem actually requires: where to start and where to stop.

Before trying to predict which range is best, build a direct correct version: enumerate every valid range, calculate its product, and retain the greatest value.

Fix a left endpoint `left` and move the right endpoint `right` one position at a time. Multiply the existing range product by the newly included `nums[right]` to obtain the extended range product. Each extension avoids multiplying the entire range again.

Add this complete enumeration function:

```python
def max_product_bruteforce(nums: list[int]) -> int:
    answer = nums[0]

    for left in range(len(nums)):
        product = 1
        for right in range(left, len(nums)):
            product *= nums[right]
            answer = max(answer, product)

    return answer
```

The two initial values serve different purposes:

- `answer = nums[0]` chooses the first element as a valid nonempty range. If the array is just `[-2]`, the answer must remain `-2`; an initial answer of `0` or `1` would incorrectly override it.
- `product = 1` is the multiplication starting point for each new left endpoint. The answer is updated only after multiplying by `nums[right]`, so the empty product `1` never becomes an answer candidate.

For example, fix `left = 0` in `[2, 3, -2, 4]`. The inner loop produces:

| `right` | Current range | `product` | `answer` |
| --- | --- | ---: | ---: |
| 0 | `[2]` | 2 | 2 |
| 1 | `[2, 3]` | 6 | 6 |
| 2 | `[2, 3, -2]` | -12 | 6 |
| 3 | `[2, 3, -2, 4]` | -48 | 6 |

The outer loop then tries `left = 1`, `2`, and `3`. Every nonempty contiguous range has exactly one pair of endpoints, so these loops cover all valid ranges.

Run these checks for the official examples, a three-element optimal range, and a negative singleton:

```python
assert max_product_bruteforce([2, 3, -2, 4]) == 6
assert max_product_bruteforce([-2, 0, -1]) == 0
assert max_product_bruteforce([-2, 3, -4]) == 24
assert max_product_bruteforce([-2]) == -2
```

This version answers the problem correctly, but its cost still grows too quickly. An array of length `n` has `n * (n + 1) / 2` nonempty contiguous ranges. At the maximum length of `20000`, that means `200010000` range extensions.

The current time complexity is `O(n²)`, with `O(1)` extra space. The next step must find intermediate results we can retain so that every left endpoint does not need another traversal of the remaining elements.

## Step 2: A Negative Number Forces Us to Keep Both Extremes

The previous version enumerates every range correctly, but ranges with neighboring right endpoints repeat work. A range ending at index `i` either contains only `nums[i]` or extends a nonempty range ending at `i - 1` by appending `nums[i]`. The second case can reuse a result from the previous position.

A first attempt might retain only "the greatest product ending at the current position." At each position, compare starting again with the current number against extending the previously best range:

```text
current ending maximum = max(current number, previous ending maximum * current number)
```

But on `[-2, 3, -4]`, that compression discards an essential candidate:

| Index | Current number | Only one ending maximum retained | Lost information |
| --- | ---: | ---: | --- |
| 0 | -2 | -2 | None |
| 1 | 3 | `max(3, -2 * 3) = 3` | The product `-6` of `[-2, 3]` |
| 2 | -4 | `max(-4, 3 * -4) = -4` | `-6 * -4 = 24` |

The greatest of those ending maxima is only `3`, whereas the brute-force version already verified that the correct answer is `24`. The sign of multiplication causes the failure: a smaller negative product may become the greatest positive product after multiplying by a new negative number.

We therefore need two states, and both must require the range to end **exactly at index `i`**:

- `max_end[i]`: the greatest product of a nonempty contiguous subarray ending at `i`.
- `min_end[i]`: the smallest product of a nonempty contiguous subarray ending at `i`.

Let the current number be `x = nums[i]`. The single-element range contributes `x`. When extending an old range, positive `x` preserves product order, negative `x` reverses it, and zero makes every extended product zero. We therefore do not need every old range product: the previous greatest and smallest products suffice.

Both new states choose from the same three candidates:

```text
x                       start a new range here
max_end[i - 1] * x       extend the previous greatest-product range
min_end[i - 1] * x       extend the previous smallest-product range
```

The greatest candidate becomes `max_end[i]`; the smallest becomes `min_end[i]`. This is the dynamic programming transition here: the problem ending at `i` builds on the already solved smaller problem ending at `i - 1`.

Replace the previous range enumeration with this paired transition:

```python
def max_product_table(nums: list[int]) -> int:
    size = len(nums)
    max_end = [0] * size
    min_end = [0] * size
    max_end[0] = min_end[0] = nums[0]

    for i in range(1, size):
        x = nums[i]
        previous_max = max_end[i - 1]
        previous_min = min_end[i - 1]

        max_end[i] = max(x, previous_max * x, previous_min * x)
        min_end[i] = min(x, previous_max * x, previous_min * x)

    return max(max_end)
```

At index `0`, the only valid ending range is `[nums[0]]`, so both states start at `nums[0]`. The loop fills every remaining position before it is read; the arrays' initial zeros never enter a candidate comparison. `previous_max` and `previous_min` are the values from the previous position, and both new states read that same old pair.

The earlier counterexample now produces:

| `i` | `x` | Three candidates | `max_end[i]` | `min_end[i]` |
| --- | ---: | --- | ---: | ---: |
| 0 | -2 | Only `[-2]` | -2 | -2 |
| 1 | 3 | `3, -6, -6` | 3 | -6 |
| 2 | -4 | `-4, -12, 24` | 24 | -12 |

Why return `max(max_end)` rather than `max_end[-1]`? The state requires a range to end at the current position, but the problem allows it to end anywhere. For `[2, 3, -2, 4]`, `max_end` is `[2, 6, -2, 4]`: the final local state is `4`, while the global answer is the earlier `6`.

Zero needs no separate branch. When `x = 0`, all three candidates are `0`, so both ending states become `0`. Later positions can still start again through the candidate `x`, while an earlier larger answer remains in an earlier `max_end` entry.

Run these checks:

```python
assert max_product_table([2, 3, -2, 4]) == 6
assert max_product_table([-2, 0, -1]) == 0
assert max_product_table([-2, 3, -4]) == 24
assert max_product_table([-2]) == -2
assert max_product_table([0, -2]) == 0
assert max_product_table([-1, -2, -3]) == 6
```

Each position now needs only a fixed number of comparisons and multiplications, reducing the time complexity to `O(n)`. The two arrays use `O(n)` extra space. This version handles negatives, zeros, and an optimal range ending anywhere; the remaining question is storage. Each transition reads only `i - 1`, so must we retain every older entry?

## Step 3: Keep Only the Previous Pair

The previous `max_end[i]` is read only by the next iteration, and `min_end[i]` has the same lifetime. Position `i + 1` never reads an earlier position, so most entries in the two arrays are already dead. One result cannot simply be deleted, though: the best range may end early, so we cannot wait until the loop finishes to scan the arrays for the global maximum.

First consider an easy-to-write but incorrect compression. If we update `max_end` first and then let `min_end` read it, the second update mixes the old maximum with the newly computed maximum. On `[-2, 3, -4, -1]`, that mistake eventually produces `96`, while the correct greatest product over all contiguous ranges is `24`.

At the start of each iteration, copy the previous position's two states into `previous_max` and `previous_min`. Compute both new states from that old snapshot, then fold the current maximum into a separate `answer`.

Replace the array version with this rolling version:

```python
def max_product_rolling(nums: list[int]) -> int:
    max_end = nums[0]
    min_end = nums[0]
    answer = nums[0]

    for i in range(1, len(nums)):
        x = nums[i]
        previous_max = max_end
        previous_min = min_end
        candidates = (x, previous_max * x, previous_min * x)

        max_end = max(candidates)
        min_end = min(candidates)
        answer = max(answer, max_end)

    return answer
```

Here `answer` replaces the final `max(max_end)` scan and stores the greatest ending-state value seen from the beginning through the current position. The loop uses `range(1, len(nums))`, so it does not create a new list such as `nums[1:]`; apart from a fixed number of integers and a tuple, no structure grows with the input.

To see why the snapshot matters, the faulty update is equivalent to:

```text
max_end = max(x, max_end * x, min_end * x)
min_end = min(x, max_end * x, min_end * x)  # max_end is already new here
```

On `[-2, 3, -4, -1]`, it incorrectly multiplies the newest maximum into the minimum again and pushes the result to `96`. The correct version reads the same old snapshot for both updates, so its ending states remain identical to the array version.

Run fixed boundary checks and a seeded differential check. The previously reviewed brute-force function is the oracle; each random array has length `1` through `8`, values from `-3` through `3`, and the loop compares 700 arrays:

```python
import random

assert max_product_rolling([2, 3, -2, 4]) == 6
assert max_product_rolling([-2, 0, -1]) == 0
assert max_product_rolling([-2, 3, -4]) == 24
assert max_product_rolling([-2]) == -2
assert max_product_rolling([0, -2]) == 0
assert max_product_rolling([-1, -2, -3]) == 6

random.seed(152)
for _ in range(700):
    size = random.randint(1, 8)
    values = [random.randint(-3, 3) for _ in range(size)]
    assert max_product_rolling(values) == max_product_bruteforce(values)
```

Only two ending states and one global answer remain. Time complexity stays `O(n)`, while extra space drops to `O(1)`. The only remaining delivery task is to place this verified rolling logic in LeetCode's required `Solution.maxProduct` method and provide the complete invariant argument.

## Step 4: Deliver the Method and Check the Invariant

The rolling function from the previous step already solves the state and space problems, but LeetCode requires the `Solution.maxProduct` entry point. Add no new transition here; place the already verified logic directly in the required method.

```python
class Solution:
    def maxProduct(self, nums: list[int]) -> int:
        max_end = nums[0]
        min_end = nums[0]
        answer = nums[0]

        for i in range(1, len(nums)):
            x = nums[i]
            previous_max = max_end
            previous_min = min_end
            candidates = (x, previous_max * x, previous_min * x)

            max_end = max(candidates)
            min_end = min(candidates)
            answer = max(answer, max_end)

        return answer
```

The method maintains this invariant: after processing index `i`, `max_end` is the greatest product among all nonempty contiguous subarrays that must end at `i`, `min_end` is the smallest such product, and `answer` is the greatest product among every valid nonempty subarray ending anywhere from index `0` through `i`.

- **Base case**: At index `0`, the only nonempty range ending there is `[nums[0]]`, so all three variables start at `nums[0]`.
- **Maintenance**: Every range ending at `i` either takes `nums[i]` alone or appends it to some nonempty range ending at `i - 1`. Multiplication by a fixed positive number preserves order, a negative number reverses it, and zero makes every product zero. The greatest and smallest extended products therefore come from the old two extrema. Including the singleton candidate `x` makes `candidates` sufficient to obtain both new extrema, and comparing the new `max_end` with `answer` keeps every possible endpoint covered.
- **Termination**: After the last index is processed, `answer` has compared the greatest ending product at every index, so it is the greatest product of any nonempty contiguous subarray.

Zero follows the same invariant without a special branch: when `x = 0`, all three candidates are `0`, so the current ending states reset to `0`; the next nonzero value can start a new range through the candidate `x`. Negative values also need no special branch because both previous extrema participate in the products, so sign reversal is covered by the same comparison. Initializing from `nums[0]` enforces the nonempty contract and never treats the empty product `1` as a candidate.

Run fixed checks through the final entry point, then compare it again with the reviewed brute-force function:

```python
import random

solution = Solution()
assert solution.maxProduct([2, 3, -2, 4]) == 6
assert solution.maxProduct([-2, 0, -1]) == 0
assert solution.maxProduct([-2, 3, -4]) == 24
assert solution.maxProduct([-2]) == -2
assert solution.maxProduct([0]) == 0
assert solution.maxProduct([0, -2, 0]) == 0
assert solution.maxProduct([-1, -2, -3]) == 6
assert solution.maxProduct([2, -5, -2, -4, 3]) == 24

random.seed(152)
for _ in range(700):
    size = random.randint(1, 8)
    values = [random.randint(-3, 3) for _ in range(size)]
    assert solution.maxProduct(values) == max_product_bruteforce(values)
```

Each element produces a fixed number of candidates, so time complexity is `O(n)`. Only three scalar states and one fixed-size tuple are retained, so extra space is `O(1)`. The problem guarantees a nonempty input, so the method does not add an empty-array branch that would change the contract.

Common mistakes are keeping only the greatest ending product, letting the `min_end` update read an already updated `max_end`, returning `max_end[-1]` as the global answer, or initializing the answer to `0`. They respectively lose sign reversals, corrupt the old-state transition, miss an earlier optimum, or mishandle an all-negative array.

The four checkpoints now form one code-growth chain: brute-force enumeration establishes the problem, two arrays preserve the extrema needed for sign reversal, rolling values remove dead history, and the final method adds only LeetCode's interface wrapper. Reference: [LeetCode 152](https://leetcode.com/problems/maximum-product-subarray/).

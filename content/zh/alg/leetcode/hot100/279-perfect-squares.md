---
title: "LeetCode 279：完全平方数，从剩余和推导动态规划"
date: 2026-10-08
draft: false
categories: ["LeetCode"]
tags: ["Hot100", "动态规划", "完全背包", "完全平方数", "LeetCode 279"]
description: "从最大平方数优先的失败例子出发，逐步构建递归、记忆化与迭代动态规划，求组成整数所需的最少完全平方数。"
keywords: ["LeetCode 279", "Perfect Squares", "完全平方数", "动态规划", "完全背包"]
---

## 题目要求

给定一个正整数 `n`，返回若干个**正的完全平方数**相加得到 `n` 时，所需的最少数量。完全平方数是整数的平方，例如 `1 = 1²`、`4 = 2²`、`9 = 3²`、`16 = 4²`。

- 和必须刚好等于 `n`。
- 同一个完全平方数可以使用任意次。
- 只返回数量，不需要返回具体分解；加数的先后顺序不影响答案。
- 约束：`1 <= n <= 10^4`。

```text
输入：n = 12
输出：3
解释：12 = 4 + 4 + 4，需要 3 个完全平方数。
```

```text
输入：n = 13
输出：2
解释：13 = 9 + 4，需要 2 个完全平方数。
```

下面使用 Python 3.9+ 逐步构建解法，适合已经理解函数、循环和基础递归，希望进一步理解动态规划的读者。

## Step 1：先确定剩余和为 0 的答案

先拿最大的平方数，会不会让数量最少？对 `n = 12`，这一选择得到：

```text
12 -> 先拿 9 -> 剩下 3 -> 再拿 1 + 1 + 1
总共：9 + 1 + 1 + 1，4 个
```

但是 `4 + 4 + 4` 只需要 3 个。当前的想法是“每次选不超过剩余和的最大平方数”，它在 `12` 上已经失败：单次拿得更多，不保证最后用得更少。

我们需要比较不同选择之后的剩余和。拿走一个平方数以后，剩下的仍然是同一种问题：用多少个平方数才能凑出这个更小的数？把这件事记成：

```text
best(x) = 相加得到剩余和 x 所需的最少平方数数量
```

先固定它的停止条件：如果 `x = 0`，目标已经完成，不需要再选，所以答案是 `0`。这里的 `0` 是分解过程中可能遇到的内部状态；题目给出的 `n` 仍然至少为 `1`。

本步只实现这个完成条件，正数的候选比较留一个明确的缺口：

```python
def best(x: int) -> int:
    if x == 0:
        return 0

    raise NotImplementedError("Positive sums need a candidate transition")
```

运行下面的检查，确认已经完成的部分与尚未实现的部分：

```python
assert best(0) == 0

try:
    best(1)
except NotImplementedError:
    pass
else:
    raise AssertionError("A positive sum should expose the missing transition")
```

现在能够说明 `best(x)` 在问什么，也能运行 `best(0) = 0` 的完成规则。但 `best(12)` 仍会抛出异常：还没有枚举正数可以选择哪些平方数，更没有比较这些选择的总数量。下一步就在这个缺口中加入候选转移。

## Step 2：枚举最后一个完全平方数

对 `12`，最后一个平方数可能是 `1`、`4` 或 `9`。如果最后选 `4`，前面的数还需要凑出 `8`；只要知道 `best(8)`，再加上当前这个 `4`，候选数量就是 `best(8) + 1`。

上一版只有 `best(0)` 的完成规则，任何正数都会抛出异常。缺少的正是“把每一种最后选择接到更小剩余和”的转移。现在把 `raise` 换成候选循环，并把 `best` 放进一个能接收原始输入 `n` 的函数：

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

`minimum = x` 是一个真实可行方案的数量：使用 `x` 个 `1` 总能凑出 `x`。所以本题所有非负剩余和都可达，不需要 `-1` 或“不可达”标记。

`root` 从 `1` 开始，依次产生 `1²`、`2²`、`3²`，直到下一个平方数超过 `x`。每次只允许正平方数，因此 `x - square` 严格小于 `x`，递归最终能到达 `0`。更小问题会再次枚举所有平方数，允许同一个值重复出现。

这条转移可以写成：

```text
best(x) = min(best(x - square) + 1)
          对所有 1 <= square <= x 的完全平方数比较
```

其中 `+ 1` 只数最后选的这一个平方数。对 `12`，慢速看三个候选：

| 最后一个平方数 | 剩余和的最少数量 | 加上最后一个 | 候选总数 |
| --- | --- | --- | ---: |
| `1` | `best(11) = 3`，`9 + 1 + 1` | `3 + 1` | 4 |
| `4` | `best(8) = 2`，`4 + 4` | `2 + 1` | 3 |
| `9` | `best(3) = 3`，`1 + 1 + 1` | `3 + 1` | 4 |

函数比较全部候选，所以保留 `3`。这也解释了为什么只走最后选 `9` 的一个分支不够。

运行小输入验证：

```python
assert num_squares_recursive(0) == 0
assert num_squares_recursive(1) == 1
assert num_squares_recursive(4) == 1
assert num_squares_recursive(12) == 3
assert num_squares_recursive(13) == 2
```

现在得到了第一个完整的递归解法，能够在小输入上运行并比较所有最后平方数。但它会沿不同路径反复求同一个 `best(x)`；搜索树增长很快，递归调用也可能过深，因此还不能把这个版本用于完整约束。下一步先处理重复计算。

## Step 3：缓存重复的剩余和

同一个剩余和会在哪里重复？上一版求 `best(12)` 时，以下两条路径都会问到 `best(8)`：

```text
12 -> 8                      选了一个 4
12 -> 11 -> 10 -> 9 -> 8      每次选 1
```

它们到达 `8` 的方式不同，但问题完全相同：允许使用所有正平方数，凑出 `8` 的最少数量是多少？先前如何到达这里，不会改变答案。

当前递归的转移已经正确，问题是它会重新展开同一个 `best(8)`。只要把已经求出的结果按剩余和 `x` 缓存，就可以复用答案。本步在内部函数上加入 `@cache`，候选循环保持不变：

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

第一次调用某个 `best(x)`，执行函数并保存结果；以后调用相同的 `x`，直接读取结果。缓存只需要 `x` 作为键，因为每次面对的平方数选择始终相同。内部函数每次调用外层入口时重新创建，因此不同输入不会共用旧缓存。

运行之前的小例子，再比较两版在一段小输入上的结果：

```python
assert num_squares_memo(12) == 3
assert num_squares_memo(13) == 2

for n in range(31):
    assert num_squares_memo(n) == num_squares_recursive(n)
```

检查缓存效果时，在本地测试中查看内部 `best` 的 `cache_info()`，一次独立的 `n = 12` 调用得到：

```text
CacheInfo(hits=13, misses=13, maxsize=None, currsize=13)
```

`misses = 13` 表示 `0` 到 `12` 这 13 个状态各自真正求解一次；`hits = 13` 表示另外 13 次询问直接复用了结果。`currsize = 13` 是已保存结果的数量。这些计数用于检查机制，不需要改变函数返回整数的接口。

现在相同剩余和只计算一次，最多缓存 `n + 1` 个状态。这样的“递归求更小问题，并记住答案”称为记忆化搜索。但缓存不会消除第一次计算时的调用链：当前循环先试 `1`，仍会形成 `best(n) -> best(n - 1) -> ... -> best(0)`。当 `n = 10000` 时，深度可能超过 Python 默认递归限制。下一步要保留转移，同时去掉递归调用栈。

## Step 4：按剩余和从小到大填表

缓存解决了重复计算，但没有解决调用栈。对于 `n = 10000`，即使每个状态只算一次，优先尝试平方数 `1` 的路径仍可能建立约一万层递归。当前 baseline 的问题是状态藏在函数调用中；我们需要让 `0` 到 `n` 的答案直接放在一张表里。

把递归函数改成表格函数。`dp[x]` 表示“刚好凑出 `x` 所需的最少完全平方数数量”。先用 `x` 个 `1` 填表，这既是每个状态的可行上界，也保留了上一节的 `minimum = x` 含义。计算 `x` 时，所有 `x - square` 都比 `x` 小，已经在前面完成：

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

这里的更新仍然是上一版的 `best(x - square) + 1`，只是把递归调用换成了已完成的 `dp[x - square]`。例如 `4` 的结果是 `1`；到了 `8`，读取 `dp[4]` 后可以得到 `dp[8] = 2`；继续到 `12`，读取 `dp[8]` 就得到 `dp[12] = 3`。同一个 `4` 因此可以连续使用三次：`4 -> 8 -> 12`。这正是本题“平方数可以重复使用”在表格更新中的体现。

把每个平方数看成一种面额，每种面额可以使用任意次，这就是完全背包的选择规则。当前写法以金额为外层循环，正序处理保证每个更小金额已经完成，因此能够继续接上同一个平方数。

对 `0` 到 `13` 填表，状态结果应为：

```text
dp = [0, 1, 2, 3, 1, 2, 3, 4, 2, 1, 2, 3, 3, 2]
```

运行表格版本，并和记忆化版本比较：

```python
assert num_squares_table(12) == 3
assert num_squares_table(13) == 2
assert num_squares_table(9999) == 4
assert num_squares_table(10000) == 1

for n in range(101):
    assert num_squares_table(n) == num_squares_memo(n)
```

最后两个断言说明：当前迭代表可以处理题目约束的上界，而不依赖递归调用栈。它还没有成为 LeetCode 要求的 `Solution.numSquares` 方法，也还没有给出完整的归纳正确性说明和复杂度收束；这些留到下一步。

## Step 5：冻结最终提交接口

上一节已经得到一个能覆盖题目约束的迭代表。现在的缺口是交付形式：LeetCode 会调用 `Solution().numSquares(n)`，并且我们还要说明为什么每个表项确实是最小数量、这段代码需要多少资源。

把上一节的表格逻辑直接放进要求的方法中。循环、初始化和更新规则都保持不变：

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

可以用一个归纳不变量检查它：处理 `x` 之前，`dp[y]` 对每个 `0 <= y < x` 都是凑出 `y` 的最少平方数数量。`dp[0] = 0` 是归纳起点。处理 `x` 时，任意最后平方数 `square` 都满足 `square <= x`，而 `x - square < x`，所以 `dp[x - square]` 已经是正确的最小值；在所有最后选择中取 `min(... + 1)`，就得到 `dp[x]` 的最小值。处理完 `n` 后，返回值就是题目要求的答案。初始化为 `x` 个 `1` 提供了一个可行上界，因此本题没有不可达分支。

对每个 `x`，循环会检查 `floor(sqrt(x))` 个根；总时间复杂度是 `O(n sqrt(n))`，表格占用 `O(n)` 额外空间。平方数按 `x` 的正序状态被重复使用，代码求的是最少数量，不是在统计不同排列。

最终边界检查：

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

本地还用不依赖这条 DP 转移的独立校验程序比对了 `1` 到 `200` 的全部输入，结果均一致。

常见误区是把 `dp[0]` 初始化成 `1`、遗漏最后平方数的 `+1`，或者为了“去重”让每个平方数只能用一次；这些都会改变已经验证的状态含义。与 [LeetCode 322：零钱兑换]({{< relref "322-coin-change.md" >}}) 一样，本题先解决更小金额，再把结果复用到更大金额；区别在于这里的可选面额由完全平方数组成。

到这里，代码、状态含义、正确性不变量、复杂度和提交接口已经连成一个完整检查点。进阶的数论定理、BFS 作为正式解法以及其他语言实现不在本教程范围内。

题目原文：[LeetCode 279：Perfect Squares](https://leetcode.com/problems/perfect-squares/)。

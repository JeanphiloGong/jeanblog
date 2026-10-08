---
title: "LeetCode 152：乘积最大子数组，从负数反例推导双状态动态规划"
date: 2026-10-08
draft: false
categories: ["LeetCode"]
tags: ["Hot100", "动态规划", "乘积最大子数组", "LeetCode 152"]
description: "先枚举非空连续区间，再用负数反例推导最大与最小乘积状态，逐步得到常数空间的 Python 解法。"
keywords: ["LeetCode 152", "乘积最大子数组", "动态规划", "最大最小乘积"]
---

## 题目要求

给定整数数组 `nums`，找到一个乘积最大的**非空连续子数组**，返回这个最大乘积。

“连续”意味着选中的元素必须挨在一起，不能跳过中间的数；“非空”意味着至少选一个元素。只需返回乘积，不需要返回区间或区间长度。

```text
输入：nums = [2, 3, -2, 4]
输出：6
解释：连续子数组 [2, 3] 的乘积最大，为 2 * 3 = 6。
```

```text
输入：nums = [-2, 0, -1]
输出：0
解释：不能跳过 0，把 -2 和 -1 当作一个子数组；选择 [0] 可得到 0。
```

约束为 `1 <= len(nums) <= 20000`、`-10 <= nums[i] <= 10`。题目保证任意子数组的乘积都在 32 位有符号整数范围内。输入不会为空，但可能包含负数和零。

## 第 1 步：先枚举每个非空连续区间

对 `[2, 3, -2, 4]`，如果直接把整个数组乘起来，得到的是 `-48`，而答案是前两个元素的乘积 `6`。因此，当前“把所有元素相乘”的做法缺少真正需要做的选择：从哪里开始，到哪里结束。

先不尝试猜哪个区间更好。最直接的正确办法，是枚举每一个合法区间，计算乘积，再保留最大的值。

固定左端点 `left` 后，让右端点 `right` 逐个向右移动。原来区间的乘积只需再乘上新加入的 `nums[right]`，就能得到扩展后区间的乘积。这样每次扩展不必从头乘一遍。

加入这个完整的枚举函数：

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

这里有两个不同的初始化：

- `answer = nums[0]`：先选第一个元素作为一个合法的非空区间。若数组只有 `[-2]`，答案必须是 `-2`，不能被初始值 `0` 或 `1` 盖掉。
- `product = 1`：这是每次重新选择左端点时的乘法起点。只有乘入 `nums[right]` 后才更新答案，所以它不会把空区间的乘积 `1` 当作候选答案。

例如，固定 `[2, 3, -2, 4]` 的 `left = 0`，内层循环依次得到：

| `right` | 当前区间 | `product` | `answer` |
| --- | --- | ---: | ---: |
| 0 | `[2]` | 2 | 2 |
| 1 | `[2, 3]` | 6 | 6 |
| 2 | `[2, 3, -2]` | -12 | 6 |
| 3 | `[2, 3, -2, 4]` | -48 | 6 |

之后再枚举 `left = 1`、`2`、`3`。每个非空连续区间都有唯一的左右端点，因此这些循环不会漏掉合法区间。

运行以下检查，验证官方示例、包含三个数的区间和负数单元素：

```python
assert max_product_bruteforce([2, 3, -2, 4]) == 6
assert max_product_bruteforce([-2, 0, -1]) == 0
assert max_product_bruteforce([-2, 3, -4]) == 24
assert max_product_bruteforce([-2]) == -2
```

这个版本已经能正确回答问题，但还需要解决规模上的代价。长度为 `n` 的数组共有 `n * (n + 1) / 2` 个非空连续区间：长度上限 `20000` 对应 `200010000` 次区间扩展。

当前时间复杂度是 `O(n²)`，额外空间是 `O(1)`。下一步需要找到可以保留的中间结果，让我们不必对每个左端点都重新遍历后面的元素。

## 第 2 步：负数迫使我们保留两个极值

上一版枚举了所有区间，答案正确，但相邻右端点对应的区间会做重复工作。比如一个区间以位置 `i` 结尾：它要么只有 `nums[i]` 一个元素，要么是在一个以 `i - 1` 结尾的非空区间后面接上 `nums[i]`。后者可以复用前一个位置的结果。

最初可能会想，只保存“以当前位置结尾的最大乘积”，每次比较“重新从当前数开始”和“延长之前最好的区间”：

```text
当前结尾最大值 = max(当前数, 前一个结尾最大值 * 当前数)
```

但在 `[-2, 3, -4]` 上，这个压缩办法会丢掉关键候选：

| 位置 | 当前数 | 只保留一个结尾最大值 | 丢失的信息 |
| --- | ---: | ---: | --- |
| 0 | -2 | -2 | 无 |
| 1 | 3 | `max(3, -2 * 3) = 3` | `[-2, 3]` 的乘积 `-6` |
| 2 | -4 | `max(-4, 3 * -4) = -4` | `-6 * -4 = 24` |

这些结尾最大值中的最大值只有 `3`，而暴力枚举已经验证正确答案是 `24`。问题出在乘法的符号：一个较小的负数乘积，乘上新的负数后，可能成为最大的正数乘积。

因此需要保留两个状态，而且都必须要求区间**恰好在位置 `i` 结尾**：

- `max_end[i]`：以 `i` 结尾的非空连续子数组中的最大乘积。
- `min_end[i]`：以 `i` 结尾的非空连续子数组中的最小乘积。

设当前数是 `x = nums[i]`。单元素区间贡献 `x`；延长旧区间时，若 `x` 为正，乘积大小顺序不变，若 `x` 为负，顺序反转，若 `x` 为零，所有延长结果都是零。所以不必保存所有旧区间的乘积，只需保留之前的最大值和最小值。

两个新状态都从同样的三个候选中选择：

```text
x                       从当前位置重新开始
max_end[i - 1] * x       接在之前最大乘积的区间后面
min_end[i - 1] * x       接在之前最小乘积的区间后面
```

取三个候选的最大值得到 `max_end[i]`，取最小值得到 `min_end[i]`。这就是这里的动态规划：把以 `i` 结尾的问题，建立在已经求出的、以 `i - 1` 结尾的较小问题上。

用这个成对的转移替换上一版的区间枚举：

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

初始化位置 `0` 时，唯一合法区间就是 `[nums[0]]`，所以两个状态都是 `nums[0]`。其余位置会被循环逐个填上，数组初始的零不参与候选比较。`previous_max` 和 `previous_min` 表示上一个位置的值，两个新状态都读取这同一对旧值。

对之前的反例，逐步更新结果为：

| `i` | `x` | 三个候选 | `max_end[i]` | `min_end[i]` |
| --- | ---: | --- | ---: | ---: |
| 0 | -2 | 只有 `[-2]` | -2 | -2 |
| 1 | 3 | `3, -6, -6` | 3 | -6 |
| 2 | -4 | `-4, -12, 24` | 24 | -12 |

最后为什么是 `max(max_end)`，而不是 `max_end[-1]`？状态要求区间在当前位置结尾，但题目允许它在任意位置结尾。对 `[2, 3, -2, 4]`，`max_end` 是 `[2, 6, -2, 4]`：最后一个状态是 `4`，全局答案却是此前的 `6`。

零也不需要额外分支。`x = 0` 时，三个候选都是 `0`，两个结尾状态都变成 `0`；后续仍能通过候选 `x` 从新位置开始，已经出现过的更大答案则保留在之前的 `max_end` 条目中。

运行这些检查：

```python
assert max_product_table([2, 3, -2, 4]) == 6
assert max_product_table([-2, 0, -1]) == 0
assert max_product_table([-2, 3, -4]) == 24
assert max_product_table([-2]) == -2
assert max_product_table([0, -2]) == 0
assert max_product_table([-1, -2, -3]) == 6
```

现在每个位置只做固定次数的比较和乘法，时间复杂度降为 `O(n)`，两个数组占用 `O(n)` 额外空间。这一版已经能处理负数、零和任意结尾的最优区间；接下来要处理的是存储：每次转移只读取 `i - 1`，更早的状态还需要全部留下来吗？

## 第 3 步：只保留上一个位置的状态

上一版的 `max_end[i]` 只会被下一轮读取，`min_end[i]` 也是如此。位置 `i + 1` 不会再访问更早的位置，因此两个数组保存的大部分条目都已经没有用途。不过有一个结果不能直接删掉：最优区间可能在很早的位置结束，不能等循环结束后才从数组里重新寻找全局最大值。

先看一个容易写错的压缩方式。如果先更新 `max_end`，再让 `min_end` 读取它，就会把“旧的最大值”和“新的最大值”混在一起。对 `[-2, 3, -4, -1]`，这种错误写法会在最后得到 `96`，而所有连续区间的正确最大乘积是 `24`。

所以每一轮开始时先复制上一个位置的两个状态：`previous_max` 和 `previous_min`。两个新状态都只从这份旧快照计算，计算完后再把当前的最大值加入独立的 `answer`。

把数组版本改成下面的滚动版本：

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

这里的 `answer` 替代了 `max(max_end)` 的最后扫描，保存从开头到当前位置为止见过的最大结尾状态。循环使用 `range(1, len(nums))`，不会创建 `nums[1:]` 这样的新列表；因此除了固定数量的整数和元组外，没有随输入增长的额外结构。

用错误更新顺序理解快照的必要性。错误代码相当于：

```text
max_end = max(x, max_end * x, min_end * x)
min_end = min(x, max_end * x, min_end * x)  # 这里的 max_end 已经是新值
```

在 `[-2, 3, -4, -1]` 上，它会错误地把最新的最大值再次乘入最小值，最终把答案推到 `96`。正确版本两次更新都读取同一轮开始时的旧快照，得到的结尾状态仍与数组版完全相同。

运行固定边界和随机差分检查。随机检查使用前面已经审查过的暴力函数作为 oracle；每个随机数组长度为 `1` 到 `8`，元素范围为 `-3` 到 `3`，共比较 700 个数组：

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

现在只保留两个结尾状态和一个全局答案，时间复杂度仍为 `O(n)`，额外空间降为 `O(1)`。下一步只剩交付问题：把已经验证的滚动逻辑放进 LeetCode 要求的 `Solution.maxProduct` 方法，并补齐完整的不变量说明。

## 第 4 步：交付最终方法并检查不变量

上一版的滚动函数已经解决了状态和空间问题，但 LeetCode 需要的入口是 `Solution.maxProduct`。这里不再增加新的转移，只把上一版已经验证的逻辑直接放进题目要求的方法中。

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

这个方法维护以下不变量：处理完位置 `i` 后，`max_end` 是所有“必须在 `i` 结束”的非空连续子数组中的最大乘积，`min_end` 是其中的最小乘积，而 `answer` 是从位置 `0` 到 `i` 为止所有合法非空连续子数组的最大乘积。

- **初始成立**：位置 `0` 只有一个非空结尾区间 `[nums[0]]`，所以三个变量都初始化为 `nums[0]`。
- **保持不变量**：以 `i` 结尾的每个区间要么只取 `nums[i]`，要么在某个以 `i - 1` 结尾的非空区间后接上它。乘以固定的正数保持大小顺序，乘以负数反转顺序，乘以零则都变成零，因此所有延长结果的最大值和最小值一定能从旧的两个极值中取得。再加入单元素候选 `x`，`candidates` 就足以求出两个新极值。`answer` 再与新的 `max_end` 比较，继续覆盖所有可能的结束位置。
- **循环结束**：最后一个位置处理完后，`answer` 已经比较过每个位置的最大结尾乘积，因此就是所有非空连续子数组中的最大值。

零自然包含在同一个不变量里：当 `x = 0` 时，三个候选都是 `0`，当前结尾状态重置为 `0`；下一个非零元素仍可通过候选 `x` 开始新的区间。负数也不需要单独分支，因为旧的最大和最小状态同时参与乘法，符号翻转已经被候选比较覆盖。非空约束由 `nums[0]` 初始化保证，代码没有把空区间的乘积 `1` 当作答案。

运行最终入口的固定检查，并再次与已经验证的暴力函数做差分：

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

循环每个元素只处理固定数量的候选，所以时间复杂度是 `O(n)`；只保留三个标量和一个固定大小的元组，额外空间复杂度是 `O(1)`。题目保证输入非空，因此没有添加空数组分支来改变方法契约。

常见错误包括：只保存最大结尾乘积、更新 `min_end` 时读取已经更新过的 `max_end`、把 `max_end[-1]` 当作全局答案，以及把答案初始化为 `0`。它们分别会漏掉负数翻转、破坏旧状态、漏掉较早结束的最优区间，或错误处理全负数组。

现在四个检查点已经连成一条代码演进链：暴力枚举证明问题，双数组保存符号翻转所需的两个极值，滚动变量去掉无用历史，最终方法只增加 LeetCode 的接口包装。参考：[LeetCode 152 题目](https://leetcode.com/problems/maximum-product-subarray/)。

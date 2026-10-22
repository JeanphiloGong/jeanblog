---
title: "LeetCode 1282：用户分组，从相同容量分桶到满组即输出"
date: 2026-10-10T10:00:00+08:00
draft: false
categories: ["LeetCode"]
tags: ["数组", "哈希表", "分组", "LeetCode 1282"]
description: "从用户编号与组大小的区别出发，先按容量收集再分块，逐步写出满组即输出的 Python 解法，解释列表引用陷阱、正确性与线性复杂度。"
summary: "相同组大小的人可以放在一起，但一个桶不一定就是一个组。用两个连续版本理解容量分组，并学会验证不唯一的答案。"
keywords: ["LeetCode 1282", "用户分组", "Group the People", "哈希表分组", "Python 列表引用"]
---

## 题目要求

给定整数数组 `groupSizes`，共有 `n` 个人，编号为 `0` 到 `n - 1`。`groupSizes[i]` 表示：**编号为 `i` 的人，所在组必须恰好有多少人**。

返回一组分组结果，每个组中存放的是人的编号，并满足：

- 每个人恰好出现在一个组中，不能重复，也不能遗漏。
- 一个组若有 `k` 个人，组内每个人对应的 `groupSizes[i]` 都必须等于 `k`。
- 组内顺序、组之间的顺序都不限；返回任意合法方案即可。
- 同组的人不要求在原数组中连续。

约束是 `1 <= n <= 500`、`1 <= groupSizes[i] <= n`。**题目保证至少存在一种合法分组。**

示例一：

```text
输入：groupSizes = [3,3,3,3,3,1,3]
一种输出：[[0,1,2],[5],[3,4,6]]
```

编号 `0、1、2` 都要求三人组，编号 `3、4、6` 也都要求三人组；编号 `5` 要求单独一组。相同的组大小可以对应多个组。

示例二：

```text
输入：groupSizes = [2,1,3,3,3,2]
一种输出：[[1],[0,5],[2,3,4]]
```

本文适合已经会写 Python 循环和列表操作、正在练习哈希表分组的读者，示例使用 Python 3.10+，预计阅读约 10 分钟。先不背模板，从“到底把什么放进组里”开始。

## Step 1：组里放的是编号，不是数组里的数值

先看一个更小的输入：`groupSizes = [2,1,2]`。

如果沿用“把相同数值放在一起”的直觉，会写出 `[[2,2],[1]]`。但这是错误答案：编号 `2` 出现了两次，编号 `0` 完全没有出现。问题不在排序，而在于混淆了**人的身份**与**这个人要求的容量**。

这一步只做一个区分：读取数组时，同时看下标和数值。

| 人的编号 | 要求的组大小 | 可以如何安排 |
| --- | --- | --- |
| 0 | 2 | 与编号 2 组成二人组 |
| 1 | 1 | 单独成组 |
| 2 | 2 | 与编号 0 组成二人组 |

因此 `[[0,2],[1]]` 是合法答案。逐项检查：`[0,2]` 的长度为 `2`，两位成员都要求 `2`；`[1]` 的长度为 `1`，该成员也要求 `1`。编号 `0、1、2` 恰好各出现一次。

由此先缩小问题：**暂时不安排全部分组，只把要求相同组大小的人的编号收集到一起。** 同组的人必须要求同一个组大小；如果有人要求两人组、有人要求三人组，他们不可能共享一个组。

现在我们能读懂输入，也能手工检查这个小答案。还缺的是：编号 `0` 和 `2` 中间隔着别人，程序要把先遇到的人记在哪里，才能在稍后找到同类？

## Step 2：按要求的组大小，暂存人的编号

当前基线还是手工配对。在 `[2,1,2]` 中，先遇到编号 `0`，再遇到编号 `1`，最后才遇到与 `0` 相容的编号 `2`。如果只盯着相邻位置，就无法保留前面已经找到的人。

因此增加一个字典 `buckets`：键是要求的组大小，值是目前遇到的、要求这个大小的人的编号列表。例如 `buckets[2] = [0,2]`，表示这两个人都需要二人组。这里把这样的列表叫作一个“桶”，只是为了说明它是暂存同类的容器。

先写一个只收集、不安排最终分组的可运行版本：

```python
def collect_by_size(group_sizes: list[int]) -> dict[int, list[int]]:
    buckets: dict[int, list[int]] = {}
    for person, size in enumerate(group_sizes):
        if size not in buckets:
            buckets[size] = []
        buckets[size].append(person)
    return buckets


assert collect_by_size([2, 1, 2]) == {2: [0, 2], 1: [1]}
assert collect_by_size([2, 2, 2, 2]) == {2: [0, 1, 2, 3]}
```

`enumerate` 同时给出下标 `person` 和数值 `size`：用 `size` 找桶，把 `person` 放进去。第二次遇到大小 `2` 时，沿用已有列表，不会覆盖先前保存的编号。

第一个断言解决了“同类不相邻”的问题。但第二个断言也揭示了边界：四个人都要求二人组，收集后得到一个四人桶。**同一个桶代表相同要求，不代表它已经是一个合法组。**

现在这个版本能把同类编号完整收集起来，还不能直接返回所有桶作为答案。下一步只需解决一个问题：一个桶的人数超过组容量时，该怎样拆？

## Step 3：把同一个桶按容量切成多个组

当前版本对 `[2,2,2,2]` 得到 `{2: [0,1,2,3]}`。四个人都需要二人组，把整个列表返回会让每个人所在的组都超员。

这里缺的不是新的分类方式，而是容量限制。对于大小为 `2` 的桶，每次取两个人即可：`[0,1]`、`[2,3]`。组内没有其他配对限制，因此不用继续搜索哪两个人“更合适”。

在上一版后面增加一个切分循环：保留收集代码，把原来的 `return buckets` 换成按容量切片并返回结果。

```python
def group_people_two_pass(group_sizes: list[int]) -> list[list[int]]:
    buckets: dict[int, list[int]] = {}
    for person, size in enumerate(group_sizes):
        if size not in buckets:
            buckets[size] = []
        buckets[size].append(person)

    result: list[list[int]] = []
    for size, people in buckets.items():
        for start in range(0, len(people), size):
            result.append(people[start:start + size])
    return result


assert group_people_two_pass([2, 2, 2, 2]) == [[0, 1], [2, 3]]
```

`start` 每次向后移动 `size`，切片 `people[start:start + size]` 取出这一组。上面的断言检查这个实现的具体切分结果，并不是要求原题只能接受这种排列。

会不会最后剩下一个不足 `size` 人的尾巴？在题目的有效输入范围内不会。假设要求大小 `k` 的人共有 `m` 个，这些人只能组成大小为 `k` 的组，不能被其他容量的组接收。既然题目保证有解，`m` 就必须是 `k` 的倍数。因此每个切片都恰好够人。

检查两个题目示例时，不需要对比唯一标准排列，而是逐组核对容量和编号覆盖：

| 输入 | 本实现返回 | 检查结果 |
| --- | --- | --- |
| `[3,3,3,3,3,1,3]` | `[[0,1,2],[3,4,6],[5]]` | 两个三人组、一个单人组，编号 `0..6` 各一次 |
| `[2,1,3,3,3,2]` | `[[0,5],[1],[2,3,4]]` | 二人组、单人组、三人组，编号 `0..5` 各一次 |

这已经是第一个完整正确的解法。哈希表操作按平均常数时间计算，收集过程为 `O(n)`；切片虽然复制列表，但所有切片加起来只复制 `n` 个编号，因此切分也是 `O(n)`，额外暂存空间为 `O(n)`，返回结果另占 `O(n)`。

它没有超时问题，但必须先保存所有人，再进入第二轮切分。对于 `[2,2,2,2]`，遇到编号 `1` 时，`[0,1]` 其实已经够人了。能不能当时就把这一组交出去，让桶只保留还没凑齐的人？

## Step 4：凑满一组就输出，但要换一张新列表

当前基线是“两轮法”：第一轮把所有编号放进桶，第二轮再切片。它已经正确，问题只是完成的组要等到扫描结束后才输出。

这在编号 `0、1` 都要求二人组时很直观：把 `1` 放进 `buckets[2]` 后，列表长度已经等于容量 `2`。此时可以立刻把它加入 `result`，并让 `buckets[2]` 重新代表下一组的暂存空间。

把上一版的收集循环替换为下面这个版本；只增加“长度达到容量”的分支，切分第二轮不再需要：

```python
def group_people(group_sizes: list[int]) -> list[list[int]]:
    buckets: dict[int, list[int]] = {}
    result: list[list[int]] = []

    for person, size in enumerate(group_sizes):
        if size not in buckets:
            buckets[size] = []
        buckets[size].append(person)
        if len(buckets[size]) == size:
            result.append(buckets[size])
            buckets[size] = []

    return result


assert group_people([2, 2, 2, 2]) == [[0, 1], [2, 3]]
assert group_people([3, 3, 3, 3, 3, 1, 3]) == [
    [0, 1, 2], [5], [3, 4, 6]
]
```

关键是最后一行 `buckets[size] = []`：它把字典里的桶**重新绑定**到一个新列表。不能写成下面这样：

```python
bad_bucket = [0, 1]
bad_result = []
bad_result.append(bad_bucket)
bad_bucket.clear()
assert bad_result == [[]]
```

`bad_result` 保存的是同一个列表对象的引用；清空暂存列表，也就清空了已经交付的结果。重新赋值 `buckets[size] = []` 则让旧列表继续归 `result` 所有，新列表负责下一组。这个生命周期变化不是装饰，而是“同一个大小可能需要多个组”时保持旧结果的必要条件。

按输入顺序走一遍 `[2,1,2,2,2,1]`：

| 读到的人 | 对应桶 | 是否满组 | 结果 |
| --- | --- | --- | --- |
| 0 (2) | `[0]` | 否 | `[]` |
| 1 (1) | `[1]` | 是 | `[[1]]` |
| 2 (2) | `[0,2]` | 是 | `[[1],[0,2]]` |
| 3 (2) | `[3]` | 否 | `[[1],[0,2]]` |
| 4 (2) | `[3,4]` | 是 | `[[1],[0,2],[3,4]]` |
| 5 (1) | `[5]` | 是 | `[[1],[0,2],[3,4],[5]]` |

因为合法输入中每个容量桶的人数最终都能被该容量整除，所以扫描结束时不会留下未完成的桶。这个版本减少了一轮显式切分，但总工作量仍是 `O(n)`；它不是把复杂度从 `O(n)` 降成更低的数量级，而是让“完成条件”在最早时刻发生。

现在我们有了无需第二轮的正确函数。还缺最后一步：把同一逻辑放进 LeetCode 要求的 `Solution.groupThePeople` 接口，并用不依赖输出顺序的检查来证明完整代码。

## Step 5：封装成提交代码，并证明它为什么正确

当前基线 `group_people` 已经完成算法。LeetCode 只要求一个类方法，因此这里只做接口封装，不再改变循环、完成条件或列表重新绑定规则：

```python
from typing import List


class Solution:
    def groupThePeople(self, groupSizes: List[int]) -> List[List[int]]:
        buckets: dict[int, list[int]] = {}
        result: list[list[int]] = []

        for person, size in enumerate(groupSizes):
            if size not in buckets:
                buckets[size] = []
            buckets[size].append(person)
            if len(buckets[size]) == size:
                result.append(buckets[size])
                buckets[size] = []

        return result
```

### 正确性：扫描时维护什么

对每个组大小 `k`，在扫描的任意时刻，`buckets[k]` 只保存“已经遇到、要求大小为 `k`，但还没有被交付到结果中的人”。每读到一个这样的人，就把他的编号追加进去；当长度达到 `k` 时，这个列表恰好是一个合法的 `k` 人组，于是加入结果，并换成一张新空列表。

这个不变量带来三点结论：

1. **组内容量正确。** 只有 `len(buckets[k]) == k` 才会输出，因此输出组长度就是组内每个人要求的大小。
2. **没有重复或遗漏。** 每个编号只在 `enumerate` 中加入一次；加入结果后旧列表不再写入，后续编号进入新列表。于是每个人恰好属于一个输出组。
3. **扫描结束时不会残留半组。** 对要求大小为 `k` 的所有人，题目保证存在合法答案；他们不能和其他大小的人混组，所以人数必然是 `k` 的倍数。每凑满一批就输出，最后不会留下不足 `k` 人的桶。

### 复杂度

每个人只被读取、追加和输出一次。按哈希表操作的平均 `O(1)` 计算，时间复杂度为 `O(n)`。桶和返回结果都保存编号，辅助空间与输出空间都为 `O(n)`；这里的“一次扫描”是代码组织上的改进，不意味着比 `O(n)` 更快。

### 如何验证不唯一的答案

不要把返回结果和某一个固定排列直接比较。可以用下面的检查条件：

```python
def validate(group_sizes: list[int], groups: list[list[int]]) -> None:
    flat = [person for group in groups for person in group]
    assert sorted(flat) == list(range(len(group_sizes)))
    assert all(
        group_sizes[person] == len(group)
        for group in groups
        for person in group
    )


for sizes in ([3, 3, 3, 3, 3, 1, 3], [2, 1, 3, 3, 3, 2], [1], [2, 2, 2, 2]):
    validate(sizes, Solution().groupThePeople(sizes))
```

还应覆盖 `n = 500` 的边界，以及由若干完整小组随机打乱得到的合法输入。验证的是“每个人出现一次、每组容量匹配”，而不是组的排列顺序。

## 小结

这道题的关键不是把数组排序，而是先区分两个角色：数组的**下标是人**，数组的**数值是这个人要求的组容量**。沿着这个区分，代码自然经历了三个版本：

1. 用按容量分桶保存同类编号；
2. 收集完成后，每次按容量切出一个合法组；
3. 扫描过程中一旦凑满就输出，并用新列表承接下一组。

最后一个细节尤其值得迁移到工程代码：当一个列表已经交给结果集合后，不要原地清空它来复用；重新绑定一个新列表，才能保证已交付的数据不被后续写入影响。

## 练习与延伸

- 把 `buckets` 改成 `collections.defaultdict(list)`，比较初始化逻辑是否更清楚。
- 写一个不依赖排序的覆盖检查，思考如何在约束扩大时减少验证成本。
- 对比 [LeetCode 49：字母异位词分组](https://leetcode.com/problems/group-anagrams/)：两题都先构造分组键，但本题的键直接来自输入的容量，而且还多了“一个桶可以拆成多个组”的完成条件。

参考：[LeetCode 1282 官方题目](https://leetcode.com/problems/group-the-people-given-the-group-size-they-belong-to/)。

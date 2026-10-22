---
title: "LeetCode 49：字母异位词分组，从逐组比较到构造分组键"
subtitle: "先判断两个词是否同组，再用排序结果和字符计数建立索引"
date: 2026-10-10T10:00:00+08:00
draft: false
summary: "用 Python 从两个字符串的异位词判断出发，先写出逐组扫描的正确解，再推导排序键与字符计数键，理解哈希表分组为什么成立。"
categories: ["LeetCode"]
tags: ["Hot100", "哈希表", "字符串", "LeetCode 49"]
keywords: ["字母异位词分组", "Group Anagrams", "分组键", "字符计数", "Python"]
readingTime: 10
---

## 题目要求

给定字符串数组 `strs`，把字母异位词放进同一组，返回所有分组。两个词互为字母异位词，意味着它们包含的字母及每个字母的出现次数相同，只是排列顺序可以不同。

题目示例：

```text
输入：strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
一种合法输出：[["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]
```

组与组之间、组内字符串之间的顺序都不影响答案。但输入中的每一次出现都必须保留：`["a", "a"]` 应得到 `[["a", "a"]]`，不能去重。

约束是 `1 <= len(strs) <= 10^4`，每个字符串长度在 `0..100`，字符仅含小写英文字母。空字符串是合法输入，例如 `[""]` 的结果是 `[[""]]`。

本文使用 Python 3.10+，适合已经会写循环、希望理解“按什么依据分组”的读者。

## Step 1：先缩小问题，判断两个词能不能放在一起

先不处理整个数组，只回答一个更小的问题：**给定两个词，它们是否应该同组？**

最直接的字符串比较不够用：`"eat" != "tea"`，但它们应该同组。能不能只看出现过哪些字母？下面这个自定义反例说明也不行：

```text
aab：a 出现 2 次，b 出现 1 次
abb：a 出现 1 次，b 出现 2 次
```

两者的字符集合都是 `{"a", "b"}`，却不能互相重排得到。集合丢失了次数，而我们只想消除顺序差异。

这时可以做一个更小的改动：把双方字符分别排序，再比较排序结果。

```text
eat -> aet
tea -> aet
aab -> aab
abb -> abb
```

排序保留每个字符的出现次数，只统一排列顺序。因此两个词互为异位词，当且仅当它们排序后的字符列表相等。先把这个判断写成最小可运行版本：

```python
def same_anagram(a: str, b: str) -> bool:
    return sorted(a) == sorted(b)


assert same_anagram("eat", "tea")
assert not same_anagram("aab", "abb")
assert same_anagram("", "")
assert not same_anagram("a", "aa")
assert set("aab") == set("abb")  # 集合相同仍不足以判定异位词
```

现在能准确判断两个词是否同组，也保留了重复字符的信息；但还不能给整个数组分组。下一步要决定：读到一个新词时，它应该进入哪个已有分组，还是另开一组？

## Step 2：拿每组的第一个词作为代表

已有的 `same_anagram` 只能回答两个词是否同组。读到第三个词 `"tan"` 时，我们还需要记住此前已经形成的分组。

先用列表 `groups` 保存结果。每个分组非空，因此可以拿 `group[0]` 当作代表：新词只要与代表互为异位词，就应该加入这一组；检查完所有代表仍不匹配，就创建只含新词的分组。

为什么不用比较组里的每一个词？如果组内已有的词与代表排序后都相同，而新词也与代表排序后相同，那么它自然与这一组所有词都匹配。

在上一步的 `same_anagram` 定义后面接上下面的函数，就得到第一版完整解：

```python
def group_scan(strs: list[str]) -> list[list[str]]:
    groups: list[list[str]] = []

    for word in strs:
        for group in groups:
            if same_anagram(word, group[0]):
                group.append(word)
                break
        else:
            groups.append([word])

    return groups


assert group_scan(["eat", "tea", "tan", "ate", "nat", "bat"]) == [
    ["eat", "tea", "ate"], ["tan", "nat"], ["bat"]
]
assert group_scan(["", "", "a", "a", "b"]) == [
    ["", ""], ["a", "a"], ["b"]
]
```

这里的 `else` 对应内层 `for`，不是 `if`。Python 的 `for ... else` 表示：**循环没有被 `break` 提前结束时，执行 `else`**。所以找到匹配就追加并退出；一个匹配都没找到，才新建分组。`groups` 最初为空时，内层循环一次也不执行，同样会进入 `else`，创建第一组。

慢速跟踪前四个词：

| 新词 | 检查已有代表 | 动作 | 处理后的 `groups` |
| --- | --- | --- | --- |
| `eat` | 还没有代表 | 新建一组 | `[["eat"]]` |
| `tea` | 与 `eat` 匹配 | 追加到第一组 | `[["eat", "tea"]]` |
| `tan` | 与 `eat` 不匹配 | 新建一组 | `[["eat", "tea"], ["tan"]]` |
| `ate` | 与 `eat` 匹配 | 追加到第一组 | `[["eat", "tea", "ate"], ["tan"]]` |

每轮只会执行一次追加或新建，因而相同字符串也不会被去重。上面的断言检查了本实现实际产生的顺序；题目本身并不要求这个顺序。

这版已经能正确完成整道题。它还缺的是效率：如果每个词都属于不同分组，第 `i` 个词就要检查前面 `i - 1` 个代表，而且每次比较还会重新排序两个词。

## Step 3：把排序结果变成字典键

上一版的瓶颈不是“判断异位词”本身，而是为了找到所属分组，必须从第一组开始逐个试。既然 `same_anagram` 已经把词转换成了稳定的排序结果，能不能把这个结果直接拿来定位分组？

先看一个词经过排序后的形式：

```text
eat -> aet
tea -> aet
tan -> ant
```

相同的排序结果，正好代表相同的异位词类别。`sorted(word)` 返回的是列表，不能直接作为字典键；把它拼成字符串后，得到一个不可变、可哈希的键，例如 `"aet"`。

于是我们把上一版的 `groups` 列表换成字典 `groups_by_key`：键是排序后的字符串，值仍然是原始字符串列表。这样不会把 `"eat"` 改写成 `"aet"`，输出仍保留输入中的原词。

这一次只替换“寻找分组”的方式，分组内追加和最后收集结果的逻辑不变：

```python
def group_by_sorted(strs: list[str]) -> list[list[str]]:
    groups_by_key: dict[str, list[str]] = {}

    for word in strs:
        key = "".join(sorted(word))
        if key not in groups_by_key:
            groups_by_key[key] = []
        groups_by_key[key].append(word)

    return list(groups_by_key.values())


assert group_by_sorted(["eat", "tea", "tan", "ate", "nat", "bat"]) == [
    ["eat", "tea", "ate"], ["tan", "nat"], ["bat"]
]
assert group_by_sorted(["", "", "a", "a", "b"]) == [
    ["", ""], ["a", "a"], ["b"]
]
```

现在查找分组只需一次字典查找（哈希表的平均情况），不再扫描已有分组。键相同当且仅当排序后字符序列相同，因此它仍然遵守前一步建立的异位词判定。

这版已经是清晰、正确且常用的解法。它仍会为每个词排序；而题目限定了字符集合只有 26 个小写英文字母，下一步可以继续问：我们真的需要把字符排成顺序吗？

## Step 4：用 26 个计数构造最终键

排序的目的只是让“字母及其次数”变得可比较。题目已经保证字符是小写英文字母，所以可以直接记录 26 个字母各出现了几次：`counts[0]` 对应 `a`，`counts[1]` 对应 `b`，依此类推。

例如：

```text
eat -> (1, 0, 0, 0, 1, ..., 1, ..., 0)
tea -> (1, 0, 0, 0, 1, ..., 1, ..., 0)
aab -> (2, 1, 0, ..., 0)
abb -> (1, 2, 0, ..., 0)
```

前两者的 26 维计数完全相同，后两者不同。列表不能作为字典键，但把它转成不可变的元组即可。这个键比排序结果更直接：每个字符只访问一次，单词长度为 `k` 时构造键是 `O(k)`。

在上一版中，只替换 `key` 的构造方式，字典分组循环保持不变：

```python
def count_key(word: str) -> tuple[int, ...]:
    counts = [0] * 26
    for char in word:
        counts[ord(char) - ord("a")] += 1
    return tuple(counts)


assert count_key("eat") == count_key("tea")
assert count_key("aab") != count_key("abb")
assert count_key("") == (0,) * 26
```

现在把这个键接回之前的字典循环，形成题目平台需要的 `class Solution`。类包装没有新增算法；新增的只有按约束构造计数元组。

```python
class Solution:
    def groupAnagrams(self, strs: list[str]) -> list[list[str]]:
        groups_by_key: dict[tuple[int, ...], list[str]] = {}

        for word in strs:
            counts = [0] * 26
            for char in word:
                counts[ord(char) - ord("a")] += 1
            key = tuple(counts)

            if key not in groups_by_key:
                groups_by_key[key] = []
            groups_by_key[key].append(word)

        return list(groups_by_key.values())


solution = Solution()
assert solution.groupAnagrams(["eat", "tea", "tan", "ate", "nat", "bat"]) == [
    ["eat", "tea", "ate"], ["tan", "nat"], ["bat"]
]
assert solution.groupAnagrams(["", "", "a", "a", "b"]) == [
    ["", ""], ["a", "a"], ["b"]
]
```

这里的循环不变量是：处理完当前 `word` 后，`groups_by_key[key]` 恰好保存到目前为止所有具有该计数向量的原始词。计数向量相等，当且仅当两个词每个字母的出现次数都相等，也就当且仅当它们互为异位词。因此每个词进入正确的组，且每次只追加一次，所有输入出现都会保留。

### 复杂度

设所有字符串的总长度为 `S`，输入有 `n` 个词，最终有 `g` 个分组。每个字符只参与一次计数，构造键还要创建固定 26 个位置的元组，因此时间复杂度是 `O(S + 26n)`；因为 `26` 是常数，也可以写成 `O(S + n)`。只有在额外知道每个词至少有一个字符时，才可以进一步把它写成 `O(S)`；本题允许空字符串，所以这里保留更精确的 `O(S + n)` 表述。哈希表操作按平均情况计算。除输出本身外，字典及键占用 `O(26g)`，保存分组中的输入引用占用 `O(n)`。

如果不特别利用小写字母约束，Step 3 的排序键版本更通用，时间是 `O(Σ k log k)`；它也可能更容易迁移到其他字符集。优化不是无条件更好，而是用固定字符集换取线性计数。

## 边界与常见错误

- 空字符串的计数键是 26 个零，多个空字符串会进入同一组，不能跳过。
- 计数必须保留次数；只用 `set(word)` 会把 `aab` 和 `abb` 错误地合并。
- 返回的是原始词列表，不是排序后的键；键只负责定位分组。
- 本文的 `ord(char) - ord("a")` 依赖题目保证的小写英文字母。面对大小写、Unicode 或其他字符集，应重新选择键的表示方式。
- 题目不要求输出顺序，断言中的顺序只是当前遍历顺序，不是额外约束。

## 小结

这道题的关键不是背一个 `defaultdict` 模板，而是逐步回答“什么相等意味着应该同组”：

1. 两个词先通过排序结果判断异位关系。
2. 代表词扫描把这个判断扩展成第一版完整解。
3. 排序结果变成字典键，消除已有分组的线性查找。
4. 在固定 26 字母约束下，用计数元组避免排序，得到最终 `Solution`。

练习时可以先保留 Step 2 的慢速版本，用它作为小规模测试的参考实现，再对比最终版本的分组结果。这样优化改变的是定位方式，而不是题目语义。

## 参考

- [LeetCode 49：Group Anagrams](https://leetcode.com/problems/group-anagrams/)
- [Doocs 题解镜像（英文）](https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0049.Group%20Anagrams/README_EN.md)

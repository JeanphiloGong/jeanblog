---
title: "LeetCode 609：在文件系统中查找重复文件，从解析记录到按内容分组"
subtitle: "先读懂 name(content)，再把两两比较变成内容到完整路径的映射"
date: 2026-10-10T10:00:00+08:00
draft: false
categories: ["LeetCode"]
tags: ["哈希表", "字符串", "分组", "LeetCode 609"]
description: "逐步解决 LeetCode 609：解析目录与文件描述，写出两两比较的正确基线，再按内容哈希分组，并分析路径拼接成本和真实文件系统的边界。"
summary: "重复的是文件内容，返回的却是完整路径。通过五个连续检查点，把输入解析、正确分组、哈希优化和平台提交串起来。"
keywords: ["LeetCode 609", "Find Duplicate File in System", "重复文件", "哈希分组", "Python"]
readingTime: 12
---

## 题目要求：比较内容，返回完整路径

给定字符串数组 `paths`，每个字符串描述一个目录及其中的文件：

```text
目录路径 文件名(内容) 文件名(内容) ...
root/a 1.txt(abcd) 2.txt(efgh)
```

需要把**内容完全相同**的文件放在同一组，返回每组的完整文件路径。每组至少有两个文件；只有一个文件的内容不需要出现在答案里。组与组之间、组内路径之间的顺序都不重要。

例如：

```text
输入：
[
  "root/a 1.txt(abcd) 2.txt(efgh)",
  "root/c 3.txt(abcd)",
  "root/c/d 4.txt(efgh)",
  "root 4.txt(efgh)"
]

一种合法输出：
[
  ["root/a/1.txt", "root/c/3.txt"],
  ["root/a/2.txt", "root/c/d/4.txt", "root/4.txt"]
]
```

这里 `1.txt` 和 `3.txt` 名字不同，但内容都是 `abcd`，所以属于同组。反过来，文件名相同不意味着内容相同。目录也可以只有 `root`，不能假设一定存在子目录。

题目给出的主要约束是：

- `1 <= paths.length <= 2 * 10^4`。
- 每条目录描述的长度不超过 `3000`，所有描述的字符总数不超过 `5 * 10^5`。
- 每条描述至少包含一个文件，目录与文件信息之间使用单个空格分隔。
- 每条描述对应不同目录，同一目录中的文件或目录不会重名。

注意，这些是已经按题目格式编码好的字符串，不是让程序真的打开磁盘上的目录。本文使用 Python 3.10+，适合已经了解循环、列表和字典，想练习“解析后分组”的读者。

## 步骤 1：先只解析一个文件

先缩小问题：**已知目录 `root/a` 和一条文件信息 `1.txt(abcd)`，能否得到完整路径和内容？**

目前我们只知道输入和输出格式。直接比较整段 `1.txt(abcd)` 会把文件名也带进去，导致 `1.txt(abcd)` 与 `3.txt(abcd)` 被错误地区分；只取文件名，又无法判断重复。

因此先做一件事：把一个文件描述拆成“用于返回的完整路径”和“用于比较的内容”。

```python
directory = "root/a"
entry = "1.txt(abcd)"

name, content_and_end = entry.split("(", 1)
content = content_and_end[:-1]
full_path = directory + "/" + name

assert name == "1.txt"
assert content == "abcd"
assert full_path == "root/a/1.txt"
assert "root" + "/" + name == "root/1.txt"
assert "root/a/b" + "/" + name == "root/a/b/1.txt"
```

`split("(", 1)` 只在第一个左括号处切开，结果是 `1.txt` 和 `abcd)`；`[:-1]` 去掉表示文件描述结束的最后一个右括号。不要使用 `strip("()")`，我们要去掉的是指定位置的分隔符，不是随意清理两端字符。

这一解析方式依赖题目提供的 `文件名(内容)` 格式。它不是一个能够处理任意操作系统文件名、转义和带空格内容的通用解析器。

**本步检查点：** 已经能从一条文件信息得到 `(完整路径, 内容)`。但当前程序只处理写死的一个文件，还不能遍历整份输入。

## 步骤 2：把所有目录展开成文件记录

当前版本会解析一个文件，但示例第一行就有两个文件，而且同内容的另一个文件可能出现在后面的目录。如果只拿每行的第一个文件比较，就会漏掉 `root/a/2.txt`。

这一步不判断重复，只把刚才的解析放进两层循环：外层读取目录描述，内层读取这个目录中的文件。为了让后续比较可以访问所有文件，用 `records` 保存 `(完整路径, 内容)`。

```python
def collect_files(paths: list[str]) -> list[tuple[str, str]]:
    records = []

    for line in paths:
        parts = line.split(" ")
        directory = parts[0]

        for entry in parts[1:]:
            name, content_and_end = entry.split("(", 1)
            content = content_and_end[:-1]
            full_path = directory + "/" + name
            records.append((full_path, content))

    return records


example_paths = [
    "root/a 1.txt(abcd) 2.txt(efgh)",
    "root/c 3.txt(abcd)",
    "root/c/d 4.txt(efgh)",
    "root 4.txt(efgh)",
]

assert collect_files(example_paths) == [
    ("root/a/1.txt", "abcd"),
    ("root/a/2.txt", "efgh"),
    ("root/c/3.txt", "abcd"),
    ("root/c/d/4.txt", "efgh"),
    ("root/4.txt", "efgh"),
]
```

`parts[0]` 是目录，不是文件，所以内层循环从 `parts[1:]` 开始。这里按空格切分，仍然依据题目给出的序列化格式；不要把它直接用于解析任意真实文件内容。

检查上面的断言：四条目录描述展开成五条文件记录，第一行的第二个文件没有遗漏，只有 `root` 的目录也正确补上了 `/`。

**本步检查点：** 所有文件都已经变成可比较的内容和可返回的路径。还缺少“哪些记录属于同一个重复组”的判定。

## 步骤 3：先写出不会漏组的两两比较

现在的基线是 `collect_files`：它能给出所有 `(路径, 内容)`。最直接的下一问是：选一条记录，和后面的每条记录比较内容，是否相等就放进同一组？

这个想法还要处理一个具体问题：如果三个文件内容都为 `abcd`，单独输出 `(A, B)` 和 `(A, C)` 会把同一个重复组拆成两组。我们需要让一条记录只被分配到一个内容组，并且不要输出只有一个成员的组。

先用 `assigned` 记录某条记录是否已经被前面的组接收。每次找到一个尚未分配的记录，把它作为 `seed`，再扫描它后面的记录；相同内容的记录加入当前 `group`，同时标记为已分配。

```python
def find_duplicate_pairwise(paths: list[str]) -> list[list[str]]:
    records = collect_files(paths)
    assigned = [False] * len(records)
    result = []

    for i, (seed_path, seed_content) in enumerate(records):
        if assigned[i]:
            continue

        assigned[i] = True
        group = [seed_path]

        for j in range(i + 1, len(records)):
            if not assigned[j] and records[j][1] == seed_content:
                assigned[j] = True
                group.append(records[j][0])

        if len(group) > 1:
            result.append(group)

    return result


def normalize(groups: list[list[str]]) -> set[frozenset[str]]:
    return {frozenset(group) for group in groups}


three_equal = [
    "root/a 1.txt(abcd)",
    "root/b 2.txt(abcd)",
    "root/c 3.txt(abcd)",
]
assert normalize(find_duplicate_pairwise(three_equal)) == {
    frozenset({"root/a/1.txt", "root/b/2.txt", "root/c/3.txt"})
}
assert find_duplicate_pairwise(["root 1.txt(a)", "root 2.txt(b)"]) == []
assert find_duplicate_pairwise([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []
assert normalize(find_duplicate_pairwise([
    "root 1.txt(a)",
    "other 1.txt(a)",
    "other 2.txt(b)",
])) == {frozenset({"root/1.txt", "other/1.txt"})}
```

这里 `assigned[j]` 的作用是防止一个已经进入前面组的记录再次被后面的种子拿走。`len(group) > 1` 则对应题目“至少两个文件”的输出条件。

**本步检查点：** 这已经是一个正确的完整基线：不同文件名但内容相同会合并，同名但内容不同不会合并，单例会被过滤。它的缺点也很明确：内容都不相同的时候，几乎每个种子都要扫描剩余记录。

## 步骤 4：让内容成为分组键

上一版是正确的，但它在“每个文件内容都不同”的输入上会重复检查同一批记录：第一个种子看后面所有记录，第二个种子再看几乎所有剩余记录。我们已经知道真正决定分组的是 `content`，所以可以在读到每条记录时直接把路径放进对应的内容桶。

这次只替换“如何形成组”的部分，`collect_files` 不变，单例过滤也不变：

```python
def find_duplicate_by_content(paths: list[str]) -> list[list[str]]:
    records = collect_files(paths)
    groups_by_content: dict[str, list[str]] = {}

    for full_path, content in records:
        if content not in groups_by_content:
            groups_by_content[content] = []
        groups_by_content[content].append(full_path)

    return [
        group
        for group in groups_by_content.values()
        if len(group) > 1
    ]


expected = {
    frozenset({"root/a/1.txt", "root/c/3.txt"}),
    frozenset({"root/a/2.txt", "root/c/d/4.txt", "root/4.txt"}),
}
assert normalize(find_duplicate_by_content(example_paths)) == expected
assert normalize(find_duplicate_by_content(three_equal)) == {
    frozenset({"root/a/1.txt", "root/b/2.txt", "root/c/3.txt"})
}
assert find_duplicate_by_content(["root 1.txt(a)", "root 2.txt(b)"]) == []
assert find_duplicate_by_content([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []

# 独立的随机 oracle：先生成结构化记录，再分别构造输入和预期分组。
import random

rng = random.Random(609)
for _ in range(20):
    structured = []
    for index in range(rng.randint(1, 6)):
        directory = f"d{index}"
        for file_index in range(rng.randint(1, 4)):
            content = f"c{rng.randrange(5)}"
            structured.append((directory, f"f{file_index}.txt", content))

    generated_paths = []
    for directory in sorted({row[0] for row in structured}):
        entries = [
            f"{name}({content})"
            for row_directory, name, content in structured
            if row_directory == directory
        ]
        generated_paths.append(" ".join([directory, *entries]))

    oracle = {}
    for directory, name, content in structured:
        oracle.setdefault(content, set()).add(f"{directory}/{name}")
    expected_random = {
        frozenset(file_paths)
        for file_paths in oracle.values()
        if len(file_paths) > 1
    }
    assert normalize(find_duplicate_by_content(generated_paths)) == expected_random
    assert normalize(find_duplicate_pairwise(generated_paths)) == expected_random
```

现在每条记录只做一次字典定位和一次追加：相同内容自然落入同一个桶，文件名只作为桶里的路径保存。`len(group) > 1` 仍然负责排除单例，因此优化没有改变输出规则。

为什么这能覆盖所有重复组？如果两条记录内容相等，它们使用同一个字典键，都会被追加到同一个桶；如果内容不等，键不同，不会混桶。于是每个桶正好对应一个内容等价类，最后筛掉大小为 1 的桶即可。

**本步检查点：** 已经去掉了显式的两两扫描，平均情况下可以按内容键直接分组。下一步把同样的逻辑装进 LeetCode 要求的 `Solution.findDuplicate`，并把输入字符、输出路径字符和哈希表假设说清楚。

## 步骤 5：封装成提交代码，并把成本说准确

上一版已经有正确的分组逻辑，但 LeetCode 需要 `Solution.findDuplicate`。这里不再发明新算法，只把已经验证过的解析和内容分桶放进平台方法。

```python
class Solution:
    def findDuplicate(self, paths: list[str]) -> list[list[str]]:
        groups_by_content: dict[str, list[str]] = {}

        for line in paths:
            parts = line.split(" ")
            directory = parts[0]

            for entry in parts[1:]:
                name, content_and_end = entry.split("(", 1)
                content = content_and_end[:-1]
                full_path = directory + "/" + name

                if content not in groups_by_content:
                    groups_by_content[content] = []
                groups_by_content[content].append(full_path)

        return [
            group
            for group in groups_by_content.values()
            if len(group) > 1
        ]


def as_sets(groups: list[list[str]]) -> set[frozenset[str]]:
    return {frozenset(group) for group in groups}


solution = Solution()
official_paths = [
    "root/a 1.txt(abcd) 2.txt(efgh)",
    "root/c 3.txt(abcd)",
    "root/c/d 4.txt(efgh)",
    "root 4.txt(efgh)",
]
official_expected = {
    frozenset({"root/a/1.txt", "root/c/3.txt"}),
    frozenset({"root/a/2.txt", "root/c/d/4.txt", "root/4.txt"}),
}
assert as_sets(solution.findDuplicate(official_paths)) == official_expected
assert solution.findDuplicate([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []
assert as_sets(solution.findDuplicate([
    "deep/directory 1.txt(x)",
    "deep/directory 2.txt(x)",
])) == {frozenset({
    "deep/directory/1.txt",
    "deep/directory/2.txt",
})}
assert as_sets(solution.findDuplicate([
    "root/a 1.txt(same)",
    "root/b 2.txt(same)",
    "root/c 3.txt(same)",
])) == {frozenset({
    "root/a/1.txt", "root/b/2.txt", "root/c/3.txt",
})}
assert solution.findDuplicate([
    "root/a 1.txt(a)",
    "root/b 2.txt(b)",
    "root/c 3.txt(c)",
]) == []
assert solution.findDuplicate([
    "root 1.txt(a)",
    "other 1.txt(b)",
]) == []

import random

rng = random.Random(6090)
for _ in range(40):
    structured = []
    for directory_index in range(rng.randint(1, 7)):
        directory = f"random/{directory_index}"
        for file_index in range(rng.randint(1, 5)):
            structured.append(
                (directory, f"file{file_index}.txt", f"content{rng.randrange(8)}")
            )

    generated_paths = []
    directories = sorted({directory for directory, _, _ in structured})
    for directory in directories:
        entries = [
            f"{name}({content})"
            for row_directory, name, content in structured
            if row_directory == directory
        ]
        generated_paths.append(" ".join([directory, *entries]))

    oracle = {}
    for directory, name, content in structured:
        oracle.setdefault(content, set()).add(f"{directory}/{name}")
    expected_random = {
        frozenset(file_paths)
        for file_paths in oracle.values()
        if len(file_paths) > 1
    }
    assert as_sets(solution.findDuplicate(generated_paths)) == expected_random
```

### 正确性：每个内容桶就是一个等价类

遍历一条文件记录时，代码把它的完整路径追加到键为 `content` 的桶中。因此：

1. 两个文件内容相等时，键相等，它们一定进入同一个桶。
2. 两个文件内容不等时，键不等，它们不可能进入同一个桶。
3. 最后只返回大小大于 1 的桶，正好删除题目不需要的单例。

这三点同时说明了完整性和排他性：不会漏掉重复内容，也不会把不同内容混在一起。文件名和目录只参与构造返回路径，不参与分组判断。

### 复杂度：输入字符和输出路径都要计入

设：

- `S` 是所有输入目录描述的字符总数；
- `F` 是解析出的文件记录数；
- `P` 是所有要保存或返回的完整路径字符总数。

在 Python 字典平均常数时间查找的假设下，每个输入字符和文件记录只被解析、分桶一次，时间复杂度可写为 **平均 `O(S + P)`**；空间复杂度也是 **`O(S + P)`**，因为内容键、字典中的路径和最终输出都要占空间。`P` 不能简单省略：一个很长的目录前缀会在每个文件的完整路径中重复出现。

哈希表的最坏情况仍可能退化，且超长内容会带来字符串存储和哈希成本。复杂度分析描述的是这道题的序列化输入，不等于真实磁盘扫描的 I/O 成本。

### 真实文件系统中的边界

题目输入已经替我们完成了目录遍历。面对真实文件系统，还要额外决定：

- 用 DFS 还是 BFS 遍历目录，并处理权限错误、符号链接和循环；
- 大文件先按大小分组，再按分块摘要筛选候选；
- 摘要相同只能说明“可能相同”，最终仍要逐字节比较候选文件，不能把固定长度 digest 当成数学上的内容相等证明；
- 只能每次读取 1 KB 时，按块更新摘要并控制内存，而不是一次性读入整个文件。

这些是另一个“扫描真实文件系统”的问题，不能偷偷塞进本题的字符串解析代码。

## 小结

这道题的主线不是“看到重复就立刻用哈希表”，而是：

1. 先确认重复关系由**完整内容**决定；
2. 从 `文件名(内容)` 中解析出内容和完整路径；
3. 先用带 `assigned` 的两两比较写出一个可验证的正确基线；
4. 观察重复扫描的压力，把内容直接提升为字典分组键；
5. 保留单例过滤，并在提交前把路径字符、哈希假设和真实文件系统边界说清楚。

当输入已经提供了“记录 + 属性”的格式时，先把记录解析成稳定的数据，再选择分组键，通常比一开始套用某个模板更可靠。

参考：

- [LeetCode 609: Find Duplicate File in System](https://leetcode.com/problems/find-duplicate-file-in-system/)
- [题目事实镜像（Doocs）](https://raw.githubusercontent.com/doocs/leetcode/main/solution/0600-0699/0609.Find%20Duplicate%20File%20in%20System/README_EN.md)

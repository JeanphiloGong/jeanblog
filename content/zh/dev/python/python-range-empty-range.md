---
title: "Python range() 的边界：为什么循环有时一次也不执行？"
date: 2026-10-08
draft: false
description: "从 range(10, 7) 出发，解释 Python range() 的起点、终点和步长，理解空范围为什么不会报错，以及它如何自动跳过不需要处理的循环。"
tags: ["Python", "range", "循环"]
categories: ["Python"]
keywords: ["Python range", "Python 空范围", "range 起点终点", "Python 循环边界"]
---

下面这段代码不会打印任何内容，也不会报错：

```python
for x in range(10, 7):
    print(x)
```

原因很简单：**`range(10, 7)` 使用默认的正步长，起点已经超过终点，因此范围为空，循环体一次也不执行。**

本文适合刚开始学习 Python 循环的读者，阅读约需 3 分钟。

## 先看懂 range() 的三个参数

`range()` 常见的写法有三种：

```python
list(range(4))         # [0, 1, 2, 3]
list(range(2, 7))      # [2, 3, 4, 5, 6]
list(range(2, 7, 2))   # [2, 4, 6]
```

完整形式是 `range(start, stop, step)`：

- `start`：起点，省略时默认为 `0`。
- `stop`：终点，不包含在范围内。
- `step`：步长，省略时默认为 `1`。

`range()` 返回的是一个范围对象，并不是列表。这里使用 `list()`，只是为了直接观察其中的值。

对于正步长，每个值都必须满足 `x < stop`。因此，`range(2, 7)` 从 `2` 开始，每次增加 `1`，到 `6` 为止，不包含 `7`。

## 起点碰到或超过终点，会发生什么？

默认步长是 `1`，所以如果一开始就有 `start >= stop`，连第一个值都不符合条件：

```python
list(range(7, 7))   # []
list(range(10, 7))  # []
```

**空范围是正常结果，不是异常。** `for` 循环没有值可以遍历，就会直接跳过循环体，继续执行后面的代码：

```python
for x in range(10, 7):
    print(x)

print("done")
# 只输出 done
```

## 为什么遍历到 amount 要写 amount + 1？

假设我们想从 `coin` 开始，一直遍历到 `amount`，包含 `amount` 本身：

```python
for x in range(coin, amount + 1):
    ...
```

因为 `stop` 不包含在范围内，要包含 `amount`，就需要把终点设为 `amount + 1`。

当 `amount = 6` 时，可以对比三种情况：

| `coin` 与 `amount` 的关系 | 表达式 | 遍历的值 |
| --- | --- | --- |
| `coin < amount` | `range(2, 7)` | `2, 3, 4, 5, 6` |
| `coin == amount` | `range(6, 7)` | `6` |
| `coin > amount` | `range(10, 7)` | 无 |

注意：**`coin == amount` 时，循环仍然执行一次。** 此时起点是 `6`，终点是 `7`，`6 < 7` 成立。

即使 `coin` 只比 `amount` 大 `1`，范围也已经为空：

```python
list(range(7, 6 + 1))  # []，因为起点等于终点
```

## 在零钱兑换循环中，空范围正好符合需求

零钱兑换代码里可能会出现下面的遍历方式：

```python
for coin in coins:
    for x in range(coin, amount + 1):
        ...
```

这里假设硬币面值是正整数，目标金额是非负整数。内层循环只处理从当前面值到目标金额的各个金额。

如果 `coin = 10`、`amount = 6`，面值为 `10` 的硬币无法参与凑出 `0` 到 `6` 的金额。内层的 `range(10, 7)` 恰好为空，所以循环体自动跳过。

因此，如果目的只是跳过这次内层遍历，就不必再加一句：

```python
if coin > amount:
    continue
```

这个结论来自循环范围本身，不需要记忆额外的算法技巧。

## 注意：倒序遍历需要负步长

`range(10, 7)` 不会因为起点更大就自动倒着走。要倒序遍历，必须明确指定负步长：

```python
list(range(10, 7, -1))  # [10, 9, 8]
```

负步长时，每个值必须满足 `x > stop`，终点仍然不包含在范围内：

```python
list(range(7, 10, -1))  # []，步长方向无法走向终点
```

另外，步长不能是 `0`；`range(2, 7, 0)` 会抛出 `ValueError`。空范围不会报错，并不意味着所有参数组合都合法。

## 用一段代码验证边界

可以直接运行下面的代码：

```python
amount = 6

assert list(range(2, amount + 1)) == [2, 3, 4, 5, 6]
assert list(range(6, amount + 1)) == [6]
assert list(range(7, amount + 1)) == []
assert list(range(10, amount + 1)) == []
assert list(range(10, 7, -1)) == [10, 9, 8]
assert list(range(7, 10, -1)) == []

visited = []
for x in range(10, 7):
    visited.append(x)
assert visited == []

print("所有边界检查通过")
```

记住三个规则就够了：**终点不包含在内，默认步长为 `1`，没有符合边界条件的值时范围为空。** 阅读循环代码时，先把 `start`、`stop` 和 `step` 代入，就能判断循环是否执行，以及会执行哪些值。

参考：[Python 官方文档：range](https://docs.python.org/zh-cn/3/library/stdtypes.html#range)。

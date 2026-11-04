---
title: "预留失败是 false，为什么 if (!reserved) 反而会执行？"
date: 2026-11-04
draft: false
categories: ["前端"]
tags: ["JavaScript", "TypeScript", "控制流"]
description: "从 if (!executions.reserve(id, workerId)) continue 出发，逐步理解逻辑取反、if 的真假值判断，以及 !value 和 value === false 的区别。"
---

读任务调度代码时，可能会看到：

```ts
if (!executions.reserve(id, workerId)) continue;
```

假设这里的 `reserve()` 返回布尔值：成功预留任务返回 `true`，没有预留成功返回 `false`。这行代码表达的是：**预留失败，就跳过当前任务。**

但初学时很容易卡住：失败不是 `false` 吗？`if` 不是条件成立才执行吗？为什么失败时反而执行了 `continue`？

关键是：`if` 判断的是整个 `!reserved`，不是单独判断 `reserved`。

## 先拆开，别同时理解三个动作

原来的代码可以拆成：

```ts
const reserved = executions.reserve(id, workerId);

if (!reserved) {
  continue;
}
```

这里依次发生三件事：调用 `reserve()` 得到结果，对结果取反，再由 `if` 决定是否执行 `continue`。这段代码需要放在循环内，`continue` 才有意义。

先暂时忘掉任务和循环，只看逻辑取反：

```js
console.log(!true);  // false
console.log(!false); // true
```

前缀 `!` 可以读成“不”。对布尔值来说，它把 `true` 变成 `false`，把 `false` 变成 `true`。

它不会修改原来的变量：

```js
const reserved = false;

console.log(!reserved); // true
console.log(reserved);  // 仍然是 false
```

`!reserved` 是一个新的表达式，计算出相反的布尔结果；`reserved` 自己并没有变成成功。

## 为什么失败后，if 的条件会成立？

变量名可以理解为一个问题：

- `reserved`：预留成功了吗？
- `!reserved`：没有预留成功吗？

假设预留失败：

```text
预留成功了吗？       → false
没有预留成功吗？     → true
```

两句话问的不是同一个问题，所以答案相反。

接着看 `if`：括号里的表达式结果为真值时，才执行分支。在这里，取反已经产生了布尔值：

```js
const reserved = false;

if (!reserved) {
  console.log("预留失败，跳过任务");
}
```

执行过程是：

```text
reserved 为 false
        ↓
!reserved 为 true
        ↓
if 的条件成立
        ↓
输出“预留失败，跳过任务”
```

两种情况放在一起就很清楚：

| 预留结果 | reserved | !reserved | 是否进入 if |
| --- | --- | --- | --- |
| 成功 | true | false | 否 |
| 失败 | false | true | 是 |

可以把 `!false` 联想成“否定一个不成立的判断”，但不必记成数学里的“负负得正”。这里没有数字正负，也不是失败变成成功，而是**“没有成功”这个条件成立了**。

## 加回 continue：跳过的是哪部分？

下面是可以直接运行的 JavaScript 示例。用数组里的布尔值模拟预留结果，不需要数据库或子进程：

```js
const attempts = [
  { id: "任务 A", reserved: false },
  { id: "任务 B", reserved: true },
];

for (const { id, reserved } of attempts) {
  console.log(`检查 ${id}`);

  if (!reserved) {
    console.log(`跳过 ${id}`);
    continue;
  }

  console.log(`启动 ${id}`);
}
```

保存为 `reserve-demo.js`，执行 `node reserve-demo.js`，输出：

```text
检查 任务 A
跳过 任务 A
检查 任务 B
启动 任务 B
```

`continue` 跳过当前这轮循环剩余的代码，继续下一轮，所以任务 A 不会走到“启动”。它不会退出整个函数，也不像 `break` 那样结束循环。

回到原来的表达式：

```ts
if (!executions.reserve(id, workerId)) continue;
```

若 `reserve()` 返回 `false`，取反得到 `true`，执行 `continue`；若返回 `true`，取反得到 `false`，继续执行下面的代码。

这里假定 `reserve()` **同步返回布尔值**。如果它返回的是 `Promise<boolean>`，就不能用同样的读法：Promise 对象本身是真值，对它取反会得到 `false`，并不会等待预留结果。异步调用应在允许使用 `await` 的上下文中先取得结果，再判断。

同样，如果 `reserve()` 抛出异常，就没有返回布尔结果，程序也不会因为这一行自动进入 `continue` 分支。异常如何处理取决于外围代码。

## !reserved 就是 reserved === false 吗？

**只有已知 reserved 是布尔值时，这两种判断才等价。**

JavaScript 的 `!` 也接受其他类型。它先按语言规则判断值的真假，再取反，结果始终是布尔值。可以理解为：

```js
!value
// 与下面的表达式结果相同：
!Boolean(value)
```

而 `=== false` 检查的是“这个值是否严格等于布尔值 false”，不会做真假值转换。

例如：

```js
const value = 0;

console.log(!value);          // true
console.log(value === false); // false
```

`0` 是假值，所以 `!0` 是 `true`；但数字 `0` 并不严格等于布尔值 `false`。

再看几个容易混淆的值：

| value | 按真假值判断 | !value | value === false |
| --- | --- | --- | --- |
| false | 假 | true | true |
| 0 | 假 | true | false |
| 空字符串 `""` | 假 | true | false |
| undefined | 假 | true | false |
| 字符串 `"false"` | 真 | false | false |
| 空数组 `[]` | 真 | false | false |
| 空对象 `{}` | 真 | false | false |

常见假值包括 `false`、`0`、`-0`、`0n`、空字符串、`null`、`undefined` 和 `NaN`。空数组、空对象都是真值；字符串 `"false"` 也不会因为内容写着 false 就变成假值。

因此，如果你只想判断值是否为 `undefined`，就写 `value === undefined`。用 `!value` 会把 `0` 和空字符串等其他假值也一起包括进去。判断方式应对应你真正想问的问题。

## 前面的 ! 和后面的 ! 是两个东西

如果你刚读过 [waiting.shift()! 的非空断言]({{< relref "typescript-non-null-assertion.md" >}})，这里尤其容易混淆：

| 写法 | 作用 | 运行时是否生效 |
| --- | --- | --- |
| `!reserved` | JavaScript 逻辑取反，得到布尔值 | 是 |
| `waiting.shift()!` | TypeScript 非空断言，要求类型检查器按非空值处理 | 否，转为 JavaScript 时被移除 |

读 `if (!reserved)` 时，可以先口头翻译成“如果没有预留成功”。再问：没有成功这件事，现在成立吗？这样就不需要绕在“false 为什么又变成 true”里面了。

## 参考

- [MDN：Logical NOT (!)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_NOT)
- [MDN：if...else](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/if...else)
- [MDN：continue](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/continue)
- [MDN：Falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy)

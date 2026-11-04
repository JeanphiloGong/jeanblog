---
title: "waiting.shift()! 后面的感叹号，到底保证了什么？"
date: 2026-11-04
draft: false
categories: ["前端"]
tags: ["TypeScript", "JavaScript", "类型检查"]
description: "用队列取值的例子解释 TypeScript 非空断言：为什么 shift() 可能返回 undefined，后缀 ! 如何影响类型检查，以及什么时候应该改用真实的运行时判断。"
---

读 TypeScript 代码时，可能会遇到这样一行：

```ts
const task = waiting.shift()!;
```

`shift()` 是取出队列中的第一个元素，那后面的 `!` 呢？是检查成功，还是取不到元素就报错？

都不是。**后缀 `!` 是非空断言：告诉 TypeScript“我确定这个值不是 null 或 undefined”。它影响类型检查，不会在运行时检查或修复这个值。**

如果你已经了解数组和函数，却对这个符号感到困惑，可以沿着“正常取值 → 空队列 → 类型检查 → 断言失效”走一遍。

## 先不加感叹号，看看 shift() 返回什么

`shift()` 会删除并返回数组的第一个元素。连续取两次，就能看到它的两种结果：

```ts
const waiting: string[] = ["任务 A"];

console.log(waiting.shift()); // "任务 A"
console.log(waiting.shift()); // undefined
```

第二次取值时，数组已经空了。JavaScript 不会因为空数组调用了 `shift()` 就报错，而是返回 `undefined`。

因此，开启 `strictNullChecks`（或包含它的 `strict`）后，TypeScript 会这样理解：

```ts
const task = waiting.shift();
// task 的类型是 string | undefined
```

`string | undefined` 表示：这个值可能是字符串，也可能不存在。于是下面的调用不能直接通过类型检查：

```ts
const waiting: string[] = ["任务 A"];
const task = waiting.shift();

console.log(task.toUpperCase());
// 类型错误：task 可能是 undefined。
```

即使这里初始化时放了一个元素，`shift()` 的类型签名仍然允许返回 `undefined`。编译器没有把这次调用当作“一定能拿到元素”。

## 加上 !，改变的是编译器看到的类型

现在加上断言：

```ts
const waiting: string[] = ["任务 A"];
const task = waiting.shift()!;

console.log(task.toUpperCase()); // "任务 A"
```

对于这个表达式，`!` 从类型中排除了 `null` 和 `undefined`。这里原本只有 `string | undefined`，因此结果类型成为 `string`。

可以把它读成：

> 从 waiting 中取出第一个元素。我保证这次取出来的值存在。

注意，这个保证来自写代码的人，不是 TypeScript 运行后得出的结论。转换为 JavaScript 后，类型标注和非空断言都会被移除，核心代码相当于：

```js
const waiting = ["任务 A"];
const task = waiting.shift();

console.log(task.toUpperCase());
```

没有新增的 `if`，也没有自动插入的 `throw`。

## 如果保证错了，代码仍然会出错

只把数组改为空数组，其他地方不变：

```ts
const waiting: string[] = [];
const task = waiting.shift()!;

console.log(task); // undefined
console.log(task.toUpperCase()); // 运行时报 TypeError
```

类型检查可以通过，但 `task` 实际还是 `undefined`。

而且，**出错的是访问 `task.toUpperCase()`，不是执行 `shift()!` 的那一刻**。感叹号没有在运行时做任何事。如果只打印 `task`，程序就会正常打印 `undefined`。

这也解释了为什么“加个 ! 就不报红了”不等于问题已经解决：编辑器不再提醒，是因为你要求它相信你的断言。

## 什么时候这个断言有依据？

例如，在普通字符串数组上，先确认非空，再立即取值：

```ts
const waiting: string[] = ["任务 A"];

if (waiting.length > 0) {
  const task = waiting.shift()!;
  console.log(task.toUpperCase());
}
```

这里检查与取值之间没有其他操作改变数组，也没有异步等待。对于这个例子，非空判断足以支持你的断言。

不过，TypeScript 通常不会根据 `waiting.length > 0` 自动把 `shift()` 的返回类型缩小为 `string`。`!` 可以用来表达这条代码逻辑已经保证、类型系统没有推导出的事实。

这个结论有前提：数组里的元素确实是字符串，且检查之后到取出之前没有被清空。若中间有 `await`、会修改队列的调用，或者元素类型本来就允许 `undefined`，仅看 `length` 就不够了。

## 空队列是正常情况，就直接处理它

如果取不到任务本来就是正常分支，不必先保证一定有值。直接检查取出的结果更清楚：

```ts
const waiting: string[] = [];
const task = waiting.shift();

if (task !== undefined) {
  console.log(task.toUpperCase());
} else {
  console.log("暂无任务");
}
```

这次 `if` 会真实执行。TypeScript 也能据此知道，在第一个分支里，`task` 已经是字符串。

这里用 `task !== undefined`，而不是 `if (task)`：如果空字符串也是合法任务内容，后者会把空字符串一起排除。

几种写法看起来接近，作用却不同：

| 写法 | 类型检查与运行时行为 |
| --- | --- |
| `task!` | 类型检查时排除 null、undefined；运行时不检查 |
| `if (task !== undefined)` | 运行时检查，并让分支内的类型缩小；只排除 undefined |
| `task?.toUpperCase()` | task 为 null 或 undefined 时跳过调用，表达式结果为 undefined |
| `!task` | JavaScript 逻辑取反，运行时得到布尔值 |

可选链适合允许“没有结果就跳过”的场景。若业务要求必须有任务，它并不能替代错误处理；同样，也不该为了消除类型提示而随意用非空断言。

## 自己验证：通过类型检查，也可能运行失败

在已安装 TypeScript 的环境里，把前面的空队列反例保存为 `empty-queue.ts`，然后执行：

```sh
tsc --strict empty-queue.ts
node empty-queue.js
```

带 `!` 时，编译可以通过；运行时先输出 `undefined`，随后在调用字符串方法时抛出 `TypeError`。删掉 `!` 再编译，类型检查会指出 `task` 可能是 `undefined`。两次实验分别观察编译和运行，不要把它们当作同一件事。

再试着改为 `task !== undefined` 的分支判断：这次不需要断言，空队列也有明确处理方式。

以后再看到 `waiting.shift()!`，可以问一句：**这行代码之前，究竟是什么保证了队列里有可用的元素？** 如果找不到依据，感叹号就只是在隐藏不确定性。

## 参考

- [TypeScript Handbook：Non-null Assertion Operator](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#non-null-assertion-operator-postfix-)
- [TypeScript：strictNullChecks](https://www.typescriptlang.org/tsconfig/strictNullChecks.html)
- [MDN：Array.prototype.shift()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/shift)

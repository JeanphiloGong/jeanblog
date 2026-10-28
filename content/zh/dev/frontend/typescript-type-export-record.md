---
title: "TypeScript 中的 type、export 和 Record 是什么意思？"
date: 2026-10-28
draft: false
categories: ["frontend"]
tags: ["TypeScript", "JavaScript", "基础"]
description: "通过实际例子理解 TypeScript 类型别名、类型导入导出、Record 和 unknown，并区分类型检查与运行时校验。"
---

读 TypeScript 代码时，经常会看到这样的写法：

```ts
type Calls = {
  research: number;
  write: number;
  review: number;
};
```

这里的 `type` 为一类数据起了一个名字：`Calls`。它描述这类数据应该有哪些字段，以及字段的类型。

例如，下面这个对象符合 `Calls` 的定义：

```ts
const calls: Calls = {
  research: 1,
  write: 2,
  review: 2,
};
```

如果把 `research` 写成字符串，TypeScript 的类型检查会指出问题：

```ts
const calls: Calls = {
  research: "一次", // 类型错误：这里要求 number
  write: 2,
  review: 2,
};
```

`type` 本身不会创建对象，也不是一个函数。它是对数据形状的描述；真正的数据仍然由代码创建。

## `export`：让其他文件也能使用这个类型

假设有一个 `workflow.ts` 文件：

```ts
type Calls = {
  research: number;
  write: number;
  review: number;
};

export type Result = {
  topic_id: string;
  status: "completed" | "stopped";
  final_text: string | null;
};
```

`Calls` 只在这个文件内部使用。`Result` 前面加了 `export`，其他文件就可以导入它。例如，在 `display.ts` 中：

```ts
import type { Result } from "./workflow.ts";

const result: Result = {
  topic_id: "topic-1",
  status: "completed",
  final_text: "这是一篇通过审核的正文。",
};

console.log(result.status);
```

`import type` 表示这里导入的是类型。`export` 也可以用于函数、变量和类；本例导出的是类型。

## `|` 和 `null` 又是什么？

回到 `Result`：

```ts
status: "completed" | "stopped";
final_text: string | null;
```

`|` 表示可以是列出的其中一种类型或值。`status` 只能是 `"completed"` 或 `"stopped"`；`final_text` 可以是字符串，也可以是 `null`，例如审核未通过时没有最终正文。

## `Record<string, unknown>` 是什么？

教程中的阶段交付有这样一个字段：

```ts
type Artifact = {
  kind: string;
  body: Record<string, unknown>;
  sha256: string;
};
```

可以先把它理解成一个字典。`Record` 的尖括号里放了两个**类型参数**：

```ts
Record<键的类型, 值的类型>
```

所以 `Record<string, unknown>` 表示：键是字符串，值的类型尚未确定。例如：

```ts
const body: Record<string, unknown> = {
  text: "这是一篇草稿",
  revision: 1,
};

console.log(body["text"]);     // 这是一篇草稿
console.log(body["revision"]); // 1
```

`"text"` 和 `"revision"` 是键，后面分别是它们对应的值。它类似 Python 的 `dict[str, object]`；实际运行时，这里的 `body` 是一个 JavaScript 普通对象。

`unknown` 允许我们接收不同类型的值，但在使用值的具体方法之前，需要先确认类型：

```ts
const value = body["text"];

if (typeof value === "string") {
  console.log(value.toUpperCase());
}
```

这里的 `value` 在类型检查时是 `unknown`；进入 `if` 后，TypeScript 才知道它是字符串。访问不存在的键时，运行结果是 `undefined`，因此类型声明本身也不能保证某个键一定存在。

教程用同一个 `Artifact` 装研究结果和草稿，两者的 `body` 字段不同，所以用了这个较宽泛的类型。如果草稿的结构已经确定，也可以写得更明确：

```ts
type DraftBody = {
  text: string;
  revision: number;
};
```

这样看到 `DraftBody` 时，就能直接知道草稿有正文和修订号。

## 类型不会自动检查运行时数据

TypeScript 的类型用于开发时的类型检查；运行 JavaScript 时，类型本身会被移除。因此，给从 JSON 读入的数据标注一个类型，不会自动检查 JSON 的内容是否正确。

例如，外部文件可能声称状态是 `"finished"`，但 `Result` 只允许 `"completed"` 和 `"stopped"`。如果需要信任这份外部数据，应在读取边界进行实际解析或校验，再交给内部代码使用。

## 延伸阅读

- [TypeScript：类型别名](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-aliases)
- [TypeScript：模块与类型导入](https://www.typescriptlang.org/docs/handbook/2/modules.html)
- [TypeScript：Record 工具类型](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeys-type)
- [TypeScript：unknown](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown)

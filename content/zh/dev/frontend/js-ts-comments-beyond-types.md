---
title: "TypeScript 的类型已经说清楚了，注释还应该写什么？"
date: 2026-10-30
draft: false
categories: ["前端"]
tags: ["JavaScript", "TypeScript", "代码注释", "JSDoc"]
description: "从一个策略工具的函数注释出发，理解 JavaScript 的 JSDoc 类型标注、TypeScript 的文档注释，以及如何用注释说明用途、约束和副作用，避免重复翻译代码。"
---

给函数补注释时，很容易写成这样：

```ts
/**
 * Define a function that takes a strategy and template chosen earlier,
 * asks the existing strategy tool to validate that choice, and returns
 * the writing rules needed by the next stage.
 *
 * @param root - The repository directory containing the strategy tool.
 * @param choice - The strategy and template proposed by the Agent.
 * @returns WritingRules
 */
export function select(root: string, choice: Choice): WritingRules {
  // 实现略。
}
```

这段注释不全是废话。“校验之前的选择”和“仓库目录”都有用，但 `Define a function` 与 `@returns WritingRules` 并没有告诉读者更多信息。

如果你已经会写简单的 JS 或 TS 函数，却不确定注释该写多细，可以从这里入手：**先看名字与类型已经表达了什么，再补上调用者仍然不知道的事。**

## 先区分三种写法

JS 和 TS 都支持下面的注释语法：

```js
// 单行注释，适合解释局部原因。

/* 普通块注释，可以跨行。 */

/**
 * 文档注释，通常紧贴函数、类或属性声明。
 */
```

从语言执行的角度看，它们都是注释，不会自己校验参数或改变函数行为。区别在于工具如何使用它们：编辑器和文档工具可以识别 `/** ... */` 中的说明与标签，用来提供悬浮提示或生成文档。

JSDoc 定义了常用的文档标签和类型表达方式。TSDoc 则为 TypeScript 文档注释提供标准化约定。两者有交集，但并不是所有标签在所有工具中都能通用。项目使用哪种文档工具，就遵循它支持的约定；小项目不必为了写注释先搭建文档系统。

## JavaScript：注释还可以补充类型

JavaScript 函数没有 TypeScript 那样的参数类型语法，JSDoc 可以补上这部分信息。

把下面内容保存为 `reply.js`：

```js
// @ts-check

/**
 * Build reply text with a configurable prefix.
 *
 * @param {string} text - Reply body supplied by the caller.
 * @param {string} [prefix="Reply"] - Label placed before the body.
 * @returns {string} Text in the form "prefix: body".
 */
function formatReply(text, prefix = "Reply") {
  return `${prefix}: ${text}`;
}

console.log(formatReply("先确认工具调用是否完整。"));
```

运行 `node reply.js`，得到：

```text
Reply: 先确认工具调用是否完整。
```

`@param {string}` 给出类型，后面是参数名和含义；`[prefix="Reply"]` 描述可选参数及默认值。真正设置默认值的是函数中的 `prefix = "Reply"`，不是注释。

文件顶部的 `// @ts-check` 是 TypeScript 工具识别的指令。使用支持 TypeScript 的编辑器，或安装 TypeScript 后执行 `tsc --allowJs --checkJs --noEmit reply.js`，可以检查这类 JS 文件。

例如将调用改成 `formatReply(123)`，类型检查会报错，但 Node.js 运行时仍会把数字拼进字符串。**JSDoc 和 `@ts-check` 不会替你添加运行时校验。** 是否需要校验，要根据实际输入边界决定。

## TypeScript：类型留在代码里，注释解释含义

同一个函数改成 TS，可以写成：

```ts
/**
 * Build reply text with a configurable prefix.
 *
 * @param text - Reply body supplied by the caller.
 * @param prefix - Label placed before the body.
 * @returns Text in the form "prefix: body".
 */
function formatReply(text: string, prefix: string = "Reply"): string {
  return `${prefix}: ${text}`;
}

console.log(formatReply("先确认工具调用是否完整。"));
```

这里不需要再写 `@param {string}`，也不需要写 `@returns string`。类型已经由签名表达，注释负责解释参数用途和返回值的形式。

这是一份用于对照语法的完整示例，**不意味着这么简单的函数也必须写满所有标签**。如果内部调用者看名字和实现就能理解，删掉整段文档注释也可以。对外提供的库函数，则更值得补齐调用约定。

常用标签可以按需要选择：

| 标签 | 值得补充的信息 |
| --- | --- |
| `@param` | 单位、路径基准、参数用途等类型表达不出的含义 |
| `@returns` | 排序、快照、空结果等返回语义 |
| `@throws` | 实现确实会抛出、且调用者需要处理的错误条件 |
| `@example` | 不直观的调用方式与预期结果 |
| `@deprecated` | 弃用原因和可替代的接口 |

`@throws` 是文档约定，不是 TypeScript 的强制异常声明。不要因为模板里有这个标签，就推测函数会抛出哪些异常。

## 回到策略工具：哪些信息应该留下？

对于开头的 `select`，按提供的描述，可以先改为：

```ts
/**
 * Validate the Agent's proposed selection using the existing strategy tool
 * and return the writing rules for the next stage.
 *
 * @param root - Repository directory containing the strategy tool.
 * @param choice - Strategy and template proposed by the Agent.
 */
export function select(root: string, choice: Choice): WritingRules {
  // 实现略。
}
```

这段是注释改写示意，依赖原项目里的 `Choice`、`WritingRules` 和函数实现，不能单独运行。

保留 `root` 的说明，是因为 `string` 不能告诉读者它指向哪个目录。保留“校验 Agent 提议的选择”，是因为 `select` 这个名字可能让人误以为函数负责挑选策略。

至于下面这些问题，需要阅读实现才能补充：

- `root` 可以是相对路径吗？相对于哪里？
- 调用工具是否会启动子进程或写文件？
- 选择无效时，是抛异常还是返回其他状态？
- 返回的是某一版本的规则快照，还是其他形式的结果？

这些信息通常比重复参数类型更有价值，但不能靠想象填进去。尤其不能在没核实实现时写下“无副作用”或“保证返回有效规则”。

如果实现确实只校验已有选择，`validateSelection` 可能比 `select` 更准确。如果主要职责是校验并加载规则，也可以考虑 `resolveWritingRules`。名字取决于真实行为，不必仅凭这一段注释就重命名。

## 函数内部：解释为什么要这么做

下面这句只是翻译代码：

```ts
// 把版本号加一。
const nextRevision = currentRevision + 1;
```

如果系统确实保留旧版规则供历史选择引用，更有用的注释是：

```ts
// 编辑生成新版本，避免改变历史选择引用的规则。
const nextRevision = currentRevision + 1;
```

第二句说明了设计原因，但这一行加法本身并不能保证旧版本不被修改，仍要由后续实现兑现。注释应解释已经存在的规则，不能代替实现。

也不是所有注释都必须回答“为什么”。对外函数的用途概述、格式示例、单位和限制同样有用。要避免的是逐句翻译读者已经看懂的代码。

下次给函数写注释，可以先删掉 `This function...`、`Define a function...` 之类的开场，再检查剩下的句子：它是否让调用者知道了名字和类型没有表达的事情？如果没有，先考虑删除；如果有，就把那个事实直接写出来。

## 参考

- [TypeScript：支持的 JSDoc 类型与标签](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html)
- [JSDoc：`@param`](https://jsdoc.app/tags-param)
- [TSDoc：TypeScript 文档注释约定](https://tsdoc.org/)

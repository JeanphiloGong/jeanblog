---
title: "command.slice(1) 为什么能取出程序参数？"
date: 2026-11-04
draft: false
categories: ["前端"]
tags: ["JavaScript", "Node.js", "数组", "子进程"]
description: "从 spawn(command[0], command.slice(1)) 出发，理解数组下标、slice 的起止范围、新数组与浅拷贝，以及程序名和参数数组为什么要分开传递。"
---

读 Node.js 子进程代码时，可能会遇到：

```js
spawn(command[0], command.slice(1));
```

为什么第一个参数写 `[0]`，第二个却写 `slice(1)`？它们都在访问数组，但取得的东西不同：**`[0]` 取一个元素，`slice(1)` 从下标 1 开始取出后面的元素，组成一个新数组。**

这篇用一个命令数组把它拆开，适合刚接触 JavaScript 数组、正在阅读任务执行代码的读者。

## 先看 command 里面放了什么

假设数组是：

```js
const command = ["node", "worker.js", "topic-a"];
// 下标：           0          1           2
```

这里约定第一个元素是程序名，其余元素是要传给程序的参数。数组本身不会识别哪个元素是程序、哪个是参数，这是这段代码采用的数据约定。

数组下标从 0 开始，因此：

```js
command[0]; // "node"
command[1]; // "worker.js"
command[2]; // "topic-a"
```

`command[0]` 得到的是字符串 `"node"`，不是只包含一个元素的数组。

但启动程序时，参数可能不止一个。我们需要的是从第二个元素开始的整段内容，这时就可以使用 `slice`。

## slice 的数字表示下标，不是数量

常见写法是：

```js
array.slice(start, end);
```

对于这里使用的非负下标，它会取出从 `start` 开始、到 `end` 之前的元素：**包含开始位置，不包含结束位置。** 省略 `end`，就一直取到数组末尾。

所以：

| 表达式 | 结果 | 含义 |
| --- | --- | --- |
| `command[0]` | `"node"` | 取出下标 0 的一个元素 |
| `command.slice(1)` | `["worker.js", "topic-a"]` | 从下标 1 取到末尾 |
| `command.slice(1, 2)` | `["worker.js"]` | 取下标 1，不包含下标 2 |
| `command.slice(0, 2)` | `["node", "worker.js"]` | 取下标 0 和 1 |
| `command.slice(0, 1)` | `["node"]` | 返回数组，与 `command[0]` 不同 |
| `command.slice(3)` | `[]` | 开始位置已到末尾，没有元素可取 |

尤其注意 `slice(1, 2)`：第二个数字 2 是结束下标，**不是“取两个元素”**。

`slice` 也支持负下标，例如 `command.slice(-1)` 得到 `["topic-a"]`，表示从倒数第一个位置取到末尾。不过理解当前的命令拆分，只需要先掌握 `slice(1)`。

## slice 返回新数组，不会删除原数组的内容

可以直接运行这个例子：

```js
const command = ["node", "worker.js", "topic-a"];
const args = command.slice(1);

console.log(JSON.stringify(args));
console.log(JSON.stringify(command));

args.push("--verbose");

console.log(JSON.stringify(args));
console.log(JSON.stringify(command));
```

输出：

```text
["worker.js","topic-a"]
["node","worker.js","topic-a"]
["worker.js","topic-a","--verbose"]
["node","worker.js","topic-a"]
```

`args` 和 `command` 是两个数组。往 `args` 中添加元素，不会给 `command` 添加元素。

不过，`slice` 是**浅拷贝**，不是复制所有嵌套数据。如果元素是对象，新旧数组中的元素仍然可以指向同一个对象：

```js
const original = [{ topic: "a" }];
const copied = original.slice();

copied[0].topic = "b";
console.log(original[0].topic); // "b"
```

命令示例的元素都是字符串，没有这个嵌套对象修改问题。

还要避免把 `slice` 与名字相近的 `splice` 混淆：`slice` 取出一段作为新数组，`splice` 可以直接修改原数组。这里需要拆分参数，没有必要删除原命令数组中的元素。

## 为什么 spawn 要这样拆？

Node.js 的 `spawn` 常用形式是：

```js
spawn(program, args);
```

第一个参数是要启动的程序，第二个是传给程序的参数数组。因此：

```js
spawn(command[0], command.slice(1));
```

就相当于：

```js
spawn("node", ["worker.js", "topic-a"]);
```

对应命令行中常见的写法：

```sh
node worker.js topic-a
```

这里启动的程序是 `node`。`worker.js` 是交给 Node.js 执行的脚本，`topic-a` 则是脚本进一步读取的参数。

这只是对启动意图的对应说明。**默认情况下，spawn 不通过 shell 执行命令。** 参数已经由数组分好了，不需要再拼成一条命令字符串。

例如 `["worker.js", "topic a"]` 中的 `"topic a"` 会作为一个参数传给程序，不会仅因为含有空格就自动拆成两个。也不用为了空格把额外的引号字符塞进参数值里。使用 `shell: true` 会改变处理方式，不属于这里的示例。

## 一个不依赖 worker.js 的完整例子

为了直接看到子进程收到的参数，可以让 Node.js 执行一小段代码。把下面内容保存为 `spawn-demo.cjs`：

```js
const { spawn } = require("node:child_process");

const command = [
  process.execPath,
  "-e",
  "console.log(JSON.stringify(process.argv.slice(1)))",
  "topic-a",
  "topic b",
];

const child = spawn(command[0], command.slice(1), {
  stdio: "inherit",
});

child.on("error", (error) => {
  console.error(error.message);
  process.exitCode = 1;
});

child.on("close", (code, signal) => {
  console.log(`子进程结束：code=${code}, signal=${signal}`);
});
```

执行：

```sh
node spawn-demo.cjs
```

正常情况下输出：

```text
["topic-a","topic b"]
子进程结束：code=0, signal=null
```

`process.execPath` 是当前 Node.js 可执行文件的路径；`-e` 让子进程执行紧随其后的代码字符串；`stdio: "inherit"` 让子进程直接向当前终端输出。

这里有两处 `slice(1)`，它们作用相同，但处理的是不同的数组：

- 外面的 `command.slice(1)`：去掉程序路径，取得传给 Node.js 的参数。
- 代码字符串里的 `process.argv.slice(1)`：在本例的 `node -e` 模式下，跳过可执行文件路径，打印用户参数。

若改为普通的 `node worker.js topic-a`，`process.argv` 的前两项通常是可执行文件路径和脚本路径，脚本读取用户参数时一般使用 `process.argv.slice(2)`。起点取几，取决于数组里前面放了什么。

这个例子还展示了 `"topic b"` 保持为一个参数。`close` 监听器只报告进程结束信息，不是完整的任务成功判定逻辑。

最后别把拆分当成校验：空数组调用 `slice(1)` 仍然得到 `[]`，但它的 `command[0]` 是 `undefined`，不能据此正常启动程序。示例的前提是命令数组已经按约定提供了有效程序。

下次看到这行代码，可以直接读成：**取第一个元素作为程序，把剩下的元素作为参数数组，交给 spawn 启动。**

## 参考

- [MDN：Array.prototype.slice()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/slice)
- [Node.js：child_process.spawn()](https://nodejs.org/api/child_process.html#child_processspawncommand-args-options)
- [Node.js：process.argv](https://nodejs.org/api/process.html#processargv)

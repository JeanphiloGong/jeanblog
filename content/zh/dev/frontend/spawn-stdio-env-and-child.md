---
title: "spawn 的第三个参数怎么读？从 stdio、env 到 child"
date: 2026-11-04
draft: false
categories: ["前端"]
tags: ["JavaScript", "Node.js", "子进程", "环境变量"]
description: "拆解 spawn(program, args, options) 的启动配置，用父子进程实验理解 stdio 的三个位置、env 的对象展开，以及 ChildProcess 对象与进程结果的区别。"
---

理解了 [command[0] 和 command.slice(1) 如何拆分命令]({{< relref "array-slice-and-spawn-arguments.md" >}})之后，再看下面这段代码，容易被第三个参数里的数组和对象绕晕：

```js
const child = spawn(command[0], command.slice(1), {
  stdio: ["ignore", "ignore", "pipe"],
  env: { ...process.env, PI_WORKER_ID: workerId },
});
```

其实这里只给 `spawn` 传了三个参数：**启动哪个程序、给它哪些命令行参数、用什么配置启动。** `stdio` 和 `env` 是第三个参数对象里的两个属性，不是额外的第四、第五个参数。

这篇适合能读懂数组和简单对象、正在接触 Node.js 子进程的读者。

## 先把三个参数展开

假设前面有：

```js
const command = ["node", "worker.js", "topic-a"];
const workerId = "worker-123";
```

那么启动代码相当于：

```js
const child = spawn(
  "node",
  ["worker.js", "topic-a"],
  {
    stdio: ["ignore", "ignore", "pipe"],
    env: {
      ...process.env,
      PI_WORKER_ID: "worker-123",
    },
  }
);
```

可以先忽略内部细节，按下面的结构读：

```text
spawn(
  程序名,
  参数数组,
  启动配置对象
)
```

第三个参数是一个对象，它的 `stdio` 属性是数组，`env` 属性又是对象。对象和数组可以互相嵌套，并没有因为多写了一层大括号就多出一个函数参数。

启动意图对应于命令行里的 `node worker.js topic-a`。不过 `spawn` 默认不经过 shell，而是分别接收程序和参数。

## stdio 的前三个位置，分别管三条通道

一个进程通常有三条标准输入输出通道：

| 数组位置 | 通道 | 方向与常见用途 |
| --- | --- | --- |
| 0 | stdin，标准输入 | 子进程读取输入 |
| 1 | stdout，标准输出 | 子进程写普通输出，如 console.log |
| 2 | stderr，标准错误 | 子进程写诊断输出，如 console.error |

因此这里的配置可以逐项翻译：

```js
stdio: [
  "ignore", // stdin：连接空设备，不从父进程接收输入
  "ignore", // stdout：连接空设备，普通输出被丢弃
  "pipe",   // stderr：创建管道，父进程可以读取
]
```

`ignore` 不是“忽略这个配置，使用默认值”，而是让对应通道连接到空设备。子进程仍然可以执行 `console.log`，只是这些输出不会显示在父进程终端里。

`pipe` 也不表示自动显示。父进程需要读取 `child.stderr`，例如：

```js
child.stderr.on("data", (chunk) => {
  process.stderr.write(chunk);
});
```

这里把读到的数据转写到父进程的标准错误输出。一次 `data` 事件不保证刚好是一行，也不保证对应一次 `console.error`；管道传输的是数据流。

另一个常见选择是 `inherit`，让子进程直接使用父进程对应的标准流。比如 `stdio: "inherit"` 对三个标准通道都采用继承方式，适合直接在终端展示输出。

对于当前的数组配置，只有 stderr 使用管道，所以 `child.stdin` 和 `child.stdout` 是 `null`，`child.stderr` 是可读流。切换 `stdio` 配置后，读取代码也要随之调整。

## env：不是再传一个命令行参数

接着看另一个属性：

```js
env: {
  ...process.env,
  PI_WORKER_ID: workerId,
}
```

`process.env` 是父进程当前的环境变量对象。`...process.env` 把其中的属性展开到一个新对象中，再通过后面的属性设置工作者标识。

可以把它理解为：

```text
复制父进程已有的环境变量
        ↓
新增或覆盖 PI_WORKER_ID
        ↓
把这个新对象作为子进程的环境
```

对象属性按书写顺序处理。同名属性以后面的为准，因此这里即使原来已有 `PI_WORKER_ID`，也会被 `workerId` 覆盖。

`PI_WORKER_ID` 是应用自己约定的变量名，并不是 Node.js 内置的工作者编号。子进程通过下面的方式读取：

```js
const workerId = process.env.PI_WORKER_ID;
```

这和命令行参数是两条不同的传递路径：

| 父进程传递方式 | 子进程读取方式 |
| --- | --- |
| 参数数组中的 `"topic-a"` | 在脚本模式下读取 `process.argv` |
| env 中的 `PI_WORKER_ID` | 读取 `process.env.PI_WORKER_ID` |

如果省略 `env`，Node.js 默认使用父进程的环境。**如果提供了 env 对象，就不会自动再把父进程的整个环境合并进去**，所以这里显式写了 `...process.env`，保留程序可能需要的既有配置。

构造这个新对象不会修改父进程的 `process.env`。环境变量也不是父子进程间实时同步的数据通道；子进程启动后，双方再改自己的变量，不会自动更新对方。

## 跑一个实验：普通输出消失，错误输出被接住

在同一个目录创建两个文件。

先写子进程脚本 `worker.cjs`：

```js
console.log("子进程普通输出：这行会被丢弃");
console.error(`子进程 workerId=${process.env.PI_WORKER_ID}`);
console.error(`子进程 topic=${process.argv[2]}`);
```

再写父进程脚本 `parent.cjs`：

```js
const { spawn } = require("node:child_process");
const path = require("node:path");

const workerId = "worker-123";
const command = [
  process.execPath,
  path.join(__dirname, "worker.cjs"),
  "topic-a",
];

const child = spawn(command[0], command.slice(1), {
  stdio: ["ignore", "ignore", "pipe"],
  env: { ...process.env, PI_WORKER_ID: workerId },
});

child.stderr.on("data", (chunk) => {
  process.stderr.write(chunk);
});

child.on("error", (error) => {
  console.error(`子进程错误：${error.message}`);
  process.exitCode = 1;
});

child.on("close", (code, signal) => {
  console.log(`子进程结束：code=${code}, signal=${signal}`);
});
```

运行：

```sh
node parent.cjs
```

你会看到下面这些内容，但看不到“子进程普通输出”那一行：

```text
子进程 workerId=worker-123
子进程 topic=topic-a
子进程结束：code=0, signal=null
```

这里 `process.execPath` 使用当前 Node.js 的可执行文件路径，`path.join(__dirname, "worker.cjs")` 则定位到父脚本旁边的子脚本。这样不用依赖从哪个目录执行父脚本。

前两行经 stderr 输出，最后一行经 stdout 输出。如果把两条流分别重定向或交给外部日志系统，展示顺序不一定与上面的排版一致。

这个实验还有一个值得注意的结果：虽然子进程调用了 `console.error`，退出码仍然是 0。**写入 stderr 不等于进程执行失败。** 判断任务成功与否，需要结合退出码和应用自己的约定。

## child 是操作对象，不是执行结果

`spawn` 返回的是 `ChildProcess` 对象。通常子进程还在运行，父进程就已经拿到了 `child`，所以可以立即注册事件监听器或读取它的流。

它不是 Promise，也不是“进程已经结束”的结果。`await spawn(...)` 不会替你等到子进程结束。

常见操作包括：

```js
child.on("close", (code, signal) => {
  // 子进程已结束，标准输入输出流也已关闭。
});

child.on("error", (error) => {
  // 例如程序未能启动；具体处理取决于调用场景。
});
```

如果需要等待事件后继续执行，可以参考 [Promise 如何把事件转换为可等待的结果]({{< relref "promise-resolve-reject-and-return.md" >}})。这里的示例只演示启动配置和事件读取，不是完整的任务执行器。

另外，`child.kill()` 用于发送终止信号；默认信号是 `SIGTERM`。在支持相应信号语义的系统中，`child.kill("SIGKILL")` 可请求强制终止进程。不要把方法名或返回值理解成“已经确认进程结束”，仍应通过事件确认，且不同操作系统的信号行为有差异。

把这段代码连起来读，就是：**启动程序，丢弃普通输出，把诊断输出接回父进程，同时通过环境变量告诉子进程自己的工作者标识。**

## 参考

- [Node.js：child_process.spawn()](https://nodejs.org/api/child_process.html#child_processspawncommand-args-options)
- [Node.js：options.stdio](https://nodejs.org/api/child_process.html#optionsstdio)
- [Node.js：child.kill()](https://nodejs.org/api/child_process.html#childkillsignal)
- [Node.js：process.env](https://nodejs.org/api/process.html#processenv)

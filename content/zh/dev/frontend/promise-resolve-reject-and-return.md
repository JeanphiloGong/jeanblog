---
title: "Promise 里的 resolve 和 reject 从哪里来，为什么不能直接 return？"
date: 2026-11-04
draft: false
categories: ["前端"]
tags: ["JavaScript", "TypeScript", "Promise", "异步编程"]
description: "拆开 new Promise<void>((resolve, reject) => {})，解释回调参数由谁提供、resolve 与 return 的区别，以及 Promise 的结果如何通过 await 传给调用者。"
---

看到下面的代码，你可能认识每个单词，却不明白它们为什么这样组合：

```ts
await new Promise<void>((resolve, reject) => {
  // 在合适的时候通知成功或失败。
});
```

`resolve` 和 `reject` 是返回类型吗？是提前定义的函数吗？为什么 `resolve(100)` 能传出 100，而不能直接写 `return 100`？

这篇适合已经认识普通函数和参数、但刚开始接触 Promise 的读者。先记住三个角色：**`new Promise(...)` 创建表示结果的对象，构造函数把两个通知函数传给你的回调，`await` 等待这个对象的结果。**

上面的片段只展示结构：它没有通知成功或失败，也没有启动任何工作，不能当作完整的等待示例。

## 先看懂：函数也能作为参数传进来

暂时不用 Promise，先看普通 JavaScript：

```js
function run(callback) {
  const report = (value) => {
    console.log(`收到结果：${value}`);
  };

  callback(report);
}

run((done) => {
  done(100);
});
```

运行后输出：

```text
收到结果：100
```

`done` 不是内置关键字，也不需要在外面提前声明。它是回调函数的参数名：`run` 调用 `callback(report)` 时，把 `report` 这个函数传进去，回调内部用 `done` 接收它。

这个例子只是演示“函数作为参数传递”，不是 Promise 的实现。

Promise 的构造函数也会调用你提供的回调，并传入函数。区别是，它提供的两个函数负责处理所创建的 Promise 的结果。

## resolve 和 reject 是 Promise 传进来的参数

把原来的写法展开一点：

```ts
const promise = new Promise<number>(
  (resolve, reject) => {
    resolve(100);
  }
);
```

其中 `(resolve, reject) => { ... }` 是你写的箭头函数，通常叫作 executor，中文可称“执行器”。括号里的两个名字是**参数**，不是返回类型。

创建 Promise 时，JavaScript 会立即调用这个执行器，并按顺序传入两个函数：

| 参数位置 | 常用名字 | 用途 |
| --- | --- | --- |
| 第一个 | resolve | 用一个值确定 Promise 的结果，或让它跟随另一个 Promise 的结果 |
| 第二个 | reject | 让 Promise 以给定原因为失败 |

名字可以改，位置和作用不变。例如 `new Promise<number>((done, fail) => { done(100); })` 使用的机制完全相同。通常使用 `resolve` 和 `reject`，只是为了让别人更容易读懂。

它们不是 TypeScript 特有的全局函数。在普通 JavaScript 中，同样有 `new Promise((resolve, reject) => { ... })`，只是没有 `<number>` 这段类型标注。

## resolve(100) 把结果交给谁？

看一个完整的 TypeScript 示例：

```ts
async function main() {
  const promise = new Promise<number>((resolve) => {
    resolve(100);
  });

  const result = await promise;
  console.log(result); // 100
}

main().catch(console.error);
```

这里的结果传递路线是：

```text
执行器调用 resolve(100)
        ↓
这个 Promise 以 100 为成功结果
        ↓
await promise 得到 100
        ↓
result 被赋值为 100
```

`resolve(100)` 没有把 100 作为自己的函数返回值返回。它通知的是那个 Promise；`await` 之后从 Promise 的结果中得到 100。

这里传的是普通数字，所以 Promise 可以直接成功完成。如果传给 `resolve` 的是另一个 Promise，它会跟随那个 Promise 的最终结果，不能一概理解为“调用 resolve 的瞬间就成功了”。

## 为什么 return 100 不行？

因为那是从**执行器函数**返回，而 Promise 构造函数不使用执行器的返回值。

下面是一个可直接用 Node.js 运行的反例：

```js
const promise = new Promise(() => {
  return 100;
});

promise.then(() => {
  console.log("不会执行到这里");
});

console.log("执行器已经返回，但 Promise 仍未完成");
```

这里只会输出最后一行。没有调用 `resolve` 或 `reject`，也没有同步抛出异常，因此这个 Promise 一直处于等待状态。

这个例子没有安排任何定时器或其他异步资源，Node.js 可以正常退出。一个未完成的 Promise 本身不会让 Node.js 进程一直运行。

还要区分这几种写法：

| 写法 | 实际作用 |
| --- | --- |
| `resolve;` | 读取函数引用，没有调用它 |
| `resolve(100);` | 调用通知函数，把 100 交给 Promise |
| `return 100;` | 退出当前执行器；返回值被 Promise 构造函数忽略 |
| `return resolve(100);` | 先通知 Promise，再退出执行器；不是靠 return 交付 100 |

而且，**调用 resolve 不会自动退出执行器**：

```js
const promise = new Promise((resolve) => {
  console.log("1. 执行器开始");
  resolve(100);
  console.log("2. resolve 后的代码仍然执行");
});

promise.then((value) => console.log(`4. 收到 ${value}`));
console.log("3. 同步代码继续");
```

输出顺序是 1、2、3、4。执行器会立即执行，Promise 的 `.then` 回调则通过微任务在当前同步代码结束后运行。

不要把这个规则推广成“异步代码不能 return”。在 `async function` 中，`return 100` 会让该函数返回的 Promise 以 100 成功完成；在 `.then` 回调中，返回值也会影响它产生的新 Promise。**被忽略的是 `new Promise` 执行器的返回值。**

## Promise<void> 又是什么？

`Promise<number>` 描述成功结果是数字；`Promise<void>` 则表达调用者不需要使用一个成功结果值。它们都与执行器有几个参数无关。

比如模拟“等待水烧开，再泡茶”：

```ts
async function main() {
  console.log("开始烧水");

  await new Promise<void>((resolve) => {
    setTimeout(() => {
      console.log("水烧开了");
      resolve();
    }, 100);
  });

  console.log("开始泡茶");
}

main().catch(console.error);
```

这里的定时器只是模拟等待，不是在控制真实水壶。执行器立即登记定时器，然后就返回了，但 Promise 还没完成。大约 100 毫秒后，回调调用 `resolve()`，当前异步函数才能越过 `await`，继续“开始泡茶”。定时器不保证精确在 100 毫秒时执行。

`resolve()` 不带值，这个示例中的成功结果是 `undefined`；`void` 表达的是“不需要消费结果”的类型约定，不是运行时生成了一个叫 void 的值。

如果改为 `Promise<number>` 并调用 `resolve(100)`，调用者就可以写 `const temperature = await promise` 接收数字。类型标注只用于静态检查，不会自动把字符串转换成数字。

`await` 暂停的是当前异步函数的后续执行，不是阻塞整个 JavaScript 线程。其他已经安排好的任务仍有机会运行；但同一个顺序循环中位于 `await` 后面的下一轮，不会因此自动并发执行。

## reject 让等待者走失败路径

成功通过 `resolve` 交付结果；失败可以通过 `reject` 交付原因：

```ts
async function main() {
  try {
    await new Promise<void>((resolve, reject) => {
      setTimeout(() => {
        reject(new Error("水壶故障"));
      }, 100);
    });

    console.log("开始泡茶"); // 本例不会执行
  } catch (error) {
    console.log(error instanceof Error ? error.message : String(error));
  }
}

main().catch(console.error);
```

这里输出“水壶故障”。`reject` 让 Promise 失败，`await` 把这个失败表现为抛出错误，由外围 `catch` 接住。

`Promise<void>` 只描述成功结果，不表示“不会失败”，也没有为失败原因指定一个独立的泛型类型。JavaScript 允许用任意值作为失败原因，实际代码通常使用 `Error`，以便保留消息和调用栈。

执行器中**同步**抛出的异常也会让 Promise 失败。但以后在定时器回调里抛出异常，不会被 Promise 构造函数自动捕获；这个例子明确使用 `reject` 传递异步失败。

## 回到子进程：事件如何接到 await 上？

子进程 API 常通过事件通知结束，而 `await` 等待的是 Promise。手动构造 Promise，可以把这两种通知方式接起来。

例如，约定“收到 close 事件就算等待结束”，下面这个片段展示事件与 Promise 的连接：

```ts
await new Promise<void>((resolve) => {
  child.once("close", () => {
    resolve();
  });
});
```

它不是完整的子进程执行器：`child` 需要由外部提供，并且必须在事件发生前注册监听。实际实现还需要处理启动错误，以及退出码不为零时算不算失败等问题。

**close 表示子进程已结束且标准输入输出流已关闭，不等于命令执行成功。** Promise 如何成功或失败，取决于你在事件处理器中实现了什么规则，而不是写上 `Promise<void>` 就自动拥有这些能力。

## 跑一遍最小例子

将前面的“烧水”完整示例保存为 `promise-demo.ts`。在已安装 TypeScript 的环境中执行：

```sh
tsc --strict --target ES2020 promise-demo.ts
node promise-demo.js
```

预期依次输出“开始烧水”“水烧开了”“开始泡茶”。再换成失败示例，观察 `await` 后面的代码如何被跳过。

如果手里的 API 本来就返回 Promise，通常直接 `await` 它即可，不必再套一层 `new Promise`。手动构造更适合把定时器、事件等回调通知转换成可等待的结果。

读这一行代码时，可以按角色拆开：**小括号里的 resolve、reject 是传进来的函数参数，尖括号里的 void 是成功结果的类型，await 是等待结果的操作。** 它们各管一件事，就不会再混成“两个返回类型”。

## 参考

- [MDN：Promise() constructor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/Promise)
- [MDN：await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
- [TypeScript Handbook：函数的返回类型](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#return-type-annotations)
- [Node.js：子进程 close 事件](https://nodejs.org/api/child_process.html#event-close)

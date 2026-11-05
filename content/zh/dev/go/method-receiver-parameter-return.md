---
title: "Go 方法怎么读：接收者、参数和返回值分别在哪里？"
date: 2026-11-05
draft: false
categories: ["go"]
tags: ["go", "方法", "基础语法"]
description: "用商品能否购买的例子，拆解 Go 方法声明中的接收者、参数名、参数类型和返回类型，说明调用时值如何传入、结果如何返回。"
---

刚开始读 Go 代码时，下面这一行可能比方法里的判断更难理解：

```go
func (p Product) CanBuy(quantity int) bool {
    return p.OnSale && quantity > 0 && p.Stock >= quantity
}
```

`CanBuy` 是谁的方法？`quantity` 是变量名还是返回值？为什么方法名前后各有一对括号？

先用一句话读出来：**给 Product 类型定义一个 CanBuy 方法，接收购买数量 quantity，返回是否可以购买。**

这篇适合已经认识变量和简单条件判断、但不熟悉 Go 方法声明的读者。例子只判断商品是否在售、数量是否为正、库存是否够，不涉及扣减库存或创建订单。

## 先认识商品的数据

商品用一个结构体表示：

```go
type Product struct {
    OnSale bool // 是否在售
    Stock  int  // 库存数量
}
```

创建一个在售、库存为 5 的商品：

```go
product := Product{
    OnSale: true,
    Stock:  5,
}
```

于是 `product.OnSale` 是 `true`，`product.Stock` 是 `5`。这些数据属于商品；本次想买几个，则由调用方法的人另外提供。

## 方法声明中的每一部分

再看完整声明：

```go
func (p Product) CanBuy(quantity int) bool {
    return p.OnSale && quantity > 0 && p.Stock >= quantity
}
```

| 部分 | 含义 |
| --- | --- |
| `func` | 开始声明函数或方法 |
| `(p Product)` | 接收者：方法关联到 Product 类型，方法内用 p 访问这次调用的商品值 |
| `CanBuy` | 方法名，意思是“能否购买” |
| `(quantity int)` | 参数列表：参数名是 quantity，类型是 int |
| 最后的 `bool` | 返回类型：结果是布尔值 |
| `{ ... }` | 方法体：实际执行的代码 |

最容易混淆的两个位置是：

```go
(p Product)    // 接收者名 p，接收者类型 Product
(quantity int) // 参数名 quantity，参数类型 int
```

Go 在这里采用“**名字在前，类型在后**”的写法。`quantity` 是参数名，也是方法内部可以使用的局部变量名。它不是返回值。

方法名后面那对括号放参数，括号后面的 `bool` 才是返回类型。本例没有给返回值起名字。

## p 是哪个商品？quantity 又从哪里来？

假设调用代码是：

```go
requested := 2
allowed := product.CanBuy(requested)
```

`product.CanBuy(...)` 表示在这个商品值上调用方法。方法内部用接收者名 `p` 访问商品，用参数名 `quantity` 接收购买数量。

数据流可以这样看：

```text
product：在售，库存 5        requested：2
          ↓                     ↓
接收者 p：在售，库存 5       参数 quantity：2
          └──────────┬──────────┘
                     ↓
        true && 2 > 0 && 5 >= 2
                     ↓
                   true
                     ↓
            调用者的 allowed
```

`requested` 和 `quantity` 不需要同名。前者是调用者的变量名，后者是方法内部接收这个值的参数名。

你也可以不定义 `requested`，直接传值：

```go
allowed := product.CanBuy(2)
```

这里传入的整数 2，仍然由方法内部的 `quantity` 接收。

## return 返回的是判断结果

方法体只有一行：

```go
return p.OnSale && quantity > 0 && p.Stock >= quantity
```

`&&` 表示逻辑“并且”，所以三个条件都成立，结果才是 `true`：

1. 商品正在销售。
2. 购买数量大于 0。
3. 库存不少于购买数量。

`return` 把这个表达式算出的布尔结果交给调用者，同时结束本次方法执行。

例如，库存为 5 时：

| 商品是否在售 | 想买几个 | 返回结果 |
| --- | --- | --- |
| 是 | 2 | true |
| 是 | 6 | false |
| 是 | 0 | false |
| 否 | 2 | false |

返回 `true` 表示这些条件在此次检查时成立，**不表示已经买到了商品**。这个方法没有修改库存，也没有锁定库存；真正下单时如何保证库存正确，是另一个操作需要负责的事。

## 完整例子：复制就能运行

把下面代码保存为 `main.go`：

```go
package main

import "fmt"

type Product struct {
    OnSale bool
    Stock  int
}

func (p Product) CanBuy(quantity int) bool {
    return p.OnSale && quantity > 0 && p.Stock >= quantity
}

func main() {
    product := Product{OnSale: true, Stock: 5}

    requested := 2
    allowed := product.CanBuy(requested)
    fmt.Println("买 2 个：", allowed)
    fmt.Println("买 6 个：", product.CanBuy(6))
    fmt.Println("买 0 个：", product.CanBuy(0))

    product.OnSale = false
    fmt.Println("下架后买 2 个：", product.CanBuy(2))
    fmt.Println("库存仍然是：", product.Stock)
}
```

安装 Go 后执行：

```sh
go run main.go
```

预期输出：

```text
买 2 个： true
买 6 个： false
买 0 个： false
下架后买 2 个： false
库存仍然是： 5
```

可以把 `Stock: 5` 改为 `Stock: 1`，再观察“买 2 个”的结果。这样就能看到接收者里的商品数据和传入数量如何共同影响返回值。

## 为什么叫方法，而不是普通函数？

区别在于方法声明中有接收者。上面的判断也可以写成普通函数：

```go
func CanBuy(p Product, quantity int) bool {
    return p.OnSale && quantity > 0 && p.Stock >= quantity
}
```

这时两种调用形式分别是：

```go
product.CanBuy(2)   // 方法：商品写在点号前面
CanBuy(product, 2)  // 普通函数：商品作为普通参数传入
```

这个对照帮助我们看清数据从哪里来，并不意味着 Go 的方法与函数在所有语言规则上都完全相同。

还有一点：`(p Product)` 是**值接收者**，本例中的 `p` 是商品值的副本。对这个只含布尔值和整数的结构体，在方法内修改 `p.Stock` 不会改变调用者的 `product.Stock`。以后看到 `(p *Product)`，那是指针接收者，通过它可以修改所指向的商品。值复制也不等于任意嵌套数据都会被深拷贝。

当前的 `CanBuy` 只读取数据，先理解值接收者就够了。读到类似声明时，按这个顺序拆开：**谁接收调用 → 方法叫什么 → 传入什么 → 返回什么**。

## 参考

- [A Tour of Go：Methods](https://go.dev/tour/methods/1)
- [A Tour of Go：Functions](https://go.dev/tour/basics/4)
- [A Tour of Go：Pointer receivers](https://go.dev/tour/methods/4)

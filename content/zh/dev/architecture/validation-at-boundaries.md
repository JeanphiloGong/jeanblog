---
title: "分层之后，为什么每一层还在重复校验？"
date: 2026-10-25
draft: false
categories: ["架构"]
tags: ["领域模型", "Repository", "分层架构", "Python"]
description: "用一个修改商品价格的例子，区分入口类型检查、领域业务约束与存储转换，减少分层系统中的重复校验和无目的抽象。"
---

一个修改商品价格的功能，接口检查价格是不是整数，应用服务检查一次，领域对象检查一次，保存时又检查一次。代码分成了四层，同一个问题也被问了四遍。

这些检查看起来都在提高可靠性，但如果删掉其中一条，我们能说清楚究竟失去了什么保护吗？

之前的[《让领域模型和仓储模型各自负责》]({{< relref "domain-and-repository-boundaries.md" >}})讨论了模型与持久化之间的边界。这篇继续讨论一个更具体的问题：**边界已经划分之后，哪些检查应该保留，哪些可以交给前面的代码保证？**

如果你已经接触过 Python 的 dataclass 和简单分层，但觉得业务代码越来越像校验框架，可以从下面这个改价功能开始。

## 一个价格，为什么检查四遍

假设商品后台收到这样的 JSON：

```json
{
  "sku": "P1001",
  "price_cents": 9900
}
```

价格使用整数分。对于这个例子，业务还约定：价格不能为负数，停售商品不能改价。

容易写出的代码是，在入口、服务、模型和 Repository 中都放上类似判断：

```python
if type(price_cents) is not int:
    raise ValueError("价格必须是整数分")
```

问题在于，这四次检查通常处理的是同一个已经解析完成的值。只要内部代码没有重新引入不可信数据，重复检查提供的保护就很有限。

真正需要区分的是两个问题：

| 问题 | 例子 | 负责的位置 |
|---|---|---|
| 输入是否符合接口约定？ | 缺少价格、价格是字符串 | 外部输入入口 |
| 操作是否符合业务规则？ | 负价格、对停售商品改价 | 领域模型 |

应用服务负责组织操作：加载商品、调用改价方法、保存商品。Repository 负责读写和映射。它们不需要把前两类判断各自重写一份。

## 先建立一个可以信任的入口

Python 的类型注解不会自动验证运行时数据。写下 `price_cents: int`，并不能阻止调用方传入字符串。

因此，“内部可以信任类型”需要前提：外部数据进入内部流程之前，确实经过了解析和检查。

下面是这个入口。示例采用严格协议，只接受两个字段；不自动把字符串价格转换成整数。

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class ChangePriceRequest:
    sku: str
    price_cents: int


def decode_change_price(data: object) -> ChangePriceRequest:
    if not isinstance(data, dict):
        raise ValueError("请求必须是 JSON 对象")
    if set(data) != {"sku", "price_cents"}:
        raise ValueError("请求必须且只能包含 sku 和 price_cents")

    sku = data["sku"]
    if not isinstance(sku, str) or not sku.strip():
        raise ValueError("sku 必须是非空字符串")
    if type(data["price_cents"]) is not int:
        raise ValueError("价格必须是整数分")

    return ChangePriceRequest(sku.strip(), data["price_cents"])
```

这里使用 `type(value) is int`，是因为 Python 的 `bool` 是 `int` 的子类。这个接口不接受 `true` 作为价格。

入口完成之后，内部服务接收的是 `ChangePriceRequest`，不再逐层猜测字典里有什么字段。SKU 的去空格也在这里完成。

## 领域模型保留业务含义

下面的模型不重复检查价格是不是整数，但仍然保证价格不能为负数。

```python
@dataclass(frozen=True)
class Product:
    sku: str
    name: str
    price_cents: int
    on_sale: bool = True

    def __post_init__(self):
        if not self.sku or not self.name:
            raise ValueError("商品 SKU 和名称不能为空")
        if self.price_cents < 0:
            raise ValueError("商品价格不能为负数")

    def change_price(self, price_cents: int) -> "Product":
        if not self.on_sale:
            raise ValueError("停售商品不能改价")
        return replace(self, price_cents=price_cents)
```

这里有两种业务约束。

“价格不能为负数”是对象状态的约束。无论是首次创建商品，还是修改价格，得到的商品都应满足它。`replace()` 创建新对象时会执行 `__post_init__`，所以改价方法不需要再次检查负数。

“停售商品不能改价”是操作的约束。停售商品本身是合法状态，因此这个判断属于 `change_price()`，不能放进构造方法，否则连加载一个停售商品都会失败。

`__post_init__` 本身并不多余。需要审视的是里面的每条判断在维护什么规则，而不是统一删除这个方法。

## 应用服务把操作串起来

为了直接观察数据流，先使用内存仓储，不引入数据库和 Web 框架。以下代码与前面两段放在同一个文件中即可运行：

```python
class MemoryProductRepository:
    def __init__(self, products: list[Product]):
        self.products = {product.sku: product for product in products}

    def get(self, sku: str) -> Product:
        return self.products[sku]

    def save(self, product: Product) -> None:
        self.products[product.sku] = product


def change_product_price(
    repository: MemoryProductRepository,
    request: ChangePriceRequest,
) -> Product:
    product = repository.get(request.sku)
    updated = product.change_price(request.price_cents)
    repository.save(updated)
    return updated


if __name__ == "__main__":
    repository = MemoryProductRepository([
        Product("P1001", "键盘", 12900),
        Product("P1002", "鼠标", 5900, on_sale=False),
    ])
    request = decode_change_price({"sku": " P1001 ", "price_cents": 9900})
    updated = change_product_price(repository, request)
    assert updated.price_cents == 9900
    assert repository.get("P1001") == updated
    print(updated)
```

保存为 `price_example.py`，执行 `python3 price_example.py`，可以看到价格已经变成 `9900`。

数据流只有一条：

```text
外部字典 → 解析为请求对象 → 加载商品 → 执行改价规则 → 保存商品
```

应用服务没有 JSON 解析，没有 `isinstance(request, ...)`，也没有重复判断商品是否停售。读它就能看出这个用例做了什么。

示例用 `KeyError` 表示商品不存在，实际接口可以把它转换成对应的错误响应。它没有演示并发改价；如果多个请求可能同时更新同一商品，还需要版本检查或事务控制，不能用重复类型检查解决覆盖问题。

## 简化之后，错误在哪里被拒绝

分别运行下面的调用，可以观察每个错误的归属：

| 输入或操作 | 拒绝位置 | 原因 |
|---|---|---|
| 价格为 `"9900"` | `decode_change_price()` | 类型不符合输入协议 |
| 价格为 `True` | `decode_change_price()` | 布尔值不属于允许的价格输入 |
| 价格为 `-1` | `Product.__post_init__()` | 违反商品状态约束 |
| 对 `P1002` 改价 | `Product.change_price()` | 违反操作规则 |

改价失败发生在 `save()` 之前，所以仓储中的商品不会被这次操作替换。

例如，在同一文件的演示代码后面加入：

```python
before = repository.get("P1001")
try:
    request = decode_change_price({"sku": "P1001", "price_cents": -1})
    change_product_price(repository, request)
except ValueError as error:
    print(error)
else:
    raise AssertionError("负价格应该被拒绝")
assert repository.get("P1001") == before
```

好的检查不仅说明“出了错会报错”，还说明错误由谁发现，以及失败后是否留下了不应该发生的修改。

## 存储转换也要有明确目的

分层代码里的另一种复杂度，来自没有说清目的的辅助函数：

```python
def record(value):
    return json.loads(json.dumps(asdict(value), ensure_ascii=False))
```

它先把 dataclass 转成字典，再编码成 JSON，最后解码回来。对于兼容 JSON 的字段，这会把元组变成列表，并得到解码后的数据结构。

如果目的只是写入 JSON 文件，就不需要这一来一回。Repository 的保存逻辑可以直接序列化：

```python
data = asdict(product)
serialized = json.dumps(data, ensure_ascii=False, indent=2)
```

这两行仅展示序列化步骤，需要导入 `dataclasses.asdict` 和 `json`。真正的文件写入仍属于 Repository；是否需要原子替换、并发控制，要根据存储场景决定。

但不能因此把已有的 `record()` 全部机械替换掉。如果调用方正在比较历史快照，JSON 中的列表与内存里的元组直接比较可能不相等；如果快照还参与摘要计算，编码方式也是契约的一部分。

先弄清调用方需要的是“领域对象”“可写入 JSON 的数据”，还是“用于比较的统一表示”，再决定是否需要一个转换函数。只为一次保存服务的两行代码，可以先留在保存方法里。

## “入口检查一次”不等于“整个系统只检查一次”

HTTP 请求是入口，导入文件、消息队列和人工编辑的 JSON 文件也可能是入口。

如果 Repository 读取的数据允许被外部修改，就需要在那里检查存储结构并还原模型。那是新的数据边界，不是对同一个内部对象重复检查。构造领域对象时，业务约束仍然生效。

反过来，如果一个公共 Python 库允许调用者直接构造对象，那么构造方法本身就可能是外部入口，类型校验也可能应该放在那里。

因此，是否保留某条检查，要看实际调用路径，不能仅凭文件叫 `domain.py` 还是 `store.py` 决定。类型注解、静态检查和测试可以帮助维护内部契约，但不会把 Python 变成自动执行运行时类型检查的语言。

## 回到自己的代码，先检查一条路径

不必从全项目删除 `isinstance` 开始。选一个具体操作，沿着输入、解析、业务处理和保存走一遍：

1. 标出外部数据真正进入的位置。
2. 确认入口检查后，内部能够依赖哪些约定。
3. 找出表达业务状态和操作约束的判断。
4. 删除没有新增保护作用的重复检查，并验证正常行为和失败后的状态。

前一篇[《让领域模型和仓储模型各自负责》]({{< relref "domain-and-repository-boundaries.md" >}})讲的是如何划分责任。这篇的改价例子进一步要求：责任划分之后，后面的代码应当能够使用前面已经建立的保证。

下一次遇到一个看起来“更安全”的辅助方法，可以先问：**它处理的是哪个边界上的具体风险？如果删除它，哪一种错误会失去防护？**

## 系列阅读：分层与边界

1. [让领域模型和仓储模型各自负责]({{< relref "domain-and-repository-boundaries.md" >}})：模型职责。
2. [分层之后，为什么每一层还在重复校验？]({{< relref "validation-at-boundaries.md" >}})：校验位置。
3. [让 Agent 少写防御性代码：每条校验都要有理由]({{< relref "validation-needs-a-reason.md" >}})：校验的必要性与具体简化方法。

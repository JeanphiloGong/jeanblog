---
title: "仓储有自己的返回模型，不等于每层都要复制领域对象"
date: 2026-10-25
draft: true
categories: ["架构"]
tags: ["领域模型", "Repository", "分层架构", "Python", "AI 编程"]
description: "从策略工具的 Record 转换链出发，区分存储模型、仓储查询结果与接口响应，用可运行的 JSON 仓储例子说明何时组合领域对象、何时才需要独立映射模型。"
---

“仓储应该有明确的返回模型”，很容易被 Agent 实现成另一件事：给每个领域类复制一个 Record，再为每个 Record 添加 `from_domain()` 和 `to_domain()`。

字段几乎一样，只是 tuple 换成 list。读写一次文件，却要经过好几组转换。代码看起来分层完整，读者反而找不到真正读取文件的地方。

我在一个写作策略工具的教程里遇到了这个问题。起初，仓储和接口输出都靠几个函数拼字典；提出“用 dataclass 定义仓储返回结构”后，下一版又走到了另一个极端：策略、模板、修订、材料要求各自多了一套存储类。

这里真正混淆的是两个概念：**存储中如何表示数据，与仓储向调用者返回什么，是两份不同的约定。**

本文接着[《让领域模型和仓储模型各自负责》]({{< relref "domain-and-repository-boundaries.md" >}})讨论这个区别，适合已经认识 dataclass 和 Repository、正在整理小型工具边界的读者。代码用 Python 3.10 标准库运行，不引入 ORM 或映射框架。

## 两个转换方法，分别在做什么

下面是问题的缩小示意，暂不作为推荐实现：

```python
@dataclass
class TemplateRecord:
    name: str
    structure: list[str]

    @classmethod
    def from_domain(cls, template):
        return cls(template.name, list(template.structure))

    def to_domain(self):
        return Template(self.name, tuple(self.structure))
```

`from_domain()` 把领域对象变成存储对象，`to_domain()` 反向恢复。这两个方法有明确作用，也没有天然错误。但如果新类只是为了把 tuple 换成 list，就应该先问：这个中间对象到底由谁使用？

Python 的 JSON 编码器本来就能把 tuple 编码为数组。读取时，在仓储里按字段构造领域对象，也能完成恢复。为了这样一次读写复制一个类，并不是分层本身的要求。

如果数据库列名与领域字段不同，或者旧文件格式要独立演进，映射仍然需要存在。问题在于是否需要额外的类和方法承载它，而不是能否消灭所有转换。

## “仓储模型”这个名字包含了什么

先把三个常被混叫成“仓储模型”的东西分开：

| 类型 | 解决的问题 | 例子 |
|---|---|---|
| 存储表示 | 数据怎样进入数据库或文件 | ORM 行、JSON 文件中的 schema_version |
| 仓储结果 | 一次查询向调用者交付什么 | 商品与更新时间、模板与原始文件字节、分页查询结果 |
| 接口响应 | 客户端或 Agent 能拿到什么 | 展示字段、选择快照、响应中的版本号 |

存储表示不一定需要独立的 Python 类。JSON 可以在仓储方法里直接组织；ORM 则通常已有表达表结构的类。

仓储结果也不一定要复制领域字段。一个已有的招标信息服务中，详情结果组合了 `Tender`、机构信息、附件，以及创建时间、处理状态等读取元数据。列表查询则直接构造只包含列表所需字段的 dataclass。两种结果服务于不同的调用需求。

这类 dataclass 主要是在说“这次查询返回什么”。它可以只有字段，没有转换方法。写入方法接收领域对象后，直接在仓储内部构造数据库记录；不必要求查询结果也能反向转换成可写入的领域对象。

因此，读与写并不一定对称。展示列表中的 `favorited` 可能来自关联查询，它不需要出现在用于修改招标信息的领域对象中。

## 需要原始字节时，组合已有的领域目录

回到策略工具。应用要加载一份目录、修改规则，再保存。保存前还希望发现文件在读取后已经变化，避免用旧内容覆盖新内容。

查询实际需要的是：

```python
@dataclass(frozen=True)
class CatalogData:
    catalog: Catalog
    original_bytes: bytes
```

`catalog` 是恢复好的领域对象，`original_bytes` 是恢复它所用的同一份文件字节。应用使用前者执行规则变化，把后者作为保存的比较依据。

这里不用复制 `Catalog` 的所有字段，也不用给 `CatalogData` 增加 `from_domain()`。它是一次读取的结果，不是领域目录的另一种形态。

如果读取目录只为了展示，完全不需要附带文件字节，那么仓储直接返回 `Catalog` 就可以。给每个返回值都套一层 dataclass，同样会制造多余结构。

还有一个细节：不要先调用一次 `load()` 恢复对象，再独立 `read_bytes()` 取得比较依据。两次读取之间若文件变化，就可能把旧对象和新字节拼在一起。应该先取得一份字节，再从这份字节恢复对象。

## 一个可以运行的 JSON 仓储

下面把场景缩小到只含模板名称和版本的目录。名称不能为空，版本必须大于零；调用者负责在外部入口解析字段类型。目录文件由程序维护，不接受绕过入口的独立编辑，也不承诺为损坏文件逐字段生成友好提示。

`save()` 使用临时文件和替换操作，防止正常写入失败时留下半份目录；`expected_bytes` 用于发现保存前已经发生的变化。示例假定只有一个写入者，比较与替换之间没有锁。

保存为 `catalog_repository.py`，执行 `python3 catalog_repository.py`：

```python
from dataclasses import asdict, dataclass, replace
import json
import os
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory


@dataclass(frozen=True)
class Template:
    name: str
    revision: int

    def __post_init__(self):
        if not self.name.strip() or self.revision < 1:
            raise ValueError("模板名称不能为空，版本必须大于零")


@dataclass(frozen=True)
class Catalog:
    templates: tuple[Template, ...]


@dataclass(frozen=True)
class CatalogData:
    catalog: Catalog
    original_bytes: bytes


class JsonCatalogRepository:
    def __init__(self, path: Path):
        self.path = path

    def load(self) -> CatalogData:
        original_bytes = self.path.read_bytes()
        data = json.loads(original_bytes)
        catalog = Catalog(tuple(Template(**item) for item in data["templates"]))
        return CatalogData(catalog=catalog, original_bytes=original_bytes)

    def save(self, catalog: Catalog, *, expected_bytes: bytes) -> None:
        temporary = None
        try:
            with NamedTemporaryFile(
                mode="w", encoding="utf-8", dir=self.path.parent, delete=False
            ) as stream:
                temporary = Path(stream.name)
                json.dump(asdict(catalog), stream, ensure_ascii=False, indent=2)
            if self.path.read_bytes() != expected_bytes:
                raise ValueError("目录已变化，请重新读取")
            os.replace(temporary, self.path)
        finally:
            if temporary is not None and temporary.exists():
                temporary.unlink()


with TemporaryDirectory() as folder:
    path = Path(folder) / "catalog.json"
    path.write_text(
        '{"templates": [{"name": "问题与解释", "revision": 1}]}',
        encoding="utf-8",
    )
    repository = JsonCatalogRepository(path)
    loaded = repository.load()
    assert loaded.original_bytes == path.read_bytes()
    assert loaded.catalog.templates == (Template("问题与解释", 1),)

    updated = Catalog((replace(loaded.catalog.templates[0], name="机制与边界", revision=2),))
    repository.save(updated, expected_bytes=loaded.original_bytes)
    assert repository.load().catalog == updated
    print("读写往返正确：问题与解释 v1 → 机制与边界 v2")

    saved_bytes = path.read_bytes()
    try:
        repository.save(loaded.catalog, expected_bytes=loaded.original_bytes)
    except ValueError as error:
        assert str(error) == "目录已变化，请重新读取"
    else:
        raise AssertionError("旧结果不应覆盖新目录")
    assert path.read_bytes() == saved_bytes
    assert list(path.parent.iterdir()) == [path]
    print("过期保存被拒绝，现有目录保留，临时文件已清理")
```

这个例子没有独立的 `TemplateRecord`、`CatalogRecord` 或通用 mapper。读取方法直接恢复领域对象，保存方法直接展开字段；`CatalogData` 只承载查询结果。

正常输出是两行：

```text
读写往返正确：问题与解释 v1 → 机制与边界 v2
过期保存被拒绝，现有目录保留，临时文件已清理
```

它验证的是读写往返和过期结果拒绝，不能证明多进程并发安全。两个进程仍可能同时通过比较；`os.replace()` 只负责一次替换，不是条件更新。需要并发写入时，应使用合适的锁或数据库事务，不能把“带了一份原始字节”解释成事务能力。示例也没有承诺掉电后的持久性。

## 接口输出仍然有自己的约定

仓储已经返回领域对象，并不意味着 Agent 可以直接消费整个 `CatalogData`。

例如，Agent 只需要模板名称和当前版本。原始文件字节用于内部保存检查，不属于响应：

```python
@dataclass(frozen=True)
class TemplateResponse:
    name: str
    revision: int


response = TemplateResponse(
    name=loaded.catalog.templates[0].name,
    revision=loaded.catalog.templates[0].revision,
)
```

这是接在上例读取结果之后的接口示意。接口决定公开哪些字段，领域对象提供业务值，仓储结果提供读取事实。即使几个模型当前碰巧有同名字段，也不必因此把它们当作同一个协议。

同样，写作工具的选择快照、选择理由和规则哈希，属于对外输出与交接的约定。它们不应该仅仅因为仓储方便拼字典，就成为仓储文件格式的一部分。

这里独立定义响应的理由是存在一个明确的客户端契约。如果代码只是在内部打印模板名称，不需要为了打印动作另建一套响应类。

## 什么时候值得保留独立存储模型

上面的直接映射适合受控格式和较小的转换。以下情况则有实际理由增加存储模型或专门的映射方法：

- 存储使用旧字段或特殊编码，领域对象已经采用另一种表达，双方要独立演进。
- 需要同时读取不同文件版本，并明确处理数据迁移。
- 映射包含较多嵌套结构，被多个真实读写入口复用，提取后更容易独立验证。
- 使用 ORM，已有表结构、关联和持久化生命周期需要表达。

`from_domain()`、`to_domain()` 可以是组织这些转换的一种方式。它们的存在应该由转换复杂度和复用需求解释，不能由“所有模型都应该有一对转换方法”解释。

另一类独立模型是查询投影：列表只需要名称、状态和统计值，不需要加载完整聚合。这时让仓储直接返回 `TemplateListItem` 很合理，但不必为它提供一个毫无用途的反向转换。

## 把新增模型的理由写进 Agent 规则

[上一篇]({{< relref "validation-needs-a-reason.md" >}})要求每条校验先说明理由。模型和转换也需要同样的约束：

```text
新增模型前，说明谁定义它、谁消费它，以及它与现有模型的实际差异。
不要因为存在一个层，就复制一套字段相同的模型。

区分存储表示、仓储查询结果和接口响应。
查询结果可以组合领域对象与元数据，也可以是按查询需要裁剪的投影。
没有额外结果需求时，直接返回领域对象。

默认在仓储方法内直接映射；只有明确的格式差异、独立版本或真实复用
需要时，才引入 Record、mapper 或双向转换方法。
需要保持一致的对象与元数据，从同一次读取中取得。
```

审查时可以挑一个新类，只问一句：“删掉它以后，哪份真实契约会变得无法表达？”如果仅仅是少了一次 `from_domain()` 调用，却没有丢失任何信息或隔离任何变化，这个类很可能可以省掉。

先找到调用方需要的结果，再决定模型和映射。这样 Repository 依然有明确边界，也不必为每一层维护一份几乎相同的字段清单。

## 系列阅读：分层与边界

1. [让领域模型和仓储模型各自负责]({{< relref "domain-and-repository-boundaries.md" >}})：模型职责与仓储契约。
2. [分层之后，为什么每一层还在重复校验？]({{< relref "validation-at-boundaries.md" >}})：入口与业务规则的分工。
3. [让 Agent 少写防御性代码：每条校验都要有理由]({{< relref "validation-needs-a-reason.md" >}})：如何判断检查是否必要。
4. 本篇：查询结果的组合、存储映射与重复模型的取舍。

参考：[Python dataclass 与 asdict](https://docs.python.org/3/library/dataclasses.html)、[JSON 编码器支持的类型](https://docs.python.org/3/library/json.html#json.JSONEncoder)、[os.replace 的语义](https://docs.python.org/3/library/os.html#os.replace)。

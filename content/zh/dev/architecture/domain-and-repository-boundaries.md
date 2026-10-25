---
title: "让领域模型和仓储模型各自负责"
date: 2026-09-09
draft: false
categories: ["架构"]
tags: ["领域模型", "Repository", "ORM", "分层架构"]
description: "用清晰的边界分开领域模型、仓储契约、ORM 模型和 API 模型，让业务规则不被数据库和接口绑架。"
---

在业务系统逐渐变复杂时，一个对象往往同时被拿来承载业务规则、数据库字段和 HTTP 响应。开始时这很省事，后来却会出现连锁问题：数据库新增一个内部字段，接口结构跟着变化；领域代码为了查询数据而依赖 ORM；仓储把 SQLAlchemy 行对象直接交给控制器。

更稳定的做法，是让不同边界拥有不同模型，并通过转换连接它们：

```text
Controller -> Domain Model -> Repository Contract -> ORM Model -> Database
     |                              ^
     +------ API Request/Response ---+
```

这里的重点不是增加更多目录，而是明确每个模型对谁负责。也不需要为每个领域类都复制一套存储类：仓储结果可以直接组合领域对象与读取元数据，只有确实不同的格式或契约才需要独立表示。

## 四类模型，各自回答一个问题

**领域模型**回答“什么状态符合业务规则”。例如商品必须有 SKU 和名称，价格不能为负数。SKU 是业务标识，数据库自增 `id` 则不必进入领域对象。领域模型不应该导入 SQLAlchemy、Pydantic 或 Web 框架。

**ORM 模型**回答“如何把数据存进数据库”。它可以拥有自增主键、外键、索引、时间戳和数据库约束，但这些细节不必进入领域对象。

**Repository 契约**回答“应用可以怎样保存和读取数据”。它的参数和返回值属于仓储边界：保存商品可以返回 `StoredProduct`，其中包含领域对象和 `updated_at`；分页查询可以返回 `ProductPage` 这样的查询模型。

**API 模型**回答“客户端能发送和接收什么”。请求模型负责输入形状，响应模型负责 HTTP 输出。例如商品响应可以包含 `updated_at`，并将其序列化为日期字符串；这不意味着商品领域对象也需要这个字段。

## 一个最小例子

假设一个后台允许按 SKU 保存商品，再查询商品信息。下面使用同步 SQLAlchemy 2.x 展示关键代码，省略数据库连接和 HTTP 路由配置。

领域对象放在 `app/domain/product.py`，只表达业务状态。价格统一使用人民币分，避免浮点数表示金额：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Product:
    sku: str
    name: str
    price_cents: int

    def __post_init__(self):
        if not self.sku or not self.name:
            raise ValueError("商品 SKU 和名称不能为空")
        if self.price_cents < 0:
            raise ValueError("商品价格不能为负数")
```

数据库模型放在 `infra/db/models.py`。自增主键、唯一约束和更新时间属于存储设计：

```python
from datetime import datetime

from sqlalchemy import DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class ProductRecord(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    sku: Mapped[str] = mapped_column(unique=True)
    name: Mapped[str]
    price_cents: Mapped[int]
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))
```

Repository 及其对外结果类型放在 `app/repositories/product_repository.py`。调用方从这里导入 `StoredProduct`，不需要了解数据库行的结构：

```python
from dataclasses import dataclass
from datetime import datetime, timezone

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.domain.product import Product
from infra.db.models import ProductRecord


@dataclass(frozen=True)
class StoredProduct:
    product: Product
    updated_at: datetime


class ProductRepository:
    def __init__(self, session: Session):
        self.session = session

    def save(self, product: Product) -> StoredProduct:
        record = self.session.scalar(
            select(ProductRecord).where(ProductRecord.sku == product.sku)
        )
        if record is None:
            record = ProductRecord(sku=product.sku)
            self.session.add(record)
        record.name = product.name
        record.price_cents = product.price_cents
        record.updated_at = datetime.now(timezone.utc)
        self.session.flush()
        return StoredProduct(product=product, updated_at=record.updated_at)

    def get_by_sku(self, sku: str) -> StoredProduct | None:
        record = self.session.scalar(
            select(ProductRecord).where(ProductRecord.sku == sku)
        )
        if record is None:
            return None
        return StoredProduct(
            product=Product(
                sku=record.sku,
                name=record.name,
                price_cents=record.price_cents,
            ),
            updated_at=record.updated_at,
        )
```

这里由调用方管理事务并执行提交，`flush()` 本身不代表数据已经提交。示例没有处理并发创建同一 SKU 的冲突，真实应用需要根据唯一约束处理相应的事务失败。

控制器只负责把 API 请求转换成领域对象，再把仓储结果转换成响应模型。ORM 行不会越过 Repository，领域对象也不会知道数据库表名。

## 为什么仓储要拥有自己的返回类型

如果 Repository 返回 ORM 行，调用方会开始读取 `record.id`、`record.price_cents`，数据库结构就变成了隐含的应用接口。以后更换数据库或调整存储格式，影响会扩散到 worker、控制器和测试。

如果 Repository 直接返回 API 响应模型，基础设施层又反过来依赖 Web 层，后台任务和命令行程序很难复用。

因此，需要附带持久化元数据或查询投影时，可以定义仓储自己的命名结果类型；没有额外需求时，也可以直接返回领域对象。仓储结果不应携带 HTTP 包装、状态码或 Pydantic 响应模型。

## 实施时检查四条边界

1. 领域模型的字段和方法是否都能用业务语言解释？
2. ORM 的内部主键、索引和序列化格式是否停留在基础设施层？
3. Repository 是否只暴露稳定、命名明确的参数和结果类型？
4. API 模型是否只在控制器边界出现？

外部的 `sku` 等输入在路由或消息入口规范化一次，例如去除首尾空格；入口同时确保价格是整数，之后 Repository 可以信任已经规范化的值。价格不能为负数则是领域模型负责的业务规则。测试分别验证领域规则、ORM 映射和 API 转换，而不是只验证一条端到端路径。

## 小结

领域模型保护业务规则，ORM 模型服务数据库，Repository 契约隔离持久化细节，API 模型稳定客户端协议。四者之间允许转换，但不互相冒充。这样的组织方式会多出少量映射代码，却能把变化限制在真正发生变化的边界内。

延伸阅读：[《分层之后，为什么每一层还在重复校验？》]({{< relref "validation-at-boundaries.md" >}})用一个修改商品价格的可运行例子，进一步区分入口类型检查、领域业务约束与存储转换。

## 系列阅读：分层与边界

1. [让领域模型和仓储模型各自负责]({{< relref "domain-and-repository-boundaries.md" >}})：模型职责。
2. [分层之后，为什么每一层还在重复校验？]({{< relref "validation-at-boundaries.md" >}})：校验位置。
3. [让 Agent 少写防御性代码：每条校验都要有理由]({{< relref "validation-needs-a-reason.md" >}})：校验的必要性与具体简化方法。

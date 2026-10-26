---
title: "从 Agent 到 JSON 文件：分层系统的数据如何流转"
date: 2026-10-25
draft: true
categories: ["架构"]
tags: ["领域模型", "Repository", "分层架构", "Python", "AI 编程"]
description: "沿一次模板编辑和一次策略选择，追踪 Agent 输入、CLI 解析、应用编排、领域操作、JSON 仓储与响应输出的完整数据流，说明领域模型如何与必要参数共同贯穿业务过程。"
---

一个写作策略工具，Agent 需要查看策略、选择模板，也需要修改模板里的写作规则。为它划分领域层、应用层、仓储层和 CLI 后，接下来最容易困惑的是：一次调用到底传递什么？

Agent 发来的是 JSON，文件里存的也是 JSON。中间经过几层，是否每经过一层都要转换成一个新模型？请求对象能不能进入应用服务？仓储返回对象后，谁负责把它展示给 Agent？

这些问题需要放进同一条数据流里看。

[上一篇]({{< relref "repository-results-without-model-copies.md" >}})区分了存储表示、仓储结果和接口响应。这篇把它们接起来，沿一次模板编辑从输入走到保存和输出，再对照一次只读的策略选择。适合已经认识 dataclass 和简单分层，希望看清各部分如何协作的读者。

**领域模型承载核心业务数据和规则。CLI 处理与 Agent 的交互格式，仓储处理与 JSON 文件的转换，应用层组织整个操作。内部传递现有领域对象和必要参数，不必逐层复制字段。**

## 先把两端与中间画出来

假设 Agent 要把模板的写作结构从“提出问题、解释机制”改成“先给结论、解释机制与边界”，并声明自己依据的是第 1 版。

整条编辑链路如下。读取文件、执行规则、写入文件有先后顺序，并不是让数据穿过四个目录就自动完成了操作。

```text
Agent 提交编辑 JSON
        │
        ▼
CLI 解析输入 → 构造 TemplateRevision
        │
        │  template_id + proposed + expected_revision
        ▼
应用层开始一次编辑
        │
        ├─ ① 仓储读取 JSON 文件
        │       └─ 恢复 Template，附带同一次读取的 original_bytes
        │
        ├─ ② 调用 Template.revise(proposed, expected_revision=...)
        │       └─ 检查版本，产生更新后的 Template
        │
        ├─ ③ 仓储保存更新后的 Template
        │       └─ 映射为 JSON，检查文件是否已变化，再替换
        │
        └─ ④ 返回更新后的 Template
                │
                ▼
CLI 构造 TemplateResponse → 编码为 JSON → Agent
```

CLI 与仓储面对的是两个不同的外部约定。Agent 的输入字段可以叫 `template_id`，文件里的标识可以叫 `id`；Agent 响应可以只返回当前版本和写作结构，文件则保留存储所需的结构。

两端都使用 JSON，也不意味着接口协议和存储格式必须相同。

## 第一步：入口把外部格式变成可用的业务输入

Agent 提交的编辑请求可以是：

```json
{
  "template_id": "question-answer",
  "expected_revision": 1,
  "structure": ["先给结论", "解释机制与边界"]
}
```

CLI 首先按照输入协议检查字段和类型，再把 `structure` 转成领域模型使用的 tuple，构造准备采用的 `TemplateRevision`。这里约定新修订接续 Agent 声明的版本，因此拟定版本为 `expected_revision + 1`，随后还要与仓储中的真实版本核对。

接下来传给应用函数的，是：

| 值 | 表达什么 |
|---|---|
| `template_id` | 修改哪个已有模板 |
| `proposed: TemplateRevision` | 准备采用的新修订内容 |
| `expected_revision` | 这次修改依据哪个旧版本 |

接口输入对象在这里完成任务。应用函数不需要继续接收 `EditTemplateInput`，也不需要为这三个值再复制一个 Command。

这不是禁止应用层拥有输入模型。如果命令需要排队、持久化或作为整体多处传递，可以再定义。但在当前流程里，领域对象与明确参数已经能表达调用。

## 第二步：应用层加载真实状态，领域对象决定能不能改

Agent 提供的是修改意图，不能把它声明的版本当作文件里的当前状态。

应用层先请仓储读取模板。仓储负责从 JSON 恢复 `Template`，应用层取得的就是可以执行业务操作的对象，不需要再次遍历 JSON 字段。

然后调用模板的修订方法，检查两件事：

- 当前版本是否仍等于 `expected_revision`。不相等，说明这次修改依据已经过期。
- 新修订是否接续当前版本。不能从第 1 版直接跳到第 8 版。

“加载、调用、保存”的顺序由应用层组织；版本规则由领域对象维护。即使以后把 CLI 换成 HTTP，修改模板仍然需要遵守同样的规则。

这也解释了为什么领域对象可以贯穿核心流程：`TemplateRevision` 从入口构造后直接参与领域操作，`Template` 从仓储恢复后直接被修改，再交回仓储。内部无需经历“接口字典 → 应用字典 → 领域字典 → 存储字典”的反复拆装。

## 第三步：仓储把领域对象写回文件

仓储接收更新后的 `Template`，在保存方法里完成 JSON 映射。它不需要接收 `TemplateResponse`，也不负责决定 Agent 应该看到哪些字段。

读取时还可以返回一个很小的结果：

```text
LoadedTemplate
├─ template: Template
└─ original_bytes: bytes
```

这里增加的不是另一份模板字段，而是同一次读取附带的文件事实。应用层把 `original_bytes` 原样交给保存方法，作为检查文件是否已经变化的依据；领域对象不需要知道这些字节。

这就是“领域模型加其他参数贯穿流程”的具体含义。业务内容、预期版本和文件字节承担不同职责，不应为了统一类型，全部塞进 `Template`。

保存成功后应用层才返回结果。如果领域规则拒绝修改，就不会调用保存。如果保存发现文件已经变化，就不会返回成功结果。

## 第四步：接口选择输出字段，返回给 Agent

应用层返回更新后的 `Template`。CLI 从中取出标识、当前版本和写作结构，构造自己的响应，再编码为 JSON。

文件中的原始字节不会进入响应。领域对象以后增加内部字段，也不应因为接口直接序列化整个对象，就自动成为公开协议。

同样，应用层不需要为了返回结果而导入 CLI 的响应模型。它完成业务操作，接口完成对外表达。

## 把这条路径放进一个可以运行的文件

下面是流程的缩小示例，使用 Python 3.10 标准库。只保留一个模板、一份文件和当前修订；完整策略工具的目录、历史修订、归档与恢复没有在这里展开，也不意味着实际工具应丢弃历史。

这里约定目录文件由程序维护，仓储直接映射；Agent JSON 是输入边界，类型在 CLI 检查一次。结构不能为空、版本接续和过期修改拒绝属于业务规则。

保存为 `template_flow.py`。为便于复制，代码放在一个文件中，注释标出了各部分职责；实际项目可以按对应模块拆开，数据流不因此改变。

```python
import argparse
from dataclasses import asdict, dataclass, replace
import json
import os
from pathlib import Path
import sys
from tempfile import NamedTemporaryFile


# 领域层：新内容与当前模板都使用业务模型。
@dataclass(frozen=True)
class TemplateRevision:
    revision: int
    structure: tuple[str, ...]

    def __post_init__(self):
        if self.revision < 1:
            raise ValueError("版本必须大于零")
        if not self.structure or any(not step.strip() for step in self.structure):
            raise ValueError("模板必须包含非空的写作步骤")


@dataclass(frozen=True)
class Template:
    id: str
    current: TemplateRevision

    def revise(self, proposed: TemplateRevision, *, expected_revision: int) -> "Template":
        if self.current.revision != expected_revision:
            raise ValueError("模板版本已变化，请重新读取")
        if proposed.revision != self.current.revision + 1:
            raise ValueError("新修订必须接续当前版本")
        return replace(self, current=proposed)


# 仓储结果：领域对象加上同一次读取的事实。
@dataclass(frozen=True)
class LoadedTemplate:
    template: Template
    original_bytes: bytes


class JsonTemplateRepository:
    def __init__(self, path: Path):
        self.path = path

    def load(self) -> LoadedTemplate:
        original_bytes = self.path.read_bytes()
        data = json.loads(original_bytes)
        current = data["current"]
        template = Template(
            id=data["id"],
            current=TemplateRevision(current["revision"], tuple(current["structure"])),
        )
        return LoadedTemplate(template, original_bytes)

    def save(self, template: Template, *, expected_bytes: bytes) -> None:
        temporary = None
        try:
            with NamedTemporaryFile(
                mode="w", encoding="utf-8", dir=self.path.parent, delete=False
            ) as stream:
                temporary = Path(stream.name)
                json.dump(asdict(template), stream, ensure_ascii=False, indent=2)
            if self.path.read_bytes() != expected_bytes:
                raise ValueError("文件已变化，请重新读取")
            os.replace(temporary, self.path)
        finally:
            if temporary is not None and temporary.exists():
                temporary.unlink()


# 应用层：组织读取、领域操作和保存。
def edit_template(
    repository: JsonTemplateRepository,
    template_id: str,
    proposed: TemplateRevision,
    *,
    expected_revision: int,
) -> Template:
    loaded = repository.load()
    if loaded.template.id != template_id:
        raise KeyError(template_id)
    updated = loaded.template.revise(proposed, expected_revision=expected_revision)
    repository.save(updated, expected_bytes=loaded.original_bytes)
    return updated


# 接口层：这两个模型服务于 Agent 协议。
@dataclass(frozen=True)
class EditTemplateInput:
    template_id: str
    expected_revision: int
    structure: list[str]


@dataclass(frozen=True)
class TemplateResponse:
    template_id: str
    revision: int
    structure: tuple[str, ...]


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("file", type=Path)
    args = parser.parse_args()
    data = json.load(sys.stdin)
    if not isinstance(data, dict):
        raise ValueError("输入必须是 JSON 对象")
    request = EditTemplateInput(**data)
    if not isinstance(request.template_id, str) or not request.template_id.strip():
        raise ValueError("template_id 必须是非空字符串")
    if type(request.expected_revision) is not int:
        raise ValueError("expected_revision 必须是整数")
    if not isinstance(request.structure, list) or not all(
        isinstance(step, str) for step in request.structure
    ):
        raise ValueError("structure 必须是字符串数组")

    proposed = TemplateRevision(
        revision=request.expected_revision + 1,
        structure=tuple(request.structure),
    )
    updated = edit_template(
        JsonTemplateRepository(args.file), request.template_id, proposed,
        expected_revision=request.expected_revision,
    )
    response = TemplateResponse(
        template_id=updated.id,
        revision=updated.current.revision,
        structure=updated.current.structure,
    )
    print(json.dumps(asdict(response), ensure_ascii=False))


if __name__ == "__main__":
    main()
```

准备 `template.json`：

```json
{
  "id": "question-answer",
  "current": {
    "revision": 1,
    "structure": ["提出问题", "解释机制"]
  }
}
```

运行一次编辑：

```bash
python3 template_flow.py template.json <<'JSON'
{"template_id": "question-answer", "expected_revision": 1, "structure": ["先给结论", "解释机制与边界"]}
JSON
```

标准输出是返回给 Agent 的结果：

```json
{"template_id": "question-answer", "revision": 2, "structure": ["先给结论", "解释机制与边界"]}
```

文件也已经更新。它保留的是仓储格式：

```json
{
  "id": "question-answer",
  "current": {
    "revision": 2,
    "structure": ["先给结论", "解释机制与边界"]
  }
}
```

两份 JSON 来自同一个更新后的领域对象，却服务于不同的契约。不是先保存接口响应，再把它当成仓储数据读回来。

再次运行相同命令，会因为当前已经是第 2 版而拒绝，文件保持原样。把结构改成字符串，则在 CLI 被拒绝；把结构改成空数组，类型合法，但构造领域修订时会因为没有写作步骤而拒绝。

示例直接传播异常，以便观察拒绝位置，没有另设错误响应协议。`dataclass` 的类型注解不执行运行时校验；缺失或额外输入字段由构造签名拒绝，其余协议检查在入口完成。

保存沿用[上一篇的做法]({{< relref "repository-results-without-model-copies.md" >}})：临时文件写完后替换，拒绝在比较时已变化的文件。比较和替换之间没有锁，不能据此声称多进程并发安全，也没有承诺掉电持久性。

## 再看策略选择：读取后不必经过保存

编辑展示了完整的读写路径。选择策略与模板时，输入只有要选的 ID、版本和理由，应用层从仓储取得现有目录，让领域规则检查归属、状态和适用模式，再返回 `Selection`。

```text
Agent：策略 ID、模板 ID、版本、理由
                 ↓
CLI：解析外部输入，传入明确参数
                 ↓
应用层：读取 Catalog，调用选择规则
                 ↓
领域层：取得 Strategy、Template 的指定修订，形成 Selection
                 ↓
应用层：返回 Selection
                 ↓
CLI：构造选择响应，编码为 JSON
                 ↓
Agent：依据返回的写作规则组织正文
```

这条只读路径没有仓储保存。选择输出中的规则快照和版本属于交接约定；若后续写作任务需要保存选择结果或正文，应由对应流程负责，不能据此认定策略工具已经存储了文章。

也不必为了让每一步都接收领域对象，让 CLI 提前查询仓储。Agent 只给出 ID 时，组织读取本来就是应用用例的一部分。

## 数据流清楚之后，再决定模型放在哪里

沿上面的两条路径，可以得到一份具体的分工：

| 数据或模型 | 负责的位置 | 内部怎样使用 |
|---|---|---|
| 编辑请求、选择请求的外部格式 | CLI 接口 | 解析后提取明确参数，按需构造领域对象 |
| `TemplateRevision`、`Template`、`Selection` | 领域层 | 承载业务数据，执行修订或选择规则 |
| 模板 ID、预期版本等参数 | 用例调用 | 与领域对象一起传递，无需统一包装 |
| `LoadedTemplate` | 仓储查询结果 | 组合领域对象与同一次读取的事实 |
| 文件 JSON | 仓储 | 在读写方法中映射，不流入业务处理 |
| 编辑响应、选择响应 | CLI 接口 | 根据用例结果构造并序列化 |

应用层可以直接返回领域对象，也可以在确有额外结果信息时组合一个命名结果。存储格式确实不同、需要独立演进时，可以增加存储模型。这里的依据是实际需要表达什么，不是数一数系统有几层。

还要区分数据流与代码依赖。领域对象可能由 CLI 构造、由仓储恢复，经过应用层后再交回仓储，但领域模块不因此导入 CLI 或 JSON 仓储。数据经过哪些操作，与模块依赖谁，是两个问题。

“领域模型贯穿业务”也不是要求所有查询都加载完整对象。列表查询可以使用投影；接口只需要少数字段时，也可以从查询结果直接组织响应。

回头看自己的代码，可以选一次真实操作，逐步标出输入、读取结果、领域操作、保存参数和对外响应。每一步都能说清当前拿到的是什么、交给下一步什么，分层的作用才会体现在实际调用中。

## 系列阅读：分层与边界

1. [让领域模型和仓储模型各自负责]({{< relref "domain-and-repository-boundaries.md" >}})：模型职责与边界。
2. [分层之后，为什么每一层还在重复校验？]({{< relref "validation-at-boundaries.md" >}})：输入约定与业务约束。
3. [让 Agent 少写防御性代码：每条校验都要有理由]({{< relref "validation-needs-a-reason.md" >}})：判断检查是否必要。
4. [仓储有自己的返回模型，不等于每层都要复制领域对象]({{< relref "repository-results-without-model-copies.md" >}})：仓储结果与存储表示。
5. 本篇：从 Agent 输入到 JSON 读写，再返回 Agent 的完整数据流。

参考：[Python dataclass 与 replace](https://docs.python.org/3/library/dataclasses.html)、[JSON 编码器支持的类型](https://docs.python.org/3/library/json.html#json.JSONEncoder)、[os.replace 的语义](https://docs.python.org/3/library/os.html#os.replace)。

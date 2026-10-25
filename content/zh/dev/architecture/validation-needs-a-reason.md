---
title: "让 Agent 少写防御性代码：每条校验都要有理由"
date: 2026-10-25
draft: false
categories: ["架构"]
tags: ["领域模型", "Repository", "分层架构", "Python", "AI 编程"]
description: "从策略工具的数组转换、反馈读取和查询筛选三个例子出发，判断哪些校验有实际用途，哪些只是重复工作，并给出约束 Agent 的编码规则。"
---

一个策略选择工具，本来只需要让 Agent 查看模板、选择版本、取得写作规则。写着写着，代码却变成了另一副样子：遍历字段名、判断字典形状、检查数组元素、猜测字段类型，再把已经检查过的数据交给下一层继续检查。

每一条判断单独看都有解释，放在一起却让人看不清工具到底做什么。

这发生在一次写作策略工具教程的审查中。删除一个通用转换助手后，代码甚至更长了：检查没有减少，只是搬进了另一个函数。问题并不是少用了一个框架，也不是变量命名不够规范，而是还没决定哪些错误需要处理，就先把能想到的检查写了出来。

本文适合已经了解 Python dataclass、正在让 Agent 编写小型工具或分层应用的读者。以下例子从那次审查中缩小而来，用来说明设计取舍，不是完整策略工具的实现。

前两篇分别讨论[模型与仓储的职责]({{< relref "domain-and-repository-boundaries.md" >}})和[入口检查与业务约束的分工]({{< relref "validation-at-boundaries.md" >}})。这篇往前追问一步：**在决定校验放在哪里之前，我们为什么需要这条校验？**

## 一个数组转换，怎么长成了通用校验器

假设模板包含 `structure` 字段。JSON 中它是数组，领域对象中希望用 tuple 表示：

```python
structure = tuple(data["structure"])
```

这行代码完成转换，但不保证输入正确：

```python
assert tuple("post") == ("p", "o", "s", "t")
```

如果 Agent 把数组写成字符串，转换不会报错，错误会以另一种形式进入模型。因此，在接收 Agent 请求的地方，检查它是不是字符串数组是有具体理由的。

但这不意味着每个处理 `structure` 的方法，都应该重新执行下面三项检查：

- 它是不是数组？
- 数组里的元素是不是字符串？
- 数组是不是非空？

前两项属于输入协议；第三项取决于业务：这个模板是否允许没有正文结构？如果模型已经保证结构不能为空，入口无需重复保证一次。

同样的字段，也可能来自程序自己写出的文件。是否检查，要先确定文件的使用方式。

| 数据来源 | 能依赖的前提 | 处理方式 |
|---|---|---|
| Agent 提交的编辑 JSON | 尚未确认类型和字段 | 入口解析，拒绝不符合约定的结构 |
| 已解析的内部请求对象 | 调用者遵守既定契约 | 直接使用，不重复判断类型 |
| 同一程序写出、没有独立编辑入口的文件 | 写入与读取遵循同一格式 | 可以直接映射，加载失败时暴露错误 |
| 允许人工编辑或导入的文件 | 内容可能不符合格式 | 把文件读取当作输入边界，定义需要的检查 |

“放在仓库里”不等于“可信”。如果维护者经常手改 JSON，它就是一个真实的编辑入口。反过来，受控文件也不是永远不会损坏；选择直接映射，意味着不承诺为每一种损坏提供定制诊断，更不保证发现所有能被转换的错误类型。

## 一个能直接运行的例子

缩小需求：模板名称不能为空，结构至少有一项，每一项不能是空白文字。外部请求只接受 `name` 和 `structure`，不自动转换数字或字符串数组。

保存下面代码为 `template_boundary.py`，使用 Python 3.10 或更新版本运行：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Template:
    name: str
    structure: tuple[str, ...]

    def __post_init__(self):
        if not self.name.strip():
            raise ValueError("模板名称不能为空")
        if not self.structure or any(not step.strip() for step in self.structure):
            raise ValueError("模板必须包含非空的写作步骤")


def decode_template(data: object) -> Template:
    if not isinstance(data, dict) or set(data) != {"name", "structure"}:
        raise ValueError("请求必须包含且只能包含 name 和 structure")
    if not isinstance(data["name"], str):
        raise ValueError("name 必须是字符串")
    steps = data["structure"]
    if not isinstance(steps, list) or not all(isinstance(step, str) for step in steps):
        raise ValueError("structure 必须是字符串数组")
    return Template(name=data["name"], structure=tuple(steps))


def writing_outline(template: Template) -> str:
    return "\n".join(f"{i}. {step}" for i, step in enumerate(template.structure, 1))


if __name__ == "__main__":
    template = decode_template({
        "name": "问题与解释",
        "structure": ["提出具体问题", "解释原因与适用条件"],
    })
    print(writing_outline(template))

    cases = [
        ({"name": "问题与解释", "structure": "post"}, "structure 必须是字符串数组"),
        ({"name": "问题与解释", "structure": [42]}, "structure 必须是字符串数组"),
        ({"name": "问题与解释", "structure": []}, "模板必须包含非空的写作步骤"),
        ({"name": "问题与解释", "structure": [" "]}, "模板必须包含非空的写作步骤"),
        ({"name": " ", "structure": ["解释"]}, "模板名称不能为空"),
    ]
    for payload, expected in cases:
        try:
            decode_template(payload)
        except ValueError as error:
            assert str(error) == expected
        else:
            raise AssertionError(f"错误输入被接受：{payload!r}")
    print("正常输入和 5 个失败场景均符合约定")
```

运行 `python3 template_boundary.py`，输出：

```text
1. 提出具体问题
2. 解释原因与适用条件
正常输入和 5 个失败场景均符合约定
```

这里没有通用字段转换器，也没有单独包装一个只转发调用的服务。入口把数组变成 tuple，模型保护内容规则，`writing_outline()` 直接使用模型。

这个例子仍然有校验，因为输入确实来自外部。内部的 `writing_outline()` 没有再问一次“你真的是 Template 吗”。它依赖调用契约，而不是依赖类型注解自动执行检查。若将来把它作为允许任意调用者使用的公共库接口，需要重新评估这个边界。

## 有些“校验复杂度”，其实是重复读文件

反馈查询中还出现过这样的结构，下面只展示问题所在：

```python
# 问题片段：list 和 get 都打开同一个文件。
def list(self):
    entries = []
    for path in self.directory.glob("*.json"):
        raw = json.loads(path.read_text())
        if raw.get("record_type") == "feedback":
            entries.append(self.get(path.stem))
    return entries
```

`list()` 已经读到记录，`get()` 又读取一次。第二次读取可能还会重复解析和检查，看起来是在复用读取逻辑，实际上是在为同一份结果做两遍工作。若文件期间被修改，两次读取还可能拿到不同内容；它并没有因此成为可靠的一致性检查。

先确定目录约定，再改代码。如果目录只存反馈记录，列表可以枚举文件名后直接调用 `get()`，让每个文件只读一次。如果目录允许混合记录，就在首次解析后检查类型，并用那份数据完成恢复。

这里不需要立即新增通用文件读取框架。真正需要决定的是：**列表返回字典，还是已经恢复的 Feedback？** 如果应用层接下来需要按反馈对象的字段筛选，恢复对象应由仓储负责，应用层不必再删除元数据、遍历 dataclass 字段并猜测类型。

## 查询条件不必对每条记录重审一次

另一个例子是按模板版本筛选反馈。业务约定：“只指定 revision 不够，还必须指定 template_id”，因为不同模板都有自己的第 1 版。

这条规则有意义，但它属于一次查询，而不是某一条反馈。如果外层查询函数已经检查了，循环内每次 `feedback.matches(...)` 再检查就是重复工作。

下面是独立可运行的缩小示例：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Feedback:
    template_id: str
    revision: int


def select_feedback(entries, template_id=None, revision=None):
    if revision is not None:
        if template_id is None:
            raise ValueError("指定版本时必须指定模板")
        if revision < 1:
            raise ValueError("版本必须大于零")
    return [entry for entry in entries
            if (template_id is None or entry.template_id == template_id)
            and (revision is None or entry.revision == revision)]


entries = [Feedback("question", 1), Feedback("question", 2), Feedback("story", 1)]
assert select_feedback(entries, "question", 1) == [entries[0]]
assert select_feedback(entries, "question") == entries[:2]
assert select_feedback(entries) == entries
try:
    select_feedback([], revision=1)
except ValueError as error:
    assert str(error) == "指定版本时必须指定模板"
else:
    raise AssertionError("空列表也必须拒绝不完整的查询条件")
print("筛选结果正确；空列表同样检查查询条件")
```

这里假定筛选参数已由 CLI 或其他入口解析为正确类型，函数负责查询规则。把检查放在循环外，还解决了另一个问题：即使列表为空，错误查询也会被拒绝。如果检查只写在每条记录的 `matches()` 中，空列表就绕过了它。

如果 `matches()` 只在这里调用，直接写筛选表达式就够了。若其他用例确实需要它，可以保留方法，但让查询入口拥有条件校验，不必每条记录重复判断。

## 测试通过，不代表这些检查都值得存在

过度防御还有一种自我强化的过程：先写大量校验，再为每种校验写测试，最后因为“删了测试会失败”，认定这些校验都是必要需求。

测试能证明实现遵守了某项约定，不能替我们决定这项约定值不值得维护。

这里要分清两种变化：

- **需求没有变。** 删除检查导致原本应拒绝的输入被接受，就是回归。不能为了绿色结果删掉测试。
- **需求已经明确改变。** 例如受控缓存文件不再承诺完整格式诊断，就应同步调整相关测试，说明哪些行为不再承诺，并继续验证真正保留的规则。

“它只是内部文件”不能由 Agent 为了减少代码自行宣布。是否允许人工编辑、错误是否需要友好报告，是使用方式和接口契约的一部分。确认这些条件之后，才谈得上删掉相应校验。

## 给 Agent 的要求，要约束判断过程

只写“不要做任何校验”太粗糙。它可能让 Agent 连过期版本、模板归属或确认后换稿的检查也删掉。反过来，“所有输入都严格校验”也容易变成层层防御。

可以在编码规则中加入下面这段：

```text
默认不新增防御性校验。新增之前，简要说明：
1. 数据来源是什么，哪个真实场景会使它无效？
2. 不检查会造成什么后果？
3. 在哪个入口检查一次，失败时如何处理？

内部代码使用已经建立的输入契约，不重复解析原始字典。
领域模型保留业务不变量；受控文件优先直接映射。
不要为少量检查引入通用框架、反射或单次使用的包装函数。
测试验证已确认的需求；契约改变时说明变化并同步调整测试。
```

这里的“说明”可以是编码前几句话，不需要给每个判断创建一份设计文档，也不需要逐条请求批准。只有业务规则或行为要求不明确时，才需要询问。

审查代码时，可以先挑一个入口，沿数据走到输出。对每条检查写一句：“它阻止了什么具体错误？”如果答案只是“以防万一”，继续追问输入由谁构造、前面已经保证了什么。对每个助手则问：“它减少了哪一处真实重复，还是只让我要多跳转一次？”

## 系列阅读：分层与边界

1. [让领域模型和仓储模型各自负责]({{< relref "domain-and-repository-boundaries.md" >}})：模型之间如何划分责任。
2. [分层之后，为什么每一层还在重复校验？]({{< relref "validation-at-boundaries.md" >}})：类型检查与业务约束如何分工。
3. 本篇：每条校验是否值得存在，以及如何约束 Agent 的实现方式。

参考：[Python 类型注解的运行时边界](https://docs.python.org/3/library/typing.html)、[dataclass 的初始化后处理](https://docs.python.org/3/library/dataclasses.html#post-init-processing)。

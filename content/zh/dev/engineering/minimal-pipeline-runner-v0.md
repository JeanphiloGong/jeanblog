---
title: "从普通函数调用推导一个最小 Pipeline Runner"
subtitle: "当最终结果不够用，执行过程本身就要变成可观察的 RunRecord"
date: 2026-05-29T00:00:00+08:00
description: "从普通函数调用开始，逐步推导一个最小顺序 Pipeline Runner：工程接入场景、run/step observation、共享 context、mode 解析、handler 表和 PipelineRunner.trigger。"
summary: "本文保留 nano-pipeline-runner v0 的完整推导链条：先证明普通函数调用什么时候足够，再通过工程接入、运行观察、失败记录、重复执行策略和共享 context 的压力，构建一个最小顺序 Pipeline Runner。"
tags: ["Pipeline", "Python", "工程实践", "可观察性", "教程"]
categories: ["工程实践"]
keywords: ["pipeline runner", "python pipeline", "workflow runner", "step observation", "run record", "from scratch", "可观察执行"]
readingTime: "约 18 分钟"
---

> 源码仓库：<https://github.com/JeanphiloGong/nano-pipeline-runner>
>
> 本文对应仓库里的 v0 checkpoint：最小顺序 Pipeline Runner。配套代码和测试都在仓库中。

## 这篇文章解决什么问题

本文不是先告诉你“pipeline runner 应该长什么样”，然后把最终代码贴出来。

更好的学习方式是从一个普通函数调用开始，问它到底缺什么。只有当当前版本真的无法满足下一个需求时，才加下一块结构。这样最后出现的 `mode`、`context`、`StepRecord`、`RunRecord` 和 `PipelineRunner.trigger()` 都不是凭空设计出来的，而是被一个个具体压力推出来的。

最终我们会得到这个最小链路：

```text
trigger(mode, context)
-> resolve workflow names
-> run handlers in order
-> record each executed step
-> return one run record
```

你可以把这篇文章理解成一个 v0：它只解决顺序 pipeline 的核心问题，不解决 DAG、并行、持久化、resume、cancel。后面那些能力都可以继续演进，但不应该抢在最小模型之前出现。

## 目标读者

适合读这篇文章的人：

- 会写 Python 函数、字典、异常处理、循环和简单 `dataclass`
- 听过 pipeline / workflow runner，但不清楚它和普通函数调用差别在哪里
- 想把“多个步骤按顺序执行”变成可观察、可测试的结构

读完以后，你应该能解释这句话：

```text
如果只需要最终结果，用普通函数；
如果执行过程本身也要被观察，才需要 pipeline runner。
```

## 真实场景

假设调用方想处理一个文件。对外看起来，它只是一个动作：

```text
process_file(file_id)
```

但这个动作内部其实有两个步骤：

```text
prepare -> enrich
```

`prepare` 创建 `enrich` 需要的状态。如果一切成功，调用方可以只关心最终结果。但只要系统需要展示进度、记录日志、返回失败原因，调用方就会开始关心更具体的信息：

- 哪个步骤失败了
- 失败之前哪些步骤已经运行
- 整次 run 最后是什么状态

普通函数调用可以完成工作。pipeline runner 有价值的地方是：**它让工作过程也变成可以返回、检查和测试的数据。**

这篇文章的边界也在这里：如果你只想得到 `enrich_result`，普通函数就够了；如果你还想得到“这次 run 是怎么走完或失败的”，才需要继续往下构建 runner。

## 工程场景

把这个例子放回工程系统里，pipeline runner 通常不是替代业务函数，而是放在业务函数外面，统一回答“这次处理过程发生了什么”。

一个常见接入点是 API 或后台任务入口：

```text
HTTP request / job message
-> build context: {"file_id": "..."}
-> runner.trigger("standard", context)
-> return or persist RunRecord
```

这时工程侧真正需要的不是 `prepare` 和 `enrich` 这两个函数本身，而是一个稳定的执行边界：

- API 层可以把 `RunRecord.status` 映射成响应状态
- 管理台可以展示 `RunRecord.steps`，告诉用户卡在哪个步骤
- 日志和告警可以记录 failed step，而不是只记录一条散乱异常
- 测试可以直接断言 step 顺序、失败截断和共享 context
- 后续接数据库时，可以把同一个 `RunRecord` 模型扩展成持久化对象

所以 v0 的工程价值不是“让两个函数按顺序执行”。这件事普通代码已经能做。它的价值是把执行入口、步骤列表、失败策略和观察结果收束到同一个地方，让多个调用点不会各自实现一套不一致的流程控制。

也因此，这个 v0 适合放在这些位置：

- API command handler：请求进来后触发一次同步处理
- worker job handler：消费一个任务后运行固定步骤
- 后台管理操作：用户点一次按钮，系统返回可解释的 run 结果
- 测试夹具：用最小 runner 验证步骤编排和失败语义

它暂时不适合承担这些职责：

- 长时间运行任务的调度
- 跨机器并发执行
- 数据库级 dedupe
- resume / cancel
- DAG 依赖调度

这些是 v1 以后可以接上的能力。v0 先把工程入口的最小契约定清楚：给我一个 `mode` 和一次 run 的 `context`，我按配置执行，并返回完整的 run observation。

## 问题压缩

真实生产里的 pipeline 系统可能会包含：

- 队列
- 重试
- 数据库存储
- resume
- cancel
- fan-out
- dedupe
- API 状态投影

这一篇故意不做这些。

我们只实现一个内存里的同步顺序 runner：

- 一个调用方调用 `trigger(mode, context)`
- 一个 `mode` 解析成固定顺序的 workflow name 列表
- 每个 workflow name 解析成一个 Python 函数
- 所有 handler 共享同一个可变 `context`
- 每个执行过的 workflow 都产生一个 step record
- 某个 workflow 失败后，run 标记为 failed，并停止后续 workflow

这个模型很小，但保留了 pipeline 的核心不变量：**声明顺序控制执行顺序，执行过的步骤在返回结果里可见。**

本篇刻意推迟这些内容：run id、持久化、graph plan、dependency wave、parallel branch、resume、cancel、API projection。

## 核心模型和不变量

最终实现会操作这些概念：

- `mode`：调用方请求的流程形态
- `workflow name`：一个内部步骤名
- `handler`：workflow name 对应的 Python 函数
- `context`：一次 run 内共享的可变字典
- `StepRecord`：一个已执行 workflow 的可见结果
- `RunRecord`：一次 trigger 的可见结果
- `PipelineRunner`：持有 mode map、handler map 和执行策略的对象

每个 checkpoint 都要保护这些规则：

- unknown mode 必须在任何 handler 运行前失败
- known mode 必须严格按配置顺序运行 workflow
- 同一次 run 里的 handler 必须收到同一个 `context`
- 每个执行过的 workflow 必须留下一个 `StepRecord`
- 一个 workflow 失败后，后续 workflow 不再运行

## 外部契约

调用方能依赖的行为：

- 调用方传入 `mode` 和 `context`
- `mode` 映射到一个固定顺序的 workflow name 列表
- workflow handler 按顺序运行
- runner 返回 `RunRecord`
- unknown mode 在工作开始前抛 `ValueError`

本文的假设：

- workflow handler 是普通 Python callable
- `context` 是调用方传进来的可变 `dict`
- v0 runner 同步执行，全部状态都在内存里

硬约束：

- 只使用 Python 标准库
- 不加 `run_id`
- 不复制 `context`
- 不把 mode 或 workflow name 强制转成字符串
- 不加入 v1 的 graph execution 行为

## 教学例子

我们从一个业务流程开始：

```text
standard -> prepare -> enrich
```

这里用 `file_id`，不用文件流。原因是 v0 要讲的是执行控制和观察结果，不是文件 IO。真实文件流可以放进 run state，但它不会改变本文要解决的核心压力。

我们先把这个流程写成普通业务函数：

```text
file_id -> prepared_file -> enrich_result
```

然后先承认：如果调用方只要最终结果，普通函数就够了。只有当多个流程都需要相同的 step observation 和 failure policy 时，pipeline runner 才出现。

## 从零实现

### Step 1：从普通函数调用开始

引入 runner 之前，先写最简单能工作的代码。

这一步是一个完整 checkpoint，直接在当前脚本里写：

```python
def prepare(file_id):
    return {"file_id": file_id, "prepared": True}


def enrich(prepared_file):
    return {"saw_prepared": prepared_file["prepared"]}


file_id = "file-1"
prepared_file = prepare(file_id)
enrich_result = enrich(prepared_file)
```

这段代码在 happy path 下完全正确。关键不是“两个函数运行了”，而是它们怎样运行：

- `file_id` 是原始业务输入
- `prepare(file_id)` 创建中间值 `prepared_file`
- `enrich(prepared_file)` 依赖这个中间值
- 顺序只体现在调用方那两行代码里
- `prepared_file` 和 `enrich_result` 是两个分散的值，不是一个可观察的 run

检查基线：

```python
assert prepared_file == {"file_id": "file-1", "prepared": True}
assert enrich_result == {"saw_prepared": True}
```

如果需求到这里结束，就应该停下来。此时没有 pipeline 压力。代码简单、清楚、正确。

下一个需求必须是普通函数调用不能返回的东西。如果 `enrich` 失败，这段脚本可以抛异常，但它不会返回这样的 run observation：

```python
{
    "status": "failed",
    "steps": [
        {"name": "prepare", "status": "success"},
        {"name": "enrich", "status": "failed", "error": "enrich failed"},
    ],
}
```

这才是继续往下走的具体理由。缺陷不是“函数不扩展”，而是：**执行过程没有被表示成数据**。没有 run status，没有 executed step list，也没有 failed step record。

Checkpoint：我们现在有一个可工作的直接调用基线。它证明了数据依赖通过显式中间值传递。它仍然不能返回 run/step observation。

### Step 2：先包一个普通函数

在构建 pipeline 之前，先用最直接的方式消除重复调用：

```python
prepared_file = prepare(file_id)
enrich_result = enrich(prepared_file)
```

最简单的修复是普通函数。

在上一步基础上增加：

在上一步基础上增加：

```python
def run_standard(file_id):
    prepared_file = prepare(file_id)
    return enrich(prepared_file)
```

检查这个 wrapper：

```python
assert run_standard("file-1") == {"saw_prepared": True}
```

这仍然不是 pipeline，而且这正是重点。如果调用方只需要最终 enriched result，`run_standard(file_id)` 就是合适的代码量。不要因为两个函数按顺序发生，就直接引入 runner。

Checkpoint：现在基线有了一个直接业务函数。它隐藏了 handoff，但仍然只返回最终结果。它不能说哪些步骤运行了、哪个步骤失败了、整个 run 的状态是什么。

### Step 3：让一个流程返回 observation

上一步足够支持：

```python
result = run_standard("file-1")
```

但它不够支持调用方检查执行本身：

```python
record = run_standard("file-1")
assert record["status"] == "success"
assert [step["name"] for step in record["steps"]] == ["prepare", "enrich"]
```

新需求不是“做一个 pipeline”。它更窄：只为这一个流程返回一个小 record，里面包含 run status 和每个成功 step。

把 `run_standard` 替换成 observed version：

把 `run_standard` 替换成 observed version：

```python
def run_standard(file_id):
    record = {
        "name": "standard",
        "status": "running",
        "steps": [],
    }

    prepared_file = prepare(file_id)
    record["steps"].append(
        {
            "name": "prepare",
            "status": "success",
            "result": prepared_file,
        }
    )

    enrich_result = enrich(prepared_file)
    record["steps"].append(
        {
            "name": "enrich",
            "status": "success",
            "result": enrich_result,
        }
    )

    record["status"] = "success"
    return record
```

检查成功 observation：

```python
record = run_standard("file-1")

assert record["status"] == "success"
assert [step["name"] for step in record["steps"]] == ["prepare", "enrich"]
assert [step["status"] for step in record["steps"]] == ["success", "success"]
assert record["steps"][1]["result"] == {"saw_prepared": True}
```

Checkpoint：一个流程现在可以返回成功 observation。它仍然有一个明显缺口：如果 `enrich` 抛异常，函数会在 append `enrich` failed step 之前退出。

### Step 4：把异常转换成 failed step

上一步只在函数返回后追加 success step：

```python
enrich_result = enrich(prepared_file)
record["steps"].append(
    {
        "name": "enrich",
        "status": "success",
        "result": enrich_result,
    }
)
```

如果 `enrich` 抛异常，调用方得到的是异常，而不是它想要的 observation。新需求是在 step 边界捕获异常，把当前 step 记录成 failed，把 run 标记为 failed，然后停止。

把 `run_standard` 替换成 failure-aware version：

把 `run_standard` 替换成 failure-aware version：

```python
def run_standard(file_id):
    record = {
        "name": "standard",
        "status": "running",
        "steps": [],
    }

    try:
        prepared_file = prepare(file_id)
    except Exception as exc:
        record["steps"].append(
            {
                "name": "prepare",
                "status": "failed",
                "error": str(exc),
            }
        )
        record["status"] = "failed"
        return record

    record["steps"].append(
        {
            "name": "prepare",
            "status": "success",
            "result": prepared_file,
        }
    )

    try:
        enrich_result = enrich(prepared_file)
    except Exception as exc:
        record["steps"].append(
            {
                "name": "enrich",
                "status": "failed",
                "error": str(exc),
            }
        )
        record["status"] = "failed"
        return record

    record["steps"].append(
        {
            "name": "enrich",
            "status": "success",
            "result": enrich_result,
        }
    )
    record["status"] = "success"
    return record
```

为了检查失败路径，临时让 `enrich` 失败：

```python
def enrich(_prepared_file):
    raise RuntimeError("enrich failed")


failed = run_standard("file-1")

assert failed["status"] == "failed"
assert [step["name"] for step in failed["steps"]] == ["prepare", "enrich"]
assert [step["status"] for step in failed["steps"]] == ["success", "failed"]
assert failed["steps"][1]["error"] == "enrich failed"
```

Checkpoint：一个流程现在可以返回成功或失败 observation。但这仍然不强迫我们写 pipeline。对于一个流程，直接 `run_standard` 仍然可以接受。

### Step 5：加入第二个 observed flow

现在引入真正让直接函数变难维护的压力：第二个流程也需要同样的 observation 和 failure policy。

第二个流程更小：

```text
core -> prepare
```

你仍然可以直接写：

```python
def run_core(file_id):
    record = {
        "name": "core",
        "status": "running",
        "steps": [],
    }

    try:
        prepared_file = prepare(file_id)
    except Exception as exc:
        record["steps"].append(
            {
                "name": "prepare",
                "status": "failed",
                "error": str(exc),
            }
        )
        record["status"] = "failed"
        return record

    record["steps"].append(
        {
            "name": "prepare",
            "status": "success",
            "result": prepared_file,
        }
    )
    record["status"] = "success"
    return record
```

检查第二个流程：

```python
record = run_core("file-1")

assert record["name"] == "core"
assert record["status"] == "success"
assert [step["name"] for step in record["steps"]] == ["prepare"]
```

这才是第一个真实 pipeline 压力。`run_standard` 和 `run_core` 不是偶然重复业务工作，而是在重复执行策略：

- 创建 run record
- 按顺序执行每个 step
- handler 返回后追加 success step
- handler 抛异常后追加 failed step
- 第一个 failed step 后停止
- 标记整个 run success 或 failed

如果这个策略发生变化，每个 observed flow 都要同样修改。新需求是：**一个共享执行策略可以运行不同 step list。**

Checkpoint：两个直接 observed flow 都能工作，但 observation 和 failure policy 被复制了。正是这个复制，而不是“有两个函数调用”，才证明 pipeline runner 有价值。

### Step 6：给 step 一个共享 run context

如果要用一个 loop 运行一组 step，每个 step 就需要同样的调用形状。现在函数签名不一致：

```python
prepared_file = prepare(file_id)
enrich_result = enrich(prepared_file)
```

`prepare` 接收 `file_id`。`enrich` 接收 `prepared_file`。generic loop 如果要处理这种差异，就必须为每一对 step 写特殊 handoff 逻辑，这会把业务链条重新塞回 runner。

新需求是一个共享 run state。本文里这个 state 就是一个字典，叫 `context`。

替换函数签名和调用方式：

替换函数签名和调用方式：

```python
def prepare(context):
    context["prepared"] = True
    return {"prepared": True}


def enrich(context):
    return {"saw_prepared": context["prepared"]}


context = {"file_id": "file-1"}
prepare_result = prepare(context)
enrich_result = enrich(context)
```

现在两个 step 都收到同一种输入形状。`prepare` 写入 `context`，`enrich` 从同一个字典读取。

检查共享状态：

```python
assert prepare_result == {"prepared": True}
assert enrich_result == {"saw_prepared": True}
assert context == {"file_id": "file-1", "prepared": True}
```

Checkpoint：handler 现在可以共享同一个 run context。runner 不需要知道 `prepare` 为 `enrich` 创建了哪个中间值；它只需要把同一个 `context` 传给每个 step。

### Step 7：命名 observed flow plan

上一步让每个 handler 有了相同输入形状。现在两个 observed flow 的差异只剩有序 step list：

```python
mode_workflows = {
    "core": ["prepare"],
    "standard": ["prepare", "enrich"],
}
```

到这里，外部流程选择器可以叫 `mode` 了。这个名字是被需求推出来的：现在有多个 observed flow，调用方必须选择触发哪一个。

新需求是一张表：把调用方语言 `mode` 转换成内部 workflow name 列表。

增加 mode table 和 resolver：

```python
mode_workflows = {
    "core": ["prepare"],
    "standard": ["prepare", "enrich"],
}


def resolve_workflows(mode):
    workflows = mode_workflows.get(mode)
    if workflows is None:
        raise ValueError(f"unknown mode: {mode}")
    return workflows
```

检查 flow plan：

```python
assert resolve_workflows("standard") == ["prepare", "enrich"]
assert resolve_workflows("core") == ["prepare"]

try:
    resolve_workflows("extract")
except ValueError as exc:
    assert str(exc) == "unknown mode: extract"
else:
    raise AssertionError("unknown mode should fail")
```

Checkpoint：现在基线能把调用方 mode 解析成声明过的 workflow list。它还不能执行这些 workflow name。

### Step 8：抽出一个 observed executor

上一步能产生 workflow name：

```python
resolve_workflows("standard")  # ["prepare", "enrich"]
```

名字本身不可执行。你可以写分支：

```python
if workflow_name == "prepare":
    prepare(context)
```

但这样 execution loop 里会为每个 workflow 加一个特殊分支。新需求是一张表：把 workflow name 转成 handler 函数。

增加 handler table、handler resolver 和共享 executor：

```python
handlers = {
    "prepare": prepare,
    "enrich": enrich,
}


def resolve_handler(workflow_name):
    handler = handlers.get(workflow_name)
    if handler is None:
        raise ValueError(f"missing handler: {workflow_name}")
    return handler


def run_observed(mode, context):
    workflow_names = resolve_workflows(mode)
    record = {
        "mode": mode,
        "workflows": list(workflow_names),
        "context": context,
        "status": "running",
        "steps": [],
    }

    for workflow_name in workflow_names:
        handler = resolve_handler(workflow_name)
        try:
            result = handler(record["context"])
        except Exception as exc:
            record["steps"].append(
                {
                    "name": workflow_name,
                    "status": "failed",
                    "error": str(exc),
                }
            )
            record["status"] = "failed"
            return record
        record["steps"].append(
            {
                "name": workflow_name,
                "status": "success",
                "result": result,
            }
        )

    record["status"] = "success"
    return record
```

检查一个策略是否同时服务两个流程：

```python
standard = run_observed("standard", {"file_id": "file-1"})
core = run_observed("core", {"file_id": "file-1"})

assert standard["status"] == "success"
assert standard["workflows"] == ["prepare", "enrich"]
assert [step["name"] for step in standard["steps"]] == ["prepare", "enrich"]

assert core["status"] == "success"
assert core["workflows"] == ["prepare"]
assert [step["name"] for step in core["steps"]] == ["prepare"]
```

也要检查 unknown mode 边界：

```python
events = []


def prepare_for_unknown(_context):
    events.append("prepare")


handlers = {"prepare": prepare_for_unknown}

try:
    run_observed("extract", {"file_id": "file-1"})
except ValueError as exc:
    assert str(exc) == "unknown mode: extract"
else:
    raise AssertionError("unknown mode should fail")

assert events == []

handlers = {
    "prepare": prepare,
    "enrich": enrich,
}
```

Checkpoint：现在基线是：

```text
mode -> workflow names -> handler calls -> observed run
```

这就是最小 pipeline core：配置顺序 + 共享执行策略。它仍然返回普通字典，所以结果形状很容易写错。

### Step 9：把 observation dictionary 替换成 record

上一步手写字典：

```python
record["steps"].append(
    {
        "name": workflow_name,
        "status": "success",
        "result": result,
    }
)
```

这能工作，但 step 的 shape 已经变成契约。新需求是直接命名这个 shape：

- 每个执行过的 workflow 对应一个 `StepRecord`
- 每次 trigger 对应一个 `RunRecord`

增加 record：

```python
from dataclasses import dataclass, field
from typing import Any


@dataclass
class StepRecord:
    name: str
    status: str
    result: Any = None
    error: str | None = None


@dataclass
class RunRecord:
    mode: str
    workflows: list[str]
    context: dict[str, Any]
    status: str = "queued"
    steps: list[StepRecord] = field(default_factory=list)
```

然后把 `run_observed` 替换成返回 record 的版本：

```python
def run_observed(mode, context):
    workflow_names = resolve_workflows(mode)
    record = RunRecord(
        mode=mode,
        workflows=list(workflow_names),
        context=context,
        status="running",
    )

    for workflow_name in workflow_names:
        handler = resolve_handler(workflow_name)
        try:
            result = handler(record.context)
        except Exception as exc:
            record.steps.append(
                StepRecord(
                    name=workflow_name,
                    status="failed",
                    error=str(exc),
                )
            )
            record.status = "failed"
            return record
        record.steps.append(
            StepRecord(
                name=workflow_name,
                status="success",
                result=result,
            )
        )

    record.status = "success"
    return record
```

检查 record：

```python
context = {"file_id": "file-1"}
record = run_observed("standard", context)

assert record.status == "success"
assert record.mode == "standard"
assert record.workflows == ["prepare", "enrich"]
assert record.context is context
assert context == {"file_id": "file-1", "prepared": True}
assert [step.name for step in record.steps] == ["prepare", "enrich"]
assert [step.status for step in record.steps] == ["success", "success"]
assert record.steps[1].result == {"saw_prepared": True}
```

注意这里没有 `run_id`，没有持久化，也没有队列。当前压力只要求内存里的 observation 和稳定结果 shape。

Checkpoint：基线现在有了最终结果模型。它仍然是 script-shaped code：全局配置 + helper functions。

### Step 10：组装 Runner API

上一步能工作，但还不是可复用 API。每个调用点如果都复制这段形状：

```python
workflow_names = resolve_workflows("standard")
record = RunRecord(...)

for workflow_name in workflow_names:
    handler = resolve_handler(workflow_name)
    ...
```

就会产生执行策略漂移。比如一个调用点忘记在 step 边界捕获异常，同一个 mode 就会出现两种失败行为。

v0 的最终需求是：一个对象持有稳定配置，一个方法执行一次 run：

```text
PipelineRunner(...).trigger(mode, context)
```

最后把前面已经推导出的结构组装成一个完整 checkpoint，目标是 `src/nano_pipeline/models.py` 和 `src/nano_pipeline/runner.py`。

最终 checkpoint 拆成两个小文件。先是状态模型：

```python
"""Small state model for the v0 pipeline runner."""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any


@dataclass
class StepRecord:
    """One executed workflow step."""

    name: str
    status: str
    result: Any = None
    error: str | None = None


@dataclass
class RunRecord:
    """One pipeline run created from a mode."""

    mode: str
    workflows: list[str]
    context: dict[str, Any]
    status: str = "queued"
    steps: list[StepRecord] = field(default_factory=list)
```

然后是 runner：

```python
"""v0 minimal sequential pipeline runner.

The runner owns one decision: a mode resolves to a list of workflow names, and
those workflow handlers run in that exact order.
"""

from __future__ import annotations

from collections.abc import Callable, Mapping
from typing import Any

from .models import RunRecord, StepRecord

WorkflowHandler = Callable[[dict[str, Any]], Any]


class PipelineRunner:
    """Run mode-defined workflows in order and record step outcomes."""

    def __init__(
        self,
        *,
        mode_workflows: Mapping[str, list[str]],
        handlers: Mapping[str, WorkflowHandler],
    ) -> None:
        self._mode_workflows = dict(mode_workflows)
        self._handlers = dict(handlers)

    def trigger(self, mode: str, context: dict[str, Any]) -> RunRecord:
        """Create and execute one run for a known mode."""
        workflow_names = self._resolve_workflows(mode)
        record = RunRecord(
            mode=mode,
            workflows=list(workflow_names),
            context=context,
            status="running",
        )

        for workflow_name in workflow_names:
            handler = self._resolve_handler(workflow_name)
            try:
                result = handler(record.context)
            except Exception as exc:
                record.steps.append(
                    StepRecord(
                        name=workflow_name,
                        status="failed",
                        error=str(exc),
                    )
                )
                record.status = "failed"
                return record
            record.steps.append(
                StepRecord(
                    name=workflow_name,
                    status="success",
                    result=result,
                )
            )

        record.status = "success"
        return record

    def _resolve_workflows(self, mode: str) -> list[str]:
        workflows = self._mode_workflows.get(mode)
        if workflows is None:
            raise ValueError(f"unknown mode: {mode}")
        return workflows

    def _resolve_handler(self, workflow_name: str) -> WorkflowHandler:
        handler = self._handlers.get(workflow_name)
        if handler is None:
            raise ValueError(f"missing handler: {workflow_name}")
        return handler
```

这个 checkpoint 没有增加新的 pipeline 行为。它只是包装前面已经构建出来的行为：

- mode resolution 变成 `_resolve_workflows`
- handler resolution 变成 `_resolve_handler`
- handler 返回后仍然追加 success record
- handler 边界异常仍然追加 failed record
- `context` 仍然是调用方传入的字典，不复制

构造函数复制两张配置表，让 runner 持有稳定配置。这和 `context` 不同：配置是 runner setup；`context` 是一次 run 的共享状态，必须继续被 handler 和调用方共享。

检查 happy path：

```python
events = []


def prepare(context):
    events.append(("prepare", dict(context)))
    context["prepared"] = True
    return {"prepared": True}


def enrich(context):
    events.append(("enrich", dict(context)))
    return {"saw_prepared": context["prepared"]}


runner = PipelineRunner(
    mode_workflows={"standard": ["prepare", "enrich"]},
    handlers={"prepare": prepare, "enrich": enrich},
)

context = {"file_id": "file-1"}
record = runner.trigger("standard", context)

assert record.status == "success"
assert record.mode == "standard"
assert record.workflows == ["prepare", "enrich"]
assert record.context is context
assert context == {"file_id": "file-1", "prepared": True}
assert [step.name for step in record.steps] == ["prepare", "enrich"]
assert [step.status for step in record.steps] == ["success", "success"]
assert record.steps[1].result == {"saw_prepared": True}
assert events == [
    ("prepare", {"file_id": "file-1"}),
    ("enrich", {"file_id": "file-1", "prepared": True}),
]
```

保留两个边界检查。

unknown mode 必须在任何 handler 运行前失败：

```python
events = []


def prepare_for_unknown(_context):
    events.append("prepare")


unknown_runner = PipelineRunner(
    mode_workflows={"standard": ["prepare"]},
    handlers={"prepare": prepare_for_unknown},
)

try:
    unknown_runner.trigger("extract", {"file_id": "file-1"})
except ValueError as exc:
    assert str(exc) == "unknown mode: extract"
else:
    raise AssertionError("unknown mode should fail")

assert events == []
```

workflow 失败要记录当前 failed step，并停止后续 step：

```python
def failing_enrich(_context):
    raise RuntimeError("enrich failed")


failing_runner = PipelineRunner(
    mode_workflows={"standard": ["prepare", "enrich"]},
    handlers={"prepare": prepare, "enrich": failing_enrich},
)
failed = failing_runner.trigger("standard", {"file_id": "file-1"})

assert failed.status == "failed"
assert [step.status for step in failed.steps] == ["success", "failed"]
assert failed.steps[1].error == "enrich failed"
```

Checkpoint：这就是完整 v0 runner。它故意很小：没有 run id，没有 context copy，没有 string coercion，没有 queue，没有 storage，也没有 resume logic。

## 辅助结构的边界

`run_standard` 最早出现，是为了证明更简单的替代方案。如果调用方只需要最终结果，直接 wrapper 就够了，不需要 pipeline。

`resolve_workflows` 在两个 observed flow 开始复制同一套执行策略之后才出现。它接收 `mode`，返回 workflow name list，或者在任何 handler 运行前抛 `ValueError`。它不修改 `context`。

`resolve_handler` 在 workflow name 需要变成可执行函数时出现。它接收一个 workflow name，返回 callable handler，或者抛 `ValueError`。它不运行 handler。

`StepRecord` 在手写 step dictionary 变成契约时出现。它保存一个已执行 workflow 的可见结果：name、status、optional result、optional error。

`RunRecord` 在 observed run dictionary 变成契约时出现。它保存 mode、declared workflows、共享 context、run status 和 step records。

`PipelineRunner.trigger` 在 script-shaped execution chain 可能被复制到多个调用点时出现。它接收 `mode` 和调用方传入的 `context`，通过 handler 修改这个 context，并返回一个 `RunRecord`。

## 常见误区

- 只说“函数不扩展”，却没有指出真正缺陷：直接调用不能返回 run/step observation。
- 在 hand-built observation dictionary 变成真实契约之前，就提前引入 record。
- 没有 run identity 需求时，提前加入 `run_id`。
- 复制 `context`，导致本文要教的共享状态规则被破坏。
- 没有压力例子就把 mode 或 workflow name 强制转成字符串。
- 在整个 loop 外层捕获异常，而不是在当前 workflow 边界捕获。
- workflow 失败后继续执行后续 workflow，却没有明确改变 v0 failure policy。
- 把 graph execution、persistence 或 resume behavior 塞进 v0。

## 验证方式

最终项目可以这样验证：

```bash
git clone https://github.com/JeanphiloGong/nano-pipeline-runner.git
cd nano-pipeline-runner
PYTHONPATH=src python3 -m unittest discover -s tests
```

测试至少应该证明：

- happy path：`standard -> prepare -> enrich` 按顺序运行
- shared context：`record.context is context`
- step records：两个成功 workflow 都可见
- failure path：`enrich` 失败会产生 failed step，并把 run 标记为 failed
- invalid mode：unknown mode 在任何 handler 运行前失败

对文章本身，可以检查这些偏移：

- Step 1 在命名缺陷之前，先解释原始直接调用代码
- 每一步都从上一个可见 checkpoint 继续
- 每个代码改动要么是局部修改，要么是最终组装 checkpoint
- 没有 helper 在压力出现前提前出现
- 最终代码是 Step 10 checkpoint，不是脱离推导链条的代码堆放
- 代码片段没有引入 `run_id`、context copy、string coercion 或 v1 graph behavior

## 下一步

下一篇可以从这个 v0 runner 出发，引入一个新的压力：

```text
"chain" 和 "parallel" 需要不同 dependency shape，
但 v0 runner 只能表达固定顺序的 workflow list。
```

这个压力会自然推导到 graph plan、dependency validation、ready wave 和 failed dependency blocking。那些是 v1 关注的内容，所以不放进这篇 v0。

源码仓库在这里：<https://github.com/JeanphiloGong/nano-pipeline-runner>。

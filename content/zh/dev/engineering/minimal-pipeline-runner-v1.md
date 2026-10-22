---
title: "从顺序 Pipeline Runner 演进到 Graph Runner"
subtitle: "当 workflow list 不能表达分支依赖时，用 dependency graph 和 ready wave 推动执行"
date: 2026-06-03T00:00:00+08:00
description: "在最小顺序 Pipeline Runner 的基础上，继续推导 graph pipeline runner：为什么 workflow list 不够用，为什么要用 dependency map、ready wave、GraphNodeRecord 和失败分支跳过。"
summary: "本文是最小 Pipeline Runner 系列的 v1 后续章节：从 v0 的顺序 workflow list 出发，推导 dependency graph、ready wave、graph validation、node record 和 failed dependency blocking。"
tags: ["Pipeline", "Python", "工程实践", "Graph", "教程"]
categories: ["工程实践"]
keywords: ["graph pipeline runner", "python pipeline", "workflow graph", "ready wave", "dependency graph", "DAG runner"]
readingTime: "约 12 分钟"
---

> 源码仓库：<https://github.com/JeanphiloGong/nano-pipeline-runner>
>
> 本文对应仓库里的 v1 checkpoint：Graph Pipeline Runner。建议先读上一篇：
> [从普通函数调用推导一个最小 Pipeline Runner](/zh/dev/engineering/minimal-pipeline-runner-v0/)。

## 这篇文章解决什么问题

上一篇 v0 做了一件事：把普通函数调用演进成一个可观察的顺序 pipeline runner。

它的核心链路是：

```text
trigger(mode, context)
-> resolve workflow names
-> run handlers in order
-> record each executed step
-> return one run record
```

这个 v0 很有用，但它有一个明确边界：`mode` 只能解析成一个有序 list。

也就是说，v0 很擅长表达：

```text
prepare -> enrich -> extract
```

但当流程变成这样时：

```text
prepare
  -> enrich
  -> extract
```

list 就开始说谎了。它必须把 `enrich` 和 `extract` 排出一个先后顺序，可事实上它们只是共享同一个上游 `prepare`。

本文解决的就是这个问题：**如何从顺序 pipeline runner 演进到 dependency graph runner。**

## 目标读者

适合读这篇文章的人：

- 已经理解 v0 的 `mode`、`context`、handler、run record 和 step record
- 想知道 pipeline 什么时候应该从 list 升级成 graph
- 想理解 ready wave、dependency map、node status 和 failed dependency blocking

读完以后，你应该能解释这句话：

```text
graph runner 的重点不是“并发”，而是“谁在当前状态下已经 ready”。
```

## v0 留下的真实压力

先回到 v0 的数据结构：

```python
mode_workflows = {
    "chain": ["prepare", "enrich", "extract"],
}
```

这对顺序链路很好。runner 只需要：

```python
for workflow_name in workflow_names:
    handler = resolve_handler(workflow_name)
    handler(context)
```

问题出现在第二种模式。假设我们想表达：

```text
prepare
  -> enrich
  -> extract
```

如果仍然用 list，只能写成：

```python
mode_workflows = {
    "parallel": ["prepare", "enrich", "extract"],
}
```

或者：

```python
mode_workflows = {
    "parallel": ["prepare", "extract", "enrich"],
}
```

但这两种写法都不是事实。它们都在暗示一个下游步骤排在另一个下游步骤前面。

真正想表达的是：

```text
prepare 成功后，enrich 和 extract 都 ready。
```

这就是 v1 的压力来源。不是“list 不高级”，而是：

```text
list 可以表达一个顺序，但不能表达依赖关系。
```

## 为什么用 dependency map

v1 把 workflow list 替换成 dependency map：

```python
graph_plans = {
    "chain": {
        "prepare": (),
        "enrich": ("prepare",),
        "extract": ("enrich",),
    },
    "parallel": {
        "prepare": (),
        "enrich": ("prepare",),
        "extract": ("prepare",),
    },
}
```

读法是：

```text
这个 node 依赖哪些上游 node？
```

所以：

```python
"enrich": ("prepare",)
```

表示：

```text
enrich 只有在 prepare success 后才能运行。
```

为什么不用这种出边表？

```python
"prepare": ("enrich", "extract")
```

这种写法适合画图，因为它回答的是：

```text
prepare 后面连着谁？
```

但 runner 执行时最常问的问题是：

```text
这个 queued node 现在能不能运行？
```

dependency map 正好直接回答这个问题：

```python
def find_ready_nodes(record):
    ready = []
    for name, node in record.nodes.items():
        if node.status != "queued":
            continue
        if all(
            record.nodes[dependency].status == "success"
            for dependency in node.dependencies
        ):
            ready.append(name)
    return ready
```

对一个 node 来说，只要所有依赖都是 `success`，它就是 ready。

这也是为什么 `prepare` 一开始就是 ready：

```python
prepare.dependencies == []
all([]) == True
```

没有前置条件，就意味着前置条件已经满足。

## ready wave 是什么

有了 dependency map，runner 不再问：

```text
list 里的下一个 workflow 是谁？
```

而是反复问：

```text
当前有哪些 queued node 已经 ready？
```

对 parallel 图来说，初始状态是：

```text
prepare queued
enrich queued
extract queued
```

第一轮：

```python
find_ready_nodes(record) == ["prepare"]
```

执行完 `prepare` 后：

```text
prepare success
enrich queued
extract queued
```

第二轮：

```python
find_ready_nodes(record) == ["enrich", "extract"]
```

这两个列表就是 wave：

```python
record.waves == [["prepare"], ["enrich", "extract"]]
```

注意，本文的 v1 仍然是同步 runner。`["enrich", "extract"]` 不代表真的开了两个线程。它表示：

```text
这两个节点在同一时刻都满足依赖，都可以进入当前执行波次。
```

真实并发可以以后加，但不是 v1 的核心。

## GraphRunRecord 为什么不再用 steps list

v0 的 run record 可以这样存：

```python
record.steps = [
    StepRecord(name="prepare", status="success"),
    StepRecord(name="enrich", status="success"),
]
```

这适合顺序执行，因为下一步永远是 list 里的下一个 item。

但 graph runner 需要频繁回答：

```text
enrich 依赖的 prepare 现在是什么状态？
extract 依赖的 prepare 现在是什么状态？
```

所以 v1 用 node name 建索引：

```python
@dataclass
class GraphNodeRecord:
    name: str
    dependencies: list[str]
    status: str = "queued"
    result: Any = None
    error: str | None = None


@dataclass
class GraphRunRecord:
    mode: str
    context: dict[str, Any]
    nodes: dict[str, GraphNodeRecord]
    waves: list[list[str]] = field(default_factory=list)
    status: str = "queued"
```

查询某个步骤状态时，直接按名字取：

```python
record.nodes["enrich"].status
record.nodes["extract"].result
```

所以 v1 的 observation 重点变成：

```text
run.status：整次 run 的状态
run.waves：执行波次
run.nodes[name]：某个 node 的状态、结果和错误
```

## 为什么要先 validate graph

dependency map 能表达正确图，也能表达错误图：

```python
bad_missing = {
    "prepare": ("missing",),
}

bad_cycle = {
    "prepare": ("extract",),
    "extract": ("prepare",),
}
```

这两种错误都必须在 handler 运行前失败。

原因很简单：如果图结构本身无效，runner 不应该先执行一半 workflow，再告诉你配置错了。

所以 v1 先做 graph validation：

```python
def validate_graph(dependencies):
    for name, depends_on in dependencies.items():
        for dependency in depends_on:
            if dependency not in dependencies:
                raise ValueError(f"unknown dependency for {name}: {dependency}")

    visiting = set()
    visited = set()

    def visit(name):
        if name in visited:
            return
        if name in visiting:
            raise ValueError(f"cycle detected at {name}")
        visiting.add(name)
        for dependency in dependencies[name]:
            visit(dependency)
        visiting.remove(name)
        visited.add(name)

    for name in dependencies:
        visit(name)
```

这个函数的边界应该保持干净：

```text
只检查 graph。
不运行 handler。
不创建 run record。
不修改 graph。
```

这就是为什么它叫 `validate_graph`，不是 `prepare_graph`。

## trigger loop 怎么推进图

v0 可以用一个 `for`：

```python
for workflow_name in workflow_names:
    run(workflow_name)
```

v1 不知道有几轮 wave。下一轮是谁，取决于上一轮执行后的 node status。

所以主循环是：

```python
while True:
    ready_nodes = find_ready_nodes(record)
    if not ready_nodes:
        break

    record.waves.append(ready_nodes)

    for node_name in ready_nodes:
        node = record.nodes[node_name]
        node.status = "running"
        handler = resolve_handler(node_name)
        node.result = handler(record.context)
        node.status = "success"
```

可以把它理解成：

```text
while：一波一波推进 graph
for：执行当前 wave 里的每个 node
```

parallel 的过程就是：

```text
while 第 1 轮:
    wave = ["prepare"]
    for 跑 prepare

while 第 2 轮:
    wave = ["enrich", "extract"]
    for 跑 enrich
    for 跑 extract

while 第 3 轮:
    wave = []
    break
```

这就是 graph runner 和 sequential runner 的核心差异。

## 失败语义为什么要变

v0 的失败规则是：

```text
某一步失败，后面全部不跑。
```

这对 list 是合理的。因为 list 只有一条线，后面的步骤都被认为依赖前面的执行顺序。

但 graph 不是这样。

看 chain：

```text
prepare -> enrich -> extract
```

如果 `enrich` 失败，`extract` 必须跳过：

```text
prepare success
enrich failed
extract skipped
```

但 parallel 是：

```text
prepare
  -> enrich
  -> extract
```

如果 `enrich` 失败，`extract` 不应该被跳过，因为它不依赖 `enrich`：

```text
prepare success
enrich failed
extract success
```

所以 v1 的失败规则变成：

```text
失败只向依赖它的下游传播，不阻断独立分支。
```

对应 helper 是：

```python
def mark_dependents_skipped(record, failed_node):
    changed = True
    while changed:
        changed = False
        for node in record.nodes.values():
            if node.status != "queued":
                continue
            if any(
                record.nodes[dependency].status in {"failed", "skipped"}
                for dependency in node.dependencies
            ):
                node.status = "skipped"
                node.error = f"dependency failed: {failed_node}"
                changed = True
```

它不是从失败节点往后走，而是扫描所有还没运行的 queued node：

```text
如果某个 queued node 的任意依赖已经 failed/skipped，
这个 queued node 也不能再运行，标记为 skipped。
```

`changed` 的作用是处理多层下游：

```text
prepare -> enrich -> extract -> publish
```

如果 `enrich` failed，第一轮会把 `extract` 标记为 skipped。下一轮再发现 `publish` 依赖已经 skipped 的 `extract`，于是继续传播。

## 最终 runner 长什么样

把前面的结构收束起来，v1 的主入口大概是这样：

```python
class GraphPipelineRunner:
    def trigger(self, mode: str, context: dict[str, Any]) -> GraphRunRecord:
        dependencies = self._resolve_graph(mode)
        self._validate_graph(dependencies)
        record = self._create_graph_record(mode, dependencies, context)

        while True:
            ready_nodes = self._find_ready_nodes(record)
            if not ready_nodes:
                break
            record.waves.append(ready_nodes)

            for node_name in ready_nodes:
                node = record.nodes[node_name]
                node.status = "running"
                handler = self._resolve_handler(node_name)
                try:
                    node.result = handler(record.context)
                except Exception as exc:
                    node.status = "failed"
                    node.error = str(exc)
                    self._mark_dependents_skipped(
                        record,
                        failed_node=node_name,
                    )
                    continue
                node.status = "success"

        if all(node.status == "success" for node in record.nodes.values()):
            record.status = "success"
        elif any(node.status == "failed" for node in record.nodes.values()):
            record.status = "failed"
        else:
            record.status = "blocked"
        return record
```

这个版本相比 v0 多了几件事：

- `mode -> workflow list` 变成 `mode -> dependency graph`
- `StepRecord list` 变成 `GraphNodeRecord by name`
- 顺序 `for` 变成 `while ready wave`
- 执行前增加 missing dependency / cycle validation
- 失败后只跳过依赖失败节点的下游

## v1 仍然没有解决什么

这一版仍然不是生产 workflow engine。

它故意不解决：

- 持久化
- run id
- resume
- cancel
- 真实并发 worker
- API 状态投影
- 去重和幂等

这些能力都可以接在 v1 后面，但不应该混进 v1 的核心教学里。

因为 v1 的教学目标只有一个：

```text
把顺序 pipeline 的执行形状，从 list 升级成 dependency graph。
```

## 常见误区

### 误区 1：graph runner 等于并发 runner

不等于。

v1 里的 wave 表示“这些节点同时 ready”，不是“这些节点已经被多线程并发执行”。

同步 runner 也可以记录 ready wave。真实并发只是后续实现策略。

### 误区 2：出边表一定比依赖表更自然

如果你在画图，出边表很自然：

```python
"prepare": ("enrich", "extract")
```

但如果你在写 runner，依赖表更直接：

```python
"enrich": ("prepare",)
"extract": ("prepare",)
```

因为 runner 最常问的是：

```text
这个 node 的依赖都成功了吗？
```

### 误区 3：v0 的失败规则可以直接搬到 v1

不能。

v0 的“失败后停止后续步骤”适合一条线。v1 的 graph 里，一个节点失败只应该影响依赖它的下游，不应该影响独立分支。

### 误区 4：只记录 waves 就够了

不够。

`waves` 只能告诉你执行波次，不能快速回答：

```text
extract 当前是什么状态？
enrich 的错误是什么？
prepare 的 result 是什么？
```

所以 v1 需要：

```python
record.nodes["extract"].status
record.nodes["enrich"].error
```

## 小结

v0 和 v1 的分工很清楚：

```text
v0：把顺序执行变成可观察 run。
v1：把顺序执行形状升级成 dependency graph。
```

v1 的核心不是“更复杂”，而是解决 v0 的一个具体缺口：

```text
workflow list 不能表达多个下游节点在同一个上游成功后同时 ready。
```

所以我们引入：

- dependency map：描述每个 node 依赖谁
- graph validation：坏图在执行前失败
- GraphNodeRecord：按 node name 查询状态
- ready wave：动态找当前能运行的一批 node
- failed dependency blocking：失败只向下游依赖传播

如果你已经实现了 v0，那么 v1 是一个自然的下一步。下一篇可以继续往工程系统走：给 graph run 加 `run_id` 和持久化，让 resume / cancel 有一个稳定的目标。

源码仓库在这里：<https://github.com/JeanphiloGong/nano-pipeline-runner>。

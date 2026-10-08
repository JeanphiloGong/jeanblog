---
title: "Redis 任务队列系列（1）：用 Python 和 List 实现最小生产消费模型"
date: 2026-10-08
draft: false
categories: ["分布式系统"]
tags: ["Redis", "Python", "任务队列", "异步", "生产者消费者"]
keywords: ["Python Redis 任务队列", "Redis List 队列", "LPUSH BRPOP", "redis-py", "生产者消费者"]
description: "使用 redis-py 和 Redis List 写出一个可以运行的最小任务队列，理解 LPUSH、BRPOP、阻塞消费、FIFO 与任务丢失边界。"
summary: "这是 Redis 任务队列系列的第一篇：从一个具体的计算任务开始，用 Python 实现生产者和消费者，先掌握 Redis List 的入队与阻塞出队，再看为什么它还不能直接作为可靠的生产任务队列。"
---

> 这是“用 Python 实现 Redis 任务队列”系列的第 1 篇。本文先使用 Redis List 建立最小模型；下一篇会把“任务已经取出但 worker 中断”的处理中状态补上。

## 先处理一个真实任务

假设 API 收到用户提交的五个项目，需要在后台逐个计算匹配度。接口不希望等待计算完成，于是把每个项目转换为一条任务：

```text
API 进程：把任务写入 Redis
worker：等待并取出任务
worker：执行计算并记录结果
前端：通过业务接口查询状态
```

这里先只关心“任务如何等待和领取”。Redis 不负责业务结果的最终结构，计算结果仍然可以写入 PostgreSQL。

本篇选择 Redis List，是因为它只需要两个核心动作：生产者使用 `LPUSH` 入队，消费者使用 `BRPOP` 阻塞等待并取出。这个版本适合建立心智模型，不包含重试、超时接管、优先级和结果后端。

系列导航：

1. **本文**：Redis List + Python 的最小任务队列
2. **下一篇**：处理中队列、确认和 worker 中断后的恢复
3. **后续**：Redis Streams 消费者组、Pending 消息与 `XAUTOCLAIM`
4. **选型篇**：什么时候用 Redis，什么时候用 RabbitMQ、Celery 或 PostgreSQL

## 先准备 Redis 和 Python 客户端

本地没有 Redis 时，可以用 Docker 启动一个临时实例：

```bash
docker run --rm --name redis-task-demo -p 6379:6379 redis:7
```

新建虚拟环境并安装 Python 客户端：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install redis
```

`redis-py` 是 Python 客户端，真正保存任务的是 Redis 服务。默认示例连接 `redis://localhost:6379/0`，也可以通过 `REDIS_URL` 改成其他地址。

先确认服务可连接：

```bash
redis-cli ping
# PONG
```

## 生产者只负责把任务放进去

先写一个生产者。每条消息使用 JSON 表示，至少包含业务任务编号和输入内容。任务编号由业务方生成，后续查询状态或实现幂等时都需要它。

```python
# redis_task_queue.py
import argparse
import json
import os
from typing import Any

import redis


QUEUE_KEY = "demo:tasks"


def connect() -> redis.Redis:
    return redis.Redis.from_url(
        os.getenv("REDIS_URL", "redis://localhost:6379/0"),
        decode_responses=True,
    )


def publish(r: redis.Redis, task_id: int, value: str) -> None:
    message: dict[str, Any] = {
        "task_id": task_id,
        "value": value,
    }
    r.lpush(QUEUE_KEY, json.dumps(message, ensure_ascii=False))
    print(f"enqueued task_id={task_id}")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--value", default="demo")
    parser.add_argument("--count", type=int, default=5)
    args = parser.parse_args()

    if args.count <= 0:
        raise SystemExit("--count must be positive")

    r = connect()
    for task_id in range(1, args.count + 1):
        publish(r, task_id, f"{args.value}-{task_id}")


if __name__ == "__main__":
    main()
```

运行：

```bash
python redis_task_queue.py --count 5
```

`LPUSH` 把新任务放到列表左侧。消费者从右侧 `BRPOP`，因此最早加入的任务最先被取出：

```text
第一次 LPUSH：      [task-1]
第二次 LPUSH：      [task-2, task-1]
BRPOP 取出：        task-1
```

如果只是想观察队列长度，可以执行：

```bash
redis-cli LLEN demo:tasks
redis-cli LRANGE demo:tasks 0 -1
```

生产者在这里的成功含义是“Redis 接受了消息”。它不表示计算已经开始，也不表示业务结果已经写入。

## 消费者在没有任务时阻塞等待

消费者使用 `BRPOP`。`B` 表示 blocking：队列为空时，Redis 不会立即返回空结果，而是在指定秒数内等待新消息。这样 worker 不需要每秒轮询，也不会因为短暂没有任务而退出。

```python
def consume_once(r: redis.Redis, timeout: int = 5) -> bool:
    item = r.brpop(QUEUE_KEY, timeout=timeout)
    if item is None:
        return False

    _, raw_message = item
    message = json.loads(raw_message)
    print(f"received task_id={message['task_id']} value={message['value']}")
    return True
```

把它接成一个持续运行的 worker：

```python
import time


def run_worker(r: redis.Redis) -> None:
    while True:
        item = r.brpop(QUEUE_KEY, timeout=5)
        if item is None:
            print("no task; waiting")
            continue

        _, raw_message = item
        message = json.loads(raw_message)
        try:
            print(f"start task_id={message['task_id']}")
            time.sleep(1)  # 用真实的业务计算替换
            print(f"succeeded task_id={message['task_id']}")
        except Exception as exc:
            print(f"failed task_id={message['task_id']} error={exc}")
```

将生产者和消费者拆成两个终端运行：

```bash
# 终端 1
python redis_task_queue.py --count 5

# 终端 2
python -c 'from redis_task_queue import connect, run_worker; run_worker(connect())'
```

如果消费者启动得更晚，任务仍会留在 Redis List 中。消费者启动后，`BRPOP` 会依次取出它们。启动两个相同的 worker，就能让两个消费者共同处理同一个队列；但每个任务只会被一个 `BRPOP` 调用取出。

## 为什么这个版本还不可靠

`BRPOP` 的关键语义是：**任务返回给消费者时，已经从待处理 List 删除。**这使得同一条消息不会一直停留在待处理列表中，但也带来一个直接的故障窗口：

```text
1. worker 从 Redis BRPOP 取出 task-7
2. Redis 已经删除 task-7
3. worker 还没完成计算就被杀死
4. task-7 不在待处理队列，也没有完成记录
```

这叫“取出后丢失”问题。它和任务是否异步无关，而是确认时机没有被表达出来。

不能只在 `BRPOP` 后写一句“处理失败就重新入队”来解决所有情况。worker 可能在捕获异常之前就崩溃，也可能计算已经产生外部副作用，却在写完成状态前断电。下一篇会把任务先移动到处理中 List，成功后显式确认，并讨论处理中任务的超时恢复。

### 阻塞消费和可靠消费是两件事

`BRPOP` 解决的是等待方式：没有消息时阻塞，避免固定间隔轮询。它没有解决以下问题：

| 问题 | 本篇 List + `BRPOP` 是否解决 |
| --- | --- |
| 空队列时等待 | 是 |
| 多个 worker 分摊任务 | 是 |
| 任务顺序（同一 List） | 是，按 FIFO 取出 |
| 处理成功后确认 | 否 |
| worker 中断后重新认领 | 否 |
| 业务结果幂等 | 否 |
| 失败重试和死信 | 否 |

因此，这段代码适合演示和某些“丢失也可以接受”的临时任务。涉及扣费、通知、数据写入或用户明确等待结果时，还需要补充可靠投递和业务幂等。

## 任务状态应该放在哪里

Redis List 只保存等待中的消息。如果前端需要查询状态，通常还要单独保存任务状态：

```text
task:{task_id} = {
  status: "pending | running | succeeded | failed",
  result: ...,
  error: ...
}
```

生产者可以在 `LPUSH` 后写 `pending`，消费者取出后写 `running`，结束后写 `succeeded` 或 `failed`。但如果这两次 Redis 操作之间进程中断，状态和队列仍可能不一致。可靠方案需要明确状态变更顺序、幂等键和恢复路径；不能因为有一个 `status` 字段，就自动拥有事务。

如果业务结果本来就保存在 PostgreSQL，Redis 可以只作为消息代理，任务状态和结果仍由业务数据库管理。选择 Redis 不等于必须把所有业务数据都放进 Redis。

## 用多少 worker 才合适

假设每个任务平均处理 `2` 秒：

```text
单 worker 理论吞吐 = 60 / 2 = 30 个任务/分钟
4 个 worker 理论上限 = 4 × 30 = 120 个任务/分钟
```

这是理想上限。实际吞吐还受 Redis 网络、数据库连接池、模型服务并发和任务耗时长尾影响。如果下游模型只允许两个并发请求，启动十个 worker 只会让更多任务同时等待或触发限流。

可以先观察三个指标：

- `LLEN demo:tasks`：等待积压长度；
- 单任务从入队到开始执行的等待时间；
- 处理失败、重新入队和最终失败的数量。

队列长度长期增长说明输入速率高于消费能力。增加 worker 只是一个选择，也可以限制生产者、降低单次提交数量或优化任务耗时。

## 本篇代码的边界

本篇完成了一个可运行的最小生产消费链：生产者使用 `LPUSH`，消费者使用 `BRPOP`，空队列时阻塞，多个 worker 可以分摊同一个 List。

它没有承诺“恰好执行一次”。`BRPOP` 之后的进程故障会丢失任务；即使补上重试，外部副作用也可能重复。因此，下一篇将把“等待中”和“处理中”拆开，加入确认、超时和重新认领，再比较它与 Redis Streams 消费者组的差异。

## 参考与延伸阅读

- [redis-py 文档](https://redis.readthedocs.io/en/stable/)
- [Redis LPUSH](https://redis.io/docs/latest/commands/lpush/)
- [Redis BRPOP](https://redis.io/docs/latest/commands/brpop/)
- [Redis BLMOVE](https://redis.io/docs/latest/commands/blmove/)
- [Python argparse](https://docs.python.org/3/library/argparse.html)

---
title: "Pydantic 的 field_validator：字段什么时候校验，返回值去了哪里？"
date: 2026-10-22T17:00:00+08:00
draft: false
categories: ["Python"]
tags: ["Python", "Pydantic", "FastAPI"]
description: "用考试成绩校验理解 Pydantic v2 的 field_validator：自动调用、返回值、ValueError 和校验顺序。"
---

在请求模型里看到 `@field_validator("score")`，很容易以为它只是给方法加了一个标记：这个方法由谁调用？返回的值又去了哪里？

它的作用是告诉 Pydantic：**校验 `score` 字段时，自动执行下面的方法，并把返回值作为这个字段的最终值。** 本文适合已经会写 `BaseModel`、但还不熟悉自定义校验的读者，示例使用 Pydantic v2。

## 类型正确，还不够

假设接口接收一次考试成绩：

```json
{
  "score": 120
}
```

`int` 可以说明成绩应当是整数，但 `120` 也是整数。对于满分 100 分的考试，还需要检查成绩是否在 0～100 之间。

这些规则可以放进字段校验器，在请求进入业务处理前统一完成。

## 一个可以直接运行的例子

把下面代码保存为 `example.py`，通过 `uv run --with 'pydantic>=2,<3' python example.py` 执行：

```python
from pydantic import BaseModel, ValidationError, field_validator


class ExamResult(BaseModel):
    score: int

    @field_validator("score")
    @classmethod
    def validate_score(cls, value: int) -> int:
        if not 0 <= value <= 100:
            raise ValueError("成绩必须在 0～100 之间")
        return value


result = ExamResult(score=85)
assert result.score == 85
print(result.score)

try:
    ExamResult(score=120)
except ValidationError as exc:
    error = exc.errors()[0]
    assert error["loc"] == ("score",)
    print(error["msg"])
else:
    raise AssertionError("应该拒绝超过满分的成绩")
```

输出：

```text
85
Value error, 成绩必须在 0～100 之间
```

本例是为了观察校验器的执行过程。实际只有范围限制时，直接写 `score: int = Field(ge=0, le=100)` 更简单，记得从 `pydantic` 导入 `Field`。遇到内置约束无法直接表达的规则时，再编写自定义校验器。

## 这段代码什么时候执行？

调用 `ExamResult(...)` 时，Pydantic 先按 `int` 处理字段，再调用 `validate_score()`。这是 `field_validator` 默认的 `mode="after"` 行为。

因此，这个方法拿到的 `value` 已经是通过基础校验的整数。例如默认非严格模式下，输入 `score="85"` 会先被转换成整数 `85`，再进行范围检查。

方法返回的值成为 `result.score`。调用方不需要再手动执行 `validate_score()`。

这里要区分两个装饰器的职责：

- `@field_validator("score")`：向 Pydantic 注册这个字段的校验方法。
- `@classmethod`：让方法接收类 `cls`，而不是实例 `self`；本例不需要使用 `cls`。

如果类型校验已经失败，例如传入 `score="abc"`，默认的 after 校验器不会继续处理这个字段。

## 为什么抛出 ValueError，外面却捕获 ValidationError？

在校验器内部，`ValueError` 表示“这个值不符合要求”。Pydantic 会收集它，并生成带有字段位置的 `ValidationError`，所以调用方能知道是哪个字段出错。

如果这个模型作为 FastAPI 的请求体，正常的请求校验失败默认返回 HTTP 422；项目自定义异常处理后，响应格式和状态码可能不同。Pydantic 本身只负责数据校验，不负责 HTTP 响应。

## 使用时记住三个边界

**校验成功后要返回值。** 即使没有修改输入，也应该 `return value`。遗漏返回值会让方法返回 `None`，不能指望 Pydantic 自动保留原值。

**after 与 before 不同。** `mode="before"` 在基础类型校验之前执行，拿到的可能是任意原始输入，不能直接假设是整数。这里没有提前处理原始输入的需求，默认 after 就够了。

**模型校验不等于持续监视对象。** 正常构造或调用 `model_validate()` 会执行校验；默认情况下，之后直接执行 `result.score = 120` 不会重新校验。需要赋值时也校验，可以显式配置 `validate_assignment=True`。

可以试着把输入分别改成 `0`、`100`、`-1` 和 `"abc"`，观察哪些值能通过，以及错误来自基础类型还是自己的校验方法。

## 参考

- [Pydantic：Field validators](https://docs.pydantic.dev/latest/concepts/validators/#field-validators)
- [Pydantic：Validation errors](https://docs.pydantic.dev/latest/errors/errors/)

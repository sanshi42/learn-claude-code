# s03: TodoWrite (待办写入)

`s01 > s02 > [ s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"没有计划的 agent 走哪算哪"* -- 先列步骤再动手, 完成率翻倍。
>
> **Harness 层**: 规划 -- 让模型不偏航, 但不替它画航线。

## 问题

多步任务中, 模型会丢失进度 -- 重复做过的事、跳步、跑偏。对话越长越严重: 工具结果不断填满上下文, 系统提示的影响力逐渐被稀释。一个 10 步重构可能做完 1-3 步就开始即兴发挥, 因为 4-10 步已经被挤出注意力了。

## 解决方案

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------	------------+
                          |
              if rounds_since_todo >= 3:
                append <reminder> after tool_result blocks
```

## 工作原理

1. TodoManager 存储带状态的项目。同一时间只允许一个 `in_progress`。

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

2. `todo` 工具和其他工具一样加入 dispatch map。

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

3. nag reminder: 模型连续 3 轮以上不调用 `todo` 时注入提醒。计数按模型响应轮次计算, 不是按一轮里调用了多少个非 `todo` 工具计算。

```python
results = []
used_todo = False
for block in response.content:
    if block.type == "tool_use":
        output = TOOL_HANDLERS[block.name](**block.input)
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": str(output),
        })
        if block.name == "todo":
            used_todo = True

rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.append({
        "type": "text",
        "text": "<reminder>Update your todos.</reminder>",
    })
```

注意: reminder 被追加在本轮 `results` 末尾, 也就是所有 `tool_result` 后面。不要把 text 插到工具结果前面, 也不要在下一次 API 调用前修改上一条工具结果消息的开头; Claude API 要求 `tool_result` blocks 必须在该 user 消息 content 数组中排第一。

"同时只能有一个 in_progress" 强制顺序聚焦。nag reminder 制造问责压力 -- 你不更新计划, 系统就追着你问。

## 相对 s02 的变更

| 组件           | 之前 (s02)       | 之后 (s03)                     |
|----------------|------------------|--------------------------------|
| Tools          | 4                | 5 (+todo)                      |
| 规划           | 无               | 带状态的 TodoManager           |
| Nag 注入       | 无               | 3 轮后注入 `<reminder>`        |
| Agent loop     | 简单分发         | + rounds_since_todo 计数器     |

## 试一试

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`

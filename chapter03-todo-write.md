## 第三章：TodoWrite - 如何让LLM规划任务？

### 1. 问题引入

在多步骤任务中，LLM会失去对进度的追踪。它会重复工作、跳过步骤，或者偏离目标。长时间对话会让问题更严重——系统提示被工具结果淹没。一个10步的重构任务，可能完成1-3步后LLM就开始即兴发挥，因为它忘记了4-10步。

**核心矛盾**：LLM需要完成复杂的多步骤任务，但它天生不擅长追踪长期进度。

### 2. 解决方案

#### 2.1 核心设计

```
+----------+      +-------+      +---------+
|   User   | ---> |  LLM  | ---> | Tools   |
|  prompt  |      |       |      | + todo  |
+----------+      +---+---+      +----+----+
                          ^             |
                          |  tool_result|
                          +-------------+
                                |
                    +-----------+-----------+
                    | TodoManager state     |
                    | [ ] task A            |
                    | [>] task B <- doing   |
                    | [x] task C            |
                    +-----------------------+
                                |
                    if rounds_since_todo >= 3:
                      inject <reminder>
```

**TodoManager**：存储任务状态，同一时间只能有一个`in_progress`任务。

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

**强制单一焦点**：同一时间只允许一个任务处于`in_progress`状态。

**Nag Reminder**：如果LLM连续3轮没有调用`todo`工具，自动注入提醒。

```python
if rounds_since_todo >= 3:
    results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

#### 2.2 示例数据

**Todo工具Schema**：

```python
{"name": "todo", "description": "Update task list. Track progress on multi-step tasks.",
 "input_schema": {"type": "object", "properties": {"items": {"type": "array", 
   "items": {"type": "object", "properties": {"id": {"type": "string"}, 
   "text": {"type": "string"}, 
   "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}}, 
   "required": ["id", "text", "status"]}}}, "required": ["items"]}}
```

**任务状态流转**：

```
[ ] #1: 添加类型提示
    |
    v (开始执行)
[>] #1: 添加类型提示  <- in_progress
    |
    v (完成)
[x] #1: 添加类型提示  <- completed
[ ] #2: 添加docstrings
[>] #2: 添加docstrings
```

**完整交互示例**：

```
User: "Refactor hello.py"
       |
       v
LLM调用todo: [{"id": "1", "text": "Read hello.py", "status": "in_progress"}, 
              {"id": "2", "text": "Add type hints", "status": "pending"}, 
              {"id": "3", "text": "Add docstrings", "status": "pending"}]
       |
       v
TodoManager返回: "[>] #1: Read hello.py\n[ ] #2: Add type hints\n[ ] #3: Add docstrings\n(0/3 completed)"
       |
       v (继续执行直到步骤完成)
       ...
       |
       v
3轮无todo调用后，自动注入: "<reminder>Update your todos.</reminder>"
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

尝试任务：

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`

**观察点**：
- LLM首次调用`todo`时如何规划任务
- 当任务状态更新时，TodoManager返回什么
- 当LLM忘记更新todo时，nag reminder何时出现

**思考点**：
- 为什么限制只能有一个`in_progress`任务？这对LLM行为有什么影响？
- nag reminder为什么插入到`results`的开头而不是末尾？
- 如果LLM在nag reminder出现后仍然忽略todo，会发生什么？

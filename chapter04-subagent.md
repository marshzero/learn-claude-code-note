## 第四章：Subagent - 如何分解复杂任务？

### 1. 问题引入

随着Agent工作时间增长，`messages`数组不断膨胀。每次文件读取、每次bash输出的结果都永久留在上下文中。当需要回答"这个项目使用什么测试框架？"时，可能需要读取5个文件，但父Agent只需要一个答案："pytest"。

更糟的是，如果让一个Agent同时处理多个复杂任务，它的上下文会被各种中间结果填满，模型容易迷失在细节中。

**核心矛盾**：复杂任务需要探索多个文件，但父Agent的上下文不应该被中间过程污染。

### 2. 解决方案

#### 2.1 核心设计

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh context
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

**关键洞察**：进程隔离天然带来上下文隔离。

**子Agent与父Agent的区别**：
- 子Agent从空的`messages=[]`开始
- 子Agent只有基础工具，没有`task`工具（防止递归嵌套）
- 子Agent最多运行30轮（安全限制）
- 子Agent只返回最终文本摘要，中间过程全部丢弃

#### 2.2 示例数据

**task工具Schema**：

```python
{"name": "task",
 "description": "Spawn a subagent with fresh context. It shares the filesystem but not conversation history.",
 "input_schema": {"type": "object", 
   "properties": {"prompt": {"type": "string"}, "description": {"type": "string"}}, 
   "required": ["prompt"]}}
```

**task工具调用示例**：

```json
{
  "type": "tool_use",
  "name": "task",
  "id": "tool_001",
  "input": {
    "prompt": "Find what testing framework this project uses. Read pyproject.toml, setup.py, or requirements.txt if they exist.",
    "description": "detect testing framework"
  }
}
```

**task工具返回结果**：

```json
{
  "type": "tool_result",
  "tool_use_id": "tool_001",
  "content": "This project uses pytest as the testing framework. Found in pyproject.toml: [tool.pytest.ini_options] section."
}
```

**数据流转详细过程**：

```
Step 1: User请求
Parent messages: [
  {"role": "user", "content": "What testing framework does this project use?"}
]

Step 2: LLM决定调用task
Parent -> API: messages + tools(PARENT_TOOLS)
Response: assistant with tool_use block

Step 3: run_subagent被调用，创建子Agent上下文
Subagent messages: [
  {"role": "user", "content": "Find what testing framework..."}  ← 干净上下文
]

Step 4: 子Agent循环
Subagent -> API: messages + tools(CHILD_TOOLS)
Response: tool_use(read_file: pyproject.toml)

Step 5: 子Agent处理tool_result
Subagent messages: [
  {"role": "user", "content": "Find what testing framework..."},
  {"role": "assistant", "content": [tool_use(read_file: pyproject.toml)]},
  {"role": "user", "content": [{"type": "tool_result", "content": "[tool.pytest.ini_options]..."}]}
]
... 继续多轮 ...

Step 6: 子Agent返回摘要
子Agent messages: [..., assistant(content: "This project uses pytest...")]
run_subagent返回: "This project uses pytest..."

Step 7: 父Agent收到结果
Parent messages: [
  {"role": "user", "content": "What testing framework..."},
  {"role": "assistant", "content": [tool_use(task)]},
  {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "tool_001", "content": "This project uses pytest..."}]}
]
```

**内存中的message数组对比**：

```
不使用Subagent（所有中间结果累积在context中）:
messages = [
  {"role": "user", "content": "What testing framework?"},
  {"role": "assistant", "content": [tool_use: read_file(pyproject.toml)]},
  {"role": "user", "content": [{"type": "tool_result", "content": "[tool.pytest..."}]},  ← 累积
  {"role": "assistant", "content": [tool_use: read_file(tests/)]},
  {"role": "user", "content": [{"type": "tool_result", "content": "test_*.py files..."}]},  ← 累积
  {"role": "assistant", "content": [tool_use: read_file(requirements.txt)]},
  {"role": "user", "content": [{"type": "tool_result", "content": "no pytest here"}]},    ← 累积
  {"role": "assistant", "content": "pytest"}  ← 最终答案
]
↑ 每次文件读取都留下痕迹，context持续膨胀

使用Subagent（父context干净，只有一条tool_result）:
Parent messages = [
  {"role": "user", "content": "What testing framework?"},
  {"role": "assistant", "content": [tool_use: task]},
  {"role": "user", "content": [{"type": "tool_result", "content": "This project uses pytest"}]}  ← 仅摘要
]

Subagent messages (被丢弃):
  [
    {"role": "user", "content": "Find..."},
    {"role": "assistant", "content": [tool_use: read_file...]},  ← 丢弃
    {"role": "user", "content": [...]},                          ← 丢弃
    ...30轮中间过程全部丢弃...
  ]
```

**关键区别总结**：

| 对比项 | 无Subagent | 有Subagent |
|--------|-----------|------------|
| 父messages增长 | 每个tool_result都累积 | 只增加一条tool_result |
| 子Agent上下文 | N/A | 独立，结束后丢弃 |
| 内存占用 | 持续增长 | 稳定 |
| 文件系统访问 | 共享 | 共享（通过WORKDIR） |
| 返回值 | 分散的tool_results | 汇总的摘要文本 |

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

尝试任务：

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`

**观察点**：
- 子Agent的输出是什么格式返回给父Agent的
- 父Agent的`messages`数组在调用`task`前后有什么变化
- 子Agent是否有权限调用`task`工具（应该没有）

**思考点**：
- 为什么子Agent不能拥有`task`工具？这防止了什么问题？
- 如果子Agent需要访问父Agent之前读取的文件内容，该怎么处理？
- 30轮的安全限制如何防止子Agent陷入无限循环？

## 第一章：让LLM调用工具——Agent Loop的设计

### 1. 问题引入

当我们让大语言模型（LLM）帮我们完成一个任务时，比如"读取当前目录下的所有文件"，会发生什么？

理论上LLM可以分析这个问题，但它**无法真正执行**任何操作——它不知道当前目录有什么文件，无法读取文件系统，更无法运行测试验证结果。

传统的解决方案是：LLM给出步骤指引，用户手动执行每个步骤，再把结果粘贴回给LLM。这中间的"翻译"工作全部由人完成，LLM只是一个"顾问"。

**核心问题：如何让LLM真正"动手"做事，而不是只给建议？**

---

### 2. 解决方案

答案出乎意料地简单——**用循环**。

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop)
```

这个循环的逻辑是：
1. 用户提出请求
2. LLM分析请求，决定是否需要调用工具
3. 如果需要，执行工具并获得结果
4. 把结果反馈给LLM，让它继续分析
5. 重复直到LLM认为任务完成

#### 2.1 核心设计

整个循环依赖两个关键设计：

**stop_reason —— 何时停止**

LLM的回复中包含一个关键字段 `stop_reason`，它告诉程序LLM为什么停止生成：

- **`tool_use`**：LLM想要调用工具，循环继续
- **`end_turn`**：LLM认为任务完成，循环终止
- **`max_tokens`**：达到输出上限，可能是异常终止

```python
response = client.messages.create(
    model=MODEL, messages=messages, tools=TOOLS
)

if response.stop_reason != "tool_use":
    return  # LLM说任务完成了，退出循环
```

用一个条件判断控制整个流程的退出。

**tool_use 块 —— 传递什么任务**

LLM的回复是一个内容块（block）列表，每个块可能是：

- **`text`**：纯文本回复
- **`tool_use`**：工具调用请求

```python
for block in response.content:
    if block.type == "tool_use":
        tool_name = block.name
        tool_input = block.input
        tool_id = block.id
```

#### 2.2 示例数据

一次完整的工具调用数据流转：

**用户请求：**
```python
{"role": "user", "content": "List all Python files in this directory"}
```

**LLM响应（stop_reason="tool_use"）：**
```python
{
    "stop_reason": "tool_use",
    "content": [
        {
            "type": "tool_use",
            "name": "bash",
            "id": "tool_123",
            "input": {"command": "ls -la *.py"}
        }
    ]
}
```

**工具执行结果：**
```python
{"role": "user", "content": [
    {
        "type": "tool_result",
        "tool_use_id": "tool_123",
        "content": "hello.py\ngreet.py\nmain.py"
    }
]}
```

**最终LLM响应（stop_reason="end_turn"）：**
```python
{
    "stop_reason": "end_turn",
    "content": [
        {
            "type": "text",
            "text": "当前目录下有3个Python文件：hello.py、greet.py、main.py"
        }
    ]
}
```

---

### 3. 实战尝试

运行示例代码：

```bash
python agents/s01_agent_loop.py
```

试试这些命令：
1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`

你会看到LLM如何一步步思考、调用工具、获取结果、再继续思考的完整过程。

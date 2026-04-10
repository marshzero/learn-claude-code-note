## 第零章：Messages Array - 如何让LLM记住对话？

### 1. 问题引入

LLM API调用是**无状态的**（stateless）。每次API调用都是独立的，服务器不保留任何之前对话的信息。

```
Without messages array (stateless):
+---------+      +---------+
|  Call 1 |      |  Call 2 |
| "Hello" |      | "Hello" |  ← No memory of Call 1
+---------+      +---------+
```

**实际问题场景**：

```
用户第1轮: "我的名字是Alice"
API调用: {"content": "我的名字是Alice"} → "你好Alice!"
        ↑ 无状态，服务器不记住任何事

用户第2轮: "我叫什么名字？"
API调用: {"content": "我叫什么名字？"} → "我不知道"
        ↑ 服务器无法知道第1轮说了什么！
```

**核心矛盾**：LLM每次调用都是全新的开始，但人类对话需要连贯性。

### 2. 解决方案

将完整的对话历史存储在`messages`数组中，每次API调用都传递这个数组。

```
WITH messages array (stateful):
messages = [
    {"role": "user", "content": "我的名字是Alice"},
    {"role": "assistant", "content": "你好Alice!"},
    {"role": "user", "content": "我叫什么名字？"},  ← 记得之前的对话
]
                 ↓
        +---------+
        |  Call 1 | → messages = [第1轮对话]
        +---------+
                 ↓
        +---------+
        |  Call 2 | → messages = [第1轮对话, 第2轮对话]  ← 完整历史
        +---------+
```

**关键洞察**：Append to messages, never overwrite（追加到messages，永不覆盖）。

### 3. 示例数据

**messages数组结构**：

```python
messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "My name is Alice."},
    {"role": "assistant", "content": "Nice to meet you, Alice!"},
    {"role": "user", "content": "What is my name?"},
    {"role": "assistant", "content": "Your name is Alice."},
]
```

| 字段 | 说明 |
|------|------|
| `role` | 角色：`system`、`user`、`assistant` |
| `content` | 消息内容文本 |

**角色说明**：

| 角色 | 用途 |
|------|------|
| `system` | 系统提示词，设置Agent行为规则（通常只放一次） |
| `user` | 用户消息 |
| `assistant` | AI回复 |

**完整对话流程**：

```python
# 初始化空数组
messages = []

# 第1轮对话
messages.append({"role": "user", "content": "My name is Alice."})
response = chat(messages)  # API调用
messages.append({"role": "assistant", "content": response})

# 第2轮对话
messages.append({"role": "user", "content": "What is my name?"})
response = chat(messages)  # API调用，包含完整历史
messages.append({"role": "assistant", "content": response})

# 第3轮对话
messages.append({"role": "user", "content": "I like coding in Python."})
response = chat(messages)  # API调用，包含前两轮历史
messages.append({"role": "assistant", "content": response})

# 此时messages包含3轮完整对话
print(messages)
# [
#   {"role": "user", "content": "My name is Alice."},
#   {"role": "assistant", "content": "Nice to meet you, Alice!"},
#   {"role": "user", "content": "What is my name?"},
#   {"role": "assistant", "content": "Your name is Alice."},
#   {"role": "user", "content": "I like coding in Python."},
#   {"role": "assistant", "content": "Python is great for data science!"},
# ]
```

### 4. 代码示例

**基本聊天函数**：

```python
def chat(messages: list) -> str:
    """Send messages to LLM and return assistant's text response."""
    response = client.messages.create(
        model=MODEL,
        system=SYSTEM,
        messages=messages,  # ← 每次都传递完整历史
        max_tokens=8000,
    )
    text = "".join(b.text for b in response.content if hasattr(b, "text"))
    return text

# 使用
messages = []
messages.append({"role": "user", "content": "My name is Alice."})
response = chat(messages)
messages.append({"role": "assistant", "content": response})
```

**对话循环**：

```python
messages = []
while True:
    user_input = input("You: ")
    messages.append({"role": "user", "content": user_input})

    response = chat(messages)
    messages.append({"role": "assistant", "content": response})

    print(f"Assistant: {response}")
```

### 5. 为什么不用全局变量或数据库？

LLM API不知道也不在乎你用什么存储。只要每次API调用时把正确的`messages`数组传进去，它就能"记住"。

```
方案A: 全局变量
global messages  # 每次API调用时需要手动传递

方案B: 数据库
messages = db.fetch_history(user_id)  # 从数据库读取
messages.append(new_message)
db.save(messages)  # 存回数据库

方案C: 文件
messages = json.load(open("history.json"))
messages.append(new_message)
json.dump(messages, open("history.json", "w"))

方案D: 内存数组（本教程采用）
messages = []  # 最简单，每次对话直接操作数组

所有方案的核心一样：确保每次API调用时传正确的messages数组。
```

### 6. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s00_messages_array.py
```

观察输出：
1. 第1轮：告诉LLM名字
2. 第2轮：询问名字，LLM正确回答
3. 对比：如果不传历史数组，LLM无法回答

**思考点**：
- messages数组会无限增长吗？（hint：会的，这导致s06的上下文压缩）
- system消息放在哪里？只放一次还是每次都放？
- 如果messages数组太长超出LLM的context window怎么办？

### 7. 与s01的关系

s01在s00的基础上增加了，s00是地基，s01在其上建造。
- **工具调用**：LLM可以调用bash等工具
- **循环机制**：`while True`直到LLM停止调用工具
- **工具结果反馈**：tool_result被加回messages数组


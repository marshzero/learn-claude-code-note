## 第二章：Tool Use - 如何扩展工具集？

### 1. 问题引入

上一章的Agent Loop只能调用`bash`工具，这意味着所有操作都通过Shell命令完成。问题随之而来：

- `cat`命令会不可预测地截断输出
- `sed`在处理特殊字符时容易失败
- 每次bash调用都是一个不受约束的安全攻击面
- 无法在工具层面实施路径沙箱隔离

**核心矛盾**：LLM需要执行文件操作，但bash命令无法提供安全、可控的操作界面。

### 2. 解决方案

#### 2.1 核心设计

关键洞察：**扩展工具集不需要修改Agent Loop本身**。只需要在dispatch map中注册新的处理器。

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+
```

**路径沙箱**：防止Agent访问工作区外的文件。

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

**Dispatch Map**：用字典替代if/elif链，新增工具只需添加条目。

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

**工具执行**：循环体内通过名称查找处理器，保持不变。

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
        results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
```

#### 2.2 示例数据

**工具定义Schema**：

```python
TOOLS = [
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit_file", "description": "Replace exact text in file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}}, "required": ["path", "old_text", "new_text"]}},
]
```

**完整的数据流转**：

```
User: "Read the file requirements.txt"
       |
       v
LLM decides to call read_file tool
       |
       v
Tool Dispatch: read_file -> run_read("requirements.txt")
       |
       v
run_read() -> safe_path() -> Path.read_text()
       |
       v
{"type": "tool_result", "content": "flask==2.0\nrequests==2.28..."}
       |
       v
LLM receives result, continues reasoning
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

尝试任务：

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`

**观察点**：
- 注意`bash`和其他工具的区别——`bash`是通用Shell，`read_file`是专用文件读取
- `safe_path()`如何阻止路径穿越攻击（如`../../etc/passwd`）
- 新增一个工具需要改几行代码？哪些地方需要改？

**思考点**：
- 为什么`edit_file`需要`old_text`和`new_text`两个参数，而不是直接传新内容？
- 如果要添加一个`list_directory`工具，需要修改哪几个地方？

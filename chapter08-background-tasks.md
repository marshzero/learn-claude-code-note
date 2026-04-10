## 第八章：Background Tasks - 如何支持异步任务？

### 1. 问题引入

有些命令需要很长时间：`npm install`、`pytest`、`docker build`。在阻塞式循环中，模型只能空闲等待。如果用户说"安装依赖的同时创建配置文件"，Agent只能串行执行，无法并行。

**核心矛盾**：模型在等待慢命令完成时无法做任何事，但很多工作其实可以同时进行。

### 2. 解决方案

#### 2.1 核心设计

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]        (parallel)
             |          |
             +-- notification queue --> [results injected]
```

**关键洞察**：启动后台任务后立即返回，Agent无需等待命令完成即可继续工作。

**BackgroundManager**：

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}  # task_id -> {status, result, command}
        self._notification_queue = []  # completed task results
        self._lock = threading.Lock()

    def run(self, command: str) -> str:
        """Start a background thread, return task_id immediately."""
        task_id = str(uuid.uuid4())[:8]
        self.tasks[task_id] = {"status": "running", "result": None, "command": command}
        thread = threading.Thread(
            target=self._execute, args=(task_id, command), daemon=True
        )
        thread.start()
        return f"Background task {task_id} started: {command[:80]}"

    def _execute(self, task_id: str, command: str):
        """Thread target: run subprocess, capture output, push to queue."""
        try:
            r = subprocess.run(command, shell=True, cwd=WORKDIR,
                capture_output=True, text=True, timeout=300)
            output = (r.stdout + r.stderr).strip()[:50000]
            status = "completed"
        except subprocess.TimeoutExpired:
            output = "Error: Timeout (300s)"
            status = "timeout"
        self.tasks[task_id]["status"] = status
        self.tasks[task_id]["result"] = output
        with self._lock:
            self._notification_queue.append({
                "task_id": task_id, "status": status,
                "command": command[:80], "result": output[:500],
            })
```

#### 2.2 示例数据

**新工具Schema**：

```python
{"name": "background_run",
 "description": "Run command in background thread. Returns task_id immediately.",
 "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}}

{"name": "check_background",
 "description": "Check background task status. Omit task_id to list all.",
 "input_schema": {"type": "object", "properties": {"task_id": {"type": "string"}}}}
```

**并行执行完整交互**：

```
User: "Install npm dependencies, run pytest, and lint in parallel. While they run, create config.yaml"

Agent调用:
  background_run(command="npm install")      → "Background task a1b2c3d4 started: npm install"
  background_run(command="pytest")             → "Background task e5f6g7h8 started: pytest"
  background_run(command="pylint .")           → "Background task i9j0k1l2 started: pylint ."

Agent立即继续执行:
  write_file(path="config.yaml", content="...")

========= 其他工作进行中，后台任务并行执行 =========

========= 当后台任务完成时，drain_notifications触发 =========

Agent收到:
  <background-results>
  [bg:a1b2c3d4] completed: npm install success
  [bg:e5f6g7h8] completed: 15 passed, 2 failed
  [bg:i9j0k1l2] completed: No issues found
  </background-results>
  Noted background results.
```

**时间线可视化**：

```
无后台任务（串行）:
|--- npm install (30s) ---|--- pytest (20s) ---|--- pylint (15s) ---|--- create config ---|
Total: 50s+ (30+20+15)

有后台任务（并行）:
|--- npm install (30s) ---|--- create config ---|
|--- pytest (20s) ---|                       ← 并行
|--- pylint (15s) ---|                      ← 并行
Total: 30s+ (最长任务的时间)
```

**drain_notifications机制详解**：

```python
def agent_loop(messages: list):
    while True:
        # 关键：每次LLM调用前先drain通知队列
        notifs = BG.drain_notifications()
        if notifs and messages:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
            )
            messages.append({"role": "user", 
                "content": f"<background-results>\n{notif_text}\n</background-results>"})
            messages.append({"role": "assistant", 
                "content": "Noted background results."})

        response = client.messages.create(...)  # LLM调用
        # ... 处理tool_use ...
```

**notification数据结构**：

```json
{
  "task_id": "a1b2c3d4",
  "status": "completed",
  "command": "npm install",
  "result": "added 1256 packages in 30s"
}
```

**check_background使用示例**：

```
User: "Check if those background tasks are done"

Agent调用:
  check_background()  → "a1b2c3d4: [completed] npm install
                          e5f6g7h8: [running] pytest
                          i9j0k1l2: [completed] pylint ."

Agent调用:
  check_background(task_id="e5f6g7h8")  → "[running] pytest\n(running)"
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

尝试任务：

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`

**观察点**：
- `background_run`返回后Agent是否立即继续工作
- 后台任务完成时，结果如何注入到对话中
- 多个后台任务是否真的并行执行

**思考点**：
- 为什么后台任务完成后结果是通过`<background-results>`注入而不是直接返回？
- 如果后台任务运行时间很长，notification queue会一直增长吗？
- daemon线程和普通线程有什么区别？

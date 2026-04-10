## 第十一章：Autonomous Agents - 如何实现自主Agent？

### 1. 问题引入

s09-s10中，teammates只在收到明确指令时才工作。Lead必须为每个任务单独指派。10个待处理任务？Lead需要逐个分配。

更糟的是，如果context compression（s06）发生，Agent可能会忘记自己是谁——身份丢失。

**核心矛盾**：Lead不想事事亲力亲为，但Agent不会主动找活干。

### 2. 解决方案

#### 2.1 核心设计

**自主Agent的工作流程**：

```
Teammate生命周期：

1. Spawn后进入WORK阶段
2. 任务完成后（或调用idle工具）进入IDLE阶段
3. IDLE阶段：每5秒检查一次inbox和任务板
4. 发现新任务？领取并进入WORK阶段
5. 60秒内什么都没找到？自动shutdown

+-------+
| spawn |
+---+---+
    |
    v
+-------+   有tool_use   +-------+
| WORK  | <------------ |  LLM  |
+---+---+               +-------+
    |
    | stop_reason != tool_use 或 idle工具被调用
    v
+--------+
|  IDLE  | 每5秒轮询一次，最长60秒
+---+----+
    |
    +---> 检查inbox --> 有消息? --> 返回WORK
    |
    +---> 扫描任务板 --> 有未领取任务? --> 领取并返回WORK
    |
    +---> 60秒超时 --> shutdown
```

**关键洞察**：Agent自己找活干，不需要Lead分配。

#### 2.2 示例数据

**新工具Schema**：

```python
{"name": "idle",
 "description": "Signal that you have no more work. Enters idle polling phase.",
 "input_schema": {"type": "object", "properties": {}}}

{"name": "claim_task",
 "description": "Claim a task from the task board by ID.",
 "input_schema": {"type": "object", "properties": {"task_id": {"type": "integer"}}, "required": ["task_id"]}}
```

**scan_unclaimed_tasks逻辑**：

```python
def scan_unclaimed_tasks() -> list:
    """找到所有满足条件的任务：pending状态 + 无owner + 无blockedBy"""
    unclaimed = []
    for f in TASKS_DIR.glob("task_*.json"):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

**完整自主协作示例**：

```
初始状态：
.tasks/
  task_1.json  {"id":1, "subject":"Write tests", "status":"pending", "owner":null, "blockedBy":[]}
  task_2.json  {"id":2, "subject":"Fix bug", "status":"pending", "owner":null, "blockedBy":[]}
  task_3.json  {"id":3, "subject":"Refactor", "status":"pending", "owner":null, "blockedBy":[]}

.teammates/
  alice (role: coder): working
  bob (role: coder): idle
```

**alice完成任务后进入IDLE**：

```
alice: 完成了task_1，调用idle()
  → 进入IDLE阶段
  → 每5秒扫描任务板

T+0s: scan_unclaimed_tasks() → [task_2, task_3]
  → claim_task(task_id=2, owner="alice")
  → task_2.json变为: {"status":"in_progress", "owner":"alice"}
  → alice收到: "<auto-claimed>Task #2: Fix bug</auto-claimed>"
  → 返回WORK阶段，执行task_2

bob也在IDLE：
T+0s: scan_unclaimed_tasks() → [task_3]  (task_2已被alice领取)
  → claim_task(task_id=3, owner="bob")
  → bob收到task_3，进入WORK阶段
```

**Identity re-injection（身份重注入）**：

Context compression后，messages可能变得很短，Agent忘记自己是谁：

```python
# 压缩后messages只有2-3条
messages = [
    {"role": "user", "content": "..."}  # 可能只有1-2条
]

# 如果len(messages) <= 3，插入身份块
if len(messages) <= 3:
    messages.insert(0, {
        "role": "user",
        "content": "<identity>You are 'alice', role: coder, team: default. Continue your work.</identity>"
    })
    messages.insert(1, {
        "role": "assistant",
        "content": "I am alice. Continuing."
    })

# 现在messages恢复正常，Agent记得自己是谁
```

**IDLE轮询机制详解**：

```python
# IDLE阶段：最多轮询12次（60s / 5s）
for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):
    time.sleep(POLL_INTERVAL)  # 等待5秒

    # 1. 检查inbox
    inbox = BUS.read_inbox(name)
    if inbox:
        messages.append({"role": "user", "content": f"<inbox>{inbox}</inbox>"})
        return True  # 有消息，返回WORK

    # 2. 扫描任务板
    unclaimed = scan_unclaimed_tasks()
    if unclaimed:
        task = unclaimed[0]
        claim_task(task["id"], name)
        messages.append({"role": "user", "content": f"<auto-claimed>Task #{task['id']}: {task['subject']}</auto-claimed>"})
        return True  # 领取到任务，返回WORK

# 3. 60秒内什么都没找到，shutdown
return False  # 超时，Agent退出
```

**任务板状态变化**：

```
初始:
[ ] #1: Write tests @null
[ ] #2: Fix bug @null
[ ] #3: Refactor @null

alice领取task_1:
[>] #1: Write tests @alice

alice完成task_1后，进入IDLE，领取task_2:
[x] #1: Write tests @alice
[>] #2: Fix bug @alice

bob同时在IDLE，领取task_3:
[x] #1: Write tests @alice
[>] #2: Fix bug @alice
[>] #3: Refactor @bob
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

尝试任务：

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. Type `/tasks` to see the task board with owners
5. Type `/team` to monitor who is working vs idle

**观察点**：
- teammates如何在IDLE阶段自动领取任务
- 任务被领取后owner字段如何变化
- IDLE阶段超时后teammate状态如何变为shutdown

**思考点**：
- 为什么teammate会调用idle工具？什么情况下会这样做？
- 如果两个teammate同时扫描到同一个任务会发生什么？（hint：_claim_lock）
- identity re-injection为什么只在messages <= 3时触发？

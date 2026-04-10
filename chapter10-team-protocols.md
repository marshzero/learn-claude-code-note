## 第十章：Team Protocols - 如何定义Agent间的协议？

### 1. 问题引入

s09中teammates可以工作和通信，但缺乏结构化协调。想象一个场景：alice正在写一个重要文件，写了80%，这时lead说"停，这个需求取消了"。

**直接terminate线程的问题**：
- alice正在写文件 → 线程被强制终止 → 文件损坏！
- alice正在运行测试 → 线程被强制终止 → 测试状态不一致！

**没有shutdown协议时**：
```
Lead: "alice别干了"
  → 只能terminate线程 → 文件可能损坏、config.json状态过时
```

**Plan approval问题**：当lead说"重构auth模块"时，teammate立即开始。对于高风险变更，lead应该先审核计划，避免做无用功。

**核心矛盾**：Agent间通信需要规则，否则协作混乱且无法协调。

### 2. 解决方案

#### 2.1 核心设计

两种协议本质相同：**一方发请求，另一方回复，同一个ID关联两条消息**。

**Shutdown协议**：
```
1. Lead发消息给alice: "请停止" + 带上ID=001
2. alice回复: "好的" + 带上ID=001
   (或者: "不行，我还没完成" + ID=001)
```

**Plan approval协议**：
```
1. alice发消息给lead: "我的计划是..." + 带上ID=002
2. lead回复: "批准" + 带上ID=002
   (或者: "不行，重做" + ID=002)
```

**核心机制**：用request_id把"请求"和"回复"关联起来。

```python
# 发请求时记录
shutdown_requests["abc123"] = {"target": "alice", "status": "pending"}

# alice回复时查找
shutdown_requests["abc123"]["status"] = "approved"
```

**关键洞察**：两个协议本质一样，区别只是谁先发起、内容是什么。

#### 2.2 示例数据

**Tracker数据结构**：

```python
shutdown_requests = {
    "abc123": {"target": "alice", "status": "pending"},
    "def456": {"target": "bob", "status": "approved"},
}

plan_requests = {
    "xyz789": {"from": "alice", "plan": "Refactor auth module...", "status": "pending"},
}
```

**新工具Schema**：

```python
{"name": "shutdown_request",
 "description": "Request a teammate to shut down gracefully. Returns a request_id for tracking.",
 "input_schema": {"type": "object", "properties": {"teammate": {"type": "string"}}, "required": ["teammate"]}}

{"name": "shutdown_response",
 "description": "Respond to a shutdown request. Approve to shut down, reject to keep working.",
 "input_schema": {"type": "object", 
   "properties": {"request_id": {"type": "string"}, "approve": {"type": "boolean"}, "reason": {"type": "string"}}, 
   "required": ["request_id", "approve"]}}

{"name": "plan_approval",
 "description": "Submit a plan for lead approval. Provide plan text.",
 "input_schema": {"type": "object", "properties": {"plan": {"type": "string"}}, "required": ["plan"]}}
```

**Shutdown协议完整交互**：

```
Step 1: Lead发送shutdown_request
Agent调用:
  shutdown_request(teammate="alice")
  → 生成 req_id="abc123"
  → shutdown_requests["abc123"] = {"target": "alice", "status": "pending"}
  → BUS.send("lead", "alice", "Please shut down gracefully.", "shutdown_request", {"request_id": "abc123"})
  → "Shutdown request abc123 sent to 'alice' (status: pending)"

Step 2: alice收到shutdown_request
alice的inbox中有:
  {"type": "shutdown_request", "from": "lead", "content": "Please shut down gracefully.", "request_id": "abc123"}

alice调用:
  shutdown_response(request_id="abc123", approve=true, reason="Work completed")
  → shutdown_requests["abc123"]["status"] = "approved"
  → BUS.send("alice", "lead", "Work completed", "shutdown_response", {"request_id": "abc123", "approve": true})
  → should_exit = True, alice的teammate_loop退出

Step 3: alice状态变为shutdown
config.json中:
  {"name": "alice", "role": "coder", "status": "shutdown"}
```

**Shutdown协议 - reject场景**：

```
Step 1: Lead发送shutdown_request
Agent调用:
  shutdown_request(teammate="bob")
  → 生成 req_id="def456"
  → shutdown_requests["def456"] = {"target": "bob", "status": "pending"}
  → "Shutdown request def456 sent to 'bob' (status: pending)"

Step 2: bob正在关键任务中，拒绝shutdown
bob的inbox中有:
  {"type": "shutdown_request", "from": "lead", "content": "Please shut down gracefully.", "request_id": "def456"}

bob调用:
  shutdown_response(request_id="def456", approve=false, reason="Running final test suite, 2 minutes left")
  → shutdown_requests["def456"]["status"] = "rejected"
  → BUS.send("bob", "lead", "Running final test suite, 2 minutes left", "shutdown_response", {"request_id": "def456", "approve": false})
  → should_exit = False, bob继续工作

Step 3: Lead检查状态
Agent调用:
  shutdown_response(request_id="def456")
  → {"target": "bob", "status": "rejected"}

bob继续正常工作，知道shutdown请求被拒绝
```

---

**Plan Approval协议完整交互**：

```
Step 1: alice提交plan
alice调用:
  plan_approval(plan="Step 1: Read auth.py\nStep 2: Extract validator\nStep 3: Add tests\nStep 4: Commit")
  → 生成 req_id="xyz789"
  → plan_requests["xyz789"] = {"from": "alice", "plan": "...", "status": "pending"}
  → BUS.send("alice", "lead", plan_text, "plan_approval_response", {"request_id": "xyz789", "plan": plan_text})
  → "Plan submitted (request_id=xyz789). Waiting for lead approval."

Step 2: Lead收到plan
Lead的inbox中有:
  {"type": "plan_approval_response", "from": "alice", "content": "Step 1: Read auth.py...", "request_id": "xyz789", "plan": "..."}

Step 3: Lead审核并回复
Agent调用:
  plan_approval(request_id="xyz789", approve=true, feedback="Good plan, proceed.")
  → plan_requests["xyz789"]["status"] = "approved"
  → BUS.send("lead", "alice", "Good plan, proceed.", "plan_approval_response", {"request_id": "xyz789", "approve": true})

Step 4: alice收到回复
alice的inbox中有:
  {"type": "plan_approval_response", "from": "lead", "content": "Good plan, proceed.", "request_id": "xyz789", "approve": true}
alice继续执行plan中的步骤
```

**reject场景示例**：

```
Lead调用:
  plan_approval(request_id="xyz789", approve=false, feedback="Too risky. Add rollback plan first.")
  → plan_requests["xyz789"]["status"] = "rejected"
  → BUS.send("lead", "alice", "Too risky. Add rollback plan first.", "plan_approval_response", {"request_id": "xyz789", "approve": false})

alice收到:
  {"type": "plan_approval_response", "from": "lead", "content": "Too risky. Add rollback plan first.", "approve": false}
alice需要修改plan并重新提交
```

---

**Teammate的系统提示词变化**：

```python
# s09
sys_prompt = f"You are '{name}', role: {role}, at {WORKDIR}. Use send_message to communicate."

# s10
sys_prompt = (
    f"You are '{name}', role: {role}, at {WORKDIR}. "
    f"Submit plans via plan_approval before major work. "
    f"Respond to shutdown_request with shutdown_response."
)
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

尝试任务：

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. Type `/team` to monitor statuses

**观察点**：
- shutdown后teammate的状态如何变化
- plan被reject后teammate如何响应
- request_id如何在两个消息间关联

**思考点**：
- 为什么两种协议都使用相同的FSM结构？
- 如果teammate收到shutdown_request但正在关键任务中，应该怎么处理？
- plan_approval和直接开始工作相比，有什么优势？

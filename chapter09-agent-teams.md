## 第九章：Agent Teams - 如何实现多Agent协作？

### 1. 问题引入

s04的Subagent是一次性的：生成、工作、返回摘要、销毁。没有身份，两次调用之间没有记忆。s08的后台任务可以运行Shell命令，但无法做出LLM引导的决策。

真正的团队协作需要：
1. 持久化的Agent，生命周期超越单个prompt
2. 身份和生命周期管理
3. Agent之间的通信通道

**核心矛盾**：单个Agent能力有限，但多个Agent需要协调、通信和状态共享。

### 2. 解决方案

#### 2.1 核心设计

**Subagent vs Teammate**：

```
Subagent (s04):  spawn -> execute -> return summary -> destroyed
Teammate (s09):  spawn -> work -> idle -> work -> ... -> shutdown
```

**团队结构**：

```
.team/
  config.json                   ← 团队成员列表 + 状态
  inbox/
    alice.jsonl               ← 追加式消息队列
    bob.jsonl
    lead.jsonl

通信流程:
  +--------+    send("alice","bob")    +--------+
  | alice  | -----------------------> |  bob   |
  | loop   |    bob.jsonl << {json}    |  loop  |
  +--------+                          +--------+
       ^
       |        BUS.read_inbox("alice")
       +---- alice.jsonl -> 读取并清空 ----+
```

**关键洞察**：能够相互对话的持久化Agent。

#### 2.2 示例数据

**config.json结构**：

```json
{
  "team_name": "default",
  "members": [
    {"name": "alice", "role": "coder", "status": "idle"},
    {"name": "bob", "role": "tester", "status": "working"}
  ]
}
```

**MessageBus的JSONL格式**：

```jsonl
{"type": "message", "from": "alice", "content": "Fixed the bug in parser.py", "timestamp": 1699999999.123}
{"type": "message", "from": "bob", "content": "Tests passing now", "timestamp": 1699999999.456}
```

**新工具Schema**：

```python
{"name": "spawn_teammate",
 "description": "Spawn a persistent teammate that runs in its own thread.",
 "input_schema": {"type": "object", 
   "properties": {"name": {"type": "string"}, "role": {"type": "string"}, "prompt": {"type": "string"}}, 
   "required": ["name", "role", "prompt"]}}

{"name": "send_message",
 "description": "Send a message to a teammate's inbox.",
 "input_schema": {"type": "object", 
   "properties": {"to": {"type": "string"}, "content": {"type": "string"}, "msg_type": {"type": "string"}}, 
   "required": ["to", "content"]}}

{"name": "broadcast",
 "description": "Send a message to all teammates.",
 "input_schema": {"type": "object", 
   "properties": {"content": {"type": "string"}}, 
   "required": ["content"]}}

{"name": "list_teammates",
 "description": "List all teammates with name, role, status.",
 "input_schema": {"type": "object", "properties": {}}}
```

**完整协作示例**：

```
User: "Spawn alice (coder) to fix the parser bug, and bob (tester) to verify"

Agent调用:
  spawn_teammate(name="alice", role="coder", prompt="Fix the bug in parser.py")
  → "Spawned 'alice' (role: coder)"
  spawn_teammate(name="bob", role="tester", prompt="Run pytest and verify all tests pass")
  → "Spawned 'bob' (role: tester)"

Agent调用:
  list_teammates()
  → "Team: default
     alice (coder): working
     bob (tester): working"

--- alice 和 bob 在各自线程中并行工作 ---

alice的工作:
  _teammate_loop(name="alice", role="coder", prompt="Fix the bug in parser.py")
  messages = [{"role": "user", "content": "Fix the bug in parser.py"}]
  执行: read_file("parser.py") → edit_file(...) → bash("git commit...")
  完成后alice调用 send_message(to="bob", content="Bug fixed in parser.py")
  → bob.jsonl追加: {"type": "message", "from": "alice", "content": "Bug fixed...", "timestamp": ...}

bob的工作:
  _teammate_loop(name="bob", role="tester", prompt="Run pytest...")
  检查inbox: 发现alice的消息
  messages.append({"role": "user", "content": inbox消息})
  执行: bash("pytest") → 查看结果 → send_message(to="alice", content="All tests passing!")
```

**teammate_loop执行流程**：

```python
def _teammate_loop(self, name: str, role: str, prompt: str):
    sys_prompt = f"You are '{name}', role: {role}, at {WORKDIR}. Use send_message to communicate."
    messages = [{"role": "user", "content": prompt}]

    for _ in range(50):  # 最多50轮
        inbox = BUS.read_inbox(name)  # 每次循环检查inbox
        for msg in inbox:
            messages.append({"role": "user", "content": json.dumps(msg)})

        response = client.messages.create(...)  # LLM调用

        if response.stop_reason != "tool_use":
            break

        for block in response.content:
            if block.type == "tool_use":
                output = self._exec(name, block.name, block.input)
                results.append({"type": "tool_result", ...})
        messages.append({"role": "user", "content": results})

    # 循环结束后变为idle
    member["status"] = "idle"
```

**broadcast示例**：

```python
# Agent广播消息给所有队友
broadcast(content="Phase 1 complete, moving to integration testing")

# 内部实现: 遍历所有队友，逐一发送
def broadcast(sender, content, teammates):
    for name in teammates:
        if name != sender:
            BUS.send(sender, name, content, "broadcast")
```

**与s04 Subagent的对比**：

| 特性 | s04 Subagent | s09 Teammate |
|------|-------------|--------------|
| 生命周期 | 单次调用 | 持久化（可多次任务） |
| 状态 | 无 | 有（working/idle/shutdown） |
| 内存 | 销毁后丢失 | 可保持（直到shutdown） |
| 通信 | 无 | 有（send_message/inbox） |
| 身份 | 无 | 有（name/role） |

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

尝试任务：

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. Type `/team` to see the team roster with statuses
5. Type `/inbox` to manually check the lead's inbox

**观察点**：
- teammates如何通过jsonl文件传递消息
- spawn后teammate的状态变化
- inbox如何实现"读取并清空"

**思考点**：
- 为什么inbox要用JSONL格式而不是普通JSON数组？
- 如果teammate崩溃了，其他teammate如何知道？
- 消息类型有message、broadcast、shutdown_request等，s10会用哪些？

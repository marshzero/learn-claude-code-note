# learn-claude-code 教程总结

## 概述

learn-claude-code教程通过12个章节，完整构建了一个多Agent协作系统。每章建立在前一章的基础上，逐步增加复杂性。

教程地址: https://learn.shareai.run/

代码地址: https://github.com/shareAI-lab/learn-claude-code

本文是对该教程的总结，仅供学习和参考。

## 章节架构

```
s01-s06: 单Agent基础设施
┌─────────────────────────────────────────────────────────┐
│ s01 Agent Loop      │ 基础框架：LLM + 工具调用循环       │
│ s02 Tool Use        │ 扩展工具集：dispatch map模式       │
│ s03 Todo Write      │ 任务规划：TodoManager状态机        │
│ s04 Subagent        │ 上下文隔离：子Agent独立消息队列    │
│ s05 Skill Loading   │ 按需加载：两层注入机制             │
│ s06 Context Compact │ 上下文压缩：三层压缩策略           │
└─────────────────────────────────────────────────────────┘
                              ↓
s07-s12: 多Agent协作系统
┌─────────────────────────────────────────────────────────┐
│ s07 Task System    │ 持久化任务图：依赖管理             │
│ s08 Background     │ 异步执行：后台线程+通知队列         │
│ s09 Agent Teams    │ 多Agent通信：JSONL邮箱机制         │
│ s10 Team Protocols │ 协作协议：request_id握手机制      │
│ s11 Autonomous     │ 自主Agent：IDLE轮询+任务领取      │
│ s12 Worktree       │ 任务隔离：git worktree目录隔离     │
└─────────────────────────────────────────────────────────┘
```

## 核心概念演进

### 1. 工具调用 (s01-s02)

**s01**: 基础的Agent循环，LLM决定调用`bash`工具。

**s02**: 引入dispatch map，新增`read_file`、`write_file`、`edit_file`，通过`safe_path()`实现路径沙箱。

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"]),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

### 2. 任务规划 (s03)

TodoManager实现简单的状态追踪：
- 状态：`pending` → `in_progress` → `completed`
- Nag reminder：3轮无更新则注入提醒

### 3. 上下文管理 (s04-s06)

**s04 Subagent**: 子Agent拥有独立消息队列，避免污染父Agent上下文。

**s05 Skill Loading**: 两层注入
- Layer 1: 技能名称（系统提示词，~100 tokens/skill）
- Layer 2: 技能内容（按需加载，~2000 tokens/skill）

**s06 Context Compact**: 三层压缩
- Layer 1: micro_compact（每轮自动，tool_result→占位符）
- Layer 2: auto_compact（阈值触发，全量→摘要）
- Layer 3: manual compact（Agent手动触发）

### 4. 多Agent协作 (s07-s12)

**s07 Task System**: 持久化任务图
```json
{
  "id": 1,
  "subject": "Design API",
  "status": "pending",
  "blockedBy": [],
  "blocks": [2, 3]
}
```

**s08 Background Tasks**: 后台线程执行
- `background_run`: 启动后台任务，立即返回task_id
- `check_background`: 查询任务状态
- drain_notifications: 每轮LLM调用前注入完成通知

**s09 Agent Teams**: 基于JSONL的邮箱通信
```
.team/
  config.json       # 团队成员状态
  inbox/
    alice.jsonl     # 追加式消息队列
    bob.jsonl
```

**s10 Team Protocols**: 两种握手机制
- Shutdown协议：优雅终止teammate
- Plan approval协议：任务审批

**s11 Autonomous Agents**: IDLE轮询
- WORK → IDLE → (检查inbox | 扫描任务板 | 超时shutdown)
- identity re-injection: 压缩后重注身份

**s12 Worktree Isolation**: git worktree隔离
- 每个task绑定独立的git分支和目录
- 防止并行任务相互干扰

## 数据流总览

```
用户请求
    ↓
Agent Loop (s01)
    ↓
Tool Dispatch (s02)
    ↓
TodoManager规划 (s03) ←→ SkillLoader技能 (s05)
    ↓
Subagent隔离 (s04) ←→ Context压缩 (s06)
    ↓
TaskManager任务图 (s07)
    ↓
BackgroundManager异步 (s08)
    ↓
TeammateManager多Agent (s09)
    ↓
Protocol Handlers协议 (s10)
    ↓
AutonomousIdle循环 (s11)
    ↓
WorktreeManager隔离 (s12)
```

## 关键设计模式

1. **Dispatch Map Pattern**: 新增工具只需添加handler，无需修改循环体
2. **Two-Layer Injection**: 名称vs内容分离，按需加载
3. **Request-Response Correlation**: 用request_id关联配对的消息
4. **Append-Only Inbox**: JSONL追加写入，避免读写冲突
5. **State Machine**: 单一状态转换规则（如pending→approved/rejected）

## 文件结构

```
learn-claude-code/
├── agents/
│   ├── s01_agent_loop.py
│   ├── s02_tool_use.py
│   ├── s03_todo_write.py
│   ├── s04_subagent.py
│   ├── s05_skill_loading.py
│   ├── s06_context_compact.py
│   ├── s07_task_system.py
│   ├── s08_background_tasks.py
│   ├── s09_agent_teams.py
│   ├── s10_team_protocols.py
│   ├── s11_autonomous_agents.py
│   └── s12_worktree_task_isolation.py
├── chapter01-agent-loop.md
├── chapter02-tool-use.md
├── chapter03-todo-write.md
├── chapter04-subagent.md
├── chapter05-skill-loading.md
├── chapter06-context-compact.md
├── chapter07-task-system.md
├── chapter08-background-tasks.md
├── chapter09-agent-teams.md
├── chapter10-team-protocols.md
├── chapter11-autonomous-agents.md
├── chapter12-worktree-isolation.md
└── chapter-summary.md
```

## 进一步探索

- 如何实现任务的优先级调度？
- 如何处理teammate崩溃的容错？
- 如何添加更多协议类型（如vote、delegate）？
- 如何集成外部服务（如GitHub PR、Slack通知）？

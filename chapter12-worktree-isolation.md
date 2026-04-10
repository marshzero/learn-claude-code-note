## 第十二章：Worktree Isolation - 如何实现任务隔离？

### 1. 问题引入

s09-s11中，多个teammates并行工作，但都在同一个目录里。这意味着：
- 两个teammate可能同时修改同一个文件
- 无法并行在同一个文件的不同版本上工作
- 危险操作（如大范围重构）无法安全隔离测试

**核心矛盾**：并行任务需要隔离，否则会相互干扰。

### 2. 解决方案

#### 2.1 核心设计

**Git Worktree**：每个task绑定一个独立的git分支和目录。

```
主仓库 (main):
  /project/
  
Worktree (auth-refactor):
  /project/.worktrees/auth-refactor/
  - 基于main分支创建
  - 独立的工作目录
  - 可以独立commit/branch
  
Worktree (api-v2):
  /project/.worktrees/api-v2/
  - 另一个独立工作目录
```

**任务与Worktree绑定**：

```
.tasks/task_12.json           .worktrees/index.json
{                              {
  "id": 12,                     "worktrees": [
  "subject": "auth refactor",     {
  "status": "in_progress",       "name": "auth-refactor",
  "worktree": "auth-refactor"    "path": ".../.worktrees/auth-refactor",
}                                "task_id": 12,
                                  "status": "active"
                                }]
                              }
```

**关键洞察**：通过目录隔离实现并行执行，互不干扰。

#### 2.2 示例数据

**新工具Schema**：

```python
{"name": "worktree_create",
 "description": "Create a git worktree and optionally bind it to a task.",
 "input_schema": {"type": "object", 
   "properties": {"name": {"type": "string"}, "task_id": {"type": "integer"}, "base_ref": {"type": "string"}}, 
   "required": ["name"]}}

{"name": "worktree_run",
 "description": "Run a shell command in a named worktree directory.",
 "input_schema": {"type": "object", 
   "properties": {"name": {"type": "string"}, "command": {"type": "string"}}, 
   "required": ["name", "command"]}}

{"name": "worktree_keep",
 "description": "Mark a worktree as kept in lifecycle state without removing it.",
 "input_schema": {"type": "object", 
   "properties": {"name": {"type": "string"}}, 
   "required": ["name"]}}

{"name": "worktree_remove",
 "description": "Remove a worktree and optionally mark its bound task completed.",
 "input_schema": {"type": "object", 
   "properties": {"name": {"type": "string"}, "force": {"type": "boolean"}, "complete_task": {"type": "boolean"}}, 
   "required": ["name"]}}

{"name": "task_bind_worktree",
 "description": "Bind a task to a worktree name.",
 "input_schema": {"type": "object", 
   "properties": {"task_id": {"type": "integer"}, "worktree": {"type": "string"}, "owner": {"type": "string"}}, 
   "required": ["task_id", "worktree"]}}
```

**完整工作流示例**：

```
初始状态：主仓库中有一个main分支

Step 1: 创建任务
Agent调用:
  task_create(subject="Refactor authentication module")
  → 返回: {"id": 1, "subject": "Refactor authentication", "status": "pending", "worktree": ""}

Step 2: 为任务创建worktree
Agent调用:
  worktree_create(name="auth-refactor", task_id=1, base_ref="main")
  → git worktree add -b wt/auth-refactor .worktrees/auth-refactor main
  → 返回: {"name": "auth-refactor", "path": "...", "branch": "wt/auth-refactor", "task_id": 1, "status": "active"}
  
  同时自动绑定:
  task_bind_worktree(task_id=1, worktree="auth-refactor")
  → task_1.json更新: {"worktree": "auth-refactor", "status": "in_progress"}

Step 3: 在worktree中工作
Agent调用:
  worktree_run(name="auth-refactor", command="python -m pytest tests/")
  → 在 .worktrees/auth-refactor/ 目录中运行pytest
  
  worktree_run(name="auth-refactor", command="git add . && git commit -m 'refactor auth'")
  → 在独立的分支中提交

Step 4: 完成工作 - 选择keep或remove

选项A: Keep (保留更改)
  worktree_keep(name="auth-refactor")
  → worktree状态变为"kept"，保留在文件系统中
  → 可以后续merge到main

选项B: Remove (放弃更改)
  worktree_remove(name="auth-refactor", complete_task=true)
  → git worktree remove .worktrees/auth-refactor
  → 删除worktree目录
  → task状态变为"completed"
```

**worktree状态流转**：

```
worktree生命周期:
  active -> kept (保留更改)
       \-> removed (删除worktree，complete_task=true时标记task完成)
       \-> removed (删除worktree，放弃更改)
```

**EventBus追踪生命周期**：

```python
# .worktrees/events.jsonl 中记录所有事件
{"event": "worktree.create.before", "ts": 1699999999.0, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "base_ref": "main"}}
{"event": "worktree.create.after", "ts": 1699999999.1, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "path": "...", "status": "active"}}
{"event": "worktree.remove.before", "ts": 1700000000.0, "task": {"id": 1}, "worktree": {"name": "auth-refactor"}}
{"event": "worktree.remove.after", "ts": 1700000000.1, "task": {"id": 1}, "worktree": {"name": "auth-refactor", "status": "removed"}}
```

**并行任务示例**：

```
场景：需要同时重构auth模块和api模块

1. 创建两个任务
   task_create(subject="Refactor auth")      → id=1
   task_create(subject="Refactor API")      → id=2

2. 创建两个worktree
   worktree_create(name="auth-refactor", task_id=1)
   worktree_create(name="api-refactor", task_id=2)

3. 并行工作
   worktree_run(name="auth-refactor", command="python -m pytest auth/")
   worktree_run(name="api-refactor", command="python -m pytest api/")
   
   两个teammate可以同时在不同的目录工作，互不干扰！

4. 分别完成
   worktree_keep(name="auth-refactor")   → auth重构完成，保留更改
   worktree_remove(name="api-refactor")   → api重构取消，放弃更改
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

尝试任务：

1. `Create a task, then create a worktree for it. Run some commands in the worktree.`
2. `Create two tasks with worktrees, run commands in both in parallel.`
3. `Use worktree_keep to keep changes, or worktree_remove to discard.`
4. `Check worktree_events to see the lifecycle events.`

**观察点**：
- worktree创建后，主仓库的.git/worktrees目录发生了什么
- `worktree_run`和普通`bash`工具的区别
- worktree_remove和worktree_keep的区别

**思考点**：
- 为什么worktree需要绑定到task？这样设计有什么好处？
- 如果一个teammate在worktree中工作时，另一个teammate删除了这个worktree，会发生什么？
- worktree的base_ref参数有什么用？

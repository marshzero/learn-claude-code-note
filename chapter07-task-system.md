## 第七章：Task System - 如何管理多任务？

### 1. 问题引入

s03的TodoManager是一个简单的内存清单：没有顺序、没有依赖、只有完成/未完成两种状态。但实际目标有结构：
- 任务B依赖任务A
- 任务C和D可以并行执行
- 任务E需要等待C和D都完成

没有明确的依赖关系，Agent无法判断：
- 哪些任务可以执行
- 哪些任务被阻塞
- 哪些任务可以并行运行

更重要的是，清单位于内存中，s06的上下文压缩会将其清空。

**核心矛盾**：复杂任务需要依赖图来协调，但对话上下文无法持久保存状态。

### 2. 解决方案

#### 2.1 核心设计

将清单升级为**持久化的任务图**。每个任务是`.tasks/`目录下的JSON文件，包含状态和依赖关系（`blockedBy`、`blocks`）。

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

Task Graph (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+   +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

Ordering:     task 1 必须在 task 2 和 task 3 之前完成
Parallelism:  task 2 和 task 3 可以同时执行
Dependencies:  task 4 等待 task 2 和 task 3 都完成
Status:       pending -> in_progress -> completed
```

**关键洞察**：状态存在于对话之外，所以上下文压缩不会丢失。

#### 2.2 示例数据

**Task JSON结构**：

```json
{
  "id": 1,
  "subject": "Setup project structure",
  "description": "Create directories and initial files",
  "status": "pending",
  "blockedBy": [],
  "blocks": [2, 3],
  "owner": ""
}
```

| 字段 | 说明 |
|------|------|
| `id` | 唯一标识符 |
| `subject` | 任务主题 |
| `description` | 详细描述 |
| `status` | `pending` / `in_progress` / `completed` |
| `blockedBy` | 依赖的任务ID列表（这些任务必须先完成） |
| `blocks` | 被当前任务阻塞的任务ID列表 |
| `owner` | 任务所有者（用于多Agent场景） |

**任务工具Schema**：

```python
# task_create
{"name": "task_create",
 "input_schema": {"type": "object", "properties": {"subject": {"type": "string"}, "description": {"type": "string"}}, "required": ["subject"]}}

# task_update
{"name": "task_update",
 "input_schema": {"type": "object", "properties": {"task_id": {"type": "integer"}, "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}, "addBlockedBy": {"type": "array", "items": {"type": "integer"}}, "addBlocks": {"type": "array", "items": {"type": "integer"}}}, "required": ["task_id"]}}

# task_list
{"name": "task_list", "input_schema": {"type": "object", "properties": {}}}

# task_get
{"name": "task_get", "input_schema": {"type": "object", "properties": {"task_id": {"type": "integer"}}, "required": ["task_id"]}}
```

**4任务依赖图示例**：

```
User: "Create tasks: design API, then frontend and backend in parallel, finally integration test"
```

**Task Graph (DAG)**：

```
            +------------+
            | task_1     |
            | Design API |
            | (无依赖)   |
            +-----+------+
                  |
        +---------+---------+
        |                   |
        v                   v
+------------+       +------------+
| task_2     |       | task_3     |
| Frontend   |       | Backend    |
| blocked:1  |       | blocked:1  |
+------------+       +------------+
        |                   |
        +---------+---------+
                  |
                  v
            +------------+
            | task_4     |
            | Integration|
            | blocked:2,3|
            +------------+
```

**创建任务并设置依赖**：

```python
# 1. 创建4个任务
task_create(subject="Design API")        → id=1
task_create(subject="Frontend dev")      → id=2
task_create(subject="Backend dev")        → id=3
task_create(subject="Integration test")  → id=4

# 2. 设置依赖关系
task_update(task_id=2, addBlockedBy=[1])   # Frontend blocked by Design
task_update(task_id=3, addBlockedBy=[1])   # Backend blocked by Design
task_update(task_id=4, addBlockedBy=[2, 3])  # Integration blocked by both

# 3. 磁盘上的task文件
.tasks/
  task_1.json  {"id":1, "status":"pending", "blockedBy":[], "blocks":[2,3]}
  task_2.json  {"id":2, "status":"pending", "blockedBy":[1], "blocks":[4]}
  task_3.json  {"id":3, "status":"pending", "blockedBy":[1], "blocks":[4]}
  task_4.json  {"id":4, "status":"pending", "blockedBy":[2,3], "blocks":[]}
```

**task_list状态演进**：

```
初始状态:
[ ] #1: Design API
[ ] #2: Frontend dev (blocked by: [1])
[ ] #3: Backend dev (blocked by: [1])
[ ] #4: Integration test (blocked by: [2, 3])

Phase 1: task_1完成
[x] #1: Design API
[ ] #2: Frontend dev  ← 可执行
[ ] #3: Backend dev   ← 可执行
[ ] #4: Integration test (blocked by: [2, 3])

Phase 2: task_2和task_3并行执行（可同时进行）
[x] #1: Design API
[>] #2: Frontend dev  ← 执行中
[>] #3: Backend dev   ← 执行中
[ ] #4: Integration test (blocked by: [2, 3])

Phase 3: task_2完成，task_4仍阻塞
[x] #1: Design API
[x] #2: Frontend dev
[ ] #3: Backend dev   ← 仍在执行
[ ] #4: Integration test (blocked by: [2, 3])

Phase 4: task_3完成，task_4解封
[x] #1: Design API
[x] #2: Frontend dev
[x] #3: Backend dev
[ ] #4: Integration test  ← 可执行
```

**依赖解析详解**：

```python
# 当task_1(Design API)完成时调用:
_clear_dependency(1)
# 遍历task_2: blockedBy=[1] → 移除1 → blockedBy=[]
# 遍历task_3: blockedBy=[1] → 移除1 → blockedBy=[]
# task_2和task_3现在可以并行执行！

# 当task_2(Frontend)完成时调用:
_clear_dependency(2)
# 遍历task_4: blockedBy=[2,3] → 移除2 → blockedBy=[3]
# task_4还在阻塞（等待task_3）

# 当task_3(Backend)完成时调用:
_clear_dependency(3)
# 遍历task_4: blockedBy=[3] → 移除3 → blockedBy=[]
# task_4现在可以执行！
```

**与s03 TodoManager的对比**：

| 特性 | s03 TodoManager | s07 TaskManager |
|------|-----------------|-----------------|
| 存储位置 | 内存 | `.tasks/` 目录 (磁盘) |
| 依赖关系 | 无 | `blockedBy` / `blocks` |
| 并行执行 | 不支持 | 支持（多个任务可同时ready） |
| 汇聚依赖 | 不支持 | 支持（task等待多个上游） |
| 状态 | completed / 未完成 | pending / in_progress / completed |
| 持久化 | 上下文压缩会丢失 | 永久保存 |
| 适用场景 | 快速单会话 | 多步骤长期项目 |

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

尝试任务：

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`

**观察点**：
- 任务以什么格式保存在磁盘上
- 当一个任务完成时，其他被阻塞的任务发生了什么变化
- `blockedBy` 和 `blocks` 的区别

**思考点**：
- 如果任务A和B都完成后，任务C依赖[A, B]，那么_clear_dependency是如何知道同时移除A和B的？
- 双向更新（add_blocks 同时更新 blocked task 的 blockedBy）有什么作用？
- 这个任务系统如何成为 s08（后台任务）、s09（多Agent协作）、s12（工作树隔离）的基础？

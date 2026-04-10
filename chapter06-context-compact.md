## 第六章：Context Compact - 如何管理上下文长度？

### 1. 问题引入

上下文窗口是有限的。一次`read_file`读取1000行文件大约消耗4000 tokens。读取30个文件并运行20个bash命令后，上下文就会超过100,000 tokens。Agent无法在大型代码库上工作而不进行压缩。

**核心矛盾**：长时间工作会导致上下文持续膨胀，但Agent需要访问历史信息来完成连贯的任务。

### 2. 解决方案

#### 2.1 核心设计

三层压缩策略，逐步加强：

```
Every turn:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        (silent, every turn)
  Replace tool_result > 3 turns old
  with "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              Save transcript to .transcripts/
              LLM summarizes conversation.
              Replace all messages with [summary].
                    |
                    v
            [Layer 3: compact tool]
              Model calls compact explicitly.
              Same summarization as auto_compact.
```

**关键洞察**：Agent可以有策略地遗忘，保持长期工作。

#### 2.2 示例数据

**micro_compact的KEEP_RECENT逻辑**：

代码中`tool_results[:-KEEP_RECENT]`表示保留最后3个，其余压缩。如果有10个tool_results：

```
tool_results = [t1, t2, t3, t4, t5, t6, t7, t8, t9, t10]
                ↓
tool_results[:-3] = [t1, t2, t3, t4, t5, t6, t7]  → 压缩
tool_results[-3:] = [t8, t9, t10]                  → 保留
```

**Layer 1 - micro_compact执行过程**：

```python
# 假设有6个tool_results在messages中 (KEEP_RECENT=3)
messages = [
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "t1", "content": "content of a.txt (150 chars)"},  # 索引0 → 压缩
        {"type": "tool_result", "tool_use_id": "t2", "content": "content of b.txt (200 chars)"},  # 索引1 → 压缩
        {"type": "tool_result", "tool_use_id": "t3", "content": "content of c.txt (180 chars)"},  # 索引2 → 压缩
        {"type": "tool_result", "tool_use_id": "t4", "content": "content of d.txt (160 chars)"},  # 索引3 → 保留
        {"type": "tool_result", "tool_use_id": "t5", "content": "content of e.txt (190 chars)"},  # 索引4 → 保留
        {"type": "tool_result", "tool_use_id": "t6", "content": "content of f.txt (170 chars)"},  # 索引5 → 保留
    ]}
]

# micro_compact后 (只压缩超过100字符的结果)
messages = [
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "t1", "content": "[Previous: used read_file]"},  # 压缩
        {"type": "tool_result", "tool_use_id": "t2", "content": "[Previous: used read_file]"},  # 压缩
        {"type": "tool_result", "tool_use_id": "t3", "content": "[Previous: used read_file]"},  # 压缩
        {"type": "tool_result", "tool_use_id": "t4", "content": "content of d.txt (160 chars)"},  # 保留
        {"type": "tool_result", "tool_use_id": "t5", "content": "content of e.txt (190 chars)"},  # 保留
        {"type": "tool_result", "tool_use_id": "t6", "content": "content of f.txt (170 chars)"},  # 保留
    ]}
]
```

**Layer 2 - auto_compact执行过程**：

```python
# Step 1: 估算token数
estimate_tokens(messages) = len(str(messages)) // 4
# 如果超过THRESHOLD(50000)，触发auto_compact

# Step 2: 保存完整记录到磁盘
TRANSCRIPT_DIR.mkdir(exist_ok=True)
transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
with open(transcript_path, "w") as f:
    for msg in messages:
        f.write(json.dumps(msg, default=str) + "\n")
print(f"[transcript saved: {transcript_path}]")

# Step 3: 调用LLM生成摘要
conversation_text = json.dumps(messages, default=str)[:80000]
response = client.messages.create(
    model=MODEL,
    messages=[{"role": "user", "content":
        "Summarize this conversation for continuity. Include: "
        "1) What was accomplished, 2) Current state, 3) Key decisions made. "
        "Be concise but preserve critical details.\n\n" + conversation_text}],
    max_tokens=2000,
)
summary = response.content[0].text

# Step 4: 用摘要替换整个对话历史
messages = [
    {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
    {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
]
```

**compact工具Schema**：

```python
{"name": "compact",
 "description": "Trigger manual conversation compression.",
 "input_schema": {"type": "object", "properties": {"focus": {"type": "string", "description": "What to preserve in the summary"}}}}
```

**三层压缩对比**：

| 层级 | 触发方式 | 压缩粒度 | Token节省 |
|------|----------|----------|----------|
| Layer 1 micro_compact | 每轮自动 | tool_result → 占位符 | ~30-50% |
| Layer 2 auto_compact | 阈值触发 | 全量 → 摘要 | ~80-90% |
| Layer 3 manual compact | Agent调用 | 全量 → 摘要 | ~80-90% |

**数据流转完整示例（多轮对话过程）**：

```
Turn 1:
  User: "Read a.txt"
  messages: [{"role": "user", "content": "Read a.txt"}]
  
  micro_compact检查: 0个tool_result，不压缩
  
  LLM决定调用 read_file(a.txt)
  → 执行tool，生成tool_result: "content of a.txt"
  
  messages: [
    {"role": "user", "content": "Read a.txt"},
    {"role": "assistant", "content": [tool_use(id=t1, name=read_file)]},
    {"role": "user", "content": [tool_result(id=t1, content="content of a.txt")]}
  ]

Turn 2:
  User: "Read b.txt"
  messages: [Turn 1的全部内容]
  
  micro_compact检查: 1个tool_result < 3，不压缩
  
  LLM决定调用 read_file(b.txt)
  → 执行tool，生成tool_result: "content of b.txt"
  
  messages增加: assistant(tool_use:t2) + user(tool_result:t2)

Turn 3:
  User: "Read c.txt"
  messages: [..., t1, t2]
  
  micro_compact检查: 2个tool_result < 3，不压缩
  
  LLM决定调用 read_file(c.txt)
  → 执行tool，生成tool_result: "content of c.txt"
  
  messages增加: assistant(tool_use:t3) + user(tool_result:t3)

Turn 4 (关键转折点):
  User: "Read d.txt"
  messages: [..., t1, t2, t3]
  
  ===== micro_compact执行 (LLM调用之前) =====
  收集tool_results: [t1, t2, t3]  (注意：t4此时还没有！)
  len(tool_results) = 3
  3 <= KEEP_RECENT(3)，不压缩
  ===== micro_compact结束 =====
  
  LLM决定调用 read_file(d.txt)
  → 执行tool，生成tool_result: "content of d.txt"  ← 这是t4!
  
  messages: [..., t1, t2, t3, assistant(tool_use:t4), user(tool_result:t4)]

Turn 5:
  User: "Read e.txt"
  messages: [..., t1, t2, t3, t4]
  
  ===== micro_compact执行 =====
  收集tool_results: [t1, t2, t3, t4]
  len(tool_results) = 4 > 3，执行压缩
  to_clear = tool_results[:-3] = [t1]  → 压缩
  保留 = tool_results[-3:] = [t2, t3, t4]  → 保留
  ===== micro_compact结束 =====
  
  messages: [..., t1被替换为"[Previous: used read_file]", t2完整, t3完整, t4完整, ...]
  
  LLM继续工作...

Turn 10:
  messages中累计10个tool_results: t1-t10
  micro_compact执行:
    to_clear = tool_results[:-3] = [t1, t2, t3, t4, t5, t6, t7]  → 全部压缩
    保留 = tool_results[-3:] = [t8, t9, t10]                      → 保留

Turn 20:
  messages累计20个tool_results
  估算token数 > 50000 (THRESHOLD)
  ===== auto_compact触发 =====
  1. 保存完整messages到 .transcripts/transcript_1699999999.jsonl
  2. 调用LLM生成摘要:
     "Summarize this conversation for continuity. Include: 
      1) What was accomplished, 2) Current state, 3) Key decisions made."
  3. messages被替换为2条消息
  ===== auto_compact结束 =====

Turn 21:
  Agent在干净上下文中继续工作
  如需恢复历史，可读取transcript文件
```

**LLM生成的摘要示例**：

```json
{
  "role": "user",
  "content": "[Conversation compressed. Transcript: .transcripts/transcript_1699999999.jsonl]\n\n## Conversation Summary\n\n**1) What was accomplished:**\n- Read and analyzed 20 Python files across the agents/ directory\n- Identified patterns: all agents follow the same tool_handler pattern\n- Created TodoManager class in s03 with single in_progress constraint\n- Implemented SkillLoader with YAML frontmatter parsing\n\n**2) Current state:**\n- Working on context compression (s06)\n- Most recent files examined: s05_skill_loading.py, s06_context_compact.py\n- Last tool call was reading a configuration file\n\n**3) Key decisions made:**\n- Skills stored in skills/{name}/SKILL.md format\n- Three-layer compression: micro (per-turn), auto (threshold), manual (on-demand)\n- Keep last 3 tool_results in micro_compact to preserve recent context\n\n**Pending tasks:**\n- Continue testing context compression behavior\n- Verify transcript recovery works correctly"
}
```

压缩后的messages变成：

```python
messages = [
    {"role": "user", "content": "[Conversation compressed. Transcript: .transcripts/transcript_1699999999.jsonl]\n\n## Conversation Summary\n\n**1) What was accomplished:**\n..."},
    {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."}
]
```

**transcript恢复示例**：

```python
# 从磁盘恢复完整对话
transcript_path = ".transcripts/transcript_1699999999.jsonl"
with open(transcript_path) as f:
    original_messages = [json.loads(line) for line in f]
# original_messages包含压缩前的完整历史
```

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

尝试任务：

1. `Read every Python file in the agents/ directory one by one` (观察micro_compact如何替换旧结果)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`

**观察点**：
- `micro_compact`执行后，tool_result的内容发生了什么变化
- 当token估计超过50000时，发生了什么
- 压缩后的messages长什么样

**思考点**：
- micro_compact为什么要保留最后KEEP_RECENT个tool_result而不是全部替换？
- 压缩后的摘要丢失了什么信息？如何从transcript恢复？
- 如果一个工具的结果被micro_compact压缩后，Agent需要再次参考它，该怎么办？

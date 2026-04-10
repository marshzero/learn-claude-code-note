## 第五章：Skill Loading - 如何动态加载技能？

### 1. 问题引入

我们希望Agent遵循特定领域的工作流程：Git规范、测试模式、代码审查清单。但如果把所有技能都塞进系统提示词，会造成严重的Token浪费。

假设有10个技能，每个2000 tokens，仅技能描述就消耗20000 tokens，而大多数技能对任何给定任务都是无关的。

**核心矛盾**：Agent需要领域知识来正确工作，但预加载所有知识会导致上下文膨胀。

### 2. 解决方案

#### 2.1 核心设计

采用两层注入机制：

```
System prompt (Layer 1 -- 始终存在):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

当 model 调用 load_skill("git") 时:
+--------------------------------------+
| tool_result (Layer 2 -- 按需加载):    |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

**关键洞察**：Layer 1放技能名称（便宜），Layer 2放技能内容（按需）。

**技能文件结构**：

```
skills/
  pdf/
    SKILL.md       # --- \n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # --- \n name: code-review\n description: Review code\n ---\n ...
```

**SkillLoader**：扫描`skills/*/SKILL.md`文件，用目录名作为技能标识符。

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills_dir = skills_dir
        self.skills = {}
        self._load_all()

    def _load_all(self):
        for f in sorted(self.skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

#### 2.2 示例数据

**SKILL.md格式**（YAML frontmatter + body）：

```markdown
---
name: git
description: Git workflow helpers
tags: version-control, git
---
# Git Workflow

## Branch Naming
- feature/: 新功能
- fix/: 修复
- hotfix/: 紧急修复

## Commit Message Format
<type>: <subject>

Types:
- feat: 新功能
- fix: 修复
- docs: 文档
- style: 格式
- refactor: 重构
- test: 测试
- chore: 构建

## Workflow
1. Create branch from main
2. Make changes with clear commits
3. Push and create PR
4. Review and merge
```

**Layer 1注入（System Prompt）**：

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
  - git: Git workflow helpers
  - test: Testing best practices
  - code-review: Code review checklist
  - pdf: PDF processing"""
```

**load_skill工具Schema**：

```python
{"name": "load_skill",
 "description": "Load specialized knowledge by name.",
 "input_schema": {"type": "object", "properties": {"name": {"type": "string", "description": "Skill name to load"}}, "required": ["name"]}}
```

**load_skill调用示例**：

```json
{
  "type": "tool_use",
  "name": "load_skill",
  "id": "tool_001",
  "input": {"name": "git"}
}
```

**load_skill返回示例**：

```json
{
  "type": "tool_result",
  "tool_use_id": "tool_001",
  "content": "<skill name=\"git\">\n# Git Workflow\n\n## Branch Naming...\n</skill>"
}
```

**两层注入数据流转**：

```
Step 1: 初始化
SkillLoader扫描skills/目录
  → skills/git/SKILL.md
  → skills/test/SKILL.md
  → skills/code-review/SKILL.md
每个skill注册到self.skills字典

Step 2: Agent启动
System Prompt注入Layer 1（技能名称）:
  "Skills available:\n  - git: Git workflow helpers\n  - test: Testing best practices"

Step 3: User请求
User: "I need to commit this fix"

Step 4: Agent决定加载技能
Agent调用 load_skill("git")

Step 5: 技能内容注入
Tool Result返回Layer 2（完整技能内容）:
  "<skill name=\"git\">\n# Git Workflow\n## Branch Naming...\n## Commit Message Format...\n</skill>"

Step 6: Agent使用技能知识
Agent按照git技能中的规范生成commit message
```

**Token开销对比**：

| 方案 | Token消耗 | 说明 |
|------|----------|------|
| 全量预加载 | 10 skills × 2000 = 20000 | 大多数技能对任务无用 |
| 两层注入 | 10 × 100 (names) + 按需 ≈ 2000-4000 | 按需加载，只花需要的 |

### 3. 实战尝试

运行代码：

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

尝试任务：

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`

**观察点**：
- 系统提示词中的技能列表是什么样的
- 调用`load_skill`后，返回的内容是什么格式
- `<skill>`标签如何帮助Agent识别技能内容的边界

**思考点**：
- 为什么技能内容通过`tool_result`返回而不是修改System Prompt？
- 如果一个技能有多种用途（如"test"既可以指单元测试也可以指端到端测试），如何设计技能描述？
- 技能可以被动态卸载吗？什么时候需要卸载？

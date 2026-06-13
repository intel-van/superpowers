# 文档审阅系统实现计划

> **对于 agent 工作者：** 必需：使用 superpowers:subagent-driven-development（如果有子 agent 可用）或 superpowers:executing-plans 来实现此计划。

**目标：** 在 brainstorming 和 writing-plans 技能中添加规范和计划文档审阅循环。

**架构：** 在每个技能目录中创建审阅者提示模板。修改技能文件，在文档创建后添加审阅循环。使用 Task 工具配合通用子 agent 进行审阅者调度。

**技术栈：** Markdown 技能文件，通过 Task 工具调度子 agent

**规范：** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## 块 1：规范文档审阅者

此块向 brainstorming 技能添加规范文档审阅者。

### 任务 1：创建规范文档审阅者提示模板

**文件：**
- 创建：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **步骤 1：** 创建审阅者提示模板文件

```markdown
# 规范文档审阅者提示模板

使用此模板来调度规范文档审阅者子 agent。

**目的：** 验证规范是否完整、一致，并准备好进行实现计划。

**调度时机：** 规范文档写入 docs/superpowers/specs/ 之后

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**审阅者返回：** 状态、问题（如有）、建议
```

- [ ] **步骤 2：** 验证文件创建成功

运行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **步骤 3：** 提交

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### 任务 2：向 Brainstorming 技能添加审阅循环

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **步骤 1：** 读取当前的 brainstorming 技能

运行：`cat skills/brainstorming/SKILL.md`

- [ ] **步骤 2：** 在 "After the Design" 之后添加审阅循环部分

找到 "After the Design" 部分，在文档之后、实现之前添加新的 "Spec Review Loop" 部分：

```markdown
**Spec Review Loop:**
After writing the spec document:
1. Dispatch spec-document-reviewer subagent (see spec-document-reviewer-prompt.md)
2. If ❌ Issues Found:
   - Fix the issues in the spec document
   - Re-dispatch reviewer
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to implementation setup

**Review loop guidance:**
- Same agent that wrote the spec fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect
```

- [ ] **步骤 3：** 验证更改

运行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
预期：显示新的审阅循环部分

- [ ] **步骤 4：** 提交

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## 块 2：计划文档审阅者

此块向 writing-plans 技能添加计划文档审阅者。

### 任务 3：创建计划文档审阅者提示模板

**文件：**
- 创建：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **步骤 1：** 创建审阅者提示模板文件

```markdown
# Plan Document Reviewer Prompt Template

Use this template when dispatching a plan document reviewer subagent.

**Purpose:** Verify the plan chunk is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** Each plan chunk is written

```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**审阅者返回：** 状态、问题（如有）、建议
```

- [ ] **步骤 2：** 验证文件已创建

运行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **步骤 3：** 提交

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### 任务 4：向 Writing-Plans 技能添加审阅循环

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **步骤 1：** 读取当前的技能文件

运行：`cat skills/writing-plans/SKILL.md`

- [ ] **步骤 2：** 添加逐块审阅部分

在 "Execution Handoff" 部分之前添加：

```markdown
## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.
```

- [ ] **步骤 3：** 更新任务语法示例以使用复选框

将 Task Structure 部分更改为显示复选框语法：

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **步骤 4：** 验证审阅循环部分已添加

运行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
预期：显示新的审阅循环部分

- [ ] **步骤 5：** 验证任务语法示例已更新

运行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
预期：显示复选框语法 `### Task N:`

- [ ] **步骤 6：** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## 块 3：更新计划文档头部

此块更新计划文档头部模板以引用新的复选框语法要求。

### 任务 5：更新 Writing-Plans 技能中的计划头部模板

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **步骤 1：** 读取当前计划头部模板

运行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **步骤 2：** 更新头部模板以引用复选框语法

计划头部应注明任务和步骤使用复选框语法。更新头部注释：

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **步骤 3：** 验证更改

运行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
预期：显示更新后的头部，包含复选框语法说明

- [ ] **步骤 4：** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```

---

[← 返回主 README](../../../../README.md) · [English Version](../../../superpowers/plans/2026-01-22-document-review-system.md)

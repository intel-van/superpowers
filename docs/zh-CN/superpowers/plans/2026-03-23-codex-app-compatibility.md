# Codex App 兼容性实现计划

> **对于 agent 工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务实现此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 使 `using-git-worktrees`、`finishing-a-development-branch` 及相关技能能够在 Codex App 的沙箱化 worktree 环境中工作，同时不破坏现有行为。

**架构：** 在两个技能开始时进行只读环境检测（`git-dir` vs `git-common-dir`）。如果已在链接的 worktree 中，跳过创建。如果在分离的 HEAD 上，发出交接负载而非 4 选项菜单。沙箱回退捕获 worktree 创建期间的权限错误。

**技术栈：** Git、Markdown（技能文件是指令文档，而非可执行代码）

**规范：** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 文件结构

| 文件 | 职责 | 操作 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | Worktree 创建 + 隔离 | 添加步骤 0 检测 + 沙箱回退 |
| `skills/finishing-a-development-branch/SKILL.md` | 分支完成工作流 | 添加步骤 1.5 检测 + 清理守卫 |
| `skills/subagent-driven-development/SKILL.md` | 使用子 agent 执行计划 | 更新集成描述 |
| `skills/executing-plans/SKILL.md` | 内联执行计划 | 更新集成描述 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 平台参考 | 添加检测 + 完成文档 |

---

### 任务 1：向 `using-git-worktrees` 添加步骤 0

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:14-15`（在概述之后，目录选择流程之前插入）

- [ ] **步骤 1：读取当前技能文件**

完整读取 `skills/using-git-worktrees/SKILL.md`。确定确切的插入点：在 "Announce at start" 行（第 14 行）之后和 "## Directory Selection Process"（第 16 行）之前。

- [ ] **步骤 2：插入步骤 0 部分**

在概述部分和 "## Directory Selection Process" 之间插入以下内容：

```markdown
## Step 0: Check if Already in an Isolated Workspace

Before creating a worktree, check if one already exists:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**If `GIT_DIR` differs from `GIT_COMMON`:** You are already inside a linked worktree (created by the Codex App, Claude Code's Agent tool, a previous skill run, or the user). Do NOT create another worktree. Instead:

1. Run project setup (auto-detect package manager as in "Run Project Setup" below)
2. Verify clean baseline (run tests as in "Verify Clean Baseline" below)
3. Report with branch state:
   - On a branch: "Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD: "Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

After reporting, STOP. Do not continue to Directory Selection or Creation Steps.

**If `GIT_DIR` equals `GIT_COMMON`:** Proceed with the full worktree creation flow below.

**Sandbox fallback:** If you proceed to Creation Steps but `git worktree add -b` fails with a permission error (e.g., "Operation not permitted"), treat this as a late-detected restricted environment. Fall back to the behavior above — run setup and baseline tests in the current directory, report accordingly, and STOP.
```

- [ ] **步骤 3：验证插入**

再次读取文件。确认：
- 步骤 0 出现在概述和目录选择流程之间
- 文件的其余部分（目录选择、安全验证、创建步骤等）保持不变
- 没有重复部分或损坏的 markdown

- [ ] **步骤 4：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection (PRI-823)

Skip worktree creation when already in a linked worktree. Includes
sandbox fallback for permission errors on git worktree add."
```

---

### 任务 2：更新 `using-git-worktrees` 集成部分

**文件：**
- 修改：`skills/using-git-worktrees/SKILL.md:211-215`（集成 > Called by）

- [ ] **步骤 1：更新三个 "Called by" 条目**

将第 212-214 行从：

```markdown
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

改为：

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 2：验证集成部分**

读取集成部分。确认所有三个条目已更新，"Pairs with" 保持不变。

- [ ] **步骤 3：提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions (PRI-823)

Clarify that skill ensures a workspace exists, not that it always creates one."
```

---

### 任务 3：向 `finishing-a-development-branch` 添加步骤 1.5

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md:38`（在步骤 1 之后，步骤 2 之前插入）

- [ ] **步骤 1：读取当前技能文件**

完整读取 `skills/finishing-a-development-branch/SKILL.md`。确定插入点：在 "**If tests pass:** Continue to Step 2."（第 38 行）之后和 "### Step 2: Determine Base Branch"（第 40 行）之前。

- [ ] **步骤 2：插入步骤 1.5 部分**

在步骤 1 和步骤 2 之间插入以下内容：

```markdown
### Step 1.5: Detect Environment

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Path A — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` is empty (externally managed worktree, detached HEAD):**

First, ensure all work is staged and committed (`git add` + `git commit`).

Then present this to the user (do NOT present the 4-option menu):

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

Branch name: use ticket ID if available (e.g., `pri-823/codex-compat`), otherwise slugify the first 5 words of the plan title, otherwise omit. Avoid sensitive content in branch names.

Skip to Step 5 (cleanup is a no-op — see guard below).

**Path B — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` exists (externally managed worktree, named branch):**

Proceed to Step 2 and present the 4-option menu as normal.

**Path C — `GIT_DIR` equals `GIT_COMMON` (normal environment):**

Proceed to Step 2 and present the 4-option menu as normal.
```

- [ ] **步骤 3：验证插入**

再次读取文件。确认：
- 步骤 1.5 出现在步骤 1 和步骤 2 之间
- 步骤 2-5 保持不变
- 路径 A 交接包含提交 SHA 和数据丢失警告
- 路径 B 和 C 正常进入步骤 2

- [ ] **步骤 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection (PRI-823)

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu. Includes commit SHA and data loss warning."
```

---

### 任务 4：向 `finishing-a-development-branch` 添加步骤 5 清理守卫

**文件：**
- 修改：`skills/finishing-a-development-branch/SKILL.md`（步骤 5：清理 Worktree — 按部分标题查找，行号在任务 3 后已偏移）

- [ ] **步骤 1：读取当前步骤 5 部分**

找到 `skills/finishing-a-development-branch/SKILL.md` 中的 "### Step 5: Cleanup Worktree" 部分（行号在任务 3 插入后已偏移）。当前步骤 5 为：

```markdown
### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

- [ ] **步骤 2：在现有逻辑之前添加清理守卫**

将步骤 5 部分替换为：

```markdown
### Step 5: Cleanup Worktree

**First, check if worktree is externally managed:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

If `GIT_DIR` differs from `GIT_COMMON`: skip worktree removal — the host environment owns this workspace.

**Otherwise, for Options 1 and 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

注意：原文说 "For Options 1, 2, 4" 但快速参考表和常见错误部分说 "Options 1 & 4 only." 此编辑使步骤 5 与这些部分保持一致。

- [ ] **步骤 3：验证替换**

读取步骤 5。确认：
- 清理守卫（重新检测）首先出现
- 对非外部管理的 worktree，保留现有移除逻辑
- "Options 1 and 4"（而非 "1, 2, 4"）与快速参考表和常见错误匹配

- [ ] **步骤 4：提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard (PRI-823)

Re-detect externally managed worktree at cleanup time and skip removal.
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only, matching Quick Reference and Common Mistakes."
```

---

### 任务 5：更新 `subagent-driven-development` 和 `executing-plans` 中的集成行

**文件：**
- 修改：`skills/subagent-driven-development/SKILL.md:268`
- 修改：`skills/executing-plans/SKILL.md:68`

- [ ] **步骤 1：更新 `subagent-driven-development`**

将第 268 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 2：更新 `executing-plans`**

将第 68 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **步骤 3：验证两个文件**

读取 `skills/subagent-driven-development/SKILL.md` 的第 268 行和 `skills/executing-plans/SKILL.md` 的第 68 行。确认两者都说 "Ensures isolated workspace (creates one or verifies existing)"。

- [ ] **步骤 4：提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): update worktree Integration descriptions (PRI-823)

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one."
```

---

### 任务 6：向 `codex-tools.md` 添加环境检测文档

**文件：**
- 修改：`skills/using-superpowers/references/codex-tools.md:25`（在末尾追加）

- [ ] **步骤 1：读取当前文件**

完整读取 `skills/using-superpowers/references/codex-tools.md`。确认它在 multi_agent 部分之后结束于第 25-26 行。

- [ ] **步骤 2：追加两个新部分**

在文件末尾添加：

```markdown

## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.

## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

- [ ] **步骤 3：验证添加内容**

读取完整文件。确认：
- 两个新部分出现在现有内容之后
- Bash 代码块正确渲染（未转义）
- 存在指向步骤 0 和步骤 1.5 的交叉引用

- [ ] **步骤 4：提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs (PRI-823)

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt."
```

---

### 任务 7：自动化测试 — 环境检测

**文件：**
- 创建：`tests/codex-app-compat/test-environment-detection.sh`

- [ ] **步骤 1：创建测试目录**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **步骤 2：编写检测测试脚本**

创建 `tests/codex-app-compat/test-environment-detection.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# Test environment detection logic from PRI-823
# Tests the git-dir vs git-common-dir comparison used by
# using-git-worktrees Step 0 and finishing-a-development-branch Step 1.5

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# Helper: run detection and return "linked" or "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== Test 1: Normal repo detection ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Normal repo detected as normal"
else
  log_fail "Normal repo detected as '$result' (expected 'normal')"
fi

echo "=== Test 2: Linked worktree detection ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree detected as linked"
else
  log_fail "Linked worktree detected as '$result' (expected 'linked')"
fi

echo "=== Test 3: Detached HEAD detection ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch is empty"
else
  log_fail "Detached HEAD: branch is '$branch' (expected empty)"
fi

echo "=== Test 4: Linked worktree + detached HEAD (Codex App simulation) ==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App simulation: linked + detached HEAD"
else
  log_fail "Codex App simulation: result='$result', branch='$branch'"
fi

echo "=== Test 5: Cleanup guard — linked worktree should NOT remove ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Cleanup guard: linked worktree correctly detected (would skip removal)"
else
  log_fail "Cleanup guard: expected 'linked', got '$result'"
fi

echo "=== Test 6: Cleanup guard — main repo SHOULD remove ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Cleanup guard: main repo correctly detected (would proceed with removal)"
else
  log_fail "Cleanup guard: expected 'normal', got '$result'"
fi

# Cleanup worktree before temp dir removal
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== Results: $PASS passed, $FAIL failed ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **步骤 3：使其可执行并运行**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

预期输出：6 通过，0 失败。

- [ ] **步骤 4：提交**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat (PRI-823)

Tests git-dir vs git-common-dir comparison in normal repo, linked
worktree, detached HEAD, and cleanup guard scenarios."
```

---

### 任务 8：最终验证

**文件：**
- 读取：所有 5 个修改的技能文件

- [ ] **步骤 1：运行自动化检测测试**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

预期：6 通过，0 失败。

- [ ] **步骤 2：读取每个修改的文件并验证更改**

端到端读取每个文件：
- `skills/using-git-worktrees/SKILL.md` — 步骤 0 存在，其余不变
- `skills/finishing-a-development-branch/SKILL.md` — 步骤 1.5 存在，清理守卫存在，其余不变
- `skills/subagent-driven-development/SKILL.md` — 第 268 行已更新
- `skills/executing-plans/SKILL.md` — 第 68 行已更新
- `skills/using-superpowers/references/codex-tools.md` — 末尾有两个新部分

- [ ] **步骤 3：验证没有意外更改**

```bash
git diff --stat HEAD~7
```

应显示恰好 6 个文件已更改（5 个技能文件 + 1 个测试文件）。没有其他文件被修改。

- [ ] **步骤 4：运行现有测试套件**

如果测试运行器存在：
```bash
# Run skill-triggering tests
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "Skill triggering tests not available in this environment"

# Run SDD integration test
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "SDD integration test not available in this environment"
```

注意：这些测试需要 Claude Code 配合 `--dangerously-skip-permissions`。如果不可用，记录回归测试应手动运行。

---

[← 返回主 README](../../../../README.md) · [English Version](../../../superpowers/plans/2026-03-23-codex-app-compatibility.md)

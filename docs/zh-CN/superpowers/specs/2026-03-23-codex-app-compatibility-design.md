# Codex App 兼容性：工作树和收尾技能适配

使 superpowers 技能在 Codex App 的沙盒工作树环境中工作，不破坏现有的 Claude Code 或 Codex CLI 行为。

**工单：** PRI-823

## 动机

Codex App 在它管理的工作树内运行代理 — 分离 HEAD，位于 `$CODEX_HOME/worktrees/` 下，带有 Seatbelt 沙盒，阻止 `git checkout -b`、`git push` 和网络访问。三个 superpowers 技能假设无限制的 git 访问：`using-git-worktrees` 创建带有命名分支的手动工作树，`finishing-a-development-branch` 按分支名合并/推送/创建 PR，`subagent-driven-development` 依赖两者。

Codex CLI（开源终端工具）没有这个冲突 — 它没有内置的工作树管理。我们的手动工作树方法填补了那里的隔离空白。问题专门针对 Codex App。

## 实证发现

2026-03-23 在 Codex App 中测试：

| 操作 | workspace-write 沙盒 | Full access 沙盒 |
|---|---|---|
| `git add` | 可用 | 可用 |
| `git commit` | 可用 | 可用 |
| `git checkout -b` | **被阻止**（无法写入 `.git/refs/heads/`） | 可用 |
| `git push` | **被阻止**（网络 + `.git/refs/remotes/`） | 可用 |
| `gh pr create` | **被阻止**（网络） | 可用 |
| `git status/diff/log` | 可用 | 可用 |

额外发现：
- `spawn_agent` 子代理 **共享** 父线程的文件系统（通过标记文件测试确认）
- 无论工作树从哪个分支启动，App 标题栏中都会显示"创建分支"按钮
- App 的原生收尾流程：创建分支 → 提交模态框 → 提交并推送 / 提交并创建 PR
- `network_access = true` 配置在 macOS 上静默失效（问题 #10390）

## 设计：只读环境检测

三个只读 git 命令可用于检测环境，无副作用：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

推导出的两个信号：

- **IN_LINKED_WORKTREE：** `GIT_DIR != GIT_COMMON` — 代理在由其他程序（Codex App、Claude Code Agent 工具、之前的技能运行或用户）创建的工作树中
- **ON_DETACHED_HEAD：** `BRANCH` 为空 — 没有命名分支

为什么用 `git-dir != git-common-dir` 而不是检查 `show-toplevel`：
- 在普通仓库中，两者都解析到同一个 `.git` 目录
- 在链接工作树中，`git-dir` 是 `.git/worktrees/<名称>` 而 `git-common-dir` 是 `.git`
- 在子模块中，两者相等 — 避免 `show-toplevel` 可能产生的误报
- 通过 `cd && pwd -P` 解析处理相对路径问题（`git-common-dir` 在普通仓库中返回相对路径 `.git`，但在工作树中返回绝对路径）和符号链接（macOS 上 `/tmp` → `/private/tmp`）

### 决策矩阵

| 链接工作树？ | 分离 HEAD？ | 环境 | 操作 |
|---|---|---|---|
| 否 | 否 | Claude Code / Codex CLI / 普通 git | 完整技能行为（不变） |
| 是 | 是 | Codex App 工作树（workspace-write） | 跳过工作树创建；在收尾时传递负载 |
| 是 | 否 | Codex App（Full access）或手动工作树 | 跳过工作树创建；完整收尾流程 |
| 否 | 是 | 不常见（手动分离 HEAD） | 正常创建工作树；在收尾时警告 |

## 变更

### 1. `using-git-worktrees/SKILL.md` — 添加步骤 0（约 12 行）

在"概述"和"目录选择过程"之间的新章节：

**步骤 0：检查是否已在隔离工作空间中**

运行检测命令。如果 `GIT_DIR != GIT_COMMON`，完全跳过工作树创建。改为：
1. 跳转到创建步骤下的"运行项目设置" — `npm install` 等是幂等的，运行起来更安全
2. 然后"验证干净基线" — 运行测试
3. 报告分支状态：
   - 在分支上："已在 `<路径>` 上的 `<分支名>` 分支的隔离工作空间中。测试通过。准备实施。"
   - 分离 HEAD："已在 `<路径>` 的隔离工作空间中（分离 HEAD，外部管理）。测试通过。注意：收尾时需要创建分支。准备实施。"

如果 `GIT_DIR == GIT_COMMON`，继续完整的创建工作树流程（不变）。

当步骤 0 触发时跳过安全检查（.gitignore 检查）— 对外部创建的工作树不相关。

更新集成部分的"被调用者"条目。将每个条目的描述从上下文特定文本改为："确保隔离工作空间（创建一个或验证已有的）"。例如，`subagent-driven-development` 条目从"必需：开始前设置隔离工作空间"改为"必需：确保隔离工作空间（创建一个或验证已有的）"。

**沙盒回退：** 如果 `GIT_DIR == GIT_COMMON` 且技能继续执行创建步骤，但 `git worktree add -b` 因权限错误失败（例如，Seatbelt 沙盒拒绝），则将其视为延迟检测到的受限环境。回退到步骤 0 的"已在工作空间中"行为 — 跳过创建，在当前目录中运行设置和基线测试，相应报告。

在步骤 0 报告后停止。不要继续执行目录选择或创建步骤。

**其他所有内容不变：** 目录选择、安全检查、创建步骤、项目设置、基线测试、快速参考、常见错误、红旗。

### 2. `finishing-a-development-branch/SKILL.md` — 添加步骤 1.5 + 清理守护（约 20 行）

**步骤 1.5：检测环境**（在步骤 1"验证测试"之后，步骤 2"确定基础分支"之前）

运行检测命令。三个路径：

- **路径 A** 完全跳过步骤 2 和 3（不需要基础分支或选项）。
- **路径 B 和 C** 照常执行步骤 2（确定基础分支）和步骤 3（展示选项）。

**路径 A — 外部管理工作树 + 分离 HEAD**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 为空）：

首先，确保所有工作已暂存并提交（`git add` + `git commit`）。Codex App 的收尾控制基于已提交的工作。

然后向用户展示以下内容（不展示 4 选项菜单）：

```
实施完成。所有测试通过。
当前 HEAD：<完整提交 SHA>

此工作空间由外部管理（分离 HEAD）。
我无法从此处创建分支、推送或打开 PR。

⚠ 这些提交位于分离 HEAD 上。如果您不创建分支，
它们可能会在此工作空间被清理时丢失。

如果您的宿主应用程序提供这些控制：
- "创建分支" — 命名分支，然后提交/推送/创建 PR
- "移交到本地" — 将更改移至您的本地检出

建议的分支名：<工单 ID/简短描述>
建议的提交信息：<工作总结>
```

分支名推导：如果工单 ID 可用则使用（例如 `pri-823/codex-compat`），否则将计划标题的前 5 个词转换为 slug，如果都不适用则省略建议。避免在分支名中包含敏感内容（漏洞描述、客户名称）。

跳到步骤 5（对于外部管理工作树，清理是无操作的）。

**路径 B — 外部管理工作树 + 命名分支**（`GIT_DIR != GIT_COMMON` 且 `BRANCH` 存在）：

照常展示 4 选项菜单。（步骤 5 的清理守护将独立重新检测外部管理状态。）

**路径 C — 正常环境**（`GIT_DIR == GIT_COMMON`）：

照常展示 4 选项菜单（不变）。

**步骤 5 清理守护：**

在清理时重新运行 `GIT_DIR` 与 `GIT_COMMON` 检测（不依赖之前的技能输出 — 收尾技能可能在不同会话中运行）。如果 `GIT_DIR != GIT_COMMON`，跳过 `git worktree remove` — 宿主环境拥有此工作空间。

否则，照常检查和移除。注意：现有步骤 5 文本说"对于选项 1、2、4"，但快速参考表和常见错误部分说"仅选项 1 和 4"。新的守护在此现有逻辑之前添加，不会改变哪些选项触发清理。

**其他所有内容不变：** 选项 1-4 逻辑、快速参考、常见错误、红旗。

### 3. `subagent-driven-development/SKILL.md` 和 `executing-plans/SKILL.md` — 每处一行编辑

两个技能都有相同的集成部分行。从：
```
- superpowers:using-git-worktrees - 必需：开始前设置隔离工作空间
```
改为：
```
- superpowers:using-git-worktrees - 必需：确保隔离工作空间（创建一个或验证已有的）
```

**其他所有内容不变：** 调度/审查循环、提示模板、模型选择、状态处理、红旗。

### 4. `codex-tools.md` — 添加环境检测文档（约 15 行）

末尾的两个新章节：

**环境检测：**

```markdown
## 环境检测

创建工作树或收尾分支的技能应在继续前使用只读 git 命令检测其环境：

\```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
\```

- `GIT_DIR != GIT_COMMON` → 已在链接工作树中（跳过创建）
- `BRANCH` 为空 → 分离 HEAD（无法从沙盒分支/推送/创建 PR）

参见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch`
步骤 1.5 了解每个技能如何使用这些信号。
```

**Codex App 收尾：**

```markdown
## Codex App 收尾

当沙盒阻止分支/推送操作时（外部管理工作树中的分离 HEAD），代理提交所有工作并告知用户使用 App 的原生控制：

- **"创建分支"** — 命名分支，然后通过 App UI 提交/推送/创建 PR
- **"移交到本地"** — 将工作传输到用户的本地检出

代理仍然可以运行测试、暂存文件，并输出建议的分支名、提交信息和 PR 描述供用户复制。
```

## 不更改的内容

- `implementer-prompt.md`、`spec-reviewer-prompt.md`、`code-quality-reviewer-prompt.md` — 子代理提示不变
- `executing-plans/SKILL.md` — 仅一行集成描述更改（与 `subagent-driven-development` 相同）；所有运行时行为不变
- `dispatching-parallel-agents/SKILL.md` — 无工作树或收尾操作
- `.codex/INSTALL.md` — 安装过程不变
- 4 选项收尾菜单 — 为 Claude Code 和 Codex CLI 完全保留
- 完整工作树创建流程 — 为非工作树环境完全保留
- 子代理调度/审查/迭代循环 — 不变（文件系统共享已确认）

## 范围总结

| 文件 | 变更 |
|---|---|
| `skills/using-git-worktrees/SKILL.md` | +12 行（步骤 0） |
| `skills/finishing-a-development-branch/SKILL.md` | +20 行（步骤 1.5 + 清理守护） |
| `skills/subagent-driven-development/SKILL.md` | 1 行编辑 |
| `skills/executing-plans/SKILL.md` | 1 行编辑 |
| `skills/using-superpowers/references/codex-tools.md` | +15 行 |

5 个文件中新增/更改约 50 行。零新增文件。零破坏性变更。

## 未来考虑

如果第三个技能需要相同的检测模式，将其提取到共享的 `references/environment-detection.md` 文件中（方案 B）。目前不需要 — 仅 2 个技能使用它。

## 测试计划

### 自动化测试（在 Claude Code 中实施后运行）

1. 普通仓库检测 — 断言 IN_LINKED_WORKTREE=false
2. 链接工作树检测 — `git worktree add` 测试工作树，断言 IN_LINKED_WORKTREE=true
3. 分离 HEAD 检测 — `git checkout --detach`，断言 ON_DETACHED_HEAD=true
4. 收尾技能移交输出 — 验证受限环境中的移交消息（非 4 选项菜单）
5. **步骤 5 清理守护** — 创建链接工作树（`git worktree add /tmp/test-cleanup -b test-cleanup`），`cd` 进入，运行步骤 5 清理检测（`GIT_DIR` 与 `GIT_COMMON`），断言它不会调用 `git worktree remove`。然后 `cd` 回主仓库，运行相同检测，断言它会调用 `git worktree remove`。之后清理测试工作树。

### 手动 Codex App 测试（5 个测试）

1. 工作树线程中的检测（workspace-write）— 验证 GIT_DIR != GIT_COMMON，空分支
2. 工作树线程中的检测（Full access）— 相同检测，不同的沙盒行为
3. 收尾技能移交格式 — 验证代理发出移交负载，而非 4 选项菜单
4. 完整生命周期 — 检测 → 提交 → 收尾检测 → 正确行为 → 清理
5. **本地线程中的沙盒回退** — 启动 Codex App 的**本地线程**（workspace-write 沙盒）。提示："使用 superpowers 技能 `using-git-worktrees` 设置隔离工作空间以实施小型变更。" 预检查：`git checkout -b test-sandbox-check` 应失败并返回 `Operation not permitted`。预期：技能检测到 `GIT_DIR == GIT_COMMON`（普通仓库），尝试 `git worktree add -b`，遇到 Seatbelt 拒绝，回退到步骤 0 的"已在工作空间中"行为 — 运行设置、基线测试，从当前目录报告就绪。通过：代理优雅恢复，没有模糊的错误消息。失败：代理打印原始 Seatbelt 错误、重试或给出令人困惑的输出。

### 回归测试

- 现有 Claude Code 技能触发测试仍然通过
- 现有 subagent-driven-development 集成测试仍然通过
- 正常 Claude Code 会话：完整工作树创建 + 4 选项收尾仍然可用

---

[← 返回主 README](../../../../README.md) · [English Version](../../../superpowers/specs/2026-03-23-codex-app-compatibility-design.md)

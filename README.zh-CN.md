[![English](https://img.shields.io/badge/lang-en-blue)](README.md)

# Superpowers（超能力）

Superpowers 是一套完整的软件开发生成方法论，专为你的编码代理（coding agent）打造。它基于一组可组合的技能（skills）和初始指令构建，让代理能自动使用这些技能。

## 快速开始

为你的代理赋予超能力：[Claude Code](#claude-code)、[Codex CLI](#codex-cli)、[Codex App](#codex-app)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[OpenCode](#opencode)、[Cursor](#cursor)、[GitHub Copilot CLI](#github-copilot-cli)。

## 工作原理

从你启动编码代理的那一刻起，它就开始发挥作用。当代理看到你在构建某些东西时，它*不会*直接跳入编码。相反，它会退后一步，询问你真正想要做什么。

一旦它从对话中提炼出需求规格，就会以足够短的段落展示给你，让你能真正阅读和消化。

在你确认设计之后，代理会制定一个实施计划。这个计划清晰到即使是一个充满热情但品味不佳、没有判断力、没有项目背景且厌恶测试的初级工程师也能遵循。它强调真正的红/绿 TDD、YAGNI（你不会需要它）和 DRY。

接下来，一旦你说"开始"，它就会启动一个*子代理驱动开发*流程，让代理处理每个工程任务，检查和审查他们的工作，并持续推进。Claude 经常能够自主工作数小时而不偏离你制定的计划。

还有更多功能，但以上是系统的核心。由于技能会自动触发，你无需做任何特殊操作。你的编码代理只是拥有了超能力。

## 赞助

如果 Superpowers 帮助你完成了能赚钱的事情，并且你愿意的话，我将非常感激你考虑[赞助我的开源工作](https://github.com/sponsors/obra)。

谢谢！

- Jesse

## 安装

安装方式因运行环境而异。如果你使用多个环境，请为每个环境单独安装 Superpowers。

### Claude Code

Superpowers 可通过[官方 Claude 插件市场](https://claude.com/plugins/superpowers)获取

#### 官方市场

- 从 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场为 Claude Code 提供 Superpowers 及其他相关插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从此市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Codex CLI

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Codex App

Superpowers 可通过[官方 Codex 插件市场](https://github.com/openai/plugins)获取。

- 在 Codex App 中，点击侧边栏的 Plugins。
- 你应该会在 Coding 部分看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+` 并按照提示操作。

### Factory Droid

- 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 安装扩展：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 后续更新：

  ```bash
  gemini extensions update superpowers
  ```

### OpenCode

OpenCode 使用自己的插件安装机制；即使已在其��他环境中使用，也需要单独安装。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或在插件市场中搜索 "superpowers"。

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

## 基本工作流

1. **brainstorming（头脑风暴）** - 在编写代码前激活。通过提问完善粗略想法，探索备选方案，分章节展示设计以供验证。保存设计文档。

2. **using-git-worktrees（使用 Git 工作树）** - 设计批准后激活。在新分支上创建隔离工作空间，运行项目设置，验证测试基线。

3. **writing-plans（编写计划）** - 设计批准后激活。将工作分解为小任务（每项 2-5 分钟）。每个任务都包含精确文件路径、完整代码和验证步骤。

4. **subagent-driven-development（子代理驱动开发）或 executing-plans（执行计划）** - 计划就绪后激活。为每个任务分派全新的子代理，进行两阶段审查（先规范合规性，再代码质量），或按批次执行并设有人工检查点。

5. **test-driven-development（测试驱动开发）** - 实施期间激活。强制执行 RED-GREEN-REFACTOR：编写失败测试，观察失败，编写最少代码，观察通过，提交。删除在测试之前编写的代码。

6. **requesting-code-review（请求代码审查）** - 任务之间激活。对照计划审查，按严重程度报告问题。严重问题会阻止后续进度。

7. **finishing-a-development-branch（完成开发分支）** - 任务完成时激活。验证测试，呈现选项（合并/PR/保留/丢弃），清理工作树。

**代理在任何任务之前都会检查相关技能。** 强制工作流，而非建议。

## 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因分析流程（包含根因追踪、纵深防御、条件等待技术）
- **verification-before-completion** - 确保问题真正修复

**协作**
- **brainstorming** - 苏格拉底式设计精炼
- **writing-plans** - 详细实施计划
- **executing-plans** - 批量执行+检查点
- **dispatching-parallel-agents** - 并发子代理工作流
- **requesting-code-review** - 预审查清单
- **receiving-code-review** - 回应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 快速迭代+两阶段审查（先规范合规性，再代码质量）

**元**
- **writing-skills** - 按照最佳实践创建新技能（包含测试方法）
- **using-superpowers** - 技能系统介绍

## 理念

- **测试驱动开发** - 始终先写测试
- **系统化优于临时方案** - 流程胜过猜测
- **降低复杂性** - 简单性是首要目标
- **证据优于断言** - 在宣称成功前验证

阅读[原始发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 深度阅读

- [OpenCode 集成指南](docs/README.opencode.md) — 在 OpenCode.ai 上安装和使用 Superpowers
- [测试指南](docs/testing.md) — 如何测试 Superpowers 技能
- [跨平台 Polyglot Hooks](docs/windows/polyglot-hooks.md) — Windows/macOS/Linux 钩子支持
- [实施计划](docs/zh-CN/plans/) — 过往功能的开发计划
- [设计规格书](docs/zh-CN/superpowers/specs/) — 过往功能的设计文档

### 内部设计文档（英文）

- [Document Review System](docs/superpowers/specs/2026-01-22-document-review-system-design.md)
- [Visual Brainstorming Refactor](docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md)
- [Zero-Dependency Brainstorm Server](docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md)
- [Codex App Compatibility](docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md)
- [Worktree Rototill](docs/superpowers/specs/2026-04-06-worktree-rototill-design.md)

## 贡献

Superpowers 的一般贡献流程如下。请注意，我们通常不接受新的技能贡献，所有对技能的更新必须在所有支持的编码代理上兼容。

1. Fork 仓库
2. 切换到 'dev' 分支
3. 为你的工作创建分支
4. 遵循 `writing-skills` 技能来创建和测试新技能或修改现有技能
5. 提交 PR，务必填写 PR 模板。

完整的指南请参见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新方式因编码代理而异，但通常是自动的。

## 许可证

MIT 许可证 — 详见 LICENSE 文件

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 及 [Prime Radiant](https://primeradiant.com) 团队构建。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz)获取社区支持、提问和分享你用 Superpowers 构建的内容
- **Issues**：https://github.com/obra/superpowers/issues
- **发布通知**：[注册](https://primeradiant.com/superpowers/)获取新版通知

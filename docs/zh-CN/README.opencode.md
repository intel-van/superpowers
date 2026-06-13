# Superpowers for OpenCode

在 [OpenCode.ai](https://opencode.ai) 中使用 Superpowers 的完整指南。

## 安装

将 superpowers 添加到 `opencode.json`（全局或项目级别）的 `plugin` 数组中：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。插件会通过 OpenCode 的插件管理器安装并注册所有技能。

通过提问来验证：「Tell me about your superpowers」

OpenCode 使用自己的插件安装方式。如果你同时还使用 Claude Code、Codex 或其他工具，请分别为每个工具安装 Superpowers。

### 从旧的符号链接安装方式迁移

如果你之前通过 `git clone` 和符号链接安装了 superpowers，请移除旧的配置：

```bash
# Remove old symlinks
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# Optionally remove the cloned repo
rm -rf ~/.config/opencode/superpowers

# Remove skills.paths from opencode.json if you added one for superpowers
```

然后按照上面的安装步骤操作。

## 使用方法

### 查找技能

使用 OpenCode 原生的 `skill` 工具列出所有可用技能：

```
use skill tool to list skills
```

### 加载技能

```
use skill tool to load superpowers/brainstorming
```

### 个人技能

在 `~/.config/opencode/skills/` 下创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目技能

在项目内的 `.opencode/skills/` 下创建项目特定的技能。

**技能优先级：** 项目技能 > 个人技能 > Superpowers 技能

## 更新

OpenCode 通过基于 git 的包规范安装 Superpowers。某些 OpenCode 和 Bun 版本会在锁定文件或缓存中固定已解析的 git 依赖，因此重启后可能不会获取最新的 Superpowers 提交。如果更新未生效，请清除 OpenCode 的包缓存或重新安装插件。

要固定特定版本，请使用分支或标签：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 工作原理

该插件做两件事：

1. 通过 `experimental.chat.system.transform` 钩子**注入引导上下文**，使每次对话都具备 superpowers 感知能力。
2. 通过 `config` 钩子**注册技能目录**，使 OpenCode 无需符号链接或手动配置即可发现所有 superpowers 技能。

### 工具映射

为 Claude Code 编写的技能会自动适配 OpenCode：

- `TodoWrite` → `todowrite`
- `Task` 与子代理 → OpenCode 的 `@mention` 系统
- `Skill` 工具 → OpenCode 原生的 `skill` 工具
- 文件操作 → 原生的 OpenCode 工具

## 故障排除

### 插件未加载

1. 检查 OpenCode 日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 验证 `opencode.json` 中的插件行是否正确
3. 确保你运行的是最新版本的 OpenCode

### Windows 安装问题

某些 Windows 版 OpenCode 在 git 后端插件规范方面存在上游安装程序问题，包括 `git+https` URL 的缓存路径以及 Bun 即使在正常终端中可用的 `git.exe` 也无法找到的问题。如果 OpenCode 无法安装插件，请尝试使用系统 npm 安装并将 OpenCode 指向本地包：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中使用已安装的包路径：

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### 技能未找到

1. 使用 OpenCode 的 `skill` 工具列出可用技能
2. 检查插件是否正在加载（见上文）
3. 每个技能都需要包含有效 YAML 前置元数据的 `SKILL.md` 文件

### 引导未出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.system.transform` 钩子
2. 配置更改后重启 OpenCode

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode 文档：https://opencode.ai/docs/

---

[← 返回主 README](../../README.md) · [English Version](../README.opencode.md)

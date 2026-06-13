# 测试 Superpowers 技能

本文档描述了如何测试 Superpowers 技能，特别是针对诸如 `subagent-driven-development` 等复杂技能的集成测试。

## 概述

测试涉及子代理、工作流和复杂交互的技能，需要在无头模式下运行实际的 Claude Code 会话，并通过会话记录验证其行为。

## 测试结构

```
tests/
├── claude-code/
│   ├── test-helpers.sh                    # 共享测试工具
│   ├── test-subagent-driven-development-integration.sh
│   ├── analyze-token-usage.py             # Token 分析工具
│   └── run-skill-tests.sh                 # 测试运行器（如果存在）
```

## 运行测试

### 集成测试

集成测试使用实际技能运行真实的 Claude Code 会话：

```bash
# 运行 subagent-driven-development 集成测试
cd tests/claude-code
./test-subagent-driven-development-integration.sh
```

**注意：** 集成测试可能需要 10-30 分钟，因为它们会执行包含多个子代理的实际实现计划。

### 环境要求

- 必须从 **superpowers 插件目录** 运行（而非临时目录）
- 必须安装 Claude Code 并可作为 `claude` 命令使用
- 必须启用本地开发市场：`~/.claude/settings.json` 中包含 `"superpowers@superpowers-dev": true`

## 集成测试：subagent-driven-development

### 测试内容

集成测试验证 `subagent-driven-development` 技能是否正确：

1. **计划加载**：在开始时读取一次计划
2. **完整任务文本**：向子代理提供完整的任务描述（不要求它们读取文件）
3. **自我审查**：确保子代理在汇报前进行自我审查
4. **审查顺序**：先进行规范合规审查，再进行代码质量审查
5. **审查循环**：发现问题时使用审查循环
6. **独立验证**：规范审查者独立阅读代码，不信任实现者的报告

### 工作原理

1. **设置**：创建一个临时 Node.js 项目，包含最小实现计划
2. **执行**：在无头模式下使用该技能运行 Claude Code
3. **验证**：解析会话记录（`.jsonl` 文件）以验证：
   - 调用了 Skill 工具
   - 分派了子代理（Task 工具）
   - 使用了 TodoWrite 进行跟踪
   - 创建了实现文件
   - 测试通过
   - Git 提交显示了正确的工作流
4. **Token 分析**：按子代理显示 Token 使用情况

### 测试输出

```
========================================
 Integration Test: subagent-driven-development
========================================

Test project: /tmp/tmp.xyz123

=== Verification Tests ===

Test 1: Skill tool invoked...
  [PASS] subagent-driven-development skill was invoked

Test 2: Subagents dispatched...
  [PASS] 7 subagents dispatched

Test 3: Task tracking...
  [PASS] TodoWrite used 5 time(s)

Test 6: Implementation verification...
  [PASS] src/math.js created
  [PASS] add function exists
  [PASS] multiply function exists
  [PASS] test/math.test.js created
  [PASS] Tests pass

Test 7: Git commit history...
  [PASS] Multiple commits created (3 total)

Test 8: No extra features added...
  [PASS] No extra features added

=========================================
 Token Usage Analysis
=========================================

Usage Breakdown:
----------------------------------------------------------------------------------------------------
Agent           Description                          Msgs      Input     Output      Cache     Cost
----------------------------------------------------------------------------------------------------
main            Main session (coordinator)             34         27      3,996  1,213,703 $   4.09
3380c209        implementing Task 1: Create Add Function     1          2        787     24,989 $   0.09
34b00fde        implementing Task 2: Create Multiply Function     1          4        644     25,114 $   0.09
3801a732        reviewing whether an implementation matches...   1          5        703     25,742 $   0.09
4c142934        doing a final code review...                    1          6        854     25,319 $   0.09
5f017a42        a code reviewer. Review Task 2...               1          6        504     22,949 $   0.08
a6b7fbe4        a code reviewer. Review Task 1...               1          6        515     22,534 $   0.08
f15837c0        reviewing whether an implementation matches...   1          6        416     22,485 $   0.07
----------------------------------------------------------------------------------------------------

TOTALS:
  Total messages:         41
  Input tokens:           62
  Output tokens:          8,419
  Cache creation tokens:  132,742
  Cache read tokens:      1,382,835

  Total input (incl cache): 1,515,639
  Total tokens:             1,524,058

  Estimated cost: $4.67
  (at $3/$15 per M tokens for input/output)

========================================
 Test Summary
========================================

STATUS: PASSED
```

## Token 分析工具

### 使用方法

分析任何 Claude Code 会话的 Token 使用情况：

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

### 查找会话文件

会话记录存储在 `~/.claude/projects/` 中，工作目录路径经过编码：

```bash
# 示例：/Users/yourname/Documents/GitHub/superpowers/superpowers
SESSION_DIR="$HOME/.claude/projects/-Users-yourname-Documents-GitHub-superpowers-superpowers"

# 查找最近的会话
ls -lt "$SESSION_DIR"/*.jsonl | head -5
```

### 显示内容

- **主会话使用情况**：协调者（您或主 Claude 实例）的 Token 使用情况
- **每个子代理的细分**：每次 Task 调用包含：
  - 代理 ID
  - 描述（从提示中提取）
  - 消息数量
  - 输入/输出 Token
  - 缓存使用情况
  - 预估成本
- **总计**：总体 Token 使用情况和成本估算

### 理解输出

- **高缓存读取**：良好——表示提示缓存正常工作
- **主会话输入 Token 高**：正常——协调者拥有完整上下文
- **每个子代理成本相近**：正常——每个子代理获得相似的任务复杂度
- **每项任务成本**：每个子代理的典型范围是 $0.05-$0.15，具体取决于任务

## 故障排除

### 技能未加载

**问题**：运行无头测试时找不到技能

**解决方案**：
1. 确保从 superpowers 目录运行：`cd /path/to/superpowers && tests/...`
2. 检查 `~/.claude/settings.json` 的 `enabledPlugins` 中包含 `"superpowers@superpowers-dev": true`
3. 验证技能是否存在于 `skills/` 目录中

### 权限错误

**问题**：Claude 被阻止写入文件或访问目录

**解决方案**：
1. 使用 `--permission-mode bypassPermissions` 标志
2. 使用 `--add-dir /path/to/temp/dir` 授予对测试目录的访问权限
3. 检查测试目录的文件权限

### 测试超时

**问题**：测试耗时过长并超时

**解决方案**：
1. 增加超时时间：`timeout 1800 claude ...`（30 分钟）
2. 检查技能逻辑中是否存在无限循环
3. 审视子代理任务的复杂度

### 找不到会话文件

**问题**：测试运行后找不到会话记录

**解决方案**：
1. 检查 `~/.claude/projects/` 中正确的项目目录
2. 使用 `find ~/.claude/projects -name "*.jsonl" -mmin -60` 查找最近的会话
3. 确认测试确实运行了（检查测试输出中的错误）

## 编写新的集成测试

### 模板

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# Create test project
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT

# Set up test files...
cd "$TEST_PROJECT"

# Run Claude with skill
PROMPT="Your test prompt here"
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT" \
  --allowed-tools=all \
  --add-dir "$TEST_PROJECT" \
  --permission-mode bypassPermissions \
  2>&1 | tee output.txt

# Find and analyze session
WORKING_DIR_ESCAPED=$(echo "$SCRIPT_DIR/../.." | sed 's/\\//-/g' | sed 's/^-//')
SESSION_DIR="$HOME/.claude/projects/$WORKING_DIR_ESCAPED"
SESSION_FILE=$(find "$SESSION_DIR" -name "*.jsonl" -type f -mmin -60 | sort -r | head -1)

# Verify behavior by parsing session transcript
if grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"; then
    echo "[PASS] Skill was invoked"
fi

# Show token analysis
python3 "$SCRIPT_DIR/analyze-token-usage.py" "$SESSION_FILE"
```

### 最佳实践

1. **始终清理**：使用 trap 清理临时目录
2. **解析记录**：不要 grep 面向用户的输出——应解析 `.jsonl` 会话文件
3. **授予权限**：使用 `--permission-mode bypassPermissions` 和 `--add-dir`
4. **从插件目录运行**：只有从 superpowers 目录运行时技能才会加载
5. **展示 Token 使用情况**：始终包含 Token 分析以了解成本
6. **测试真实行为**：验证实际创建的文件、测试是否通过、提交是否完成

## 会话记录格式

会话记录是 JSON Lines (JSONL) 文件，每行是一个 JSON 对象，表示一条消息或工具结果。

### 关键字段

```json
{
  "type": "assistant",
  "message": {
    "content": [...],
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703
    }
  }
}
```

### 工具结果

```json
{
  "type": "user",
  "toolUseResult": {
    "agentId": "3380c209",
    "usage": {
      "input_tokens": 2,
      "output_tokens": 787,
      "cache_read_input_tokens": 24989
    },
    "prompt": "You are implementing Task 1...",
    "content": [{"type": "text", "text": "..."}]
  }
}
```

`agentId` 字段链接到子代理会话，`usage` 字段包含该特定子代理调用的 Token 使用情况。

---

[← 返回主 README](../../README.md) · [English Version](../testing.md)

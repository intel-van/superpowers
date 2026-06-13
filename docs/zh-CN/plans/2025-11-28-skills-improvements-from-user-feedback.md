# 基于用户反馈的技能改进

**日期：** 2025-11-28
**状态：** 草案
**来源：** 两个在真实开发场景中使用 superpowers 的 Claude 实例

---

## 执行摘要

两个 Claude 实例提供了来自实际开发会话的详细反馈。他们的反馈揭示了当前技能中**系统性的缺陷**，这些问题导致尽管遵循了技能，仍出现了本可避免的 bug。

**关键洞察：** 这些是问题报告，而不仅仅是解决方案建议。问题是真实的；解决方案需要仔细评估。

**核心主题：**
1. **验证缺口** - 我们验证操作成功，但未验证它们是否达到了预期结果
2. **进程卫生** - 后台进程积累并在子代理之间相互干扰
3. **上下文优化** - 子代理获得了太多无关信息
4. **缺乏自我反思** - 没有提示在交接前批判自己的成果
5. **模拟安全性** - 模拟可能在没有检测的情况下偏离接口
6. **技能激活** - 技能存在但没有被读取/使用

---

## 发现的问题

### 问题 1：配置变更验证缺口

**发生了什么：**
- 子代理测试了"OpenAI 集成"
- 设置了 `OPENAI_API_KEY` 环境变量
- 收到了状态码 200 的响应
- 报告"OpenAI 集成正常工作"
- **但是**响应中包含 `"model": "claude-sonnet-4-20250514"` — 实际使用的是 Anthropic

**根本原因：**
`verification-before-completion` 检查操作是否成功，但未检查结果是否反映了预期的配置变更。

**影响：** 高 — 对集成测试产生错误信心，bug 流入生产环境

**典型失败模式：**
- 切换 LLM 提供商 → 验证状态码 200 但不检查模型名称
- 启用功能开关 → 验证无错误但不检查功能是否激活
- 更改环境 → 验证部署成功但不检查环境变量

---

### 问题 2：后台进程积累

**发生了什么：**
- 会话期间分派了多个子代理
- 每个子代理都启动了后台服务进程
- 进程积累（4 个以上服务器在运行）
- 过期进程仍然绑定到端口
- 后来的端到端测试命中了配置错误的过期服务器
- 产生混淆/错误的测试结果

**根本原因：**
子代理是无状态的——不知道之前子代理的进程。没有清理协议。

**影响：** 中高 — 测试命中错误的服务器，假通过/假失败，调试困惑

---

### 问题 3：子代理提示中的上下文膨胀

**发生了什么：**
- 标准方法：让子代理阅读完整计划文件
- 实验：仅提供任务 + 模式 + 文件 + 验证命令
- 结果：更快、更专注，单次尝试完成更常见

**根本原因：**
子代理在无关的计划章节上浪费 token 和注意力。

**影响：** 中 — 执行速度变慢，失败次数增加

**有效的方法：**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 问题 4：交接前缺乏自我反思

**发生了什么：**
- 添加了自我反思提示："用全新的眼光审视你的工作——有什么可以改进的？"
- 任务 5 的实施者发现测试失败是由于实现 bug 而非测试 bug
- 追踪到第 99 行：`strings.Join(metadata.Entrypoint, " ")` 产生了无效的 Docker 语法
- 如果没有自我反思，只会报告"测试失败"而不说明根本原因

**根本原因：**
实施者不会自然地退后一步批判自己的工作

**影响：** 中 — 实施者本可发现的 bug 被移交给审查者

---

### 问题 5：模拟接口漂移

**发生了什么：**
```typescript
// 接口定义了 close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// 代码（有 bug）调用了 cleanup()
await adapter.cleanup();

// 模拟（匹配了 bug）定义了 cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // 错误！
  })),
}));
```
- 测试通过了
- 运行时崩溃："adapter.cleanup is not a function"

**根本原因：**
模拟是从有 bug 的代码调用的内容推导出来的，而不是从接口定义推导的。TypeScript 无法捕获带有错误方法名的内联模拟。

**影响：** 高 — 测试提供错误信心，运行时崩溃

**为什么 testing-anti-patterns 没有阻止这个问题：**
该技能涵盖了测试模拟行为和在缺乏理解的情况下进行模拟，但没有涵盖"从接口而非实现推导模拟"这个特定模式。

---

### 问题 6：代码审查者文件访问

**发生了什么：**
- 分派了代码审查子代理
- 找不到测试文件："该文件在仓库中似乎不存在"
- 文件实际存在
- 审查者不知道需要先显式读取文件

**根本原因：**
审查者提示中不包含显式的文件读取指令。

**影响：** 低中 — 审查失败或不完整

---

### 问题 7：修复工作流延迟

**发生了什么：**
- 实施者在自我反思期间发现了 bug
- 实施者知道如何修复
- 当前工作流：报告 → 我分派修复者 → 修复者修复 → 我验证
- 额外的往返增加了延迟而没有增加价值

**根本原因：**
当实施者已经诊断出问题时，实施者和修复者角色之间的僵化分离。

**影响：** 低 — 延迟，但没有正确性问题

---

### 问题 8：技能未被读取

**发生了什么：**
- `testing-anti-patterns` 技能存在
- 人类和子代理在编写测试之前都没有读取它
- 本可以防止一些问题（尽管不是全部——见问题 5）

**根本原因：**
没有强制要求子代理读取相关技能。没有提示包含技能读取。

**影响：** 中 — 技能投入如果未被使用则被浪费

---

## 提议的改进

### 1. verification-before-completion：添加配置变更验证

**添加新章节：**

```markdown
## 验证配置变更

当测试配置、提供商、功能开关或环境的变更时：

**不要只验证操作是否成功。验证输出是否反映了预期的变更。**

### 常见失败模式

操作成功是因为*某个*有效配置存在，但并不是你打算测试的那个配置。

### 示例

| 变更 | 不充分 | 必要 |
|--------|-------------|----------|
| 切换 LLM 提供商 | 状态码 200 | 响应包含预期的模型名称 |
| 启用功能开关 | 无错误 | 功能行为实际激活 |
| 更改环境 | 部署成功 | 日志/变量引用新环境 |
| 设置凭据 | 认证成功 | 认证的用户/上下文正确 |

### 门控函数

```
在声称配置变更有效之前：

1. 识别：此次变更后应该有什么不同？
2. 定位：在何处可以观察到这种差异？
   - 响应字段（模型名称、用户 ID）
   - 日志行（环境、提供商）
   - 行为（功能激活/未激活）
3. 运行：显示可观察差异的命令
4. 验证：输出包含预期的差异
5. 然后才能：声称配置变更有效

红旗标志：
  - "请求成功"而未检查内容
  - 检查了状态码但未检查响应体
  - 验证了无错误但未有正面确认
```

**为什么这有效：**
强制验证的是**意图**，而不仅仅是操作成功。

---

### 2. subagent-driven-development：为端到端测试添加进程卫生

**添加新章节：**

```markdown
## 端到端测试的进程卫生

当分派启动服务（服务器、数据库、消息队列）的子代理时：

### 问题

子代理是无状态的——它们不知道之前子代理启动的进程。后台进程会持续存在并干扰后续测试。

### 解决方案

**在分派端到端测试子代理之前，在提示中包含清理指令：**

```
在启动任何服务之前：
1. 终止现有进程：pkill -f "<service-pattern>" 2>/dev/null || true
2. 等待清理：sleep 1
3. 验证端口空闲：lsof -i :<port> && echo "错误：端口仍在使用中" || echo "端口空闲"

测试完成后：
1. 终止你启动的进程
2. 验证清理：pgrep -f "<service-pattern>" || echo "清理成功"
```

### 示例

```
任务：运行 API 服务器的端到端测试

提示中包含：
"在启动服务器之前：
- 终止任何现有服务器：pkill -f 'node.*server.js' 2>/dev/null || true
- 验证端口 3001 空闲：lsof -i :3001 && exit 1 || echo '端口可用'

测试完成后：
- 终止你启动的服务器
- 验证：pgrep -f 'node.*server.js' || echo '清理已验证'"
```

### 为什么这很重要

- 过期进程使用错误配置处理请求
- 端口冲突导致静默失败
- 进程积累拖慢系统
- 混乱的测试结果（命中错误的服务器）
```

**权衡分析：**
- 增加了提示的模板代码
- 但防止了非常令人困惑的调试
- 对于端到端测试子代理是值得的

---

### 3. subagent-driven-development：添加精简上下文选项

**修改步骤 2：使用子代理执行任务**

**修改前：**
```
Read that task carefully from [plan-file].
```

**修改后：**
```
## 上下文方法

**完整计划（默认）：**
在任务复杂或有依赖关系时使用：
```
Read Task N from [plan-file] carefully.
```

**精简上下文（适用于独立任务）：**
在任务独立且基于模式时使用：
```
You are implementing: [1-2句任务描述]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**使用精简上下文的情况：**
- 任务遵循现有模式（添加类似的测试，实现类似的功能）
- 任务是自包含的（不需要来自其他任务的上下文）
- 模式引用足够充分（例如，"follow TestE2E_FeatureOptionValidation"）

**使用完整计划的情况：**
- 任务依赖于其他任务
- 需要理解整体架构
- 需要上下文的复杂逻辑
```

**示例：**
```
精简上下文提示：

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**为什么这有效：**
减少 token 使用，提高专注度，在适当时加快完成速度。

---

### 4. subagent-driven-development：添加自我反思步骤

**修改步骤 2：使用子代理执行任务**

**添加到提示模板：**

```
完成后，在报告之前：

退后一步，用全新的眼光审视你的工作。

问自己：
- 这真的解决了指定的任务吗？
- 有没有我没想到的边缘情况？
- 我是否正确遵循了模式？
- 如果测试失败，根本原因是什么（实现 bug 还是测试 bug）？
- 这个实现有什么可以改进的地方？

如果在反思中发现任何问题，立即修复。

然后报告：
- 你实现了什么
- 自我反思发现（如果有）
- 测试结果
- 更改的文件
```

**为什么这有效：**
在交接前捕获实施者自己可以发现的 bug。文档记录案例：通过自我反思发现了 entrypoint bug。

**权衡：**
每个任务增加约 30 秒，但在审查前捕获问题。

---

### 5. requesting-code-review：添加显式文件读取

**修改代码审查者模板：**

**在开头添加：**

```markdown
## 需要审查的文件

在分析之前，读取这些文件：

1. [列出 diff 中更改的具体文件]
2. [被更改引用但未修改的文件]

使用 Read 工具加载每个文件。

如果找不到文件：
- 检查 diff 中的确切路径
- 尝试其他位置
- 报告："无法定位 [路径] - 请验证文件是否存在"

在读取实际代码之前，不要进行审查。
```

**为什么这有效：**
显式指令防止"找不到文件"的问题。

---

### 6. testing-anti-patterns：添加模拟接口漂移反模式

**添加新的反模式 6：**

```markdown
## 反模式 6：从实现推导的模拟

**违规示例：**
```typescript
// 代码（有 bug）调用了 cleanup()
await adapter.cleanup();

// 模拟（匹配了 bug）有 cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// 接口（正确的）定义了 close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**为什么这是错误的：**
- 模拟将 bug 编码进了测试
- TypeScript 无法捕获带有错误方法名的内联模拟
- 测试通过是因为代码和模拟都是错误的
- 使用真实对象时运行时崩溃

**修复方法：**
```typescript
// ✅ 正确：从接口推导模拟

// 步骤 1：打开接口定义（PlatformAdapter）
// 步骤 2：列出其中定义的方法（close, initialize, 等）
// 步骤 3：精确地模拟这些方法

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // 来自接口！
};

// 现在测试会失败，因为代码调用了不存在的 cleanup()
// 该失败在运行前就揭示了 bug
```

### 门控函数

```
在编写任何模拟之前：

  1. 停下 - 不要查看被测试的代码
  2. 找到：依赖项的接口/类型定义
  3. 读取：接口文件
  4. 列出：接口中定义的方法
  5. 模拟：仅模拟那些方法，使用完全相同的名称
  6. 不要：查看你的代码调用了什么

  如果测试因为代码调用了模拟中不存在的内容而失败：
    ✅ 好 - 测试在你的代码中发现了一个 bug
    修复代码以调用正确的接口方法
    而不是修改模拟

  红旗标志：
    - "我会模拟代码调用的内容"
    - 从实现复制方法名
    - 没有读取接口就编写模拟
    - "测试失败了，所以我把这个方法加到模拟里"
```

**检测方法：**

当你看到运行时错误"X is not a function"而测试通过时：
1. 检查 X 是否被模拟
2. 比较模拟方法到接口方法
3. 查找方法名不匹配
```

**为什么这有效：**
直接针对反馈中的失败模式。

---

### 7. subagent-driven-development：要求测试子代理读取技能

**当任务涉及测试时，添加到提示模板：**

```markdown
在编写任何测试之前：

1. 读取 testing-anti-patterns 技能：
   使用 Skill 工具：superpowers:testing-anti-patterns

2. 在以下情况下应用该技能的门控函数：
   - 编写模拟
   - 向生产类添加方法
   - 模拟依赖项

这不是可选的。违反反模式的测试将在审查中被拒绝。
```

**为什么这有效：**
确保技能被实际使用，而不仅仅是存在。

**权衡：**
增加了每个任务的时间，但防止了整类 bug。

---

### 8. subagent-driven-development：允许实施者修复自我发现的问题

**修改步骤 2：**

**当前：**
```
Subagent reports back with summary of work.
```

**提议：**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**为什么这有效：**
当实施者已经知道修复方法时减少延迟。文档记录案例：本可以为 entrypoint bug 节省一次往返。

**权衡：**
提示稍微复杂，但端到端更快。

---

## 实施计划

### 阶段 1：高影响、低风险（先做）

1. **verification-before-completion：配置变更验证**
   - 清晰的添加，不改变现有内容
   - 解决高影响问题（测试中的错误信心）
   - 文件：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：模拟接口漂移**
   - 添加新的反模式，不修改现有内容
   - 解决高影响问题（运行时崩溃）
   - 文件：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：显式文件读取**
   - 向模板简单添加
   - 修复具体问题（审查者找不到文件）
   - 文件：`skills/requesting-code-review/SKILL.md`

### 阶段 2：中等更改（仔细测试）

4. **subagent-driven-development：进程卫生**
   - 添加新章节，不改变工作流
   - 解决中高影响（测试可靠性）
   - 文件：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自我反思**
   - 更改提示模板（风险较高）
   - 但有文档证明能捕获 bug
   - 文件：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：技能阅读要求**
   - 增加提示开销
   - 但确保技能被实际使用
   - 文件：`skills/subagent-driven-development/SKILL.md`

### 阶段 3：优化（先验证）

7. **subagent-driven-development：精简上下文选项**
   - 增加复杂性（两种方法）
   - 需要验证是否会导致混淆
   - 文件：`skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development：允许实施者修复**
   - 更改工作流（风险较高）
   - 优化，而非 bug 修复
   - 文件：`skills/subagent-driven-development/SKILL.md`

---

## 开放问题

1. **精简上下文方法：**
   - 是否应该将其作为基于模式的任务的默认方式？
   - 如何决定使用哪种方法？
   - 过于精简是否会遗漏重要上下文的风险？

2. **自我反思：**
   - 是否会显著减慢简单任务？
   - 是否只应适用于复杂任务？
   - 如何防止"反思疲劳"导致其变得机械？

3. **进程卫生：**
   - 这应该放在 subagent-driven-development 还是一个单独的技能中？
   - 是否适用于端到端测试之外的其他工作流？
   - 如何处理进程应该持续存在的情况（开发服务器）？

4. **技能阅读强制：**
   - 是否应该要求所有子代理读取相关技能？
   - 如何保持提示不会过长？
   - 过度文档化和失去重点的风险？

---

## 成功指标

如何知道这些改进有效？

1. **配置验证：**
   - 零次"测试通过但使用了错误配置"的情况
   - Jesse 不说"这实际上并不是你以为在测试的内容"

2. **进程卫生：**
   - 零次"测试命中了错误的服务器"的情况
   - 端到端测试运行时没有端口冲突错误

3. **模拟接口漂移：**
   - 零次"测试通过但运行时因缺少方法而崩溃"的情况
   - 模拟和接口之间没有方法名不匹配

4. **自我反思：**
   - 可衡量：实施者报告是否包含自我反思发现？
   - 定性：是否有更少的 bug 进入代码审查阶段？

5. **技能阅读：**
   - 子代理报告引用技能门控函数
   - 代码审查中的反模式违规更少

---

## 风险与缓解

### 风险：提示膨胀
**问题：** 添加所有这些要求使提示变得压倒性
**缓解：**
- 分阶段实施（不要一次性添加所有内容）
- 使某些添加成为条件性的（仅端到端测试的卫生）
- 考虑不同任务类型的模板

### 风险：分析瘫痪
**问题：** 过多的反思/验证会拖慢执行
**缓解：**
- 保持门控函数快速（秒级，而非分钟级）
- 使精简上下文初始为选择加入
- 监控任务完成时间

### 风险：虚假安全感
**问题：** 遵循清单并不能保证正确性
**缓解：**
- 强调门控函数是最低要求，而非最高要求
- 在技能中保留"运用判断力"的语言
- 文档说明技能捕获常见失败，而非所有失败

### 风险：技能分歧
**问题：** 不同技能给出矛盾的建议
**缓解：**
- 检查所有技能更改的一致性
- 文档说明技能如何交互（集成部分）
- 在部署前使用实际场景测试

---

## 建议

**立即执行第一阶段：**
- verification-before-completion：配置变更验证
- testing-anti-patterns：模拟接口漂移
- requesting-code-review：显式文件读取

**在最终确定之前与 Jesse 测试第二阶段：**
- 获取关于自我反思影响的反馈
- 验证进程卫生方法
- 确认技能阅读要求值得开销

**暂缓第三阶段，等待验证：**
- 精简上下文需要实际测试
- 实施者修复工作流更改需要仔细评估

这些更改解决了用户记录的实际问题，同时最小化了使技能变差的风险。

---

[← 返回主 README](../../../README.md) · [English Version](../../plans/2025-11-28-skills-improvements-from-user-feedback.md)

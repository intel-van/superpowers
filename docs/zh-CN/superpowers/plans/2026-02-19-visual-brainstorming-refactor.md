# 可视化 Brainstorming 重构实现计划

> **对于 agent 工作者：** 必需：使用 superpowers:subagent-driven-development（如果有子 agent 可用）或 superpowers:executing-plans 来实现此计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将可视化 brainstorming 从阻塞式 TUI 反馈模型重构为非阻塞式"浏览器显示，终端命令"架构。

**架构：** 浏览器成为交互式显示器；终端保持为对话通道。服务器将用户事件写入每个屏幕的 `.events` 文件，Claude 在下一轮读取该文件。消除 `wait-for-feedback.sh` 和所有 `TaskOutput` 阻塞。

**技术栈：** Node.js (Express, ws, chokidar)，vanilla HTML/CSS/JS

**规范：** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 文件映射

| 文件 | 操作 | 职责 |
|------|--------|---------------|
| `lib/brainstorm-server/index.js` | 修改 | 服务器：添加 `.events` 文件写入，新屏幕时清除，替换 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | 修改 | 模板：移除反馈底部栏，添加内容占位符 + 选择指示器 |
| `lib/brainstorm-server/helper.js` | 修改 | 客户端 JS：移除发送/反馈函数，缩小为点击捕获 + 指示器更新 |
| `lib/brainstorm-server/wait-for-feedback.sh` | 删除 | 不再需要 |
| `skills/brainstorming/visual-companion.md` | 修改 | 技能指令：将循环重写为非阻塞流程 |
| `tests/brainstorm-server/server.test.js` | 修改 | 测试：更新以适配新模板结构和 helper.js API |

---

## 块 1：服务器、模板、客户端、测试、技能

### 任务 1：更新 `frame-template.html`

**文件：**
- 修改：`lib/brainstorm-server/frame-template.html`

- [ ] **步骤 1：移除反馈底部栏 HTML**

将 feedback-footer div（第 227-233 行）替换为选择指示器栏：

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above, then return to the terminal</span>
  </div>
```

同时将 `#claude-content` 内的默认内容（第 220-223 行）替换为内容占位符：

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **步骤 2：将反馈底部栏 CSS 替换为指示器栏 CSS**

移除 `.feedback-footer`、`.feedback-footer label`、`.feedback-row` 以及 `.feedback-footer` 内的 textarea/button 样式（第 82-112 行）。

添加指示器栏 CSS：

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **步骤 3：验证模板渲染**

运行测试套件以检查模板是否仍能加载：
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：测试 1-5 仍应通过。测试 6-8 可能失败（预期——它们断言旧结构）。

- [ ] **步骤 4：提交**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### 任务 2：更新 `index.js` — 内容注入和 `.events` 文件

**文件：**
- 修改：`lib/brainstorm-server/index.js`

- [ ] **步骤 1：为 `.events` 文件写入编写失败测试**

在 `tests/brainstorm-server/server.test.js` 的 Test 4 区域之后添加一个新测试，该测试发送带有 `choice` 字段的 WebSocket 事件并验证 `.events` 文件被写入：

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **步骤 2：运行测试以验证其失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 — `.events` 文件尚不存在。

- [ ] **步骤 3：为 `.events` 文件在新屏幕时清除编写失败测试**

添加另一个测试：

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **步骤 4：运行测试以验证其失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试失败 — 推送屏幕时 `.events` 未清除。

- [ ] **步骤 5：在 `index.js` 中实现 `.events` 文件写入**

在 WebSocket `message` 处理器中（`index.js` 第 74-77 行），在 `console.log` 之后添加：

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

在 chokidar `add` 处理器中（第 104-111 行），添加 `.events` 清除：

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **步骤 6：将 `wrapInFrame` 替换为注释占位符注入**

替换 `wrapInFrame` 函数（`index.js` 第 27-32 行）：

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **步骤 7：运行所有测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新的 `.events` 测试通过。现有测试可能仍因旧断言而失败（在任务 4 中修复）。

- [ ] **步骤 8：提交**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### 任务 3：简化 `helper.js`

**文件：**
- 修改：`lib/brainstorm-server/helper.js`

- [ ] **步骤 1：移除 `sendToClaude` 函数**

删除 `sendToClaude` 函数（第 92-106 行）— 函数体和页面接管 HTML。

- [ ] **步骤 2：移除 `window.send` 函数**

删除 `window.send` 函数（第 120-129 行）— 已绑定到已移除的发送按钮。

- [ ] **步骤 3：移除表单提交和输入变化处理器**

删除表单提交处理器（第 57-71 行）和输入变化处理器（第 73-89 行），包括 `inputTimeout` 变量。

- [ ] **步骤 4：移除 `pageshow` 事件监听器**

删除之前添加的 `pageshow` 监听器（不再需要清除 textarea）。

- [ ] **步骤 5：将点击处理器缩小为仅 `[data-choice]`**

将点击处理器（第 36-55 行）替换为更窄的版本：

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **步骤 6：在选择点击时添加指示器栏更新**

在点击处理器的 `sendEvent` 调用之后添加：

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **步骤 7：从 `window.brainstorm` API 中移除 `sendToClaude`**

更新 `window.brainstorm` 对象（第 132-136 行）以移除 `sendToClaude`：

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **步骤 8：运行测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **步骤 9：提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### 任务 4：更新测试以适配新结构

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`

**注意：** 以下行号引用自_原始_文件。任务 2 在文件前面插入了新测试，因此实际行号会偏移。通过它们的 `console.log` 标签（例如 "Test 5:"、"Test 6:"）查找测试。

- [ ] **步骤 1：更新测试 5（完整文档断言）**

找到测试 5 的断言 `!fullRes.body.includes('feedback-footer')`。将其改为：完整文档也不应有指示器栏（它们按原样提供）：

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **步骤 2：更新测试 6（片段包装）**

第 125 行：将 `feedback-footer` 断言替换为指示器栏断言：

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

同时验证内容占位符已被替换（片段内容出现，占位符注释不出现）：

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **步骤 3：更新测试 7（helper.js API）**

第 140-142 行：更新断言以反映新的 API 表面：

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **步骤 4：将测试 8（sendToClaude 主题）替换为指示器栏测试**

替换测试 8（第 145-149 行）— `sendToClaude` 不再存在。改为测试指示器栏：

```javascript
    // Test 8: Indicator bar uses CSS variables (theme support)
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **步骤 5：运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **步骤 6：提交**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### 任务 5：删除 `wait-for-feedback.sh`

**文件：**
- 删除：`lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **步骤 1：验证没有其他文件导入或引用 `wait-for-feedback.sh`**

搜索代码库：
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

预期引用：仅 `visual-companion.md`（在任务 6 中重写）和可能的发布说明（历史性，保留原样）。

- [ ] **步骤 2：删除文件**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **步骤 3：运行测试以确认没有破坏任何内容**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过（没有测试引用此文件）。

- [ ] **步骤 4：提交**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### 任务 6：重写 `visual-companion.md`

**文件：**
- 修改：`skills/brainstorming/visual-companion.md`

- [ ] **步骤 1：更新"工作原理"描述（第 18 行）**

将关于"以 JSON 格式"接收反馈的句子替换为：

```markdown
The server watches a directory for HTML files and serves the newest one to the browser. You write HTML content, the user sees it in their browser and can click to select options. Selections are recorded to a `.events` file that you read on your next turn.
```

- [ ] **步骤 2：更新片段描述（第 20 行）**

从框架模板提供的描述中移除"feedback footer"：

```markdown
**Content fragments vs full documents:** If your HTML file starts with `<!DOCTYPE` or `<html`, the server serves it as-is (just injects the helper script). Otherwise, the server automatically wraps your content in the frame template — adding the header, CSS theme, selection indicator, and all interactive infrastructure. **Write content fragments by default.** Only write full documents when you need complete control over the page.
```

- [ ] **步骤 3：重写"循环"部分（第 36-61 行）**

将整个"循环"部分替换为：

```markdown
## The Loop

1. **Write HTML** to a new file in `screen_dir`:
   - Use semantic filenames: `platform.html`, `visual-style.html`, `layout.html`
   - **Never reuse filenames** — each screen gets a fresh file
   - Use Write tool — **never use cat/heredoc** (dumps noise into terminal)
   - Server automatically serves the newest file

2. **Tell user what to expect and end your turn:**
   - Remind them of the URL (every step, not just first)
   - Give a brief text summary of what's on screen (e.g., "Showing 3 layout options for the homepage")
   - Ask them to respond in the terminal: "Take a look and let me know what you think. Click to select an option if you'd like."

3. **On your next turn** — after the user responds in the terminal:
   - Read `$SCREEN_DIR/.events` if it exists — this contains the user's browser interactions (clicks, selections) as JSON lines
   - Merge with the user's terminal text to get the full picture
   - The terminal message is the primary feedback; `.events` provides structured interaction data

4. **Iterate or advance** — if feedback changes current screen, write a new file (e.g., `layout-v2.html`). Only move to the next question when the current step is validated.

5. Repeat until done.
```

- [ ] **步骤 4：替换"用户反馈格式"部分（第 165-174 行）**

替换为：

```markdown
## Browser Events Format

When the user clicks options in the browser, their interactions are recorded to `$SCREEN_DIR/.events` (one JSON object per line). The file is cleared automatically when you push a new screen.

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

The full event stream shows the user's exploration path — they may click multiple options before settling. The last `choice` event is typically the final selection, but the pattern of clicks can reveal hesitation or preferences worth asking about.

If `.events` doesn't exist, the user didn't interact with the browser — use only their terminal text.
```

- [ ] **步骤 5：更新"编写内容片段"描述（第 65 行）**

移除"feedback footer"引用：

```markdown
Write just the content that goes inside the page. The server wraps it in the frame template automatically (header, theme CSS, selection indicator, and all interactive infrastructure).
```

- [ ] **步骤 6：更新参考部分（第 200-203 行）**

移除 helper.js 参考描述中的"JS API" — API 现在已最小化。保留路径引用：

```markdown
## Reference

- Frame template (CSS reference): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script (client-side): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **步骤 7：提交**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### 任务 7：最终验证

- [ ] **步骤 1：运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试通过。

- [ ] **步骤 2：手动冒烟测试**

手动启动服务器并验证流程端到端工作：

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

编写测试片段，在浏览器中打开，点击选项，验证 `.events` 文件被写入，验证指示器栏更新。然后停止服务器：

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **步骤 3：验证没有残留引用**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

预期：除发布说明和规范/计划文档（历史性）外无匹配。

- [ ] **步骤 4：如有清理需要，进行最终提交**

```bash
git status
# Review untracked/modified files, stage specific files as needed, commit if clean
```

---

[← 返回主 README](../../../../README.md) · [English Version](../../../superpowers/plans/2026-02-19-visual-brainstorming-refactor.md)

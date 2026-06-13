# 可视化头脑风暴伴侣实施计划

> **给代理工作者的提示：** 必需子技能：使用 superpowers:executing-plans 逐个任务地实施此计划。

**目标：** 为 Claude 提供一个基于浏览器的可视化头脑风暴伴侣——在终端对话旁展示原型、模拟和交互选择。

**架构：** Claude 将 HTML 写入临时文件。本地 Node.js 服务器监视该文件并提供带有自动注入辅助库的服务。用户交互通过 WebSocket 流向服务器标准输出，Claude 在后台任务输出中看到这些交互。

**技术栈：** Node.js, Express, ws (WebSocket), chokidar (文件监视)

---

## 任务 1：创建服务器基础

**文件：**
- 创建：`lib/brainstorm-server/index.js`
- 创建：`lib/brainstorm-server/package.json`

**步骤 1：创建 package.json**

```json
{
  "name": "brainstorm-server",
  "version": "1.0.0",
  "description": "Visual brainstorming companion server for Claude Code",
  "main": "index.js",
  "dependencies": {
    "chokidar": "^3.5.3",
    "express": "^4.18.2",
    "ws": "^8.14.2"
  }
}
```

**步骤 2：创建可启动的最小服务器**

```javascript
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const chokidar = require('chokidar');
const fs = require('fs');
const path = require('path');

const PORT = process.env.BRAINSTORM_PORT || 3333;
const SCREEN_FILE = process.env.BRAINSTORM_SCREEN || '/tmp/brainstorm/screen.html';
const SCREEN_DIR = path.dirname(SCREEN_FILE);

// 确保屏幕目录存在
if (!fs.existsSync(SCREEN_DIR)) {
  fs.mkdirSync(SCREEN_DIR, { recursive: true });
}

// 如果不存在则创建默认屏幕
if (!fs.existsSync(SCREEN_FILE)) {
  fs.writeFileSync(SCREEN_FILE, `<!DOCTYPE html>
<html>
<head>
  <title>Brainstorm Companion</title>
  <style>
    body { font-family: system-ui, sans-serif; padding: 2rem; max-width: 800px; margin: 0 auto; }
    h1 { color: #333; }
    p { color: #666; }
  </style>
</head>
<body>
  <h1>Brainstorm Companion</h1>
  <p>Waiting for Claude to push a screen...</p>
</body>
</html>`);
}

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// 跟踪连接的浏览器以发送重新加载通知
const clients = new Set();

wss.on('connection', (ws) => {
  clients.add(ws);
  ws.on('close', () => clients.delete(ws));

  ws.on('message', (data) => {
    // 用户交互事件 - 写入标准输出供 Claude 读取
    const event = JSON.parse(data.toString());
    console.log(JSON.stringify({ type: 'user-event', ...event }));
  });
});

// 提供当前屏幕内容并注入 helper.js
app.get('/', (req, res) => {
  let html = fs.readFileSync(SCREEN_FILE, 'utf-8');

  // 在 </body> 之前注入辅助脚本
  const helperScript = fs.readFileSync(path.join(__dirname, 'helper.js'), 'utf-8');
  const injection = `<script>\n${helperScript}\n</script>`;

  if (html.includes('</body>')) {
    html = html.replace('</body>', `${injection}\n</body>`);
  } else {
    html += injection;
  }

  res.type('html').send(html);
});

// 监视屏幕文件的更改
chokidar.watch(SCREEN_FILE).on('change', () => {
  console.log(JSON.stringify({ type: 'screen-updated', file: SCREEN_FILE }));
  // 通知所有浏览器重新加载
  clients.forEach(ws => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'reload' }));
    }
  });
});

server.listen(PORT, '127.0.0.1', () => {
  console.log(JSON.stringify({ type: 'server-started', port: PORT, url: `http://localhost:${PORT}` }));
});
```

**步骤 3：运行 npm install**

运行：`cd lib/brainstorm-server && npm install`
预期：依赖项已安装

**步骤 4：测试服务器启动**

运行：`cd lib/brainstorm-server && timeout 3 node index.js || true`
预期：看到带有 `server-started` 和端口信息的 JSON

**步骤 5：提交**

```bash
git add lib/brainstorm-server/
git commit -m "feat: add brainstorm server foundation"
```

---

## 任务 2：创建辅助库

**文件：**
- 创建：`lib/brainstorm-server/helper.js`

**步骤 1：创建具有事件自动捕获功能的 helper.js**

```javascript
(function() {
  const WS_URL = 'ws://' + window.location.host;
  let ws = null;
  let eventQueue = [];

  function connect() {
    ws = new WebSocket(WS_URL);

    ws.onopen = () => {
      // 发送任何已排队的事件
      eventQueue.forEach(e => ws.send(JSON.stringify(e)));
      eventQueue = [];
    };

    ws.onmessage = (msg) => {
      const data = JSON.parse(msg.data);
      if (data.type === 'reload') {
        window.location.reload();
      }
    };

    ws.onclose = () => {
      // 1 秒后重新连接
      setTimeout(connect, 1000);
    };
  }

  function send(event) {
    event.timestamp = Date.now();
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(event));
    } else {
      eventQueue.push(event);
    }
  }

  // 自动捕获交互元素上的点击
  document.addEventListener('click', (e) => {
    const target = e.target.closest('button, a, [data-choice], [role="button"], input[type="submit"]');
    if (!target) return;

    // 不捕获常规链接导航
    if (target.tagName === 'A' && !target.dataset.choice) return;

    e.preventDefault();

    send({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice || null,
      id: target.id || null,
      className: target.className || null
    });
  });

  // 自动捕获表单提交
  document.addEventListener('submit', (e) => {
    e.preventDefault();
    const form = e.target;
    const formData = new FormData(form);
    const data = {};
    formData.forEach((value, key) => { data[key] = value; });

    send({
      type: 'submit',
      formId: form.id || null,
      formName: form.name || null,
      data: data
    });
  });

  // 自动捕获输入变更（防抖）
  let inputTimeout = null;
  document.addEventListener('input', (e) => {
    const target = e.target;
    if (!target.matches('input, textarea, select')) return;

    clearTimeout(inputTimeout);
    inputTimeout = setTimeout(() => {
      send({
        type: 'input',
        name: target.name || null,
        id: target.id || null,
        value: target.value,
        inputType: target.type || target.tagName.toLowerCase()
      });
    }, 500); // 500ms 防抖
  });

  // 暴露接口以供显式使用
  window.brainstorm = {
    send: send,
    choice: (value, metadata = {}) => send({ type: 'choice', value, ...metadata })
  };

  connect();
})();
```

**步骤 2：验证 helper.js 语法有效**

运行：`node -c lib/brainstorm-server/helper.js`
预期：无语法错误

**步骤 3：提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "feat: add browser helper library for event capture"
```

---

## 任务 3：为服务器编写测试

**文件：**
- 创建：`tests/brainstorm-server/server.test.js`
- 创建：`tests/brainstorm-server/package.json`

**步骤 1：创建测试 package.json**

```json
{
  "name": "brainstorm-server-tests",
  "version": "1.0.0",
  "scripts": {
    "test": "node server.test.js"
  }
}
```

**步骤 2：编写服务器测试**

```javascript
const { spawn } = require('child_process');
const http = require('http');
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');
const assert = require('assert');

const SERVER_PATH = path.join(__dirname, '../../lib/brainstorm-server/index.js');
const TEST_PORT = 3334;
const TEST_SCREEN = '/tmp/brainstorm-test/screen.html';

// 清理测试目录
function cleanup() {
  if (fs.existsSync(path.dirname(TEST_SCREEN))) {
    fs.rmSync(path.dirname(TEST_SCREEN), { recursive: true });
  }
}

async function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function fetch(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => resolve({ status: res.statusCode, body: data }));
    }).on('error', reject);
  });
}

async function runTests() {
  cleanup();

  // 启动服务器
  const server = spawn('node', [SERVER_PATH], {
    env: { ...process.env, BRAINSTORM_PORT: TEST_PORT, BRAINSTORM_SCREEN: TEST_SCREEN }
  });

  let stdout = '';
  server.stdout.on('data', (data) => { stdout += data.toString(); });
  server.stderr.on('data', (data) => { console.error('Server stderr:', data.toString()); });

  await sleep(1000); // 等待服务器启动

  try {
    // 测试 1：服务器启动并输出 JSON
    console.log('Test 1: Server startup message');
    assert(stdout.includes('server-started'), 'Should output server-started');
    assert(stdout.includes(TEST_PORT.toString()), 'Should include port');
    console.log('  PASS');

    // 测试 2：GET / 返回注入了 helper 的 HTML
    console.log('Test 2: Serves HTML with helper injected');
    const res = await fetch(`http://localhost:${TEST_PORT}/`);
    assert.strictEqual(res.status, 200);
    assert(res.body.includes('brainstorm'), 'Should include brainstorm content');
    assert(res.body.includes('WebSocket'), 'Should have helper.js injected');
    console.log('  PASS');

    // 测试 3：WebSocket 连接和事件转发
    console.log('Test 3: WebSocket relays events to stdout');
    stdout = ''; // 重置 stdout 捕获
    const ws = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws.on('open', resolve));

    ws.send(JSON.stringify({ type: 'click', text: 'Test Button' }));
    await sleep(100);

    assert(stdout.includes('user-event'), 'Should relay user events');
    assert(stdout.includes('Test Button'), 'Should include event data');
    ws.close();
    console.log('  PASS');

    // 测试 4：文件更改触发重新加载通知
    console.log('Test 4: File change notifies browsers');
    const ws2 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws2.on('open', resolve));

    let gotReload = false;
    ws2.on('message', (data) => {
      const msg = JSON.parse(data.toString());
      if (msg.type === 'reload') gotReload = true;
    });

    // 修改屏幕文件
    fs.writeFileSync(TEST_SCREEN, '<html><body>Updated</body></html>');
    await sleep(500);

    assert(gotReload, 'Should send reload message on file change');
    ws2.close();
    console.log('  PASS');

    console.log('\nAll tests passed!');

  } finally {
    server.kill();
    cleanup();
  }
}

runTests().catch(err => {
  console.error('Test failed:', err);
  process.exit(1);
});
```

**步骤 3：运行测试**

运行：`cd tests/brainstorm-server && npm install ws && node server.test.js`
预期：所有测试通过

**步骤 4：提交**

```bash
git add tests/brainstorm-server/
git commit -m "test: add brainstorm server integration tests"
```

---

## 任务 4：将可视化伴侣添加到头脑风暴技能

**文件：**
- 修改：`skills/brainstorming/SKILL.md`
- 创建：`skills/brainstorming/visual-companion.md`（支持文档）

**步骤 1：创建支持文档**

创建 `skills/brainstorming/visual-companion.md`：

```markdown
# 可视化伴侣参考

## 启动服务器

作为后台任务运行：

```bash
node ${PLUGIN_ROOT}/lib/brainstorm-server/index.js
```

告诉用户："我已启动可视化伴侣，访问 http://localhost:3333 - 在浏览器中打开它。"

## 推送屏幕

将 HTML 写入 `/tmp/brainstorm/screen.html`。服务器监视该文件并自动刷新浏览器。

## 读取用户响应

检查后台任务输出中的 JSON 事件：

```json
{"type":"user-event","type":"click","text":"Option A","choice":"optionA","timestamp":1234567890}
{"type":"user-event","type":"submit","data":{"notes":"My feedback"},"timestamp":1234567891}
```

事件类型：
- **click**：用户点击了按钮或 `data-choice` 元素
- **submit**：用户提交了表单（包含所有表单数据）
- **input**：用户在字段中输入（防抖 500ms）

## HTML 模式

### 选项卡片

```html
<div class="options">
  <button data-choice="optionA">
    <h3>Option A</h3>
    <p>Description</p>
  </button>
  <button data-choice="optionB">
    <h3>Option B</h3>
    <p>Description</p>
  </button>
</div>
```

### 交互式原型

```html
<div class="mockup">
  <header data-choice="header">App Header</header>
  <nav data-choice="nav">Navigation</nav>
  <main data-choice="main">Content</main>
</div>
```

### 带备注的表单

```html
<form>
  <label>Priority: <input type="range" name="priority" min="1" max="5"></label>
  <textarea name="notes" placeholder="Additional thoughts..."></textarea>
  <button type="submit">Submit</button>
</form>
```

### 显式 JavaScript

```html
<button onclick="brainstorm.choice('custom', {extra: 'data'})">Custom</button>
```
```

**步骤 2：向头脑风暴技能添加可视化伴侣部分**

在 `skills/brainstorming/SKILL.md` 的"关键原则"之后添加：

```markdown

## 可视化伴侣（可选）

当头脑风暴涉及视觉元素——UI 原型、线框图、交互原型时，使用基于浏览器的可视化伴侣。

**何时使用：**
- 展示受益于视觉比较的 UI/UX 选项
- 显示线框图或布局选项
- 收集结构化反馈（评分、表单）
- 原型化点击交互

**工作原理：**
1. 以后台任务方式启动服务器
2. 告诉用户打开 http://localhost:3333
3. 将 HTML 写入 `/tmp/brainstorm/screen.html`（自动刷新）
4. 检查后台任务输出中的用户交互

终端仍然是主要的对话界面。浏览器是视觉辅助工具。

**参考：** 参见此技能目录中的 `visual-companion.md`，了解 HTML 模式和 API 详情。
```

**步骤 3：验证编辑内容**

运行：`grep -A5 "Visual Companion" skills/brainstorming/SKILL.md`
预期：显示新章节

**步骤 4：提交**

```bash
git add skills/brainstorming/
git commit -m "feat: add visual companion to brainstorming skill"
```

---

## 任务 5：将服务器添加到插件忽略列表（可选清理）

**文件：**
- 检查 `.gitignore` 是否需要为 lib/brainstorm-server 排除 node_modules

**步骤 1：检查当前 gitignore**

运行：`cat .gitignore 2>/dev/null || echo "No .gitignore"`

**步骤 2：如果需要则添加 node_modules**

如果尚未存在，添加：
```
lib/brainstorm-server/node_modules/
```

**步骤 3：如果更改则提交**

```bash
git add .gitignore
git commit -m "chore: ignore brainstorm-server node_modules"
```

---

## 总结

完成所有任务后：

1. **服务器**在 `lib/brainstorm-server/` - 监视 HTML 文件并转发事件的 Node.js 服务器
2. **辅助库**自动注入 - 捕获点击、表单、输入
3. **测试**在 `tests/brainstorm-server/` - 验证服务器行为
4. **头脑风暴技能**已更新，包含可视化伴侣部分和 `visual-companion.md` 参考文档

**使用方法：**
1. 以后台任务启动服务器：`node lib/brainstorm-server/index.js &`
2. 告诉用户打开 `http://localhost:3333`
3. 将 HTML 写入 `/tmp/brainstorm/screen.html`
4. 检查任务输出中的用户事件

---

[← 返回主 README](../../../README.md) · [English Version](../../plans/2026-01-17-visual-brainstorming.md)

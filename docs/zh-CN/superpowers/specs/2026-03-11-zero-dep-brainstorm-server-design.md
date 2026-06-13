# 零依赖头脑风暴服务器

将头脑风暴伴侣服务器的 vender 版 node_modules（express, ws, chokidar — 714 个跟踪文件）替换为仅使用 Node.js 内置模块的零依赖 `server.js`。

## 动机

将 node_modules  vendored 到 git 仓库中带来了供应链风险：冻结的依赖无法获得安全补丁，714 个第三方代码文件未经审计就被提交，对 vendored 代码的修改看起来像普通提交。虽然实际风险较低（仅本地开发服务器），但消除它是直接了当的。

## 架构

单个 `server.js` 文件（约 250-300 行），使用 `http`、`crypto`、`fs` 和 `path`。该文件承担两个角色：

- **直接运行时**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **通过 require 引入时**（`require('./server.js')`）：导出用于单元测试的 WebSocket 协议函数

### WebSocket 协议

仅实现 RFC 6455 的文本帧：

**握手：** 使用 SHA-1 + RFC 6455 魔幻 GUID 从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务器）：** 处理三种掩码长度编码：
- 小：负载 < 126 字节
- 中：126-65535 字节（16 位扩展）
- 大：> 65535 字节（64 位扩展）

使用 4 字节掩码键进行 XOR 解掩码。返回 `{ opcode, payload, bytesConsumed }` 或 `null`（缓冲区不完整时）。拒绝未掩码的帧。

**帧编码（服务器到客户端）：** 未掩码帧，使用相同的三种长度编码。

**处理的操作码：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。无法识别的操作码返回状态码 1003（不支持的数据）的关闭帧。

**有意跳过的：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。对于本地主机客户端之间的小型 JSON 文本消息，这些是不必要的。扩展和子协议在握手中协商 — 通过不声明它们，它们永远不会被激活。

**缓冲区累积：** 每个连接维护一个缓冲区。收到 `data` 事件时，追加数据并循环调用 `decodeFrame` 直到返回 null 或缓冲区为空。

### HTTP 服务器

三个路由：

1. **`GET /`** — 按修改时间提供屏幕目录中最新的 `.html` 文件。检测完整文档 vs 片段，将片段包装在框架模板中，注入 helper.js。返回 `text/html`。当没有 `.html` 文件时，提供硬编码的等待页面（"等待 Claude 推送屏幕..."）并注入 helper.js。
2. **`GET /files/*`** — 从屏幕目录提供静态文件，通过硬编码扩展名映射（html、css、js、png、jpg、gif、svg、json）查找 MIME 类型。未找到时返回 404。
3. **其他所有请求** — 404。

WebSocket 升级通过 HTTP 服务器上的 `'upgrade'` 事件处理，与请求处理程序分开。

### 配置

环境变量（均为可选）：

- `BRAINSTORM_PORT` — 绑定端口（默认：随机高位端口 49152-65535）
- `BRAINSTORM_HOST` — 绑定接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 启动 JSON 中 URL 的主机名（默认：主机为 `127.0.0.1` 时为 `localhost`，否则与主机相同）
- `BRAINSTORM_DIR` — 屏幕目录路径（默认：`/tmp/brainstorm`）

### 启动顺序

1. 如果 `SCREEN_DIR` 不存在则创建（`mkdirSync` 递归）
2. 从 `__dirname` 加载框架模板和 helper.js
3. 在配置的主机/端口上启动 HTTP 服务器
4. 在 `SCREEN_DIR` 上启动 `fs.watch`
5. 成功监听后，将 `server-started` JSON 记录到 stdout：`{ type, port, host, url_host, url, screen_dir }`
6. 将相同的 JSON 写入 `SCREEN_DIR/.server-info`，以便代理在 stdout 被隐藏（后台执行）时能查找到连接详情

### 应用层 WebSocket 消息

当收到来自客户端的 TEXT 帧时：

1. 解析为 JSON。如果解析失败，记录到 stderr 并继续。
2. 记录到 stdout 为 `{ source: 'user-event', ...event }`。
3. 如果事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每行一个事件）。

### 文件监控

`fs.watch(SCREEN_DIR)` 替代 chokidar。在 HTML 文件事件上：

- 新文件（`rename` 事件，文件存在）：如果存在则删除 `.events` 文件（`unlinkSync`），将 `screen-added` 记录到 stdout 为 JSON
- 文件变更（`change` 事件）：将 `screen-updated` 记录到 stdout 为 JSON（不清空 `.events`）
- 两个事件都：向所有已连接的 WebSocket 客户端发送 `{ type: 'reload' }`

每个文件名去抖约 100 毫秒，防止重复事件（在 macOS 和 Linux 上常见）。

### 错误处理

- 来自 WebSocket 客户端的格式错误 JSON：记录到 stderr，继续
- 未处理的操作码：以状态码 1003 关闭
- 客户端断开连接：从广播集合中移除
- `fs.watch` 错误：记录到 stderr，继续
- 无优雅关闭逻辑 — shell 脚本通过 SIGTERM 处理进程生命周期

## 变更内容

| 变更前 | 变更后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单个文件） |
| express、ws、chokidar 依赖 | 无 |
| 无静态文件服务 | `/files/*` 从屏幕目录提供文件 |

## 保持不变的内容

- `helper.js` — 无变更
- `frame-template.html` — 无变更
- `start-server.sh` — 一行更新：`index.js` 改为 `server.js`
- `stop-server.sh` — 无变更
- `visual-companion.md` — 无变更
- 所有现有服务器行为和外部契约

## 平台兼容性

- `server.js` 仅使用跨平台 Node 内置模块
- `fs.watch` 在 macOS、Linux 和 Windows 上的单一扁平目录中可靠
- Shell 脚本需要 bash（Windows 上需要 Git Bash，这是 Claude Code 所要求的）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require 引入 `server.js` 的导出来测试 WebSocket 帧编码/解码、握手计算和协议边界情况。

**集成测试**（`server.test.js`）：测试完整的服务器行为 — HTTP 服务、WebSocket 通信、文件监控、头脑风暴工作流。使用 `ws` npm 包作为仅测试用的客户端依赖（不提供给最终用户）。

---

[← 返回主 README](../../../../README.md) · [English Version](../../../superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md)

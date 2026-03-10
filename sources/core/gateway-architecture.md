# Gateway 架构 - 核心入口

**阅读时间:** 2026-03-10 11:50  
**源码路径:** `docs/concepts/architecture.md` + `dist/gateway-rpc-*.js`  
**核心职责:** Gateway 是 OpenClaw 的通信中枢，管理所有消息表面

---

## 📖 关键发现

### 1. Gateway 的核心定位

```
单一 Gateway 守护进程 = 所有通信表面的唯一入口
```

- **一个主机只有一个 Gateway**
- 管理所有 messaging surfaces：
  - WhatsApp (via Baileys)
  - Telegram (via grammY)
  - Slack, Discord, Signal, iMessage, WebChat
- 通过 **WebSocket** 暴露 API（默认 `127.0.0.1:18789`）

### 2. 三种客户端类型

| 类型 | 连接方式 | 用途 |
|------|----------|------|
| **Control-plane** | WS | macOS app, CLI, web admin |
| **Nodes** | WS (`role: node`) | 设备命令（camera, screen, location） |
| **WebChat** | WS | 静态聊天 UI |

### 3. 连接生命周期

```
Client              Gateway
  │                    │
  ├── req:connect ───► │
  │                    │ 验证 auth
  │◄── res (ok) ────── │ hello-ok + snapshot
  │                    │
  │◄── event:presence ─│
  │◄── event:tick ─────│
  │                    │
  ├── req:agent ─────► │
  │◄── res:agent ──────│ ack {runId, status}
  │◄── event:agent ────│ streaming
  │◄── res:agent ──────│ final
```

### 4. 协议设计要点

**Wire Protocol:**
- 传输：WebSocket，JSON 文本帧
- 第一帧必须是 `connect`
- 请求：`{type:"req", id, method, params}`
- 响应：`{type:"res", id, ok, payload|error}`
- 事件：`{type:"event", event, payload, seq?}`

**安全机制:**
- Token 认证（`OPENCLAW_GATEWAY_TOKEN`）
- 设备配对（device-based pairing）
- 签名挑战（`connect.challenge`）
- 幂等键（防止重复执行）

### 5. Canvas 宿主

Gateway 还提供服务：
- `/__openclaw__/canvas/` - Agent 可编辑的 HTML/CSS/JS
- `/__openclaw__/a2ui/` - A2UI 宿主
- 与 Gateway 同一端口（默认 18789）

---

## 🎨 设计亮点

### 1. 单一入口模式

**为什么这样设计？**
- 避免多个 WhatsApp 会话冲突
- 统一认证和授权
- 简化状态管理

**对比其他设计:**
```
❌ 多 Gateway: 状态同步复杂，资源浪费
✅ 单 Gateway: 简单一致，易于调试
```

### 2. WebSocket 长连接

**优势:**
- 实时事件推送（`event:agent`, `event:presence`）
- 双向通信（请求 + 事件流）
- 低延迟

**对比 REST:**
```
REST: 轮询 → 延迟高，浪费资源
WS:   推送 → 实时，高效
```

### 3. 设备配对机制

```
新设备连接 → 需要批准 → 颁发 device token → 后续连接免审
```

**安全考虑:**
- 本地连接（loopback）可自动批准
- 远程连接必须明确批准
- 签名绑定 `platform + deviceFamily`

### 4. 幂等性设计

**问题:** 网络重试导致重复执行  
**解决:** 幂等键（idempotency keys）

```javascript
{
  method: "send",
  idempotencyKey: "unique-key-123",  // 短期去重缓存
  params: {...}
}
```

---

## ❓ 疑问/待探索

1. **Gateway 如何管理多个 Channel 的并发连接？**
   - 需要查看 `channels/` 目录实现

2. **事件序列号（seq）如何保证不丢失？**
   - 文档说"Events are not replayed"，客户端如何处理断线？

3. **Node 角色的具体能力有哪些？**
   - `caps/commands/permissions` 的完整列表

4. **Canvas 宿主如何与 Agent 交互？**
   - 需要查看 `canvas/` 相关代码

---

## 🔗 关联概念

- [[Gateway 协议]] - `/gateway/protocol`
- [[设备配对]] - `/channels/pairing`
- [[安全模型]] - `/gateway/security`
- [[Channel 路由]] - 消息如何路由到不同平台

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Gateway RPC CLI | `dist/gateway-rpc-*.js` | CLI 调用 Gateway 的封装 |
| 架构文档 | `docs/concepts/architecture.md` | 完整架构说明 |
| 协议定义 | `src/gateway/` (待查) | TypeBox schemas |

---

_下次心跳继续：Agent Loop 实现_

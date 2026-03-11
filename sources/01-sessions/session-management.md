# Session 管理 - 会话状态与隔离

**阅读时间:** 2026-03-10 13:05  
**源码路径:** `docs/concepts/session.md`  
**核心职责:** 管理会话状态、隔离策略、持久化存储

---

## 📖 关键发现

### 1. Session 的核心设计原则

```
OpenClaw treats one direct-chat session per agent as primary.
```

**关键概念:**
- **Direct chats** → 默认 collapse 到 `agent:<agentId>:main`
- **Group/Channel chats** → 独立的 session key
- **Gateway 是唯一真相源** (source of truth)

### 2. DM 会话隔离策略 (`dmScope`)

| 模式 | Session Key 格式 | 适用场景 |
|------|-----------------|----------|
| `main` (默认) | `agent:<agentId>:main` | 单用户，连续性优先 |
| `per-peer` | `agent:<agentId>:dm:<peerId>` | 按发送者隔离 |
| `per-channel-peer` | `agent:<agentId>:<channel>:dm:<peerId>` | **推荐**：多渠道多用户 |
| `per-account-channel-peer` | `agent:<agentId>:<channel>:<accountId>:dm:<peerId>` | 多账号 inbox |

### 3. 安全 DM 模式（重要！）

**问题场景:**
```
默认设置 (dmScope: "main") 的风险：

1. Alice 发送私密消息（医疗预约）
2. Bob 问："我们刚才在聊什么？"
3. Agent 可能用 Alice 的上下文回答 Bob → 隐私泄露！
```

**解决方案:**
```json5
{
  session: {
    dmScope: "per-channel-peer"  // 隔离每个用户的上下文
  }
}
```

**何时启用:**
- ✅ 有多个 DM 允许列表条目
- ✅ 设置 `dmPolicy: "open"`
- ✅ 多个手机号/账号可以发消息
- ✅ 多用户 inbox

### 4. Gateway 是真相源

```
所有 session state 由 Gateway 拥有
UI clients 必须查询 Gateway，不能直接读本地文件
```

**存储位置:**
```
Gateway Host:
  Store:    ~/.openclaw/agents/<agentId>/sessions/sessions.json
  Transcripts: ~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl
```

**远程模式注意:**
- Session store 在 remote gateway host 上
- Token counts 来自 gateway 的 store fields
- Clients 不解析 JSONL transcripts

### 5. Session 维护机制 (Maintenance)

**默认配置:**
```json5
{
  session: {
    maintenance: {
      mode: "warn",           // warn | enforce
      pruneAfter: "30d",      // 删除超过 30 天的
      maxEntries: 500,        // 最多 500 条
      rotateBytes: "10mb",    // sessions.json 超过 10MB 轮转
      resetArchiveRetention: "14d"
    }
  }
}
```

**维护流程 (enforce 模式):**
```
1. 删除超过 pruneAfter 的旧条目
2. 限制条目数到 maxEntries（先删除最旧的）
3. 归档被删除条目的 transcripts
4. 清理旧的 *.deleted 和 *.reset 归档
5. 轮转 sessions.json（超过 rotateBytes）
6. 如果设置了 maxDiskBytes，执行磁盘预算
```

### 6. Session Key 映射规则

```
Direct Chats (遵循 dmScope):
  main:                     agent:<agentId>:main
  per-peer:                 agent:<agentId>:dm:<peerId>
  per-channel-peer:         agent:<agentId>:<channel>:dm:<peerId>
  per-account-channel-peer: agent:<agentId>:<channel>:<accountId>:dm:<peerId>

Group Chats (总是隔离):
  agent:<agentId>:<channel>:group:<id>
  agent:<agentId>:<channel>:channel:<id>
  Telegram topics: ...:topic:<threadId>
```

**Identity Links (跨渠道合并):**
```json5
{
  session: {
    identityLinks: {
      "telegram:123456": "alice",  // Telegram ID → 统一身份
      "slack:U123456": "alice"     // Slack ID → 同一人共享 session
    }
  }
}
```

---

## 🎨 设计亮点

### 1. 灵活的隔离策略

**设计哲学:** "Continuity vs Security 可配置"

```
单用户场景 → dmScope: "main" (连续性)
多用户场景 → dmScope: "per-channel-peer" (安全性)
```

**对比其他设计:**
```
❌ 固定隔离：无法适应不同场景
✅ 可配置策略：灵活选择
```

### 2. Gateway 中心化模型

**优势:**
- 单一真相源，避免状态不一致
- 远程客户端无需访问本地文件
- 统一的 token 计数和统计

**架构:**
```
┌─────────────┐
│   Gateway   │ ← sessions.json (真相源)
└──────┬──────┘
       │ WS
       ├──────────┐
       │          │
┌──────▼──────┐ ┌─▼──────────┐
│  macOS App  │ │  WebChat   │
│  (只读查询)  │ │  (只读查询) │
└─────────────┘ └────────────┘
```

### 3. 自动化维护机制

**问题:** Session store 无限增长  
**解决:** 分层维护策略

```
Time-based:   pruneAfter "30d"
Count-based:  maxEntries 500
Size-based:   rotateBytes "10mb"
Disk-based:   maxDiskBytes "1gb" (可选)
```

**执行顺序:**
```
1. 时间过滤 (最老优先)
2. 数量限制 (FIFO)
3. 归档 transcripts
4. 清理归档
5. 轮转主文件
6. 磁盘预算 (如果配置)
```

### 4. Identity Links 设计

**场景:** 同一用户多渠道联系  
**解决:** 统一身份映射

```
User "Alice":
  Telegram: telegram:123456
  Slack:    slack:U123456
  WhatsApp: whatsapp:8613800138000

配置 identityLinks 后：
→ 所有渠道共享一个 session
→ 上下文连续，体验一致
```

---

## ❓ 疑问/待探索

1. **Session 迁移机制？**
   - 文档提到 Legacy `group:<id>` keys 仍被识别
   - 迁移路径是什么？

2. **Telegram topic sessions 的隔离？**
   - `.../<SessionId>-topic-<threadId>.jsonl`
   - 如何管理大量 topic？

3. **维护性能优化？**
   - 大 session stores 的 write latency
   - 如何平衡维护成本和实时性？

4. **Identity Links 的自动发现？**
   - 手动配置 vs 自动识别同一用户
   - 隐私边界如何界定？

---

## 🔗 关联概念

- [[Gateway 架构]] - 会话存储的真相源
- [[Agent Loop]] - 会话状态在循环中的使用
- [[Context 组装]] - Session 如何注入上下文
- [[安全模型]] - DM 隔离的安全考虑
- [[Memory]] - Pre-compaction memory flush

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Session 文档 | `docs/concepts/session.md` | 完整会话管理说明 |
| Session Store | `~/.openclaw/agents/<agentId>/sessions/sessions.json` | 运行时存储 |
| Transcripts | `~/.openclaw/agents/<agentId>/sessions/*.jsonl` | 会话记录 |

---

## 🔧 当前配置验证

**你的配置:**
```json5
{
  "session": {
    "dmScope": "per-channel-peer"  // ✅ 安全模式已启用
  }
}
```

**评价:** 配置正确，适合多用户场景！

---

_下次心跳继续：Context 组装机制_

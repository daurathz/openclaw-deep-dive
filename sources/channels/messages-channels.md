# Messages & Channels - 消息流与通道系统

**阅读时间:** 2026-03-11 00:40  
**源码路径:** `docs/concepts/messages.md` + `docs/channels/`  
**核心职责:** 管理 inbound 消息路由、会话映射、队列、流式输出

---

## 📖 关键发现

### 1. 消息流（高阶层）

```
Inbound message
    ↓
routing/bindings → session key
    ↓
queue (if run active)
    ↓
agent run (streaming + tools)
    ↓
outbound replies (channel limits + chunking)
```

**关键配置:**
- `messages.*` - 前缀、队列、群组行为
- `agents.defaults.*` - 块流式和分块默认值
- `channels.*` - 各渠道的 caps 和流式开关

### 2. Inbound Dedupe（去重）

**问题:** 重连后渠道可能重复投递消息  
**解决:** 短期缓存

```
Cache key: channel/account/peer/session/message id
          ↓
重复投递 → 检查缓存 → 跳过 agent run
```

### 3. Inbound Debouncing（防抖）

**功能:** 将同一发送者的快速连续消息批处理为单次 agent turn

**配置:**
```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,  // 默认 2 秒
      byChannel: {
        whatsapp: 5000,  // WhatsApp 5 秒
        slack: 1500,     // Slack 1.5 秒
        discord: 1500,   // Discord 1.5 秒
      }
    }
  }
}
```

**规则:**
- ✅ 仅适用于纯文本消息
- ❌ 媒体/附件立即触发
- ✅ 控制命令绕过防抖

### 4. 会话与设备

**核心原则:**
```
Sessions owned by Gateway, not clients
```

**映射规则:**
- Direct chats → collapse 到 agent main session key
- Groups/channels → 独立 session keys
- Session store → 在 gateway host 上

**多设备注意:**
```
多个设备/渠道可以映射到同一会话
  ↓
但历史不完全同步回每个客户端
  ↓
建议：长对话使用一个主设备
```

### 5. Inbound Bodies 分离

**三种 body:**
```
Body:       prompt text（发送给 agent，含渠道信封和历史包装）
CommandBody: raw user text（用于指令/命令解析）
RawBody:    legacy alias for CommandBody
```

**历史包装:**
```
[Chat messages since your last reply - for context]
[Current message - respond to this]
```

**非直接消息:**
- 当前消息体前缀添加 sender label
- 保持实时和排队/历史消息一致性

### 6. 队列和 Followups

**场景:** run 进行中时收到新消息

**模式:**
| 模式 | 行为 |
|------|------|
| `interrupt` | 中断当前 run |
| `steer` | 引导到当前 run |
| `followup` | 等待下次 turn |
| `collect` | 收集为批处理 |

**配置:**
```json5
{
  messages: {
    queue: {
      mode: "followup",
      byChannel: {
        slack: "steer",
        whatsapp: "collect"
      }
    }
  }
}
```

### 7. 流式、分块、批处理

**块流式（Block Streaming）:**
```
模型生成文本块 → 实时发送部分回复
```

**关键设置:**
```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off",      // on|off
      blockStreamingBreak: "text_end",   // text_end|message_end
      blockStreamingChunk: "minChars",   // minChars|maxChars|breakPreference
      blockStreamingCoalesce: true,      // idle-based batching
      humanDelay: 800                    // 人类停顿延迟
    }
  }
}
```

**渠道覆盖:**
- `*.blockStreaming` - 显式开启（非 Telegram 渠道）
- `*.blockStreamingCoalesce` - 批处理开关

### 8. Reasoning 可见性

**控制命令:**
```bash
/reasoning on|off|stream
```

**特点:**
- Reasoning 内容仍计入 token 使用
- Telegram 支持 reasoning 流式到 draft bubble

---

## 🎨 设计亮点

### 1. 分层 Body 设计

**创新点:** 分离 prompt 和 command

```
用户输入："@bot /status 现在几点了？"
    ↓
Body:       "[历史上下文] @bot /status 现在几点了？"
CommandBody: "/status 现在几点了？"  ← 用于解析
    ↓
Directive 剥离 → 执行 /status
剩余文本 → 发送给 agent
```

**优势:**
- 指令解析不污染上下文
- 历史保持完整
- 支持复杂渠道包装

### 2. 智能防抖机制

**设计哲学:** "批处理快速消息，减少 token 浪费"

```
用户快速发送 3 条消息:
  "你好"
  "我想问个问题"
  "关于 OpenClaw 的架构"
      ↓
Debouncing (2000ms)
      ↓
合并为单次 agent turn:
  "关于 OpenClaw 的架构"  ← 使用最新消息
```

**优化:**
- 减少重复上下文
- 降低 API 调用次数
- 提升响应质量

### 3. 队列模式灵活性

**多场景支持:**

```
Interrupt (中断):
  紧急消息 → 立即处理当前 run

Steer (引导):
  相关消息 → 添加到当前 run 上下文

Followup (后续):
  普通消息 → 等待下次 turn

Collect (收集):
  群组消息 → 批处理为摘要
```

**渠道差异化:**
```
Slack:    steer (工作对话连续性强)
WhatsApp: collect (个人消息可批处理)
Discord:  followup (群聊频率高)
```

### 4. 块流式优化

**人类化延迟:**
```
模型生成 Block 1 → 发送
    ↓
humanDelay (800ms) ← 模拟人类打字停顿
    ↓
模型生成 Block 2 → 发送
```

**分块策略:**
```
minChars:  最小字符数触发发送
maxChars:  最大字符数强制分块
breakPreference:  优先在代码块/段落处分隔
```

**优势:**
- 避免频繁小消息
- 保持代码块完整
- 提升阅读体验

### 5. Gateway 中心化会话

**设计哲学:** "单一真相源"

```
┌─────────────┐
│   Gateway   │ ← sessions.json (真相源)
└──────┬──────┘
       │
       ├──────────┐
       │          │
┌──────▼──────┐ ┌─▼──────────┐
│  Device 1   │ │  Device 2  │
│  (部分同步)  │ │  (部分同步) │
└─────────────┘ └────────────┘
```

**对比完全同步:**
```
❌ 完全同步：复杂、冲突多
✅ 中心化：简单、一致
```

---

## ❓ 疑问/待探索

1. **History buffer 的具体实现？**
   - 如何追踪"last reply"？
   - 缓冲区大小限制？

2. **队列模式切换？**
   - 运行时能否动态切换？
   - 如何影响正在进行 的 run？

3. **块流式的性能影响？**
   - 多次小发送 vs 一次大发送
   - WebSocket 开销？

4. **多渠道会话合并？**
   - identityLinks 的具体实现？
   - 冲突解决策略？

---

## 🔗 关联概念

- [[Session 管理]] - 会话映射
- [[Queueing]] - 队列机制
- [[Streaming]] - 流式输出
- [[Channel Routing]] - 渠道路由

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Messages 文档 | `docs/concepts/messages.md` | 消息流完整说明 |
| Channels | `docs/channels/` | 各渠道配置 |
| Queueing | `docs/concepts/queue.md` | 队列机制 |

---

## 🔧 实践建议

### 优化消息处理

1. **配置防抖**
   ```json5
   {
     messages: {
       inbound: {
         debounceMs: 2000,  // 减少快速消息的重复处理
         byChannel: {
           whatsapp: 5000   // WhatsApp 更长防抖
         }
       }
     }
   }
   ```

2. **启用块流式**
   ```json5
   {
     agents: {
       defaults: {
         blockStreamingDefault: "on",
         humanDelay: 800  // 人类化停顿
       }
     }
   }
   ```

3. **配置队列模式**
   ```json5
   {
     messages: {
       queue: {
         mode: "followup",  // 默认后续处理
         byChannel: {
           slack: "steer"   // Slack 引导到当前 run
         }
       }
     }
   }
   ```

4. **监控去重**
   ```bash
   /status  # 查看会话状态
   ```

---

_下次心跳继续：Channel Routing 机制_

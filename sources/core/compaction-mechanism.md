# Compaction 机制 - 上下文压缩

**阅读时间:** 2026-03-11 00:23  
**源码路径:** `docs/concepts/compaction.md`  
**核心职责:** 当会话接近上下文窗口限制时，压缩旧历史以释放空间

---

## 📖 关键发现

### 1. Compaction 的定义

```
Compaction = summarizes older conversation into a compact summary entry
             and keeps recent messages intact
```

**核心特点:**
- **持久化** - 压缩结果保存在 session JSONL 历史中
- **保留近期** - 最近的消息保持完整
- **自动触发** - 当会话接近/超过上下文窗口时

**对比 Pruning:**
```
Compaction:  总结并持久化到 JSONL
Pruning:     仅修剪旧 tool results，仅内存中，每次请求重新计算
```

### 2. 为什么需要 Compaction

**问题:**
```
每个模型都有 context window 限制（最大 token 数）
长时间对话会积累：
  - 用户消息 + 助手消息
  - Tool calls + tool results
  - 附件和转录内容

当接近限制时 → 需要压缩旧历史
```

**示例:**
```
qwen3.5-plus:
  Context Window: 1,000,000 tokens
  
实际使用:
  System prompt: ~10,000 tokens (1%)
  历史对话：可变（可能达到 90%+）
  Tool schemas: ~8,000 tokens
  
→ 需要压缩旧对话释放空间
```

### 3. 配置选项

**默认配置:**
```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard",           // default | safeguard
        reserveTokensFloor: 24000,   // 保留至少 24K tokens
        identifierPolicy: "strict",  // 保留标识符
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-5"  // 可选
      }
    }
  }
}
```

**关键参数:**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `mode` | `"safeguard"` | `default` 或 `safeguard`（分块总结） |
| `reserveTokensFloor` | `24000` | 压缩后至少保留的 token 数 |
| `identifierPolicy` | `"strict"` | 保留部署 ID、ticket ID 等 |
| `model` | unset | 可选的专用压缩模型 |

### 4. 自动压缩（默认开启）

**触发条件:**
```
当 session 接近或超过模型的 context window 时
  ↓
OpenClaw 触发 auto-compaction
  ↓
可能重试原始请求（使用压缩后的上下文）
```

**用户可见:**
```
Verbose 模式：🧹 Auto-compaction complete
/status:     🧹 Compactions: <count>
```

**压缩前:**
```
可选运行 silent memory flush
提醒模型将持久化笔记写入磁盘
→ memory/YYYY-MM-DD.md
```

### 5. 手动压缩

**命令:**
```bash
/compact Focus on decisions and open questions
```

**用途:**
- 会话感觉"陈旧"时
- 上下文膨胀时
- 主动管理 token 使用

### 6. 上下文窗口来源

**模型特定:**
```
OpenClaw 从 provider catalog 获取模型定义
  ↓
确定 context window 限制
  ↓
动态调整压缩策略
```

**示例:**
```json
{
  "id": "qwen3.5-plus",
  "contextWindow": 1000000,
  "maxTokens": 65536
}
```

### 7. OpenAI 服务端压缩

**特殊支持:**
```
OpenAI Responses API 支持服务端压缩 hints
  ↓
与本地 OpenClaw 压缩并行运行
```

**对比:**
```
本地压缩：OpenClaw 总结并持久化到 JSONL
服务端压缩：OpenAI 在 provider 侧压缩（当 store + context_management 启用）
```

### 8. Compaction vs Pruning

| 特性 | Compaction | Pruning |
|------|------------|---------|
| **持久化** | ✅ JSONL 历史 | ❌ 仅内存 |
| **范围** | 整个对话 | 仅 tool results |
| **触发** | 接近窗口限制 | 每次请求 |
| **可配置** | ✅ 多个参数 | ❌ 固定策略 |

---

## 🎨 设计亮点

### 1. 分层压缩策略

**设计哲学:** "智能总结，保留重点"

```
最近消息 (保持完整)
    ↓
压缩点 (summarization point)
    ↓
旧历史 (总结为 compact entry)
```

**优势:**
- 保留对话连贯性
- 减少 token 占用
- 支持超长会话

### 2. 标识符保护机制

**问题:** 压缩可能丢失重要标识符  
**解决:** `identifierPolicy: "strict"`

**保护内容:**
- 部署 IDs
- Ticket IDs
- Host:port pairs
- 其他不透明标识符

**配置:**
```json5
{
  identifierPolicy: "strict",  // 默认：保留所有
  // 或
  identifierPolicy: "custom",
  identifierInstructions: "Preserve deployment IDs, ticket IDs..."
}
```

### 3. 专用压缩模型

**创新点:** 可以使用不同模型进行压缩

**场景:**
```
主模型：本地小模型（快速、便宜）
压缩模型：专用大模型（总结质量高）

示例：
  primary: "ollama/llama3.1:8b"
  compaction: "openrouter/anthropic/claude-sonnet-4-5"
```

**优势:**
- 压缩质量更高
- 不影响主模型选择
- 成本优化

### 4. Silent Memory Flush

**压缩前自动触发:**
```
Session 接近压缩阈值
    ↓
Silent memory flush turn
    ↓
提醒模型写入持久化笔记
    ↓
memory/YYYY-MM-DD.md
```

**设计哲学:** "压缩前保存重要信息"

**配置:**
```json5
{
  compaction: {
    memoryFlush: {
      enabled: true,
      softThresholdTokens: 6000,
      systemPrompt: "Session nearing compaction. Store durable memories now.",
      prompt: "Write any lasting notes to memory/YYYY-MM-DD.md..."
    }
  }
}
```

### 5. 透明压缩提示

**用户可见性:**
```
🧹 Auto-compaction complete  (verbose 模式)
🧹 Compactions: <count>      (/status)
```

**对比黑盒:**
```
❌ 黑盒：用户不知道为什么对话"断"了
✅ 透明：用户知道压缩发生，可以检查
```

---

## ❓ 疑问/待探索

1. **Safeguard mode 具体实现？**
   - 分块总结的策略是什么？
   - 与 default mode 的区别？

2. **压缩算法？**
   - 使用什么 prompt 进行总结？
   - 如何保证总结质量？

3. **压缩点选择？**
   - 如何决定从哪条消息开始压缩？
   - 是否考虑对话结构？

4. **性能影响？**
   - 压缩过程耗时多久？
   - 是否阻塞原始请求？

---

## 🔗 关联概念

- [[Context 组装]] - 上下文构建
- [[Session 管理]] - 会话历史存储
- [[Memory]] - 持久化记忆
- [[Token 使用]] - token 成本计算
- [[Session Pruning]] - 工具结果修剪

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Compaction 文档 | `docs/concepts/compaction.md` | 完整压缩机制说明 |
| Session Pruning | `docs/concepts/session-pruning.md` | 修剪机制 |
| Memory | `docs/concepts/memory.md` | 记忆系统 |

---

## 🔧 实践建议

### 优化压缩策略

1. **监控 token 使用**
   ```bash
   /status  # 查看窗口使用率
   /usage tokens  # 开启 token 显示
   ```

2. **主动压缩**
   ```bash
   /compact Focus on key decisions  # 手动压缩
   ```

3. **配置专用模型**
   ```json5
   {
     compaction: {
       model: "anthropic/claude-sonnet-4-5"  # 高质量总结
     }
   }
   ```

4. **启用 memory flush**
   ```json5
   {
     compaction: {
       memoryFlush: { enabled: true }
     }
   }
   ```

---

_下次心跳继续：Session Pruning 机制_

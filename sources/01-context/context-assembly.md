# Context 组装 - 上下文构建机制

**阅读时间:** 2026-03-11 00:17  
**源码路径:** `docs/concepts/context.md`  
**核心职责:** 构建发送给 LLM 的完整上下文（系统提示 + 历史 + 工具）

---

## 📖 关键发现

### 1. Context 的定义

```
"Context" = everything OpenClaw sends to the model for a run
```

**核心组成:**
- **System prompt** (OpenClaw 构建) - 规则、工具、技能列表、时间、运行时信息
- **Conversation history** - 用户消息 + 助手消息
- **Tool calls/results** - 命令输出、文件读取、图片/音频等
- **Attachments** - 附件和转录内容

**重要区别:**
```
Context ≠ Memory

Context: 当前模型窗口内的内容（有上限）
Memory: 磁盘上存储的内容（可重新加载）
```

### 2. 上下文窗口限制

**受限于模型的 context window:**
```
示例：qwen3.5-plus
- Context Window: 1,000,000 tokens
- Max Output: 65,536 tokens
```

**实际占用:**
```
System prompt:     ~10,000 tokens (1%)
Conversation:      可变
Tool schemas:      ~8,000 tokens (JSON 格式)
Skills list:       ~500 tokens (仅元数据)
```

### 3. System Prompt 构建

**OpenClaw 每次运行都会重建 system prompt:**

```
┌─────────────────────────────────────────────┐
│ Tool list + descriptions                    │
├─────────────────────────────────────────────┤
│ Skills list (name + description + location) │
├─────────────────────────────────────────────┤
│ Workspace location                          │
├─────────────────────────────────────────────┤
│ Time (UTC + user timezone)                  │
├─────────────────────────────────────────────┤
│ Runtime metadata (host/OS/model/thinking)   │
├─────────────────────────────────────────────┤
│ Injected workspace files (Project Context)  │
└─────────────────────────────────────────────┘
```

### 4. 注入的工作空间文件 (Project Context)

**默认注入的文件:**
- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (首次运行)

**限制:**
```json5
{
  agents: {
    defaults: {
      bootstrapMaxChars: 20000,      // 每文件最大 20K 字符
      bootstrapTotalMaxChars: 150000  // 所有文件总计 150K 字符
    }
  }
}
```

**截断处理:**
- 大文件会被截断
- `/context` 命令显示 raw vs injected 大小
- 可配置截断警告：`bootstrapPromptTruncationWarning`

### 5. Skills：注入 vs 按需加载

**设计哲学:** "元数据注入，指令按需"

```
System Prompt 中：
  Skills list (紧凑) = name + description + location
                       ↓
  模型需要时：
  read SKILL.md → 获取详细指令
```

**优势:**
- 减少初始上下文占用
- 只在需要时加载技能指令
- 支持大量技能而不爆炸 token

### 6. Tools：双重成本

**Tools 影响上下文的两种方式:**

1. **Tool list text** (system prompt 中可见)
   ```
   Tool list (system prompt text): 1,032 chars (~258 tok)
   ```

2. **Tool schemas** (JSON，不可见但计数)
   ```
   Tool schemas (JSON): 31,988 chars (~7,997 tok)
   ```

**查看工具成本:**
```bash
/context detail
```

### 7. 检查上下文的命令

| 命令 | 用途 |
|------|------|
| `/status` | 快速查看窗口使用率 + 会话设置 |
| `/context list` | 注入内容 + 粗略大小（每文件 + 总计） |
| `/context detail` | 详细分解：每文件、每工具 schema、每技能 |
| `/usage tokens` | 在回复后附加 token 使用信息 |
| `/compact` | 压缩旧历史，释放窗口空间 |

### 8. 示例输出

**`/context list`:**
```
🧠 Context breakdown
Workspace: /home/openclaw/.openclaw/workspace-peter
Bootstrap max/file: 20,000 chars
System prompt (run): 38,412 chars (~9,603 tok)

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars | injected 1,742 chars
- SOUL.md: OK | raw 912 chars | injected 912 chars
- TOOLS.md: TRUNCATED | raw 54,210 chars | injected 20,962 chars
- HEARTBEAT.md: MISSING | raw 0 | injected 0

Skills list: 2,184 chars (~546 tok) (12 skills)
Tool schemas (JSON): 31,988 chars (~7,997 tok)

Session tokens: 14,250 total / ctx=32,000
```

---

## 🎨 设计亮点

### 1. 分层加载策略

**问题:** 如何支持大量技能和工具而不爆炸 token？  
**解决:** 分层加载

```
Layer 1 (必须): System prompt 基础框架
Layer 2 (可选): Skills list 元数据
Layer 3 (按需): SKILL.md 详细指令
Layer 4 (按需): Tool results 具体内容
```

**对比其他设计:**
```
❌ 全量加载：token 浪费，上下文有限
✅ 分层加载：按需读取，支持大规模
```

### 2. 透明的上下文管理

**设计哲学:** "让用户知道模型看到了什么"

```
/status      → 窗口使用率
/context     → 内容分解
/usage       → 每次交互成本
/compact     → 主动管理历史
```

**对比黑盒:**
```
❌ 黑盒：用户不知道为什么模型"忘记"了
✅ 透明：用户可以检查和优化
```

### 3. 智能截断机制

**问题:** 大文件如何处理？  
**解决:** 分层截断

```
Per-file limit:   20,000 chars
Total limit:      150,000 chars
Truncation warn:  可选注入警告
```

**优势:**
- 防止单个文件垄断上下文
- 保持重要文件完整
- 用户可配置警告

### 4. Skills 元数据设计

**创新点:** "技能即插件"

```
System Prompt:
  <skill>
    <name>weather</name>
    <description>Get weather data</description>
    <location>/path/to/SKILL.md</location>
  </skill>

模型决策:
  "我需要天气信息" → read /path/to/SKILL.md
```

**对比传统:**
```
传统：所有指令都在 system prompt
OpenClaw: 元数据 + 按需读取
```

---

## ❓ 疑问/待探索

1. **Context engine 插件？**
   - 文档提到 `plugins.slots.contextEngine`
   - 如何自定义上下文组装逻辑？

2. **Tool schemas 优化？**
   - 31KB 的 schema 是否可以压缩？
   - 是否可以按需加载 schema？

3. **Session tokens 缓存？**
   - 如何避免重复计算？
   - 缓存失效策略是什么？

4. **Compaction 策略？**
   - 何时触发自动压缩？
   - 压缩算法是什么？

---

## 🔗 关联概念

- [[System Prompt]] - 系统提示构建
- [[Session 管理]] - 会话历史存储
- [[Compaction]] - 上下文压缩
- [[Skills]] - 技能系统
- [[Tools]] - 工具定义

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Context 文档 | `docs/concepts/context.md` | 完整上下文说明 |
| System Prompt | `docs/concepts/system-prompt.md` | 系统提示构建 |
| Compaction | `docs/concepts/compaction.md` | 压缩机制 |

---

## 🔧 实践建议

### 优化上下文使用

1. **保持 bootstrap 文件简洁**
   - TOOLS.md > 20KB 会被截断
   - 考虑拆分大文件

2. **按需使用技能**
   - 技能列表有开销
   - 只启用需要的技能

3. **定期压缩历史**
   - `/compact` 释放窗口
   - 避免上下文溢出

4. **监控 token 使用**
   - `/usage tokens` 开启显示
   - 了解每次交互成本

---

_下次心跳继续：Compaction 机制_

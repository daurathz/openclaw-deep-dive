# Context 组装机制 — Bootstrap & History Context

**阅读时间:** 2026-03-11  
**源码文件:** 
- `dist/plugin-sdk/pi-embedded-helpers-E84qlHCL.js` (buildBootstrapContextFiles)
- `dist/plugin-sdk/dispatch-CJdFmoH9.js` (buildHistoryContext, finalizeInboundContext)

---

## 1. Context 组装流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│  入站消息 (Inbound Message)                                      │
│  - 通道插件接收 (Slack/Discord/Telegram/...)                     │
│  - 标准化消息格式                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  resolveInboundSessionEnvelopeContext()                         │
│  - 解析会话存储路径                                              │
│  - 获取信封格式选项                                              │
│  - 读取上次会话更新时间                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  finalizeInboundContext()                                       │
│  - 标准化文本 (换行/系统标签清理)                                │
│  - 生成 BodyForAgent / BodyForCommands                          │
│  - 解析会话标签                                                  │
│  - 媒体类型标准化                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  buildBootstrapContextFiles()                                   │
│  - 加载工作空间 Bootstrap 文件 (AGENTS.md/SOUL.md/USER.md/...)    │
│  - 应用字符限制 (maxChars/totalMaxChars)                        │
│  - 截断过长文件                                                  │
│  - 返回上下文文件数组                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  buildHistoryContext()                                          │
│  - 从历史 Map 获取条目                                            │
│  - 格式化为历史文本                                              │
│  - 拼接当前消息                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  最终 Context                                                    │
│  - Bootstrap Context (工作空间文件)                              │
│  - History Context (历史对话)                                    │
│  - Current Message (当前消息)                                    │
│  - 发送到模型                                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心函数详解

### 2.1 `resolveInboundSessionEnvelopeContext()` — 解析会话信封上下文

**职责:** 准备会话信封的上下文信息

```javascript
function resolveInboundSessionEnvelopeContext(params) {
  const { cfg, agentId, sessionKey } = params;
  
  // 1. 解析会话存储路径
  const storePath = resolveStorePath(cfg.session?.store, { agentId });
  
  // 2. 获取信封格式选项
  const envelopeOptions = resolveEnvelopeFormatOptions(cfg);
  
  // 3. 读取上次会话更新时间
  const previousTimestamp = readSessionUpdatedAt({
    storePath,
    sessionKey
  });
  
  return {
    storePath,
    envelopeOptions,
    previousTimestamp
  };
}
```

**返回:**
```javascript
{
  storePath: "~/.openclaw/sessions/peter/store.json",
  envelopeOptions: { /* 信封格式配置 */ },
  previousTimestamp: 1710067200000
}
```

---

### 2.2 `finalizeInboundContext()` — 标准化入站上下文

**职责:** 清理和标准化入站消息

```javascript
function finalizeInboundContext(ctx, opts = {}) {
  const normalized = ctx;
  
  // 1. 文本标准化
  normalized.Body = sanitizeInboundSystemTags(
    normalizeInboundTextNewlines(
      typeof normalized.Body === "string" ? normalized.Body : ""
    )
  );
  normalized.RawBody = normalizeTextField(normalized.RawBody);
  normalized.CommandBody = normalizeTextField(normalized.CommandBody);
  normalized.Transcript = normalizeTextField(normalized.Transcript);
  
  // 2. 聊天类型标准化
  const chatType = normalizeChatType(normalized.ChatType);
  if (chatType && (opts.forceChatType || normalized.ChatType !== chatType)) {
    normalized.ChatType = chatType;
  }
  
  // 3. 生成 Agent 用消息体
  normalized.BodyForAgent = sanitizeInboundSystemTags(
    normalizeInboundTextNewlines(
      opts.forceBodyForAgent ? normalized.Body : 
      normalized.BodyForAgent ?? normalized.CommandBody ?? 
      normalized.RawBody ?? normalized.Body
    )
  );
  
  // 4. 生成命令用消息体
  normalized.BodyForCommands = sanitizeInboundSystemTags(
    normalizeInboundTextNewlines(
      opts.forceBodyForCommands ? 
        normalized.CommandBody ?? normalized.RawBody ?? normalized.Body :
      normalized.BodyForCommands ?? normalized.CommandBody ?? 
      normalized.RawBody ?? normalized.Body
    )
  );
  
  // 5. 解析会话标签
  const explicitLabel = normalized.ConversationLabel?.trim();
  if (opts.forceConversationLabel || !explicitLabel) {
    const resolved = resolveConversationLabel(normalized)?.trim();
    if (resolved) normalized.ConversationLabel = resolved;
  } else {
    normalized.ConversationLabel = explicitLabel;
  }
  
  // 6. 媒体类型标准化
  const mediaCount = countMediaEntries(normalized);
  if (mediaCount > 0) {
    const mediaType = normalizeMediaType(normalized.MediaType);
    const normalizedMediaTypes = normalized.MediaTypes?.map(
      entry => normalizeMediaType(entry)
    );
    
    // 填充默认媒体类型
    let mediaTypesFinal;
    if (normalizedMediaTypes?.length > 0) {
      const filled = normalizedMediaTypes.slice();
      while (filled.length < mediaCount) {
        filled.push(DEFAULT_MEDIA_TYPE);
      }
      mediaTypesFinal = filled;
    } else if (mediaType) {
      mediaTypesFinal = [mediaType];
      while (mediaTypesFinal.length < mediaCount) {
        mediaTypesFinal.push(DEFAULT_MEDIA_TYPE);
      }
    } else {
      mediaTypesFinal = Array.from(
        { length: mediaCount }, 
        () => DEFAULT_MEDIA_TYPE
      );
    }
    
    normalized.MediaTypes = mediaTypesFinal;
    normalized.MediaType = mediaType ?? mediaTypesFinal[0] ?? DEFAULT_MEDIA_TYPE;
  }
  
  return normalized;
}
```

**标准化内容:**
- **系统标签清理** — 移除 `[[reply_to:...]]` 等内部标签
- **换行标准化** — 统一换行符格式
- **消息体分离** — `BodyForAgent` (给模型) vs `BodyForCommands` (给命令)
- **媒体类型推断** — 根据 MIME 类型或扩展名推断

---

### 2.3 `buildBootstrapContextFiles()` — 构建 Bootstrap 上下文

**职责:** 加载和格式化工作空间 Bootstrap 文件

```javascript
function buildBootstrapContextFiles(files, opts) {
  const maxChars = opts?.maxChars ?? 20000;  // 单文件最大字符
  let remainingTotalChars = Math.max(
    1, 
    Math.floor(opts?.totalMaxChars ?? Math.max(maxChars, 150000))
  );
  
  const result = [];
  
  for (const file of files) {
    if (remainingTotalChars <= 0) break;
    
    const pathValue = typeof file.path === "string" ? file.path.trim() : "";
    
    // 检查路径有效性
    if (!pathValue) {
      opts?.warn?.(
        `skipping bootstrap file "${file.name}" — missing or invalid "path" field`
      );
      continue;
    }
    
    // 处理缺失文件
    if (file.missing) {
      const cappedMissingText = clampToBudget(
        `[MISSING] Expected at: ${pathValue}`, 
        remainingTotalChars
      );
      if (!cappedMissingText) break;
      
      remainingTotalChars = Math.max(0, remainingTotalChars - cappedMissingText.length);
      result.push({ path: pathValue, content: cappedMissingText });
      continue;
    }
    
    // 检查剩余预算
    if (remainingTotalChars < MIN_BOOTSTRAP_FILE_BUDGET_CHARS) {
      opts?.warn?.(
        `remaining bootstrap budget is ${remainingTotalChars} chars 
         (<${MIN_BOOTSTRAP_FILE_BUDGET_CHARS}); skipping additional files`
      );
      break;
    }
    
    // 截断文件内容
    const fileMaxChars = Math.max(1, Math.min(maxChars, remainingTotalChars));
    const trimmed = trimBootstrapContent(file.content ?? "", file.name, fileMaxChars);
    const contentWithinBudget = clampToBudget(trimmed.content, remainingTotalChars);
    
    if (!contentWithinBudget) continue;
    
    // 记录截断警告
    if (trimmed.truncated || contentWithinBudget.length < trimmed.content.length) {
      opts?.warn?.(
        `workspace bootstrap file ${file.name} is ${trimmed.originalLength} chars 
         (limit ${trimmed.maxChars}); truncating in injected context`
      );
    }
    
    remainingTotalChars = Math.max(0, remainingTotalChars - contentWithinBudget.length);
    result.push({ path: pathValue, content: contentWithinBudget });
  }
  
  return result;
}
```

**配置参数:**
```javascript
{
  maxChars: 20000,        // 单文件最大字符
  totalMaxChars: 150000,  // 总字符预算
  warn: (msg) => { ... }  // 警告回调
}
```

**返回结构:**
```javascript
[
  {
    path: "/workspace/AGENTS.md",
    content: "# AGENTS.md - Peter's Workspace\n\n..."
  },
  {
    path: "/workspace/SOUL.md",
    content: "# SOUL.md - Who You Are\n\n..."
  },
  {
    path: "/workspace/USER.md",
    content: "[MISSING] Expected at: /workspace/USER.md"
  }
]
```

---

### 2.4 `buildHistoryContext()` — 构建历史上下文

**职责:** 拼接历史对话和当前消息

```javascript
function buildHistoryContext(params) {
  const { historyText, currentMessage } = params;
  const lineBreak = params.lineBreak ?? "\n";
  
  // 如果没有历史，直接返回当前消息
  if (!historyText.trim()) return currentMessage;
  
  // 拼接历史 + 当前消息
  return [
    HISTORY_CONTEXT_MARKER,      // 历史上下文标记
    historyText,                 // 历史文本
    "",                          // 空行分隔
    CURRENT_MESSAGE_MARKER,      // 当前消息标记
    currentMessage               // 当前消息
  ].join(lineBreak);
}
```

**标记常量:**
```javascript
const HISTORY_CONTEXT_MARKER = "## Conversation History";
const CURRENT_MESSAGE_MARKER = "## Current Message";
```

**输出示例:**
```
## Conversation History

User: 你好
Assistant: 你好！有什么可以帮助你的吗？
User: 今天天气怎么样

## Current Message

Assistant: 我需要查询天气信息...
```

---

### 2.5 `buildHistoryContextFromEntries()` — 从条目构建历史

**职责:** 从历史条目数组格式化历史文本

```javascript
function buildHistoryContextFromEntries(params) {
  const lineBreak = params.lineBreak ?? "\n";
  
  // 排除最后一条 (除非 excludeLast=false)
  const entries = params.excludeLast === false 
    ? params.entries 
    : params.entries.slice(0, -1);
  
  if (entries.length === 0) return params.currentMessage;
  
  return buildHistoryContext({
    historyText: entries.map(params.formatEntry).join(lineBreak),
    currentMessage: params.currentMessage,
    lineBreak
  });
}
```

**条目格式化:**
```javascript
const formatEntry = (entry) => {
  return `${entry.role}: ${entry.content}`;
};
```

---

### 2.6 `buildPendingHistoryContextFromMap()` — 从 Map 构建待处理历史

**职责:** 从历史 Map 获取待处理的历史条目

```javascript
function buildPendingHistoryContextFromMap(params) {
  if (params.limit <= 0) return params.currentMessage;
  
  return buildHistoryContextFromEntries({
    entries: params.historyMap.get(params.historyKey) ?? [],
    currentMessage: params.currentMessage,
    formatEntry: params.formatEntry,
    lineBreak: params.lineBreak,
    excludeLast: false
  });
}
```

**历史 Map 结构:**
```javascript
const historyMap = new Map();
// key: sessionKey
// value: [{ role, content, timestamp }, ...]
```

---

## 3. 历史管理

### 3.1 记录历史条目

```javascript
function recordPendingHistoryEntry(params) {
  const { historyMap, historyKey, entry, limit } = params;
  
  if (limit <= 0) return [];
  
  const history = historyMap.get(historyKey) ?? [];
  history.push(entry);
  
  // 超出限制时移除最旧的条目
  while (history.length > limit) {
    history.shift();
  }
  
  if (historyMap.has(historyKey)) {
    historyMap.delete(historyKey);
  }
  historyMap.set(historyKey, history);
  
  // 驱逐旧的历史键
  evictOldHistoryKeys(historyMap);
  
  return history;
}
```

### 3.2 清除历史

```javascript
function clearHistoryEntries(params) {
  const { historyMap, historyKey } = params;
  historyMap.set(historyKey, []);
}

function clearHistoryEntriesIfEnabled(params) {
  if (params.limit <= 0) return;
  clearHistoryEntries({
    historyMap: params.historyMap,
    historyKey: params.historyKey
  });
}
```

### 3.3 驱逐旧历史键

```javascript
function evictOldHistoryKeys(historyMap, maxKeys = 1000) {
  if (historyMap.size <= maxKeys) return;
  
  const keys = [...historyMap.keys()];
  const toDelete = keys.slice(0, keys.length - maxKeys);
  
  for (const key of toDelete) {
    historyMap.delete(key);
  }
}
```

---

## 4. Bootstrap 文件管理

### 4.1 标准 Bootstrap 文件

| 文件 | 用途 | 必需 |
|------|------|------|
| `AGENTS.md` | Agent 配置和规则 | ✅ |
| `SOUL.md` | Agent 人格和语气 | ✅ |
| `USER.md` | 用户信息 | ❌ |
| `TOOLS.md` | 本地工具配置 | ❌ |
| `IDENTITY.md` | Agent 身份 | ❌ |
| `HEARTBEAT.md` | 心跳任务 | ❌ |
| `BOOTSTRAP.md` | 启动任务 | ❌ |

### 4.2 Bootstrap 文件加载

```javascript
async function loadWorkspaceBootstrapFiles(workspaceDir, opts) {
  const files = [];
  const standardFiles = [
    'AGENTS.md',
    'SOUL.md',
    'USER.md',
    'TOOLS.md',
    'IDENTITY.md',
    'HEARTBEAT.md',
    'BOOTSTRAP.md'
  ];
  
  for (const fileName of standardFiles) {
    const filePath = path.join(workspaceDir, fileName);
    
    try {
      const content = await fs.readFile(filePath, 'utf-8');
      files.push({
        name: fileName,
        path: filePath,
        content,
        missing: false
      });
    } catch (err) {
      if (err.code === 'ENOENT') {
        files.push({
          name: fileName,
          path: filePath,
          content: '',
          missing: true
        });
      } else {
        throw err;
      }
    }
  }
  
  return files;
}
```

---

## 5. 字符预算管理

### 5.1 预算层级

```
totalMaxChars (150,000)
  └── maxChars per file (20,000)
      └── 实际内容 (动态截断)
```

### 5.2 截断策略

```javascript
function trimBootstrapContent(content, fileName, maxChars) {
  if (content.length <= maxChars) {
    return {
      content,
      truncated: false,
      originalLength: content.length,
      maxChars
    };
  }
  
  // 截断到最大字符数
  const truncated = content.slice(0, maxChars - 3) + '...';
  
  return {
    content: truncated,
    truncated: true,
    originalLength: content.length,
    maxChars
  };
}
```

### 5.3 预算钳制

```javascript
function clampToBudget(content, budget) {
  if (budget <= 0) return null;
  if (content.length <= budget) return content;
  return content.slice(0, budget - 3) + '...';
}
```

---

## 6. Context 注入流程

### 6.1 完整流程

```javascript
async function prepareAgentContext(params) {
  const { cfg, sessionKey, workspaceDir, message } = params;
  
  // 1. 解析会话信封上下文
  const envelopeCtx = resolveInboundSessionEnvelopeContext({
    cfg,
    agentId: resolveAgentIdFromSessionKey(sessionKey),
    sessionKey
  });
  
  // 2. 标准化入站上下文
  const ctx = {
    Body: message,
    SessionKey: sessionKey,
    // ... 其他字段
  };
  const finalizedCtx = finalizeInboundContext(ctx);
  
  // 3. 加载 Bootstrap 文件
  const bootstrapFiles = await loadWorkspaceBootstrapFiles(workspaceDir);
  const contextFiles = buildBootstrapContextFiles(bootstrapFiles, {
    maxChars: cfg.bootstrap?.maxChars ?? 20000,
    totalMaxChars: cfg.bootstrap?.totalMaxChars ?? 150000,
    warn: (msg) => log.warn(msg)
  });
  
  // 4. 获取历史上下文
  const historyCtx = buildPendingHistoryContextFromMap({
    historyMap: sessionHistoryMap,
    historyKey: sessionKey,
    currentMessage: finalizedCtx.Body,
    limit: cfg.history?.limit ?? 50,
    formatEntry: (e) => `${e.role}: ${e.content}`
  });
  
  // 5. 组装最终 Context
  return {
    bootstrap: contextFiles,
    history: historyCtx,
    current: finalizedCtx
  };
}
```

### 6.2 发送到模型

```javascript
const systemPrompt = [
  "# System Context",
  "",
  "## Bootstrap Files",
  ...contextFiles.map(f => `\n### ${f.path}\n\n${f.content}`),
  "",
  "## Conversation",
  historyCtx
].join("\n");

await model.generate({
  system: systemPrompt,
  messages: [{ role: "user", content: finalizedCtx.Body }]
});
```

---

## 7. 关键设计模式

### 7.1 分层预算

- **总预算** — 防止超出模型上下文限制
- **单文件预算** — 防止单个文件占用过多
- **动态截断** — 根据剩余预算调整

### 7.2 惰性加载

- Bootstrap 文件按需加载
- 历史条目按需格式化
- 避免不必要的内存占用

### 7.3 优雅降级

- 缺失文件标记为 `[MISSING]`
- 预算不足时跳过后续文件
- 截断时记录警告

---

## 8. 故障诊断

### 8.1 Bootstrap 文件未加载

```bash
# 检查文件存在
ls -la <workspace>/*.md

# 查看加载警告
openclaw logs tail | grep bootstrap
```

### 8.2 历史上下文丢失

```bash
# 检查会话存储
cat ~/.openclaw/sessions/<agent>/store.json

# 查看历史限制配置
openclaw config get history.limit
```

### 8.3 上下文超出预算

```bash
# 查看截断警告
openclaw logs tail | grep truncating

# 调整预算配置
openclaw config set bootstrap.totalMaxChars 200000
```

---

## 9. 相关文档

- [Bootstrap 文件](https://docs.openclaw.ai/workspace/bootstrap)
- [会话历史](https://docs.openclaw.ai/sessions/history)
- [Context 管理](https://docs.openclaw.ai/context)

---

**阶段 4 - Context 组装** 完成 ✅

**下一步:** 其他核心模块或总结

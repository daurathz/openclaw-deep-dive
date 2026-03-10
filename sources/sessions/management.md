# 会话管理系统 — ACP Session Manager

**阅读时间:** 2026-03-10  
**源码文件:** 
- `dist/plugin-sdk/dispatch-CJdFmoH9.js` (AcpSessionManager, SessionActorQueue, RuntimeCache)
- `dist/plugin-sdk/sessions-*.js` (会话存储操作)

---

## 1. 会话管理系统概览

```
┌─────────────────────────────────────────────────────────────────┐
│  AcpSessionManager                                              │
│  - 管理 ACP (Agent Coding Protocol) 会话运行时                    │
│  - 会话初始化/状态查询/身份协调                                   │
│  - 运行时缓存与逐出                                              │
│  - 并发控制与队列管理                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  SessionActorQueue                                              │
│  - 会话级操作序列化 (避免并发冲突)                               │
│  - 每个会话一个 Actor 队列                                        │
│  - 防止同一会话的并发修改                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  RuntimeCache                                                   │
│  - 缓存活跃的运行时句柄                                          │
│  - 空闲超时逐出 (idle TTL)                                      │
│  - 并发会话数限制                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  会话存储 (Session Store)                                       │
│  - JSON 文件存储 (~/.openclaw/sessions/<agent>/store.json)      │
│  - 会话元数据 (sessionId, updatedAt, channel, to, ...)          │
│  - 使用量统计 (inputTokens, outputTokens, totalTokens)          │
│  - 模型覆盖 (providerOverride, modelOverride)                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心类详解

### 2.1 `AcpSessionManager` — ACP 会话管理器

**类结构:**

```javascript
class AcpSessionManager {
  constructor(deps = DEFAULT_DEPS) {
    this.deps = deps;
    this.actorQueue = new SessionActorQueue();      // 会话操作队列
    this.runtimeCache = new RuntimeCache();          // 运行时缓存
    this.activeTurnBySession = new Map();            // 活跃 Turn 跟踪
    this.turnLatencyStats = {                        // Turn 延迟统计
      completed: 0,
      failed: 0,
      totalMs: 0,
      maxMs: 0
    };
    this.errorCountsByCode = new Map();              // 错误计数
    this.evictedRuntimeCount = 0;                    // 逐出计数
  }
  
  // 解析会话状态
  resolveSession(params) {
    const sessionKey = normalizeSessionKey(params.sessionKey);
    if (!sessionKey) return { kind: "none", sessionKey };
    
    // 读取会话元数据
    const acp = this.deps.readSessionEntry({
      cfg: params.cfg,
      sessionKey
    })?.acp;
    
    if (acp) return {
      kind: "ready",
      sessionKey,
      meta: acp  // ACP 元数据
    };
    
    if (isAcpSessionKey(sessionKey)) return {
      kind: "stale",
      sessionKey,
      error: resolveMissingMetaError(sessionKey)
    };
    
    return { kind: "none", sessionKey };
  }
  
  // 获取可观察性快照
  getObservabilitySnapshot(cfg) {
    const completedTurns = this.turnLatencyStats.completed + this.turnLatencyStats.failed;
    const averageLatencyMs = completedTurns > 0 
      ? Math.round(this.turnLatencyStats.totalMs / completedTurns) 
      : 0;
    
    return {
      runtimeCache: {
        activeSessions: this.runtimeCache.size(),
        idleTtlMs: resolveRuntimeIdleTtlMs(cfg),
        evictedTotal: this.evictedRuntimeCount,
        ...this.lastEvictedAt ? { lastEvictedAt: this.lastEvictedAt } : {}
      },
      turns: {
        active: this.activeTurnBySession.size,
        queueDepth: this.actorQueue.getTotalPendingCount(),
        completed: this.turnLatencyStats.completed,
        failed: this.turnLatencyStats.failed,
        averageLatencyMs,
        maxLatencyMs: this.turnLatencyStats.maxMs
      },
      errorsByCode: Object.fromEntries(
        [...this.errorCountsByCode.entries()].toSorted(([a], [b]) => a.localeCompare(b))
      )
    };
  }
  
  // 初始化会话
  async initializeSession(input) {
    const sessionKey = normalizeSessionKey(input.sessionKey);
    const agent = normalizeAgentId(input.agent);
    
    // 逐出空闲运行时
    await this.evictIdleRuntimeHandles({ cfg: input.cfg });
    
    return await this.withSessionActor(sessionKey, async () => {
      const backend = this.deps.requireRuntimeBackend(
        input.backendId || input.cfg.acp?.backend
      );
      const runtime = backend.runtime;
      
      // 创建会话
      const handle = await runtime.ensureSession({
        sessionKey,
        agent,
        mode: input.mode,
        cwd: input.cwd
      });
      
      // 构建元数据
      const meta = {
        backend: handle.backend || backend.id,
        agent,
        runtimeSessionName: handle.runtimeSessionName,
        identity: { state: "pending", source: "ensure", lastUpdatedAt: Date.now() },
        mode: input.mode,
        cwd: handle.cwd,
        state: "idle",
        lastActivityAt: Date.now()
      };
      
      // 持久化元数据
      await this.writeSessionMeta({
        cfg: input.cfg,
        sessionKey,
        mutate: () => meta,
        failOnError: true
      });
      
      // 缓存运行时状态
      this.setCachedRuntimeState(sessionKey, {
        runtime, handle,
        backend: handle.backend || backend.id,
        agent, mode: input.mode, cwd: handle.cwd
      });
      
      return { runtime, handle, meta };
    });
  }
  
  // 会话 Turn 执行
  async runTurn(params) {
    const sessionKey = normalizeSessionKey(params.sessionKey);
    const startedAt = Date.now();
    
    return await this.withSessionActor(sessionKey, async () => {
      // 确保运行时句柄
      const { runtime, handle, meta } = await this.ensureRuntimeHandle({
        cfg: params.cfg,
        sessionKey,
        meta: params.meta
      });
      
      try {
        // 执行 Turn
        const result = await runtime.runTurn({
          handle,
          text: params.text,
          mode: params.mode,
          requestId: params.requestId,
          signal: params.signal,
          onEvent: params.onEvent
        });
        
        // 更新统计
        this.turnLatencyStats.completed += 1;
        this.turnLatencyStats.totalMs += Date.now() - startedAt;
        
        return result;
      } catch (error) {
        this.turnLatencyStats.failed += 1;
        this.recordError(error);
        throw error;
      }
    });
  }
  
  // 逐出空闲运行时
  async evictIdleRuntimeHandles(params) {
    const idleTtlMs = resolveRuntimeIdleTtlMs(params.cfg);
    const now = Date.now();
    const toEvict = [];
    
    for (const [sessionKey, state] of this.runtimeCache.entries()) {
      if (now - state.lastActivityAt > idleTtlMs) {
        toEvict.push(sessionKey);
      }
    }
    
    for (const sessionKey of toEvict) {
      await this.evictRuntimeHandle(sessionKey, "idle-timeout");
      this.evictedRuntimeCount += 1;
    }
  }
  
  // 逐出单个运行时
  async evictRuntimeHandle(sessionKey, reason) {
    const state = this.runtimeCache.get(sessionKey);
    if (!state) return;
    
    try {
      await state.runtime.close({ handle: state.handle, reason });
    } catch (error) {
      logVerbose(`acp-manager: close failed for ${sessionKey}: ${String(error)}`);
    }
    
    this.runtimeCache.delete(sessionKey);
    this.lastEvictedAt = Date.now();
  }
}
```

**状态流转:**
```
none → ready (初始化) → idle/active (运行中) → evicted (逐出)
```

---

### 2.2 `SessionActorQueue` — 会话操作队列

**职责:** 序列化同一会话的操作，避免并发冲突

```javascript
class SessionActorQueue {
  constructor() {
    this.queueBySession = new Map();  // 每个会话一个队列
  }
  
  async withActor(sessionKey, fn) {
    // 获取或创建会话队列
    let queue = this.queueBySession.get(sessionKey);
    if (!queue) {
      queue = new PromiseQueue();
      this.queueBySession.set(sessionKey, queue);
    }
    
    try {
      // 序列化执行
      return await queue.enqueue(fn);
    } finally {
      // 队列为空时清理
      if (queue.isEmpty()) {
        this.queueBySession.delete(sessionKey);
      }
    }
  }
  
  getTotalPendingCount() {
    let total = 0;
    for (const queue of this.queueBySession.values()) {
      total += queue.pendingCount;
    }
    return total;
  }
}
```

**设计意图:**
- 防止同一会话的并发修改 (如同时两个 Turn)
- 自动清理空闲队列 (节省内存)
- 透明的序列化 (调用方无需关心)

---

### 2.3 `RuntimeCache` — 运行时缓存

**职责:** 缓存活跃的 ACP 运行时句柄

```javascript
class RuntimeCache {
  constructor() {
    this.cache = new Map();  // sessionKey → RuntimeState
  }
  
  set(sessionKey, state) {
    this.cache.set(sessionKey, {
      ...state,
      lastActivityAt: Date.now()
    });
  }
  
  get(sessionKey) {
    const state = this.cache.get(sessionKey);
    if (state) {
      state.lastActivityAt = Date.now();  // 更新活跃时间
    }
    return state;
  }
  
  delete(sessionKey) {
    return this.cache.delete(sessionKey);
  }
  
  size() {
    return this.cache.size;
  }
  
  entries() {
    return this.cache.entries();
  }
}
```

**缓存策略:**
- **活跃更新** — 每次访问更新 `lastActivityAt`
- **空闲逐出** — 超过 `idleTtlMs` 未访问的运行时被逐出
- **并发限制** — 配置 `acp.maxConcurrentSessions` 限制最大并发数

---

## 3. 会话键格式

### 3.1 构建函数

```javascript
function buildAgentSessionKey(params) {
  const { agentId, channel, accountId, peer, dmScope, identityLinks } = params;
  
  // 格式: agent:<agentId>:<mainKey>:<channel>:<accountId>:<peerKind>:<peerId>
  const parts = ["agent", agentId, DEFAULT_MAIN_KEY, channel];
  
  if (accountId) parts.push(accountId);
  if (peer?.kind && peer?.id) parts.push(peer.kind, peer.id);
  
  return parts.join(":");
}

function buildAgentMainSessionKey(params) {
  const { agentId, mainKey = DEFAULT_MAIN_KEY } = params;
  return `agent:${agentId}:${mainKey}`;
}
```

### 3.2 会话键示例

| 类型 | 格式 | 示例 |
|------|------|------|
| Main Session | `agent:<id>:main` | `agent:peter:main` |
| DM Session | `agent:<id>:main:<channel>:<account>:dm:<userId>` | `agent:peter:main:slack:T123:dm:U456` |
| Group Session | `agent:<id>:main:<channel>:<account>:group:<groupId>` | `agent:peter:main:slack:T123:group:G789` |
| Channel Session | `agent:<id>:main:<channel>:<channelId>` | `agent:peter:main:discord:C123` |

---

## 4. 会话存储结构

### 4.1 存储文件

**路径:** `~/.openclaw/sessions/<agentId>/store.json`

```json
{
  "agent:peter:main": {
    "sessionId": "uuid-1234",
    "updatedAt": 1710067200000,
    "channel": "webchat",
    "to": null,
    "accountId": null,
    "chatType": "direct",
    "thinkingLevel": "medium",
    "verboseLevel": "on",
    "providerOverride": "openai",
    "modelOverride": "gpt-4o",
    "inputTokens": 1500,
    "outputTokens": 800,
    "totalTokens": 2300,
    "cacheRead": 100,
    "cacheWrite": 50,
    "skillsSnapshot": {
      "prompt": "Available skills:...",
      "skills": [...],
      "version": 1
    },
    "ttsAuto": "on",
    "authProfileOverride": "profile-123",
    "claudeCliSessionId": "cli-session-456"
  },
  "agent:peter:main:slack:T123:dm:U456": {
    "sessionId": "uuid-5678",
    "updatedAt": 1710067100000,
    "channel": "slack",
    "to": "U456",
    "accountId": "T123",
    "chatType": "direct",
    ...
  }
}
```

### 4.2 会话元数据字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `sessionId` | string | 会话唯一标识 (UUID) |
| `updatedAt` | number | 最后更新时间戳 (ms) |
| `channel` | string | 通道类型 (slack/discord/webchat) |
| `to` | string | 对话目标 (用户 ID/频道 ID) |
| `accountId` | string | 账户 ID |
| `chatType` | string | 聊天类型 (direct/group) |
| `thinkingLevel` | string | 思考级别 (low/medium/high) |
| `verboseLevel` | string | 详细级别 (on/off/full) |
| `providerOverride` | string | 模型提供商覆盖 |
| `modelOverride` | string | 模型覆盖 |
| `inputTokens` | number | 输入 Token 数 |
| `outputTokens` | number | 输出 Token 数 |
| `totalTokens` | number | 总 Token 数 |
| `skillsSnapshot` | object | 技能快照 |
| `ttsAuto` | string | TTS 自动模式 |
| `authProfileOverride` | string | 认证配置覆盖 |

---

## 5. 会话操作流程

### 5.1 会话初始化

```javascript
// 1. 标准化会话键
const sessionKey = normalizeSessionKey(input.sessionKey);

// 2. 逐出空闲运行时
await manager.evictIdleRuntimeHandles({ cfg });

// 3. 获取会话锁 (Actor 队列)
await manager.withSessionActor(sessionKey, async () => {
  // 4. 获取后端运行时
  const backend = deps.requireRuntimeBackend(cfg.acp?.backend);
  const runtime = backend.runtime;
  
  // 5. 创建会话
  const handle = await runtime.ensureSession({
    sessionKey, agent, mode, cwd
  });
  
  // 6. 构建元数据
  const meta = {
    backend: backend.id,
    agent,
    runtimeSessionName: handle.runtimeSessionName,
    identity: { state: "pending", source: "ensure" },
    mode, cwd,
    state: "idle",
    lastActivityAt: Date.now()
  };
  
  // 7. 持久化元数据
  await manager.writeSessionMeta({ cfg, sessionKey, mutate: () => meta });
  
  // 8. 缓存运行时状态
  manager.setCachedRuntimeState(sessionKey, { runtime, handle, ... });
  
  return { runtime, handle, meta };
});
```

### 5.2 会话 Turn 执行

```javascript
// 1. 标准化会话键
const sessionKey = normalizeSessionKey(params.sessionKey);
const startedAt = Date.now();

// 2. 获取会话锁
await manager.withSessionActor(sessionKey, async () => {
  // 3. 确保运行时句柄
  const { runtime, handle, meta } = await manager.ensureRuntimeHandle({
    cfg: params.cfg, sessionKey, meta: params.meta
  });
  
  try {
    // 4. 执行 Turn
    const result = await runtime.runTurn({
      handle,
      text: params.text,
      mode: params.mode,
      requestId: params.requestId,
      signal: params.signal,
      onEvent: params.onEvent  // 流式事件回调
    });
    
    // 5. 更新统计
    manager.turnLatencyStats.completed += 1;
    manager.turnLatencyStats.totalMs += Date.now() - startedAt;
    
    return result;
  } catch (error) {
    manager.turnLatencyStats.failed += 1;
    manager.recordError(error);
    throw error;
  }
});
```

### 5.3 会话元数据更新

```javascript
async function updateSessionStoreAfterAgentRun(params) {
  const { cfg, sessionId, sessionKey, storePath, sessionStore, result } = params;
  
  const usage = result.meta.agentMeta?.usage;
  const modelUsed = result.meta.agentMeta?.model;
  const providerUsed = result.meta.agentMeta?.provider;
  
  // 获取现有条目
  const entry = sessionStore[sessionKey] ?? {
    sessionId,
    updatedAt: Date.now()
  };
  
  // 构建更新
  const next = {
    ...entry,
    sessionId,
    updatedAt: Date.now(),
    contextTokens: resolveContextTokensForModel({ cfg, provider: providerUsed, model: modelUsed })
  };
  
  // 设置运行时模型
  setSessionRuntimeModel(next, { provider: providerUsed, model: modelUsed });
  
  // 更新使用量
  if (hasNonzeroUsage(usage)) {
    next.inputTokens = usage.input ?? 0;
    next.outputTokens = usage.output ?? 0;
    next.totalTokens = deriveSessionTotalTokens({ usage, contextTokens: next.contextTokens });
    next.cacheRead = usage.cacheRead ?? 0;
    next.cacheWrite = usage.cacheWrite ?? 0;
  }
  
  // 更新压缩计数
  if (result.meta.agentMeta?.compactionCount > 0) {
    next.compactionCount = (entry.compactionCount ?? 0) + result.meta.agentMeta.compactionCount;
  }
  
  // 持久化
  sessionStore[sessionKey] = await updateSessionStore(storePath, (store) => {
    const merged = mergeSessionEntry(store[sessionKey], next);
    store[sessionKey] = merged;
    return merged;
  });
}
```

---

## 6. 会话新鲜度评估

### 6.1 新鲜度策略

```javascript
function evaluateSessionFreshness(params) {
  const { updatedAt, now, policy } = params;
  
  // 时间阈值
  const ageMs = now - updatedAt;
  const isTimeExpired = policy.expireAfterMs && ageMs > policy.expireAfterMs;
  
  // 通道变化
  const isChannelChanged = policy.resetOnChannelChange && 
    params.currentChannel !== params.lastChannel;
  
  // 目标变化
  const isTargetChanged = policy.resetOnTargetChange && 
    params.currentTo !== params.lastTo;
  
  const fresh = !isTimeExpired && !isChannelChanged && !isTargetChanged;
  
  return { 
    fresh, 
    reason: isTimeExpired ? "time" : isChannelChanged ? "channel" : isTargetChanged ? "target" : null 
  };
}
```

### 6.2 重置策略配置

```json
{
  "session": {
    "reset": {
      "expireAfterMinutes": 60,
      "resetOnChannelChange": true,
      "resetOnTargetChange": false
    }
  }
}
```

---

## 7. 会话锁机制

### 7.1 写锁获取

```javascript
async function acquireSessionWriteLock(params) {
  const { storePath, sessionKey, maxHoldMs } = params;
  
  const lockFile = path.join(path.dirname(storePath), `.lock-${sessionKey}.json`);
  const now = Date.now();
  
  // 尝试获取锁
  try {
    const lockData = {
      pid: process.pid,
      acquiredAt: now,
      expiresAt: now + maxHoldMs
    };
    
    fs.writeFileSync(lockFile, JSON.stringify(lockData), { flag: "wx" });
    return { ok: true, lockData };
  } catch (err) {
    if (err.code === "EEXIST") {
      // 锁已存在，检查是否过期
      const existing = JSON.parse(fs.readFileSync(lockFile, "utf-8"));
      if (now > existing.expiresAt) {
        // 锁过期，强制获取
        fs.writeFileSync(lockFile, JSON.stringify({ ...lockData, forceAcquired: true }));
        return { ok: true, lockData, forceAcquired: true };
      }
      return { ok: false, reason: "locked", holder: existing };
    }
    throw err;
  }
}
```

### 7.2 锁释放

```javascript
async function releaseSessionWriteLock(params) {
  const { storePath, sessionKey } = params;
  const lockFile = path.join(path.dirname(storePath), `.lock-${sessionKey}.json`);
  
  try {
    fs.unlinkSync(lockFile);
  } catch (err) {
    if (err.code !== "ENOENT") throw err;
  }
}
```

---

## 8. 会话归档

### 8.1 归档触发

```javascript
async function archiveSessionTranscripts(params) {
  const { sessionKey, agentId, storePath } = params;
  
  // 检查会话是否已移除
  const store = loadSessionStore(storePath);
  if (!store[sessionKey]) {
    // 会话已移除，归档转录
    const transcriptPath = resolveSessionTranscriptPath({ sessionKey, agentId });
    const archivePath = resolveArchivePath({ sessionKey, agentId });
    
    await fs.rename(transcriptPath, archivePath);
  }
}
```

### 8.2 归档清理

```javascript
async function cleanupArchivedSessionTranscripts(params) {
  const { archiveDir, ttlDays } = params;
  const now = Date.now();
  const ttlMs = ttlDays * 24 * 60 * 60 * 1000;
  
  const files = await fs.readdir(archiveDir);
  for (const file of files) {
    const stat = await fs.stat(path.join(archiveDir, file));
    if (now - stat.mtimeMs > ttlMs) {
      await fs.unlink(path.join(archiveDir, file));
    }
  }
}
```

---

## 9. 可观察性

### 9.1 管理器快照

```javascript
const snapshot = manager.getObservabilitySnapshot(cfg);

// 返回:
{
  runtimeCache: {
    activeSessions: 5,
    idleTtlMs: 3600000,  // 1 小时
    evictedTotal: 12,
    lastEvictedAt: 1710067200000
  },
  turns: {
    active: 2,
    queueDepth: 0,
    completed: 150,
    failed: 3,
    averageLatencyMs: 2500,
    maxLatencyMs: 8000
  },
  errorsByCode: {
    "ACP_TURN_FAILED": 2,
    "ACP_SESSION_INIT_FAILED": 1
  }
}
```

### 9.2 诊断命令

```bash
# 查看会话状态
openclaw sessions list --all-agents

# 查看会话详情
openclaw sessions status --session-key <key>

# 清理空闲会话
openclaw sessions cleanup --idle-hours 24
```

---

## 10. 关键设计模式

### 10.1 Actor 模型

- 每个会话一个 Actor 队列
- 操作序列化 (避免并发冲突)
- 自动清理空闲队列

### 10.2 缓存逐出

```
活跃 → 空闲 (TTL 开始) → 逐出 (TTL 到期)
  ↓        ↓              ↓
访问    无访问         关闭运行时
更新活跃时间
```

### 10.3 写锁保护

- 文件锁 (`*.lock.json`)
- 过期自动释放
- 强制获取 (死锁恢复)

---

## 11. 故障诊断

### 11.1 会话未创建

```bash
# 检查会话存储
cat ~/.openclaw/sessions/<agent>/store.json

# 检查锁文件
ls -la ~/.openclaw/sessions/<agent>/*.lock.json
```

### 11.2 运行时泄漏

```bash
# 查看活跃会话数
openclaw status --usage

# 手动逐出空闲会话
openclaw sessions cleanup --force
```

### 11.3 并发冲突

```bash
# 查看队列深度 (通过诊断 API)
curl http://localhost:18789/diagnostics

# 减少并发配置
openclaw config set acp.maxConcurrentSessions 5
```

---

## 12. 相关文档

- [会话管理](https://docs.openclaw.ai/sessions)
- [ACP 运行时](https://docs.openclaw.ai/acp/runtime)
- [会话存储](https://docs.openclaw.ai/sessions/store)

---

**阶段 2 - 会话管理** 完成 ✅

**下一步:** 通道系统 或 其他核心模块

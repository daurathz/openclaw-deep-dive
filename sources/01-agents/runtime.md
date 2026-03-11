# Agent 运行时

**阅读时间:** 2026-03-10  
**源码文件:** 
- `dist/plugin-sdk/dispatch-CJdFmoH9.js` (核心调度逻辑)
- `dist/agent-scope-CZF6h_5g.js` (Agent 作用域工具)
- `dist/agents-DXgqMt7r.js` (Agent CLI 命令)

---

## 1. Agent 执行流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│  agent 命令入口                                                  │
│  - openclaw agent --message "..." --to <target>                │
│  - openclaw agent --session-key <key> --message "..."          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  prepareAgentCommandExecution()                                 │
│  - 验证参数 (message/to/sessionKey/agentId)                     │
│  - 加载配置 + 解析密钥引用                                       │
│  - 解析会话 (resolveSession)                                    │
│  - 确保工作空间 (ensureAgentWorkspace)                          │
│  - 解析模型/思考级别/超时等配置                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  agentCommandInternal()                                         │
│  - 检查 ACP 会话状态 (acpResolution)                             │
│  - 技能快照管理 (buildWorkspaceSkillSnapshot)                   │
│  - 模型选择与覆盖 (modelOverride/providerOverride)              │
│  - 运行 Agent 尝试 (runAgentAttempt)                             │
│  - 模型回退处理 (runAgentTurnWithFallback)                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  deliverAgentCommandResult()                                    │
│  - 解析交付计划 (resolveAgentDeliveryPlan)                      │
│  - 交付消息到通道 (deliverOutboundPayloads)                     │
│  - 记录日志/返回结果                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心函数详解

### 2.1 `prepareAgentCommandExecution()` — 执行准备

**职责:** 验证参数、解析会话、准备工作空间

```javascript
async function prepareAgentCommandExecution(opts, runtime) {
  // 1. 验证必需参数
  const message = opts.message ?? "";
  if (!message.trim()) throw new Error("Message is required");
  
  if (!opts.to && !opts.sessionId && !opts.sessionKey && !opts.agentId) {
    throw new Error("Pass --to, --session-id, or --agent");
  }
  
  // 2. 加载配置并解析密钥引用
  const { resolvedConfig: cfg, diagnostics } = 
    await resolveCommandSecretRefsViaGateway({
      config: loadedRaw,
      commandName: "agent",
      targetIds: getAgentRuntimeCommandSecretTargetIds()
    });
  
  // 3. 解析 Agent ID
  const agentIdOverride = opts.agentId ? normalizeAgentId(opts.agentId) : void 0;
  
  // 4. 解析会话 (核心逻辑)
  const { sessionId, sessionKey, sessionEntry, sessionStore, storePath, 
          isNewSession, persistedThinking, persistedVerbose } = resolveSession({
    cfg, to: opts.to, sessionId: opts.sessionId, 
    sessionKey: opts.sessionKey, agentId: agentIdOverride
  });
  
  // 5. 解析工作空间
  const workspaceDirRaw = normalizedSpawned.workspaceDir ?? 
    resolveAgentWorkspaceDir(cfg, sessionAgentId);
  const workspaceDir = (await ensureAgentWorkspace({
    dir: workspaceDirRaw,
    ensureBootstrapFiles: !agentCfg?.skipBootstrap
  })).dir;
  
  // 6. 解析超时配置
  const timeoutMs = resolveAgentTimeoutMs({ cfg, overrideSeconds });
  
  // 7. 获取 ACP 会话管理器
  const acpManager = getAcpSessionManager();
  const acpResolution = sessionKey ? acpManager.resolveSession({ cfg, sessionKey }) : null;
  
  return { /* 所有准备好的参数 */ };
}
```

**关键点:**
- 密钥引用在 Gateway 层面解析 (支持动态密钥)
- 会话解析支持多种输入 (`--to`, `--session-id`, `--session-key`, `--agent`)
- 工作空间自动创建 (包含 Bootstrap 文件)
- ACP 会话状态预检查

---

### 2.2 `resolveSession()` — 会话解析

**职责:** 根据输入参数解析或创建会话

```javascript
function resolveSession(opts) {
  // 1. 解析会话键
  const { sessionKey, sessionStore, storePath } = resolveSessionKeyForRequest({
    cfg: opts.cfg, to: opts.to, sessionId: opts.sessionId,
    sessionKey: opts.sessionKey, agentId: opts.agentId
  });
  
  // 2. 检查会话是否需要重置 (基于时间/通道策略)
  const resetPolicy = resolveSessionResetPolicy({
    sessionCfg: opts.cfg.session,
    resetType: resolveSessionResetType({ sessionKey }),
    resetOverride: resolveChannelResetConfig({ channel })
  });
  
  // 3. 评估会话新鲜度
  const fresh = sessionEntry ? evaluateSessionFreshness({
    updatedAt: sessionEntry.updatedAt, now, policy: resetPolicy
  }).fresh : false;
  
  // 4. 生成会话 ID (新会话或复用)
  const sessionId = opts.sessionId?.trim() || 
    (fresh ? sessionEntry?.sessionId : void 0) || 
    crypto.randomUUID();
  
  const isNewSession = !fresh && !opts.sessionId;
  
  // 5. 处理会话轮换 (清理 Bootstrap 快照)
  clearBootstrapSnapshotOnSessionRollover({
    sessionKey,
    previousSessionId: isNewSession ? sessionEntry?.sessionId : void 0
  });
  
  return {
    sessionId, sessionKey, sessionEntry, sessionStore, storePath,
    isNewSession,
    persistedThinking: fresh ? sessionEntry?.thinkingLevel : void 0,
    persistedVerbose: fresh ? sessionEntry?.verboseLevel : void 0
  };
}
```

**会话键解析逻辑:**

```javascript
function resolveSessionKeyForRequest(opts) {
  // 优先级:
  // 1. 显式 sessionKey
  // 2. sessionId → 查找对应 sessionKey
  // 3. to (电话号码/目标) → 查找或创建会话
  // 4. agentId → 使用 main session
  
  if (opts.sessionKey) {
    return { sessionKey: opts.sessionKey, ... };
  }
  
  if (opts.sessionId) {
    const key = resolveSessionKeyFromSessionId({ cfg, sessionId: opts.sessionId });
    return { sessionKey: key, ... };
  }
  
  if (opts.to) {
    // 根据目标查找现有会话或创建新会话键
    return { sessionKey: buildSessionKeyForTarget(opts.to), ... };
  }
  
  // 默认使用 Agent 的 main session
  return { sessionKey: buildAgentMainSessionKey({ agentId }), ... };
}
```

---

### 2.3 `agentCommandInternal()` — 核心执行逻辑

**职责:** 执行 Agent 推理、处理回退、持久化结果

```javascript
async function agentCommandInternal(opts, runtime, deps) {
  const prepared = await prepareAgentCommandExecution(opts, runtime);
  const { body, cfg, sessionKey, sessionStore, storePath, 
          isNewSession, sessionAgentId, workspaceDir, runId, acpManager, acpResolution } = prepared;
  
  // 1. 检查发送策略
  if (opts.deliver === true) {
    if (resolveSendPolicy({ cfg, entry: sessionEntry, sessionKey }) === "deny") {
      throw new Error("send blocked by session policy");
    }
  }
  
  // 2. ACP 会话路径 (如果 ACP 就绪)
  if (acpResolution?.kind === "ready" && sessionKey) {
    const startedAt = Date.now();
    registerAgentRunContext(runId, { sessionKey });
    
    emitAgentEvent({ runId, stream: "lifecycle", data: { phase: "start", startedAt } });
    
    // 运行 ACP turn
    await acpManager.runTurn({
      cfg, sessionKey, text: body, mode: "prompt", requestId: runId,
      onEvent: (event) => {
        if (event.type === "text_delta") {
          emitAgentEvent({ runId, stream: "assistant", data: { text, delta } });
        }
      }
    });
    
    emitAgentEvent({ runId, stream: "lifecycle", data: { phase: "end" } });
    
    // 持久化转录
    sessionEntry = await persistAcpTurnTranscript({ body, finalText, sessionKey, ... });
    
    // 交付结果
    return await deliverAgentCommandResult({ cfg, deps, runtime, opts, ... });
  }
  
  // 3. 传统路径 (非 ACP)
  // 3.1 技能快照管理
  const needsSkillsSnapshot = isNewSession || !sessionEntry?.skillsSnapshot;
  if (needsSkillsSnapshot) {
    const skillsSnapshot = buildWorkspaceSkillSnapshot(workspaceDir, { config: cfg, ... });
    // 持久化技能快照到会话
  }
  
  // 3.2 模型选择
  const configuredDefaultRef = resolveDefaultModelForAgent({ cfg, agentId: sessionAgentId });
  let provider = defaultProvider, model = defaultModel;
  
  // 检查会话覆盖
  if (sessionEntry?.modelOverride) {
    const key = modelKey(provider, model);
    if (allowedModelKeys.has(key)) {
      provider = sessionEntry.providerOverride || defaultProvider;
      model = sessionEntry.modelOverride;
    }
  }
  
  // 3.3 运行 Agent (支持回退)
  const result = await runAgentTurnWithFallback({
    body, cfg, sessionKey, sessionFile, provider, model,
    agentCfg: prepared.agentCfg, runId, timeoutMs, ...
  });
  
  // 3.4 更新会话存储 (使用量/模型信息)
  await updateSessionStoreAfterAgentRun({ cfg, sessionId, sessionKey, result, ... });
  
  // 3.5 交付结果
  return await deliverAgentCommandResult({ cfg, deps, runtime, opts, result, ... });
}
```

---

### 2.4 `runAgentTurnWithFallback()` — 带模型回退的执行

**职责:** 运行 Agent 推理，主模型失败时自动回退

```javascript
async function runAgentTurnWithFallback(params) {
  const { cfg, sessionKey, provider, model, agentCfg } = params;
  
  // 1. 解析主模型和回退模型
  const primaryModel = { provider, model };
  const fallbacks = resolveEffectiveModelFallbacks({
    cfg, agentId: params.sessionAgentId,
    runFallbacks: params.runFallbacks,
    agentFallbacks: agentCfg?.model?.fallbacks
  });
  
  // 2. 尝试主模型
  try {
    return await runAgentAttempt({
      ...params,
      provider: primaryModel.provider,
      model: primaryModel.model,
      isFallbackRetry: false
    });
  } catch (primaryError) {
    // 3. 主模型失败，尝试回退
    if (fallbacks.length === 0) throw primaryError;
    
    for (const fallback of fallbacks) {
      try {
        const result = await runAgentAttempt({
          ...params,
          provider: fallback.provider,
          model: fallback.model,
          isFallbackRetry: true
        });
        
        // 记录回退通知
        recordFallbackNotice({ sessionEntry, primaryModel, fallbackModel: fallback });
        return result;
      } catch (fallbackError) {
        // 继续尝试下一个回退
        continue;
      }
    }
    
    // 所有回退都失败
    throw primaryError;
  }
}
```

---

### 2.5 `deliverAgentCommandResult()` — 结果交付

**职责:** 解析交付目标、发送消息到通道

```javascript
async function deliverAgentCommandResult(params) {
  const { cfg, opts, outboundSession, sessionEntry, payloads, result } = params;
  
  // 1. 解析交付计划
  const deliveryPlan = resolveAgentDeliveryPlan({
    sessionEntry,
    requestedChannel: opts.replyChannel ?? opts.channel,
    explicitTo: opts.replyTo ?? opts.to,
    wantsDelivery: opts.deliver === true,
    turnSourceChannel: opts.runContext?.messageChannel,
    turnSourceTo: opts.runContext?.currentChannelId,
    turnSourceAccountId: opts.runContext?.accountId
  });
  
  // 2. 解析交付通道 (可能根据配置覆盖)
  let deliveryChannel = deliveryPlan.resolvedChannel;
  if (deliver && isInternalMessageChannel(deliveryChannel) && !explicitChannelHint) {
    deliveryChannel = (await resolveMessageChannelSelection({ cfg })).channel;
  }
  
  // 3. 解析交付目标
  const resolved = deliver && isDeliveryChannelKnown ? 
    resolveAgentOutboundTarget({ cfg, plan: deliveryPlan, targetMode }) : 
    { resolvedTarget: null, targetMode };
  
  // 4. 交付消息
  if (deliver && deliveryChannel && !isInternalMessageChannel(deliveryChannel)) {
    if (deliveryTarget) {
      await deliverOutboundPayloads({
        cfg, channel: deliveryChannel, to: deliveryTarget,
        accountId: resolvedAccountId, payloads: deliveryPayloads,
        session: outboundSession, replyToId: resolvedReplyToId,
        threadId: resolvedThreadTarget, ...
      });
    }
  }
  
  return { payloads: normalizedPayloads, meta: result.meta };
}
```

---

## 3. 会话管理

### 3.1 会话键格式

```
# Main session (Agent 主会话)
agent:<agentId>:main

# Peer session (特定通道/目标的会话)
agent:<agentId>:<mainKey>:<channel>:<target>

# 简化格式 (默认 agent)
<mainKey>
<mainKey>:<channel>:<target>
```

**示例:**
- `agent:peter:main` — Peter 的主会话
- `agent:peter:default:slack:U123456` — Peter 与 Slack 用户 U123456 的会话
- `main` — 默认 Agent 的主会话

### 3.2 会话新鲜度评估

```javascript
function evaluateSessionFreshness(params) {
  const { updatedAt, now, policy } = params;
  
  // 检查时间阈值
  const ageMs = now - updatedAt;
  const isTimeExpired = policy.expireAfterMs && ageMs > policy.expireAfterMs;
  
  // 检查通道变化
  const isChannelChanged = policy.resetOnChannelChange && 
    params.currentChannel !== params.lastChannel;
  
  // 检查目标变化
  const isTargetChanged = policy.resetOnTargetChange && 
    params.currentTo !== params.lastTo;
  
  const fresh = !isTimeExpired && !isChannelChanged && !isTargetChanged;
  
  return { fresh, reason: isTimeExpired ? "time" : isChannelChanged ? "channel" : null };
}
```

**重置策略配置:**

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

### 3.3 会话存储结构

```javascript
{
  "agent:peter:main": {
    "sessionId": "uuid-1234",
    "updatedAt": 1710067200000,
    "channel": "webchat",
    "to": null,
    "thinkingLevel": "medium",
    "verboseLevel": "on",
    "providerOverride": "openai",
    "modelOverride": "gpt-4o",
    "inputTokens": 1500,
    "outputTokens": 800,
    "totalTokens": 2300,
    "skillsSnapshot": { /* 技能快照 */ },
    "systemPromptReport": { /* 系统提示报告 */ }
  }
}
```

---

## 4. 技能快照

### 4.1 为什么需要技能快照

技能快照记录会话创建时的工作空间技能状态，确保:
- 会话期间技能行为一致 (即使工作空间技能变化)
- 支持技能变更后的会话隔离
- 便于调试和审计

### 4.2 快照构建

```javascript
function buildWorkspaceSkillSnapshot(workspaceDir, params) {
  const { config, eligibility, snapshotVersion, skillFilter } = params;
  
  // 1. 扫描工作空间技能文件
  const skillEntries = loadWorkspaceSkillEntries(workspaceDir, { config, skillFilter });
  
  // 2. 检查远程技能资格
  const remoteEligibility = eligibility?.remote ?? getRemoteSkillEligibility();
  
  // 3. 构建快照
  const snapshot = {
    version: snapshotVersion,
    skills: skillEntries.map(entry => ({
      name: entry.name,
      path: entry.path,
      hash: entry.hash,
      enabled: entry.enabled,
      tools: entry.tools?.map(t => t.name)
    })),
    capturedAt: Date.now()
  };
  
  return snapshot;
}
```

---

## 5. 模型选择与回退

### 5.1 模型优先级

```
1. 会话覆盖 (sessionEntry.modelOverride/providerOverride)
2. Agent 配置 (agents.defaults.model)
3. 全局默认 (DEFAULT_PROVIDER/DEFAULT_MODEL)
```

### 5.2 回退配置

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai:gpt-4o",
        "fallbacks": [
          { "provider": "anthropic", "model": "claude-sonnet-4-5-20250929" },
          { "provider": "openai", "model": "gpt-4o-mini" }
        ]
      }
    }
  }
}
```

### 5.3 回退触发条件

- 模型 API 超时
- 模型 API 返回错误 (5xx, 429)
- 上下文溢出错误
- 计费错误

---

## 6. ACP 运行时集成

### 6.1 ACP 会话解析

```javascript
const acpResolution = acpManager.resolveSession({ cfg, sessionKey });

// 返回类型:
// { kind: "ready", meta: { agent, workspace, ... } }
// { kind: "stale", error: Error }
// { kind: "pending" }
```

### 6.2 ACP Turn 执行

```javascript
await acpManager.runTurn({
  cfg, sessionKey, text: body, mode: "prompt", requestId: runId,
  signal: opts.abortSignal,
  onEvent: (event) => {
    if (event.type === "done") {
      stopReason = event.stopReason;
    }
    if (event.type === "text_delta" && event.stream === "output") {
      emitAgentEvent({ runId, stream: "assistant", data: { text, delta } });
    }
  }
});
```

### 6.3 ACP 转录持久化

```javascript
async function persistAcpTurnTranscript(params) {
  const { body, finalText, sessionId, sessionKey, sessionStore, storePath } = params;
  
  // 1. 解析会话转录文件
  const { sessionFile, sessionEntry } = await resolveSessionTranscriptFile({
    sessionId, sessionKey, sessionEntry, sessionStore, storePath
  });
  
  // 2. 打开会话管理器
  const sessionManager = SessionManager.open(sessionFile);
  await prepareSessionManagerForRun({ sessionManager, sessionFile, sessionId });
  
  // 3. 追加消息
  if (body) sessionManager.appendMessage({
    role: "user", content: body, timestamp: Date.now()
  });
  if (finalText) sessionManager.appendMessage({
    role: "assistant", content: [{ type: "text", text: finalText }],
    api: "openai-responses", provider: "openclaw", model: "acp-runtime",
    usage: ACP_TRANSCRIPT_USAGE, stopReason: "stop", timestamp: Date.now()
  });
  
  // 4. 发出更新事件
  emitSessionTranscriptUpdate(sessionFile);
  
  return sessionEntry;
}
```

---

## 7. 关键设计模式

### 7.1 会话隔离

- 每个会话独立存储 (按 `sessionKey` 键)
- 会话间不共享上下文 (除非显式 `sessions_send`)
- 技能快照确保会话期间技能一致性

### 7.2 渐进式回退

```
主模型 → 回退 1 → 回退 2 → ... → 抛出错误
   ↓        ↓        ↓
 失败     失败     失败
```

**优势:**
- 提高任务完成率
- 自动规避模型故障
- 记录回退通知便于审计

### 7.3 交付计划分离

```
Agent 推理 → 生成 payloads → 解析交付计划 → 执行交付
                ↓                ↓              ↓
          与通道无关      根据配置解析    调用通道插件
```

**优势:**
- Agent 逻辑与通道解耦
- 支持动态交付目标
- 便于测试和调试

---

## 8. 故障诊断

### 8.1 会话未创建

```bash
# 检查会话键格式
openclaw sessions list --all-agents

# 检查会话存储路径
openclaw config get session.store
```

### 8.2 模型回退频繁

```bash
# 检查主模型状态
openclaw models status --probe-provider openai

# 查看回退通知
openclaw sessions status --session-key <key>
```

### 8.3 技能未加载

```bash
# 检查工作空间技能
ls -la <workspace>/skills/

# 刷新技能缓存
openclaw skills status
```

---

## 9. 相关文档

- [Agent 命令](https://docs.openclaw.ai/cli/agent)
- [会话管理](https://docs.openclaw.ai/sessions)
- [模型回退](https://docs.openclaw.ai/models/fallbacks)
- [技能系统](https://docs.openclaw.ai/skills)

---

**下一步:** 消息路由 (`dist/plugin-sdk/dispatch-*.js`) — 消息如何从通道到 Agent 再到模型

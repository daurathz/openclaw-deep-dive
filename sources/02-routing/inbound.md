# 消息路由系统

**阅读时间:** 2026-03-10  
**源码文件:** 
- `dist/plugin-sdk/dispatch-CJdFmoH9.js` (核心调度逻辑)
- `dist/plugin-sdk/reply-*.js` (回复处理)

---

## 1. 消息路由流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│  通道插件接收消息 (Slack/Telegram/Discord/...)                   │
│  - 标准化消息格式                                                │
│  - 提取上下文 (Surface/Provider/To/From/SessionKey)              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  finalizeInboundContext()                                       │
│  - 标准化文本 (换行/系统标签清理)                                │
│  - 解析聊天类型 (direct/group)                                  │
│  - 处理媒体类型和计数                                            │
│  - 生成 BodyForAgent / BodyForCommands                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  resolveAgentRoute() — 核心路由决策                              │
│  - 匹配通道绑定 (bindings)                                      │
│  - 优先级：peer > guild+roles > guild > team > account > default │
│  - 返回：agentId, sessionKey, mainSessionKey                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  recordInboundSession()                                         │
│  - 记录会话元数据 (通道/目标/账户)                               │
│  - 更新 lastRoute (用于回复路由)                                │
│  - 创建会话 (如果不存在)                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  dispatchReplyFromConfig()                                      │
│  - 检查发送策略 (send policy)                                   │
│  - 触发插件钩子 (message_received)                              │
│  - 尝试 ACP 回复 (tryDispatchAcpReply)                            │
│  - 或传统回复 (getReplyFromConfig)                              │
│  - 处理工具结果/块回复/最终回复                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  回复交付                                                        │
│  - dispatcher.sendToolResult() / sendBlockReply() / sendFinalReply() │
│  - 或 routeReply() (路由回复到原始通道)                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心函数详解

### 2.1 `finalizeInboundContext()` — 上下文标准化

**职责:** 清理和标准化入站消息上下文

```javascript
function finalizeInboundContext(ctx, opts = {}) {
  // 1. 文本标准化
  ctx.Body = sanitizeInboundSystemTags(normalizeInboundTextNewlines(ctx.Body));
  ctx.RawBody = normalizeTextField(ctx.RawBody);
  ctx.CommandBody = normalizeTextField(ctx.CommandBody);
  
  // 2. 聊天类型标准化
  const chatType = normalizeChatType(ctx.ChatType);
  if (chatType) ctx.ChatType = chatType;
  
  // 3. 生成 Agent 用消息体
  ctx.BodyForAgent = sanitizeInboundSystemTags(
    normalizeInboundTextNewlines(
      opts.forceBodyForAgent ? ctx.Body : 
      ctx.BodyForAgent ?? ctx.CommandBody ?? ctx.RawBody ?? ctx.Body
    )
  );
  
  // 4. 生成命令用消息体
  ctx.BodyForCommands = sanitizeInboundSystemTags(
    normalizeInboundTextNewlines(
      opts.forceBodyForCommands ? ctx.CommandBody ?? ctx.RawBody ?? ctx.Body :
      ctx.BodyForCommands ?? ctx.CommandBody ?? ctx.RawBody ?? ctx.Body
    )
  );
  
  // 5. 解析会话标签
  const explicitLabel = ctx.ConversationLabel?.trim();
  if (opts.forceConversationLabel || !explicitLabel) {
    const resolved = resolveConversationLabel(ctx)?.trim();
    if (resolved) ctx.ConversationLabel = resolved;
  }
  
  // 6. 媒体类型标准化
  const mediaCount = countMediaEntries(ctx);
  if (mediaCount > 0) {
    const mediaType = normalizeMediaType(ctx.MediaType);
    const normalizedMediaTypes = ctx.MediaTypes?.map(entry => normalizeMediaType(entry));
    
    // 填充默认媒体类型
    let mediaTypesFinal;
    if (normalizedMediaTypes?.length > 0) {
      const filled = normalizedMediaTypes.slice();
      while (filled.length < mediaCount) filled.push(DEFAULT_MEDIA_TYPE);
      mediaTypesFinal = filled;
    } else if (mediaType) {
      mediaTypesFinal = [mediaType];
      while (mediaTypesFinal.length < mediaCount) 
        mediaTypesFinal.push(DEFAULT_MEDIA_TYPE);
    } else {
      mediaTypesFinal = Array.from({ length: mediaCount }, () => DEFAULT_MEDIA_TYPE);
    }
    
    ctx.MediaTypes = mediaTypesFinal;
    ctx.MediaType = mediaType ?? mediaTypesFinal[0] ?? DEFAULT_MEDIA_TYPE;
  }
  
  return ctx;
}
```

**标准化内容:**
- **系统标签清理** — 移除 `[[reply_to:...]]` 等内部标签
- **换行标准化** — 统一换行符格式
- **消息体分离** — `BodyForAgent` (给模型) vs `BodyForCommands` (给命令)
- **媒体类型推断** — 根据 MIME 类型或扩展名推断

---

### 2.2 `resolveAgentRoute()` — Agent 路由决策

**职责:** 根据消息上下文匹配最合适的 Agent

```javascript
function resolveAgentRoute(input) {
  const { channel, accountId, peer, guildId, teamId, memberRoleIds, cfg } = input;
  
  // 1. 获取通道 - 账户绑定
  const bindings = getEvaluatedBindingsForChannelAccount(cfg, channel, accountId);
  const bindingsIndex = getEvaluatedBindingIndexForChannelAccount(cfg, channel, accountId);
  
  // 2. 构建选择函数
  const choose = (agentId, matchedBy) => {
    const resolvedAgentId = pickFirstExistingAgentId(cfg, agentId);
    const sessionKey = buildAgentSessionKey({
      agentId: resolvedAgentId, channel, accountId, peer, dmScope, identityLinks
    }).toLowerCase();
    const mainSessionKey = buildAgentMainSessionKey({
      agentId: resolvedAgentId, mainKey: DEFAULT_MAIN_KEY
    }).toLowerCase();
    
    return {
      agentId: resolvedAgentId, channel, accountId, sessionKey, mainSessionKey,
      lastRoutePolicy: deriveLastRoutePolicy({ sessionKey, mainSessionKey }),
      matchedBy
    };
  };
  
  // 3. 优先级匹配 (tiered matching)
  const tiers = [
    {
      matchedBy: "binding.peer",
      enabled: Boolean(peer),
      candidates: collectPeerIndexedBindings(bindingsIndex, peer),
      predicate: (c) => c.match.peer.state === "valid"
    },
    {
      matchedBy: "binding.peer.parent",
      enabled: Boolean(parentPeer?.id),
      candidates: collectPeerIndexedBindings(bindingsIndex, parentPeer),
      predicate: (c) => c.match.peer.state === "valid"
    },
    {
      matchedBy: "binding.guild+roles",
      enabled: Boolean(guildId && memberRoleIds.length > 0),
      candidates: guildId ? bindingsIndex.byGuildWithRoles.get(guildId) ?? [] : [],
      predicate: (c) => hasGuildConstraint(c.match) && hasRolesConstraint(c.match)
    },
    {
      matchedBy: "binding.guild",
      enabled: Boolean(guildId),
      candidates: guildId ? bindingsIndex.byGuild.get(guildId) ?? [] : [],
      predicate: (c) => hasGuildConstraint(c.match) && !hasRolesConstraint(c.match)
    },
    {
      matchedBy: "binding.team",
      enabled: Boolean(teamId),
      candidates: teamId ? bindingsIndex.byTeam.get(teamId) ?? [] : [],
      predicate: (c) => hasTeamConstraint(c.match)
    },
    {
      matchedBy: "binding.account",
      enabled: true,
      candidates: bindingsIndex.byAccount,
      predicate: (c) => c.match.accountPattern !== "*"
    },
    {
      matchedBy: "binding.channel",
      enabled: true,
      candidates: bindingsIndex.byChannel,
      predicate: (c) => c.match.accountPattern === "*"
    }
  ];
  
  // 4. 按优先级匹配
  for (const tier of tiers) {
    if (!tier.enabled) continue;
    const matched = tier.candidates.find((candidate) => 
      tier.predicate(candidate) && 
      matchesBindingScope(candidate.match, { ...baseScope, peer: tier.scopePeer })
    );
    if (matched) {
      return choose(matched.binding.agentId, tier.matchedBy);
    }
  }
  
  // 5. 默认 Agent
  return choose(resolveDefaultAgentId(cfg), "default");
}
```

**匹配优先级:**
```
1. binding.peer        — 精确匹配对话对象 (DM/群聊 ID)
2. binding.peer.parent — 匹配父级对话 (如 Discord 频道父分类)
3. binding.guild+roles — Discord 服务器 + 角色匹配
4. binding.guild       — Discord 服务器匹配
5. binding.team        — Slack 团队匹配
6. binding.account     — 账户级别匹配
7. binding.channel     — 通道级别默认匹配
8. default             — 默认 Agent
```

**返回结构:**
```javascript
{
  agentId: "peter",
  channel: "slack",
  accountId: "T123456",
  sessionKey: "agent:peter:slack:T123456:U654321",
  mainSessionKey: "agent:peter:main",
  lastRoutePolicy: { /* 最后路由策略 */ },
  matchedBy: "binding.peer"
}
```

---

### 2.3 `recordInboundSession()` — 会话记录

**职责:** 记录入站消息的会话元数据

```javascript
async function recordInboundSession(params) {
  const { storePath, sessionKey, ctx, groupResolution, createIfMissing } = params;
  const canonicalSessionKey = normalizeSessionStoreKey(sessionKey);
  
  // 1. 记录会话元数据
  recordSessionMetaFromInbound({
    storePath,
    sessionKey: canonicalSessionKey,
    ctx,
    groupResolution,
    createIfMissing
  }).catch(params.onRecordError);
  
  // 2. 更新最后路由 (用于回复路由)
  const update = params.updateLastRoute;
  if (!update) return;
  
  const targetSessionKey = normalizeSessionStoreKey(update.sessionKey);
  await updateLastRoute({
    storePath,
    sessionKey: targetSessionKey,
    deliveryContext: {
      channel: update.channel,
      to: update.to,
      accountId: update.accountId,
      threadId: update.threadId
    },
    ctx: targetSessionKey === canonicalSessionKey ? ctx : void 0,
    groupResolution
  });
}
```

**会话元数据结构:**
```javascript
{
  sessionId: "uuid-1234",
  updatedAt: 1710067200000,
  channel: "slack",
  to: "U654321",
  accountId: "T123456",
  threadId: "1234567890.123456",
  chatType: "direct",
  lastChannel: "slack",
  lastTo: "U654321",
  lastAccountId: "T123456"
}
```

---

### 2.4 `dispatchReplyFromConfig()` — 回复调度

**职责:** 调度 Agent 回复到消息通道

```javascript
async function dispatchReplyFromConfig(params) {
  const { ctx, cfg, dispatcher } = params;
  const sessionKey = ctx.SessionKey;
  const channel = String(ctx.Surface ?? ctx.Provider ?? "unknown").toLowerCase();
  
  // 1. 诊断日志 (如果启用)
  const diagnosticsEnabled = isDiagnosticsEnabled(cfg);
  const recordProcessed = (outcome, opts) => { /* 记录处理结果 */ };
  const markProcessing = () => { /* 标记会话处理中 */ };
  const markIdle = (reason) => { /* 标记会话空闲 */ };
  
  // 2. 检查重复消息
  if (shouldSkipDuplicateInbound(ctx)) {
    recordProcessed("skipped", { reason: "duplicate" });
    return { queuedFinal: false, counts: dispatcher.getQueuedCounts() };
  }
  
  // 3. 触发插件钩子
  const hookContext = deriveInboundMessageHookContext(ctx, { messageId });
  if (hookRunner?.hasHooks("message_received")) {
    fireAndForgetHook(hookRunner.runMessageReceived(event, context));
  }
  
  // 4. 检查发送策略
  if (resolveSendPolicy({ cfg, entry, sessionKey, channel, chatType }) === "deny") {
    recordProcessed("completed", { reason: "send_policy_deny" });
    markIdle("message_completed");
    return { queuedFinal: false, counts: dispatcher.getQueuedCounts() };
  }
  
  // 5. 尝试 ACP 回复
  const acpDispatch = await tryDispatchAcpReply({
    ctx, cfg, dispatcher, sessionKey, inboundAudio, sessionTtsAuto,
    ttsChannel, shouldRouteToOriginating, originatingChannel, originatingTo,
    shouldSendToolSummaries, bypassForCommand, onReplyStart, recordProcessed, markIdle
  });
  if (acpDispatch) return acpDispatch;
  
  // 6. 传统回复路径
  const replyResult = await getReplyFromConfig(ctx, replyOptions, cfg);
  
  // 7. 处理回复
  const replies = Array.isArray(replyResult) ? replyResult : [replyResult];
  for (const reply of replies) {
    if (reply.type === "tool_result") {
      dispatcher.sendToolResult(payload);
    } else if (reply.type === "block") {
      dispatcher.sendBlockReply(payload);
    } else if (reply.type === "final") {
      dispatcher.sendFinalReply(payload);
    }
  }
  
  recordProcessed("completed");
  markIdle("message_completed");
  return { queuedFinal: true, counts: dispatcher.getQueuedCounts() };
}
```

**回复类型:**
- **tool_result** — 工具调用结果 (立即发送)
- **block** — 块回复 (流式/分批发送)
- **final** — 最终回复 (消息结束标记)

---

## 3. 路由缓存

### 3.1 缓存键构建

```javascript
function buildResolvedRouteCacheKey(params) {
  const { channel, accountId, peer, parentPeer, guildId, teamId, memberRoleIds, dmScope } = params;
  
  return JSON.stringify({
    channel,
    accountId,
    peer: peer ? `${peer.kind}:${peer.id}` : null,
    parentPeer: parentPeer ? `${parentPeer.kind}:${parentPeer.id}` : null,
    guildId,
    teamId,
    memberRoleIds: memberRoleIds.sort().join(","),
    dmScope
  });
}
```

### 3.2 缓存失效

```javascript
const MAX_RESOLVED_ROUTE_CACHE_KEYS = 1000;

if (routeCache && routeCacheKey) {
  routeCache.set(routeCacheKey, route);
  if (routeCache.size > MAX_RESOLVED_ROUTE_CACHE_KEYS) {
    routeCache.clear();  // 缓存过大时清空
    routeCache.set(routeCacheKey, route);
  }
}
```

---

## 4. 会话键构建

### 4.1 Peer 会话键

```javascript
function buildAgentSessionKey(params) {
  const { agentId, channel, accountId, peer, dmScope, identityLinks } = params;
  
  // 格式: agent:<agentId>:<mainKey>:<channel>:<accountId>:<peerKind>:<peerId>
  const parts = ["agent", agentId, DEFAULT_MAIN_KEY, channel];
  
  if (accountId) parts.push(accountId);
  if (peer?.kind && peer?.id) parts.push(peer.kind, peer.id);
  
  return parts.join(":");
}
```

### 4.2 Main 会话键

```javascript
function buildAgentMainSessionKey(params) {
  const { agentId, mainKey = DEFAULT_MAIN_KEY } = params;
  return `agent:${agentId}:${mainKey}`;
}
```

**示例:**
- `agent:peter:main` — Peter 主会话
- `agent:peter:main:slack:T123456:dm:U654321` — Peter 与 Slack 用户 U654321 的 DM 会话
- `agent:peter:main:discord:C789012` — Peter 与 Discord 频道 C789012 的会话

---

## 5. 回复路由

### 5.1 路由回复到原始通道

```javascript
const shouldRouteToOriginating = Boolean(
  !isInternalWebchatTurn && 
  isRoutableChannel(originatingChannel) && 
  originatingTo && 
  originatingChannel !== currentSurface
);

if (shouldRouteToOriginating) {
  await routeReply({
    payload,
    channel: originatingChannel,
    to: originatingTo,
    sessionKey: ctx.SessionKey,
    accountId: ctx.AccountId,
    threadId: ctx.MessageThreadId,
    cfg,
    abortSignal,
    mirror: false,
    isGroup,
    groupId
  });
} else {
  dispatcher.sendFinalReply(payload);
}
```

**路由场景:**
- Webhook 入站 → 回复到 Webhook 目标
- 跨通道消息 → 回复到原始通道
- 内部 Webchat → 直接发送

---

## 6. TTS 集成

### 6.1 TTS 自动模式

```javascript
const sessionTtsAuto = normalizeTtsAutoMode(sessionStoreEntry.entry?.ttsAuto);

// TTS 通道选择
const ttsChannel = shouldRouteToOriginating ? originatingChannel : currentSurface;

// 应用 TTS 到负载
const ttsPayload = await maybeApplyTtsToPayload({
  payload,
  cfg,
  channel: ttsChannel,
  kind: "block",  // "tool" | "block" | "final"
  inboundAudio,
  ttsAuto: sessionTtsAuto
});
```

### 6.2 TTS 自动模式

- `on` — 始终启用 TTS
- `off` — 始终禁用 TTS
- `audio-in` — 仅当入站消息包含音频时启用
- `smart` — 根据上下文智能决策

---

## 7. 钩子系统

### 7.1 插件钩子

```javascript
const hookContext = deriveInboundMessageHookContext(ctx, { messageId });

if (hookRunner?.hasHooks("message_received")) {
  fireAndForgetHook(
    hookRunner.runMessageReceived(toPluginMessageReceivedEvent(hookContext), 
                                   toPluginMessageContext(hookContext)),
    "dispatch-from-config: message_received plugin hook failed"
  );
}
```

### 7.2 内部钩子

```javascript
if (sessionKey) {
  fireAndForgetHook(
    triggerInternalHook(createInternalHookEvent("message", "received", sessionKey, {
      ...toInternalMessageReceivedContext(hookContext),
      timestamp
    })),
    "dispatch-from-config: message_received internal hook failed"
  );
}
```

**钩子类型:**
- `message_received` — 消息接收
- `message_sent` — 消息发送
- `session_created` — 会话创建
- `session_reset` — 会话重置

---

## 8. 诊断与日志

### 8.1 消息处理日志

```javascript
const recordProcessed = (outcome, opts) => {
  if (!diagnosticsEnabled) return;
  
  logMessageProcessed({
    channel,
    chatId,
    messageId,
    sessionKey,
    durationMs: Date.now() - startTime,
    outcome,  // "completed" | "skipped" | "error"
    reason: opts?.reason,  // "duplicate" | "send_policy_deny" | "fast_abort"
    error: opts?.error
  });
};
```

### 8.2 会话状态跟踪

```javascript
const markProcessing = () => {
  if (!canTrackSession || !sessionKey) return;
  
  logMessageQueued({ sessionKey, channel, source: "dispatch" });
  logSessionStateChange({ sessionKey, state: "processing", reason: "message_start" });
};

const markIdle = (reason) => {
  if (!canTrackSession || !sessionKey) return;
  logSessionStateChange({ sessionKey, state: "idle", reason });
};
```

---

## 9. 关键设计模式

### 9.1 分层匹配

```
peer (精确) → guild+roles → guild → team → account → channel → default
   ↓            ↓            ↓       ↓       ↓         ↓         ↓
 最具体                                    较通用    最通用
```

**优势:**
- 精确匹配优先
- 支持细粒度路由控制
- 回退到默认 Agent

### 9.2 回复路由分离

```
入站通道 ≠ 出站通道 (可能)
   ↓            ↓
记录 lastRoute → 回复时查找
```

**优势:**
- 支持跨通道交互
- 保持对话上下文
- 灵活的路由策略

### 9.3 流式回复

```
tool_result (立即) → block (流式) → final (结束)
      ↓                ↓              ↓
   工具结果        模型输出流      消息完成
```

**优势:**
- 低延迟工具反馈
- 流式用户体验
- 明确的消息边界

---

## 10. 故障诊断

### 10.1 消息未路由

```bash
# 检查绑定配置
openclaw config get channels.<channel>.allowFrom

# 检查会话键
openclaw sessions list --all-agents

# 查看路由日志
openclaw status --verbose
```

### 10.2 回复未发送

```bash
# 检查发送策略
openclaw config get session.reset

# 检查通道状态
openclaw channels status
```

### 10.3 会话未创建

```bash
# 检查会话存储路径
openclaw config get session.store

# 手动创建会话
openclaw agent --to <target> --message "test"
```

---

## 11. 相关文档

- [通道配置](https://docs.openclaw.ai/channels/config)
- [会话管理](https://docs.openclaw.ai/sessions)
- [插件钩子](https://docs.openclaw.ai/plugins/hooks)
- [TTS 配置](https://docs.openclaw.ai/tts)

---

**下一步:** 工具执行系统 — exec 工具的安全模型和审批流程

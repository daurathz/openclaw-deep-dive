# 通道系统 — Channel Plugin Architecture

**阅读时间:** 2026-03-10  
**源码文件:** 
- `dist/plugins-v0UiRqAc.js` (通道插件注册)
- `dist/plugin-sdk/plugins-*.js` (通道插件实现)
- `dist/channel-*.js` (通道配置与选择)

---

## 1. 通道系统概览

```
┌─────────────────────────────────────────────────────────────────┐
│  通道插件 (Channel Plugins)                                      │
│  - Slack: 消息/线程/反应/文件                                    │
│  - Discord: 频道/服务器/反应/线程                                │
│  - Telegram: 私聊/群组/频道                                     │
│  - WhatsApp: 私聊/群组                                          │
│  - SMS/iMessage: 短信                                           │
│  - Webchat: Web 界面                                            │
│  - Email: 邮件收发                                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  通道管理器 (ChannelManager)                                     │
│  - 加载通道插件                                                  │
│  - 管理账户绑定                                                  │
│  - 处理入站/出站消息                                             │
│  - 健康检查与状态监控                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Gateway WebSocket                                              │
│  - 广播通道事件                                                  │
│  - 接收通道命令                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 通道插件接口

### 2.1 插件结构

```javascript
{
  id: "slack",                    // 通道 ID
  name: "Slack",                  // 显示名称
  meta: {
    preferSessionLookupForAnnounceTarget: true
  },
  gatewayMethods: [               // Gateway RPC 方法
    "slack.send",
    "slack.history",
    "slack.reactions.add"
  ],
  accountConfigSchema: {          // 账户配置 Schema
    type: "object",
    properties: {
      token: { type: "string" },
      botUserId: { type: "string" }
    }
  },
  createRuntime: (cfg, accountId, logger) => ({
    // 运行时工厂
    start: async () => { ... },
    stop: async () => { ... },
    send: async (params) => { ... }
  })
}
```

### 2.2 插件注册

```javascript
function listChannelPlugins() {
  return [
    { id: "slack", name: "Slack", ... },
    { id: "discord", name: "Discord", ... },
    { id: "telegram", name: "Telegram", ... },
    { id: "whatsapp", name: "WhatsApp", ... },
    { id: "imessage", name: "iMessage", ... },
    { id: "webchat", name: "Webchat", ... }
  ];
}

function getChannelPlugin(id) {
  const resolvedId = String(id).trim();
  if (!resolvedId) return;
  return resolveCachedChannelPlugins().byId.get(resolvedId);
}
```

---

## 3. 通道管理器

### 3.1 管理器结构

```javascript
class ChannelManager {
  constructor(params) {
    this.loadConfig = params.loadConfig;
    this.channelLogs = params.channelLogs;       // 每通道日志
    this.channelRuntimeEnvs = params.channelRuntimeEnvs;
    this.channelRuntime = params.channelRuntime; // 插件运行时
    this.activeChannels = new Map();             // 活跃通道
  }
  
  // 获取运行时快照
  getRuntimeSnapshot() {
    const cfg = this.loadConfig();
    const channels = listChannelPlugins();
    const accounts = {};
    
    for (const channel of channels) {
      const runtime = this.channelRuntime.channel.create(channel, cfg);
      accounts[channel.id] = runtime.listAccounts(cfg);
    }
    
    return { channels, accounts };
  }
  
  // 启动通道
  async startChannel(params) {
    const { channelId, accountId, cfg } = params;
    const plugin = getChannelPlugin(channelId);
    
    if (!plugin) throw new Error(`Unknown channel: ${channelId}`);
    
    const runtime = plugin.createRuntime(cfg, accountId, this.channelLogs[channelId]);
    await runtime.start();
    
    this.activeChannels.set(`${channelId}:${accountId}`, runtime);
  }
  
  // 停止通道
  async stopChannel(params) {
    const { channelId, accountId } = params;
    const key = `${channelId}:${accountId}`;
    const runtime = this.activeChannels.get(key);
    
    if (runtime) {
      await runtime.stop();
      this.activeChannels.delete(key);
    }
  }
  
  // 标记通道登出
  markChannelLoggedOut(params) {
    const { channelId, accountId } = params;
    // 清理认证状态
  }
}
```

---

## 4. 通道配置

### 4.1 配置结构

```json
{
  "channels": {
    "slack": {
      "accounts": {
        "T123456": {
          "token": "xoxb-...",
          "botUserId": "U123456",
          "enabled": true
        }
      },
      "allowFrom": [
        { "channel": "C123", "agentId": "peter" }
      ],
      "dmPolicy": {
        "default": "allow",
        "allowFrom": []
      },
      "groupPolicy": {
        "default": "allow",
        "routeAllowlist": []
      }
    },
    "discord": {
      "accounts": {
        "client-id-123": {
          "token": "bot-token",
          "enabled": true
        }
      }
    }
  }
}
```

### 4.2 账户绑定

```javascript
function buildChannelAccountBindings(cfg, channelId, accountId) {
  const channelCfg = cfg.channels?.[channelId];
  if (!channelCfg) return [];
  
  const accountCfg = channelCfg.accounts?.[accountId];
  if (!accountCfg?.enabled) return [];
  
  const bindings = [];
  
  // 解析 allowFrom 绑定
  for (const allowFrom of channelCfg.allowFrom ?? []) {
    bindings.push({
      channelId,
      accountId,
      agentId: allowFrom.agentId,
      channelPattern: allowFrom.channel,
      peer: allowFrom.peer
    });
  }
  
  return bindings;
}
```

---

## 5. 通道选择

### 5.1 选择策略

```javascript
async function resolveMessageChannelSelection(params) {
  const { cfg } = params;
  
  // 1. 检查显式配置
  const explicitChannel = cfg.defaults?.channel;
  if (explicitChannel) {
    const plugin = getChannelPlugin(explicitChannel);
    if (plugin && hasEnabledAccount(cfg, explicitChannel)) {
      return { channel: explicitChannel };
    }
  }
  
  // 2. 检查最近使用的通道
  const recentChannel = getRecentChannelFromHistory(cfg);
  if (recentChannel && hasEnabledAccount(cfg, recentChannel)) {
    return { channel: recentChannel };
  }
  
  // 3. 默认通道
  const defaultChannel = getDefaultChannel(cfg);
  return { channel: defaultChannel };
}
```

### 5.2 通道优先级

```
1. 显式配置 (defaults.channel)
2. 最近使用 (session history)
3. 第一个可用通道
4. webchat (兜底)
```

---

## 6. 通道活动监控

### 6.1 活动跟踪

```javascript
function getChannelActivity(params) {
  const { cfg, channelId } = params;
  
  const channelCfg = cfg.channels?.[channelId];
  if (!channelCfg) return { ok: false, reason: "unknown" };
  
  const accounts = Object.keys(channelCfg.accounts ?? {});
  const enabledAccounts = accounts.filter(
    id => channelCfg.accounts[id].enabled
  );
  
  return {
    ok: true,
    channelId,
    totalAccounts: accounts.length,
    enabledAccounts: enabledAccounts.length,
    bindings: channelCfg.allowFrom?.length ?? 0
  };
}
```

### 6.2 健康检查

```javascript
async function checkChannelHealth(params) {
  const { cfg, channelId, accountId } = params;
  const plugin = getChannelPlugin(channelId);
  
  if (!plugin) return { ok: false, reason: "unknown" };
  
  try {
    const runtime = plugin.createRuntime(cfg, accountId, logger);
    const status = await runtime.healthCheck();
    
    return {
      ok: status.ok,
      channelId,
      accountId,
      latency: status.latency,
      details: status.details
    };
  } catch (error) {
    return {
      ok: false,
      channelId,
      accountId,
      error: String(error)
    };
  }
}
```

---

## 7. 通道状态问题

### 7.1 问题收集

```javascript
async function collectChannelStatusIssues(params) {
  const { cfg } = params;
  const issues = [];
  
  for (const plugin of listChannelPlugins()) {
    const channelCfg = cfg.channels?.[plugin.id];
    if (!channelCfg) continue;
    
    for (const [accountId, accountCfg] of Object.entries(channelCfg.accounts ?? {})) {
      if (!accountCfg.enabled) continue;
      
      // 检查 Token 配置
      if (!accountCfg.token) {
        issues.push({
          channel: plugin.id,
          accountId,
          severity: "error",
          message: "Missing token"
        });
      }
      
      // 检查健康状态
      const health = await checkChannelHealth({ cfg, channelId: plugin.id, accountId });
      if (!health.ok) {
        issues.push({
          channel: plugin.id,
          accountId,
          severity: "warning",
          message: health.error
        });
      }
    }
  }
  
  return issues;
}
```

### 7.2 问题修复

```javascript
async function fixChannelStatusIssues(params) {
  const { cfg, issues } = params;
  const fixed = [];
  
  for (const issue of issues) {
    if (issue.message === "Missing token") {
      // 提示用户配置 Token
      fixed.push({
        ...issue,
        fixed: false,
        instruction: `Run: openclaw channels config ${issue.channel} --account ${issue.accountId}`
      });
    }
  }
  
  return fixed;
}
```

---

## 8. 通道命令

### 8.1 CLI 命令

```bash
# 列出通道
openclaw channels list

# 通道状态
openclaw channels status

# 配置通道
openclaw channels config <channel> --account <id>

# 启用/禁用
openclaw channels enable <channel> --account <id>
openclaw channels disable <channel> --account <id>

# 健康检查
openclaw channels health <channel>
```

### 8.2 Gateway 方法

```javascript
// Gateway RPC 方法
{
  "channels.list": async () => { ... },
  "channels.status": async () => { ... },
  "channels.config.get": async (params) => { ... },
  "channels.config.set": async (params) => { ... },
  "channels.health": async (params) => { ... }
}
```

---

## 9. 支持通道

| 通道 | 功能 | 状态 |
|------|------|------|
| Slack | 消息/线程/反应/文件 | ✅ |
| Discord | 频道/服务器/反应/线程 | ✅ |
| Telegram | 私聊/群组/频道 | ✅ |
| WhatsApp | 私聊/群组 | ✅ |
| iMessage | 短信 (macOS/iOS) | ✅ |
| Webchat | Web 界面 | ✅ |
| Email | 邮件收发 | 🚧 |
| SMS | 短信 (Twilio) | 🚧 |

---

## 10. 关键设计模式

### 10.1 插件化架构

- 每个通道独立插件
- 统一接口 (createRuntime, start, stop, send)
- 热插拔 (启用/禁用不影响其他通道)

### 10.2 账户隔离

- 每通道多账户支持
- 账户级配置与状态
- 独立运行时实例

### 10.3 绑定系统

- allowFrom: 通道 → Agent 路由
- dmPolicy: DM 访问控制
- groupPolicy: 群组访问控制

---

## 11. 故障诊断

### 11.1 通道未响应

```bash
# 检查通道状态
openclaw channels status

# 健康检查
openclaw channels health slack

# 查看日志
openclaw logs tail --channel slack
```

### 11.2 账户认证失败

```bash
# 重新认证
openclaw channels config slack --account T123 --reauth

# 检查 Token
openclaw secrets list | grep SLACK
```

### 11.3 消息未路由

```bash
# 检查绑定配置
openclaw config get channels.slack.allowFrom

# 查看会话路由
openclaw sessions list --channel slack
```

---

## 12. 相关文档

- [通道配置](https://docs.openclaw.ai/channels/config)
- [Slack 集成](https://docs.openclaw.ai/channels/slack)
- [Discord 集成](https://docs.openclaw.ai/channels/discord)
- [Telegram 集成](https://docs.openclaw.ai/channels/telegram)

---

**阶段 3 - 通道系统** 完成 ✅

**下一步:** 其他核心模块或总结

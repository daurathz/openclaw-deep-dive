# 工具执行系统 — Exec 安全模型与审批流程

**阅读时间:** 2026-03-10  
**源码文件:** 
- `dist/gateway-cli-C2ZZYgwu.js` (ExecApprovalManager, createExecApprovalHandlers)
- `dist/plugin-sdk/dispatch-CJdFmoH9.js` (runExecProcess, resolveExecApprovals)

---

## 1. Exec 执行流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│  Agent 调用 exec 工具                                             │
│  - 命令文本: "ls -la /home"                                     │
│  - 超时：30 秒                                                    │
│  - 工作目录：/workspace                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  resolveExecHostApprovalContext()                               │
│  - 加载 exec approvals 配置                                      │
│  - 解析安全级别 (security): deny | allowlist | full             │
│  - 解析询问策略 (ask): off | on-miss | always                   │
│  - 检查 allowlist 匹配                                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  requiresExecApproval()                                         │
│  - ask=always → 需要审批                                        │
│  - ask=on-miss + security=allowlist + 未匹配 → 需要审批          │
│  - 否则 → 直接执行                                              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  需要审批？                                                      │
│  ├─ 是 → registerExecApprovalRequest()                          │
│  │      - 创建审批记录 (ExecApprovalManager)                    │
│  │      - 广播 exec.approval.requested 事件                     │
│  │      - 等待用户决策 (allow-once/allow-always/deny)           │
│  │      - 超时 → askFallback (deny/full)                        │
│  │                                                              │
│  └─ 否 → runExecProcess()                                       │
│         - 派生子进程/PTY                                        │
│         - 捕获 stdout/stderr                                    │
│         - 返回执行结果                                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  记录 allowlist 使用 (如果适用)                                   │
│  - 更新 lastUsedAt, lastUsedCommand, lastResolvedPath           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心配置结构

### 2.1 Exec Approvals 配置文件

**路径:** `~/.openclaw/exec-approvals.json`

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "<generated-token>"
  },
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": true
  },
  "agents": {
    "peter": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [
        {
          "id": "uuid-1234",
          "pattern": "ls *",
          "lastUsedAt": 1710067200000,
          "lastUsedCommand": "ls -la /home",
          "lastResolvedPath": "/bin/ls"
        },
        {
          "id": "uuid-5678",
          "pattern": "cat *.md",
          "lastUsedAt": 1710067100000
        }
      ]
    },
    "*": {
      "security": "full",
      "ask": "off"
    }
  }
}
```

### 2.2 安全级别 (security)

| 级别 | 说明 | 行为 |
|------|------|------|
| `deny` | 拒绝所有 | 所有 exec 调用被拒绝 |
| `allowlist` | 白名单模式 | 仅允许白名单命令，其他需要审批 |
| `full` | 完全访问 | 所有命令直接执行 (无审批) |

### 2.3 询问策略 (ask)

| 策略 | 说明 | 行为 |
|------|------|------|
| `off` | 不询问 | 直接执行 (如果 security 允许) |
| `on-miss` | 未匹配时询问 | 白名单未匹配时请求审批 |
| `always` | 总是询问 | 所有命令都需要审批 |

### 2.4 回退策略 (askFallback)

当审批超时时:

| 回退 | 说明 |
|------|------|
| `deny` | 超时 → 拒绝执行 |
| `full` | 超时 → 允许执行 |

---

## 3. 核心函数详解

### 3.1 `resolveExecApprovals()` — 解析审批配置

```javascript
function resolveExecApprovals(agentId, overrides) {
  const file = ensureExecApprovals();  // 确保配置文件存在
  
  return resolveExecApprovalsFromFile({
    file,
    agentId,
    overrides,
    path: resolveExecApprovalsPath(),
    socketPath: expandHomePrefix(file.socket?.path),
    token: file.socket?.token ?? ""
  });
}

function resolveExecApprovalsFromFile(params) {
  const file = normalizeExecApprovals(params.file);
  const defaults = file.defaults ?? {};
  const agentKey = params.agentId ?? "main";
  const agent = file.agents?.[agentKey] ?? {};
  const wildcard = file.agents?.["*"] ?? {};
  
  // 解析默认值
  const resolvedDefaults = {
    security: normalizeSecurity(defaults.security, fallbackSecurity),
    ask: normalizeAsk(defaults.ask, fallbackAsk),
    askFallback: normalizeSecurity(defaults.askFallback, fallbackAskFallback),
    autoAllowSkills: Boolean(defaults.autoAllowSkills)
  };
  
  // 解析 Agent 配置 (优先级：agent > wildcard > defaults)
  const resolvedAgent = {
    security: normalizeSecurity(
      agent.security ?? wildcard.security ?? resolvedDefaults.security,
      resolvedDefaults.security
    ),
    ask: normalizeAsk(
      agent.ask ?? wildcard.ask ?? resolvedDefaults.ask,
      resolvedDefaults.ask
    ),
    askFallback: normalizeSecurity(
      agent.askFallback ?? wildcard.askFallback ?? resolvedDefaults.askFallback,
      resolvedDefaults.askFallback
    ),
    autoAllowSkills: Boolean(
      agent.autoAllowSkills ?? wildcard.autoAllowSkills ?? resolvedDefaults.autoAllowSkills
    )
  };
  
  // 合并 allowlist (wildcard + agent)
  const allowlist = [
    ...Array.isArray(wildcard.allowlist) ? wildcard.allowlist : [],
    ...Array.isArray(agent.allowlist) ? agent.allowlist : []
  ];
  
  return {
    path: params.path,
    socketPath: params.socketPath,
    token: params.token,
    defaults: resolvedDefaults,
    agent: resolvedAgent,
    allowlist,
    file
  };
}
```

**优先级链:**
```
agent.<agentId> → agent["*"] → defaults → hardcoded fallback
```

---

### 3.2 `requiresExecApproval()` — 判断是否需要审批

```javascript
function requiresExecApproval(params) {
  return params.ask === "always" || 
         (params.ask === "on-miss" && 
          params.security === "allowlist" && 
          (!params.analysisOk || !params.allowlistSatisfied));
}
```

**决策逻辑:**
```
ask=always → 总是需要审批
ask=on-miss + security=allowlist + 未匹配白名单 → 需要审批
其他情况 → 不需要审批
```

---

### 3.3 `ExecApprovalManager` — 审批管理器

**类结构:**

```javascript
class ExecApprovalManager {
  constructor() {
    this.pending = new Map();  // 待审批记录
  }
  
  // 创建审批记录
  create(request, timeoutMs, id) {
    const now = Date.now();
    return {
      id: id || randomUUID(),
      request,
      createdAtMs: now,
      expiresAtMs: now + timeoutMs
    };
  }
  
  // 注册审批并返回决策 Promise
  register(record, timeoutMs) {
    const promise = new Promise((resolve, reject) => {
      // 保存 resolve/reject 回调
    });
    
    // 设置超时定时器
    const timer = setTimeout(() => {
      this.expire(record.id);
    }, timeoutMs);
    
    this.pending.set(record.id, { record, resolve, reject, timer, promise });
    return promise;
  }
  
  // 解决审批 (用户决策)
  resolve(recordId, decision, resolvedBy) {
    const pending = this.pending.get(recordId);
    if (!pending || pending.record.resolvedAtMs !== undefined) return false;
    
    clearTimeout(pending.timer);
    pending.record.resolvedAtMs = Date.now();
    pending.record.decision = decision;  // "allow-once" | "allow-always" | "deny"
    pending.record.resolvedBy = resolvedBy;
    pending.resolve(decision);
    
    // 宽限期后删除
    setTimeout(() => {
      if (this.pending.get(recordId) === pending) this.pending.delete(recordId);
    }, RESOLVED_ENTRY_GRACE_MS);
    
    return true;
  }
  
  // 超时失效
  expire(recordId, resolvedBy) {
    const pending = this.pending.get(recordId);
    if (!pending || pending.record.resolvedAtMs !== undefined) return false;
    
    clearTimeout(pending.timer);
    pending.record.resolvedAtMs = Date.now();
    pending.record.decision = undefined;  // null 表示超时
    pending.record.resolvedBy = resolvedBy;
    pending.resolve(null);
    
    setTimeout(() => {
      if (this.pending.get(recordId) === pending) this.pending.delete(recordId);
    }, RESOLVED_ENTRY_GRACE_MS);
    
    return true;
  }
  
  // 获取审批快照
  getSnapshot(recordId) {
    return this.pending.get(recordId)?.record ?? null;
  }
  
  // 消费 allow-once 决策 (一次性允许)
  consumeAllowOnce(recordId) {
    const entry = this.pending.get(recordId);
    if (!entry || entry.record.decision !== "allow-once") return false;
    entry.record.decision = undefined;
    return true;
  }
  
  // 等待已注册的审批决策
  awaitDecision(recordId) {
    return this.pending.get(recordId)?.promise ?? null;
  }
}
```

**审批状态流转:**
```
created → pending → resolved (allow-once/allow-always/deny)
              ↓
            expired (超时)
```

---

### 3.4 `createExecApprovalHandlers()` — Gateway 处理器

**注册的 Gateway 方法:**

```javascript
function createExecApprovalHandlers(manager, opts) {
  return {
    // 1. 请求审批
    "exec.approval.request": async ({ params, respond, context, client }) => {
      // 验证参数
      if (!validateExecApprovalRequestParams(params)) {
        respond(false, void 0, errorShape(INVALID_REQUEST, "invalid params"));
        return;
      }
      
      const p = params;
      const timeoutMs = p.timeoutMs ?? DEFAULT_EXEC_APPROVAL_TIMEOUT_MS;
      
      // 构建审批请求
      const request = {
        command: p.command,
        commandArgv: p.commandArgv,
        cwd: p.cwd,
        host: p.host,  // "host" | "node"
        nodeId: p.nodeId,
        security: p.security,
        ask: p.ask,
        agentId: p.agentId,
        sessionKey: p.sessionKey,
        turnSourceChannel: p.turnSourceChannel,
        // ...
      };
      
      // 创建审批记录
      const record = manager.create(request, timeoutMs, p.id);
      record.requestedByConnId = client?.connId;
      record.requestedByDeviceId = client?.connect?.device?.id;
      
      // 注册审批
      const decisionPromise = manager.register(record, timeoutMs);
      
      // 广播事件 (通知所有连接的客户端)
      context.broadcast("exec.approval.requested", {
        id: record.id,
        request: record.request,
        createdAtMs: record.createdAtMs,
        expiresAtMs: record.expiresAtMs
      }, { dropIfSlow: true });
      
      // 转发到外部目标 (如 Slack 通知)
      let forwardedToTargets = false;
      if (opts?.forwarder) {
        forwardedToTargets = await opts.forwarder.handleRequested({...});
      }
      
      // 如果没有审批客户端，自动超时
      if (!hasApprovalClients(context) && !forwardedToTargets) {
        manager.expire(record.id, "auto-expire:no-approver-clients");
      }
      
      // 两阶段响应 (先确认注册，再返回决策)
      if (p.twoPhase) {
        respond(true, { status: "accepted", id: record.id, ... });
      }
      
      // 等待决策
      const decision = await decisionPromise;
      respond(true, { id: record.id, decision, ... });
    },
    
    // 2. 等待决策
    "exec.approval.waitDecision": async ({ params, respond }) => {
      const decisionPromise = manager.awaitDecision(params.id);
      if (!decisionPromise) {
        respond(false, void 0, errorShape(INVALID_REQUEST, "approval expired or not found"));
        return;
      }
      
      const decision = await decisionPromise;
      respond(true, { id: params.id, decision, ... });
    },
    
    // 3. 解决审批 (用户决策)
    "exec.approval.resolve": async ({ params, respond, client, context }) => {
      const decision = params.decision;  // "allow-once" | "allow-always" | "deny"
      
      if (!manager.resolve(params.id, decision, resolvedBy)) {
        respond(false, void 0, errorShape(INVALID_REQUEST, "unknown approval id"));
        return;
      }
      
      // 广播决策事件
      context.broadcast("exec.approval.resolved", {
        id: params.id,
        decision,
        resolvedBy,
        ts: Date.now(),
        request: snapshot?.request
      });
      
      respond(true, { ok: true });
    }
  };
}
```

---

### 3.5 `runExecProcess()` — 执行进程

**职责:** 派生子进程执行命令

```javascript
async function runExecProcess(opts) {
  const sessionId = createSessionSlug();
  const execCommand = opts.execCommand ?? opts.command;
  
  // 会话状态
  const session = {
    id: sessionId,
    command: opts.command,
    sessionKey: opts.sessionKey,
    notifyOnExit: opts.notifyOnExit,
    child: void 0,
    pid: void 0,
    startedAt: Date.now(),
    cwd: opts.workdir,
    maxOutputChars: opts.maxOutput,
    totalOutputChars: 0,
    pendingStdout: [],
    pendingStderr: [],
    aggregated: "",
    tail: "",
    exited: false,
    exitCode: void 0,
    exitSignal: void 0,
    truncated: false
  };
  
  // 派生配置
  const spawnSpec = (() => {
    if (opts.sandbox) {
      // Docker 沙箱模式
      return {
        mode: "child",
        argv: ["docker", ...buildDockerExecArgs({...})],
        env: { ...opts.env, OPENCLAW_SHELL: "exec" },
        stdinMode: opts.usePty ? "pipe-open" : "pipe-closed"
      };
    }
    
    const { shell, args: shellArgs } = getShellConfig();
    const childArgv = [shell, ...shellArgs, execCommand];
    
    if (opts.usePty) {
      // PTY 模式 (交互式 CLI)
      return {
        mode: "pty",
        ptyCommand: execCommand,
        childFallbackArgv: childArgv,
        env: { ...opts.env, OPENCLAW_SHELL: "exec" },
        stdinMode: "pipe-open"
      };
    }
    
    // 普通子进程模式
    return {
      mode: "child",
      argv: childArgv,
      env: { ...opts.env, OPENCLAW_SHELL: "exec" },
      stdinMode: "pipe-closed"
    };
  })();
  
  // 派生进程
  let managedRun = await supervisor.spawn({
    runId: sessionId,
    sessionId: opts.sessionKey || sessionId,
    backendId: opts.sandbox ? "exec-sandbox" : "exec-host",
    scopeKey: opts.scopeKey,
    cwd: opts.workdir,
    env: spawnSpec.env,
    timeoutMs: opts.timeoutSec * 1000,
    captureOutput: false,
    onStdout: handleStdout,
    onStderr: handleStderr
  });
  
  // 等待完成
  const result = await managedRun.wait().then((exit) => {
    const isNormalExit = exit.reason === "exit";
    const exitCode = exit.exitCode ?? 0;
    const isShellFailure = exitCode === 126 || exitCode === 127;
    const status = isNormalExit && !isShellFailure ? "completed" : "failed";
    
    if (status === "completed") {
      return {
        status: "completed",
        exitCode,
        durationMs: Date.now() - startedAt,
        aggregated: session.aggregated.trim()
      };
    }
    
    // 失败原因
    const reason = isShellFailure 
      ? exitCode === 127 ? "Command not found" : "Command not executable"
      : exit.reason === "overall-timeout" ? "Command timed out"
      : exit.reason === "no-output-timeout" ? "Command timed out waiting for output"
      : exit.exitSignal != null ? `Command aborted by signal ${exit.exitSignal}`
      : "Command aborted";
    
    return {
      status: "failed",
      exitCode,
      exitSignal: exit.exitSignal,
      durationMs: Date.now() - startedAt,
      aggregated: session.aggregated.trim(),
      timedOut: exit.timedOut,
      reason
    };
  });
  
  return { session, pid: session.pid, promise: result, kill: () => managedRun?.cancel() };
}
```

**执行模式:**
- **child** — 普通子进程 (非交互式)
- **pty** — 伪终端 (交互式 CLI，如 vim/nano)
- **sandbox** — Docker 沙箱 (隔离执行)

---

## 4. Allowlist 匹配

### 4.1 匹配逻辑

```javascript
function matchesAllowlistPattern(command, pattern) {
  // 模式转换为正则
  const regex = patternToRegex(pattern);
  return regex.test(command);
}

function patternToRegex(pattern) {
  // 支持通配符: * (任意字符), ** (跨目录)
  // "ls *" → /^ls\s+.+$/
  // "cat **/*.md" → /^cat\s+.+\/.+\.md$/
  // "npm install" → /^npm\s+install$/
  
  const escaped = pattern
    .replace(/[.+?^${}()|[\]\\]/g, '\\$&')  // 转义正则特殊字符
    .replace(/\*\*/g, '.*')                 // ** → .*
    .replace(/\*/g, '[^\\s]+');             // * → [^\s]+
  
  return new RegExp(`^${escaped}$`);
}
```

### 4.2 匹配示例

| 模式 | 匹配命令 | 不匹配 |
|------|---------|--------|
| `ls *` | `ls -la`, `ls /home` | `ls`, `cd /home` |
| `cat **/*.md` | `cat README.md`, `cat docs/guide.md` | `cat file.txt` |
| `npm install` | `npm install` | `npm install -g`, `npm uninstall` |
| `git *` | `git status`, `git commit -m "..."` | `git` |

---

## 5. 审批决策类型

| 决策 | 说明 | 后续行为 |
|------|------|---------|
| `allow-once` | 允许一次 | 本次执行允许，下次相同命令仍需审批 |
| `allow-always` | 永久允许 | 自动添加到 allowlist，后续相同命令直接执行 |
| `deny` | 拒绝 | 命令被拒绝，不添加到 allowlist |

### 决策处理

```javascript
function handleExecApprovalDecision(decision, approvalId, command) {
  if (decision === "allow-always") {
    // 添加到 allowlist
    addAllowlistEntry(approvals, agentId, command);
  }
  
  if (decision === "allow-once") {
    // 消费一次性允许
    manager.consumeAllowOnce(approvalId);
  }
  
  // 记录使用
  if (decision.startsWith("allow")) {
    recordAllowlistUse(approvals, agentId, entry, command, resolvedPath);
  }
}
```

---

## 6. 安全模型

### 6.1 多层安全策略

```
配置层 (exec-approvals.json)
   ↓
Agent 层 (agent-specific settings)
   ↓
通配层层 (agent["*"])
   ↓
默认层 (defaults)
   ↓
硬编码回退 (hardcoded fallback)
```

### 6.2 安全级别合并

```javascript
function minSecurity(a, b) {
  const order = { deny: 0, allowlist: 1, full: 2 };
  return order[a] <= order[b] ? a : b;  // 取更严格的
}

function maxAsk(a, b) {
  const order = { off: 0, "on-miss": 1, always: 2 };
  return order[a] >= order[b] ? a : b;  // 取更频繁询问的
}

// 主机安全级别 = min(请求级别，配置级别)
const hostSecurity = minSecurity(params.security, approvals.agent.security);

// 主机询问策略 = max(请求策略，配置策略)
const hostAsk = maxAsk(params.ask, approvals.agent.ask);
```

---

## 7. 超时处理

### 7.1 审批超时

```javascript
const DEFAULT_EXEC_APPROVAL_TIMEOUT_MS = 120000;  // 2 分钟

// 超时定时器
const timer = setTimeout(() => {
  manager.expire(record.id, "timeout");
}, timeoutMs);

// 超时决策
if (!decision) {
  if (askFallback === "full") {
    // 超时允许执行
    return { approvedByAsk: true, deniedReason: null, timedOut: true };
  } else {
    // 超时拒绝执行
    return { approvedByAsk: false, deniedReason: "approval-timeout", timedOut: true };
  }
}
```

### 7.2 执行超时

```javascript
const timeoutMs = typeof opts.timeoutSec === "number" ? opts.timeoutSec * 1000 : void 0;

// 整体超时
if (exit.reason === "overall-timeout") {
  return {
    status: "failed",
    reason: `Command timed out after ${opts.timeoutSec} seconds`
  };
}

// 无输出超时
if (exit.reason === "no-output-timeout") {
  return {
    status: "failed",
    reason: "Command timed out waiting for output"
  };
}
```

---

## 8. 审计与日志

### 8.1 Allowlist 使用记录

```javascript
function recordAllowlistUse(approvals, agentId, entry, command, resolvedPath) {
  const target = agentId ?? "main";
  const agents = approvals.agents ?? {};
  const existing = agents[target] ?? {};
  
  const nextAllowlist = (existing.allowlist ?? []).map((item) => 
    item.pattern === entry.pattern ? {
      ...item,
      lastUsedAt: Date.now(),
      lastUsedCommand: command,
      lastResolvedPath: resolvedPath
    } : item
  );
  
  agents[target] = { ...existing, allowlist: nextAllowlist };
  approvals.agents = agents;
  saveExecApprovals(approvals);
}
```

### 8.2 审批事件广播

```javascript
// 请求审批
context.broadcast("exec.approval.requested", {
  id: record.id,
  request: record.request,
  createdAtMs: record.createdAtMs,
  expiresAtMs: record.expiresAtMs
});

// 解决审批
context.broadcast("exec.approval.resolved", {
  id: p.id,
  decision,
  resolvedBy,
  ts: Date.now(),
  request: snapshot?.request
});
```

---

## 9. 关键设计模式

### 9.1 两阶段提交

```
阶段 1: 注册审批 → 立即返回 { status: "accepted", id: "..." }
阶段 2: 等待决策 → 返回 { id: "...", decision: "allow-once" }
```

**优势:**
- 调用方确认注册成功后再等待
- 避免网络抖动导致重复注册

### 9.2 宽限期清理

```javascript
// 解决后不立即删除，保留宽限期
setTimeout(() => {
  if (this.pending.get(recordId) === pending) this.pending.delete(recordId);
}, RESOLVED_ENTRY_GRACE_MS);  // 通常 5-10 秒
```

**优势:**
- 允许调用方稍后查询已解决的审批
- 避免内存泄漏 (最终会清理)

### 9.3 自动超时

```javascript
// 没有审批客户端时自动超时
if (!hasApprovalClients(context) && !forwardedToTargets) {
  manager.expire(record.id, "auto-expire:no-approver-clients");
}
```

**优势:**
- 避免审批请求永久挂起
- 根据 askFallback 决策 (deny/full)

---

## 10. 故障诊断

### 10.1 查看审批配置

```bash
cat ~/.openclaw/exec-approvals.json
```

### 10.2 测试命令审批

```bash
# 执行命令 (可能触发审批)
openclaw exec "ls -la"

# 查看待审批请求
openclaw exec approvals list
```

### 10.3 修改安全级别

```bash
# 设置 Agent 安全级别
openclaw config set agents.peter.exec.security allowlist

# 设置询问策略
openclaw config set agents.peter.exec.ask on-miss
```

### 10.4 添加 Allowlist 条目

```bash
# 通过审批 UI 选择 "Allow Always"
# 或手动编辑 ~/.openclaw/exec-approvals.json
```

---

## 11. 相关文档

- [Exec 工具](https://docs.openclaw.ai/tools/exec)
- [安全模型](https://docs.openclaw.ai/security/exec-approvals)
- [审批流程](https://docs.openclaw.ai/gateway/exec-approvals)

---

**下一步:** 技能系统 — 技能的加载、注册、调用机制

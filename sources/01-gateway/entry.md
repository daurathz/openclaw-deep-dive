# Gateway 入口逻辑

**阅读时间:** 2026-03-10  
**源码文件:** 
- `openclaw.mjs` (入口包装器)
- `dist/entry.js` (CLI 入口)
- `dist/run-main-BcgwOk2p.js` (CLI 路由)
- `dist/gateway-cli-C2ZZYgwu.js` (Gateway 命令)

---

## 1. 启动流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│  openclaw.mjs                                                   │
│  - Node.js 版本检查 (≥22.12)                                    │
│  - 启用编译缓存                                                  │
│  - 加载警告过滤器                                                │
│  - 导入 dist/entry.js                                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  dist/entry.js                                                  │
│  - 检查是否主模块入口                                            │
│  - 设置进程标题 "openclaw"                                       │
│  - 实验性警告抑制 (可能重新生成子进程)                            │
│  - 处理 --version / --help 快速路径                              │
│  - 解析 CLI profile 参数                                         │
│  - 调用 runCli()                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  dist/run-main-BcgwOk2p.js                                      │
│  - 加载 .env 环境变量                                            │
│  - 确保 CLI 在 PATH 上                                            │
│  - 尝试路由 CLI 命令 (tryRouteCli)                               │
│  - 构建 Commander.js program                                     │
│  - 注册核心命令和插件命令                                         │
│  - program.parseAsync() 执行                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  dist/gateway-cli-C2ZZYgwu.js                                   │
│  - registerGatewayCli() 注册 gateway 命令                         │
│  - gateway run 命令 → runGatewayCommand()                        │
│  - startGatewayServer() 启动服务器                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 关键函数详解

### 2.1 `openclaw.mjs` — 入口包装器

```javascript
// 版本检查
const MIN_NODE_MAJOR = 22;
const MIN_NODE_MINOR = 12;

// 启用编译缓存 (Node.js 22.12+)
if (module.enableCompileCache && !process.env.NODE_DISABLE_COMPILE_CACHE) {
  module.enableCompileCache();
}

// 加载警告过滤器
await installProcessWarningFilter();

// 尝试导入入口
if (await tryImport("./dist/entry.js")) { /* OK */ }
```

**设计意图:** 
- 确保运行环境符合最低要求
- 编译缓存提升启动速度
- 过滤不必要的警告 (如 punycode 废弃警告)

---

### 2.2 `entry.js` — CLI 入口

**核心职责:**

1. **版本检查**
```javascript
ensureSupportedNodeVersion();  // ≥22.12
```

2. **实验性警告抑制** (可能重新生成子进程)
```javascript
function ensureExperimentalWarningSuppressed() {
  if (hasExperimentalWarningSuppressed()) return false;
  
  // 重新生成子进程，添加 --disable-warning=ExperimentalWarning
  const child = spawn(process.execPath, [
    "--disable-warning=ExperimentalWarning",
    ...process.execArgv,
    ...process.argv.slice(1)
  ], { stdio: "inherit", env: process.env });
  
  return true;  // 父进程退出，子进程继续
}
```

3. **Profile 解析**
```javascript
const parsed = parseCliProfileArgs(process.argv);
if (parsed.profile) {
  applyCliProfileEnv({ profile: parsed.profile });
  // 设置 OPENCLAW_PROFILE, OPENCLAW_STATE_DIR 等环境变量
}
```

4. **快速路径处理**
```javascript
if (!tryHandleRootVersionFastPath(argv) && 
    !tryHandleRootHelpFastPath(argv)) {
  import("./run-main-BcgwOk2p.js").then(({ runCli }) => runCli());
}
```

---

### 2.3 `run-main-BcgwOk2p.js` — CLI 路由系统

**路由机制:**

```javascript
const routes = [
  routeHealth,      // openclaw health
  routeStatus,      // openclaw status
  routeSessions,    // openclaw sessions
  routeAgentsList,  // openclaw agents list
  routeMemoryStatus // openclaw memory status
  // ... 更多路由
];

async function tryRouteCli(argv) {
  const path = getCommandPathWithRootOptions(argv, 2);
  const route = findRoutedCommand(path);
  if (!route) return false;
  
  await prepareRoutedCommand({ argv, commandPath: path });
  return route.run(argv);  // 执行路由命令
}

async function runCli(argv) {
  // ... 初始化 ...
  
  if (await tryRouteCli(normalizedArgv)) return;  // 快速路由
  
  // 未路由则使用 Commander.js
  const program = buildProgram();
  await program.parseAsync(parseArgv);
}
```

**设计意图:**
- 常用命令 (health/status) 直接路由，避免加载完整 CLI
- 提升响应速度，减少不必要的插件加载

---

### 2.4 `gateway-cli-C2ZZYgwu.js` — Gateway 命令

**命令结构:**

```javascript
registerGatewayCli(program)
  ↓
program.command("gateway")
  .command("run")     → addGatewayRunCommand()
  .command("status")  → addGatewayServiceCommands()
  .command("call")    → 调用 Gateway RPC 方法
  .command("health")  → 健康检查
  .command("probe")   → 探测 Gateway 可达性
  .command("discover")→ Bonjour 服务发现
```

**Gateway 启动核心:**

```javascript
async function runGatewayCommand$1(opts) {
  // 1. 解析端口、绑定模式、认证配置
  const port = parsePort(opts.port) ?? 18789;
  const bind = opts.bind ?? cfg.gateway?.bind ?? "loopback";
  
  // 2. 在锁保护下启动 Gateway
  await runGatewayLoop({
    runtime: defaultRuntime,
    lockPort: port,
    start: async () => await startGatewayServer(port, { bind, auth, tailscale })
  });
}
```

---

### 2.5 `startGatewayServer()` — 核心启动函数

**启动流程:**

```javascript
async function startGatewayServer(port = 18789, opts = {}) {
  // 1. 设置环境变量
  process.env.OPENCLAW_GATEWAY_PORT = String(port);
  
  // 2. 读取并验证配置
  let configSnapshot = await readConfigFileSnapshot();
  if (!configSnapshot.valid) {
    throw new Error(`Invalid config at ${configSnapshot.path}`);
  }
  
  // 3. 自动启用插件
  const autoEnable = applyPluginAutoEnable({ config, env });
  
  // 4. 激活 Secrets 运行时快照
  await activateRuntimeSecrets(config, { reason: "startup", activate: true });
  
  // 5. 确保 Gateway 启动认证配置
  const authBootstrap = await ensureGatewayStartupAuth({ cfg, authOverride, tailscaleOverride });
  
  // 6. 加载插件
  const { pluginRegistry, gatewayMethods } = loadGatewayPlugins({ cfg, workspaceDir });
  
  // 7. 解析运行时配置
  const runtimeConfig = await resolveGatewayRuntimeConfig({ cfg, port, bind, ... });
  
  // 8. 创建 Gateway 运行时状态
  const { httpServer, wss, clients, broadcast, ... } = 
    await createGatewayRuntimeState({ ... });
  
  // 9. 启动服务发现 (Bonjour)
  bonjourStop = await startGatewayDiscovery({ port, gatewayTls, ... });
  
  // 10. 启动维护定时器
  startGatewayMaintenanceTimers({ broadcast, ... });
  
  // 11. 启动 Cron 服务
  cron.start();
  
  // 12. 恢复待发送消息
  await recoverPendingDeliveries({ ... });
  
  // 13. 绑定 WebSocket 处理器
  attachGatewayWsHandlers({ wss, gatewayMethods, ... });
  
  // 14. 记录启动日志
  logGatewayStartup({ cfg, bindHost, port, tlsEnabled, ... });
}
```

---

## 3. 关键设计模式

### 3.1 渐进式启动

```
验证配置 → 激活密钥 → 加载插件 → 创建服务器 → 启动服务
   ↓          ↓          ↓          ↓          ↓
 失败立即   失败降级    失败继续    失败退出    失败记录
 退出       运行                    (端口占用)
```

**优势:**
- 配置错误早发现，避免启动后故障
- 密钥问题优雅降级，保持服务可用
- 插件可选，核心功能不依赖插件

### 3.2 Gateway 锁机制

```javascript
async function runGatewayLoop({ lockPort, start }) {
  const lockDir = resolveGatewayLockDir();
  const lockFile = path.join(lockDir, `${lockPort}.lock`);
  
  // 尝试获取锁
  try {
    fs.writeFileSync(lockFile, String(process.pid), { flag: "wx" });
  } catch (err) {
    if (err.code === "EEXIST") {
      throw new GatewayLockError("Gateway already running on this port");
    }
    throw err;
  }
  
  try {
    await start();  // 启动 Gateway
  } finally {
    fs.unlinkSync(lockFile);  // 释放锁
  }
}
```

**设计意图:**
- 防止同一端口重复启动
- PID 记录便于故障诊断
- 异常退出时锁文件可能残留 (需清理)

### 3.3 Secrets 运行时快照

```javascript
const activateRuntimeSecrets = async (config, params) => {
  try {
    const prepared = await prepareSecretsRuntimeSnapshot({ config });
    if (params.activate) {
      activateSecretsRuntimeSnapshot(prepared);  // 设为全局活跃快照
      logGatewayAuthSurfaceDiagnostics(prepared);
    }
    return prepared;
  } catch (err) {
    if (params.reason === "startup") {
      throw new Error(`Startup failed: required secrets unavailable`, { cause: err });
    }
    // 非启动时保持上次已知良好状态
    secretsDegraded = true;
    throw err;
  }
};
```

**设计意图:**
- 启动时密钥必须可用 (否则退出)
- 运行时密钥故障不中断服务 (降级运行)
- 支持配置热重载时密钥更新

---

## 4. 配置项关联

### 4.1 Gateway 端口

```json
{
  "gateway": {
    "port": 18789  // 默认端口
  }
}
```

**优先级:**
1. CLI `--port` 参数
2. `OPENCLAW_GATEWAY_PORT` 环境变量
3. 配置文件 `gateway.port`
4. 默认值 18789

### 4.2 绑定模式

```json
{
  "gateway": {
    "bind": "loopback"  // loopback | lan | tailnet | auto | custom
  }
}
```

**模式说明:**
- `loopback` — 仅本地访问 (127.0.0.1)
- `lan` — 局域网访问 (0.0.0.0 或具体 IP)
- `tailnet` — Tailscale 网络
- `auto` — 自动选择
- `custom` — 自定义主机

### 4.3 认证模式

```json
{
  "gateway": {
    "auth": {
      "mode": "token",     // token | password | shared-secret
      "token": "xxx"       // 自动生成或手动配置
    }
  }
}
```

---

## 5. 重要日志输出

### 5.1 启动成功

```
[gateway] Starting on ws://127.0.0.1:18789
[gateway] Auth mode: token
[gateway] Control UI: enabled at /
[gateway] Discovery: broadcasting via Bonjour
```

### 5.2 配置迁移

```
[gateway] migrated legacy config entries:
- gateway.port → moved to gateway.bind
- old.key → removed
```

### 5.3 密钥警告

```
[secrets] [SECRETS_RELOADER_DEGRADED] Failed to resolve API key
[secrets] [SECRETS_RELOADER_RECOVERED] Secret resolution recovered
```

---

## 6. 故障诊断

### 6.1 端口被占用

```bash
# 查看端口占用
openclaw gateway probe

# 强制释放端口
openclaw gateway run --force

# 或停止 Gateway 服务
openclaw gateway stop
```

### 6.2 配置验证失败

```bash
# 检查配置
openclaw doctor

# 查看配置问题
openclaw config get --json
```

### 6.3 密钥问题

```bash
# 查看密钥状态
openclaw secrets status

# 重新配置密钥
openclaw secrets set <target>
```

---

## 7. 相关文档

- [CLI Gateway](https://docs.openclaw.ai/cli/gateway)
- [配置参考](https://docs.openclaw.ai/config)
- [认证模式](https://docs.openclaw.ai/gateway/auth)
- [服务发现](https://docs.openclaw.ai/gateway/discovery)

---

**下一步:** 阅读 Agent 运行时 (`dist/agents-*.js`)

# 源码阅读进度

**阶段:** 1-4 完成  
**最后更新:** 2026-03-11 00:15 (Asia/Shanghai)  
**当前状态:** 待推送 GitHub

**Git:** 8 模块 · ~135KB 笔记

---

## 总结

**8 个核心模块已完成:**

| 模块 | 笔记 | 核心内容 |
|------|------|---------|
| Gateway 入口 | `gateway/entry.md` | CLI 路由/锁机制/Secrets 快照/渐进式启动 |
| Agent 运行时 | `agents/runtime.md` | 会话解析/技能快照/模型回退/ACP 集成 |
| 消息路由 | `routing/inbound.md` | 分层路由/回复调度/TTS/钩子系统 |
| 工具执行 | `tools/exec-security.md` | 三层安全/审批管理/Allowlist |
| 技能系统 | `skills/system.md` | 技能加载/快照/安装/调用 |
| 会话管理 | `sessions/management.md` | AcpSessionManager/Actor 队列/运行时缓存 |
| 通道系统 | `channels/system.md` | 插件架构/ChannelManager/账户绑定 |

**关键设计模式:**
- 渐进式启动/加载
- 优雅降级
- 快照隔离
- Actor 模型 (会话序列化)
- 插件化架构 (通道/技能)
- 两阶段提交 (审批)

---

**待决:** GitHub 推送 (需远程仓库地址)

---

## 已完成模块

### 阶段 1 - 核心架构 ✅

| 模块 | 文件 | 状态 | 笔记 |
|------|------|------|------|
| Gateway 入口 | `gateway/entry.md` | ✅ | CLI 启动流程、Gateway 服务器初始化、锁机制、Secrets 快照 |
| Agent 运行时 | `agents/runtime.md` | ✅ | 会话解析、技能快照、模型回退、ACP 集成、交付计划 |
| 消息路由 | `routing/inbound.md` | ✅ | 上下文标准化、Agent 路由决策、会话记录、回复调度、TTS 集成 |
| 工具执行 | `tools/exec-security.md` | ✅ | Exec 审批配置、ExecApprovalManager、安全模型、Allowlist 匹配 |
| 技能系统 | `skills/system.md` | ✅ | 技能加载、快照构建、安装机制、调用方式 |

### 阶段 2 - 会话管理 ✅

| 模块 | 文件 | 状态 | 笔记 |
|------|------|------|------|
| 会话管理 | `sessions/management.md` | ✅ | AcpSessionManager、Actor 队列、运行时缓存、会话存储 |

---

## 下一步

- [ ] Agent 运行时 (`agents/`)
- [ ] 会话管理 (`sessions/`)
- [ ] 工具系统 (`tools/`)
- [ ] 技能系统 (`skills/`)
- [ ] 通道系统 (`channels/`)

---

## 阅读顺序

1. ✅ **Gateway 入口逻辑** — `openclaw.mjs` → `entry.js` → `run-main.js` → `gateway-cli.js`
2. ⏳ **Agent 运行时** — Agent 如何启动、会话如何创建
3. ⏳ **消息路由** — 消息如何从通道到 Agent 再到模型
4. ⏳ **工具执行** — exec 工具的安全模型和审批流程
5. ⏳ **技能系统** — 技能的加载、注册、调用机制

---

## 关键发现

### Gateway 启动流程

```
openclaw.mjs (入口包装器)
  ↓
entry.js (版本检查、警告过滤、profile 解析)
  ↓
run-main-BcgwOk2p.js (CLI 路由系统)
  ↓
gateway-cli-C2ZZYgwu.js (Gateway 命令注册)
  ↓
startGatewayServer() (核心启动函数)
  ↓
createGatewayRuntimeState() (创建运行时状态)
  ↓
attachGatewayWsHandlers() (绑定 WebSocket 处理器)
```

### 核心组件

1. **CLI 路由系统** — 根据命令路径快速路由到专用处理函数
2. **Gateway 锁机制** — 防止同一端口重复启动
3. **配置热重载** — 监听 config 变化，必要时重启 Gateway
4. **Secrets 运行时快照** — 启动时激活密钥，支持热重载
5. **插件系统** — 插件在 Gateway 启动时加载并注册方法

### 设计亮点

- **渐进式启动** — 先验证配置和密钥，再启动服务器
- **优雅降级** — 密钥加载失败时保持上次已知良好状态
- **多绑定模式** — 支持 loopback/lan/tailnet/custom 多种绑定方式
- **健康检查** — 定期广播健康状态，支持服务发现

---

**备注:** 源码位于 `/home/openclaw/.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/`
- `dist/` — 编译后的 JavaScript (主要阅读目标)
- `docs/` — 文档
- `skills/` — 内置技能

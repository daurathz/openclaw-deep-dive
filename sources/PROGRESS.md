# OpenClaw 源码阅读进度

**最后更新:** 2026-03-11 16:10 (Asia/Shanghai)  
**当前状态:** 阶段 1-5 完成 ✅  
**Git:** 待提交 · 13 个模块目录 · 18 个笔记文件  
**仓库:** https://github.com/daurathz/openclaw-deep-dive

---

## 📊 总览

| 阶段 | 主题 | 模块 | 进度 |
|------|------|------|------|
| 阶段 1 | 核心架构 | Gateway, Agents, Sessions, Context | ✅ 4/4 |
| 阶段 2 | 消息流 | Routing, Channels | ✅ 2/2 |
| 阶段 3 | 工具与技能 | Tools, Skills | ✅ 2/2 |
| 阶段 4 | 持久化 | Compaction, Pruning | ✅ 2/2 |
| 阶段 5 | 自动化 | Cron, Hooks, Subagents | ✅ 3/3 |

**总计:** 13/13 模块完成 (100%) 🎉

**关键设计模式:** 渐进式启动 · 优雅降级 · 快照隔离 · Actor 模型 · 插件化架构 · 两阶段提交

---

## ✅ 已完成模块

### 阶段 1 - 核心架构

| 模块 | 目录 | 笔记文件 |
|------|------|----------|
| Gateway 入口 | `01-gateway/` | `entry.md`, `gateway-architecture.md` |
| Agent 运行时 | `01-agents/` | `runtime.md`, `agent-loop.md` |
| 会话管理 | `01-sessions/` | `management.md`, `session-management.md` |
| 上下文组装 | `01-context/` | `assembly.md`, `context-assembly.md` |

### 阶段 2 - 消息流

| 模块 | 目录 | 笔记文件 |
|------|------|----------|
| 消息路由 | `02-routing/` | `inbound.md` |
| 通道系统 | `02-channels/` | `system.md`, `messages-channels.md` |

### 阶段 3 - 工具与技能

| 模块 | 目录 | 笔记文件 |
|------|------|----------|
| 工具执行 | `03-tools/` | `exec-security.md` |
| 技能系统 | `03-skills/` | `system.md` |

### 阶段 4 - 持久化

| 模块 | 目录 | 笔记文件 |
|------|------|----------|
| 上下文压缩 | `04-compaction/` | `compaction-mechanism.md` |
| 会话修剪 | `04-pruning/` | `session-pruning.md` |

### 阶段 5 - 自动化

| 模块 | 目录 | 笔记文件 |
|------|------|----------|
| Cron 调度 | `05-cron/` | `cron-scheduler.md` |
| Workspace Hooks | `05-hooks/` | `workspace-hooks.md` |
| Subagents | `05-subagents/` | `subagent-registry.md` |

---

## 📚 阅读顺序

```
阶段 1: Gateway → Agents → Sessions → Context
阶段 2: Routing → Channels
阶段 3: Tools → Skills
阶段 4: Compaction → Pruning
阶段 5: Cron → Hooks → Subagents ✅ 完成
```

---

## 🔑 关键发现

### Gateway 启动流程
```
openclaw.mjs → entry.js → run-main.js → gateway-cli.js 
→ startGatewayServer() → createGatewayRuntimeState() 
→ attachGatewayWsHandlers()
```

### 核心设计亮点
- **渐进式启动** — 先验证配置和密钥，再启动服务器
- **优雅降级** — 密钥加载失败时保持上次已知良好状态
- **快照隔离** — 技能/会话使用独立快照，避免污染
- **Actor 模型** — 会话串行化处理，避免并发冲突
- **插件化架构** — 通道/技能作为插件动态加载
- **子 Agent 系统** — 任务分解、并行执行、结果聚合
- **Cron 调度** — 支持 at/every/cron 三种调度方式
- **Hooks 系统** — 生命周期事件拦截与扩展

---

## 📈 统计

- **总阶段数:** 5
- **已完成:** 5 阶段 (100%) ✅
- **总模块数:** 13
- **已完成:** 13 模块 (100%) ✅
- **总笔记文件:** 18
- **总代码行数:** ~15,000 行

---

**源码位置:** `/home/openclaw/.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/`
- `dist/` — 编译后的 JavaScript (主要阅读目标)
- `docs/` — 文档
- `skills/` — 内置技能

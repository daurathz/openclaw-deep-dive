# OpenClaw 源码学习笔记

**最后更新:** 2026-03-11 10:05  
**进度:** 阶段 1-4 完成 (10/14 模块)  
**GitHub:** https://github.com/daurathz/openclaw-deep-dive

---

## 📚 目录结构

```
sources/
├── README.md                 # 本文件
├── PROGRESS.md               # 详细进度追踪
│
├── 01-gateway/               # 阶段 1: Gateway 入口
│   ├── entry.md              # CLI 路由、启动流程
│   └── gateway-architecture.md
│
├── 01-agents/                # 阶段 1: Agent 运行时
│   ├── runtime.md            # Agent Loop、会话引导
│   └── agent-loop.md
│
├── 01-sessions/              # 阶段 1: 会话管理
│   ├── management.md         # AcpSessionManager、Actor 队列
│   └── session-management.md
│
├── 01-context/               # 阶段 1: 上下文组装
│   ├── assembly.md           # 上下文构建、记忆检索
│   └── context-assembly.md
│
├── 02-routing/               # 阶段 2: 消息路由
│   └── inbound.md            # 入站消息处理、路由决策
│
├── 02-channels/              # 阶段 2: 通道系统
│   ├── system.md             # 插件架构、ChannelManager
│   └── messages-channels.md
│
├── 03-tools/                 # 阶段 3: 工具执行
│   └── exec-security.md      # exec 安全模型、审批流程
│
├── 03-skills/                # 阶段 3: 技能系统
│   └── system.md             # 技能加载、快照、调用
│
├── 04-compaction/            # 阶段 4: 上下文压缩
│   └── compaction-mechanism.md
│
└── 04-pruning/               # 阶段 4: 会话修剪
    └── session-pruning.md
```

---

## 📊 进度概览

| 阶段 | 模块 | 目录 | 状态 | 笔记 |
|------|------|------|------|------|
| **阶段 1** | Gateway | `01-gateway/` | ✅ | 2 文件 |
| | Agent | `01-agents/` | ✅ | 2 文件 |
| | Sessions | `01-sessions/` | ✅ | 2 文件 |
| | Context | `01-context/` | ✅ | 2 文件 |
| **阶段 2** | Routing | `02-routing/` | ✅ | 1 文件 |
| | Channels | `02-channels/` | ✅ | 2 文件 |
| **阶段 3** | Tools | `03-tools/` | ✅ | 1 文件 |
| | Skills | `03-skills/` | ✅ | 1 文件 |
| **阶段 4** | Compaction | `04-compaction/` | ✅ | 1 文件 |
| | Pruning | `04-pruning/` | ✅ | 1 文件 |
| **阶段 5** | Cron | `05-cron/` | ⏳ | 待读 |
| | Hooks | `05-hooks/` | ⏳ | 待读 |
| | Subagents | `05-subagents/` | ⏳ | 待读 |
| | Heartbeat | `05-heartbeat/` | ⏳ | 待读 |

**总计:** 10/14 模块完成 (71%)

---

## 🎯 学习路径

### 阶段 1 - 核心架构 ✅
理解 Agent 如何启动和运行

1. Gateway 入口 → 2. Agent 运行时 → 3. 会话管理 → 4. 上下文组装

### 阶段 2 - 消息流 ✅
理解消息如何路由和交付

5. 消息路由 → 6. 通道系统

### 阶段 3 - 工具与技能 ✅
理解能力如何扩展

7. 工具执行 → 8. 技能系统

### 阶段 4 - 持久化 ✅
理解上下文如何管理

9. 上下文压缩 → 10. 会话修剪

### 阶段 5 - 自动化与扩展 ⏳
理解任务如何调度

11. Cron 调度 → 12. 钩子系统 → 13. 子 Agent → 14. 心跳机制

---

## 📝 笔记格式

每个模块的笔记包含：

```markdown
# <模块名>

## 核心概念
<模块的核心职责>

## 源码位置
- `dist/<路径>/<文件>.js`
- 关键函数：`<函数名>()`

## 关键流程
<流程图或步骤说明>

## 设计亮点
- 亮点 1
- 亮点 2

## 个人思考
<理解和疑问>
```

---

## 🔗 相关链接

- [详细进度](./PROGRESS.md)
- [项目 README](../README.md)
- [管理规范](../GOVERNANCE.md)

---

_最后更新：2026-03-11 (目录结构重组完成)_

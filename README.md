# OpenClaw 深度学习笔记 🦞

> _"EXFOLIATE! EXFOLIATE!"_ — A space lobster, probably

---

## 📖 这是什么？

这是 OpenClaw 创作者 Peter 的**系统性源码学习笔记**。

通过阅读 OpenClaw 源码（`dist/` 目录），记录：
- **架构设计哲学** - 为什么这样设计
- **底层工作机制** - 源码级别的实现细节
- **关键设计模式** - 可复用的架构模式

---

## 🗂️ 目录结构

```
openclaw-deep-dive/
├── README.md                 # 本文件 - 导航与说明
├── PROGRESS.md               # 学习进度追踪
├── HEARTBEAT.md              # 当前待执行任务
├── GOVERNANCE.md             # 项目管理规范 (待创建)
│
├── sources/                  # 📚 核心学习笔记 (按源码结构组织)
│   ├── README.md             # sources 目录导航
│   ├── PROGRESS.md           # 模块进度详情
│   │
│   ├── gateway/              # Gateway 系统
│   │   └── entry.md          # 入口逻辑、CLI 路由、启动流程
│   ├── agents/               # Agent 运行时
│   │   └── runtime.md        # Agent Loop、会话引导、技能快照
│   ├── sessions/             # 会话管理
│   │   └── management.md     # AcpSessionManager、Actor 队列
│   ├── skills/               # 技能系统
│   │   └── system.md         # 技能加载、快照、调用机制
│   ├── channels/             # 通道系统
│   │   └── system.md         # 插件架构、ChannelManager
│   ├── routing/              # 消息路由
│   │   └── inbound.md        # 入站消息处理、路由决策
│   ├── context/              # 上下文管理
│   │   └── assembly.md       # 上下文组装、记忆检索
│   └── tools/                # 工具系统
│       └── exec-security.md  # exec 安全模型、审批流程
│
├── chapters/                 # 📖 详细章节版 (历史版本，内容更详细)
│   ├── 00-getting-ready.md
│   ├── 01-gateway-architecture.md
│   ├── 02-agent-runtime.md
│   ├── 03-session-management.md
│   ├── 04-skill-system.md
│   ├── 05-security-model.md
│   ├── 06-multi-agent-routing.md
│   └── 07-building-skills.md
│
├── examples/                 # 💻 代码示例
│   ├── configs/              # 配置示例
│   └── running/              # 运行日志示例
├── diagrams/                 # 🎨 架构图
│   └── *.excalidraw
└── appendix/                 # 📎 补充材料
    └── A-data-flow.md
```

---

## 📚 如何使用

### 快速开始

1. **从 `sources/` 开始** - 按模块组织，对应 OpenClaw 源码结构
2. **查看 `sources/PROGRESS.md`** - 了解已完成模块和待读清单
3. **深入 `chapters/`** - 需要更详细解释时阅读章节版

### 学习路径推荐

```
阶段 1: 核心架构 (✅ 已完成)
  └─> gateway/entry.md → agents/runtime.md → sessions/management.md

阶段 2: 消息流 (✅ 已完成)
  └─> routing/inbound.md → channels/system.md → context/assembly.md

阶段 3: 工具与技能 (✅ 已完成)
  └─> tools/exec-security.md → skills/system.md

阶段 4: 深入扩展 (⏳ 待开始)
  └─> 记忆系统、上下文压缩、Cron 调度、钩子系统
```

---

## 📊 当前进度

| 阶段 | 模块 | 状态 | 笔记 |
|------|------|------|------|
| 阶段 1 | Gateway 入口 | ✅ | `gateway/entry.md` |
| 阶段 1 | Agent 运行时 | ✅ | `agents/runtime.md` |
| 阶段 1 | 会话管理 | ✅ | `sessions/management.md` |
| 阶段 2 | 消息路由 | ✅ | `routing/inbound.md` |
| 阶段 2 | 通道系统 | ✅ | `channels/system.md` |
| 阶段 2 | 上下文组装 | ✅ | `context/assembly.md` |
| 阶段 3 | 工具执行 | ✅ | `tools/exec-security.md` |
| 阶段 3 | 技能系统 | ✅ | `skills/system.md` |

**总计:** 8/8 核心模块完成 (100%)

详细进度见 [`sources/PROGRESS.md`](./sources/PROGRESS.md)

---

## 🔑 关键设计模式

通过源码阅读发现的核心设计模式：

| 模式 | 应用场景 | 说明 |
|------|----------|------|
| 渐进式启动 | Gateway | 先验证配置和密钥，再启动服务器 |
| 优雅降级 | Secrets 加载 | 失败时保持上次已知良好状态 |
| 快照隔离 | 技能系统 | 每次调用使用独立快照，避免污染 |
| Actor 模型 | 会话管理 | 会话串行化处理，避免并发冲突 |
| 插件化架构 | 通道系统 | 通道作为插件动态加载 |
| 两阶段提交 | 工具审批 | exec 审批需要用户确认 |
| CLI 路由 | 命令处理 | 根据命令路径快速路由到处理函数 |

---

## 🦞 关于作者

**Peter** - OpenClaw 的创作者人格

- 相信 AI 应该是实用的、可扩展的、透明的、有趣的
- 喜欢用 🦞 龙虾 emoji
- 设计哲学：Build things that matter

---

## 📞 联系

- Slack: @peter
- GitHub: https://github.com/daurathz/openclaw-deep-dive
- 社区: https://discord.com/invite/clawd

---

## 📜 许可证

MIT License

---

_最后更新：2026-03-11 (整合 main 和 master 分支)_

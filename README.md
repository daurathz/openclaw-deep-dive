# OpenClaw 深度学习指南 🦞

> _"EXFOLIATE! EXFOLIATE!"_ — A space lobster, probably

---

## 📖 这是什么？

这是 OpenClaw 创作者 Peter 为你准备的**系统性深度学习资料**。

不是简单的使用教程，而是带你深入理解：
- **为什么这样设计** - 设计哲学和权衡
- **底层如何工作** - 源码级别的机制解析
- **如何充分利用** - 最佳实践和实战案例

---

## 🎯 学习路径

```mermaid
flowchart LR
    Start[开始学习] --> Ch0[第 0 章<br/>准备工作]
    Ch0 --> Ch1[第 1 章<br/>Gateway 架构]
    Ch1 --> Ch2[第 2 章<br/>Agent 运行时]
    Ch2 --> Ch3[第 3 章<br/>会话管理]
    Ch3 --> Ch4[第 4 章<br/>技能系统]
    Ch4 --> Ch5[第 5 章<br/>安全模型]
    Ch5 --> Ch6[第 6 章<br/>多 Agent 路由]
    Ch6 --> Ch7[第 7 章<br/>实战：构建技能]
    Ch7 --> Done[🎓 毕业]
    
    style Start fill:#e1f5ff
    style Done fill:#d4edda
```

---

## 📚 章节列表

| 章节 | 主题 | 状态 | 核心内容 |
|------|------|------|----------|
| 第 0 章 | 准备工作 | 🚧 进行中 | 环境检查、概念速览、学习路径 |
| 第 1 章 | Gateway 架构详解 | ⏳ 待发布 | WebSocket 协议、连接生命周期、事件驱动 |
| 第 2 章 | Agent 运行时 | ⏳ 待发布 | 工作空间注入、会话引导、工具调用 |
| 第 3 章 | 会话管理 | ⏳ 待发布 | sessionKey 生成、DM/群聊隔离、存储结构 |
| 第 4 章 | 技能系统 | ⏳ 待发布 | 技能加载、SKILL.md 规范、工具注册 |
| 第 5 章 | 安全模型 | ⏳ 待发布 | 信任边界、配对/允许列表、沙箱隔离 |
| 第 6 章 | 多 Agent 路由 | ⏳ 待发布 | 路由决策、会话隔离、跨 Agent 通信 |
| 第 7 章 | 实战：构建技能 | ⏳ 待发布 | 完整技能开发流程 |

**图例：** 🚧 进行中 | ⏳ 待发布 | ✅ 已完成

---

## 🛠️ 如何使用

### 前置要求

- ✅ 已安装 OpenClaw (`npm install -g openclaw@latest`)
- ✅ Gateway 正常运行 (`openclaw gateway status`)
- ✅ 至少一个可用的 Agent
- ✅ 基本的 Node.js 和命令行知识

### 学习方式

1. **按顺序阅读** - 每章建立在前面章节的基础上
2. **动手实践** - 每章都有实战练习
3. **提问讨论** - Slack 随时问我 (@peter)

### 推荐节奏

```
每周 1-2 章：
  - 阅读章节内容 (1-2 小时)
  - 完成实战练习 (1-2 小时)
  - Slack 讨论问题 (灵活)
  
预计周期：6-8 周完成核心内容
```

---

## 📁 目录结构

```
openclaw-deep-dive/
├── README.md                 # 本文件 - 学习导航
├── chapters/                 # 章节内容
│   ├── 00-getting-ready.md   # 第 0 章：准备工作
│   ├── 01-gateway-architecture.md
│   ├── 02-agent-runtime.md
│   ├── 03-session-management.md
│   ├── 04-skills-system.md
│   ├── 05-security-model.md
│   ├── 06-multi-agent-routing.md
│   └── 07-build-a-skill.md
├── diagrams/                 # 独立图表文件
│   └── *.excalidraw          # Excalidraw 源文件
├── examples/                 # 代码示例
│   ├── configs/              # 配置示例
│   ├── skills/               # 技能示例
│   └── scripts/              # 脚本示例
└── appendix/                 # 附录
    ├── troubleshooting.md    # 故障诊断
    ├── cli-reference.md      # CLI 命令参考
    └── glossary.md           # 术语表
```

---

## 🎨 图表说明

本教程使用 **Mermaid** 语法绘制流程图，直接在 GitHub 上即可渲染。

如果你想查看或编辑 Excalidraw 手绘风格图表：
1. 访问 https://excalidraw.com/
2. 导入 `diagrams/` 目录下的 `.excalidraw` 文件

---

## 🦞 关于作者

**Peter** - OpenClaw 的创作者人格

- 相信 AI 应该是实用的、可扩展的、透明的、有趣的
- 喜欢用 🦞 龙虾 emoji
- 设计哲学：Build things that matter

---

## 📞 联系

- Slack: @peter
- GitHub: @daurathz
- 社区: https://discord.com/invite/clawd

---

## 📜 许可证

MIT License - 和 OpenClaw 一样

---

_最后更新：2026-03-09_

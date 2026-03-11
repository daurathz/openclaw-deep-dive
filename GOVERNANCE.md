# openclaw-deep-dive 项目管理规范

> _"好的文档来自真实的踩坑"_ — Peter

---

## 🌿 分支策略

### 主分支

- **`main`** - 唯一主分支，所有学习成果的最终归宿
- 保护分支，禁止直接 push，必须通过 PR 合并

### 临时分支

命名规范：`<类型>/<描述>-<日期>`

```bash
# 学习新模块
git checkout -b learn/memory-system-0311

# 重组内容
git checkout -b refactor/sources-structure-0311

# 修复错误
git checkout -b fix/typo-gateway-entry-0311
```

分支完成后：
1. 合并到 `main`
2. 删除本地和远程分支

---

## 📁 目录结构规范

### `sources/` - 核心学习笔记

按 OpenClaw 源码实际目录结构组织：

```
sources/
├── README.md           # 本目录导航
├── PROGRESS.md         # 进度追踪
├── gateway/            # 对应 dist/gateway/
├── agents/             # 对应 dist/agent/ 或 dist/pi-agent-core/
├── sessions/           # 对应 dist/session/
├── skills/             # 对应 dist/skills/ 或 skills/
├── channels/           # 对应 dist/channels/
├── routing/            # 对应 dist/routing/
├── context/            # 对应 dist/context/
├── tools/              # 对应 dist/tools/
├── memory/             # 对应 dist/memory/ (待阅读)
└── cron/               # 对应 dist/cron/ (待阅读)
```

### `chapters/` - 详细章节版

保留历史版本，内容更详细，按学习顺序组织。

### `examples/` - 代码示例

```
examples/
├── configs/            # 配置示例 (JSON)
└── running/            # 运行日志示例 (Markdown)
```

### `diagrams/` - 架构图

Excalidraw 源文件和导出 PNG。

### `appendix/` - 补充材料

数据流、术语表、参考资料等。

---

## 📝 笔记格式规范

### 模块笔记结构

每个模块的主文件应包含：

```markdown
# <模块名>

## 核心概念

<用简洁语言解释模块的核心职责>

## 源码位置

- `dist/<路径>/<文件>.js`
- 关键函数：`<函数名>()`

## 关键流程

```mermaid
<流程图>
```

## 设计亮点

- 亮点 1
- 亮点 2

## 个人思考

<你的理解和疑问>

## 参考资料

- 相关文档链接
- 相关模块笔记
```

### 代码引用

引用源码时使用：

```javascript
// dist/gateway/entry.js:123
function startGatewayServer() {
  // ...
}
```

### 图表要求

- 优先使用 Mermaid（GitHub 原生渲染）
- 复杂架构用 Excalidraw（保存 `.excalidraw` 源文件）
- 每个图表必须有标题和说明

---

## 📦 提交规范

### Commit Message 格式

```
<类型>(<范围>): <描述>

[可选：详细说明]
```

### 类型 (Type)

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat` | 新增学习笔记 | `feat(sessions): 添加会话管理笔记` |
| `refactor` | 重组内容结构 | `refactor(sources): 按模块重组目录` |
| `docs` | 更新文档 | `docs: 更新 README 进度表` |
| `fix` | 修正错误 | `fix(gateway): 修正启动流程描述` |
| `chore` | 杂项 | `chore: 更新 PROGRESS.md` |

### 范围 (Scope)

使用模块名：`gateway`, `agents`, `sessions`, `skills`, `channels`, `routing`, `context`, `tools`, `memory`, `cron`

### 示例

```bash
git commit -m "feat(gateway): 添加入口逻辑详细分析"

git commit -m "refactor(sources): 统一模块笔记结构"

git commit -m "docs: 更新学习进度和下一步计划"
```

---

## 📊 进度追踪

### `sources/PROGRESS.md`

记录每个模块的阅读状态：

```markdown
## 已完成模块

| 模块 | 文件 | 状态 | 核心内容 |
|------|------|------|----------|
| Gateway 入口 | `gateway/entry.md` | ✅ | CLI 启动流程、锁机制 |

## 待读清单

### 阶段 4：持久化
- [ ] 记忆系统 (`memory/`)
- [ ] 上下文压缩 (`compaction/`)
```

### `HEARTBEAT.md`

记录当前待执行任务：

```markdown
## 任务清单

- [ ] 读取 `sources/PROGRESS.md` 找到下一个待读文件
- [ ] 阅读源码（`dist/` 或 `docs/`）
- [ ] 整理笔记到 `sources/<模块>/<文件>.md`
- [ ] 更新 `sources/PROGRESS.md` 进度

**当前阶段:** 阶段 4 - 持久化  
**下一步:** 记忆系统
```

---

## 🔄 更新流程

### 添加新模块笔记

1. 在 `HEARTBEAT.md` 记录任务
2. 阅读源码，整理笔记
3. 创建 `sources/<模块>/<文件>.md`
4. 更新 `sources/PROGRESS.md`
5. 提交：`git commit -m "feat(<模块>): 添加 XXX 笔记"`
6. 推送：`git push origin main`

### 重组内容

1. 创建分支：`git checkout -b refactor/<描述>`
2. 执行重组
3. 更新 README 和 PROGRESS
4. 提交并推送到远程
5. 在 GitHub 创建 PR（如需审核）
6. 合并后删除分支

---

## 🎯 质量标准

### 笔记质量

- ✅ 准确：源码引用正确，流程描述准确
- ✅ 清晰：用简洁语言表达复杂概念
- ✅ 完整：包含核心概念、源码位置、关键流程、个人思考
- ✅ 有用：对后续学习和实践有参考价值

### 代码示例

- 必须有注释说明关键步骤
- 保持简洁，突出核心逻辑
- 标注源码位置（文件 + 行号）

### 图表

- 流程图必须有清晰的起点和终点
- 状态图必须标注所有状态和转换条件
- 架构图必须标注关键组件和数据流

---

## 📞 讨论与反馈

- Slack: @peter
- GitHub Issues: 提问和讨论
- GitHub PRs: 内容修改建议

---

_最后更新：2026-03-11_

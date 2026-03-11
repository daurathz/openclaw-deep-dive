# Cron & Heartbeat 配置实战指南

**创建时间:** 2026-03-11  
**目标读者:** OpenClaw 新手  
**目标:** 配置 Agent 实现主动效果（定时任务 + 定期检查）

---

## 🎯 学习目标

完成本指南后，你将能够：

1. ✅ 配置 Cron 定时任务，让 Agent 自动执行工作
2. ✅ 配置 Heartbeat 心跳检查，让 Agent 主动汇报进度
3. ✅ 理解 Cron 和 Heartbeat 的区别和使用场景
4. ✅ 实现一个完整的主动式 Agent 配置

---

## 📋 前置要求

- ✅ OpenClaw 已安装并运行
- ✅ Gateway 正常启动 (`openclaw gateway status`)
- ✅ 至少配置一个 Agent
- ✅ 基本的命令行操作知识

---

## 第一部分：Cron 定时任务配置

### 1.1 什么是 Cron？

**Cron** 是 OpenClaw 的定时任务调度系统，可以让 Agent 在指定时间自动执行任务。

**使用场景:**
- 📊 每日/每周报告生成
- 📥 定时数据采集
- 🔔 定时提醒通知
- 🤖 自动化工作流

### 1.2 Cron 三种调度方式

| 类型 | 参数 | 说明 | 示例 |
|------|------|------|------|
| 一次性 | `--at` | 在指定时间运行一次 | `--at +1h` (1 小时后) |
| 周期性 | `--every` | 每隔固定时间运行 | `--every 10m` (每 10 分钟) |
| Cron 表达式 | `--cron` | 复杂的定时规则 | `--cron "0 9 * * *"` (每天 9 点) |

### 1.3 实战案例 1：每日报告任务

**场景:** 每天早上 9 点生成汽车后市场报告

```bash
openclaw cron add \
  --name daisy-daily-report \
  --description "每日汽车后市场报告" \
  --cron "0 9 * * *" \
  --tz Asia/Shanghai \
  --message "生成每日汽车后市场报告，整合昨日数据形成多元观点" \
  --session isolated \
  --agent daisy \
  --model bailian/qwen3.5-plus \
  --announce \
  --channel slack \
  --timeout-seconds 600
```

**参数详解:**
- `--name`: 任务名称（唯一标识）
- `--description`: 任务描述（便于识别）
- `--cron "0 9 * * *"`: 每天 9:00 执行（5 字段 cron）
- `--tz Asia/Shanghai`: 时区设置（北京时间）
- `--message`: 发送给 Agent 的消息
- `--session isolated`: 在隔离会话执行（不影响主会话）
- `--agent daisy`: 指定执行 Agent
- `--announce`: 完成后发送通知
- `--channel slack`: 通知渠道
- `--timeout-seconds 600`: 超时时间（10 分钟）

**Cron 表达式速查:**
```
* * * * *
│ │ │ │ │
│ │ │ │ └─ 星期几 (0-7, 0 和 7 都是周日)
│ │ │ └─── 月份 (1-12)
│ │ └───── 日期 (1-31)
│ └─────── 小时 (0-23)
└───────── 分钟 (0-59)

常用示例:
"0 9 * * *"     → 每天 9:00
"0 */2 * * *"   → 每 2 小时
"*/10 * * * *"  → 每 10 分钟
"0 9 * * 1-5"   → 工作日 9:00
"0 0 1 * *"     → 每月 1 号 0:00
```

### 1.4 实战案例 2：周期性数据采集

**场景:** 每 10 分钟采集一次数据（00:00-07:00）

```bash
openclaw cron add \
  --name daisy-data-collection \
  --description "汽车后市场数据采集" \
  --every 10m \
  --message "采集汽车后市场最新数据，记录信息来源和发布日期" \
  --session isolated \
  --agent daisy \
  --announce \
  --channel slack \
  --best-effort-deliver
```

**说明:**
- `--every 10m`: 每 10 分钟执行一次
- `--best-effort-deliver`: 通知失败不标记任务失败

**⚠️ 注意:** `--every` 会从添加任务时开始计算周期，不是固定时间点。

### 1.5 实战案例 3：定时提醒任务

**场景:** 每工作 1 小时提醒休息

```bash
openclaw cron add \
  --name rest-reminder \
  --description "休息提醒" \
  --every 1h \
  --message "⏰ 休息时间到！起来活动一下，喝杯水吧~ 🦞" \
  --session main \
  --system-event "rest-reminder" \
  --announce \
  --channel slack
```

### 1.6 实战案例 4：一次性任务

**场景:** 1 小时后执行备份

```bash
openclaw cron add \
  --name backup-task \
  --description "数据备份" \
  --at +1h \
  --message "执行数据备份，保存到备份目录" \
  --session isolated \
  --agent main \
  --delete-after-run
```

**说明:**
- `--at +1h`: 1 小时后执行（支持 `+20m`, `+1h`, `+1d`）
- `--delete-after-run`: 执行后自动删除（一次性任务）

### 1.7 管理 Cron 任务

#### 查看任务列表

```bash
# 查看所有任务
openclaw cron list

# 查看包括禁用的任务
openclaw cron list --all

# JSON 格式输出
openclaw cron list --json
```

**示例输出:**
```
ID                                  Name                    Schedule                        Next        Last        Status    Target    Agent ID    Model
----------------------------------  ----------------------  --------------------------------  ----------  ----------  --------  --------  ----------  --------------------
daisy-data-collection               Daisy 数据采集          every 10m                         in 3m       2m ago      ok        isolated  daisy       bailian/qwen3.5-plus
daisy-daily-report                  Daisy 每日报告          cron 0 9 * * * @ Asia/Shanghai    in 23h      1d ago      ok        isolated  daisy       bailian/qwen3.5-plus
rest-reminder                       休息提醒                every 1h                          in 45m      15m ago     ok        main      main        bailian/qwen3.5-plus
```

#### 查看调度器状态

```bash
# 查看状态
openclaw cron status

# JSON 格式
openclaw cron status --json
```

**示例输出:**
```json
{
  "enabled": true,
  "storePath": "~/.openclaw/cron-store.json",
  "nextRunAt": "2026-03-12T09:00:00.000Z",
  "totalJobs": 3,
  "activeJobs": 3
}
```

#### 删除任务

```bash
# 删除任务
openclaw cron remove daisy-data-collection

# 禁用任务（不删除）
openclaw cron update daily-report --disabled
```

### 1.8 高级配置

#### 随机延迟防抖

防止多个任务同时执行：

```bash
openclaw cron add \
  --name hourly-sync \
  --cron "0 * * * *" \
  --stagger 5m \
  --message "整点同步数据"
```

**说明:**
- `--stagger 5m`: 在整点后的 0-5 分钟内随机执行
- 避免多个任务同时触发造成资源竞争

#### 禁用随机延迟

```bash
openclaw cron add \
  --name exact-task \
  --cron "0 9 * * *" \
  --exact \
  --message "精确时间执行"
```

#### 配置通知渠道

```bash
# 发送到特定渠道
openclaw cron add \
  --name report \
  --cron "0 9 * * *" \
  --channel slack \
  --to "+8613800138000" \
  --account daisy \
  --announce
```

**参数:**
- `--channel`: slack | telegram | discord | last
- `--to`: 目标地址（手机号/聊天 ID）
- `--account`: 多账号配置时的账号 ID

---

## 第二部分：Heartbeat 心跳配置

### 2.1 什么是 Heartbeat？

**Heartbeat** 是 Agent 的定期检查机制，让 Agent 主动检查工作状态并汇报进度。

**使用场景:**
- 📊 每 5 分钟同步工作进度
- 🔍 定期检查任务状态
- 📝 自动更新 HEARTBEAT.md
- 🦞 主动询问用户是否需要帮助

### 2.2 Cron vs Heartbeat

| 特性 | Cron | Heartbeat |
|------|------|-----------|
| **触发方式** | 定时触发 | 定期检查 |
| **执行内容** | 预设任务 | 读取 HEARTBEAT.md |
| **配置位置** | CLI 命令 | `openclaw.json` |
| **灵活性** | 固定任务 | 动态任务 |
| **适用场景** | 自动化工作流 | 进度同步/主动检查 |

**最佳实践:**
- **Cron** → 执行具体的定时任务（报告、采集等）
- **Heartbeat** → 检查工作状态、同步进度、询问用户

### 2.3 配置 Heartbeat

#### 步骤 1：编辑配置文件

打开 `~/.openclaw/openclaw.json`：

```bash
# 使用编辑器打开
code ~/.openclaw/openclaw.json
# 或
vim ~/.openclaw/openclaw.json
```

#### 步骤 2：添加 Heartbeat 配置

在 `agents.list` 中找到你的 Agent，添加 `heartbeat` 配置：

```json5
{
  "agents": {
    "list": [
      {
        "id": "peter",
        "name": "Peter - OpenClaw 创作者",
        "workspace": "/home/openclaw/.openclaw/workspace-peter",
        "agentDir": "/home/openclaw/.openclaw/agents/peter/agent",
        "heartbeat": {
          "every": "30m",                    // 检查间隔
          "model": "bailian/qwen3.5-plus",   // 使用的模型
          "session": "main",                 // 会话类型
          "directPolicy": "allow",           // 直接执行策略
          "target": "last",                  // 通知目标
          "lightContext": true,              // 轻量上下文
          "suppressToolErrorWarnings": false,
          "prompt": "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK."
        }
      }
    ]
  }
}
```

#### 步骤 3：配置参数详解

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `every` | 检查间隔 | `5m` (工作同步) / `30m` (常规检查) |
| `model` | 使用的模型 | 根据需求选择 |
| `session` | 会话类型 | `main` (主会话) |
| `directPolicy` | 执行策略 | `allow` (直接执行) |
| `target` | 通知目标 | `last` (上次聊天) |
| `lightContext` | 轻量上下文 | `true` (节省 token) |
| `prompt` | 提示词 | 见下方模板 |

#### 步骤 4：创建 HEARTBEAT.md

在工作空间创建 `HEARTBEAT.md`：

```markdown
# HEARTBEAT.md - 当前任务状态

**最后更新:** 2026-03-11 23:30  
**状态:** 正常运行中

---

## ✅ 已完成任务

- [x] 配置 Cron 定时任务
- [x] 配置 Heartbeat 心跳检查

---

## 📋 待执行任务

- [ ] 继续源码阅读
- [ ] 整理学习笔记

---

## 🎯 下一步建议

等待用户指示

---

_如无任务需要执行，回复 HEARTBEAT_OK_
```

**⚠️ 重要:** 
- 保持文件简洁（避免 token 浪费）
- 每次心跳后更新状态
- 无事时回复 `HEARTBEAT_OK`

### 2.4 Heartbeat 提示词模板

#### 模板 1：进度同步型

```markdown
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.

When there are active tasks:
- Report progress every 5 minutes
- Include completed items and next steps
- Ask if user wants to adjust priorities
```

#### 模板 2：主动询问型

```markdown
Read HEARTBEAT.md if it exists. Check for:
1. Active tasks and their status
2. Blocked items needing user input
3. Completed work to celebrate

If nothing needs attention, reply HEARTBEAT_OK.
If there are updates, send a brief progress report.
```

#### 模板 3：简单检查型

```markdown
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.
```

### 2.5 实战案例：5 分钟进度同步

**场景:** 用户希望每 5 分钟同步一次工作进度

#### 配置 openclaw.json

```json5
{
  "agents": {
    "list": [
      {
        "id": "peter",
        "name": "Peter - OpenClaw 创作者",
        "workspace": "/home/openclaw/.openclaw/workspace-peter",
        "heartbeat": {
          "every": "5m",
          "model": "bailian/qwen3.5-plus",
          "session": "main",
          "directPolicy": "allow",
          "target": "last",
          "lightContext": true,
          "prompt": "Read HEARTBEAT.md if it exists. Report progress on active tasks. If work is in progress, send a brief update every 5 minutes. When work completes, stop updates and send final report. If nothing needs attention, reply HEARTBEAT_OK."
        }
      }
    ]
  }
}
```

#### 配置 HEARTBEAT.md

```markdown
# HEARTBEAT.md - 源码阅读任务

**最后更新:** 2026-03-11 23:30  
**状态:** 进行中

---

## 📋 当前任务

### 阶段 5 源码阅读 🔄 进行中
- [x] Cron 调度系统笔记 ✅
- [x] Hooks 系统笔记 ✅
- [ ] Subagents 系统笔记 ⏳ 进行中

---

## ⏱️ 进度统计

- **开始时间:** 23:00
- **当前进度:** 2/3 (67%)
- **预计完成:** 23:45

---

## 🚧 阻塞/问题

无

---

## 🎯 下一步

完成 Subagents 笔记，然后提交到 Git

---

_工作进行时每 5 分钟同步一次进度，工作完成后停止同步直接报告_
```

### 2.6 启用/禁用 Heartbeat

#### 临时禁用

编辑 `openclaw.json`，注释掉 heartbeat 配置：

```json5
{
  "agents": {
    "list": [
      {
        "id": "peter",
        // "heartbeat": { ... }  // 注释掉
      }
    ]
  }
}
```

#### 调整间隔

```json5
{
  "heartbeat": {
    "every": "30m"  // 改为 30 分钟
  }
}
```

#### 重启 Gateway

配置更改后需要重启 Gateway：

```bash
openclaw gateway restart
```

---

## 第三部分：完整实战案例

### 案例：构建主动式数据报告 Agent

**目标:** 配置一个自动采集数据并生成报告的 Agent

#### 步骤 1：配置 Cron 任务

```bash
# 1. 数据采集（每 10 分钟）
openclaw cron add \
  --name daisy-data-collection \
  --description "汽车后市场数据采集" \
  --every 10m \
  --message "采集汽车后市场最新数据，记录信息来源和发布日期" \
  --session isolated \
  --agent daisy \
  --model bailian/qwen3.5-plus \
  --announce \
  --channel slack \
  --best-effort-deliver

# 2. 每日报告（每天 9 点）
openclaw cron add \
  --name daisy-daily-report \
  --description "每日汽车后市场报告" \
  --cron "0 9 * * *" \
  --tz Asia/Shanghai \
  --message "生成每日汽车后市场报告，整合新旧信息形成多元观点" \
  --session isolated \
  --agent daisy \
  --model bailian/qwen3.5-plus \
  --announce \
  --channel slack \
  --timeout-seconds 600
```

#### 步骤 2：配置 Heartbeat

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  "agents": {
    "list": [
      {
        "id": "daisy",
        "name": "Daisy - 汽车后市场专家",
        "workspace": "/home/openclaw/.openclaw/workspace-daisy",
        "heartbeat": {
          "every": "30m",
          "model": "bailian/qwen3.5-plus",
          "session": "main",
          "directPolicy": "allow",
          "target": "last",
          "lightContext": true,
          "prompt": "Read HEARTBEAT.md if it exists. Check data collection status and report progress. If issues detected, alert user immediately. If nothing needs attention, reply HEARTBEAT_OK."
        }
      }
    ]
  }
}
```

#### 步骤 3：创建 HEARTBEAT.md

```markdown
# HEARTBEAT.md - Daisy 数据报告任务

**最后更新:** 2026-03-11 23:30  
**状态:** 正常运行

---

## ✅ 自动任务

- [x] 数据采集 (每 10 分钟) ✅
- [x] 每日报告 (9:00) ✅

---

## 📊 最近状态

- **上次采集:** 23:20 ✅
- **上次报告:** 09:00 ✅
- **数据源数量:** 15

---

## 🚧 问题/警告

无

---

_每 30 分钟检查一次数据状态_
```

#### 步骤 4：重启 Gateway

```bash
openclaw gateway restart
```

#### 步骤 5：验证配置

```bash
# 查看 Cron 任务
openclaw cron list

# 查看 Gateway 状态
openclaw gateway status

# 查看 Heartbeat 配置
cat ~/.openclaw/openclaw.json | grep -A 10 heartbeat
```

---

## 第四部分：故障诊断

### 问题 1：Cron 任务不执行

**症状:** 任务列表中有任务，但不执行

**检查步骤:**

```bash
# 1. 检查调度器状态
openclaw cron status

# 2. 查看是否禁用
openclaw cron list --all

# 3. 检查 Gateway 日志
# (查看 Gateway 运行日志)
```

**常见原因:**
- ❌ 调度器禁用 (`cron.enabled: false`)
- ❌ 任务禁用 (`--disabled` 参数)
- ❌ Gateway 未重启

**解决方法:**

```json5
// openclaw.json 中添加/修改
{
  "cron": {
    "enabled": true  // 确保启用
  }
}
```

### 问题 2：Heartbeat 不触发

**症状:** 配置了 Heartbeat 但没有定期检查

**检查步骤:**

```bash
# 1. 检查配置语法
cat ~/.openclaw/openclaw.json | jq '.agents.list[].heartbeat'

# 2. 检查 Gateway 是否重启
openclaw gateway status

# 3. 查看 HEARTBEAT.md 是否存在
ls ~/.openclaw/workspace-*/HEARTBEAT.md
```

**常见原因:**
- ❌ 配置语法错误
- ❌ Gateway 未重启
- ❌ HEARTBEAT.md 不存在

**解决方法:**
1. 验证 JSON 语法
2. 重启 Gateway
3. 创建 HEARTBEAT.md

### 问题 3：通知不发送

**症状:** 任务执行了但没有收到通知

**检查步骤:**

```bash
# 1. 检查通道配置
openclaw channels list

# 2. 检查账号绑定
openclaw accounts list

# 3. 查看任务详情
openclaw cron list --json
```

**常见原因:**
- ❌ 通道未配置
- ❌ 账号未绑定
- ❌ `--announce` 参数缺失

**解决方法:**
```bash
# 重新配置通道
openclaw configure channels

# 添加 announce 参数
openclaw cron add --announce --channel slack ...
```

### 问题 4：Token 消耗过快

**症状:** Heartbeat 导致 token 消耗过快

**解决方法:**

```json5
{
  "heartbeat": {
    "every": "30m",        // 增加间隔
    "lightContext": true,  // 启用轻量上下文
    "prompt": "简短提示词"  // 缩短提示词
  }
}
```

---

## 第五部分：最佳实践

### Cron 最佳实践

1. **命名规范**
   ```bash
   # 好
   --name daisy-daily-report
   --name data-collection-10m
   
   # 不好
   --name test
   --name cron1
   ```

2. **添加描述**
   ```bash
   --description "每日 9 点生成汽车后市场报告"
   ```

3. **设置超时**
   ```bash
   --timeout-seconds 600  # 10 分钟超时
   ```

4. **使用隔离会话**
   ```bash
   --session isolated  # 不影响主会话
   ```

5. **配置随机延迟**
   ```bash
   --stagger 5m  # 避免整点并发
   ```

### Heartbeat 最佳实践

1. **合理设置间隔**
   - 工作同步：`5m`
   - 常规检查：`30m`
   - 低频监控：`1h`

2. **保持 HEARTBEAT.md 简洁**
   ```markdown
   # 好 - 简洁明了
   - [x] 任务 1 ✅
   - [ ] 任务 2 ⏳
   
   # 不好 - 过于冗长
   [大量详细日志...]
   ```

3. **使用轻量上下文**
   ```json5
   {
     "heartbeat": {
       "lightContext": true
     }
   }
   ```

4. **明确提示词**
   ```markdown
   If nothing needs attention, reply HEARTBEAT_OK.
   ```

5. **定期清理**
   - 完成任务后更新状态
   - 删除不再需要的任务

---

## 📚 相关文档

- [Cron 调度系统源码笔记](../sources/05-cron/cron-scheduler.md)
- [Agent Workspace 核心文件](./B-agent-workspace-files.md)
- [OpenClaw Docs - Cron](https://docs.openclaw.ai/cli/cron)
- [OpenClaw Docs - Agent Config](https://docs.openclaw.ai/gateway/agent-config)

---

_最后更新：2026-03-11 | 🦞 OpenClaw 源码阅读计划_

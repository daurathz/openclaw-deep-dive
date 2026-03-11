# Cron 调度系统

**阅读时间:** 2026-03-11 15:55  
**源码路径:** `dist/cron-cli-C7-1zdvT.js`, `dist/stagger-Bg3n7FlQ.js`  
**核心职责:** 定时任务调度与管理

---

## 核心概念

Cron 系统提供 OpenClaw 的定时任务调度能力，支持：
- **一次性任务** (`--at`): 在指定时间运行一次
- **周期性任务** (`--every`): 每隔固定时间运行
- **Cron 表达式** (`--cron`): 复杂的定时规则（5 字段或 6 字段）

---

## CLI 命令结构

### 命令注册流程

```
cron-cli.ts
├── registerCronStatusCommand()   # cron status
├── registerCronListCommand()     # cron list
├── registerCronAddCommand()      # cron add/create
└── registerCronRemoveCommand()   # cron remove
```

### 关键命令选项

| 选项 | 说明 | 示例 |
|------|------|------|
| `--name` | 任务名称（必填） | `--name daisy-daily-report` |
| `--every` | 周期运行 | `--every 10m` |
| `--cron` | Cron 表达式 | `--cron "0 9 * * *"` |
| `--at` | 一次性运行 | `--at 2026-03-11T10:00:00Z` 或 `--at +20m` |
| `--message` | Agent 消息 payload | `--message "生成日报"` |
| `--system-event` | 系统事件 payload | `--system-event "daily-backup"` |
| `--session` | 会话目标 | `--session isolated` |
| `--announce` | 发送完成通知 | `--announce` |
| `--channel` | 通知渠道 | `--channel slack` |
| `--stagger` | 随机延迟窗口 | `--stagger 5m` |
| `--exact` | 禁用随机延迟 | `--exact` |

---

## 调度类型解析

### 1. 一次性任务 (`at`)

```typescript
function parseAt(input: string): string | null {
  // 支持绝对时间：2026-03-11T10:00:00Z
  const absolute = parseAbsoluteTimeMs(raw);
  // 支持相对时间：+20m, +1h, +1d
  const dur = parseDurationMs(raw);
  return new Date(Date.now() + dur).toISOString();
}
```

### 2. 周期性任务 (`every`)

```typescript
{
  kind: "every",
  everyMs: 600000  // 10 分钟
}
```

### 3. Cron 表达式 (`cron`)

```typescript
{
  kind: "cron",
  expr: "0 9 * * *",      // 5 字段 cron
  tz: "Asia/Shanghai",    // 可选时区
  staggerMs: 300000       // 随机延迟 5 分钟
}
```

---

## 时间解析机制

### 持续时间解析

```typescript
function parseDurationMs(input: string): number | null {
  const match = raw.match(/^(\d+(?:\.\d+)?)(ms|s|m|h|d)$/i);
  const factor = {
    ms: 1,
    s: 1000,
    m: 60000,
    h: 3600000,
    d: 86400000
  }[unit];
  return Math.floor(n * factor);
}
```

### 随机延迟 (Stagger)

防止大量任务同时执行的机制：

```typescript
function parseCronStaggerMs(params): number {
  if (params.useExact) return 0;  // --exact 禁用延迟
  if (!params.staggerRaw) return;  // 无配置则使用默认
  return parseDurationMs(params.staggerRaw);  // 如 30s, 5m
}
```

---

## 任务 Payload 类型

### 1. 系统事件 (System Event)

用于主会话 (`--session main`):

```typescript
{
  kind: "systemEvent",
  text: "daily-backup"
}
```

### 2. Agent 消息 (Agent Turn)

用于隔离会话 (`--session isolated`):

```typescript
{
  kind: "agentTurn",
  message: "生成每日汽车后市场报告",
  model: "bailian/qwen3.5-plus",
  thinking: "medium",
  timeoutSeconds: 300,
  lightContext: true
}
```

---

## 通知交付 (Delivery)

### 交付模式

```typescript
const deliveryMode = sessionTarget === "isolated" && payload.kind === "agentTurn"
  ? hasAnnounce ? "announce" 
  : hasNoDeliver ? "none" 
  : "announce"
  : undefined;
```

### 交付配置

```typescript
{
  mode: "announce",           // announce | none
  channel: "slack",           // slack | telegram | discord | last
  to: "+8613800138000",       // 目标地址（可选）
  account: "daisy"            // 多账号配置（可选）
}
```

---

## 状态显示

### 列表输出格式

```
ID                                  Name                    Schedule                        Next        Last        Status    Target    Agent ID    Model
----------------------------------  ----------------------  --------------------------------  ----------  ----------  --------  --------  ----------  --------------------
daisy-data-collection               Daisy 数据采集          every 10m                         in 3m       2m ago      ok        isolated  daisy       bailian/qwen3.5-plus
daisy-daily-report                  Daisy 每日报告          cron 0 9 * * * @ Asia/Shanghai    in 23h      1d ago      ok        isolated  daisy       bailian/qwen3.5-plus
```

### 状态类型

| 状态 | 说明 |
|------|------|
| `ok` | 上次运行成功 |
| `error` | 上次运行失败 |
| `running` | 正在运行中 |
| `skipped` | 跳过（如重叠执行） |
| `idle` | 空闲（未运行过） |
| `disabled` | 已禁用 |

---

## 设计亮点

### 1. 渐进式配置

- 支持 `--disabled` 创建禁用任务
- 支持 `--delete-after-run` 一次性任务自动清理
- 支持 `--keep-after-run` 保留执行记录

### 2. 灵活的调度方式

- 绝对时间：`--at 2026-03-11T10:00:00Z`
- 相对时间：`--at +20m`
- 周期运行：`--every 10m`
- Cron 表达式：`--cron "0 9 * * *"`

### 3. 随机延迟防抖

```bash
# 多个任务在整点触发时，随机延迟 0-5 分钟
cron add --cron "0 * * * *" --stagger 5m
```

### 4. 会话隔离

- `--session main`: 系统事件，在主会话执行
- `--session isolated`: Agent 任务，在隔离会话执行

### 5. 通知交付

- `--announce`: 发送完成通知到指定渠道
- `--no-deliver`: 不发送通知
- `--channel`: 指定渠道（slack/telegram/discord/last）
- `--best-effort-deliver`: 交付失败不标记任务失败

---

## 关键函数

### Gateway RPC 调用

```typescript
// 状态查询
await callGatewayFromCli("cron.status", opts, {});

// 任务列表
await callGatewayFromCli("cron.list", opts, { includeDisabled: true });

// 添加任务
await callGatewayFromCli("cron.add", opts, {
  name, description, enabled, schedule, payload, delivery
});

// 删除任务
await callGatewayFromCli("cron.remove", opts, { id });
```

### 调度器状态检查

```typescript
async function warnIfCronSchedulerDisabled(opts) {
  const res = await callGatewayFromCli("cron.status", opts, {});
  if (res?.enabled === true) return;
  // 警告：调度器已禁用
}
```

---

## 使用示例

### 1. 每 10 分钟数据采集

```bash
openclaw cron add \
  --name daisy-data-collection \
  --every 10m \
  --message "采集汽车后市场数据" \
  --session isolated \
  --agent daisy
```

### 2. 每日 9 点生成报告

```bash
openclaw cron add \
  --name daisy-daily-report \
  --cron "0 9 * * *" \
  --tz Asia/Shanghai \
  --message "生成每日汽车后市场报告" \
  --session isolated \
  --agent daisy \
  --announce \
  --channel slack
```

### 3. 一次性任务

```bash
openclaw cron add \
  --name backup-task \
  --at +1h \
  --system-event "backup" \
  --session main \
  --delete-after-run
```

### 4. 带随机延迟的整点任务

```bash
openclaw cron add \
  --name hourly-sync \
  --cron "0 * * * *" \
  --stagger 5m \
  --message "同步数据"
```

---

## 关联概念

- [[05-hooks]] - 钩子系统（生命周期事件）
- [[01-agents/runtime]] - Agent 运行时（任务执行环境）
- [[01-sessions/management]] - 会话管理（隔离会话）

---

**备注:** Cron 调度器状态可通过 `openclaw cron status` 查看，如显示 disabled 需要在配置中设置 `cron.enabled: true` 并重启 Gateway。

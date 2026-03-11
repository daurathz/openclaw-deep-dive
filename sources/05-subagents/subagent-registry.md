# Subagents 子 Agent 系统

**阅读时间:** 2026-03-11 16:05  
**源码路径:** `dist/subagent-registry-runtime-BzpT7ZhE.js`, `dist/plugin-sdk/agents/subagent-registry.d.ts`, `dist/plugin-sdk/agents/tools/subagents-tool.d.ts`  
**核心职责:** 子 Agent 创建、管理与通信

---

## 核心概念

Subagents 系统允许一个 Agent 创建和管理多个子 Agent 实例，实现：

- **任务分解**: 将复杂任务拆分为多个子任务
- **并行执行**: 多个子 Agent 同时工作
- **结果聚合**: 收集并整合子 Agent 的输出
- **资源隔离**: 每个子 Agent 独立会话和资源

---

## Subagent 注册表

### 核心数据结构

```typescript
// dist/plugin-sdk/agents/subagent-registry.types.d.ts
type SubagentRunRecord = {
  runId: string;                    // 运行 ID
  childSessionKey: string;          // 子会话 Key
  requesterSessionKey: string;      // 请求者会话 Key
  requesterOrigin?: DeliveryContext; // 请求来源
  requesterDisplayKey: string;      // 请求者显示 Key
  task: string;                      // 任务描述
  cleanup: "delete" | "keep";       // 清理策略
  label?: string;                    // 标签
  model?: string;                    // 模型覆盖
  workspaceDir?: string;            // 工作空间
  runTimeoutSeconds?: number;       // 超时时间
  spawnMode?: "run" | "session";    // 生成模式
  
  // 生命周期时间戳
  createdAt: number;
  startedAt?: number;
  endedAt?: number;
  cleanupCompletedAt?: number;
  
  // 执行结果
  outcome?: SubagentRunOutcome;
  frozenResultText?: string | null;  // 冻结结果
  fallbackFrozenResultText?: string | null;  // 备用结果
  
  // 附件管理
  attachmentsDir?: string;
  attachmentsRootDir?: string;
  retainAttachmentsOnKeep?: boolean;
  
  // 通知控制
  expectsCompletionMessage?: boolean;
  announceRetryCount?: number;
  suppressAnnounceReason?: "steer-restart" | "killed";
  
  // 生命周期钩子
  endedHookEmittedAt?: number;
  endedReason?: SubagentLifecycleEndedReason;
  wakeOnDescendantSettle?: boolean;
};
```

---

## Subagent 工具

### 创建 Subagents 工具

```typescript
// dist/plugin-sdk/agents/tools/subagents-tool.d.ts
export declare function createSubagentsTool(opts?: {
  agentSessionKey?: string;
}): AnyAgentTool;
```

### 使用方式

在 Agent 中调用子 Agent：

```typescript
// 通过工具调用
const result = await tools.subagents({
  task: "分析这份报告的数据趋势",
  model: "bailian/qwen3-max-2026-01-23",
  timeout: 300,
  cleanup: "keep"
});
```

---

## 生命周期管理

### 1. 注册子 Agent 运行

```typescript
export declare function registerSubagentRun(params: {
  runId: string;
  childSessionKey: string;
  requesterSessionKey: string;
  requesterOrigin?: DeliveryContext;
  requesterDisplayKey: string;
  task: string;
  cleanup: "delete" | "keep";
  label?: string;
  model?: string;
  workspaceDir?: string;
  runTimeoutSeconds?: number;
  expectsCompletionMessage?: boolean;
  spawnMode?: "run" | "session";
  attachmentsDir?: string;
  attachmentsRootDir?: string;
  retainAttachmentsOnKeep?: boolean;
}): void;
```

### 2. 查询子 Agent 状态

```typescript
// 检查是否活跃
export declare function isSubagentSessionRunActive(
  childSessionKey: string
): boolean;

// 列出请求者的所有子 Agent 运行
export declare function listSubagentRunsForRequester(
  requesterSessionKey: string,
  options?: { requesterRunId?: string }
): SubagentRunRecord[];

// 统计活跃运行数
export declare function countActiveRunsForSession(
  requesterSessionKey: string
): number;

// 统计后代运行数（嵌套子 Agent）
export declare function countActiveDescendantRuns(
  rootSessionKey: string
): number;
```

### 3. 释放子 Agent

```typescript
export declare function releaseSubagentRun(runId: string): void;
```

---

## 结果交付

### 冻结结果 (Frozen Result)

```typescript
/**
 * Latest frozen completion output captured for announce delivery.
 * Seeded at first end transition and refreshed by later assistant turns
 * while completion delivery is still pending for this session.
 */
frozenResultText?: string | null;
frozenResultCapturedAt?: number;
```

### 备用结果 (Fallback Result)

```typescript
/**
 * Fallback completion output preserved across wake continuation restarts.
 * Used when a late wake run replies with NO_REPLY after the real final
 * summary was already produced by the prior run.
 */
fallbackFrozenResultText?: string | null;
fallbackFrozenResultCapturedAt?: number;
```

### 完成通知

```typescript
export declare function shouldIgnorePostCompletionAnnounceForSession(
  childSessionKey: string
): boolean;
```

---

## Steer 重启机制

当子 Agent 需要重启时（如模型切换、配置更新）：

### 标记重启

```typescript
export declare function markSubagentRunForSteerRestart(
  runId: string
): boolean;
```

### 清除重启标记

```typescript
export declare function clearSubagentRunSteerRestart(
  runId: string
): boolean;
```

### 替换运行记录

```typescript
export declare function replaceSubagentRunAfterSteer(params: {
  previousRunId: string;
  nextRunId: string;
  fallback?: SubagentRunRecord;
  runTimeoutSeconds?: number;
  preserveFrozenResultFallback?: boolean;
}): boolean;
```

---

## 请求者解析

子会话需要知道它的请求者是谁：

```typescript
export declare function resolveRequesterForChildSession(
  childSessionKey: string
): {
  requesterSessionKey: string;
  requesterOrigin?: DeliveryContext;
} | null;
```

---

## 生命周期钩子

### 子 Agent 结束钩子

```typescript
/**
 * Set after the subagent_ended hook has been emitted successfully once.
 */
endedHookEmittedAt?: number;

/**
 * Terminal lifecycle reason recorded when the run finishes.
 */
endedReason?: SubagentLifecycleEndedReason;
```

### 后代唤醒

```typescript
/**
 * Run ended while descendants were still pending and should be re-invoked 
 * once they settle.
 */
wakeOnDescendantSettle?: boolean;
```

---

## 并发控制

### 最大并发数配置

在 `openclaw.json` 中配置：

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxConcurrent": 8
      }
    }
  }
}
```

### 统计函数

```typescript
// 统计活跃后代运行数
export declare function countActiveDescendantRuns(
  rootSessionKey: string
): number;

// 统计待处理后代运行数
export declare function countPendingDescendantRuns(
  rootSessionKey: string
): number;

// 排除特定运行的待处理数
export declare function countPendingDescendantRunsExcludingRun(
  rootSessionKey: string,
  excludeRunId: string
): number;

// 列出所有后代运行
export declare function listDescendantRunsForRequester(
  rootSessionKey: string
): SubagentRunRecord[];
```

---

## 生成模式

### Run 模式（一次性）

```typescript
spawnMode: "run"
```

- 执行单个任务
- 完成后自动清理（如配置）
- 适合简单查询/分析任务

### Session 模式（持久会话）

```typescript
spawnMode: "session"
```

- 创建持久会话
- 可多次交互
- 适合复杂协作任务

---

## 设计亮点

### 1. Actor 模型

每个子 Agent 运行作为独立 Actor，串行处理消息，避免并发冲突。

### 2. 结果冻结机制

```typescript
// 在结束时捕获结果
frozenResultText: string | null;
frozenResultCapturedAt: number;

// 支持多次刷新
// 在后续 assistant turns 中更新
```

### 3. 嵌套子 Agent 支持

子 Agent 可以创建自己的子 Agent（孙 Agent），系统自动追踪层级关系。

### 4. 灵活的清理策略

```typescript
cleanup: "delete" | "keep"

// delete: 完成后删除会话和附件
// keep: 保留会话历史
```

### 5. 附件管理

```typescript
attachmentsDir?: string;           // 附件目录
attachmentsRootDir?: string;       // 附件根目录
retainAttachmentsOnKeep?: boolean; // 保留附件
```

### 6. 通知重试机制

```typescript
announceRetryCount?: number;       // 重试次数
lastAnnounceRetryAt?: number;      // 最后重试时间
```

---

## 使用示例

### 1. 创建子 Agent 进行分析

```typescript
const analysis = await tools.subagents({
  task: "分析用户提供的代码，找出潜在的性能问题",
  model: "bailian/qwen3-coder-plus",
  timeout: 600,
  cleanup: "keep",
  label: "code-review"
});
```

### 2. 并行执行多个子任务

```typescript
const [data, report, summary] = await Promise.all([
  tools.subagents({ task: "采集数据", label: "data-collection" }),
  tools.subagents({ task: "生成报告", label: "report-generation" }),
  tools.subagents({ task: "创建摘要", label: "summary" })
]);
```

### 3. 嵌套子 Agent

```typescript
// 主 Agent 创建子 Agent
const child = await tools.subagents({
  task: "完成这个复杂任务",
  spawnMode: "session"
});

// 子 Agent 内部可以继续创建孙 Agent
const grandchild = await tools.subagents({
  task: "处理子任务中的特定部分"
});
```

---

## 关联概念

- [[05-cron/cron-scheduler]] - Cron 调度（可触发子 Agent 任务）
- [[01-agents/runtime]] - Agent 运行时（子 Agent 执行环境）
- [[01-sessions/management]] - 会话管理（子会话隔离）
- [[05-hooks/workspace-hooks]] - 钩子系统（子 Agent 生命周期钩子）

---

## CLI 命令（待实现）

```bash
# 列出子 Agent 运行
openclaw subagents list --session <session-key>

# 查看子 Agent 状态
openclaw subagents status <run-id>

# 终止子 Agent 运行
openclaw subagents kill <run-id>

# 查看子 Agent 日志
openclaw subagents logs <run-id>
```

---

**备注:** Subagents 系统是 OpenClaw 实现复杂任务分解和并行执行的核心机制，与 Skills 系统一起构成了 OpenClaw 的能力扩展框架。

# Agent Loop - 核心执行循环

**阅读时间:** 2026-03-10 12:55  
**源码路径:** `docs/concepts/agent-loop.md` + `dist/pi-embedded-*.js`  
**核心职责:** Agent 的完整执行循环：接收 → 推理 → 工具 → 输出

---

## 📖 关键发现

### 1. Agent Loop 的定义

```
Agent Loop = 一次完整的 "real" run
intake → context assembly → model inference → tool execution → streaming replies → persistence
```

**关键点:**
- 一个 loop = 单次序列化执行（per session）
- 发射 lifecycle 和 stream 事件
- 保持 session 状态一致

### 2. 入口点（Entry Points）

| 入口 | 说明 |
|------|------|
| `agent` RPC | Gateway WebSocket API |
| `agent.wait` | 等待完成并返回结果 |
| CLI `agent` 命令 | 命令行调用 |

### 3. 执行流程（5 个步骤）

```
┌─────────────────────────────────────────────────────────────┐
│ 1. agent RPC                                                │
│    - 验证参数，解析 session                                 │
│    - 持久化 session metadata                                │
│    - 立即返回 { runId, acceptedAt }                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. agentCommand                                             │
│    - 解析 model + thinking/verbose 默认值                   │
│    - 加载 skills snapshot                                   │
│    - 调用 runEmbeddedPiAgent                                │
│    - 如果 embedded loop 没发射事件，补发 lifecycle end/error │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. runEmbeddedPiAgent                                       │
│    - 通过 per-session + global queues 序列化 runs           │
│    - 解析 model + auth profile，构建 pi session             │
│    - 订阅 pi 事件，流式传输 assistant/tool deltas           │
│    - 执行超时检查 → 超时就 abort                            │
│    - 返回 payloads + usage metadata                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. subscribeEmbeddedPiSession                               │
│    - 桥接 pi-agent-core 事件到 OpenClaw agent stream        │
│    - tool events → stream: "tool"                           │
│    - assistant deltas → stream: "assistant"                 │
│    - lifecycle events → stream: "lifecycle"                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. agent.wait (可选)                                        │
│    - 使用 waitForAgentJob 等待 lifecycle end/error          │
│    - 返回 { status: ok|error|timeout, startedAt, endedAt }  │
└─────────────────────────────────────────────────────────────┘
```

### 4. 队列与并发

**设计原则:** 序列化执行，防止竞争

```
Session Lane:  每个 session key 一个队列
Global Lane:   可选的全局队列（控制并发）

Channel Queue Modes:
- collect:  收集消息
- steer:    引导对话
- followup: 后续跟踪
```

**为什么这样设计？**
- 防止 tool/session races
- 保持 session history 一致

### 5. Hook 系统（可扩展点）

#### Internal Hooks (Gateway hooks)

| Hook | 触发时机 | 用途 |
|------|----------|------|
| `agent:bootstrap` | 构建 bootstrap 文件时 | 添加/删除上下文文件 |
| Command hooks | `/new`, `/reset`, `/stop` | 命令事件处理 |

#### Plugin Hooks (Agent + Gateway lifecycle)

| Hook | 触发时机 | 用途 |
|------|----------|------|
| `before_model_resolve` | session 前（无 messages） | 覆盖 provider/model |
| `before_prompt_build` | session 加载后 | 注入动态上下文 |
| `before_agent_start` | Agent 启动前 | 遗留兼容 |
| `agent_end` | 完成后 | 检查最终消息列表 |
| `before_tool_call` / `after_tool_call` | 工具调用前后 | 拦截参数/结果 |
| `tool_result_persist` | 工具结果持久化前 | 转换结果 |
| `message_received/sending/sent` | 消息生命周期 | 消息钩子 |

### 6. 事件流（Event Streams）

```
┌──────────────────┐
│ lifecycle events │ → phase: "start" | "end" | "error"
├──────────────────┤
│ assistant deltas │ → 流式文本输出
├──────────────────┤
│ tool events      │ → tool start/update/end
└──────────────────┘
```

**Chat Channel 处理:**
- Assistant deltas → 缓冲到 `delta` 消息
- Lifecycle end/error → 发射 `final` 消息

### 7. 超时控制

| 配置 | 默认值 | 说明 |
|------|--------|------|
| `agent.wait` timeout | 30s | 仅等待时间 |
| `agents.defaults.timeoutSeconds` | 600s | Agent runtime 超时 |

**超时处理:**
- `runEmbeddedPiAgent` 中的 abort timer 强制执行
- `agent.wait` 超时不会停止 Agent，只影响等待

### 8. 提前终止场景

```
可能提前结束的情况：
1. Agent timeout (abort)
2. AbortSignal (cancel)
3. Gateway disconnect / RPC timeout
4. agent.wait timeout (仅等待，不停止 Agent)
```

---

## 🎨 设计亮点

### 1. 序列化执行模型

**问题:** 并发执行会导致状态竞争  
**解决:** per-session queue + global queue

```javascript
// 伪代码示意
const sessionQueue = new Queue(sessionKey);
const globalQueue = new Queue('global');

await sessionQueue.run(async () => {
  await globalQueue.run(async () => {
    // 执行 Agent Loop
  });
});
```

**对比其他设计:**
```
❌ 并发执行：状态不一致，tool race
✅ 序列化：状态一致，易于调试
```

### 2. 事件驱动架构

**优势:**
- 实时流式输出（用户看到打字效果）
- 工具调用进度可见
- 生命周期事件可追踪

**对比轮询:**
```
轮询：延迟高，浪费资源
事件：实时，高效
```

### 3. Hook 系统设计

**设计哲学:** "可扩展但不侵入"

```
核心循环保持不变
    ↓
Hook 点提供扩展能力
    ↓
Plugin 可以拦截/修改行为
    ↓
不影响默认流程
```

**应用场景:**
- 审计日志（`before_tool_call`）
- 动态上下文（`before_prompt_build`）
- 结果转换（`tool_result_persist`）

### 4. 超时分层设计

```
Layer 1: agent.wait (30s) - 客户端等待耐心
Layer 2: timeoutSeconds (600s) - 任务执行上限
```

**为什么分层？**
- 客户端可以提前放弃等待
- Agent 后台继续执行
- 避免无限运行

---

## ❓ 疑问/待探索

1. **`runEmbeddedPiAgent` 的具体实现？**
   - pi-agent-core 是什么？
   - 如何与 LLM 交互？

2. **队列的具体实现？**
   - 使用什么数据结构？
   - 如何保证序列化？

3. **Hook 的执行顺序？**
   - 多个 hook 注册时如何排序？
   - hook 失败如何处理？

4. **Compaction 如何触发？**
   - 文档提到 auto-compaction
   - 阈值是什么？

---

## 🔗 关联概念

- [[Gateway 架构]] - 入口点
- [[Session 管理]] - session 生命周期
- [[Streaming]] - 流式输出
- [[Compaction]] - 上下文压缩
- [[Hooks]] - 钩子系统

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Agent Loop 文档 | `docs/concepts/agent-loop.md` | 完整流程说明 |
| Pi Embedded | `dist/pi-embedded-*.js` | 编译后的运行时 |
| Helpers | `dist/pi-embedded-helpers-*.js` | 辅助函数 |

---

_下次心跳继续：Session 管理实现_

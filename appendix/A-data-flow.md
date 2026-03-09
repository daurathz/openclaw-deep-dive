# 附录 A：完整数据流转图 🗺️

> "一图胜千言" - 这里是 OpenClaw 全链路数据流转的详尽图解

---

## A.1 场景一：Slack 直接消息 → Agent 响应

### 完整流程图（宏观视角）

```mermaid
flowchart TB
    subgraph User["👤 用户侧"]
        U1[用户在 Slack 打字]
        U2["发送：'hello peter'"]
    end
    
    subgraph Slack["💬 Slack 平台"]
        S1[Slack 客户端]
        S2[Slack API 服务器]
        S3[Events API / RTM]
    end
    
    subgraph Gateway["🦞 OpenClaw Gateway"]
        G1[WebSocket 接收]
        G2[认证检查<br/>配对状态]
        G3[会话路由<br/>生成 sessionKey]
        G4[Agent 选择]
        G5[消息转发]
    end
    
    subgraph Agent["🤖 Agent 运行时"]
        A1[接收消息]
        A2[注入上下文<br/>SOUL.md + USER.md]
        A3[加载会话历史]
        A4[LLM 思考]
        A5[决定调用工具]
    end
    
    subgraph Skills["🛠️ 技能/工具"]
        SK1[read]
        SK2[write]
        SK3[exec]
        SK4[brave-search]
        SK5[slack 工具]
    end
    
    subgraph External["🌐 外部服务"]
        E1[文件系统]
        E2[网络 API]
        E3[Shell 命令]
        E4[模型 API<br/>Qwen/GPT/Claude]
    end
    
    subgraph Response["📤 响应返回"]
        R1[流式文本]
        R2[工具结果]
        R3[最终响应]
    end
    
    U1 --> U2
    U2 --> S1
    S1 --> S2
    S2 --> S3
    S3 --> G1
    G1 --> G2
    G2 --> G3
    G3 --> G4
    G4 --> G5
    G5 --> A1
    A1 --> A2
    A2 --> A3
    A3 --> A4
    A4 --> A5
    A5 --> SK1
    A5 --> SK2
    A5 --> SK3
    A5 --> SK4
    A5 --> SK5
    SK1 --> E1
    SK2 --> E1
    SK3 --> E3
    SK4 --> E2
    SK5 --> S2
    A4 --> E4
    SK1 --> R2
    SK2 --> R2
    SK3 --> R2
    SK4 --> R2
    SK5 --> R2
    R2 --> A4
    A4 --> R1
    R1 --> R3
    R3 --> G5
    G5 --> G1
    G1 --> S3
    S3 --> S2
    S2 --> S1
    S1 --> U1
    
    style User fill:#e8f4f8
    style Slack fill:#4a154b,color:#fff
    style Gateway fill:#e1f5ff
    style Agent fill:#fff3cd
    style Skills fill:#d4edda
    style External fill:#f8d7da
    style Response fill:#d1ecf1
```

---

### 详细时序图（微观视角）

```mermaid
sequenceDiagram
    autonumber
    participant U as 👤 用户
    participant SC as Slack Client
    participant SA as Slack API
    participant GW as 🦞 Gateway
    participant AG as 🤖 Agent
    participant LLM as 🧠 LLM Model
    participant SK as 🛠️ Skills
    participant FS as 📁 FileSystem
    participant NET as 🌐 Network
    
    Note over U,NET: 阶段 1: 用户发送消息
    U->>SC: 输入 "hello peter"
    SC->>SA: POST /chat.postMessage
    SA->>GW: WebSocket Event<br/>{type: "message", text: "hello peter"}
    
    Note over U,NET: 阶段 2: Gateway 处理
    GW->>GW: 验证配对状态 ✓
    GW->>GW: 生成 sessionKey<br/>"agent:peter:slack:dm:U12345"
    GW->>GW: 查找目标 Agent: peter
    
    Note over U,NET: 阶段 3: Agent 引导
    GW->>AG: 转发消息 + 会话上下文
    AG->>AG: 读取 SOUL.md
    AG->>AG: 读取 USER.md
    AG->>AG: 读取 AGENTS.md
    AG->>AG: 加载最近 10 轮对话
    
    Note over U,NET: 阶段 4: LLM 思考
    AG->>LLM: 发送完整上下文<br/>(system + user message)
    LLM->>LLM: 思考...
    LLM-->>AG: 流式响应<br/>"Hello! Let me check..."
    
    Note over U,NET: 阶段 5: 工具调用（如果需要）
    AG->>SK: 调用 read 工具<br/>path: "~/.openclaw/workspace-peter/USER.md"
    SK->>FS: 读取文件
    FS-->>SK: 返回文件内容
    SK-->>AG: 工具结果
    
    Note over U,NET: 阶段 6: 继续思考
    AG->>LLM: 发送工具结果
    LLM-->>AG: 最终响应<br/>"Hi! I see you're learning OpenClaw..."
    
    Note over U,NET: 阶段 7: 响应返回
    AG-->>GW: 流式返回响应
    GW->>GW: 更新会话状态
    GW->>GW: 保存转录到 jsonl
    GW-->>SA: Slack API 发送消息
    SA-->>SC: 推送消息到客户端
    SC-->>U: 显示回复
    
    Note over GW: 会话状态更新<br/>sessions.json + *.jsonl
```

---

## A.2 场景二：群聊 @提及 → Agent 响应

```mermaid
flowchart TD
    subgraph Group["👥 群聊环境"]
        G1[用户在群里发消息]
        G2["消息：'@peter 今天天气如何？'"]
        G3[群成员可见]
    end
    
    subgraph Filter["🔍 Gateway 过滤"]
        F1{提到 bot？}
        F2{群聊允许列表？}
        F3{requireMention=true？}
    end
    
    subgraph Process["处理流程"]
        P1[生成 sessionKey<br/>agent:peter:slack:group:C12345]
        P2[加载群聊特定上下文]
        P3[调用 weather skill]
        P4[获取天气数据]
        P5[生成响应]
    end
    
    G1 --> G2
    G2 --> F1
    F1 -->|否 | Ignore[❌ 忽略消息]
    F1 -->|是 | F2
    F2 -->|不在允许列表 | Block[❌  blocked]
    F2 -->|在允许列表 | F3
    F3 -->|需要 mention 但未@| Ignore
    F3 -->|通过 | P1
    P1 --> P2
    P2 --> P3
    P3 --> P4
    P4 --> P5
    P5 --> Reply[✅ 发送回复到群聊]
    
    style Group fill:#e8f4f8
    style Filter fill:#fff3cd
    style Process fill:#d4edda
    style Ignore fill:#f8d7da
    style Block fill:#f8d7da
    style Reply fill:#d1ecf1
```

---

## A.3 场景三：Cron 定时任务 → 数据采集

```mermaid
sequenceDiagram
    participant Cron as ⏰ Cron Scheduler
    participant GW as 🦞 Gateway
    participant AG as 🤖 Agent
    participant SK as 🛠️ Skills
    participant Web as 🌐 Websites
    participant Git as 💾 GitHub
    
    Note over Cron,Git: 每天 00:00-07:00 每 10 分钟执行
    
    Cron->>GW: 触发 cron:daisy-data-collection
    GW->>GW: 创建独立会话<br/>sessionKey: "cron:daisy-data-collection"
    GW->>AG: 调用 Agent
    
    loop 每 10 分钟
        AG->>SK: 调用 brave-search
        SK->>Web: 搜索汽车后市场新闻
        Web-->>SK: 返回搜索结果
        SK-->>AG: 整理数据
        
        AG->>SK: 调用 read/write
        SK->>Git: 保存到本地仓库
        Git-->>SK: 确认保存
        
        AG->>AG: 记录数据来源<br/>(URL + 时间戳)
    end
    
    AG-->>GW: 任务完成
    GW->>GW: 更新会话状态
    GW-->>Cron: 等待下次触发
    
    Note over AG: 关键：记录信息来源<br/>建立来源认知
```

---

## A.4 场景四：报告生成 → GitHub 提交

```mermaid
flowchart LR
    subgraph Trigger["触发条件"]
        T1[每天 09:00]
        T2[或手动触发]
    end
    
    subgraph Collect["数据收集"]
        C1[读取昨夜采集数据]
        C2[整合历史报告]
        C3[分析趋势]
    end
    
    subgraph Generate["报告生成"]
        G1[调用 LLM 生成观点]
        G2[添加数据引用]
        G3[格式化 Markdown]
    end
    
    subgraph Publish["发布流程"]
        P1[git add reports/]
        P2[git commit]
        P3[git push]
        P4[可选：发送 Slack]
    end
    
    T1 --> C1
    T2 --> C1
    C1 --> C2
    C2 --> C3
    C3 --> G1
    G1 --> G2
    G2 --> G3
    G3 --> P1
    P1 --> P2
    P2 --> P3
    P3 --> P4
    
    style Trigger fill:#e1f5ff
    style Collect fill:#fff3cd
    style Generate fill:#d4edda
    style Publish fill:#d1ecf1
```

---

## A.5 参与者清单

### 内部参与者

| 参与者 | 职责 | 位置 |
|-------|------|------|
| **Gateway** | 消息路由、认证、会话管理 | `~/.nvm/versions/node/v24.14.0/lib/node_modules/openclaw/` |
| **Agent** | LLM 调用、上下文管理、工具协调 | 运行时进程 |
| **Skills** | 具体功能实现（read/write/exec/search） | `~/.openclaw/skills/` 或 `<workspace>/skills/` |
| **Sessions** | 会话状态存储 | `~/.openclaw/agents/<id>/sessions/*.jsonl` |
| **Config** | 配置管理 | `~/.openclaw/openclaw.json` |

### 外部参与者

| 参与者 | 作用 | 通信方式 |
|-------|------|---------|
| **Slack API** | 消息收发 | WebSocket/HTTP |
| **WhatsApp Web** | 消息收发 | Baileys 库 |
| **Telegram Bot API** | 消息收发 | HTTP long polling/webhook |
| **LLM Provider** | AI 模型推理 | HTTP API (Qwen/GPT/Claude) |
| **Brave Search** | 网络搜索 | HTTP API |
| **GitHub** | 代码/报告存储 | Git protocol |
| **File System** | 本地文件读写 | 系统调用 |
| **Shell** | 命令执行 | 子进程 |

---

## A.6 数据格式示例

### 1. Slack 消息（入站）

```json
{
  "type": "message",
  "channel": "D0123456789",
  "user": "U05AN0GGM3M",
  "text": "hello peter",
  "ts": "1773068843.931469",
  "client_msg_id": "abc123"
}
```

### 2. Gateway 内部消息

```json
{
  "type": "req",
  "id": "req-123",
  "method": "agent",
  "params": {
    "agentId": "peter",
    "message": "hello peter",
    "sessionId": "agent:peter:slack:dm:U05AN0GGM3M",
    "context": {
      "channel": "slack",
      "chatType": "direct",
      "sender": "daur"
    }
  }
}
```

### 3. Agent 上下文（发送给 LLM）

```json
{
  "system": "# SOUL.md\n你是 Peter，OpenClaw 的创作者...\n\n# USER.md\n用户：老鄂，程序员...",
  "messages": [
    {"role": "user", "content": "hello peter", "timestamp": "2026-03-09T23:07:25+08:00"}
  ],
  "tools": ["read", "write", "exec", "brave-search"],
  "workspace": "/home/openclaw/.openclaw/workspace-peter"
}
```

### 4. 工具调用

```json
{
  "name": "read",
  "arguments": {
    "path": "/home/openclaw/.openclaw/workspace-peter/USER.md"
  }
}
```

### 5. 工具结果

```json
{
  "name": "read",
  "result": "# USER.md - About Your Human\n- Name: 老鄂\n- Timezone: Asia/Shanghai...",
  "success": true
}
```

### 6. 会话转录（jsonl 格式）

```jsonl
{"role":"user","content":"hello peter","timestamp":"2026-03-09T23:07:25+08:00"}
{"role":"assistant","content":"嗨，老鄂！🦞 有什么可以帮你的？","timestamp":"2026-03-09T23:07:28+08:00"}
{"role":"tool","name":"read","result":"...","timestamp":"2026-03-09T23:07:29+08:00"}
```

---

## A.7 性能指标

### 典型延迟分解

```
用户发送 → Gateway 接收：     50-200ms  (网络延迟)
Gateway 认证 + 路由：         5-20ms    (内存操作)
Agent 上下文注入：            10-50ms   (文件读取)
LLM 思考 + 生成：             500-5000ms (模型推理)
工具调用（如 read）：         10-100ms  (磁盘 I/O)
工具调用（如 search）：       200-2000ms (网络请求)
响应返回 → 用户收到：         50-200ms  (网络延迟)

总计（简单消息）：            600-1000ms
总计（带工具调用）：          1000-5000ms
```

### 会话存储大小

```
典型会话（100 轮对话）：       50-200KB
每日新增（活跃用户）：         20-50KB
月度会话（30 天）：            1-5MB
```

---

## A.8 故障点诊断

### 常见问题定位

```mermaid
flowchart TD
    Prob[问题现象] --> Check1{消息没反应？}
    Check1 -->|是 | Step1[检查 Gateway 日志]
    Step1 --> Step2{看到消息接收？}
    Step2 -->|否 | Fix1[检查通道配置/配对]
    Step2 -->|是 | Step3{看到路由到 Agent？}
    Step3 -->|否 | Fix2[检查 sessionKey 生成]
    Step3 -->|是 | Step4{看到 Agent 响应？}
    Step4 -->|否 | Fix3[检查 Agent 状态/模型]
    Step4 -->|是 | Fix4[检查响应发送路径]
    
    Check1 -->|否 | Check2{响应太慢？}
    Check2 -->|是 | Step5[查看 LLM 调用时间]
    Step5 --> Step6{工具调用耗时？}
    Step6 -->|是 | Fix5[优化网络/缓存]
    Step6 -->|否 | Fix6[考虑换更快的模型]
    
    style Prob fill:#f8d7da
    style Fix1 fill:#d4edda
    style Fix2 fill:#d4edda
    style Fix3 fill:#d4edda
    style Fix4 fill:#d4edda
    style Fix5 fill:#d4edda
    style Fix6 fill:#d4edda
```

---

_这份附录应该让你对 OpenClaw 的数据流转有全景认知。有任何不清楚的地方，Slack 问我！🦞_

# Tool Permissions 工具权限配置指南

**创建时间:** 2026-03-12  
**适用场景:** 让 Agent 在会话中可以执行 shell 命令和操作文件  
**难度:** ⭐⭐ 初级

---

## 🎯 配置目标

让 Agent 可以在会话中：
- ✅ 执行 shell 命令（git, npm, node, python 等）
- ✅ 读写和编辑文件
- ✅ 运行后台进程
- ✅ 安全地使用白名单机制

---

## 📁 涉及文件

| 文件 | 用途 | 位置 |
|------|------|------|
| `openclaw.json` | 主配置文件 | `~/.openclaw/openclaw.json` |
| `exec-approvals.json` | Exec 审批配置 | `~/.openclaw/exec-approvals.json` |

---

## 🔧 配置步骤

### 步骤 1：配置 tools.profile

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  "tools": {
    "profile": "coding"  // ← 改成 "coding"
  }
}
```

**profile 选项:**

| 值 | 说明 | 适用场景 |
|------|------|----------|
| `"coding"` | 开发工具集（推荐） | 文件操作、执行命令、git 等 |
| `"full"` | 所有工具 | 需要浏览器、消息等完整能力 |
| `"minimal"` | 基础工具 | 只读/简单写入场景 |

**`"coding"` 包含的工具:**
- `read` - 读取文件
- `edit` - 编辑文件
- `write` - 写入文件
- `exec` - 执行 shell 命令
- `process` - 运行后台进程
- 其他开发相关工具

---

### 步骤 2：创建 Exec 审批配置

创建 `~/.openclaw/exec-approvals.json`：

```json
{
  "version": 1,
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": false,
      "allowlist": []
    },
    "peter": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": false,
      "allowlist": []
    },
    "daisy": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": false,
      "allowlist": []
    },
    "alfred": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": false,
      "allowlist": []
    }
  }
}
```

---

## 🔐 安全配置说明

### security 选项

| 值 | 说明 | 推荐度 |
|------|------|--------|
| `"allowlist"` | 只允许白名单命令 | ⭐⭐⭐ 推荐 |
| `"deny"` | 禁止所有 exec | ⭐⭐ 高安全场景 |
| `"full"` | 允许所有命令 | ⭐ 仅限可信环境 |

### ask 选项

| 值 | 说明 | 场景 |
|------|------|------|
| `"on-miss"` | 白名单没有时询问 | 推荐 |
| `"off"` | 从不询问 | 白名单完整时 |
| `"always"` | 每次都询问 | 高安全场景 |

### askFallback 选项

当无法询问用户时的 fallback：

| 值 | 说明 |
|------|------|
| `"deny"` | 拒绝执行 |
| `"allowlist"` | 只在白名单内允许 |
| `"full"` | 允许执行 |

---

## 📋 推荐命令白名单

在 `agents.<agentId>.allowlist` 中添加：

```json
{
  "allowlist": [
    // 版本控制
    "git",
    
    // 包管理
    "npm", "yarn", "pnpm",
    
    // 运行环境
    "node", "python", "python3", "cargo", "go",
    
    // 文件操作
    "ls", "cat", "grep", "find",
    "mkdir", "cp", "mv", "rm",
    "pwd", "head", "tail", "wc",
    
    // 网络工具
    "curl", "wget",
    
    // 其他工具
    "jq", "sort", "uniq", "cut", "tr"
  ]
}
```

---

## 🚀 重启 Gateway

配置完成后重启 Gateway：

```bash
# 方法 1：使用 openclaw 命令
openclaw gateway restart

# 方法 2：使用 systemctl
systemctl --user restart openclaw-gateway

# 方法 3：使用 node 直接运行
node /path/to/openclaw/openclaw.mjs gateway restart
```

---

## ✅ 验证配置

### 1. 检查 Gateway 状态

```bash
openclaw gateway status
```

**期望输出:**
```
Runtime: running (pid XXXXX, state active, sub running)
RPC probe: ok
Listening: 127.0.0.1:33297
```

### 2. 测试执行命令

在会话中让 Agent 执行：

```
帮我执行 `ls -la` 看看当前目录
```

或

```
用 `node --version` 检查 Node 版本
```

### 3. 查看审批配置

```bash
cat ~/.openclaw/exec-approvals.json
```

---

## 🔍 故障诊断

### 问题 1：命令无法执行

**症状:** Agent 回复无法执行命令

**检查步骤:**

1. 检查 `tools.profile` 配置
   ```bash
   cat ~/.openclaw/openclaw.json | grep profile
   ```
   应该是 `"profile": "coding"`

2. 检查 `exec-approvals.json` 是否存在
   ```bash
   ls -la ~/.openclaw/exec-approvals.json
   ```

3. 检查 security 模式
   ```bash
   cat ~/.openclaw/exec-approvals.json | grep security
   ```
   应该是 `"security": "allowlist"`

**解决方法:**
- 确保配置文件正确
- 重启 Gateway

---

### 问题 2：每次都询问

**症状:** 每次执行命令都弹出审批对话框

**原因:** `ask` 设置为 `"always"`

**解决方法:**
```json
{
  "defaults": {
    "ask": "on-miss"  // 改为 on-miss
  }
}
```

---

### 问题 3：白名单命令无法执行

**症状:** 已经在白名单中，但仍无法执行

**检查:**
1. 命令拼写是否正确
2. 命令路径是否在系统中存在
3. Gateway 是否已重启

**解决方法:**
```bash
# 检查命令是否存在
which git
which node
which npm

# 重启 Gateway
openclaw gateway restart
```

---

## 🎯 最佳实践

### 1. 最小权限原则

```json
{
  "security": "allowlist",  // ✅ 推荐
  "ask": "on-miss",         // ✅ 只在需要时询问
  "allowlist": [            // ✅ 明确列出需要的命令
    "git", "npm", "node"
  ]
}
```

### 2. 分 Agent 配置

为不同 Agent 配置不同权限：

```json
{
  "agents": {
    "main": {
      "allowlist": ["git", "npm", "node"]  // 开发工具
    },
    "daisy": {
      "allowlist": ["curl", "wget", "jq"]  // 数据采集工具
    }
  }
}
```

### 3. 定期清理白名单

定期检查并删除不再使用的命令：

```bash
# 查看白名单
cat ~/.openclaw/exec-approvals.json | jq '.agents.main.allowlist'

# 编辑白名单
code ~/.openclaw/exec-approvals.json
```

### 4. 使用 autoAllowSkills

如果使用了技能，可以启用自动允许：

```json
{
  "defaults": {
    "autoAllowSkills": true  // 自动允许技能使用的命令
  }
}
```

**⚠️ 注意:** 这会降低安全性，仅在可信环境使用。

---

## 📚 进阶配置

### Safe Bins（快速通道）

对于安全的 stdin-only 命令，可以配置为 safe bins：

```json5
{
  "tools": {
    "exec": {
      "safeBins": ["jq", "cut", "uniq", "head", "tail", "tr", "wc"]
    }
  }
}
```

**Safe Bins 特点:**
- 不需要白名单
- 只能处理 stdin 数据
- 不能读取文件
- 不能执行子命令

### 自定义 Safe Bin Profile

```json5
{
  "tools": {
    "exec": {
      "safeBins": ["myfilter"],
      "safeBinProfiles": {
        "myfilter": {
          "minPositional": 0,
          "maxPositional": 0,
          "allowedValueFlags": ["-n", "--limit"],
          "deniedFlags": ["-f", "--file"]
        }
      }
    }
  }
}
```

---

## 🔗 相关文档

- [Exec Approvals 官方文档](https://docs.openclaw.ai/tools/exec-approvals)
- [Exec Tool 官方文档](https://docs.openclaw.ai/tools/exec)
- [Gateway 配置](https://docs.openclaw.ai/gateway/configuration)
- [Agent Workspace](https://docs.openclaw.ai/concepts/agent-workspace)

---

## 📝 配置检查清单

配置完成后检查：

- [ ] `tools.profile` 设置为 `"coding"`
- [ ] `exec-approvals.json` 已创建
- [ ] `security` 设置为 `"allowlist"`
- [ ] `ask` 设置为 `"on-miss"`
- [ ] `allowlist` 包含需要的命令
- [ ] Gateway 已重启
- [ ] 测试命令可以正常执行

---

## 🎉 完成！

现在你的 Agent 可以安全地在会话中执行命令了！

**示例用法:**
```
# 文件操作
帮我列出当前目录的所有文件

# Git 操作
查看 git 状态

# 运行代码
用 node 运行这个脚本

# 系统信息
检查 Node 和 npm 版本
```

---

_最后更新：2026-03-12 | 🦞 OpenClaw 配置经验总结_

# Workspace Hooks 系统

**阅读时间:** 2026-03-11 16:00  
**源码路径:** `dist/hooks-cli-BcHZWu40.js`, `dist/workspace-CRnWKAzq.js`, `dist/plugin-sdk/agents/bootstrap-hooks.d.ts`  
**核心职责:** 工作空间生命周期钩子与事件拦截

---

## 核心概念

Hooks 系统允许在 OpenClaw 生命周期的关键节点插入自定义逻辑：

- **Bootstrap Hooks**: Agent 启动时加载工作空间配置
- **Lifecycle Hooks**: 会话/任务的生命周期事件
- **Webhook Handlers**: 外部事件触发

---

## Hook 目录结构

```
~/.config/openclaw/hooks/
├── my-hook/
│   ├── HOOK.md              # Hook 元数据（必填）
│   ├── handler.ts           # 处理器（TypeScript）
│   ├── handler.js           # 处理器（JavaScript）
│   ├── index.ts             # 或 index.js
│   └── package.json         # 可选（npm 包形式）
```

### HOOK.md 格式

```markdown
---
name: my-custom-hook
description: 自定义钩子功能
version: 1.0.0
---

# Hook 说明

...
```

---

## Hook 安装流程

### 1. 验证 Hook ID

```typescript
function validateHookId(hookId): string | null {
  if (!hookId) return "invalid hook name: missing";
  if (hookId === "." || hookId === "..") 
    return "invalid hook name: reserved path segment";
  if (hookId.includes("/") || hookId.includes("\\")) 
    return "invalid hook name: path separators not allowed";
  return null;
}
```

### 2. 解析安装目录

```typescript
function resolveHookInstallDir(hookId, hooksDir): string {
  const hooksBase = hooksDir 
    ? resolveUserPath(hooksDir) 
    : path.join(CONFIG_DIR, "hooks");
  
  const targetDirResult = resolveSafeInstallDir({
    baseDir: hooksBase,
    id: hookId,
    invalidNameMessage: "invalid hook name: path traversal detected"
  });
  
  return targetDirResult.path;
}
```

### 3. 验证 Hook 包

```typescript
async function validateHookDir(hookDir): Promise<void> {
  // 检查 HOOK.md
  if (!await fileExists(path.join(hookDir, "HOOK.md")))
    throw new Error(`HOOK.md missing in ${hookDir}`);
  
  // 检查处理器文件
  const hasHandler = await Promise.all([
    "handler.ts", "handler.js", "index.ts", "index.js"
  ].map(f => fileExists(path.join(hookDir, f))));
  
  if (!hasHandler.some(Boolean))
    throw new Error(`handler.ts/handler.js/index.ts/index.js missing`);
}
```

---

## Hook 包 (Hook Pack)

支持通过 npm 包形式分发多个 Hook：

### package.json 配置

```json
{
  "name": "@myorg/openclaw-hooks",
  "version": "1.0.0",
  "openclaw": {
    "hooks": [
      "./hooks/bootstrap",
      "./hooks/pre-request",
      "./hooks/post-response"
    ]
  }
}
```

### 安装流程

```typescript
async function installHookPackageFromDir(params): Promise<InstallResult> {
  // 1. 读取 package.json
  const manifest = await readJsonFile(manifestPath);
  
  // 2. 验证 openclaw.hooks 配置
  const hookEntries = await ensureOpenClawHooks(manifest);
  
  // 3. 解析 Hook 包 ID
  const hookPackId = pkgName 
    ? unscopedPackageName(pkgName) 
    : path.basename(params.packageDir);
  
  // 4. 验证每个 Hook 目录
  for (const entry of hookEntries) {
    const hookDir = path.resolve(params.packageDir, entry);
    await validateHookDir(hookDir);
    const hookName = await resolveHookNameFromDir(hookDir);
    resolvedHooks.push(hookName);
  }
  
  // 5. 安装到目标目录
  return { ok: true, hookPackId, resolvedHooks };
}
```

---

## Bootstrap Hooks

在 Agent 启动时加载工作空间配置：

### 类型定义

```typescript
// dist/plugin-sdk/agents/bootstrap-hooks.d.ts
export declare function applyBootstrapHookOverrides(params: {
  files: WorkspaceBootstrapFile[];
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  sessionId?: string;
  agentId?: string;
}): Promise<WorkspaceBootstrapFile[]>;
```

### 工作流程

```
Agent 启动
  ↓
加载工作空间文件 (SOUL.md, USER.md, AGENTS.md, etc.)
  ↓
应用 Bootstrap Hook Overrides
  ↓
最终配置文件集合
```

### 使用场景

- 动态注入配置
- 环境变量覆盖
- 条件性文件加载
- 多工作空间同步

---

## Hook 状态管理

### 状态报告结构

```typescript
interface HookStatus {
  hookId: string;
  installed: boolean;
  version?: string;
  lastLoadedAt?: number;
  loadError?: string;
  manifest?: HookManifest;
}
```

### CLI 命令

```bash
# 查看 Hook 状态
openclaw hooks status

# 列出已安装 Hook
openclaw hooks list

# 安装 Hook
openclaw hooks install ./my-hook

# 卸载 Hook
openclaw hooks uninstall my-hook

# 重新加载 Hook
openclaw hooks reload my-hook
```

---

## 安全机制

### 1. 路径遍历防护

```typescript
if (!isPathInside(params.packageDir, hookDir))
  return { ok: false, error: "hook entry escapes package directory" };

if (!isPathInsideWithRealpath(params.packageDir, hookDir, { requireRealpath: true }))
  return { ok: false, error: "hook entry resolves outside package directory" };
```

### 2. Hook ID 验证

- 不允许 `/` 或 `\` 字符
- 不允许 `.` 或 `..`  reserved names
- 路径规范化检查

### 3. 完整性校验

```typescript
if (params.expectedHookPackId && params.expectedHookPackId !== hookPackId)
  return { ok: false, error: "hook pack id mismatch" };
```

---

## 设计亮点

### 1. 插件化架构

Hook 作为独立插件安装，不影响核心系统。

### 2. 多种安装方式

- **本地目录**: `openclaw hooks install ./my-hook`
- **npm 包**: `openclaw hooks install @myorg/hooks`
- **Git 仓库**: `openclaw hooks install github:user/repo`

### 3. 热重载支持

```bash
openclaw hooks reload my-hook
```

### 4. 依赖管理

Hook 包可通过 `package.json` 声明依赖：

```json
{
  "name": "@myorg/hooks",
  "dependencies": {
    "openclaw-plugin-sdk": "^1.0.0"
  }
}
```

### 5. 类型安全

提供 TypeScript 类型定义：

```typescript
import type { HookHandler } from "@openclaw/plugin-sdk";

export const handler: HookHandler = async (event, context) => {
  // 类型安全的 Hook 处理器
};
```

---

## 使用示例

### 1. 安装本地 Hook

```bash
cd ~/.config/openclaw/hooks
git clone https://github.com/myuser/my-hook.git
openclaw hooks install ./my-hook
```

### 2. 安装 npm 包 Hook

```bash
openclaw hooks install @openclaw/community-hooks
```

### 3. 查看 Hook 状态

```bash
openclaw hooks status --json
```

输出：
```json
{
  "hooks": [
    {
      "hookId": "community-hooks",
      "installed": true,
      "version": "1.2.0",
      "lastLoadedAt": 1710144000000,
      "hooks": ["bootstrap", "pre-request"]
    }
  ]
}
```

---

## 关联概念

- [[05-cron/cron-scheduler]] - Cron 调度（定时触发 Hook）
- [[01-agents/runtime]] - Agent 运行时（Bootstrap Hook 执行点）
- [[03-skills/system]] - 技能系统（类似的插件架构）

---

**备注:** Hook 系统采用与 Skills 类似的插件化架构，但专注于生命周期事件而非工具扩展。

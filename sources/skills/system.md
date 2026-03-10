# 技能系统 — 加载、注册、调用机制

**阅读时间:** 2026-03-10  
**源码文件:** 
- `dist/skills-D9zp-Tdj.js` (技能加载、快照构建)
- `dist/skills-install-NiZ_P1cj.js` (技能安装)
- `dist/agent-scope-CZF6h_5g.js` (工作空间技能过滤)

---

## 1. 技能系统流程概览

```
┌─────────────────────────────────────────────────────────────────┐
│  技能目录结构                                                    │
│  <workspace>/skills/                                            │
│  ├── brave-search/          # 搜索技能                          │
│  │   ├── SKILL.md           # 技能定义 (Frontmatter + 描述)      │
│  │   ├── skill.sh           # 技能脚本                          │
│  │   └── references/        # 参考文档                          │
│  ├── github/                                                    │
│  │   ├── SKILL.md                                               │
│  │   └── skill.sh                                               │
│  └── ...                                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  loadSkillEntries() — 扫描技能目录                               │
│  - 遍历 skills/ 子目录                                           │
│  - 解析 SKILL.md Frontmatter                                    │
│  - 验证技能结构 (name/description/invocation)                   │
│  - 返回 SkillEntry[]                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  filterSkillEntries() — 过滤技能                                 │
│  - 根据 skillFilter 过滤 (include/exclude)                       │
│  - 检查环境依赖 (requiredEnv)                                  │
│  - 检查远程技能资格 (eligibility.remote)                        │
│  - 返回符合条件的技能列表                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  buildWorkspaceSkillSnapshot() — 构建技能快照                    │
│  - 生成技能提示文本 (formatSkillsForPrompt)                     │
│  - 记录技能元数据 (name/primaryEnv/requiredEnv)                 │
│  - 应用提示限制 (applySkillsPromptLimits)                       │
│  - 返回 Snapshot { prompt, skills, version }                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  技能调用                                                        │
│  ├─ 命令调用：/<skill-name> [args]                              │
│  ├─ 工具调用：skill 工具 → 执行技能脚本                          │
│  └─ 内部调用：resolveSkillsPromptForRun() 注入系统提示           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 技能目录结构

### 2.1 标准技能结构

```
skills/
└── <skill-name>/
    ├── SKILL.md           # 必需：技能定义
    ├── skill.sh           # 必需：Bash 脚本 (或 skill.js/skill.py)
    ├── references/        # 可选：参考文档
    │   └── README.md
    └── scripts/           # 可选：辅助脚本
        └── helper.sh
```

### 2.2 SKILL.md 格式

```markdown
---
name: brave-search
description: Search the web using Brave Search API
version: 1.0.0
author: OpenClaw Team

# 调用配置
invocation:
  type: command      # command | tool | internal
  userInvocable: true  # 用户是否可直接调用
  disableModelInvocation: false  # 是否对模型隐藏

# 环境依赖
requires:
  env:
    - BRAVE_API_KEY

# 安装配置
install:
  - id: npm-install
    kind: npm
    package: brave-search-cli
  
  - id: download
    kind: download
    url: https://example.com/skill.tar.gz

# 命令分发 (可选)
command-dispatch: tool
command-tool: brave_search
command-arg-mode: raw
---

# 技能描述

详细说明技能的功能、使用方法、注意事项等。

## Usage

```bash
./skill.sh "query"
```

## Examples

...
```

---

## 3. 核心函数详解

### 3.1 `loadSkillEntries()` — 加载技能条目

```javascript
function loadSkillEntries(workspaceDir, opts) {
  const skillsDir = path.join(workspaceDir, "skills");
  const entries = [];
  
  // 1. 扫描 skills 目录
  const skillDirs = fs.readdirSync(skillsDir, { withFileTypes: true })
    .filter(dirent => dirent.isDirectory())
    .map(dirent => path.join(skillsDir, dirent.name));
  
  for (const skillDir of skillDirs) {
    // 2. 解析 SKILL.md
    const skillMdPath = path.join(skillDir, "SKILL.md");
    if (!fs.existsSync(skillMdPath)) continue;
    
    const frontmatter = parseFrontmatter(skillMdPath);
    const skill = {
      name: frontmatter.name?.trim(),
      description: frontmatter.description?.trim(),
      version: frontmatter.version?.trim(),
      author: frontmatter.author?.trim(),
      baseDir: skillDir
    };
    
    // 3. 验证必需字段
    if (!skill.name) {
      skillsLogger.warn(`Skill missing name: ${skillDir}`);
      continue;
    }
    
    // 4. 解析调用配置
    const invocation = frontmatter.invocation ?? {
      type: "command",
      userInvocable: true,
      disableModelInvocation: false
    };
    
    // 5. 解析环境依赖
    const requires = frontmatter.requires ?? {};
    
    // 6. 解析安装配置
    const install = Array.isArray(frontmatter.install) ? frontmatter.install : [];
    
    // 7. 构建技能条目
    entries.push({
      skill,
      frontmatter,
      invocation,
      metadata: {
        primaryEnv: requires.env?.[0],
        requires: { env: requires.env ?? [] }
      },
      install
    });
  }
  
  return entries;
}
```

**返回结构:**
```javascript
{
  skill: { name, description, version, author, baseDir },
  frontmatter: { /* 完整 Frontmatter */ },
  invocation: { type, userInvocable, disableModelInvocation },
  metadata: { primaryEnv, requires: { env: [] } },
  install: [{ id, kind, ... }]
}
```

---

### 3.2 `filterSkillEntries()` — 过滤技能

```javascript
function filterSkillEntries(entries, config, skillFilter, eligibility) {
  return entries.filter((entry) => {
    // 1. 技能过滤器 (include/exclude)
    if (skillFilter) {
      if (skillFilter.include?.length > 0) {
        if (!skillFilter.include.includes(entry.skill.name)) return false;
      }
      if (skillFilter.exclude?.length > 0) {
        if (skillFilter.exclude.includes(entry.skill.name)) return false;
      }
    }
    
    // 2. 环境依赖检查
    const requiredEnv = entry.metadata?.requires?.env ?? [];
    for (const envVar of requiredEnv) {
      if (!process.env[envVar]) {
        skillsLogger.debug(`Skill ${entry.skill.name} requires ${envVar}`);
        // 根据配置决定是否跳过
        if (config?.skills?.skipMissingEnv !== true) return false;
      }
    }
    
    // 3. 远程技能资格检查
    if (eligibility?.remote?.enabled === false) {
      if (entry.skill.remote === true) return false;
    }
    
    return true;
  });
}
```

---

### 3.3 `buildWorkspaceSkillSnapshot()` — 构建技能快照

```javascript
function buildWorkspaceSkillSnapshot(workspaceDir, opts) {
  const { eligible, prompt, resolvedSkills } = 
    resolveWorkspaceSkillPromptState(workspaceDir, opts);
  
  const skillFilter = normalizeSkillFilter(opts?.skillFilter);
  
  return {
    prompt,  // 格式化后的技能提示文本
    skills: eligible.map((entry) => ({
      name: entry.skill.name,
      primaryEnv: entry.metadata?.primaryEnv,
      requiredEnv: entry.metadata?.requires?.env?.slice()
    })),
    ...skillFilter === void 0 ? {} : { skillFilter },
    resolvedSkills,  // 完整技能对象数组
    version: opts?.snapshotVersion
  };
}

function resolveWorkspaceSkillPromptState(workspaceDir, opts) {
  const eligible = filterSkillEntries(
    opts?.entries ?? loadSkillEntries(workspaceDir, opts),
    opts?.config,
    opts?.skillFilter,
    opts?.eligibility
  );
  
  // 过滤可调用技能 (用于提示)
  const promptEntries = eligible.filter(
    (entry) => entry.invocation?.disableModelInvocation !== true
  );
  
  const resolvedSkills = promptEntries.map((entry) => entry.skill);
  
  // 应用提示限制 (避免超出上下文)
  const { skillsForPrompt, truncated } = applySkillsPromptLimits({
    skills: resolvedSkills,
    config: opts?.config
  });
  
  return {
    eligible,
    prompt: [
      opts?.eligibility?.remote?.note?.trim(),  // 远程技能说明
      truncated ? `⚠️ Skills truncated: included ${skillsForPrompt.length} of ${resolvedSkills.length}` : "",
      formatSkillsForPrompt(compactSkillPaths(skillsForPrompt))
    ].filter(Boolean).join("\n"),
    resolvedSkills
  };
}
```

**快照结构:**
```javascript
{
  prompt: "Available skills:\n- brave-search: Search the web\n- github: GitHub operations",
  skills: [
    { name: "brave-search", primaryEnv: "BRAVE_API_KEY", requiredEnv: ["BRAVE_API_KEY"] },
    { name: "github", primaryEnv: "GITHUB_TOKEN", requiredEnv: ["GITHUB_TOKEN"] }
  ],
  resolvedSkills: [{ name, description, version, ... }, ...],
  version: 1
}
```

---

### 3.4 `formatSkillsForPrompt()` — 格式化技能提示

```javascript
function formatSkillsForPrompt(skills) {
  // 生成模型可读的技能列表
  const lines = skills.map((skill) => {
    const name = skill.name;
    const description = skill.description?.trim() || "No description";
    return `- ${name}: ${description}`;
  });
  
  return [
    "# Available Skills",
    "",
    "You have access to the following skills:",
    "",
    ...lines,
    "",
    "Usage:",
    "- Command: /<skill-name> [args]",
    "- Tool: Call the skill tool with appropriate parameters"
  ].join("\n");
}
```

**示例输出:**
```
# Available Skills

You have access to the following skills:

- brave-search: Search the web using Brave Search API
- github: GitHub operations (issues, PRs, repos)
- weather: Get current weather and forecasts
- slack: Send messages to Slack channels

Usage:
- Command: /<skill-name> [args]
- Tool: Call the skill tool with appropriate parameters
```

---

### 3.5 `applySkillsPromptLimits()` — 应用提示限制

```javascript
function applySkillsPromptLimits(params) {
  const { skills, config } = params;
  const maxSkills = config?.skills?.prompt?.maxSkills ?? 50;
  const maxChars = config?.skills?.prompt?.maxChars ?? 5000;
  
  if (skills.length <= maxSkills) {
    const prompt = formatSkillsForPrompt(skills);
    if (prompt.length <= maxChars) {
      return { skillsForPrompt: skills, truncated: false };
    }
  }
  
  // 截断技能列表
  const skillsForPrompt = [];
  let currentChars = 0;
  
  for (const skill of skills) {
    const skillLine = `- ${skill.name}: ${skill.description || ""}\n`;
    if (currentChars + skillLine.length > maxChars) break;
    if (skillsForPrompt.length >= maxSkills) break;
    
    skillsForPrompt.push(skill);
    currentChars += skillLine.length;
  }
  
  return { skillsForPrompt, truncated: true };
}
```

---

### 3.6 `installSkill()` — 安装技能

```javascript
async function installSkill(params) {
  const timeoutMs = Math.min(Math.max(params.timeoutMs ?? 300000, 1000), 900000);
  
  // 1. 查找技能
  const entry = loadWorkspaceSkillEntries(resolveUserPath(params.workspaceDir))
    .find((item) => item.skill.name === params.skillName);
  
  if (!entry) {
    return { ok: false, message: `Skill not found: ${params.skillName}` };
  }
  
  // 2. 查找安装配置
  const spec = findInstallSpec(entry, params.installId);
  if (!spec) {
    return { ok: false, message: `Installer not found: ${params.installId}` };
  }
  
  // 3. 收集安装前警告
  const warnings = await collectSkillInstallScanWarnings(entry);
  
  // 4. 执行安装
  if (spec.kind === "download") {
    return withWarnings(await installDownloadSpec({ entry, spec, timeoutMs }), warnings);
  }
  
  // 5. 构建安装命令
  const command = buildInstallCommand(spec, resolveSkillsInstallPreferences(params.config));
  if (command.error) {
    return withWarnings({ ok: false, message: command.error }, warnings);
  }
  
  // 6. 确保依赖安装 (brew/uv/go)
  if (spec.kind === "brew") {
    const brewExe = hasBinary("brew") ? "brew" : resolveBrewExecutable();
    if (!brewExe) return withWarnings(resolveBrewMissingFailure(spec), warnings);
  }
  
  if (spec.kind === "uv") {
    const uvInstallFailure = await ensureUvInstalled({ spec, brewExe, timeoutMs });
    if (uvInstallFailure) return withWarnings(uvInstallFailure, warnings);
  }
  
  if (spec.kind === "go") {
    const goInstallFailure = await ensureGoInstalled({ spec, brewExe, timeoutMs });
    if (goInstallFailure) return withWarnings(goInstallFailure, warnings);
  }
  
  // 7. 执行安装命令
  return withWarnings(await executeInstallCommand({
    argv: command.argv,
    timeoutMs,
    env: command.env
  }), warnings);
}
```

**安装配置类型:**
```javascript
// NPM 安装
{ kind: "npm", package: "brave-search-cli" }

// Brew 安装
{ kind: "brew", formula: "wget" }

// UV (Python) 安装
{ kind: "uv", package: "requests" }

// Go 安装
{ kind: "go", module: "github.com/example/cli" }

// 下载
{ 
  kind: "download", 
  url: "https://example.com/skill.tar.gz",
  extract: true 
}
```

---

## 4. 技能调用机制

### 4.1 命令调用

```javascript
// 用户输入：/brave-search "query"
// 解析为技能命令

function buildWorkspaceSkillCommandSpecs(workspaceDir, opts) {
  const userInvocable = filterSkillEntries(
    opts?.entries ?? loadSkillEntries(workspaceDir, opts),
    opts?.config,
    opts?.skillFilter,
    opts?.eligibility
  ).filter((entry) => entry.invocation?.userInvocable !== false);
  
  const specs = [];
  const used = new Set();
  
  for (const entry of userInvocable) {
    const rawName = entry.skill.name;
    const base = sanitizeSkillCommandName(rawName);  // brave-search → brave-search
    const unique = resolveUniqueSkillCommandName(base, used);  // 避免冲突
    
    const description = entry.skill.description?.trim() || rawName;
    
    // 命令分发配置
    const dispatch = (() => {
      const kindRaw = (entry.frontmatter?.["command-dispatch"] ?? "").trim().toLowerCase();
      if (kindRaw !== "tool") return;
      
      const toolName = (entry.frontmatter?.["command-tool"] ?? "").trim();
      if (!toolName) return;
      
      const argModeRaw = (entry.frontmatter?.["command-arg-mode"] ?? "").trim().toLowerCase();
      const argMode = !argModeRaw || argModeRaw === "raw" ? "raw" : null;
      
      return { kind: "tool", toolName, argMode: "raw" };
    })();
    
    specs.push({
      name: unique,
      skillName: rawName,
      description,
      ...dispatch ? { dispatch } : {}
    });
  }
  
  return specs;
}
```

**命令格式:**
```
/<skill-name> [args]

示例:
/brave-search "latest AI news"
/github create-issue --title "Bug" --body "Description"
/weather Beijing
```

---

### 4.2 工具调用

```javascript
// 技能作为工具被模型调用
// 通过 command-dispatch: tool 配置

{
  "name": "brave_search",
  "description": "Search the web using Brave Search",
  "parameters": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "Search query" }
    },
    "required": ["query"]
  }
}
```

**SKILL.md 配置:**
```markdown
---
command-dispatch: tool
command-tool: brave_search
command-arg-mode: raw
---
```

---

### 4.3 系统提示注入

```javascript
function resolveSkillsPromptForRun(params) {
  // 优先使用快照中的提示
  const snapshotPrompt = params.skillsSnapshot?.prompt?.trim();
  if (snapshotPrompt) return snapshotPrompt;
  
  // 或动态生成
  if (params.entries && params.entries.length > 0) {
    const prompt = buildWorkspaceSkillsPrompt(params.workspaceDir, {
      entries: params.entries,
      config: params.config
    });
    return prompt.trim() ? prompt : "";
  }
  
  return "";
}
```

**注入位置:**
```
System Prompt:
You are a helpful assistant with access to these skills:

# Available Skills
- brave-search: Search the web
- github: GitHub operations
...
```

---

## 5. 技能快照管理

### 5.1 快照版本

```javascript
const SKILL_SNAPSHOT_VERSION = 1;

{
  version: 1,
  prompt: "...",
  skills: [...],
  resolvedSkills: [...],
  skillFilter: { include: [], exclude: [] }
}
```

### 5.2 快照持久化

```javascript
// 会话创建时保存技能快照
if (needsSkillsSnapshot) {
  const skillsSnapshot = buildWorkspaceSkillSnapshot(workspaceDir, {
    config: cfg,
    eligibility: { remote: getRemoteSkillEligibility() },
    snapshotVersion: skillsSnapshotVersion,
    skillFilter
  });
  
  // 持久化到会话存储
  sessionEntry.skillsSnapshot = skillsSnapshot;
  await persistSessionEntry({ sessionStore, sessionKey, entry: sessionEntry });
}
```

### 5.3 快照刷新

```bash
# 刷新技能缓存
openclaw skills check

# 重新构建技能快照
openclaw agent --session-key <key> --message "/refresh"
```

---

## 6. 远程技能

### 6.1 远程技能资格

```javascript
const eligibility = {
  remote: {
    enabled: true,   // 是否允许远程技能
    note: "Remote skills available from clawhub.com"
  }
};
```

### 6.2 远程技能同步

```javascript
async function syncSkillsToWorkspace(params) {
  const sourceDir = resolveUserPath(params.sourceWorkspaceDir);
  const targetDir = resolveUserPath(params.targetWorkspaceDir);
  
  if (sourceDir === targetDir) return;
  
  await serializeByKey(`syncSkills:${targetDir}`, async () => {
    const targetSkillsDir = path.join(targetDir, "skills");
    const entries = loadSkillEntries(sourceDir, { config: params.config });
    
    // 清空目标目录
    await fsp.rm(targetSkillsDir, { recursive: true, force: true });
    await fsp.mkdir(targetSkillsDir, { recursive: true });
    
    const usedDirNames = new Set();
    
    // 复制技能
    for (const entry of entries) {
      const dest = resolveSyncedSkillDestinationPath({
        targetSkillsDir,
        entry,
        usedDirNames
      });
      
      if (dest) {
        await fsp.cp(entry.skill.baseDir, dest, {
          recursive: true,
          force: true
        });
      }
    }
  });
}
```

---

## 7. 关键设计模式

### 7.1 技能隔离

- 每个技能独立目录
- 技能间不共享状态 (除非显式)
- 快照确保会话期间技能一致性

### 7.2 渐进式加载

```
扫描目录 → 解析 Frontmatter → 验证结构 → 过滤 → 格式化提示
   ↓           ↓              ↓          ↓         ↓
 快速发现    元数据提取     必需字段    环境检查   模型可读
```

### 7.3 提示限制

- 最大技能数量 (默认 50)
- 最大字符数 (默认 5000)
- 超限自动截断 + 警告

---

## 8. 故障诊断

### 8.1 技能未加载

```bash
# 检查技能目录
ls -la <workspace>/skills/

# 验证 SKILL.md
cat <workspace>/skills/<skill>/SKILL.md

# 查看技能状态
openclaw skills status
```

### 8.2 环境依赖缺失

```bash
# 检查环境变量
echo $BRAVE_API_KEY

# 配置环境变量
openclaw secrets set BRAVE_API_KEY
```

### 8.3 技能安装失败

```bash
# 查看详细错误
openclaw skills install <skill> --verbose

# 手动安装依赖
npm install -g brave-search-cli
```

---

## 9. 相关文档

- [技能系统](https://docs.openclaw.ai/skills)
- [技能开发](https://docs.openclaw.ai/skills/develop)
- [技能安装](https://docs.openclaw.ai/skills/install)
- [Frontmatter 参考](https://docs.openclaw.ai/skills/skill-md)

---

**阶段 1 - 核心架构** 已完成 ✅

**下一步:** 通道系统 或 会话管理 (根据需求)

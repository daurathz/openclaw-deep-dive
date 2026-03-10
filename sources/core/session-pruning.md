# Session Pruning - 会话修剪机制

**阅读时间:** 2026-03-11 00:28  
**源码路径:** `docs/concepts/session-pruning.md`  
**核心职责:** 每次 LLM 调用前，从内存上下文中修剪旧的工具结果，减少上下文膨胀

---

## 📖 关键发现

### 1. Pruning 的定义

```
Session pruning trims old tool results from the in-memory context
right before each LLM call.
```

**核心特点:**
- **不修改磁盘** - 不重写 `.jsonl` 历史文件
- **仅内存操作** - 每次请求前动态修剪
- **仅工具结果** - 用户/助手消息永不修改

**对比 Compaction:**
```
Compaction:  总结并持久化到 JSONL
Pruning:     仅内存中修剪 tool results，每次请求重新计算
```

### 2. 为什么需要 Pruning

**问题:**
```
Anthropic prompt caching 只在 TTL 内有效
  ↓
会话空闲超过 TTL
  ↓
下次请求需要重新缓存完整 prompt → 成本高
  ↓
修剪后：减少 cacheWrite 大小
```

**成本优化:**
```
修剪前：
  完整历史 → cacheWrite 大 → 成本高

修剪后：
  修剪历史 → cacheWrite 小 → 成本低
  缓存窗口重置 → 后续请求复用缓存
```

### 3. 触发条件

**何时运行:**
```
mode: "cache-ttl" 启用
  AND
最后一次 Anthropic 调用 > ttl (默认 5m)
  ↓
执行修剪
  ↓
TTL 窗口重置
```

**仅适用于:**
- ✅ Anthropic API 调用
- ✅ OpenRouter Anthropic 模型
- ❌ 其他 provider

### 4. 智能默认值（Anthropic）

**OAuth/setup-token 配置:**
```json5
{
  contextPruning: { mode: "cache-ttl" },
  heartbeat: "1h"
}
```

**API key 配置:**
```json5
{
  contextPruning: { mode: "cache-ttl" },
  heartbeat: "30m",
  cacheRetention: "short"  // 5m
}
```

**自动匹配:**
- `cacheRetention: "short"` → TTL 5m
- `cacheRetention: "long"` → TTL 1h

### 5. 修剪策略

**保护规则:**
- ✅ 用户消息 - 永不修改
- ✅ 助手消息 - 永不修改
- ✅ 最后 `keepLastAssistants` 条助手消息的工具结果
- ✅ 包含 image blocks 的工具结果
- ❌ 旧的工具结果 - 可修剪

**默认配置:**
```json5
{
  keepLastAssistants: 3,      // 保护最近 3 条
  softTrimRatio: 0.3,         // 软修剪保留 30%
  hardClearRatio: 0.5,        // 硬修剪清除 50%
  minPrunableToolChars: 50000 // 最小可修剪大小
}
```

### 6. 软修剪 vs 硬修剪

**Soft-trim (软修剪):**
```
适用于：超大的工具结果
策略：保留头尾，中间插入 "..."
示例:
  [前 1500 字符]
  ... (省略 8000 字符)
  [后 1500 字符]
  [原始大小：10000 字符]
```

**Hard-clear (硬修剪):**
```
适用于：需要大幅减少
策略：完全替换为占位符
示例:
  [Old tool result content cleared]
```

### 7. 上下文窗口估算

**计算顺序:**
```
1. models.providers.*.models[].contextWindow override
           ↓
2. Model definition contextWindow (registry)
           ↓
3. Default: 200000 tokens
```

**转换公式:**
```
chars ≈ tokens × 4
```

**配置 cap:**
```json5
{
  agents: {
    defaults: {
      contextTokens: 100000  // 最小窗口限制
    }
  }
}
```

### 8. 工具选择配置

**允许/拒绝列表:**
```json5
{
  contextPruning: {
    tools: {
      allow: ["exec", "read"],  // 只修剪这些工具
      deny: ["*image*"]         // 不修剪包含 image 的
    }
  }
}
```

**匹配规则:**
- 支持 `*` 通配符
- Deny 优先
- 大小写不敏感
- 空 allow 列表 = 所有工具

---

## 🎨 设计亮点

### 1. 分层修剪策略

**设计哲学:** "渐进式修剪，保护重要内容"

```
Layer 1: 保护用户/助手消息 (永不修改)
Layer 2: 保护最近助手消息的工具结果
Layer 3: 保护含图片的工具结果
Layer 4: 软修剪旧工具结果 (保留头尾)
Layer 5: 硬修剪超大工具结果 (完全清除)
```

**优势:**
- 保留对话连贯性
- 减少上下文膨胀
- 优化缓存成本

### 2. TTL 感知机制

**创新点:** 与 Anthropic cache TTL 联动

```
TTL 窗口 (5m):
  缓存有效 → 不修剪
      ↓
超过 TTL:
  缓存失效 → 修剪后重新缓存
      ↓
TTL 重置:
  新缓存建立 → 后续请求复用
```

**成本优化:**
```
不修剪：
  完整历史重新缓存 → cacheWrite 大

修剪后：
  修剪历史重新缓存 → cacheWrite 小
```

### 3. 图片保护机制

**规则:** 包含 image blocks 的工具结果永不修剪

**原因:**
- 图片通常是关键上下文
- 重新生成成本高
- 可能无法恢复

**实现:**
```javascript
if (toolResult.blocks.some(b => b.type === "image")) {
  skipPrune();  // 跳过修剪
}
```

### 4. 软修剪算法

**智能截断:**
```
原始：[..................10000 字符..................]
            ↓
修剪：[前 1500] ... (省略 7000) ... [后 1500]
            ↓
结果：保留上下文，减少 70% token
```

**参数:**
```json5
{
  softTrim: {
    maxChars: 4000,    // 最大保留
    headChars: 1500,   // 头部保留
    tailChars: 1500    // 尾部保留
  }
}
```

### 5. 与 Compaction 分离

**设计哲学:** "职责分离，各司其职"

```
Pruning:
  - 频率：每次请求
  - 范围：仅 tool results
  - 持久化：否（仅内存）
  - 目的：减少缓存成本

Compaction:
  - 频率：接近窗口限制
  - 范围：整个对话
  - 持久化：是（JSONL）
  - 目的：释放窗口空间
```

---

## ❓ 疑问/待探索

1. **为什么只支持 Anthropic？**
   - 其他 provider 的缓存策略不同？
   - 未来会扩展支持？

2. **TTL 重置的具体实现？**
   - 如何追踪"最后一次 Anthropic 调用"？
   - 多并发请求如何处理？

3. **软修剪的质量保证？**
   - 如何确保头尾包含关键信息？
   - 是否会丢失重要上下文？

4. **性能影响？**
   - 每次请求前修剪的开销？
   - 大工具结果的修剪耗时？

---

## 🔗 关联概念

- [[Compaction]] - 上下文压缩
- [[Context 组装]] - 上下文构建
- [[Token 使用]] - 缓存成本
- [[Anthropic]] - Provider 特定优化

---

## 📊 源码位置

| 文件 | 路径 | 说明 |
|------|------|------|
| Pruning 文档 | `docs/concepts/session-pruning.md` | 完整修剪机制 |
| Compaction | `docs/concepts/compaction.md` | 压缩机制对比 |
| Config Reference | `docs/gateway/configuration-reference.md` | 配置参数 |

---

## 🔧 实践建议

### 优化修剪策略

1. **启用 TTL 感知修剪**
   ```json5
   {
     agents: {
       defaults: {
         contextPruning: {
           mode: "cache-ttl",
           ttl: "5m"  // 匹配 cacheRetention: "short"
         }
       }
     }
   }
   ```

2. **保护关键工具**
   ```json5
   {
     contextPruning: {
       tools: {
         allow: ["exec", "read"],
         deny: ["*image*", "browser"]  // 保护图片和浏览器结果
       }
     }
   }
   ```

3. **调整保护数量**
   ```json5
   {
     contextPruning: {
       keepLastAssistants: 5  // 保护最近 5 条（默认 3）
     }
   }
   ```

4. **监控缓存成本**
   ```bash
   /usage tokens  # 查看 cacheRead/cacheWrite
   ```

---

_阶段 1：核心架构完成！下一步：阶段 2 - 消息流_

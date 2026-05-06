# LLM Wiki 源码解析 — Two-Step Ingest 核心逻辑

> 核心文件：[ingest.ts](../src/lib/ingest.ts)、[ingest-queue.ts](../src/lib/ingest-queue.ts)

## 1. 设计理念

Two-Step Ingest 受 Karpathy 的 Chain-of-Thought 启发：让 LLM 先"思考"再"输出"，而不是一步到位生成 wiki 页面。

- **Step 1 (Analysis)**：LLM 扮演研究分析师，阅读源文档并生成结构化分析（关键实体、概念、论点、与现有 wiki 的连接、矛盾）
- **Step 2 (Generation)**：LLM 扮演 wiki 维护者，基于分析结果生成 wiki 页面文件和审查项

这种分离带来两个好处：
1. **质量提升**：Analysis 步骤提供了"思考空间"，Generation 步骤可以专注于格式化输出
2. **可审查性**：分析结果对用户可见，用户可以在 chat 模式下与 LLM 讨论后再写入

## 2. 两种摄入模式

### 2.1 自动模式 (autoIngest)

```
用户导入文档 → autoIngest() → 自动完成分析+生成+写入
```

**入口**：`autoIngest(projectPath, sourcePath, llmConfig, signal?, folderContext?)`

这是**批量摄入**的标准路径，由 `ingest-queue.ts` 调用。全自动、无需用户干预。

### 2.2 交互模式 (startIngest + executeIngestWrites)

```
用户导入文档 → startIngest() → 在聊天面板展示分析
                                      │
                                      ▼ 用户可在对话中讨论/补充
                                      │
                              executeIngestWrites() → 写入 wiki 页面
```

**入口**：`startIngest()` → 用户交互 → `executeIngestWrites()`

这是**单文档交互式**路径，用户可以在分析后提供指导，LLM 结合讨论历史生成更精准的页面。

## 3. autoIngest 完整流程

### 3.1 流程图

```
autoIngest(projectPath, sourcePath, llmConfig)
     │
     ▼ withProjectLock ─── 项目级互斥锁
     │
     ▼ autoIngestImpl()
     │
     ├── Step 0: 读取上下文
     │   ├── sourceContent = readFile(sourcePath)
     │   ├── schema = readFile(project/schema.md)
     │   ├── purpose = readFile(project/purpose.md)
     │   ├── index = readFile(project/wiki/index.md)
     │   └── overview = readFile(project/wiki/overview.md)
     │
     ├── Step 0.5: 缓存检查
     │   └── checkIngestCache() → 命中则跳过 LLM 调用
     │
     ├── Step 0.6: 图片提取与描述
     │   ├── extractAndSaveSourceImages() → Rust IPC
     │   ├── captionMarkdownImages() → VLM 流式描述
     │   └── multimodalConfig.enabled=false 时：
     │       └── 从 sourceContent 中剥离 ![](url) 引用
     │
     ├── Step 1: Analysis (LLM 流式调用)
     │   ├── system = buildAnalysisPrompt(purpose, index, content)
     │   ├── user = "Analyze this source document: ..."
     │   └── streamChat() → analysis 累积
     │
     ├── Step 2: Generation (LLM 流式调用)
     │   ├── system = buildGenerationPrompt(schema, purpose, index, ...)
     │   ├── user = "Stage 1 Analysis (context only) + Original Source"
     │   └── streamChat() → generation 累积
     │
     ├── Step 3: 写入文件
     │   ├── parseFileBlocks(generation) → 解析 FILE blocks
     │   ├── isSafeIngestPath() → 路径安全检查
     │   ├── contentMatchesTargetLanguage() → 语言守卫
     │   ├── writeFileBlocks() → 按类型写入策略：
     │   │   ├── log.md → 追加
     │   │   ├── index.md/overview.md → 覆盖
     │   │   └── 其他页面 → 合并 sources 字段后覆盖
     │   └── 回退：无 source summary 时创建最小存根
     │
     ├── Step 3.5: 图片注入
     │   └── injectImagesIntoSourceSummary()
     │       ├── 标记注入区域 <!-- llm-wiki:embedded-images -->
     │       └── 幂等替换（重摄入不累积旧引用）
     │
     ├── Step 4: 解析审查项
     │   └── parseReviewBlocks() → review-store.addItems()
     │
     ├── Step 5: 缓存保存
     │   └── saveIngestCache() — 硬写入失败时跳过缓存
     │
     ├── Step 6: 向量嵌入
     │   └── embedPage() → chunkMarkdown → fetchEmbedding → vector_upsert_chunks
     │
     └── 返回 writtenPaths[]
```

### 3.2 项目级互斥锁

`withProjectLock()` 确保同一项目的两个并发摄入调用排队执行。原因：

- Analysis 步骤读取 `wiki/index.md`
- Generation 步骤覆盖 `wiki/index.md`
- 无序列化时，两个调用会基于相同的旧状态生成"更新"并互相覆盖

### 3.3 缓存机制

```typescript
// ingest-cache.ts
checkIngestCache(projectPath, fileName, sourceContent)
  → 计算源文件内容的哈希
  → 与 ingest-cache.json 中已存储的哈希比较
  → 命中 → 返回已生成的文件列表（跳过 LLM 调用）
  → 未命中 → 返回 null（走完整管道）
```

缓存命中时，图片提取和描述管线仍会运行（幂等操作），确保旧版本摄入的文件在功能升级后能自动收敛到新管线的合约。

### 3.4 内容截断

当源文件超过 50,000 字符时截断：
```typescript
const truncatedContent = enrichedSourceContent.length > 50000
  ? enrichedSourceContent.slice(0, 50000) + "\n\n[...truncated...]"
  : enrichedSourceContent
```

## 4. FILE Block 解析器

### 4.1 格式规范

```
---FILE: wiki/path/to/page.md---
(文件内容，含 YAML frontmatter)
---END FILE---
```

### 4.2 parseFileBlocks() — 关键实现

这是系统中最精巧的解析器之一，处理了 6 类已知的 LLM 输出边界情况：

| 编号 | 问题 | 解决方案 |
|------|------|----------|
| H1 | Windows CRLF 行尾 | 预处理统一为 LF |
| H2 | 流截断（LLM 未输出关闭标记） | 将未关闭的 block 作为 warning 报告 |
| H3 | 标记内空白/大小写变体 | 大小写不敏感 + 容忍额外空白 |
| H5 | 代码块内的 `---END FILE---` | 跟踪代码围栏状态，仅在外部识别关闭标记 |
| H6 | 空路径 | 作为 warning 跳过并报告 |
| 安全 | 路径遍历攻击 | `isSafeIngestPath()` 验证 |

### 4.3 isSafeIngestPath() — 路径安全守卫

```typescript
function isSafeIngestPath(p: string): boolean {
  // 拒绝：空/空白、控制字符、绝对路径、Windows 盘符
  // 拒绝：任何 ".." 段
  // 必须以 "wiki/" 开头
}
```

**威胁模型**：攻击者在源文档中植入 prompt injection，诱导 LLM 输出 `---FILE: ../../../etc/passwd---`，写入系统文件。由于 LLM 生成的路径不可信，必须在解析边界拦截。

### 4.4 代码围栏追踪

```typescript
// 追踪 ``` 或 ~~~ 围栏状态
const FENCE_LINE = /^\s{0,3}(```+|~~~+)/

// 仅在围栏外部识别 ---END FILE---
if (fenceMarker === null && CLOSER_LINE.test(line)) {
  closed = true
  break
}
```

这防止了 LLM 写的"关于 ingest 格式本身的说明文档"中的 `---END FILE---` 被误识别为块关闭标记。

## 5. 文件写入策略

`writeFileBlocks()` 对不同类型的页面采用不同策略：

| 页面类型 | 策略 | 原因 |
|----------|------|------|
| `wiki/log.md` | **追加** | 日志条目是增量事件 |
| `wiki/index.md` | **覆盖** | 列表页，内容完全由当前 wiki 状态决定 |
| `wiki/overview.md` | **覆盖** | 概述页需全面更新 |
| 其他内容页 | **合并 sources 后覆盖** | 多个源文件可能贡献同一页面 |

**sources 合并**（`sources-merge.ts`）：读取磁盘上已有的 `sources: [...]` 字段，与新内容中的 sources 做大小写不敏感去重合并。防止重摄入时覆盖为单一 source（导致后续删除流程误判为单源页面而删除）。

### 5.1 语言守卫

每个 FILE block 的内容都会经过 `contentMatchesTargetLanguage()` 检查：
- 剥离 frontmatter 和代码/数学块
- 对正文前 1500 字符调用 `detectLanguage()`
- CJK 目标语言接受 CJK 变体；拉丁目标语言接受拉丁族
- 跳过 `log.md`、`entities/`、`sources/` 页面（这些页面合法引用跨语言专有名词）

### 5.2 硬失败 vs 软丢弃

- **软丢弃**（语言不匹配、路径遍历、空路径）：代表确定性决策，**缓存保存仍可进行**
- **硬失败**（磁盘满、权限拒绝、OS 级错误）：代表意外损失，**缓存保存必须跳过**（否则后续重摄入将永远回放部分结果）

## 6. 审查项解析

```
---REVIEW: contradiction | Conflicting definitions---
The analysis found conflicts with existing wiki content.
OPTIONS: Create Page | Skip
PAGES: wiki/concepts/attention.md, wiki/entities/transformer.md
SEARCH: automated technical debt detection | software quality metrics LLM
---END REVIEW---
```

解析为 `ReviewItem`：
- `type`: 映射到 contradiction/duplicate/missing-page/suggestion/confirm
- `options`: 从 `OPTIONS:` 行解析，默认 `[Approve, Skip]`
- `affectedPages`: 从 `PAGES:` 行解析
- `searchQueries`: 从 `SEARCH:` 行解析，供 Deep Research 使用

## 7. 摄入队列 (ingest-queue.ts)

### 7.1 设计目标

- **持久化**：队列状态保存到 `.llm-wiki/ingest-queue.json`，应用崩溃/关闭后可恢复
- **并发控制**：单项目单线程处理（受 `withProjectLock` 约束）
- **可中断**：`AbortController` 取消正在进行的 LLM 调用
- **清理**：取消时删除已写入的部分文件 + LanceDB 中的对应向量

### 7.2 生命周期

```
enqueueIngest()  ──▶  [pending]  ──▶  processNext()  ──▶  [processing]
                                                     │
                                    ┌────────────────┼────────────────┐
                                    ▼                ▼                ▼
                               [done → 移除]   [failed → 等待重试]   [cancelled → 清理+移除]
                                    │                │
                                    ▼                ▼
                            processedSinceDrain   retryCount++
                                    │                │
                                    ▼                ▼
                           onQueueDrained()     MAX_RETRIES=3 → [failed]
                                    │
                                    ▼
                           sweepResolvedReviews()
```

### 7.3 项目切换握手

```
pauseQueue(oldProject)
  ├── 中止当前 LLM 调用
  ├── 处理中的任务回退为 pending
  ├── 持久化到旧项目的磁盘文件
  └── 清空内存

restoreQueue(newProjectId, newProjectPath)
  ├── 从磁盘加载队列
  ├── 过滤跨项目任务（防御性检查）
  ├── 处理中任务回退为 pending
  └── 恢复处理
```

### 7.4 陈旧上下文守卫

`processNext()` 和 `onQueueDrained()` 都有 `currentProjectId !== projectId` 检查，防止项目切换后孤儿回调写入错误的项目。

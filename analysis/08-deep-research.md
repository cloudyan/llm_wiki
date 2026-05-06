# LLM Wiki 源码解析 — Deep Research 深度研究

> 核心文件：[deep-research.ts](../src/lib/deep-research.ts)、[research-store.ts](../src/stores/research-store.ts)

## 1. 功能概述

Deep Research 是 LLM Wiki 的"主动知识获取"功能：对审查项中的开放问题自动进行 Web 搜索、LLM 综合，并将结果写入 wiki 再自动摄入。

```
审查项 (REVIEW block)
     │ "missing-page" / "suggestion" + SEARCH 查询
     ▼
queueResearch(projectPath, topic, llmConfig, searchConfig, searchQueries?)
     │
     ├── Web 搜索 (多查询合并去重)
     │
     ├── LLM 综合 (流式输出到 UI)
     │
     ├── 保存为 wiki/queries/research-*.md
     │
     └── 自动摄入 (autoIngest)
         └── 生成实体、概念、交叉引用
```

## 2. 任务队列

### 2.1 并发控制

```typescript
// research-store.ts
interface ResearchState {
  tasks: ResearchTask[]
  maxConcurrent: number      // 最大并发任务数
  panelOpen: boolean          // 面板是否打开
}
```

`processQueue()` 确保运行中的任务不超过 `maxConcurrent`：

```typescript
function processQueue(projectPath, llmConfig, searchConfig) {
  const running = store.getRunningCount()
  const available = store.maxConcurrent - running
  for (let i = 0; i < available; i++) {
    const next = store.getNextQueued()
    if (!next) break
    executeResearch(projectPath, next.id, next.topic, llmConfig, searchConfig)
  }
}
```

### 2.2 任务生命周期

```
[queued] → [searching] → [synthesizing] → [saving] → [done]
                │              │               │
                ▼              ▼               ▼
            [error]        [error]          [error]
```

## 3. 执行流程

### 3.1 Step 1: Web 搜索

```typescript
// 支持多个搜索查询（来自 REVIEW block 的 SEARCH 字段）
const queries = task.searchQueries ?? [topic]

for (const query of queries) {
  const results = await webSearch(query, searchConfig, 5)
  // 合并去重（按 URL）
}
```

**搜索配置**：当前仅支持 Tavily API：

```typescript
interface SearchApiConfig {
  provider: "tavily" | "none"
  apiKey: string
}
```

### 3.2 Step 2: LLM 综合

```typescript
// System Prompt 关键指令：
"1. 综合搜索结果为完整的 wiki 页面"
"2. 引用来源使用 [N] 标记"
"3. 使用 [[wikilink]] 链接到已有 wiki 页面"
"4. 注意矛盾和知识空白"
"5. 建议进一步查找的来源"
"6. 中性、百科全书式语调"

// User Message:
"Research topic: {topic}
 Web Search Results: [1] Title (source) snippet ...
 Synthesize into a wiki page."
```

**交叉引用**：LLM 综合时会读取现有 `wiki/index.md`，确保新研究页面能通过 `[[wikilink]]` 链接到已有知识。

**流式展示**：LLM 的综合输出通过 `updateTask(taskId, { synthesis: accumulated })` 实时更新 UI。

### 3.3 Step 3: 保存与自动摄入

```typescript
// 1. 清理 <thinking> 块
const cleanedSynthesis = accumulated
  .replace(/<think(?:ing)?>[\s\S]*?<\/think(?:ing)?>/gi, "")
  .trimStart()

// 2. 构建研究页面
const pageContent = [
  "---",
  `type: query`,
  `title: "Research: ${topic}"`,
  `created: ${date}`,
  `origin: deep-research`,
  `tags: [research]`,
  "---",
  "",
  `# Research: ${topic}`,
  "",
  cleanedSynthesis,
  "",
  "## References",
  "",
  references,       // 编号链接列表
].join("\n")

// 3. 写入 wiki/queries/research-{slug}-{date}.md
await writeFile(filePath, pageContent)

// 4. 自动摄入 — 生成实体、概念、交叉引用
autoIngest(pp, `${pp}/${savedPath}`, llmConfig).catch(...)
```

**关键设计**：研究页面保存后立即调用 `autoIngest()`，将综合结果作为新的源文档重新处理。这使得一次深度研究不仅产出研究页面本身，还会生成对应的实体页、概念页，并更新 index.md 和 overview.md。

## 4. 与审查系统的联动

### 4.1 触发路径

```
[ingest.ts] parseReviewBlocks()
     │
     ├── type: "missing-page" + SEARCH 查询
     │   → 审查项出现在 Review 面板
     │
     └── 用户点击 "Deep Research" 按钮
         → queueResearch(projectPath, title, llmConfig, searchConfig, searchQueries)
```

审查项的 `searchQueries` 字段由 LLM 在 Generation 阶段生成，专为 Web 搜索优化的关键词组合：

```
SEARCH: automated technical debt detection AI generated code | software quality metrics LLM code generation | static analysis tools agentic software development
```

### 4.2 结果闭环

```
Deep Research → 保存研究页面 → autoIngest → 生成实体/概念页
                                                    │
                                                    ▼
                                          下次摄入时，这些页面
                                          会被 index.md 收录，
                                          知识图谱会新增节点和边
```

## 5. 错误处理

- **搜索无结果**：直接标记为 done，synthesis = "No web results found."
- **LLM 流式错误**：标记为 error，保存错误信息
- **文件写入失败**：try-catch 包裹，不影响其他任务
- **自动摄入失败**：`.catch()` 静默处理，研究页面仍保留

## 6. 队列接力

```
onTaskFinished()
     │
     └── setTimeout(100ms) → processQueue()
         └── 处理下一个排队任务
```

100ms 延迟让 React 有时间渲染状态更新，避免 UI 冻结。

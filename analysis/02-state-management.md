# LLM Wiki 源码解析 — 前端状态管理与数据流

## 1. 状态管理选型：Zustand

LLM Wiki 使用 Zustand 5 作为全局状态管理方案，选择原因：
- **零样板代码**：无需 action/reducer 定义，直接 `set({ key: value })`
- **细粒度订阅**：组件通过 selector 只订阅所需切片，避免不必要的重渲染
- **外部可访问**：`useStore.getState()` 允许非 React 代码（lib/ 中的业务逻辑）直接读写状态
- **持久化简单**：配合 `@tauri-apps/plugin-store` 实现磁盘持久化

## 2. Store 架构

### 2.1 Store 清单

| Store | 文件 | 职责 |
|-------|------|------|
| `wiki-store` | [wiki-store.ts](../src/stores/wiki-store.ts) | 项目、文件树、LLM/嵌入/多模态/搜索配置 |
| `chat-store` | [chat-store.ts](../src/stores/chat-store.ts) | 对话历史、流式消息、当前模式 |
| `activity-store` | [activity-store.ts](../src/stores/activity-store.ts) | 摄入队列进度追踪 |
| `review-store` | [review-store.ts](../src/stores/review-store.ts) | 审查项（矛盾/重复/缺失/建议） |
| `research-store` | [research-store.ts](../src/stores/research-store.ts) | Deep Research 任务状态 |
| `update-store` | [update-store.ts](../src/stores/update-store.ts) | 版本更新检测与提示 |

### 2.2 wiki-store — 核心状态枢纽

`wiki-store` 是最核心的 Store，承载了项目全局状态和所有 LLM 相关配置：

```typescript
// 关键状态切片
interface WikiStoreState {
  // 项目上下文
  project: WikiProject | null        // 当前项目
  fileTree: FileNode[]               // 文件树
  selectedFile: string | null        // 选中的 wiki 页面
  activeView: string                 // 当前视图 (wiki/chat/graph/settings/...)
  dataVersion: number                // 数据版本号（触发缓存失效）

  // LLM 配置
  llmConfig: LlmConfig               // 主 LLM 提供商配置
  providerConfigs: Record<string, Partial<LlmConfig>>  // 预设覆盖
  activePresetId: string             // 活跃预设 ID

  // 向量搜索配置
  embeddingConfig: EmbeddingConfig

  // 多模态配置
  multimodalConfig: MultimodalConfig  // 图片描述管线开关

  // 搜索 API 配置
  searchApiConfig: SearchApiConfig    // Tavily Web 搜索

  // 输出语言
  outputLanguage: OutputLanguage      // "auto" | "English" | "Chinese" | ...
}
```

**关键设计点：**
- `dataVersion` 是一个单调递增计数器，`bumpDataVersion()` 在文件写入后调用，让图谱缓存和搜索结果自动失效
- `providerConfigs` + `activePresetId` 实现了预设系统：预设定义默认值，用户覆盖存在 `providerConfigs[presetId]` 中，`resolveConfig()` 合并两者
- `llmConfig.maxContextSize` 以**字符数**为单位（非 token），这是系统的一个设计怪癖

### 2.3 chat-store — 对话与摄入模式

```typescript
interface ChatStoreState {
  messages: ChatMessage[]             // 对话消息
  mode: "chat" | "ingest"            // 当前模式
  streaming: boolean                  // 是否正在流式输出
  streamContent: string               // 累积的流式内容
  ingestSource: string | null         // 摄入的源文件路径
  activeConversation: string | null   // 当前对话 ID
  conversations: Conversation[]       // 对话列表
}
```

**模式切换逻辑：**
- `mode: "ingest"` — 用户导入文档后进入，LLM 分析源文档并在对话中展示
- `mode: "chat"` — 自由对话模式，支持"Save to Wiki"将讨论结果写入 wiki
- `ingestSource` 在 `startIngest()` 中设置，`executeIngestWrites()` 读取以确定图片注入目标

### 2.4 activity-store — 摄入进度追踪

```typescript
interface ActivityItem {
  id: string
  type: "ingest" | "embed" | "research" | "sweep"
  title: string
  status: "running" | "done" | "error"
  detail: string          // 人类可读的进度描述
  filesWritten: string[]  // 已写入的文件列表
}
```

Activity 面板实时展示摄入进度，`detail` 字段随管道推进动态更新：
- "Reading source..." → "Step 1/2: Analyzing source..." → "Step 2/2: Generating wiki pages..." → "Writing files..."

### 2.5 review-store — 审查项管理

```typescript
interface ReviewItem {
  id: string
  type: "contradiction" | "duplicate" | "missing-page" | "suggestion" | "confirm"
  title: string
  description: string
  sourcePath: string
  affectedPages?: string[]
  searchQueries?: string[]   // Deep Research 搜索查询
  options: { label: string; action: string }[]
  resolved: boolean
  createdAt: number
}
```

审查项由 LLM 在 Generation 阶段通过 `---REVIEW:...---END REVIEW---` 块生成，`sweep-reviews.ts` 会在队列排空后自动清理已解决的审查项。

## 3. 持久化架构

### 3.1 持久化分层

```
┌─────────────────────────────────────────────────┐
│  App Startup (App.tsx → init())                  │
│                                                   │
│  ┌───────────────┐  ┌───────────────────────┐    │
│  │ project-store │  │ @tauri-apps/plugin-   │    │
│  │ (自定义 JSON) │  │ store (KV store)      │    │
│  └───────────────┘  └───────────────────────┘    │
│         │                      │                  │
│         ▼                      ▼                  │
│  loadLlmConfig()         loadLanguage()           │
│  loadProviderConfigs()   loadSearchApiConfig()    │
│  loadEmbeddingConfig()   loadMultimodalConfig()   │
│  loadOutputLanguage()    getLastProject()         │
│  loadActivePresetId()   getRecentProjects()       │
└─────────────────────────────────────────────────┘
```

### 3.2 项目级持久化

每个项目的运行时状态持久化在 `{project}/.llm-wiki/` 目录下：

| 文件 | 内容 | 读写时机 |
|------|------|----------|
| `ingest-queue.json` | 摄入队列（pending/failed 任务） | 入队/完成/取消/暂停/恢复 |
| `ingest-cache.json` | 源文件内容哈希 → 已生成文件列表 | 摄入完成时保存，缓存命中时跳过 |
| `image-caption-cache.json` | 图片 SHA-256 → 描述文本 | 图片描述完成后缓存 |
| `lancedb/` | LanceDB 向量索引 | 嵌入/搜索/删除 |

### 3.3 项目切换的原子性保证

项目切换时，`resetProjectState()` 被调用来清空所有项目相关状态：

```typescript
// App.tsx → handleProjectOpened()
const { resetProjectState } = await import("@/lib/reset-project-state")
await resetProjectState()  // 必须等待完成！
setProject(proj)           // 然后才设置新项目
```

这防止了跨项目状态污染（旧项目的文件树/审查项/对话泄漏到新项目）。`pauseQueue()` 也会在切换前将运行中的任务回退为 `pending` 并持久化到磁盘。

## 4. 跨组件数据流

### 4.1 典型数据流：文档摄入

```
[Sources View]
     │ 用户选择文件
     ▼
enqueueIngest(projectId, sourcePath)
     │
     ▼
[ingest-queue.ts] processNext()
     │ 调用 autoIngest()
     ▼
[ingest.ts] autoIngestImpl()
     │ 1. 读取源文件 (Tauri IPC)
     │ 2. 缓存检查 (ingest-cache)
     │ 3. 图片提取 (Tauri IPC)
     │ 4. 图片描述 (VLM streamChat)
     │ 5. Step 1: LLM Analysis (streamChat)
     │ 6. Step 2: LLM Generation (streamChat)
     │ 7. 解析 FILE blocks → 写入文件
     │ 8. 解析 REVIEW blocks → review-store
     │ 9. 向量嵌入 (embedding.ts)
     ▼
[wiki-store] bumpDataVersion() → 图谱/搜索缓存失效
[activity-store] 更新状态为 "done"
[review-store] 新增审查项
```

### 4.2 典型数据流：搜索

```
[Search View]
     │ 用户输入查询
     ▼
searchWiki(projectPath, query)
     │
     ├──→ tokenizeQuery() → 中英文分词
     │         │
     │         ▼
     │    searchFiles() → 逐文件评分
     │         │
     │         ▼
     │    tokenRank = Map<page, rank>
     │
     ├──→ searchByEmbedding() → 向量搜索
     │         │
     │         ▼
     │    vectorRank = Map<page, rank>
     │
     └──→ RRF Fusion: score = 1/(K+tokenRank) + 1/(K+vectorRank)
              │
              ▼
         排序 → 返回 Top 20 结果
```

### 4.3 Store 与业务逻辑的交互模式

```
         ┌──────────┐
         │ Component │──subscribe──▶ Store Slice
         └────┬─────┘
              │ dispatch action
              ▼
         ┌──────────┐
         │  Store    │──getState()──▶ Business Logic (lib/)
         └──────────┘◀──setState()─── Business Logic (lib/)
```

**关键模式**：`lib/` 中的业务逻辑通过 `useStore.getState()` 直接读写 Store，不需要经过 React 组件中转。这使得 Ingest Queue、Deep Research 等异步流程可以直接更新 UI 状态。

## 5. 初始化流程

应用启动时 `App.tsx` 的 `init()` 函数执行以下步骤（按顺序）：

1. **加载 LLM 配置** → `loadLlmConfig()` → `wiki-store.setLlmConfig()`
2. **加载预设配置** → `loadProviderConfigs()` → 重新解析活跃预设（合并默认值 + 用户覆盖）
3. **加载搜索/嵌入/多模态/语言配置**
4. **恢复上次项目** → `getLastProject()` → `openProject()` → `handleProjectOpened()`
5. **项目打开后**：
   - 重置项目状态（`resetProjectState()`）
   - 恢复摄入队列（`restoreQueue()`）
   - 通知 Clip Server 当前项目
   - 加载文件树
   - 加载审查项和对话历史

**延迟初始化**：更新检测在挂载后 1.5 秒触发，避免与最重的启动工作竞争。

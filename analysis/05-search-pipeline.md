# LLM Wiki 源码解析 — 搜索管道与知识图谱

> 核心文件：[search.ts](../src/lib/search.ts)、[graph-relevance.ts](../src/lib/graph-relevance.ts)、[embedding.ts](../src/lib/embedding.ts)、[context-budget.ts](../src/lib/context-budget.ts)

## 1. 混合搜索架构

LLM Wiki 实现了三层搜索信号融合：

```
用户查询 "attention mechanism"
        │
        ├──── Token Search ────┐
        │   (分词+关键词匹配)    │
        │                      │
        ├──── Vector Search ───┤── RRF Fusion ──▶ Top 20 结果
        │   (语义相似度)        │
        │                      │
        └──── Graph Expansion ─┘   (聊天上下文组装时)
            (4信号相关性)
```

## 2. Token Search — 关键词匹配

### 2.1 中英文分词 (tokenizeQuery)

```typescript
function tokenizeQuery(query: string): string[] {
  // 1. 按空白和标点分割
  // 2. 过滤停用词 (中英文)
  // 3. CJK 文本特殊处理：
  //    - 长度>2 的 CJK token → 生成 bigrams + 单字 + 原始 token
  //    - 例: "默会知识" → ["默会", "会知", "知识", "默", "会", "知", "识", "默会知识"]
  // 4. 去重
}
```

**设计权衡**：不做完整 NLP 分词（不需要 jieba 等重型库），bigrams 提供足够的中文短语覆盖。

### 2.2 多层评分信号

```
score = FILENAME_EXACT_BONUS (200)      // 文件名精确匹配
      + PHRASE_IN_TITLE_BONUS (50)      // 标题包含完整查询短语
      + contentPhraseOcc × 20            // 正文短语出现次数 (上限10)
      + titleTokenScore × 5             // 标题 token 匹配数
      + contentTokenScore × 1           // 正文 token 匹配数
```

**信号优先级**：精确匹配 >> 短语匹配 >> token 匹配。当用户输入 "attention" 时，`attention.md` 必须排名第一，无论其他页面多少次提到这个词。

### 2.3 并发读取控制

```typescript
const SEARCH_READ_CONCURRENCY = 16  // 最大并发 readFile 数

// 分批读取，避免 IPC 通道饱和
for (let i = 0; i < files.length; i += SEARCH_READ_CONCURRENCY) {
  const batch = files.slice(i, i + SEARCH_READ_CONCURRENCY)
  const batchResults = await Promise.all(batch.map(/* ... */))
}
```

实测 200 文件项目，并发超过 16-32 后反而变慢（IPC 通道排队）。

### 2.4 图片引用提取

搜索结果附带页面中的图片引用：

```typescript
interface SearchResult {
  path: string
  title: string
  snippet: string
  titleMatch: boolean
  score: number
  images: ImageRef[]  // ![](url) 提取结果
}
```

UI 区分"alt 文本匹配查询"和"图片恰好在匹配页面上"。

## 3. Vector Search — 语义搜索

### 3.1 嵌入管线

```
embedPage(projectPath, pageId, title, content, cfg)
     │
     ├── chunkMarkdown(content, { targetChars: 1000, overlapChars: 200 })
     │       │
     │       ▼
     │   [Chunk { index, text, headingPath }]
     │
     ├── for each chunk:
     │   ├── enrichChunkForEmbedding(title, chunk)
     │   │       → pageTitle + headingPath + chunkText
     │   └── fetchEmbedding(enrichedText, cfg)
     │           ├── 成功 → 收集向量
     │           └── 失败 → auto-halve retry (最多3次)
     │
     └── vectorUpsertChunks(projectPath, pageId, rows)
             → Tauri IPC → LanceDB 写入
```

### 3.2 分块策略 (text-chunker.ts)

Markdown 感知的分块器：
- 按标题层级切分（`#`、`##`、`###` 为自然边界）
- `headingPath` 记录面包屑（如 `## Background > ### Attention`）
- 重叠区域（`overlapChars`）防止段落边界的语义断裂
- 跳过 frontmatter

### 3.3 向量搜索流程

```
searchByEmbedding(projectPath, query, cfg, topK=10)
     │
     ├── fetchEmbedding(query, cfg) → queryEmb
     │
     ├── vectorSearchChunks(projectPath, queryEmb, topK × 3)
     │       → 过量获取 topK×3 个 chunk
     │
     └── 按 page_id 分组，计算 blended score:
         blended = max(chunk_scores) + min(tail_sum × 0.3, 1 - max_score)
         │
         └── 排序 → 返回 Top K 页面
```

**分组评分策略**：一个页面有多个匹配 chunk 时，主分数取最高 chunk 分数，辅助分数取其余 chunk 的加权和（上限封顶防止单页面靠 chunk 数量碾压）。

### 3.4 Auto-Halve 重试

嵌入请求在"输入过长"错误时自动减半重试：

```typescript
fetchEmbedding(text, cfg, maxRetries = 3)
  → HTTP 413 / "too long" / "exceeds context"
  → text = text.slice(0, text.length / 2)
  → 重试，直到 64 字符下限
```

## 4. RRF 融合 (Reciprocal Rank Fusion)

### 4.1 算法

```
fused(p) = Σ 1/(K + rank_L(p))   其中 L ∈ {token, vector}
K = 60 (Cormack et al., SIGIR 2009)
```

**为什么用 RRF 而非分数加权**：
- Token score 范围：1-400
- Vector cosine 范围：0-1
- 绝对分数不可比较，加权求和会使数值大的信号主导
- RRF 只看排名，完全消除分数量纲差异

### 4.2 实现

```typescript
// 1. Token 侧排名快照
const tokenSorted = [...results].sort((a, b) => b.score - a.score)
const tokenRank = new Map<string, number>()  // page → rank

// 2. Vector 侧排名
const vectorRank = new Map<string, number>() // page_id → rank

// 3. Vector-only 页面补充到 results 中
//    (token 搜索没找到但向量搜索命中的页面)

// 4. RRF 融合
for (const r of results) {
  r.score = (tokenRank ? 1/(60+tokenRank) : 0)
          + (vectorRank ? 1/(60+vectorRank) : 0)
}
```

## 5. 知识图谱相关性模型

### 5.1 RetrievalGraph 构建

```typescript
buildRetrievalGraph(projectPath, dataVersion)
     │
     ├── 缓存检查: dataVersion 匹配则返回缓存
     │
     ├── 遍历 wiki/ 下所有 .md 文件
     │   ├── extractFrontmatter() → title, type, sources
     │   └── extractWikilinks() → [[target]] 链接列表
     │
     ├── 第一遍: 构建 raw nodes
     │
     ├── 第二遍: 解析 wikilinks → 建立有向边
     │   └── resolveTarget() → 容忍大小写/空格/连字符差异
     │
     └── 构建 RetrievalGraph { nodes: Map<id, RetrievalNode> }
```

### 5.2 四信号相关性模型

`calculateRelevance(nodeA, nodeB, graph)` 计算两个节点的相关性分数：

| 信号 | 权重 | 描述 |
|------|------|------|
| **直接链接** | 3.0 | A→B 或 B→A 的 [[wikilink]] |
| **源重叠** | 4.0 | 共享相同的源文件（sources 字段交集） |
| **共同邻居** | 1.5 | Adamic-Adar 指标：1/log(degree(neighbor)) |
| **类型亲和** | 1.0 | 页面类型组合的预定义权重矩阵 |

**类型亲和矩阵**（部分）：

```typescript
TYPE_AFFINITY = {
  entity:  { concept: 1.2, entity: 0.8, source: 1.0, synthesis: 1.0 },
  concept: { entity: 1.2, concept: 0.8, source: 1.0, synthesis: 1.2 },
  source:  { entity: 1.0, concept: 1.0, source: 0.5, synthesis: 1.0 },
  // ...
}
```

设计直觉：`concept↔entity` 高亲和（概念涉及实体），`source↔source` 低亲和（同一源文件的不同摘要不太相关），`concept↔synthesis` 高亲和（综合分析基于概念）。

### 5.3 图谱在聊天中的使用

```
用户提问
     │
     ├── Token + Vector 搜索 → 初始结果集
     │
     └── Graph Expansion:
         for each result page:
           getRelatedNodes(pageId, graph, limit=5)
           → 扩展到强相关页面
           → 填充聊天上下文
```

## 6. Context Budget — Token 预算分配

### 6.1 预算结构

```
┌─────────────────────────────────────────────────────┐
│              maxCtx (100%)                           │
├──────┬───────────────┬──────────────────┬───────────┤
│ idx  │   pages       │  history + sys   │  resp     │
│  5%  │    50%        │    ~30%          │   15%     │
└──────┴───────────────┴──────────────────┴───────────┘
```

```typescript
interface ContextBudget {
  maxCtx: number           // 全部上下文窗口
  responseReserve: number  // 15% — 给 LLM 回答留空间
  indexBudget: number      // 5% — 索引摘要（列出所有页面标题）
  pageBudget: number       // 50% — 检索到的 wiki 页面内容
  maxPageSize: number      // 单页面截断上限
}
```

### 6.2 maxPageSize 计算

```typescript
maxPageSize = Math.min(
  pageBudget,                                      // 不超过总页面预算
  Math.max(PER_PAGE_FLOOR, pageBudget * 0.3)       // 至少 5K，通常 30%
)
```

早期版本硬编码 30K 上限，在长上下文模型上浪费了预算。现在随 `pageBudget` 线性缩放。

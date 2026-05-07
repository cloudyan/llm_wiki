# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

LLM Wiki 是一个基于 Tauri v2 的跨平台桌面应用，实现 Karpathy 的 LLM Wiki 模式：将文档自动构建为结构化的、互相链接的知识库。

**核心特性**：Two-Step Chain-of-Thought Ingest、知识图谱（sigma.js + graphology）、可选向量搜索（LanceDB）、Deep Research、Chrome 浏览器插件。

## 开发命令

```bash
# 前端开发
npm run dev          # Vite 开发服务器（端口 1420）

# Tauri 应用
npm run tauri dev    # 开发模式（热重载）
npm run tauri build  # 生产构建（含 typecheck + vite build）

# 代码质量
npm run typecheck    # TypeScript 类型检查（strict 模式，noUnusedLocals/noUnusedParameters）
npm run build        # typecheck + vite build

# 测试
npm test             # 运行所有测试（mock + real-LLM）
npm run test:mocks   # 仅运行 mock 测试（排除 *.real-llm.test.ts）
npm run test:llm     # 仅运行 real-LLM 测试（需要 .env.test.local 中的 API key）

# 运行单个测试文件
npx vitest run src/lib/path-utils.test.ts
npx vitest run src/lib/ingest-parse.test.ts
```

**测试架构**：
- Mock 测试：默认运行，不依赖真实 LLM API，使用 `src/test-helpers/mock-stream-chat.ts` 模拟 `streamChat`
- Real-LLM 测试：文件名后缀为 `*.real-llm.test.ts`，需要 `.env.test.local` 配置 API key，串行执行（`--no-file-parallelism`）
- Property 测试：使用 `fast-check`（`*.property.test.ts`）进行随机化测试
- Scenario 测试：`*.scenarios.test.ts` 测试端到端流程
- Integration 测试：`*.integration.test.ts` 测试跨模块交互

## 路径别名

Vite 和 TypeScript 使用 `@/*` → `./src/*` 别名。代码中统一使用 `@/` 导入，不要用相对路径跨目录导入。

## 架构概览

### 三层内容架构（Karpathy 设计）

```
项目目录结构：
  purpose.md       # 目标与意图（LLM 每次 ingest/query 都读取）
  schema.md        # Wiki 结构规则与页面类型定义
  raw/sources/     # 原始文档（immutable，不可修改）
  raw/assets/      # 本地图片
  wiki/            # LLM 生成的 wiki 页面（index.md 是导航入口）
  .llm-wiki/       # 应用配置、聊天历史、review items、缓存
  .obsidian/       # Obsidian vault 配置（自动生成）
```

### 前端 (`src/`)

- **状态管理** (`src/stores/`)：Zustand，所有状态持久化（Tauri Store）
  - `wiki-store.ts` - 项目、文件树、LLM 配置、视图状态（含 LlmConfig/EmbeddingConfig/MultimodalConfig 类型）
  - `chat-store.ts` - 多会话对话历史
  - `activity-store.ts` - 摄入队列进度可视化
  - `review-store.ts` - 审查项（异步 human-in-the-loop）
  - `research-store.ts` - Deep Research 状态
  - `update-store.ts` - 版本更新检查

- **核心业务逻辑** (`src/lib/`)
  - `ingest.ts` - Two-Step Ingest（分析 → 生成），FILE block 解析
  - `ingest-sanitize.ts` - 清理 LLM 生成的 wiki 页面（去代码围栏、修复 frontmatter）
  - `ingest-cache.ts` - SHA256 增量缓存，未变更源文件自动跳过
  - `ingest-queue.ts` - 持久化摄入队列，串行处理，崩溃恢复
  - `search.ts` - 混合搜索：tokenized + vector（RRF 融合）+ graph expansion
  - `llm-client.ts` + `llm-providers.ts` - 多提供商流式聊天（OpenAI/Anthropic/Google/Ollama/Custom/Claude Code CLI/Minimax），provider-specific wire format
  - `claude-cli-transport.ts` - Claude Code CLI 子进程流式传输
  - `graph-relevance.ts` - 4 信号相关性模型
  - `graph-insights.ts` - 知识图谱洞察（惊喜连接、知识缺口）
  - `context-budget.ts` - Token 预算分配（60/20/5/15）
  - `deep-research.ts` - Web 研究 + 自动摄入（Tavily/SerpApi）
  - `page-merge.ts` - 重新摄入时的 wiki 页面合并策略
  - `project-mutex.ts` - 项目级操作互斥锁（防止并发 ingest）
  - `reasoning-detector.ts` - `<think>` / `<tool_call>tag` reasoning block 检测与流式展示

- **组件** (`src/components/`)
  - `layout/` - AppLayout、侧边栏、面板布局（react-resizable-panels）
  - `chat/` - 聊天输入、消息渲染、cited references panel
  - `editor/` - Milkdown wiki 编辑器（WYSIWYG）+ wiki-reader（只读预览）
  - `graph/` - Sigma.js 知识图谱可视化（ForceAtlas2 布局）
  - `settings/` - LLM 提供商配置（sections/ 分模块）
  - `sources/` - 源文件管理
  - `ui/` - shadcn/ui 基础组件（button, dialog, input 等）

- **Tauri 命令封装** (`src/commands/fs.ts`)：调用 Rust 后端

- **国际化** (`src/i18n/`)：react-i18next，英文 `en.json` + 中文 `zh.json`

### 后端 (`src-tauri/`)

- `src/lib.rs` - Tauri 应用入口，注册命令和插件
- `src/commands/`
  - `fs.rs` - 文件读写、Office/PDF 解析（pdfium-render、docx-rs、calamine）
  - `vectorstore.rs` - LanceDB 向量操作
  - `project.rs` - 项目创建/打开
  - `extract_images.rs` - PDF/Office 图片提取
  - `claude_cli.rs` - Claude Code CLI 子进程管理

### Chrome 扩展 (`extension/`)

Manifest V3 扩展，通过 `tiny_http` 本地服务器（端口 19827）与主应用通信。使用 Readability.js + Turndown.js 提取网页内容。

## 关键设计决策

### Two-Step Ingest
1. **Analysis**：LLM 分析源文档 → 结构化分析结果
2. **Generation**：基于分析结果生成 wiki 页面

FILE block 格式：`---FILE: path---\ncontent\n---END FILE---`

**安全机制**：
- `ingest-sanitize.ts` 清理 LLM 输出（去代码围栏包裹、修复 `frontmatter:` 前缀、修复 wikilink 列表格式）
- `path-utils.ts` 的 `normalizePath()` 统一处理 Windows/Linux 路径差异（`\` → `/`），被 40+ 文件引用
- 所有路径操作使用 `normalizePath()`，禁止使用原始反斜杠路径

### 搜索管道
1. Tokenized search（支持中英文分词，CJK bigram）
2. 可选 Vector search（RRF 融合，LanceDB ANN）
3. Graph expansion（4 信号相关性，2-hop 衰减遍历）
4. Budget control（60% wiki / 20% chat history / 5% index / 15% system）

### LLM 多提供商架构
- `llm-providers.ts` 定义统一接口 `ChatMessage` + `ContentBlock`（文本/图片），各 provider 的 `buildBody` 转换为原生 wire format
- `llm-client.ts` 的 `streamChat` 是核心入口，根据 `provider` 字段分发到对应的实现
- Claude Code CLI 模式：通过 `claude-cli-transport.ts` spawn 子进程，流式读取 stdout
- Reasoning 模式：`reasoning-detector.ts` 检测 `<think>` block，流式展示 thinking 过程

### 并发控制
- `project-mutex.ts` 保证同一项目的 ingest 操作串行执行
- `ingest-queue.ts` 持久化队列，崩溃后可恢复，失败自动重试（最多 3 次）
- `dedup-queue.ts` 防止重复摄入同一文件

### 平台差异
- macOS：关闭按钮隐藏窗口（Cmd+Q 退出）
- Windows/Linux：关闭时弹出确认对话框

### i18n
- `src/i18n/en.json` 和 `src/i18n/zh.json` 必须保持键的完全一致（有 parity 测试校验）
- 添加新 UI 文字时，两个语言文件都要更新

## 类型定义

核心类型在 `src/types/wiki.ts`：`WikiProject`（含稳定 UUID）、`FileNode`、`WikiPage`

配置类型在 `src/stores/wiki-store.ts`：
- `LlmConfig` - LLM 提供商配置（provider/apiKey/model/customEndpoint/apiMode/reasoning）
- `EmbeddingConfig` - 向量嵌入配置（含 chunking 参数）
- `MultimodalConfig` - 多模态（图片描述）配置
- `SearchProvider` / `SearchApiConfig` - 搜索提供商（tavily/serpapi/none）
- `ReasoningConfig` - 思考模式配置（auto/off/low/medium/high/max/custom）

## 测试辅助

`src/test-helpers/` 提供测试基础设施：
- `mock-stream-chat.ts` - 可控的 streamChat mock，支持挂起/手动完成/取消
- `deferred.ts` - Promise 手动控制（用于异步测试）
- `fs-temp.ts` - 临时文件系统辅助
- `real-content.ts` - 真实内容样本
- `load-test-env.ts` - 加载 `.env.test.local`（real-LLM 测试需要）

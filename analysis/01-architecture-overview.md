# LLM Wiki 源码解析 — 项目架构总览

> 版本：v0.4.3 | 基于 Tauri v2 + React 19 + Zustand + Rust

## 1. 项目定位

LLM Wiki 实现了 Karpathy 的 **LLM Wiki 模式**：将文档自动构建为结构化、互相链接的知识库。核心思想是"LLM 驱动的知识提取与组织"——用户丢入源文档，系统自动分析、生成 wiki 页面、建立知识图谱、支持混合搜索和深度研究。

## 2. 技术栈总览

```
┌─────────────────────────────────────────────────────────────┐
│                      LLM Wiki Desktop                       │
├──────────────────────┬──────────────────────────────────────┤
│    Frontend (TS/React)          │    Backend (Rust/Tauri)   │
├──────────────────────┼──────────────────────────────────────┤
│  React 19            │  Tauri v2 (protocol-asset)           │
│  Zustand 5 (状态)     │  pdfium-render (PDF解析)             │
│  Vite 8 (构建)       │  calamine (Excel/ODS)                │
│  Milkdown (Wiki编辑) │  docx-rs (Word)                      │
│  Sigma.js (图谱可视化)│  LanceDB (向量存储)                   │
│  graphology (图计算) │  tiny_http (Web Clip服务)             │
│  TailwindCSS 4       │  sha2 (图片去重)                      │
│  i18next (国际化)     │  tokio (异步运行时)                   │
└──────────────────────┴──────────────────────────────────────┘
```

## 3. 顶层目录结构

```
llm_wiki/
├── src/                          # 前端源码
│   ├── App.tsx                   # 应用入口、项目生命周期
│   ├── main.tsx                  # React 挂载点
│   ├── stores/                   # Zustand 状态管理
│   │   ├── wiki-store.ts         # 项目/LLM/嵌入/多模态配置
│   │   ├── chat-store.ts         # 对话历史
│   │   ├── activity-store.ts     # 摄入队列进度
│   │   ├── review-store.ts       # 审查项
│   │   ├── research-store.ts     # Deep Research
│   │   └── update-store.ts       # 版本更新检测
│   ├── lib/                      # 核心业务逻辑
│   │   ├── ingest.ts             # Two-Step Ingest 引擎
│   │   ├── ingest-queue.ts       # 持久化摄入队列
│   │   ├── search.ts             # 混合搜索 (token + vector + RRF)
│   │   ├── llm-client.ts         # 多提供商流式聊天
│   │   ├── llm-providers.ts      # 提供商配置适配层
│   │   ├── graph-relevance.ts    # 4信号相关性模型
│   │   ├── context-budget.ts     # Token 预算分配
│   │   ├── deep-research.ts      # Web 研究 + 自动摄入
│   │   ├── embedding.ts          # RAG 向量嵌入管线
│   │   ├── text-chunker.ts       # Markdown 分块
│   │   ├── image-caption-pipeline.ts # VLM 图片描述
│   │   ├── sweep-reviews.ts      # 审查项自动清理
│   │   └── ...                   # 工具/辅助模块
│   ├── components/               # UI 组件
│   │   ├── layout/               # 布局：AppLayout、侧边栏、面板
│   │   ├── chat/                 # 聊天输入与消息渲染
│   │   ├── editor/               # Milkdown Wiki 编辑器
│   │   ├── graph/                # Sigma.js 知识图谱
│   │   ├── settings/             # LLM 提供商配置
│   │   ├── sources/              # 源文件管理
│   │   ├── search/               # 搜索视图
│   │   ├── review/               # 审查视图
│   │   └── project/              # 项目创建/欢迎
│   ├── commands/                 # Tauri 命令封装
│   │   └── fs.ts                 # 文件系统 IPC 桥接
│   ├── types/                    # TypeScript 类型定义
│   └── i18n/                     # 国际化资源
│
├── src-tauri/                    # Rust 后端
│   ├── src/
│   │   ├── lib.rs                # Tauri 入口、命令注册、窗口管理
│   │   ├── clip_server.rs        # tiny_http Web Clip 守护进程
│   │   ├── panic_guard.rs        # 第三方库 panic 边界
│   │   ├── commands/
│   │   │   ├── fs.rs             # 文件读写、文档解析
│   │   │   ├── vectorstore.rs    # LanceDB 向量操作
│   │   │   ├── project.rs        # 项目创建/打开
│   │   │   ├── extract_images.rs # 图片提取与保存
│   │   │   └── claude_cli.rs     # Claude CLI 子进程
│   │   └── types/
│   │       └── wiki.rs           # Rust 侧类型定义
│   ├── Cargo.toml                # Rust 依赖
│   └── tauri.conf.json           # Tauri 配置
│
├── extension/                    # Chrome 扩展 (Manifest V3)
└── package.json                  # Node 依赖与脚本
```

## 4. 核心架构模式

### 4.1 分层架构

```
┌────────────────────────────────────────────┐
│           UI Layer (React Components)       │  用户交互、视图渲染
├────────────────────────────────────────────┤
│        State Layer (Zustand Stores)         │  全局状态、跨组件通信
├────────────────────────────────────────────┤
│       Business Logic Layer (src/lib/)       │  核心算法、管道编排
├────────────────────────────────────────────┤
│      IPC Bridge (commands/fs + invoke)      │  前后端通信
├────────────────────────────────────────────┤
│     System Layer (Rust Commands + OS)       │  文件系统、文档解析、向量存储
└────────────────────────────────────────────┘
```

### 4.2 前后端关系：宿主-寄生架构

Tauri v2 下，前端运行在 Rust 进程内的 WebView 沙箱中，两者是**宿主-寄生**关系：

```
┌─────────────────────────────────────────────────┐
│  Tauri 进程 (Rust)                               │
│  ┌───────────────────────────────────────────┐  │
│  │  WebView (系统浏览器引擎)                   │  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │  前端 (src/) — React + TypeScript    │  │  │
│  │  │  运行在 WebView 沙箱内，             │  │  │
│  │  │  无文件系统/网络原生权限             │  │  │
│  │  └─────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────┘  │
│                                                  │
│  Rust 后端 (src-tauri/) — 原生系统访问            │
│  ├── 文件系统读写 (read_file / write_file)       │
│  ├── PDF/Office 文档解析 (pdfium / calamine)     │
│  ├── LanceDB 向量存储 (vector_upsert_chunks)     │
│  ├── HTTP 服务器 (tiny_http, 端口 19827)         │
│  └── 子进程管理 (Claude CLI)                     │
└─────────────────────────────────────────────────┘
```

**通信机制**：前端通过 `invoke()` 调用 Rust 注册的命令，本质是进程内 IPC：

```typescript
// 前端 src/commands/fs.ts
import { invoke } from "@tauri-apps/api/core"
export async function readFile(path: string): Promise<string> {
  return invoke("read_file", { path })  // → Rust #[tauri::command] read_file
}
```

```rust
// 后端 src-tauri/src/commands/fs.rs
#[tauri::command]
fn read_file(path: String) -> Result<String, String> {
    std::fs::read_to_string(&path).map_err(|e| e.to_string())
}
```

**职责分界**：

| 层面 | 前端 (src/) | 后端 (src-tauri/) |
|------|------------|-------------------|
| UI 渲染 | 全部负责 | 不参与 |
| 状态管理 | Zustand Stores | 无状态（命令式） |
| 业务逻辑 | 管道编排、搜索融合、图谱计算 | 文件 IO、文档解析、向量存储 |
| LLM 调用 | 组装 prompt + 解析流式响应 | 不参与（前端直接 HTTP） |
| 文件系统 | 只能通过 invoke 间接访问 | 直接访问，拥有完整权限 |
| 网络请求 | 经 `tauri-plugin-http` 代理（Rust 发出，绕 CORS） | tiny_http 本地服务器 |

**关键约束**：

1. **前端不能直接读写文件** — WebView 沙箱无文件系统权限，所有 IO 必须经 Rust 命令
2. **LLM 请求从前端发起但经 Rust 代理** — `tauri-plugin-http` 让请求从 Rust 进程发出，绕过浏览器 CORS/CSP 限制
3. **Rust 后端无业务状态** — 每次 invoke 是独立的，业务状态全在前端 Zustand 中维护
4. **Clip Server 是唯一独立运行的后端服务** — 监听 19827 端口，Chrome 扩展直接与它通信，不经过前端

### 4.4 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 状态管理 | Zustand (无 reducer) | 轻量、直接修改、无样板代码 |
| 编辑器 | Milkdown (ProseMirror) | Markdown 原生、支持数学公式 |
| 图谱 | graphology + sigma.js | 内存图计算 + GPU 加速渲染 |
| 向量存储 | LanceDB (嵌入式) | 零服务器依赖、Rust 原生绑定 |
| Web Clip | tiny_http 本地服务器 | Chrome 扩展 → HTTP → 主应用 |
| 文档解析 | pdfium + calamine + docx-rs | 全格式覆盖、Rust 原生性能 |
| 通信协议 | Tauri IPC (invoke) | 类型安全、零序列化开销 |

### 4.5 数据流核心

```
用户导入文档
     │
     ▼
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│ Ingest   │────▶│ LLM Analysis │────▶│ LLM Generate │
│ Queue    │     │ (Step 1)     │     │ (Step 2)     │
└──────────┘     └──────────────┘     └──────────────┘
     │                                       │
     │         ┌──────────────┐              │
     │         │ Image Extract│              │
     │         │ (Rust async) │              │
     │         └──────────────┘              │
     │                                       ▼
     │                              ┌──────────────┐
     │                              │ Parse FILE   │
     │                              │ Blocks       │
     │                              └──────────────┘
     │                                       │
     │              ┌────────────────────────┤
     │              ▼                        ▼
     │     ┌──────────────┐        ┌──────────────┐
     │     │ Write Wiki   │        │ Parse REVIEW │
     │     │ Pages        │        │ Blocks       │
     │     └──────────────┘        └──────────────┘
     │              │                        │
     │              ▼                        ▼
     │     ┌──────────────┐        ┌──────────────┐
     │     │ Embed Pages  │        │ Review Store │
     │     │ (Optional)   │        │              │
     │     └──────────────┘        └──────────────┘
     │              │
     ▼              ▼
┌──────────────────────────────────────────────┐
│              Wiki Knowledge Base              │
│  wiki/ (entities, concepts, sources, ...)    │
└──────────────────────────────────────────────┘
     │
     ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Token Search │  │ Vector Search│  │ Graph Expand │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       └──────────┬──────┘─────────────────┘
                  ▼
          ┌──────────────┐
          │  RRF Fusion  │
          └──────────────┘
                  │
                  ▼
          ┌──────────────┐
          │ Chat Context │
          │ Assembly     │
          └──────────────┘
```

## 5. 项目目录约定

每个 Wiki 项目遵循固定目录结构：

```
{project}/
├── schema.md               # Wiki 结构定义
├── purpose.md              # 项目目的说明
├── .obsidian/              # Obsidian 兼容配置
├── raw/
│   ├── sources/            # 原始源文档
│   └── assets/             # 媒体资源
├── wiki/
│   ├── index.md            # 页面索引
│   ├── log.md              # 研究日志
│   ├── overview.md         # 项目概述
│   ├── entities/           # 命名实体
│   ├── concepts/           # 概念/技术
│   ├── sources/            # 文档摘要
│   ├── queries/            # 研究问题
│   ├── comparisons/        # 对比分析
│   ├── synthesis/          # 交叉综述
│   └── media/              # 提取的图片
└── .llm-wiki/
    ├── ingest-queue.json   # 摄入队列持久化
    ├── ingest-cache.json   # 摄入缓存
    ├── image-caption-cache.json
    └── lancedb/            # 向量索引
```

## 6. 系列文档导航

| 序号 | 文档 | 重点 |
|------|------|------|
| 02 | [前端状态管理与数据流](02-state-management.md) | Zustand Stores、持久化、跨组件通信 |
| 03 | [Two-Step Ingest 核心逻辑](03-ingest-pipeline.md) | 分析→生成→写入→审查 完整管道 |
| 04 | [LLM 多提供商客户端](04-llm-client.md) | 7种提供商适配、流式解析、多模态 |
| 05 | [搜索管道与知识图谱](05-search-pipeline.md) | Token+Vector RRF融合、4信号相关性 |
| 06 | [Tauri 后端命令系统](06-tauri-backend.md) | Rust 命令、文档解析、向量存储 |
| 07 | [Chrome 扩展与本地通信](07-chrome-extension.md) | Web Clip、tiny_http、项目联动 |
| 08 | [Deep Research 深度研究](08-deep-research.md) | Web搜索→LLM综合→自动摄入 |
| 09 | [UI 组件体系与布局](09-ui-components.md) | 面板布局、编辑器、图谱可视化 |
| 10 | [安全设计与平台适配](10-security-platform.md) | 路径安全、CSP、跨平台行为 |

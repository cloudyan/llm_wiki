# LLM Wiki 源码解析 — UI 组件体系与布局

## 1. 布局架构

### 1.1 整体结构

```
┌─────────────────────────────────────────────────────────────┐
│                      AppLayout                               │
├──────────┬──────────────────────────────────────────────────┤
│          │                                                   │
│  Icon    │              ContentArea                          │
│  Sidebar │    ┌─────────────────────────────────────┐       │
│          │    │         react-resizable-panels       │       │
│  [Wiki]  │    ├──────────┬──────────────────────────┤       │
│  [Chat]  │    │  Sidebar │      Main Panel          │       │
│  [Graph] │    │  Panel   │                          │       │
│  [Src]   │    │          │                          │       │
│  [Set]   │    │  file    │  wiki editor / chat /    │       │
│          │    │  tree /  │  graph / settings /      │       │
│          │    │  search  │  sources / ...           │       │
│          │    │          │                          │       │
│          │    ├──────────┴──────────────────────────┤       │
│          │    │        Optional Bottom Panel         │       │
│          │    │  (preview / research / activity)     │       │
│          │    └─────────────────────────────────────┘       │
└──────────┴──────────────────────────────────────────────────┘
```

### 1.2 组件层次

```
App.tsx
├── WelcomeScreen          (无项目时)
├── CreateProjectDialog    (创建项目对话框)
└── AppLayout              (有项目时)
    ├── IconSidebar         (左侧图标导航栏)
    ├── ContentArea         (主内容区域)
    │   ├── SidebarPanel    (可调整大小的侧边面板)
    │   │   ├── FileTree    (文件树视图)
    │   │   ├── KnowledgeTree (知识分类树)
    │   │   ├── SearchView  (搜索面板)
    │   │   └── ChatBar     (聊天入口)
    │   ├── MainPanel       (主面板，根据 activeView 切换)
    │   │   ├── WikiEditor  (Milkdown 编辑器)
    │   │   ├── ChatPanel   (聊天面板)
    │   │   ├── GraphView   (Sigma.js 图谱)
    │   │   ├── SourcesView (源文件管理)
    │   │   ├── ReviewView  (审查项)
    │   │   ├── LintView    (Wiki 健康)
    │   │   └── SettingsView(设置)
    │   └── OptionalPanels  (可折叠的底部面板)
    │       ├── PreviewPanel(源文件预览)
    │       ├── ResearchPanel (Deep Research)
    │       └── ActivityPanel (摄入进度)
    └── UpdateBanner        (版本更新提示)
```

## 2. 核心组件详解

### 2.1 AppLayout

**文件**：[app-layout.tsx](../src/components/layout/app-layout.tsx)

主布局容器，使用 `react-resizable-panels` 实现可拖拽的面板分割：

- 左侧 `IconSidebar`：固定宽度的图标导航
- 中间 `SidebarPanel`：可调整宽度的侧边栏
- 右侧 `ContentArea`：主内容区域

### 2.2 WikiEditor — Milkdown 编辑器

**文件**：[wiki-editor.tsx](../src/components/editor/wiki-editor.tsx)

基于 Milkdown (ProseMirror) 的 Markdown 编辑器：

- **实时预览**：Markdown 原生渲染
- **数学公式**：KaTeX 支持 (`remark-math` + `rehype-katex`)
- **Wikilink**：`[[page-name]]` 语法高亮与导航
- **自动保存**：防抖保存到文件系统
- **主题**：Nord 主题

### 2.3 ChatPanel — 聊天面板

**文件**：[chat-panel.tsx](../src/components/chat/chat-panel.tsx)

双模式聊天界面：

- **Chat 模式**：自由对话，支持"Save to Wiki"
- **Ingest 模式**：展示源文档分析流，支持"Write to Wiki"

**上下文组装流程**：

```
chat-panel 组装 prompt:
     │
     ├── 读取 wiki/index.md → 摘要作为上下文
     │
     ├── 搜索相关页面:
     │   ├── searchWiki() → token + vector 结果
     │   ├── getRelatedNodes() → 图谱扩展
     │   └── computeContextBudget() → 预算分配
     │
     ├── 按 budget 填充:
     │   ├── indexBudget → 页面标题列表
     │   ├── pageBudget → 相关页面内容 (截断到 maxPageSize)
     │   └── responseReserve → 留空
     │
     └── 构建 system prompt + 历史 + 用户消息
```

### 2.4 GraphView — 知识图谱

**文件**：[graph-view.tsx](../src/components/graph/graph-view.tsx)

基于 sigma.js + graphology 的交互式知识图谱：

```
buildRetrievalGraph() → graphology Graph
     │
     ├── sigma.js 渲染:
     │   ├── 节点 = wiki 页面
     │   ├── 边 = wikilink / source overlap
     │   └── 节点大小 = degree
     │
     ├── 社区检测:
     │   └── graphology-communities-louvain → 节点着色
     │
     ├── 布局:
     │   └── ForceAtlas2 → 物理模拟布局
     │
     └── 交互:
         ├── 悬停 → 高亮相关节点
         ├── 点击 → 导航到 wiki 页面
         └── 拖拽 → 调整节点位置
```

**节点类型颜色映射**：
- entity → 蓝色
- concept → 绿色
- source → 橙色
- synthesis → 紫色
- query → 红色

### 2.5 SettingsView — 设置面板

**文件**：[settings-view.tsx](../src/components/settings/settings-view.tsx)

多区域设置界面：

| 区域 | 组件 | 功能 |
|------|------|------|
| LLM Provider | [llm-provider-section.tsx](../src/components/settings/sections/llm-provider-section.tsx) | 提供商选择、API Key、模型、预设 |
| Embedding | [embedding-section.tsx](../src/components/settings/sections/embedding-section.tsx) | 向量嵌入端点、模型、分块配置 |
| Multimodal | [multimodal-section.tsx](../src/components/settings/sections/multimodal-section.tsx) | 图片描述开关、VLM 配置 |
| Web Search | [web-search-section.tsx](../src/components/settings/sections/web-search-section.tsx) | Tavily API Key |
| Output | [output-section.tsx](../src/components/settings/sections/output-section.tsx) | 输出语言选择 |
| Interface | [interface-section.tsx](../src/components/settings/sections/interface-section.tsx) | 界面语言、主题 |
| About | [about-section.tsx](../src/components/settings/sections/about-section.tsx) | 版本信息、更新检查 |

### 2.6 预设系统

**文件**：[llm-presets.ts](../src/components/settings/llm-presets.ts)、[preset-resolver.ts](../src/components/settings/preset-resolver.ts)

```typescript
// 预设定义
const LLM_PRESETS = [
  { id: "openai-gpt4o", name: "GPT-4o", provider: "openai", model: "gpt-4o", ... },
  { id: "anthropic-sonnet", name: "Claude Sonnet", provider: "anthropic", model: "claude-sonnet-4-20250514", ... },
  { id: "ollama-local", name: "Ollama (Local)", provider: "ollama", ... },
  // ...
]

// 解析逻辑
resolveConfig(preset, userOverride, currentFallback)
  → 合并优先级: userOverride > preset defaults > currentFallback
```

## 3. 通用 UI 组件

基于 shadcn/ui (TailwindCSS + CVA)：

| 组件 | 文件 | 用途 |
|------|------|------|
| Button | [button.tsx](../src/components/ui/button.tsx) | 操作按钮 |
| Dialog | [dialog.tsx](../src/components/ui/dialog.tsx) | 模态对话框 |
| Input | [input.tsx](../src/components/ui/input.tsx) | 文本输入 |
| Resizable | [resizable.tsx](../src/components/ui/resizable.tsx) | 可调整大小的面板 |
| ScrollArea | [scroll-area.tsx](../src/components/ui/scroll-area.tsx) | 自定义滚动区域 |
| Tooltip | [tooltip.tsx](../src/components/ui/tooltip.tsx) | 工具提示 |

## 4. 视图切换逻辑

```typescript
// wiki-store 管理 activeView
type ActiveView = "wiki" | "chat" | "graph" | "sources" | "settings" | "search" | "review" | "lint"

// IconSidebar 触发切换
onClick={() => setActiveView("wiki")}

// MainPanel 根据 activeView 渲染对应组件
switch (activeView) {
  case "wiki": return <WikiEditor />
  case "chat": return <ChatPanel />
  case "graph": return <GraphView />
  // ...
}
```

## 5. 国际化 (i18n)

基于 i18next + react-i18next：

- **界面语言**：与 LLM 输出语言独立
- **支持语言**：中文、英文等
- **语言切换**：`i18n.changeLanguage(lang)` → 持久化到 `loadLanguage()`
- **LLM 输出语言**：由 `outputLanguage` 独立控制，影响 `buildLanguageDirective()` 生成的 prompt 指令

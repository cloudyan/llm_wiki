# LLM Wiki 源码解析 — Tauri 后端命令系统

> 核心文件：[lib.rs](../src-tauri/src/lib.rs)、[fs.rs](../src-tauri/src/commands/fs.rs)、[vectorstore.rs](../src-tauri/src/commands/vectorstore.rs)、[project.rs](../src-tauri/src/commands/project.rs)、[extract_images.rs](../src-tauri/src/commands/extract_images.rs)、[claude_cli.rs](../src-tauri/src/commands/claude_cli.rs)、[clip_server.rs](../src-tauri/src/clip_server.rs)

## 1. 应用入口 (lib.rs)

### 1.1 初始化流程

```rust
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_opener::init())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_store::Builder::new().build())
        .plugin(tauri_plugin_http::init())
        .setup(|app| {
            // 1. 设置 PDFium 资源目录
            // 2. 启动 Clip Server 守护线程
            // 3. 初始化 ClaudeCliState
            Ok(())
        })
        .register_uri_scheme_protocol(...)  // asset:// 协议
        .on_window_event(|window, event| {
            // 跨平台窗口关闭行为
        })
        .invoke_handler(tauri::generate_handler![
            cmd::fs::read_file,
            cmd::fs::write_file,
            cmd::fs::list_directory,
            // ... 所有命令
        ])
        .run(tauri::generate_context!())
}
```

### 1.2 跨平台窗口行为

| 平台 | 关闭行为 | 退出方式 |
|------|----------|----------|
| macOS | 隐藏窗口 | Cmd+Q |
| Windows/Linux | 弹出确认对话框 | 确认后退出 |
| macOS | Dock 点击 | 恢复隐藏的窗口 |

### 1.3 Panic Guard

`panic_guard.rs` 为第三方解析库提供 panic 捕获边界：

```rust
pub fn run_guarded<F, R>(label: &str, f: F) -> Result<R, String>
where F: FnOnce() -> R + UnwindSafe
{
    std::panic::catch_unwind(f)
        .map_err(|_| format!("{} panicked — malformed input?", label))
}
```

pdfium/calamine/docx-rs 在畸形输入上会 panic 而非返回 Error，没有这个守卫会直接崩溃 Tauri 进程。

## 2. 文件系统命令 (fs.rs)

### 2.1 命令清单

| 命令 | 功能 | 复杂度 |
|------|------|--------|
| `read_file(path)` | 读取文件内容，自动检测格式并提取文本 | ★★★★★ |
| `write_file(path, contents)` | 写入文件内容 | ★ |
| `list_directory(path)` | 返回递归文件树 | ★★ |
| `copy_file/copy_directory` | 复制操作 | ★ |
| `delete_file(path)` | 删除文件/目录 | ★ |
| `file_exists(path)` | 快速存在性检查 | ★ |
| `create_directory(path)` | 创建目录 | ★ |
| `preprocess_file(path)` | 预提取并缓存文档文本 | ★★★ |
| `find_related_wiki_pages(project, source)` | 查找引用源文件的 wiki 页面 | ★★ |
| `read_file_as_base64(path)` | 读取文件为 base64 + MIME 类型 | ★★ |

### 2.2 read_file — 文档解析核心

这是后端最复杂的命令，实现了多格式文档的自动文本提取：

```
read_file(path)
     │
     ├── 判断文件类型
     │
     ├── .pdf → pdfium 全文提取
     │   ├── 全局 PDFIUM 单例 (OnceLock)
     │   ├── PDFIUM_LOCK Mutex 串行化
     │   └── extract_pdf_markdown() → 每页文本 + 行内图片标记
     │
     ├── .xlsx/.xls/.ods → calamine 提取
     │   └── 每个 Sheet → 表格文本
     │
     ├── .docx → docx-rs 或 ZIP 回退
     │   └── 段落文本 + 行内图片标记
     │
     ├── .pptx → ZIP 解析
     │   └── 每张 Slide → 文本 + 图片标记
     │
     ├── .md/.txt → 直接读取
     │
     └── 其他 → 尝试 UTF-8 读取
```

### 2.3 PDF 提取详解

**PDFium 配置**：
- 全局单例 `PDFIUM: OnceLock<Pdfium>` + `PDFIUM_LOCK: Mutex`
- 平台特定库路径：
  - Windows: `pdfium.dll`、`libpdfium.dll`、`pdfium/` 子目录
  - macOS: Frameworks/Resources 中的 `libpdfium.dylib`
  - Linux: `libpdfium.so`
- 所有 PDF 操作串行化（pdfium 非线程安全）

**extract_pdf_markdown()** — 统一文本+图片提取：
```rust
fn extract_pdf_markdown(pdfium: &Pdfium, path: &str) -> Result<String> {
    // 1. 打开 PDF
    // 2. 遍历每页:
    //    a. 提取页面文本
    //    b. 提取页面图片 → 保存到 wiki/media/<slug>/page-{n}-{idx}.png
    //    c. 在文本中插入 ![](abs_path) 标记
    // 3. 组合为 Markdown 输出
}
```

### 2.4 缓存机制

提取结果缓存为源文件旁边的 `.cache/<filename>.txt`：

```
raw/sources/paper.pdf
.cache/paper.pdf.txt   ← 提取的文本缓存
```

缓存失效：源文件修改时间变化时重新提取。

## 3. 向量存储 (vectorstore.rs)

### 3.1 LanceDB 集成

数据库路径：`{project}/.llm-wiki/lancedb`

### 3.2 Schema V1 (Legacy)

```rust
Schema V1:
  page_id: Utf8
  vector: FixedSizeList<Float32, dim>
```

### 3.3 Schema V2 (Current) — Chunk 级向量

```rust
Schema V2:
  chunk_id: Utf8       // "${page_id}#${chunk_index}"
  page_id: Utf8
  chunk_index: UInt32
  chunk_text: Utf8     // 原始 chunk 文本
  heading_path: Utf8   // 面包屑 "## A > ### B"
  vector: FixedSizeList<Float32, dim>
```

### 3.4 安全验证

`page_id` 只允许 `[a-zA-Z0-9_-]`，防止注入式删除过滤器。

### 3.5 V1 → V2 迁移

```rust
vector_legacy_row_count()  // 统计 V1 行数（供迁移 UI 显示）
vector_drop_legacy()       // 迁移完成后删除 V1 表
```

## 4. 项目管理 (project.rs)

### 4.1 create_project

```
create_project(name, path)
     │
     ├── 创建目录结构:
     │   ├── schema.md, purpose.md
     │   ├── .obsidian/ (兼容配置)
     │   ├── raw/sources/, raw/assets/
     │   └── wiki/ (index, log, overview, entities, concepts, ...)
     │
     └── 返回 WikiProject { name, path }
```

### 4.2 open_project

```
open_project(path)
     │
     ├── 验证路径存在
     ├── 验证是目录
     └── 返回 WikiProject { name: 目录名, path }
```

## 5. 图片提取 (extract_images.rs)

### 5.1 数据结构

```rust
struct ExtractedImage {
    index: u32,           // 文档内顺序
    mime_type: String,    // image/png, image/jpeg
    page: Option<u32>,    // PDF 页码 / PPTX 幻灯片号
    width: u32,
    height: u32,
    data_base64: String,  // 图片数据
    sha256: String,       // 去重哈希
}

struct SavedImage {
    // ... 同上，加上：
    rel_path: String,     // 相对于 wiki 根
    abs_path: String,     // 绝对路径
}
```

### 5.2 过滤阈值

- 最小尺寸：100×100 像素
- 最大数量：500 张/文档
- 格式：PNG、JPEG

### 5.3 Office 格式图片提取

```
PPTX: .rels 文件 → slide-to-media 映射 → 提取每页图片
DOCX: 无页码信息 (page = None)
XLSX: 媒体目录提取
```

## 6. Claude CLI 集成 (claude_cli.rs)

### 6.1 子进程管理

```rust
struct ClaudeCliState {
    processes: Arc<Mutex<HashMap<String, Child>>>,  // stream_id → Child
}
```

### 6.2 生命周期

```
claude_cli_spawn(stream_id, model, messages)
     │
     ├── 构建命令:
     │   claude -p --output-format stream-json
     │         --input-format stream-json
     │         --verbose --model {model}
     │
     ├── stdin 写入 JSON 消息流
     │
     ├── tokio::spawn 读取 stdout:
     │   └── 每行 → Tauri Event "claude-cli:{stream_id}"
     │
     └── 进程退出:
         └── Tauri Event "claude-cli:{stream_id}:done"
             (含 exit_code + stderr)
```

### 6.3 错误处理

- macOS Gatekeeper 隔离：检测 `LGQuarantine` 错误，提示用户移除隔离属性
- 版本检测：3 秒超时
- 进程清理：`claude_cli_kill()` 通过 SIGTERM/SIGKILL 终止

## 7. Web Clip Server (clip_server.rs)

### 7.1 架构

本地 HTTP 守护进程，基于 `tiny_http`：

```
Chrome 扩展 ──HTTP──▶ localhost:19827 ──▶ Clip Server ──▶ 项目文件系统
```

### 7.2 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/status` | 服务器状态 |
| GET | `/project` | 当前活跃项目 |
| POST | `/project` | 设置当前项目 |
| GET | `/projects` | 所有已知项目 |
| POST | `/projects` | 更新项目列表 |
| GET | `/clips/pending` | 待处理剪辑 |
| POST | `/clip` | 创建新剪辑 |

### 7.3 剪辑文件格式

```markdown
---
type: clip
title: "{title}"
url: "{url}"
clipped: {date}
origin: web-clip
sources: []
tags: [web-clip]
---

# {title}

Source: {url}

{content}
```

### 7.4 容错设计

- 端口冲突时最多重试 3 次绑定（间隔 2 秒）
- 崩溃后最多自动重启 10 次（间隔 5 秒）
- 状态机：starting → running / port_conflict / error

## 8. 类型系统 (types/wiki.rs)

```rust
#[derive(Serialize, Deserialize, Clone)]
struct WikiProject {
    name: String,
    path: String,  // 始终正斜杠规范化
}

#[derive(Serialize, Deserialize, Clone)]
struct FileNode {
    name: String,
    path: String,           // 始终正斜杠规范化
    is_dir: bool,
    children: Option<Vec<FileNode>>,
}
```

路径规范化确保 Windows 反斜杠不会泄漏到前端。

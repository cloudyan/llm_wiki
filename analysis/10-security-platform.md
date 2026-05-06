# LLM Wiki 源码解析 — 安全设计与平台适配

## 1. 安全设计

### 1.1 路径安全：isSafeIngestPath()

**威胁模型**：LLM 生成的文件路径不可信。攻击者可在源文档中植入 prompt injection，诱导 LLM 输出：

```
---FILE: ../../../etc/passwd---
malicious content
---END FILE---
```

**防护实现**（[ingest.ts](../src/lib/ingest.ts)）：

```typescript
function isSafeIngestPath(p: string): boolean {
  if (typeof p !== "string" || p.trim().length === 0) return false
  if (/[\x00-\x1f]/.test(p)) return false           // NUL/控制字符
  if (p.startsWith("/") || p.startsWith("\\")) return false  // 绝对路径
  if (/^[a-zA-Z]:/.test(p)) return false             // Windows 盘符
  const normalized = p.replace(/\\/g, "/")
  if (normalized.split("/").some(seg => seg === "..")) return false  // 路径遍历
  if (!normalized.startsWith("wiki/")) return false   // 必须在 wiki/ 下
  return true
}
```

**纵深防御**：即使 `isSafeIngestPath()` 被绕过，Tauri 的文件系统权限仍然限制在应用沙箱内。但 `fs.rs::write_file` 是通用命令，不做路径沙箱，所以解析层的守卫是关键防线。

### 1.2 CSP (Content Security Policy)

**配置**：[tauri.conf.json](../src-tauri/tauri.conf.json)

```json
{
  "security": {
    "csp": "default-src 'self'; connect-src 'self' https: http://localhost:* http://127.0.0.1:*; img-src 'self' https: asset: data:; style-src 'self' 'unsafe-inline'; media-src 'self' https: asset: data:"
  }
}
```

**关键限制**：
- `connect-src`：仅允许 self、https、localhost、127.0.0.1（LLM 端点必需）
- `img-src`：允许 asset 协议和 data URL（图片预览必需）
- `style-src`：`unsafe-inline`（TailwindCSS 运行时必需）
- `media-src`：允许 asset 和 data（音频/视频预览）

### 1.3 API Key 存储

- API Key 通过 `@tauri-apps/plugin-store` 存储在本地文件系统
- 存储路径：Tauri 应用的 app data 目录
- **未加密**：桌面应用的常见做法，依赖 OS 文件权限保护
- Key 不出现在日志或 LLM 输出中

### 1.4 Tauri 权限模型

**Capabilities**：[default.json](../src-tauri/capabilities/default.json)

```json
{
  "permissions": [
    "core:default",
    "opener:default",
    "dialog:default",
    "store:default",
    "http:default"       // 完整 HTTP/HTTPS 访问
  ]
}
```

**http:default** 允许访问所有 URL，这是 LLM 端点多样性的必要妥协。

### 1.5 图片提取安全

- SHA-256 去重防止缓存投毒
- 最小尺寸过滤 (100×100) 防止 1px tracking pixel
- 数量上限 (500/文档) 防止资源耗尽
- `panic_guard.rs` 防止畸形 PDF/Office 崩溃进程

### 1.6 LanceDB page_id 验证

```rust
// vectorstore.rs — 防止注入式删除
fn validate_page_id(page_id: &str) -> Result<()> {
    if !page_id.chars().all(|c| c.is_alphanumeric() || c == '-' || c == '_' || c == '.') {
        return Err("Invalid page_id");
    }
    Ok(())
}
```

### 1.7 Clip Server 安全

- 绑定 `0.0.0.0:19827`（LAN 可访问）— 这是设计选择，允许同一网络的设备剪辑
- 无认证（桌面应用的常见做法）
- 剪辑内容不直接执行脚本（Markdown 格式化）

## 2. 平台适配

### 2.1 窗口管理

| 平台 | 关闭行为 | 退出 | 恢复 |
|------|----------|------|------|
| macOS | 隐藏窗口 (Cmd+W) | Cmd+Q | Dock 点击恢复 |
| Windows | 确认对话框 | 对话框确认 | N/A |
| Linux | 确认对话框 | 对话框确认 | N/A |

**实现**（[lib.rs](../src-tauri/src/lib.rs)）：

```rust
.on_window_event(|window, event| {
    match event {
        WindowEvent::CloseRequested { api, .. } => {
            #[cfg(target_os = "macos")]
            {
                window.hide().unwrap();
                api.prevent_close();
            }
            #[cfg(not(target_os = "macos"))]
            {
                // 弹出确认对话框
            }
        }
        // macOS: Dock 点击恢复
        WindowEvent::Destroyed => { /* cleanup */ }
    }
})
```

### 2.2 PDFium 库路径

不同平台的 PDFium 动态库位置不同：

```rust
fn find_pdfium_lib() -> Option<PathBuf> {
    #[cfg(target_os = "windows")]
    { /* 搜索 pdfium.dll, libpdfium.dll, pdfium/ 子目录 */ }

    #[cfg(target_os = "macos")]
    { /* Frameworks/Resources 中的 libpdfium.dylib */ }

    #[cfg(target_os = "linux")]
    { /* libpdfium.so */ }
}
```

### 2.3 路径规范

**Rust 侧**：所有路径统一为正斜杠：

```rust
fn normalize_path(path: &str) -> String {
    path.replace('\\', "/")
}
```

**TypeScript 侧**：`path-utils.ts` 提供跨平台路径工具：

```typescript
function normalizePath(p: string): string {
    return p.replace(/\\/g, "/")
}
```

这确保 Windows 反斜杠不会泄漏到 JSON 序列化、Tauri IPC 调用、或文件树显示中。

### 2.4 HTTP 请求差异

| 平台 | WebView 引擎 | Origin 头 | CORS 行为 |
|------|-------------|----------|-----------|
| macOS | WebKit | `tauri://localhost` | 严格 |
| Windows | WebView2 (Chromium) | `http://tauri.localhost` | 标准 |
| Linux | WebKitGTK | `tauri://localhost` | 严格 |

**Ollama 兼容**：`localLlmOriginHeader()` 统一设置 `Origin: http://localhost`，绕过所有平台的 CORS 差异。

### 2.5 Obsidian 兼容

项目创建时自动生成 `.obsidian/` 配置：

```json
// .obsidian/app.json
{ "attachmentFolderPath": "raw/assets" }

// .obsidian/appearance.json
{ "baseFontSize": 16, "cssTheme": "" }

// .obsidian/core-plugins.json
{ "file-explorer": true, "graph-view": true, ... }
```

这让用户可以用 Obsidian 打开同一个项目目录，获得基本的兼容体验。

## 3. 资源管理

### 3.1 Asset Protocol

Tauri 的 `asset://` 协议允许 WebView 加载本地文件：

```rust
.register_uri_scheme_protocol("asset", |ctx, request| {
    // 将本地文件路径转换为 WebView 可访问的 URL
})
```

配置中启用了 `"scope": ["**"]`（全目录访问），这是文件管理器类应用的必要设置。

### 3.2 内存管理

- **图谱缓存**：`buildRetrievalGraph()` 结果缓存，通过 `dataVersion` 失效
- **搜索并发**：`SEARCH_READ_CONCURRENCY = 16` 限制 IPC 并发
- **流式处理**：LLM 响应逐行解析，不累积完整响应到内存（除 `accumulated` 字符串）
- **PDFium 互斥**：全局 `PDFIUM_LOCK` 串行化，防止多线程同时使用非线程安全的 pdfium

## 4. 错误处理策略

| 层级 | 策略 | 示例 |
|------|------|------|
| Rust 命令 | `Result<T, String>` + `panic_guard` | PDF 解析 panic → 用户友好错误 |
| Tauri IPC | 前端 try-catch | 文件不存在 → 静默返回 "" |
| 业务逻辑 | 非致命错误静默，致命错误传播 | 嵌入失败 → 跳过，不阻断摄入 |
| UI | Error Boundary | 组件崩溃 → 降级显示 |
| 网络 | 分类错误消息 | DNS 失败 vs 超时 vs 403 |

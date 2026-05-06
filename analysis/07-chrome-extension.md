# LLM Wiki 源码解析 — Chrome 扩展与本地通信

> 核心文件：[extension/](../extension/)、[clip_server.rs](../src-tauri/src/clip_server.rs)、[clip-watcher.ts](../src/lib/clip-watcher.ts)

## 1. 架构概览

Chrome 扩展通过本地 HTTP 服务器与主应用通信，实现 Web 内容剪辑：

```
┌──────────────┐    HTTP POST     ┌──────────────┐    文件写入     ┌──────────────┐
│  Chrome 扩展  │ ──────────────▶ │  Clip Server  │ ────────────▶ │  项目文件系统  │
│  (Manifest V3)│ ◀────────────── │  (tiny_http)  │               │              │
└──────────────┘    HTTP GET      │  Port: 19827  │               └──────────────┘
                                  └──────────────┘
                                        ▲
                                        │ 项目状态同步
                                        │
                                  ┌──────────────┐
                                  │  前端 App     │
                                  │  (clip-watcher)│
                                  └──────────────┘
```

## 2. 通信协议

### 2.1 请求方向：Chrome → 主应用

| 端点 | 方法 | 用途 |
|------|------|------|
| `/status` | GET | 检测主应用是否运行 |
| `/project` | GET | 获取当前活跃项目 |
| `/projects` | GET | 获取所有已知项目（用于项目选择器） |
| `/clip` | POST | 提交网页剪辑 |

### 2.2 请求方向：主应用 → Clip Server

| 端点 | 方法 | 用途 |
|------|------|------|
| `/project` | POST | 项目切换时通知服务器 |
| `/projects` | POST | 更新项目列表（Chrome 扩展的项目选择器需要） |

### 2.3 请求方向：主应用前端 → Clip Server

| 端点 | 方法 | 用途 |
|------|------|------|
| `/clips/pending` | GET | 轮询待处理的剪辑 |

## 3. Chrome 扩展结构

### 3.1 Manifest V3 配置

```json
{
  "manifest_version": 3,
  "name": "LLM Wiki Clipper",
  "permissions": ["activeTab", "scripting"],
  "action": { "default_popup": "popup.html" },
  "background": { "service_worker": "background.js" },
  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["content.js"]
  }]
}
```

### 3.2 扩展组件

| 组件 | 职责 |
|------|------|
| `popup.html/js` | 弹出界面：项目选择、剪辑预览、提交 |
| `background.js` | Service Worker：消息路由、状态管理 |
| `content.js` | 内容脚本：页面内容提取、Markdown 转换 |

### 3.3 剪辑流程

```
用户点击扩展图标
     │
     ▼
[popup.js] 显示弹出界面
     │
     ├── GET /status → 检测主应用状态
     ├── GET /project → 获取当前项目
     ├── GET /projects → 获取项目列表
     │
     ├── 用户选择项目 (POST /project)
     │
     ├── content.js 提取页面内容:
     │   ├── 标题
     │   ├── 正文 (Readability 或自定义提取)
     │   ├── URL
     │   └── 转换为 Markdown
     │
     └── POST /clip → 提交剪辑
         body: { project, title, url, content }
```

## 4. Clip Server 详解

### 4.1 服务器生命周期

```
应用启动 → start_clip_server()
     │
     ├── 绑定 0.0.0.0:19827
     │   ├── 成功 → status = running
     │   ├── 端口冲突 → 重试 3 次 (间隔 2s)
     │   └── 全部失败 → status = port_conflict
     │
     ├── 请求循环
     │   └── 每个请求 → 新线程处理
     │
     └── 崩溃恢复
         └── 自动重启 (最多 10 次, 间隔 5s)
```

### 4.2 剪辑处理

```rust
POST /clip
     │
     ├── 解析 JSON body: { project, title, url, content }
     │
     ├── 确定目标项目路径
     │
     ├── 生成文件名: clip-{slug}-{date}.md
     │
     ├── 写入 raw/sources/ 目录:
     │   ---\n
     │   type: clip\n
     │   title: "{title}"\n
     │   url: "{url}"\n
     │   clipped: {date}\n
     │   origin: web-clip\n
     │   sources: []\n
     │   tags: [web-clip]\n
     │   ---\n
     │   \n
     │   # {title}\n
     │   \n
     │   Source: {url}\n
     │   \n
     │   {content}
     │
     └── 标记为 "pending" → 等待自动摄入
```

### 4.3 状态同步

当用户在主应用中切换项目时：

```
App.tsx → handleProjectOpened()
     │
     ├── POST /project { path: proj.path }
     │
     └── POST /projects { projects: [...] }
         └── 所有最近项目列表
```

这让 Chrome 扩展的项目选择器始终保持与主应用同步。

## 5. 前端 Clip Watcher

### 5.1 轮询机制

`clip-watcher.ts` 在应用启动时调用 `startClipWatcher()`：

```
startClipWatcher()
     │
     ├── 定期 GET /clips/pending
     │
     └── 发现新剪辑 → 自动摄入
         └── enqueueIngest(projectId, sourcePath)
```

### 5.2 自动摄入

剪辑文件写入 `raw/sources/` 后，与手动导入的文件走相同的摄入管线。Clip 类型文件有特殊的 frontmatter（`origin: web-clip`），LLM 在分析时会参考这些元数据。

## 6. 安全考量

- **CORS**：Clip Server 不设置 CORS 头，Chrome 扩展通过 `background.js`（Service Worker 不受 CORS 限制）代理请求
- **认证**：无认证机制（本地服务，仅绑定 localhost）
- **端口固定**：19827 端口在代码中硬编码，主应用和扩展必须一致
- **内容过滤**：剪辑内容经过 Markdown 转换，去除脚本标签等危险内容

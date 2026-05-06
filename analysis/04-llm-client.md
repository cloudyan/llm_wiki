# LLM Wiki 源码解析 — LLM 多提供商客户端

> 核心文件：[llm-client.ts](../src/lib/llm-client.ts)、[llm-providers.ts](../src/lib/llm-providers.ts)、[claude-cli-transport.ts](../src/lib/claude-cli-transport.ts)

## 1. 架构概览

LLM Wiki 的 LLM 客户端采用**提供商适配器模式**，统一 7 种不同 LLM 后端的调用方式：

```
                 streamChat(config, messages, callbacks, signal, overrides)
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                                     ▼
         getProviderConfig(config)              streamViaClaudeCodeCli()
                    │                                     │
        ┌──────────┼─────────────────────────┐          子进程传输
        ▼         ▼         ▼         ▼       ▼         (stdin/stdout)
    OpenAI    Anthropic   Google   Ollama   Custom
    协议      协议       协议     OpenAI   双模式
                                   兼容     (OpenAI/Anthropic)
```

## 2. 提供商配置适配

### 2.1 ProviderConfig 接口

```typescript
interface ProviderConfig {
  url: string                                          // POST 目标 URL
  headers: Record<string, string>                      // 认证 + Content-Type
  buildBody: (messages: ChatMessage[], overrides?: RequestOverrides) => unknown
  parseStream: (line: string) => string | null         // SSE 行解析
}
```

每个提供商实现这 4 个字段，`streamChat()` 统一使用这个接口。

### 2.2 七种提供商

| 提供商 | 线协议 | URL | 认证 | 流式解析 |
|--------|--------|-----|------|----------|
| **OpenAI** | Chat Completions | `api.openai.com/v1/chat/completions` | `Bearer {key}` | `parseOpenAiLine` |
| **Anthropic** | Messages | `api.anthropic.com/v1/messages` | `x-api-key` + `anthropic-version` | `parseAnthropicLine` |
| **Google** | Gemini | `generativelanguage.googleapis.com/...streamGenerateContent?alt=sse` | `x-goog-api-key` | `parseGoogleLine` |
| **Ollama** | OpenAI 兼容 | `{ollamaUrl}/v1/chat/completions` | 无 / `Origin: http://localhost` | `parseOpenAiLine` |
| **Custom** | 双模式 | 用户自定义 | 可选 Bearer | 根据 apiMode 选择 |
| **MiniMax** | Anthropic 兼容 | `api.minimax.io/anthropic/v1/messages` | `Bearer {key}` | `parseAnthropicLine` |
| **Claude Code** | 子进程 | N/A | N/A | Tauri 事件流 |

### 2.3 关键适配细节

#### OpenAI 家族（覆盖最广）

`buildOpenAiBody()` 生成标准 Chat Completions 请求体，所有 OpenAI 兼容端点（DeepSeek、Groq、Zhipu、Kimi、xAI）共用。

**多模态翻译**：
```typescript
// ChatMessage.content 支持 string | ContentBlock[]
type ContentBlock =
  | { type: "text"; text: string }
  | { type: "image"; mediaType: string; dataBase64: string }

// OpenAI: image → { type: "image_url", image_url: { url: "data:..." } }
// Anthropic: image → { type: "image", source: { type: "base64", media_type, data } }
// Google: image → { inline_data: { mime_type, data } }
```

纯文本消息保持 `content: string`（向后兼容），只有实际包含图片时才切换到数组形式。

#### Anthropic 家族

- System 消息提取到顶层 `system` 字段（Anthropic 不支持 system 角色在 messages 中）
- `max_tokens` 是必填字段，默认 4096
- `stop` → `stop_sequences`，`top_p`/`top_k` 保持 snake_case
- URL 构建：`buildAnthropicUrl()` 智能处理各种用户输入格式（避免 `/v1/v1/messages`）

**Bearer 认证**：MiniMax 和阿里云百炼的 Anthropic 网关使用 `Authorization: Bearer` 而非原生 `x-api-key`，`requiresBearerAuth()` 检测这些端点。

#### Google Gemini

- 采样参数必须嵌套在 `generationConfig` 下且使用 camelCase：`topP`、`topK`、`maxOutputTokens`、`stopSequences`
- System 消息提取到 `systemInstruction.parts`
- 角色映射：`assistant` → `model`
- 思维链过滤：`thought: true` 的 part 被跳过，不泄漏推理文本

#### Ollama / 本地 LLM

**Origin 头策略**是此模块最复杂的适配逻辑：

```typescript
function localLlmOriginHeader(): Record<string, string> {
  return { Origin: "http://localhost" }
}
```

**问题链**：
1. Tauri WebKit 在 Windows 上发送 `Origin: http://tauri.localhost`
2. Ollama 的 `OLLAMA_ORIGINS` 默认不接受该 origin → 403
3. 不能发送真实 origin（如 `http://192.168.0.20:11434`），因为 LAN 服务器不在 Ollama 的默认允许列表
4. `http://localhost` 是 Ollama 无条件接受的 origin

**Qwen3 思维模式**：`/qwen[-_]?3/i.test(model)` 时注入 `chat_template_kwargs: { enable_thinking: false }`，配合 llama.cpp 的 `--jinja` 模式。

## 3. 流式聊天引擎 (streamChat)

### 3.1 核心流程

```
streamChat(config, messages, callbacks, signal, overrides)
     │
     ├── [claude-code] → 子进程传输，不走 HTTP
     │
     ├── getProviderConfig(config) → 构建请求
     │
     ├── 创建超时 AbortController (30分钟)
     │   └── 合并用户取消信号
     │
     ├── httpFetch(url, { method: "POST", headers, body, signal })
     │   └── 错误分类：
     │       ├── signal.aborted → onDone() (用户取消)
     │       ├── timeoutFired → onError("timed out")
     │       ├── isFetchNetworkError → onError("Network error...")
     │       └── !response.ok → onError("HTTP 4xx/5xx")
     │
     └── 读取 SSE 流
         ├── parseLines(chunk, buffer) → 按行切分
         └── 逐行 parseStream() → onToken() / 跳过空行
```

### 3.2 超时与取消

- **30 分钟超时**：应对大上下文推理模型的长时间响应
- **双层 AbortController**：用户取消 + 超时取消，通过 `timeoutFired` 标志区分
- **流中断处理**：`reader.read()` 失败时，区分用户取消（onDone）和网络断开（onError）

### 3.3 SSE 行解析

三种流式协议都使用 SSE 格式（`data: ...`），但 JSON 结构不同：

```javascript
// OpenAI: data: {"choices":[{"delta":{"content":"token"}}]}
// Anthropic: data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"token"}}
// Google: data: {"candidates":[{"content":{"parts":[{"text":"token"}]}}]}
```

## 4. Claude Code CLI 子进程传输

当 `provider === "claude-code"` 时，走完全不同的传输层：

```
[前端] → claude_cli_spawn(stream_id, model, messages)
              │
              ▼ [Rust]
         tokio::process::Command("claude")
           args: -p --output-format stream-json --input-format stream-json --verbose --model {model}
              │
              ▼ stdout → 逐行读取
         Tauri Event: "claude-cli:{stream_id}" → 前端
              │
              ▼ 进程退出
         Tauri Event: "claude-cli:{stream_id}:done" → 前端
```

- 使用 `stdin` 传入 JSON 消息流
- `stdout` 通过 Tauri 事件系统实时推送到前端
- `ClaudeCliState` 用 `Arc<Mutex<HashMap<String, Child>>>` 管理子进程
- 支持 `claude_cli_kill(stream_id)` 终止运行中的子进程
- macOS Gatekeeper 隔离错误有专门的修复提示

## 5. 请求覆盖 (RequestOverrides)

统一采样参数接口，各提供商的 `buildBody()` 负责翻译：

```typescript
interface RequestOverrides {
  temperature?: number    // OpenAI: temperature | Gemini: generationConfig.temperature
  top_p?: number         // OpenAI: top_p | Anthropic: top_p | Gemini: generationConfig.topP
  top_k?: number         // Anthropic: top_k | Gemini: generationConfig.topK
  max_tokens?: number    // OpenAI: max_tokens | Anthropic: max_tokens | Gemini: maxOutputTokens
  stop?: string | string[] // OpenAI: stop | Anthropic: stop_sequences | Gemini: stopSequences
}
```

## 6. Tauri HTTP 插件

所有 HTTP 请求通过 `@tauri-apps/plugin-http` 发出（`tauri-fetch.ts` 封装），绕过 WebKit CSP 和 CORS 限制：

- Rust 侧启用 `unsafe-headers` feature 允许 Origin 头透传
- 请求从 Tauri Rust 进程发出，而非 WebView，避免浏览器 CORS 拦截
- `isFetchNetworkError()` 检测 WebKit/Chromium 的网络错误（不透明错误信息）

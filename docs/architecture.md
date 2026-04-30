# Claude Code Proxy 架构文档

## 1. 项目概述

Claude Code Proxy 是一个 API 协议转换代理，将 Claude API（`/v1/messages`）的请求格式转换为 OpenAI Chat Completions API 格式，转发给任意 OpenAI 兼容的 LLM 提供商（OpenAI、Azure OpenAI、GLM、DeepSeek、Ollama 等），再将响应转回 Claude API 格式，使 Claude Code CLI 能通过代理使用非 Anthropic 的模型。

## 2. 整体架构

```
Claude Code CLI
      │
      │ Claude API 格式 (HTTP/SSE)
      ▼
┌─────────────────────────────────────────────┐
│              FastAPI Server (:8082)          │
│                                             │
│  ┌─────────────┐    ┌──────────────────┐    │
│  │  endpoints   │───▶│ request_converter │   │
│  │  (路由层)    │    │ (Claude→OpenAI)   │    │
│  └──────┬──────┘    └──────────────────┘    │
│         │                                   │
│         │           ┌──────────────────┐    │
│         │           │  model_manager    │    │
│         │           │  (模型映射)       │    │
│         │           └──────────────────┘    │
│         │                                   │
│         ▼                                   │
│  ┌─────────────┐                            │
│  │   client     │  OpenAI SDK               │
│  │  (上游通信)  │─────────────────┐          │
│  └──────┬──────┘                 │          │
│         │                        │          │
│         ▼                        ▼          │
│  ┌──────────────┐    ┌─────────────────┐    │
│  │response_conv  │    │  OpenAI API /   │    │
│  │(OpenAI→Claude)│◀───│  GLM / DeepSeek │    │
│  └──────────────┘    │  / Ollama / ...  │    │
│                       └─────────────────┘    │
│  ┌──────────────┐                           │
│  │   config      │  环境变量 / .env          │
│  └──────────────┘                           │
└─────────────────────────────────────────────┘
```

## 3. 启动流程

```
start_proxy.py → src/main.py:main()
    │
    ├─ Config() 加载环境变量（.env 文件自动加载 by python-dotenv）
    │   └─ 校验 OPENAI_API_KEY 必填
    │
    ├─ logging.basicConfig() 配置日志级别
    │
    ├─ 注册全局异常处理器 claude_error_handler
    │
    ├─ 加载 APIRouter（endpoints.py）
    │   └─ 实例化 OpenAIClient（AsyncOpenAI / AsyncAzureOpenAI）
    │
    └─ uvicorn.run() 启动 HTTP 服务
```

## 4. 请求处理全流程

### 4.1 非流式请求

```
Claude Code ──POST /v1/messages──▶ validate_api_key()
       │                              │
       │                              ▼
       │                        convert_claude_to_openai()
       │                         ├─ 模型映射 (model_manager)
       │                         ├─ system → system message
       │                         ├─ user/assistant messages 转换
       │                         ├─ tool_use → function_call
       │                         ├─ tool_result → tool message
       │                         ├─ max_tokens 限制 (min~max)
       │                         └─ thinking.enabled → max_tokens≥16000
       │
       │                        OpenAIClient.create_chat_completion()
       │                         └─ openai SDK → 上游 API
       │
       │                        convert_openai_to_claude_response()
       │                         ├─ reasoning_content → thinking block
       │                         ├─ content → text block
       │                         ├─ tool_calls → tool_use blocks
       │                         ├─ finish_reason → stop_reason
       │                         └─ usage 映射 (4 字段)
       │
       ◀─────── Claude API JSON 响�回───────
```

### 4.2 流式请求

```
Claude Code ──POST /v1/messages?stream=true──▶
       │
       │  StreamingResponse(SSE)
       │
       │  convert_openai_streaming_to_claude_with_cancellation()
       │   ├─ yield: message_start
       │   ├─ yield: ping
       │   │
       │   │  async for line in openai_stream:
       │   │   ├─ delta.reasoning_content → yield: thinking_delta
       │   │   ├─ delta.content → yield: text_delta
       │   │   ├─ delta.tool_calls → yield: input_json_delta
       │   │   └─ usage → 累积到 usage_data
       │   │
       │   │  每 chunk 检查 http_request.is_disconnected()
       │   │
       │   ├─ yield: content_block_stop (关闭所有 block)
       │   ├─ yield: message_delta (含 stop_reason + usage)
       │   └─ yield: message_stop
       │
       ◀──────── SSE 事件流 ────────────────
```

## 5. 各模块详细说明

### 5.1 入口与服务器

**`src/main.py`** — 应用入口

- 创建 FastAPI 实例
- 注册全局 `@app.exception_handler(Exception)`：将所有未捕获异常转为 Claude API 格式 `{"type":"error","error":{...}}`
- 加载 API Router
- `main()` 函数：解析 `--help`，打印配置摘要，调用 `uvicorn.run()`

**`src/api/endpoints.py`** — API 路由

| 端点 | 方法 | 说明 |
|------|------|------|
| `/v1/messages` | POST | 核心端点，消息转换 |
| `/v1/messages/count_tokens` | POST | Token 估算（4字符≈1token） |
| `/health` | GET | 健康检查 |
| `/test-connection` | GET | 测试上游 API 连通性 |
| `/` | GET | 服务信息 |

关键机制：
- `validate_api_key()`：通过 `Depends()` 注入，校验 `x-api-key` 或 `Authorization` header
- 模块级实例化 `OpenAIClient`（单例，连接池复用）
- 流式请求用 `StreamingResponse` + async generator，支持客户端断连取消

### 5.2 请求转换 — `src/conversion/request_converter.py`

`convert_claude_to_openai()` 主函数：

| Claude 字段 | 转换逻辑 |
|---|---|
| `model` | `model_manager.map_claude_model_to_openai()` |
| `system` (str/list) | → `{"role":"system","content":"..."}` |
| `messages[i].role=user` | → user message，支持 text + image 多模态 |
| `messages[i].role=assistant` | → assistant message，text + tool_calls |
| `messages[i]` 含 tool_result | → `{"role":"tool","tool_call_id":"...","content":"..."}` |
| `thinking.type=thinking` | **跳过**（OpenAI 不支持） |
| `tools` | → `tools: [{type:"function", function:{name,description,parameters}}]` |
| `tool_choice` | auto→auto, any→auto, tool→{type:"function",function:{name}} |
| `max_tokens` | `min(max(max_tokens, min_tokens_limit), max_tokens_limit)` |
| `thinking.enabled=true` | `max_tokens` 至少 16000 |
| `temperature` / `top_p` / `stop_sequences` | 直接透传 |

### 5.3 响应转换 — `src/conversion/response_converter.py`

#### 非流式：`convert_openai_to_claude_response()`

```
OpenAI response.choices[0].message
    │
    ├─ reasoning_content ──▶ content[0] = {type:"thinking", thinking:"..."}
    ├─ content ────────────▶ content[1] = {type:"text", text:"..."}
    └─ tool_calls ─────────▶ content[2+] = {type:"tool_use", id, name, input}

finish_reason 映射:
    stop → end_turn, length → max_tokens, tool_calls → tool_use

usage 映射:
    prompt_tokens → input_tokens
    completion_tokens → output_tokens
    prompt_tokens_details.cached_tokens → cache_read_input_tokens
    (cache_creation_input_tokens 始终为 0)
```

#### 流式：`convert_openai_streaming_to_claude()` / `convert_openai_streaming_to_claude_with_cancellation()`

核心状态机：

```
thinking_started=False, text_started=False

收到 delta.reasoning_content:
  若 !thinking_started → yield content_block_start(thinking, index=0)
  yield content_block_delta(thinking_delta)

收到 delta.content:
  若 thinking_started && !text_started → yield content_block_stop(thinking)
                                        → text_block_index = thinking_block_index + 1
  若 !text_started → yield content_block_start(text, index=text_block_index)
  yield content_block_delta(text_delta)

收到 delta.tool_calls:
  首次收到 id+name → yield content_block_start(tool_use, index=text_block_index+N)
  参数累积完整 JSON → yield content_block_delta(input_json_delta)

流结束:
  关闭所有未关闭的 block
  yield message_delta (stop_reason + usage)
  yield message_stop

异常:
  HTTPException / Exception → yield event:error (Claude 格式)
  不再 re-raise（SSE 流已开始，re-raise 客户端收不到）
```

带取消版本额外逻辑：
- 每个 chunk 前检查 `http_request.is_disconnected()`
- 断连时调用 `openai_client.cancel_request(request_id)` 设置 asyncio.Event
- 从流式 chunk 提取 usage（含 `cache_read_input_tokens`）

### 5.4 客户端 — `src/core/client.py`

`OpenAIClient` 封装了 `AsyncOpenAI` / `AsyncAzureOpenAI`：

| 方法 | 说明 |
|---|---|
| `create_chat_completion()` | 非流式，支持取消（asyncio.wait 竞速） |
| `create_chat_completion_stream()` | 流式，自动加 `stream_options.include_usage=true` |
| `cancel_request()` | 设置取消 Event，中断进行中的请求 |
| `classify_openai_error()` | 错误分类（区域限制/密钥/限流/模型/账单） |

取消机制：每个请求分配 `request_id`，维护 `active_requests: Dict[str, asyncio.Event]`。非流式用 `asyncio.wait` 竞速完成与取消；流式每 chunk 检查 Event 状态。

日志：
- `logger.info`：请求概要（model, stream）、响应概要（model, usage）、错误详情
- `logger.debug`：完整 request/response body（截断 2000 字符）

### 5.5 配置 — `src/core/config.py`

通过环境变量 / `.env` 文件加载：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `OPENAI_API_KEY` | 无（必填） | 上游 API 密钥 |
| `ANTHROPIC_API_KEY` | 无 | 客户端验证密钥（不设则跳过验证） |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | 上游 API 地址 |
| `AZURE_API_VERSION` | 无 | Azure API 版本（有值则用 Azure 客户端） |
| `BIG_MODEL` | `gpt-4o` | Opus 请求映射 |
| `MIDDLE_MODEL` | 同 BIG_MODEL | Sonnet 请求映射 |
| `SMALL_MODEL` | `gpt-4o-mini` | Haiku 请求映射 |
| `HOST` / `PORT` | `0.0.0.0` / `8082` | 服务监听地址 |
| `LOG_LEVEL` | `INFO` | 日志级别 |
| `MAX_TOKENS_LIMIT` | `4096` | max_tokens 上限 |
| `MIN_TOKENS_LIMIT` | `100` | max_tokens 下限 |
| `REQUEST_TIMEOUT` | `90` | 请求超时（秒） |
| `MAX_RETRIES` | `2` | 最大重试次数 |
| `CUSTOM_HEADER_*` | 无 | 自定义上游请求头 |

自定义 Header 规则：`CUSTOM_HEADER_X_API_KEY` → `X-API-KEY`（前缀去掉，下划线转连字符）

### 5.6 模型映射 — `src/core/model_manager.py`

```
输入模型名
  │
  ├─ 以 gpt-/o1- 开头 → 原样透传
  ├─ 以 ep-/doubao-/deepseek- 开头 → 原样透传
  ├─ 含 "haiku" → SMALL_MODEL
  ├─ 含 "sonnet" → MIDDLE_MODEL
  ├─ 含 "opus" → BIG_MODEL
  └─ 其他 → BIG_MODEL（兜底）
```

### 5.7 数据模型 — `src/models/claude.py`

```
ClaudeMessagesRequest
  ├─ model: str
  ├─ max_tokens: int
  ├─ messages: List[ClaudeMessage]
  │   └─ content: Union[str, List[Text|Image|ToolUse|ToolResult|Thinking]]
  ├─ system: Optional[Union[str, List[SystemContent]]]
  ├─ tools: Optional[List[ClaudeTool]]
  ├─ tool_choice: Optional[Dict]
  ├─ thinking: Optional[ClaudeThinkingConfig]
  │   ├─ enabled: bool
  │   └─ budget_tokens: Optional[int]
  ├─ stream: Optional[bool]
  ├─ temperature: Optional[float]
  ├─ top_p: Optional[float]
  └─ stop_sequences: Optional[List[str]]
```

### 5.8 常量 — `src/core/constants.py`

| 类别 | 常量 | 值 |
|---|---|---|
| 角色 | `ROLE_USER/ASSISTANT/SYSTEM/TOOL` | user/assistant/system/tool |
| 内容类型 | `CONTENT_TEXT/IMAGE/TOOL_USE/TOOL_RESULT/THINKING` | text/image/tool_use/tool_result/thinking |
| 工具 | `TOOL_FUNCTION` | function |
| 停止原因 | `STOP_END_TURN/MAX_TOKENS/TOOL_USE/ERROR` | end_turn/max_tokens/tool_use/error |
| SSE 事件 | `EVENT_MESSAGE_START/STOP/DELTA/CONTENT_BLOCK_START/STOP/DELTA/PING` | message_start/message_stop/... |
| Delta 类型 | `DELTA_TEXT/DELTA_INPUT_JSON/DELTA_THINKING` | text_delta/input_json_delta/thinking_delta |

## 6. SSE 事件流格式

Claude API 的流式响应是标准 SSE 协议，每条消息格式为 `event: <type>\ndata: <json>\n\n`。

完整事件序列：

```
event: message_start       ← 消息开始，含 id/role/model/usage
data: {"type":"message_start","message":{...}}

event: ping
data: {"type":"ping"}

event: content_block_start  ← thinking block（如有 reasoning_content）
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start  ← text block
data: {"type":"content_block_start","index":1,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: message_delta        ← 消息结束，含 stop_reason + usage
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{...}}

event: message_stop
data: {"type":"message_stop"}
```

## 7. Thinking 支持

### 数据流

```
GLM/DeepSeek 上游响应:
  非流式: message.reasoning_content = "思考过程"
  流式:   delta.reasoning_content = "思考chunk"
       ↓ 转换
Claude API 格式:
  content_block_start(index=0, type=thinking)
  thinking_delta × N
  content_block_stop(index=0)
  content_block_start(index=1, type=text)
  text_delta × N
  content_block_stop(index=1)
```

### 请求侧

- `ClaudeThinkingConfig.enabled=True` 时，`max_tokens` 至少 16000
- 历史消息中的 `thinking` block 在转换时跳过（OpenAI 不支持）

### 响应侧

- 延迟发送 `content_block_start`（不再在开头固定发 text block）
- 先检测 `reasoning_content`，有则先发 thinking block，再发 text block
- 三种场景全覆盖：有思考+文本 / 仅文本 / 仅思考

## 8. 错误处理机制

### 三层错误处理

| 层级 | 位置 | 机制 |
|---|---|---|
| 全局 | `main.py` `claude_error_handler` | `@app.exception_handler(Exception)` 所有未捕获异常→Claude 格式 JSON |
| 路由 | `endpoints.py` | try/except 捕获 HTTPException 和 Exception，分类处理 |
| 流式内部 | `response_converter.py` | SSE 流已开始后，异常→yield `event:error`（Claude 格式），不 re-raise |

### Claude API 错误格式

```json
{
  "type": "error",
  "error": {
    "type": "authentication_error | rate_limit_error | invalid_request_error | api_error",
    "message": "错误描述"
  }
}
```

## 9. Usage 字段映射

| Claude API | OpenAI API | 说明 |
|---|---|---|
| `input_tokens` | `prompt_tokens` | 输入 token 数 |
| `output_tokens` | `completion_tokens` | 输出 token 数 |
| `cache_creation_input_tokens` | 无对应 | 始终为 0 |
| `cache_read_input_tokens` | `prompt_tokens_details.cached_tokens` | 缓存命中 token 数 |

非流式和流式均已完整映射 4 个字段。

## 10. 依赖说明

| 依赖 | 版本 | 用途 |
|---|---|---|
| `fastapi[standard]` | ≥0.115.11 | Web 框架 |
| `uvicorn` | ≥0.34.0 | ASGI 服务器 |
| `pydantic` | ≥2.0.0 | 数据校验/序列化 |
| `python-dotenv` | ≥1.0.0 | .env 文件加载 |
| `openai` | ≥1.54.0 | OpenAI SDK（含异步客户端） |

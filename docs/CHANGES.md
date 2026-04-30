# 变更日志 (Changelog)

## 2026-04-29

### Thinking 支持

将 OpenAI 的 `reasoning_content` 转换为 Claude API 的 `thinking` content blocks，支持展示推理过程。

- **请求转换**：`thinking.enabled=true` 时自动提升 `max_tokens` 至 16000
- **响应转换**：非流式/流式均支持，thinking block 始终置于 text block 之前
- **历史消息处理**：thinking blocks 在传输时自动跳过（OpenAI 不支持）

* `constants.py` - 新增 `CONTENT_THINKING`、`DELTA_THINKING` 常量
* `claude.py` - 新增 `ClaudeContentBlockThinking`、`ClaudeThinkingConfig.budget_tokens`
* `request_converter.py` - `convert_claude_to_openai()`、`convert_claude_assistant_message()`
* `response_converter.py` - `convert_openai_to_claude_response()`、`convert_openai_streaming_to_claude*()`

---

### Usage 字段完善

补全缺失字段，适配 GLM API。

- 新增 `cache_creation_input_tokens`（始终为 0）
- 新增 `cache_read_input_tokens`（从 GLM `prompt_cache_hit_tokens` 或 OpenAI `prompt_tokens_details.cached_tokens` 提取）
- 流式响应从 chunk 提取实际 usage（不再硬编码为 0）

* `response_converter.py` - `convert_openai_to_claude_response()`、`convert_openai_streaming_to_claude*()`

---

### 流式错误处理

SSE 流中异常不再 re-raise，改为 yield Claude 格式的 `event: error` 事件，确保流式响应异常能被 Claude Code 正确解析。

* `response_converter.py` - `convert_openai_streaming_to_claude*()`

---

### 上游 HTTP 日志

记录上游请求/响应，便于调试和问题排查。

- **INFO 级别**：请求概要（model, stream）、响应概要（model, usage）
- **DEBUG 级别**：完整 request/response body（截断 2000 字符）

* `client.py` - `create_chat_completion()`、`create_chat_completion_stream()`

---

### 全局错误处理

所有未捕获异常以 Claude API 格式返回（`{"type":"error","error":{...}}`），确保 Claude Code 能统一解析各类错误。

* `main.py` - `claude_error_handler()`

---

### 流式响应结构优化

延迟 content block 启动，初始事件仅包含 message_start 和 ping，content_block_start 在收到实际内容时才发送。

* `response_converter.py` - `convert_openai_streaming_to_claude*()`

---

### 移除默认 User-Agent

不再添加 `User-Agent: claude-proxy/1.0.0`，由 OpenAI SDK 使用其默认值。

* `client.py` - `OpenAIClient.__init__()`

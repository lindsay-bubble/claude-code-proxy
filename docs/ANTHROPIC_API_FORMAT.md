# Anthropic API 标准格式文档

[TOC]

---



## 简要对照

### OpenAi 返回格式 → Anthropic API 格式

**非流式：**

```
OpenAI:  message.reasoning_content + message.content
    ↓
Claude:  content[0] = {type:"thinking", thinking: reasoning_content}
        content[1] = {type:"text", text: content}
```

**流式：**

```
OpenAI:  delta.reasoning_content chunks → delta.content chunks
    ↓
Claude:  content_block_start(thinking) → thinking_delta×N → content_block_stop
         content_block_start(text) → text_delta×N → content_block_stop
```



## 请求格式

### 完整请求示例

```typescript
{
  model: "claude-sonnet-4-6",
  messages: [
    {
      role: "user",
      content: "帮我分析这个代码文件"
    },
    {
      role: "assistant",
      content: [
        { type: "text", text: "好的..." },
        { type: "tool_use", id: "123", name: "ReadTool", input: {...} }
      ]
    },
    {
      role: "user",
      content: [
        { type: "tool_result", tool_use_id: "123", content: "..." }
      ]
    }
  ],
  system: "你是一个AI助手...",
  max_tokens: 32000,
  thinking: { type: "adaptive" },
  temperature: undefined,  // thinking启用时不发送
  tools: [...]            // 工具定义
}
```

### 关键请求参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| `model` | string | 模型名称，如 `claude-sonnet-4-6` |
| `messages` | Message[] | 消息历史数组 |
| `system` | string | 系统提示词 |
| `max_tokens` | number | 最大输出 tokens |
| `thinking` | object | 思考配置，见下方 |
| `temperature` | number | 温度参数（仅thinking禁用时使用） |
| `tools` | Tool[] | 可用工具定义 |
| `betas` | string[] | Beta特性头，如 `["output-128k-2025-02-19"]` |

### Thinking 参数

**文件**: `src/services/api/claude.ts:1596-1630`

```typescript
// 自适应思考（新模型）
{
  type: "adaptive"
}

// 预算思考（旧模型或自定义）
{
  type: "enabled",
  budget_tokens: 20000
}

// 禁用思考
undefined
```

### Tools 参数格式

```typescript
{
  name: "ReadTool",
  description: "读取文件内容",
  input_schema: {
    type: "object",
    properties: {
      file_path: { type: "string" }
    },
    required: ["file_path"]
  }
}
```

---

## 响应格式

### 非流式响应

```json
{
  "id": "msg_xxxxx",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "thinking",
      "thinking": "模型思考过程..."
    },
    {
      "type": "text",
      "text": "最终回答..."
    },
    {
      "type": "tool_use",
      "id": "123",
      "name": "ReadTool",
      "input": { "file_path": "..." }
    }
  ],
  "model": "claude-sonnet-4-6",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 1500,
    "output_tokens": 500,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0
  }
}
```

### 流式响应事件序列

流式响应通过 Server-Sent Events (SSE) 返回多个事件：

```typescript
// 1. message_start - 消息开始
{
  type: "message_start",
  message: {
    id: "msg_xxxxx",
    type: "message",
    role: "assistant",
    content: [],
    model: "claude-sonnet-4-6",
    stop_reason: null,
    stop_sequence: null,
    usage: { input_tokens: 1500, output_tokens: 0 }
  }
}

// 2. content_block_start - 内容块开始
{
  type: "content_block_start",
  index: 0,
  content_block: {
    type: "thinking",  // 或 "text", "tool_use"
    thinking: ""       // 对于thinking块
  }
}

// 3. content_block_delta - 内容增量
{
  type: "content_block_delta",
  index: 0,
  delta: {
    type: "thinking_delta",  // 或 "text_delta"
    thinking: "思考内容片段"
  }
}

// 4. content_block_stop - 内容块结束
{
  type: "content_block_stop",
  index: 0
}

// 5. message_delta - 消息级增量
{
  type: "message_delta",
  delta: {
    stop_reason: "end_turn",
    stop_sequence: null
  },
  usage: {
    output_tokens: 500
  }
}

// 6. message_stop - 消息结束
{
  type: "message_stop"
}
```



# AI Provider Integration Overview

> **Complete documentation for LibreChat's AI provider abstraction layer**

## Provider Summary

LibreChat supports **5 major AI providers** with a unified abstraction layer:

| Provider | Status | Streaming | Vision | Tools | Reasoning |
|----------|--------|-----------|--------|-------|-----------|
| **OpenAI** | ✅ Active | ✅ SSE | ✅ GPT-4V+ | ✅ Native | ✅ O1/O3 |
| **Anthropic** | ✅ Active | ✅ SSE | ✅ Claude 3+ | ✅ Native | ✅ Extended Thinking |
| **Google** | ✅ Active | ✅ SSE | ✅ Gemini Vision | ✅ Native | ✅ Thinking Config |
| **Ollama** | ✅ Active | ✅ Native | ✅ With images | ⚠️ Limited | ❌ |
| **Azure OpenAI** | ✅ Active | ✅ SSE | ✅ | ✅ | ✅ |

## Architecture Pattern

**Inheritance Hierarchy:**

```
BaseClient (Abstract)
├── Properties: apiKey, sender, contextStrategy, maxContextTokens
├── Abstract Methods: setOptions(), getCompletion(), buildMessages()
└── Concrete Methods: sendMessage(), loadHistory(), handleContextStrategy()
    │
    ├── OpenAIClient
    │   └── Specializations: Azure, OpenRouter, O1/O3 reasoning, DALL-E
    │
    ├── AnthropicClient
    │   └── Specializations: Prompt caching, extended thinking
    │
    ├── GoogleClient
    │   └── Specializations: Vertex AI, GenAI SDK, thinking config
    │
    └── OllamaClient (Minimal inheritance)
        └── Specializations: Local models, OpenAI format adapter
```

## Key Features

### Universal Token Counting

All providers use **tiktoken** (cl100k_base encoding) for consistent token estimation:

```javascript
// BaseClient.getTokenCountForMessage()
const getTokenCount = (text) => {
  const encoder = new Tiktoken(cl100k_base.bpe_ranks, ...);
  const tokens = encoder.encode(text);
  encoder.free();
  return tokens.length;
};
```

### Context Management

**Two Strategies:**

1. **Discard:** Removes oldest messages to fit context window
2. **Summarize:** Compresses message history (provider-specific)

**Implementation:** `BaseClient.handleContextStrategy()` (lines 437-583)

### Real-Time Streaming

**All providers use SSE (Server-Sent Events):**

```
Client ← SSE Connection → Server ← AI Provider Stream
    │
    ├── "message" events (text chunks)
    ├── "content" events (structured content)
    ├── "step" events (tool calls)
    └── "final" event (completion)
```

### Tool/Function Calling

**16+ Built-in Tools:**
- Calculator, Google Search, Wolfram Alpha
- Web Search (customizable providers)
- Code Execution (sandboxed)
- File Search (RAG)
- Image Generation (DALL-E, Stable Diffusion, Flux)
- MCP Integration (Model Context Protocol servers)

## Provider-Specific Features

### OpenAI

**File:** `/home/user/LibreChat/api/app/clients/OpenAIClient.js` (1,207 lines)

**Key Features:**
- O1/O3 reasoning models (`max_completion_tokens`)
- Azure OpenAI support (dynamic endpoints)
- OpenRouter compatibility
- Reasoning token separation
- DALL-E integration

**Streaming Handler:** `SplitStreamHandler` with `<think>` tag parsing

### Anthropic

**File:** `/home/user/LibreChat/api/app/clients/AnthropicClient.js` (991 lines)

**Key Features:**
- Prompt caching (ephemeral cache control)
- Extended thinking with budget control
- Message grouping (consecutive same-role)
- Retry logic (exponential backoff)
- Custom stream handler for delta extraction

**Cache Control:**
```javascript
if (supportsCacheControl) {
  system: [{
    type: 'text',
    text: systemMessage,
    cache_control: { type: 'ephemeral' }
  }]
}
```

### Google

**File:** `/home/user/LibreChat/api/app/clients/GoogleClient.js` (994 lines)

**Key Features:**
- Dual SDK support (Vertex AI + GenAI)
- Thinking configuration (`thinkingBudget`)
- Gemini-specific formatting (parts array)
- Safety settings
- Vision with `inlineData` format

**SDK Selection:**
```javascript
if (project_id) {
  return new ChatVertexAI(options);  // Enterprise
} else {
  return new GenAI(apiKey).getGenerativeModel({ model });  // Direct API
}
```

### Ollama

**File:** `/home/user/LibreChat/api/app/clients/OllamaClient.js` (168 lines)

**Key Features:**
- Local model support
- OpenAI format adapter
- Simplified streaming
- Model discovery via `/api/tags`
- Image support (base64 encoding)

## File References

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| BaseClient | `/api/app/clients/BaseClient.js` | 1-1419 | Abstract base class |
| OpenAI | `/api/app/clients/OpenAIClient.js` | 1-1207 | OpenAI/Azure implementation |
| Anthropic | `/api/app/clients/AnthropicClient.js` | 1-991 | Claude implementation |
| Google | `/api/app/clients/GoogleClient.js` | 1-994 | Gemini/Vertex implementation |
| Ollama | `/api/app/clients/OllamaClient.js` | 1-168 | Local models |
| TextStream | `/api/app/clients/TextStream.js` | 1-61 | Streaming utility |
| Token Counter | `/api/server/utils/countTokens.js` | 1-38 | Universal token counting |

## Related Documentation

- [OpenAI Integration](openai.md) - Detailed OpenAI client docs
- [Anthropic Integration](anthropic.md) - Claude-specific features
- [Google Integration](google.md) - Gemini/Vertex AI details
- [Provider Abstraction](provider-abstraction.md) - Base client architecture

---

**Last Updated:** 2025-11-18
**Version:** 1.0.0

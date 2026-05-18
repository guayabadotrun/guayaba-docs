# Chat

Send messages to your agents programmatically. Supports synchronous JSON responses and real-time SSE streaming.

```
POST /agents/{id}/chat
```

**Auth**: Master key or agent key with `chat` scope.
**Cost**: 1 base credit on a successful response, plus model-dependent LLM usage billed asynchronously from runtime usage events.

Chat also requires the agent to be `running` and the selected framework to support the chat capability. OpenClaw is the current framework and backs this endpoint with its OpenAI-compatible runtime gateway.

## Request

```json
{
  "message": "What's the weather in Madrid?",
  "sessionKey": "api:550e8400-e29b-41d4-a716-446655440000",
  "stream": false
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | string | Yes | The user message (max 32,000 chars) |
| `sessionKey` | string | No | Reuse an existing session. Must match `api:[a-zA-Z0-9-]+`, max 255 chars. Auto-generated if omitted. |
| `stream` | boolean | No | `true` for SSE streaming, `false` (default) for sync JSON |

## Sync Response

When `stream` is `false` (default), the API waits for the full agent response and returns it as JSON.

```bash
curl -X POST https://api.guayaba.run/api/v1/agents/550e8400-.../chat \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello, what can you do?"}'
```

**Response** (`200`):
```json
{
  "success": true,
  "data": {
    "sessionKey": "api:550e8400-e29b-41d4-a716-446655440000",
    "message": {
      "role": "assistant",
      "content": "I can help you with a variety of tasks..."
    }
  }
}
```

## Streaming Response (SSE)

When `stream` is `true`, the response is a `text/event-stream`. The backend wraps the gateway's OpenAI-compatible chunk stream with two envelope events of its own:

1. A first `session` event with the resolved `sessionKey`.
2. The forwarded OpenAI `chat.completion.chunk` stream, terminated by `data: [DONE]`.
3. A final `done` event with the same `sessionKey`.

If the backend cannot reach the agent gateway, a single `error` event is emitted in place of the upstream stream.

```bash
curl -X POST https://api.guayaba.run/api/v1/agents/550e8400-.../chat \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Explain quantum computing", "stream": true}'
```

**Event stream**:
```
data: {"type":"session","sessionKey":"api:550e8400-..."}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","content":"Quantum"}}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" computing is"}}]}

data: [DONE]

data: {"type":"done","sessionKey":"api:550e8400-..."}
```

### Event Types

| Type | Origin | Description |
|---|---|---|
| `session` | Guayaba backend | First event. Contains the resolved `sessionKey`. |
| `chat.completion.chunk` | Forwarded from the gateway (standard OpenAI shape — has no `type` field) | Incremental assistant content under `choices[0].delta.content`. Tool calls appear inline under `choices[0].delta.tool_calls`. |
| `[DONE]` | Forwarded from the gateway | Sentinel marking the end of the upstream stream. |
| `done` | Guayaba backend | Final envelope with the `sessionKey`. |
| `error` | Guayaba backend | Emitted only when the backend cannot connect to the agent gateway. Replaces the upstream stream. |

## Sessions

Each chat message belongs to a **session**. Sessions allow multi-turn conversations where the agent remembers previous context.

- If you omit `sessionKey`, a new session is created automatically with an `api:` prefix.
- To continue a conversation, pass the `sessionKey` returned in the first response.
- Sessions created via the API are prefixed with `api:` to distinguish them from UI sessions.

```bash
# First message — new session created
curl -X POST .../chat -d '{"message": "What is Rust?"}'
# Response: sessionKey = "api:abc123..."

# Follow-up — same session
curl -X POST .../chat -d '{"message": "How does its borrow checker work?", "sessionKey": "api:abc123..."}'
```

You can manage sessions (list, rename, archive, delete) through the [Sessions endpoints](agents.md#sessions).

## Tool Calls

Agents can use tools (web search, code execution, etc.) during a conversation. In streaming mode, tool calls appear inline as `delta.tool_calls` deltas inside the upstream `chat.completion.chunk` events — there is no separate `tool` envelope frame. In sync mode, the final response includes the result after all tools have completed.

Tool calls are auto-approved — no user confirmation is needed when using the API.

## Errors

| Code | Meaning |
|---|---|
| `401` | Invalid or missing API key |
| `402` | Insufficient credits |
| `403` | Key lacks `chat` scope or cannot access this agent |
| `404` | Agent not found |
| `409` | Agent is not running yet, including while it is `starting`, or the framework does not support chat |
| `422` | Validation error (missing `message`, too long, etc.) |
| `502` | Agent container unreachable |

See [Errors](errors.md) for the full error reference.

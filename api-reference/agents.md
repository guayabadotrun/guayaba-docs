# Agents

## List Agents

Returns all agents owned by the API key's user.

```
GET /agents
```

**Auth**: Any key (master or agent).

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `per_page` | integer | 100 | Results per page |
| `page` | integer | 1 | Page number |

```bash
curl https://api.guayaba.run/api/v1/agents \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "My Agent",
      "status": "running",
      "channels": ["telegram"],
      "model_provider": "openrouter",
      "settings": { "model": "openai/gpt-4.1" },
      "created_at": "2026-03-15T10:30:00Z"
    }
  ],
  "current_page": 1,
  "last_page": 1,
  "per_page": 100,
  "total": 1
}
```

---

## Show Agent

Returns full details for a specific agent.

```
GET /agents/{id}
```

**Auth**: Master key (any agent) or agent key with `agent:read` scope (own agent only).

```bash
curl https://api.guayaba.run/api/v1/agents/550e8400-... \
  -H "Authorization: Bearer g_agent_YOUR_KEY"
```

---

## Create Agent

Creates a new AI agent.

```
POST /agents
```

**Auth**: Master key only. **Cost**: 10 credits.

```bash
curl -X POST https://api.guayaba.run/api/v1/agents \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Bot",
    "llm_provider_name": "openrouter",
    "llm_provider_model": "openai/gpt-4.1",
    "personality": "You are a helpful support agent.",
    "vibe": "Friendly, concise, professional.",
    "knowledge_seed": ["Refund policy: 30 days."],
    "framework_id": "FRAMEWORK_UUID"
  }'
```

**Response** (`201`):
```json
{
  "success": true,
  "message": "Agent created successfully",
  "data": { "id": "...", "name": "Support Bot", "status": "created" }
}
```

**Optional**: pass `graft_slug` (and optionally `graft_overrides`) to apply
a marketplace template (see [GRAFTs](grafts.md)). The GRAFT's `schema.defaults`
fill in any field omitted in the request; fields you provide always win.

---

## Update Agent

Updates an existing agent. Uses PATCH semantics — only provided fields are changed.

```
PUT /agents/{id}
```

**Auth**: Master key only. **Cost**: 10 credits.

**Secrets use PATCH semantics**: send `null` or `""` to delete a secret; omit the key entirely to keep it.

```bash
curl -X PUT https://api.guayaba.run/api/v1/agents/550e8400-... \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Bot",
    "settings": { "model": "anthropic/claude-sonnet-4.6" }
  }'
```

---

## Delete Agent

Deletes an agent. Stops it first if running. Cascades to API keys and sessions.

```
DELETE /agents/{id}
```

**Auth**: Master key only. **Cost**: Free.

```bash
curl -X DELETE https://api.guayaba.run/api/v1/agents/550e8400-... \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

---

## Runtime Control

All runtime endpoints require `agent:manage` scope (master keys always pass).

### Get Status

```
GET /agents/{id}/status
```

Returns current status, uptime, and timestamps. **Free.**

```bash
curl https://api.guayaba.run/api/v1/agents/550e8400-.../status \
  -H "Authorization: Bearer g_agent_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "status": "running",
  "last_boot_execution": "2026-04-09T08:00:00Z",
  "last_stop_execution": null,
  "uptime_total_seconds": 86400,
  "current_uptime": 3600,
  "server_timestamp": 1744185600000
}
```

### Health Check

```
GET /agents/{id}/health
```

Returns detailed health for a running agent. **Free.**

### Start Agent

```
POST /agents/{id}/start
```

Boots the agent or resumes from paused. **Cost**: 5 credits.

### Stop Agent

```
POST /agents/{id}/stop
```

Stops the agent and accumulates uptime. **Free.**

### Pause Agent

```
POST /agents/{id}/pause
```

Pauses without stopping the container. **Free.**

### Reload Config

```
POST /agents/{id}/reload
```

Hot-reloads configuration without restarting. Agent must be running. **Cost**: 1 credit.

### Get Logs

```
GET /agents/{id}/logs
```

Returns the agent's runtime log buffer. **Free.**

**Query parameters**:

| Parameter | Type | Description |
|---|---|---|
| `limit` | integer | Max entries to return (from tail). Default: 200 |
| `level` | string | Comma-separated levels: `info`, `warn`, `error`. Omit for all. |

Level filtering is applied **before** the limit — requesting `?level=warn&limit=50` returns up to 50 warn entries from the entire buffer.

**Sanitization**: Log messages are sanitized before delivery. Internal IP addresses and ports are replaced with `[internal]`, and filesystem paths with `[agent-workspace]`. Certain expected infrastructure warnings are suppressed entirely. Raw logs are only available via direct container access.

```bash
curl "https://api.guayaba.run/api/v1/agents/550e8400-.../logs?level=warn,error&limit=100" \
  -H "Authorization: Bearer g_agent_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "logs": [
    {
      "ts": "2026-04-20T10:00:00.000Z",
      "level": "warn",
      "msg": "Config warning: missing optional field"
    },
    {
      "ts": "2026-04-20T10:00:02.000Z",
      "level": "error",
      "msg": "Connection timeout after 15s"
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `ts` | string | ISO 8601 timestamp |
| `level` | string | `info`, `warn`, or `error` |
| `msg` | string | Log message text |

---

## Agent Status Values

| Status | Description |
|---|---|
| `created` | Agent created but never started |
| `active` | Provisioning / booting |
| `running` | Running and accepting requests |
| `paused` | Paused (container alive, not processing) |
| `stopped` | Stopped (no container) |
| `failed` | Failed to start or crashed |

---

## Sessions

Manage conversation sessions. All session endpoints require `agent:manage` scope.

### List Sessions

```
GET /agents/{id}/sessions
```

### Get Session History

```
POST /agents/{id}/sessions/history
```

```json
{ "sessionKey": "api:550e8400-...", "limit": 100 }
```

### Rename Session

```
POST /agents/{id}/sessions/rename
```

```json
{ "key": "api:550e8400-...", "label": "Support Conversation #1" }
```

### Archive Session

```
POST /agents/{id}/sessions/archive
```

```json
{ "key": "api:550e8400-..." }
```

### Delete Session

```
POST /agents/{id}/sessions/delete
```

```json
{ "key": "api:550e8400-..." }
```

---

## Archives

### List Archives

```
GET /agents/{id}/archives
```

### Restore Archive

```
POST /agents/{id}/archives/restore
```

```json
{ "filename": "archive-file.json", "label": "Restored Session" }
```

### Delete Archive

```
POST /agents/{id}/archives/delete
```

```json
{ "filename": "archive-file.json" }
```

---

## Channels (Telegram)

Manage Telegram pairing. All endpoints require `channels` scope.

### List Pairing Requests

```
GET /agents/{id}/channels/telegram/pairing
```

### List Approved Pairings

```
GET /agents/{id}/channels/telegram/pairing/approved
```

### Approve Pairing

```
POST /agents/{id}/channels/telegram/pairing/approve
```

```json
{ "code": "PAIRING_CODE" }
```

### Revoke Pairing

```
POST /agents/{id}/channels/telegram/pairing/revoke
```

```json
{ "userId": "TELEGRAM_USER_ID" }
```

### Reject Pairing

```
POST /agents/{id}/channels/telegram/pairing/reject
```

```json
{ "code": "PAIRING_CODE" }
```

---

## Files

Upload files to an agent's knowledge base. Requires `files` scope. The agent must be running.

```
POST /agents/{id}/files
```

```json
{
  "filename": "faq.txt",
  "content": "Base64 or plain text content...",
  "mimeType": "text/plain"
}
```

---

## Catalogs

Read-only reference data. Any API key type. Responses are cached for 10 minutes.

| Endpoint | Description |
|---|---|
| `GET /catalogs/frameworks` | Available agent frameworks |
| `GET /catalogs/frameworks/{id}/clients` | Clients for a framework |
| `GET /catalogs/regions` | Deployment regions |
| `GET /catalogs/hardware` | Hardware configurations |
| `GET /catalogs/models` | Available AI models |

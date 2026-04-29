# Errors

All error responses follow a consistent format:

```json
{
  "error": "ErrorCategory",
  "message": "Human-readable explanation"
}
```

## Status Codes

| Code | Error | Description |
|---|---|---|
| `401` | Unauthorized | Missing, invalid, revoked, or expired API key |
| `402` | Payment Required | Insufficient credits for the operation |
| `403` | Forbidden | No active subscription, key lacks required scope, wrong agent, or master key required |
| `404` | Not Found | Agent or resource does not exist |
| `409` | Conflict | Agent is not in the required state (e.g., chatting with a stopped agent) |
| `422` | Validation Error | Request body failed validation |
| `429` | Too Many Requests | Rate limit exceeded |
| `502` | Bad Gateway | Agent container returned an error or is unreachable |
| `503` | Service Unavailable | Agent container is not running |

## Error Examples

### 401 — Invalid API Key

```json
{
  "error": "Unauthorized",
  "message": "Invalid or revoked API key"
}
```

### 402 — Insufficient Credits

```json
{
  "error": "Payment Required",
  "message": "Insufficient credits for this operation",
  "required_credits": 10,
  "available_credits": 3
}
```

### 403 — Missing Scope

```json
{
  "error": "Forbidden",
  "message": "This API key lacks the required scope: agent:manage"
}
```

### 403 — Master Key Required

```json
{
  "error": "Forbidden",
  "message": "This endpoint requires a master API key"
}
```

### 403 — No Active Master Key (Agent Key Creation)

```json
{
  "error": "Forbidden",
  "message": "You must have an active master API key before creating agent keys"
}
```

### 403 — No Active Subscription

```json
{
  "error": "Forbidden",
  "message": "An active subscription is required to use the API"
}
```

### 404 — Agent Not Found

```json
{
  "error": "Not Found",
  "message": "Agent not found"
}
```

### 422 — Validation Error

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "message": ["The message field is required."],
    "sessionKey": ["The session key must start with 'api:'."]
  }
}
```

### 502 — Gateway Error

```json
{
  "error": "Bad Gateway",
  "message": "Failed to communicate with the agent container"
}
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401` on every request | Key was revoked or regenerated | Create a new key from the dashboard |
| `403` "active subscription required" | Subscription expired or canceled | Reactivate your subscription |
| `403` "lacks the required scope" | Agent key missing the needed scope | Create a new agent key with the correct scopes |
| `403` "master API key required" | Using an agent key on a master-only endpoint | Use your master key for CRUD and billing operations |
| `402` on create/start | Low credit balance | Check balance with `GET /billing/credits` and top up |
| `409` on chat/reload | Agent is not running | Start the agent first with `POST /agents/{id}/start` |
| `502` on chat | Agent container crashed or is unresponsive | Check agent health with `GET /agents/{id}/health` |

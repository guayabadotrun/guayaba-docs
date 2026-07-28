# Guayaba Public API v1

The Guayaba Public API lets you create, manage, and chat with your AI agents programmatically — no web UI needed.

## Base URL

```
https://api.guayaba.run/api/v1
```

All endpoints are relative to this base URL.

## Quick Start

### 1. Create an API key

Go to **Settings → API Keys** in the [Guayaba dashboard](https://app.guayaba.run) and create a **Master Key**.

> Each API key is bound to the organization you are in when you create it. To act on a different organization, switch organizations in the dashboard header and mint a new key. See [Organizations](../getting-started/organizations.md).

### 2. List your agents

```bash
curl https://api.guayaba.run/api/v1/agents \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

### 3. Start an agent

```bash
curl -X POST https://api.guayaba.run/api/v1/agents/{id}/start \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

### 4. Chat with your agent

```bash
curl -X POST https://api.guayaba.run/api/v1/agents/{id}/chat \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello, what can you do?"}'
```

## Requirements

- An active Guayaba subscription (any tier).
- At least one API key (Master or Agent).
- Sufficient credits for billed operations.

## What's Next

- [Authentication](authentication.md) — Key types, scopes, and how auth works.
- [Agents](agents.md) — Create, update, delete, and control your agents.
- [Chat](chat.md) — Send messages with sync and streaming responses.
- [Billing & Credits](billing.md) — How credits work and what operations cost.
- [Dashboard API](manager-api.md) — Session-authenticated dashboard-only observability endpoints.
- [Errors](errors.md) — Error format and status codes.

## OpenAPI Spec

The full OpenAPI 3.0 specification is available for import into tools like Postman, Insomnia, or code generators:

- [Download openapi-v1.yaml](https://api.guayaba.run/api/docs/openapi-v1.yaml)

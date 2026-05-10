# Authentication

All API requests require an API key sent as a **Bearer token**:

```
Authorization: Bearer <your-api-key>
```

## Key Types

Guayaba uses a two-tier key system:

### Master Key (`g_master_`)

- **One active key per backend account context.** Creating a new one revokes the previous master key and all its agent keys in that context.
- Full access to every endpoint and every agent available to that context.
- Required for: creating/updating/deleting agents, billing endpoints, and managing API keys.
- Format: `g_master_` + 48 hex characters (57 chars total).

### Agent Key (`g_agent_`)

- **One per agent.** Tied to a single agent by ID.
- Limited access controlled by **scopes** you choose at creation time.
- Cannot access other agents — returns `403` if attempted.
- Format: `g_agent_` + 48 hex characters (56 chars total).

## Creating Keys

Keys are managed from the Guayaba dashboard:

- **Master Key**: Settings → API Keys → Create Master Key
- **Agent Key**: Agent Detail → API Keys → Create Agent Key

> **You must have an active master key before creating agent keys.** If your master key is revoked, you need to create a new one first.

> The full key is shown **only once** at creation. Store it securely — lost keys cannot be recovered, only regenerated.

## Scopes

Scopes control what an agent key can do. Master keys bypass all scope checks.

| Scope | Description | Included by default |
|---|---|---|
| `agent:read` | View agent details and configuration | Always |
| `agent:manage` | Start, stop, pause, reload. Manage sessions and archives | No |
| `channels` | Manage Telegram pairing (list, approve, revoke, reject) | No |
| `files` | Upload files to the agent | No |
| `chat` | Send messages to the agent via API | No |

`agent:read` is always included automatically in every agent key.

## Subscription Requirement

All endpoints require the key's backend account context to have an **active subscription** (status: `active` or `trialing`). New accounts are automatically enrolled on the free **Hobby** plan at registration, so this requirement is met from day one — no additional setup needed. If the subscription is inactive, every request returns:

```
403 Forbidden
{
  "error": "Forbidden",
  "message": "An active subscription is required to use the API"
}
```

> **Exception — GRAFT authoring endpoints.** `POST /api/v1/grafts/validate`, `POST /api/v1/grafts`, and `POST` or `PUT /api/v1/grafts/{slug}/assets/{type}` only require a valid **master API key** (`RequireMasterApiKey`). They do not enforce `RequireActiveSubscription`, so authors can validate and push GRAFT bundles without an active subscription.

## Key Revocation

- **Revoking a master key cascades** — all agent keys under the same backend account context are also revoked.
- Regenerating a key revokes the old one and creates a new one.
- Revoked keys return `401 Unauthorized` immediately.

## Key Expiration

Keys can optionally be set to expire at a specific date. By default, keys do not expire. Expired keys behave the same as revoked keys (`401`).

## Example

```bash
# Using a master key
curl https://api.guayaba.run/api/v1/agents \
  -H "Authorization: Bearer g_master_a1b2c3d4e5f6..."

# Using an agent key (with agent:read scope)
curl https://api.guayaba.run/api/v1/agents/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer g_agent_f6e5d4c3b2a1..."
```

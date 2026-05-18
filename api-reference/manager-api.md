# Dashboard API

Most public automation should use the [Public API v1](introduction.md). This page documents the dashboard-only **observability** and **runtime budget** endpoints used by the Guayaba web app.

## Base URL

```
https://api.guayaba.run/api
```

This API is authenticated with your dashboard session, not with `g_master_*` or `g_agent_*` API keys. Do not use Public API v1 keys against these endpoints. The scope here is **only the observability + runtime budget endpoints below** — these routes are documented so dashboard behavior is explicit and **are not a stable automation surface**. They may change without notice; treat the [Public API v1](introduction.md) as the contract for any third-party integration.

## Observability Agents

```
GET /observability/agents
```

Returns a paginated list of agents in your current backend account context with latest observability summaries.

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | 1 | Page number |
| `per_page` | integer | 25 | Results per page |
| `range` | `1h`, `6h`, `24h`, `7d`, `30d` | `24h` | Time range used to compute the per-row usage totals and pending counts |

**Response fields**:

- `agent.uptime_total_seconds`: persisted closed-session uptime.
- `infrastructure.available`: whether infrastructure metrics are available for this environment.
- `usage.totals.tokens`: runtime-reported token total for the range when available.
- `usage.totals.usage_usd`: ready provider spend for the range; `null` while recent cost enrichment is still pending.
- `usage.totals.credits`: credits already charged by LLM usage billing; `null` while billing is still pending.
- `usage.pending.enrichment_count`: persisted events waiting for provider cost enrichment.
- `usage.pending.billing_count`: persisted or enriched events waiting for credit billing.

```json
{
  "current_page": 1,
  "data": [
    {
      "agent": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "tenant_id": "TENANT_UUID",
        "name": "Support Agent",
        "status": "running",
        "model": "openrouter/anthropic/claude-sonnet-4.6",
        "uptime_total_seconds": 3600,
        "updated_at": "2026-05-07T10:00:00Z"
      },
      "infrastructure": {
        "source": "unavailable",
        "available": false,
        "reason": "Infrastructure metrics are not configured for this environment.",
        "latest": {
          "cpu_percent": null,
          "memory_mb": null,
          "disk_bytes": null,
          "network_rx_bytes": null,
          "network_tx_bytes": null
        }
      },
      "usage": {
        "source": "llm_usage_events",
        "available": true,
        "token_precision": "exact",
        "totals": {
          "tokens": 2000,
          "usage_usd": null,
          "credits": null
        },
        "pending": {
          "enrichment_count": 1,
          "billing_count": 1
        }
      }
    }
  ],
  "per_page": 25,
  "total": 1
}
```

## Agent Observability Summary

```
GET /agents/{id}/observability/summary
```

Returns the detailed observability data shown in an agent's Resources tab.

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `range` | `1h`, `6h`, `24h`, `7d`, `30d` | `24h` | Time range for metric series |

The response has the same `agent`, `infrastructure`, and `usage` concepts as `GET /observability/agents`, but includes time-series data under `infrastructure.metrics` and `usage.series`. Token and credit series are cumulative running totals over the selected range. Spend series use per-bucket ready provider spend. Pending spend or credit points use `null` until cost enrichment and billing are ready, so pending usage is not displayed as zero spend.

## Dashboard Observability Summary

```
GET /observability/summary
```

Returns current-context LLM usage series for the dashboard Observability page.

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `range` | `1h`, `6h`, `24h`, `7d`, `30d` | `24h` | Time range for usage series |

```json
{
  "success": true,
  "data": {
    "tenant_id": "TENANT_UUID",
    "usage": {
      "source": "llm_usage_events",
      "available": true,
      "token_precision": "exact",
      "range": {
        "from": "2026-05-07T09:00:00Z",
        "to": "2026-05-07T10:00:00Z"
      },
      "series": {
        "tokens": { "unit": "tokens", "points": [{ "timestamp": "2026-05-07T09:30:00Z", "value": 2000 }] },
        "usage_usd": { "unit": "USD", "points": [{ "timestamp": "2026-05-07T09:30:00Z", "value": null }] },
        "credits": { "unit": "credits", "points": [{ "timestamp": "2026-05-07T09:30:00Z", "value": null }] }
      },
      "totals": {
        "tokens": 2000,
        "usage_usd": null,
        "credits": null
      },
      "pending": {
        "enrichment_count": 1,
        "billing_count": 1
      },
      "model_count": 1,
      "model_breakdown": [],
      "model_series": { "unit": "tokens", "models": [] },
      "agent_activity": { "unit": "calls", "active_agent_count": 1, "total_calls": 1, "agents": [] }
    }
  }
}
```

Token and credit chart points are cumulative running totals. `model_series.models[].points` stay per-interval token counts by model for the Model Usage stacked bars, and `agent_activity.agents[].points` are cumulative call counts.

## OpenRouter Usage Proxy

```
GET /agents/{id}/usage
```

Returns OpenRouter key usage for the agent without exposing the agent's OpenRouter API key to the browser.

Possible responses:

- `200`: OpenRouter usage data.
- `403`: the agent is not available to the authenticated dashboard session.
- `404`: the agent does not exist or has no OpenRouter key.
- `502`: OpenRouter usage could not be retrieved.

## Daily Runtime

```
GET /subscription/daily-runtime
```

Returns today's runtime budget usage for the authenticated dashboard session's current tier.

```json
{
  "success": true,
  "data": {
    "seconds_used": 1830,
    "limit_seconds": 3600,
    "persisted_seconds_used": 1200,
    "live_seconds_used": 630
  }
}
```

Field meanings:

- `seconds_used`: effective total for the current UTC day.
- `limit_seconds`: daily tier limit, or `null` for unlimited tiers.
- `persisted_seconds_used`: closed-session seconds already saved today.
- `live_seconds_used`: live `starting`/`running` session seconds calculated at read time.

The live portion is calculated on demand rather than written every minute, which avoids double-counting when agents stop, fail, or are stopped by quota enforcement.

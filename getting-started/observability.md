# Observability

Guayaba includes observability views for understanding whether your agents are running, how long they have been active, and how much LLM usage they are generating.

You can see observability in two places:

- The **Resources** tab on an agent detail page shows per-agent infrastructure and usage charts.
- The **Observability** page shows a table of your agents with uptime, infrastructure, and usage summaries.

## Runtime And Uptime

Agent uptime has two related values:

- **Persisted uptime**: completed running time that has already been saved after an agent stops or fails.
- **Effective uptime**: persisted uptime plus the current live session for agents that are running or starting.

The dashboard uses effective values so running agents do not appear stuck at `0` or an old value while they are still active.

## Hobby Daily Runtime

The Hobby plan includes a daily running-time limit. The limit is counted across your running agents and resets at midnight UTC.

The daily counter is also effective: it combines already-saved closed sessions with live running time. When the daily limit is reached, Guayaba stops running agents and prevents new starts until the next UTC day.

See [Billing & Credits](../api-reference/billing.md) for plan limits and billing behavior.

## Infrastructure Metrics

In production, Guayaba can show infrastructure metrics such as CPU, memory, disk, and network usage. These metrics are read on demand from the production telemetry backend.

In local development or environments where telemetry is not configured, infrastructure charts may show as unavailable. This is expected and does not mean your agent usage or runtime accounting is broken.

## LLM Usage

Usage data comes from runtime model-call events:

- **Tokens** are reported by the agent runtime after each model call, so they can appear quickly.
- **Spend** is shown only after provider cost enrichment is complete. For managed OpenRouter calls, Guayaba enriches the runtime event asynchronously from OpenRouter generation data.
- **Credits** are shown only after the usage billing batch has run.

This distinction is intentional: recent tokens may be visible while spend or credits are still processing. The dashboard shows a small processing note in that case and uses empty values rather than showing pending cost as `$0`.

## Data Freshness

Runtime values are calculated live when the dashboard reads them. Token usage is usually available shortly after a model call is observed by the runtime. Cost and credit values can lag behind very recent messages because enrichment and billing run asynchronously.

If an agent is currently running, the most reliable view of remaining Hobby time is the dashboard header or billing/profile area, which refreshes periodically.

For the dashboard API shapes behind these views, see [Dashboard API](../api-reference/manager-api.md). These endpoints use your dashboard session, not Public API v1 keys.

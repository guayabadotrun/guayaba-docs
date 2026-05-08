# Billing & Credits

API operations that consume resources are billed in **credits**. Credits are checked before executing the operation — if you don't have enough, the request is rejected with `402`.

## Credit Costs

| Operation | Credits | Endpoint |
|---|---|---|
| Create agent | 10 | `POST /agents` |
| Update agent | 10 | `PUT /agents/{id}` |
| Start agent | 5 | `POST /agents/{id}/start` |
| Reload config | 1 | `POST /agents/{id}/reload` |
| Chat message | 1 + LLM cost | `POST /agents/{id}/chat` |
| Stop / Pause / Delete agent | 0 | — |
| All read operations (GET) | 0 | — |
| Key management | 0 | — |

> Credit costs are provisional and subject to change.

**Chat billing note**: The base credit (1) is charged on successful completion. The LLM cost (model-dependent token usage) is recorded by the runtime and charged asynchronously after provider cost data is ready, same as when using the web UI.

## Checking Your Balance

```
GET /billing/credits
```

**Auth**: Master key only.

```bash
curl https://api.guayaba.run/api/v1/billing/credits \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response**:
```json
{
  "success": true,
  "data": {
    "total_credits": 450,
    "subscription_credits": 400,
    "purchased_credits": 50
  }
}
```

## Subscription Info

```
GET /billing
```

Returns your current subscription details (tier, status, period dates, trial info).

**Auth**: Master key only.

## Available Tiers

```
GET /billing/tiers
```

Returns all subscription tiers with pricing, credit allowances, and feature lists.

**Auth**: Master key only.

## Insufficient Credits

When you don't have enough credits for an operation, the API returns:

```
402 Payment Required
{
  "error": "Payment Required",
  "message": "Insufficient credits for this operation",
  "required_credits": 10,
  "available_credits": 3
}
```

The operation is **not** executed. No credits are deducted.

## Subscription Plans

| Plan | Price | Running time | Agents | Credit allowance |
|---|---|---|---|---|
| **Hobby** | Free | 1 hour/day | 1 | 2,000 (welcome grant) |
| **Early Access** | $4.90/mo | Unlimited | 1 | 3,000/month |
| **Pro** | $49.00/mo | Unlimited | 3 | 10,000/month |

Every new account is automatically enrolled on the **Hobby** plan and receives 2,000 one-time welcome credits immediately — no credit card required. Paid plans include recurring subscription credits. You can upgrade at any time from the billing dashboard.

### Daily running-time limit (Hobby)

Hobby accounts may run agents for a combined total of **1 hour per UTC day** across all their agents. When the limit is reached, running agents are automatically stopped and the agent start endpoint returns:

```
403 Forbidden
{
  "success": false,
  "message": "Your Hobby plan includes 60 minutes of agent running time per day. You have used your full allowance for today. Your limit resets at midnight UTC."
}
```

The counter resets at **midnight UTC** every day. Public API v1 does not currently expose a daily-runtime read endpoint; use the dashboard to see remaining Hobby time.

## Subscription Requirement

All public API endpoints require an active subscription (`active` or `trialing` status). New accounts are always enrolled on Hobby (active) at registration, so this condition is met automatically. If your subscription is canceled, past due, or absent:

```
403 Forbidden
{
  "error": "Forbidden",
  "message": "An active subscription is required to use the API"
}
```

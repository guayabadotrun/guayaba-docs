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

**Chat billing note**: The base credit (1) is charged on successful completion. The LLM cost (model-dependent token usage) is charged separately by the runtime, same as when using the web UI.

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

## Subscription Requirement

All API endpoints require an active subscription (`active` or `trialing` status). If your subscription is canceled, past due, or absent:

```
403 Forbidden
{
  "error": "Forbidden",
  "message": "An active subscription is required to use the API"
}
```

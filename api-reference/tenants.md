# Tenants (Organizations)

Endpoints for managing the organization your API key is bound to, and for creating new organizations, accepting invitations, and listing every organization you belong to.

The API uses **`tenant_id`** as the internal field name for organization IDs. In user-facing prose the platform calls these "organizations" — see [Organizations](../getting-started/organizations.md) for the concept.

**Auth**: Every route on this page requires an API key. Unless noted otherwise, a **master key** (`g_master_…`) is required. Only `GET /tenant` is reachable by agent-scoped keys.

**Organization resolution**: Master keys are bound to exactly one organization at creation time. All `/tenant/*` routes act on that organization implicitly — the organization ID is never accepted in the URL or the body. To act on a different organization, mint a fresh master key inside it.

---

## Show the Current Organization

Returns the organization the invoking key is bound to, plus your role in it and a minimal subscription snapshot.

```
GET /tenant
```

**Auth**: Master key **or** agent key. This is the only tenant route that accepts agent keys.

```bash
curl https://api.guayaba.run/api/v1/tenant \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Acme Robotics",
    "kind": "organization",
    "slug": "acme-robotics",
    "image_url": null,
    "owner_user_id": "0e1f2a3b-4c5d-6e7f-8091-a2b3c4d5e6f7",
    "member_count": 4,
    "role": "admin",
    "is_default": false,
    "subscription": {
      "tier_id": "b7ef1a30-1111-4444-8888-aaaaaaaaaaaa",
      "status": "active",
      "started_at": "2026-06-01T09:00:00Z"
    }
  }
}
```

`kind` is `"personal"` for the organization created at signup (displayed as **Personal** in the UI) and `"organization"` for every organization created afterwards.

---

## Update the Current Organization

Renames the organization or changes its avatar. `kind` and `slug` are immutable via this endpoint.

```
PATCH /tenant
```

**Auth**: Master key. Requires **admin** or **owner**.

**Body**:

| Field | Type | Description |
|---|---|---|
| `name` | string | New organization name. Max 120 chars. |
| `image_url` | string \| null | Public URL of an avatar image. Send `null` to clear. |

```bash
curl -X PATCH https://api.guayaba.run/api/v1/tenant \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Robotics EU"}'
```

**Errors**: `403 insufficient_tenant_role` if the caller is a member.

---

## Delete the Current Organization

Soft-deletes the organization. Only named organizations can be deleted this way — a user's Personal is permanent.

```
DELETE /tenant
```

**Auth**: Master key. Requires **owner**.

**Body**:

| Field | Type | Description |
|---|---|---|
| `confirm` | string | Must exactly match the organization's current `slug`. |

**Prerequisites** — the organization must first be wound down:

1. No agents (delete every agent first — that will stop and clean each one up).
2. No active API keys (revoke all `g_master_…` and `g_agent_…` keys in the organization, except the one you are calling with; it gets revoked as part of the delete).
3. Subscription cancelled (`status = canceled` or `incomplete_expired`).
4. No `pending` payments.

Anything unmet returns `409` with an `errors[]` array enumerating the offending resources so you can fix them and retry. The top-level `error`/`message` mirror the first blocker. Possible blocker codes: `agents_still_running`, `agents_exist`, `active_api_keys`, `active_subscription`, `pending_payments`.

```bash
curl -X DELETE https://api.guayaba.run/api/v1/tenant \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"confirm": "acme-robotics"}'
```

**Errors**:
- `403 insufficient_tenant_role` — caller is not the owner.
- `409` — one or more prerequisites are unmet; `errors[]` lists them.
- `422 confirmation_mismatch` — `confirm` did not match the current slug.
- `422 cannot_delete_personal_tenant` — the bound organization is Personal.

Deletion is a soft-delete. It is not reversible from the API.

---

## Leave the Current Organization

Removes the caller's membership from the organization and **revokes the invoking master key** in the same transaction. Any subsequent call using that key will fail with `401 api_key_revoked`.

```
POST /tenant/leave
```

**Auth**: Master key. Any role **except** owner. Owners must first transfer ownership or delete the organization.

```bash
curl -X POST https://api.guayaba.run/api/v1/tenant/leave \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "success": true,
  "message": "You have left the organization. This API key has been revoked."
}
```

**Errors**:
- `422 owner_must_transfer_before_leaving` — the caller is the owner. Transfer ownership first.
- `422 cannot_leave_personal_tenant` — a Personal organization cannot be left.

---

## Transfer Ownership

Moves the `owner` role to another existing member. The previous owner becomes an **admin** in the same operation.

```
POST /tenant/transfer-ownership
```

**Auth**: Master key. Requires **owner**.

**Body**:

| Field | Type | Description |
|---|---|---|
| `new_owner_user_id` | uuid | User ID of an existing member of the organization. |
| `confirm` | boolean | Must be exactly `true`. |

```bash
curl -X POST https://api.guayaba.run/api/v1/tenant/transfer-ownership \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "new_owner_user_id": "0e1f2a3b-4c5d-6e7f-8091-a2b3c4d5e6f7",
    "confirm": true
  }'
```

**Errors**:
- `403 insufficient_tenant_role` — caller is not the owner.
- `422 confirmation_mismatch` — `confirm` was not exactly `true`.
- `422` — `new_owner_user_id` is not a member of the organization.

---

## List Members

Returns everyone currently a member of the organization.

```
GET /tenant/members
```

**Auth**: Master key. Any role.

```bash
curl https://api.guayaba.run/api/v1/tenant/members \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "success": true,
  "data": [
    {
      "user_id": "0e1f2a3b-4c5d-6e7f-8091-a2b3c4d5e6f7",
      "name": "Ada Lovelace",
      "email": "ada@acme.example",
      "role": "owner",
      "joined_at": "2026-05-01T12:00:00Z"
    }
  ]
}
```

---

## Change a Member's Role

Sets the target member's role to `admin` or `member`. To move the `owner` role, use [transfer-ownership](#transfer-ownership) instead.

```
PATCH /tenant/members/{userId}
```

**Auth**: Master key. Requires **admin** or **owner**.

**Body**:

| Field | Type | Description |
|---|---|---|
| `role` | string | `admin` or `member`. |

```bash
curl -X PATCH https://api.guayaba.run/api/v1/tenant/members/0e1f2a3b-... \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"role": "admin"}'
```

**Errors**:
- `403 insufficient_tenant_role`.
- `404` — target user is not a member of this organization.
- `422 cannot_modify_owner_membership` — targeting the current owner.

---

## Remove a Member

Removes the target user from the organization.

```
DELETE /tenant/members/{userId}
```

**Auth**: Master key. Requires **admin** or **owner**.

Rules:

- Cannot target the current owner (`422 cannot_modify_owner_membership`).
- Cannot target yourself (`422 use_leave_endpoint_for_self`) — call [`POST /tenant/leave`](#leave-the-current-organization) instead so your key can be auto-revoked.

```bash
curl -X DELETE https://api.guayaba.run/api/v1/tenant/members/0e1f2a3b-... \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

---

## List Pending Invitations

Returns invitations that are still pending — not expired, not revoked, not accepted.

```
GET /tenant/invitations
```

**Auth**: Master key. Requires **admin** or **owner**.

**Response** (`200`):
```json
{
  "success": true,
  "data": [
    {
      "id": "3f1c9b7a-...",
      "role": "member",
      "invited_by": {
        "id": "0e1f2a3b-...",
        "name": "Ada Lovelace",
        "email": "ada@acme.example"
      },
      "expires_at": "2026-07-29T12:00:00Z",
      "created_at": "2026-07-22T12:00:00Z"
    }
  ]
}
```

---

## Create an Invitation

Mints a one-time invitation URL. The token appears in the response **once** — it is not retrievable afterwards.

```
POST /tenant/invitations
```

**Auth**: Master key. Requires **admin** or **owner**.

**Body**:

| Field | Type | Description |
|---|---|---|
| `role` | string | `admin` or `member`. `owner` cannot be invited — use transfer-ownership after acceptance. |

```bash
curl -X POST https://api.guayaba.run/api/v1/tenant/invitations \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"role": "member"}'
```

**Response** (`201`):
```json
{
  "success": true,
  "data": {
    "id": "3f1c9b7a-...",
    "invite_url": "https://app.guayaba.run/invite/inv_5f9b2c1d3e4a4b8c9d0e1f2a3b4c5d6e",
    "role": "member",
    "expires_at": "2026-07-29T12:00:00Z"
  }
}
```

Send `invite_url` to the invitee out-of-band. The link works logged-out (they will see a preview page), expires after 7 days, and can only be redeemed once.

**Errors**:
- `403 insufficient_tenant_role`.
- `422 cannot_invite_to_personal_tenant` — a Personal organization does not support invitations.

---

## Revoke an Invitation

Cancels a pending invitation so the link stops working. Idempotent — revoking an already-revoked, already-accepted, or expired invitation returns `200` with the invitation's current `status`; no error is raised.

```
DELETE /tenant/invitations/{invitationId}
```

**Auth**: Master key. Requires **admin** or **owner**.

```bash
curl -X DELETE https://api.guayaba.run/api/v1/tenant/invitations/3f1c9b7a-... \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Errors**:
- `404` — invitation not found on this organization.

---

## Create a New Organization

Creates a brand-new named organization owned by the caller's user.

```
POST /tenants
```

**Auth**: Master key. Any role in the invoking key's organization.

> ⚠️ **The invoking master key does NOT gain access to the new organization.** Master keys are organization-bound. After creating the organization, switch to it in the dashboard header and mint a fresh master key inside it before calling `/tenant/*` against it.

**Body**:

| Field | Type | Description |
|---|---|---|
| `name` | string | Display name. Required. Max 120 chars. |
| `slug` | string | Optional URL-safe slug. Derived from `name` if omitted. |
| `image_url` | string \| null | Optional avatar URL. |

```bash
curl -X POST https://api.guayaba.run/api/v1/tenants \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Robotics", "slug": "acme-robotics"}'
```

**Response** (`201`):
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Acme Robotics",
    "kind": "organization",
    "slug": "acme-robotics",
    "image_url": null,
    "owner_user_id": "0e1f2a3b-...",
    "member_count": 1,
    "role": "owner",
    "is_default": false,
    "subscription": {
      "tier_id": "b7ef1a30-...",
      "status": "active",
      "started_at": "2026-07-22T12:00:00Z"
    }
  }
}
```

The organization is provisioned with an active Hobby subscription and **no** welcome credits.

**Errors**:
- `409` — the requested slug is already taken.
- `422` — validation error.

---

## Accept an Invitation

Redeems an invitation URL. The caller (`api_keys.user_id`) becomes a member of the invited organization with the role encoded in the invitation.

```
POST /invitations/{token}/accept
```

**Auth**: Master key of any organization the caller belongs to.

`{token}` is the raw token embedded in the `invite_url` (the part after `/invite/`).

```bash
curl -X POST https://api.guayaba.run/api/v1/invitations/inv_5f9b2c1d3e4a4b8c9d0e1f2a3b4c5d6e/accept \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "data": {
    "tenant_id": "550e8400-...",
    "role": "member"
  }
}
```

> The master key you used to accept the invitation still points at its **original** organization. To operate against the newly-joined organization, mint a master key inside it.

**Errors**:
- `404` — token not found.
- `409 invitation_already_accepted` — invitation has already been redeemed.
- `410` — invitation is expired (`invitation_expired`), revoked (`invitation_revoked`), or the organization no longer exists (`tenant_no_longer_exists`).

---

## List Your Memberships

Returns every organization the caller's user belongs to — the only endpoint on the Public API that reads across organizations.

```
GET /user/memberships
```

**Auth**: Master key. Agent keys are rejected with `403`.

```bash
curl https://api.guayaba.run/api/v1/user/memberships \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "success": true,
  "data": [
    {
      "tenant_id": "0e1f2a3b-...",
      "name": "Ada Lovelace",
      "kind": "personal",
      "slug": "ada-lovelace",
      "role": "owner",
      "is_default": true
    },
    {
      "tenant_id": "550e8400-...",
      "name": "Acme Robotics",
      "kind": "organization",
      "slug": "acme-robotics",
      "role": "admin",
      "is_default": false
    }
  ]
}
```

---

## Common Errors

Every route on this page can return one or more of the following error envelopes on top of standard `401 Unauthorized`.

| Status | Error code | When |
|---|---|---|
| `403` | `insufficient_tenant_role` | Caller's role is below the route's minimum. Envelope carries `required` and `have`. |
| `403` | `tenant_access_denied` | The caller's membership on the bound organization has been removed. Mint a new master key elsewhere or ask an admin to invite you back. |
| `409` | (multiple) | Delete-organization prerequisites are unmet — the envelope lists the offending resources (e.g. active subscription, remaining agents, active API keys). |
| `410` | `invitation_expired` / `invitation_revoked` / `invitation_already_accepted` | Attempted to accept or revoke an invitation that is no longer pending. |
| `422` | `confirmation_mismatch` | The `confirm` value did not match the expected sentinel (organization slug for delete, boolean `true` for transfer-ownership). No side effects applied. |
| `422` | `owner_must_transfer_before_leaving` | Owner tried to leave; transfer ownership first. |
| `422` | `cannot_delete_personal_tenant` / `cannot_leave_personal_tenant` / `cannot_invite_to_personal_tenant` | Attempted a named-organization-only action on a Personal organization. |
| `422` | `cannot_modify_owner_membership` | Attempted to update or remove the current owner's membership. |
| `422` | `use_leave_endpoint_for_self` | Attempted to remove yourself via `DELETE /tenant/members/{userId}`; call `POST /tenant/leave` instead. |

# GRAFTs

GRAFTs are reusable agent templates published in the marketplace. Selecting a
GRAFT when creating an agent pre-fills personality, channels, model, and
hardware defaults so users can launch a working agent without filling every
field by hand. The `POST /agents` endpoint accepts an optional `graft_slug`
to apply the template at creation time.

The marketplace is read-only over the public API; authoring and publishing
flows live in the Guayaba Manager UI.

> **Schema contract.** Each GRAFT exposes a single declarative `schema`
> object that drives both the agent-creation wizard and the backend resolver.
> The summary below covers what API consumers need to construct a request.

---

## List GRAFTs

Returns every active GRAFT in the marketplace, optionally filtered by
framework or category.

```
GET /grafts
```

**Auth**: Any key (master or agent).

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `framework` | string | — | Restrict to GRAFTs compatible with this framework slug (e.g. `openclaw`). |
| `category` | string | — | Restrict to GRAFTs in this category slug (e.g. `support`). |
| `tier` | string | — | `free` or `premium`. Unknown values are ignored. |
| `search` | string | — | Case-insensitive substring match against name + short description, plus exact tag match. Blank values are ignored. |
| `sort` | string | `name` | One of `name` (alphabetical), `newest` (most recent first), `popular` (highest `installs_count` first). Non-`name` sorts use `name` as a stable secondary key. |

```bash
curl "https://api.guayaba.run/api/v1/grafts?framework=openclaw&category=support&sort=popular" \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`):
```json
{
  "data": [
    {
      "id": "b5a51000-0000-4000-8000-000000000002",
      "slug": "customer-support-pro",
      "name": "Customer Support Pro",
      "short_description": "Friendly support agent ready for chat and Telegram.",
      "description": "A polished customer-support agent with empathetic tone, escalation guidance and Telegram channel pre-wired. Provide your company name and a Telegram bot token.",
      "author_name": "Guayaba",
      "version": "1.0.0",
      "icon_url": null,
      "cover_image_url": null,
      "tags": ["support", "telegram"],
      "schema": {
        "schema_version": 2,
        "framework_constraints": ["openclaw"],
        "defaults": {
          "channels": ["telegram"],
          "personality": "You are a friendly customer support agent for {{company_name}}. You answer concisely, ask clarifying questions, and escalate when uncertain.",
          "vibe": "Empathetic, concise, escalates when uncertain.",
          "knowledge_seed": ["Refund policy: 30 days."],
          "settings": {
            "model": "anthropic/claude-sonnet-4.6",
            "thinking": "medium",
            "extra_instructions": "Always greet the user warmly with \"Hi, I'm {{agent_persona}} from {{company_name}}!\". Never invent product details — say you'll check."
          }
        },
        "fields": [
          {
            "id": "company_name",
            "label": "Company name",
            "type": "text",
            "required": true,
            "placeholder": "Acme Inc.",
            "help": "Used in greetings and the agent personality.",
            "owned_by": "graft",
            "placement": { "step": "identity", "after": "name", "order": 100 },
            "validation": {
              "min_length": 2,
              "max_length": 80,
              "backend": "required|string|min:2|max:80"
            }
          },
          {
            "id": "agent_persona",
            "label": "How should the agent introduce itself?",
            "type": "text",
            "required": false,
            "placeholder": "Sam",
            "default": "your support assistant",
            "owned_by": "graft",
            "placement": { "step": "identity", "after": "company_name", "order": 110 }
          },
          {
            "id": "telegram_token",
            "label": "Telegram bot token",
            "type": "secret",
            "required": false,
            "binding": "settings.secrets.TELEGRAM_BOT_TOKEN",
            "owned_by": "channel:telegram"
          }
        ]
      },
      "status": "active",
      "tier": "free",
      "price_credits": 0,
      "installs_count": 142,
      "categories": [{ "slug": "support", "name": "Support" }],
      "frameworks": [
        { "id": "0199...", "slug": "openclaw", "name": "OpenClaw" }
      ],
      "created_at": "2026-04-21T10:00:00Z",
      "updated_at": "2026-04-21T10:00:00Z"
    }
  ]
}
```

The list and detail endpoints return identical shapes — the list endpoint
already includes the full `schema`, so most clients don't need to call the
detail endpoint at all.

---

## Show GRAFT

Returns a single GRAFT.

```
GET /grafts/{slug}
```

**Auth**: Any key.

```bash
curl https://api.guayaba.run/api/v1/grafts/customer-support-pro \
  -H "Authorization: Bearer g_master_YOUR_KEY"
```

**Response** (`200`): same shape as the list entry above, wrapped in `{ "data": { … } }`.

`404 Not Found` is returned when the slug does not match any active GRAFT.

> **Fresh-install note.** On a new installation only `blank-agent` is pre-seeded. Example slugs such as `customer-support-pro` shown in this documentation may not exist locally unless they have been uploaded via `graft-cli push` or a staging process.

---

## The `schema` object — what API consumers need

API-consumer summary of the schema keys:

| Key | Type | Description |
|---|---|---|
| `schema_version` | integer | Always `2`. v1 was removed before any client shipped. |
| `framework_constraints` | string[] | Framework slugs this GRAFT is compatible with. |
| `defaults` | object | Agent definition the resolver merges in. Same snake_case keys as the agent definition (`personality`, `vibe`, `knowledge_seed`, `channels`, `llm_provider_model`, `settings.thinking`, `settings.extra_instructions`, `settings.secrets.*`, …). String leaves may contain `{{field_id}}` placeholders. |
| `fields[]` | array | User-facing inputs the GRAFT introduces on top of what the wizard already renders. See below. |
| `steps[]` | array | Optional extra wizard steps the GRAFT contributes. Most GRAFTs leave this empty. |

### Fields

Each `fields[]` entry declares one input. Relevant keys for an API consumer:

| Key | Description |
|---|---|
| `id` | Unique within this GRAFT. Referenced by `{{id}}` tokens inside `defaults` and used as the key in `graft_overrides`. |
| `label` / `placeholder` / `help` | UI hints; safe to ignore when calling the API directly. |
| `type` | One of `text`, `textarea`, `select`, `toggle`, `secret`, `number`, `boolean`, `csv`. `secret` values MUST be supplied via the corresponding `binding` path under `settings.secrets.*`. |
| `required` | Whether the field MUST have a value before the resolver runs. |
| `default` | Value the resolver uses if the request omits it. |
| `binding` | Dot-notation path inside the agent definition where the value lives (e.g. `settings.secrets.TELEGRAM_BOT_TOKEN`). When omitted, the field is **interpolation-only** — the value is captured in `graft_overrides` and used to fill `{{id}}` tokens, but never persisted on the agent itself. |
| `owned_by` | `graft` (the GRAFT renders this field) / `wizard:<id>` (delegated to a native wizard input) / `channel:<slug>` (delegated to the channel's secret form). API consumers can ignore this — it only affects UI rendering. |
| `validation` | Per-field constraints. The relevant key for API consumers is `backend`, a Laravel-style rule string the backend enforces. |
| `placement` | UI-only. Tells the wizard which step / section / position to render the field in. Required when `owned_by == "graft"`. API consumers can ignore this field. |
| `visible_when` / `required_when` | UI-only conditional rendering predicates (`equals`, `not_equals`, `contains`, `not_contains`). API consumers can ignore them. |

> **Authoring a GRAFT?** See the [authoring guide](../guides/authoring-grafts.md) for the full reference on `fields`, `steps`, `placement`, validation, and `{{token}}` interpolation.

### Variable interpolation

Strings inside `defaults` may reference field values with `{{field_id}}`
syntax. The rules:

- The token MUST be a single field `id` declared in `fields[]`. No nested
  paths, no expressions.
- Substitution is single-pass: `{{a}}` → value-of-`a`. Nested chains are not
  followed. Flatten in the field declarations if you need that.
- Missing/empty values **leave the literal token in place** (the API keeps
  the raw `{{company_name}}` rather than producing an empty string). This is
  intentional — it makes unfilled values obvious in the resolved agent
  definition.
- Substitution applies to string leaves only. Arrays of strings are walked
  recursively; numbers / booleans / arrays of non-strings pass through.

The interpolator runs server-side during `POST /agents` regardless of
whether the caller has already resolved variables locally — defence in depth
for non-wizard clients.

---

## Apply a GRAFT when creating an agent

Pass `graft_slug` (and optionally `graft_overrides`) in the `POST /agents`
body. The backend:

1. Loads the GRAFT by slug.
2. Collects per-field values from `graft_overrides.<id>` (primary source)
   and from the request's own `binding` paths (fallback) so bound fields can
   be supplied either way.
3. Runs `TemplateInterpolator::compile(schema.defaults, values)` to resolve
   `{{var}}` tokens.
4. Merges the resolved template with the request payload — **user-supplied
   fields always win** — and records the lineage (graft id + version) on the
   resulting agent.

```bash
curl -X POST https://api.guayaba.run/api/v1/agents \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Support",
    "framework_id": "FRAMEWORK_UUID",
    "graft_slug": "customer-support-pro",
    "graft_overrides": {
      "company_name": "Acme",
      "agent_persona": "Sam"
    },
    "settings": {
      "secrets": {
        "TELEGRAM_BOT_TOKEN": "123456:ABC-..."
      }
    }
  }'
```

Notes:

- `graft_slug` is optional. Omit it (and `graft_overrides`) to create a
  blank agent.
- `name` is required even when a GRAFT is applied.
- `graft_overrides` keys are the GRAFT's `field.id` values (e.g.
  `company_name`) — **not** binding paths. Bound fields may also be
  submitted directly at their `binding` path under `settings.*`; the
  resolver checks `graft_overrides` first, then falls back.
- Fields the GRAFT declares in `defaults` (e.g. `personality`, `channels`,
  `llm_provider_model`) are filled in only when the request omits them.
- The backend re-validates every `field.validation.backend` rule before
  accepting the request — invalid values return `422` with field-level
  error messages.
- `installs_count` on the GRAFT is incremented after a successful create.

---

## Authoring: validate, push, and assets

These three endpoints power the manager UI's "Export GRAFT" modal and the
[`@guayaba/graft-cli`](https://www.npmjs.com/package/@guayaba/graft-cli)
toolkit (`graft validate`, `graft push`). They write into your
account's **personal storage area** — personal grafts are private to your
account and are not published in the marketplace until you submit them for
review.

All three require a **master** API key. Agent-scoped keys are rejected with `403`.
Generate a master key from the manager UI: *Account → API Keys → New master key*.

### Validate a GRAFT envelope

```
POST /grafts/validate
```

Runs the authoritative server-side validator. Returns `200` with warnings
(no errors) or `422` with a field-level `errors` map.

```bash
curl -X POST https://api.guayaba.run/api/v1/grafts/validate \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d @graft.json
```

**Body** — an envelope with `metadata` and `schema`:

```json
{
  "metadata": {
    "slug": "my-graft",
    "name": "My Graft",
    "version": "0.1.0",
    "tags": [],
    "category_slugs": [],
    "framework_slugs": ["openclaw"],
    "tier": "free",
    "price_credits": 0
  },
  "schema": { "schema_version": 2, "defaults": { ... }, "fields": [ ... ] }
}
```

**Response** (`200`):

```json
{ "data": { "valid": true, "warnings": [] } }
```

**Errors**: `422` with `{ "message": "...", "errors": { "<field>": ["..."] } }`.

### Push a GRAFT bundle to personal storage

```
POST /grafts
```

Uploads a complete versioned GRAFT — metadata + schema envelope **plus**
the `graft.tar.gz` bundle produced by `@guayaba/graft-cli`. The bundle must
contain `metadata.json`, `schema.json`, and one `skills/<name>.tar.gz` per
installed skill. The pair `(slug, version)` is **immutable** — re-pushing
the same pair returns `409` with an error on `metadata.version`. Bump the
version field to push again. Slugs are owned per-author: a different user
pushing your slug gets a `409` on `metadata.slug`.

**Wire format**: `multipart/form-data` with three fields.

| Field      | Content                                                                 |
|------------|-------------------------------------------------------------------------|
| `metadata` | JSON-encoded `GraftMetadata` (same shape as `POST /grafts/validate`).   |
| `schema`   | JSON-encoded `GraftSchema`.                                             |
| `bundle`   | `graft.tar.gz` (binary, `application/gzip`). Max 200 MB.                |

```bash
curl -X POST https://api.guayaba.run/api/v1/grafts \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -F "metadata=$(cat metadata.json)" \
  -F "schema=$(cat schema.json)" \
  -F "bundle=@./graft.tar.gz;type=application/gzip"
```

**Response** (`201`):

```json
{
  "data": {
    "id":            "GRAFT_UUID",
    "slug":          "my-graft",
    "version":       "0.1.0",
    "version_id":    "VERSION_UUID",
    "bundle_s3_key": "personal/USER_ID/my-graft/0.1.0/graft.tar.gz",
    "is_personal":   true
  }
}
```

**Errors**:

- `422` — envelope or bundle failed validation. Same shape as `POST /grafts/validate`.
- `409` — either `(slug, version)` already exists for you (`errors.metadata.version`) or the slug is owned by a different author (`errors.metadata.slug`).

### Upload an icon or cover image

```
POST /grafts/{slug}/assets/{type}
```

Where `type` is `icon` or `cover`. The body is `multipart/form-data` with
a single `file` field. A real `PUT` request to the same path is also accepted
for REST clients, but the CLI and manager UI use plain `POST`.

| Type    | Allowed MIME types                              | Max size |
|---------|-------------------------------------------------|----------|
| `icon`  | `image/png`, `image/jpeg`, `image/webp`         | 1 MB     |
| `cover` | `image/png`, `image/jpeg`, `image/webp`         | 4 MB     |

Assets are stored **unversioned** at
`personal/{user_id}/{slug}/{type}.{ext}` — re-uploading replaces the
previous file in place and sweeps any stale extensions for that type
(so switching from `.png` to `.webp` won't leave both behind).

```bash
curl -X POST https://api.guayaba.run/api/v1/grafts/my-graft/assets/icon \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -F "file=@./icon.png"
```

**Response** (`200`):

```json
{
  "data": {
    "slug": "my-graft",
    "type": "icon",
    "path": "personal/USER_ID/my-graft/icon.png"
  }
}
```

**Errors**:

- `422` — file rejected (wrong MIME type, exceeds size limit, slug invalid).
- `404` — slug parameter is malformed (must be kebab-case).

### Recommended ordering

The CLI and UI both upload assets first, envelope last:

1. `POST /grafts/{slug}/assets/icon`   (optional)
2. `POST /grafts/{slug}/assets/cover`  (optional)
3. `POST /grafts`

If any asset upload returns `422`, the envelope `POST` is skipped — this
prevents personal grafts from referencing half-uploaded artwork.

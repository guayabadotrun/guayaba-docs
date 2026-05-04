# Authoring GRAFTs — the declarative schema

A **GRAFT** is a reusable agent template that ships as a single
`graft.tar.gz` bundle. The bundle contains your agent's prose
(personality, vibe, instructions), its skill tarballs, a manifest for
the marketplace, and — at the heart of it — a **declarative schema**
that tells the install wizard which inputs to ask the user for, how to
validate them, and where to drop the answers inside the agent
definition.

This guide is the reference for that schema. It is aimed at GRAFT
authors who want to go beyond what `graft init` generates and shape
their install wizard by hand.

For the request/response surface (apply, validate, push), see
[grafts.md](../api-reference/grafts.md). For the CLI workflow that produces the
bundle, see the [`@guayaba/graft-cli`](https://www.npmjs.com/package/@guayaba/graft-cli)
README.

---

## TL;DR

```jsonc
{
  "schema_version": 2,
  "framework_constraints": ["openclaw"],

  // Author-frozen agent config. String leaves can reference user
  // input via {{field_id}} placeholders.
  "defaults": {
    "personality": "You are {{agent_persona}}, a support rep at {{company_name}}.",
    "vibe": "warm, concise",
    "channels": ["telegram"],
    "settings": {
      "model": "anthropic/claude-sonnet-4.6",
      "thinking": "medium",
      "extra_instructions": "Always greet with: Hi, I'm {{agent_persona}} from {{company_name}}!"
    }
  },

  // User-facing inputs the install wizard will render.
  "fields": [
    {
      "id": "company_name",
      "label": "Company name",
      "type": "text",
      "required": true,
      "placeholder": "Acme Inc.",
      "owned_by": "graft",
      "placement": { "step": "identity", "after": "name", "order": 100 }
    },
    {
      "id": "agent_persona",
      "label": "Agent persona name",
      "type": "text",
      "required": true,
      "default": "Sam",
      "owned_by": "graft",
      "placement": { "step": "identity", "after": "company_name" }
    },
    {
      "id": "telegram_token",
      "label": "Telegram bot token",
      "type": "secret",
      "required": false,
      "binding": "settings.secrets.TELEGRAM_BOT_TOKEN",
      "owned_by": "channel:telegram"
    }
  ],

  // Optional: extra wizard steps & sections this GRAFT contributes.
  "steps": [
    {
      "id": "identity",
      "label": "Identity",
      "sections": [
        { "id": "basics", "title": "Who is this agent?" }
      ]
    }
  ]
}
```

The wizard renders `fields[]` according to their `placement`, validates
the input client- and server-side, interpolates the `{{token}}`s into
`defaults`, merges the result with the request body, and creates the
agent.

---

## Top-level shape

| Key | Required | Type | What it does |
|---|---|---|---|
| `schema_version` | yes | integer | Must be exactly `2`. v1 was removed before any client shipped. |
| `framework_constraints` | no | string[] | Frameworks this GRAFT runs on. Today only `openclaw`. |
| `defaults` | no | object | Author-frozen agent config. Same snake_case keys as the agent definition. String leaves may contain `{{field_id}}` placeholders. |
| `fields` | no | array | User inputs the GRAFT introduces. Render order, validation, binding all live here. |
| `steps` | no | array | Optional UI grouping (extra wizard steps & sections). Pure cosmetics — the resolver ignores them. |

Unknown top-level keys are rejected.

---

## `defaults` — what the GRAFT commits to

`defaults` is the agent definition the GRAFT ships with. At apply
time the resolver:

1. Substitutes every `{{field_id}}` token using user-supplied values.
2. Merges the resolved object with the request payload —
   **request fields always win**.

That means a GRAFT can opinionate everything (personality, model,
channels, secrets…) while still letting the user override individual
keys at create time.

Allowed keys mirror the universal agent fields:

| Key | Type |
|---|---|
| `personality` | string (≤ 10 000 chars on openclaw) |
| `vibe` | string (≤ 2 000 chars on openclaw) |
| `knowledge_seed` | string[] (≤ 100 entries, each ≤ 5 000 chars on openclaw) |
| `channels` | string[] (today only `["telegram"]` is whitelisted) |
| `settings.model` | string (e.g. `"anthropic/claude-sonnet-4.6"`) |
| `settings.thinking` | string (`"off"`, `"low"`, `"medium"`, `"high"`) |
| `settings.extra_instructions` | string (≤ 10 000 chars on openclaw) |
| `settings.secrets.*` | object — usually filled via `{{token}}` from a `secret` field |

Per-framework caps are enforced by the `*FrameworkSchema` (see
`OpenClawFrameworkSchema` for openclaw).

> **Reference integrity.** Every `{{token}}` inside `defaults` MUST
> resolve to a `field.id` declared in `fields[]`. Undeclared tokens
> cause `422` at validate / push time.

---

## `fields[]` — user inputs

Each entry declares one input the install wizard will render (or
delegate). Allowed keys:

```jsonc
{
  // Identity (required)
  "id":    "company_name",       // unique within this GRAFT, snake_case
  "label": "Company name",        // user-facing label
  "type":  "text",               // see "Field types" below

  // Configuration (optional)
  "required":    true,
  "placeholder": "Acme Inc.",
  "help":        "Used inside the personality prompt.",
  "default":     "MyCo",
  "options":     [ { "value": "...", "label": "..." } ],   // select / toggle only

  // Where the value lands (optional)
  "binding":  "settings.secrets.API_KEY",
  "owned_by": "graft",            // "graft" | "wizard:<id>" | "channel:<slug>"

  // Where the input renders (required when owned_by = "graft")
  "placement": {
    "step":    "identity",
    "section": "basics",
    "after":   "name",            // mutually exclusive with "before"
    "order":   100
  },

  // Validation (optional)
  "validation": {
    "min_length": 2,
    "max_length": 80,
    "pattern":    "^[A-Z]",       // regex without delimiters
    "enum":       ["red", "green", "blue"],
    "backend":    "required|string|max:80"   // verbatim Laravel rule
  },

  // Conditional rendering (optional)
  "visible_when":  { "field": "enable_telegram", "equals": true },
  "required_when": { "field": "enable_telegram", "equals": true }
}
```

### Field types

| `type` | Renders as | Notes |
|---|---|---|
| `text` | single-line text input | Default choice for short strings |
| `textarea` | multi-line text input | For long prose. Native wizard fields `wizard:personality` and `wizard:vibe` render as a Markdown editor (write + live preview); custom GRAFT-owned `textarea` fields render as a plain textarea. |
| `secret` | masked input | Never logged. Pair with a `binding` under `settings.secrets.*` |
| `select` | dropdown | Requires `options: [{value, label}, …]` |
| `toggle` | segmented two-way control | Requires `options` (typically two entries) |
| `number` | numeric input | Accepts integers and floats |
| `boolean` | checkbox | `true` / `false` |
| `csv` | text input parsed as array | Comma-separated, trimmed |

### `binding` — where the value lands

A dot-notation path inside the agent definition. The resolver writes
the field's value at this path.

```jsonc
"binding": "settings.secrets.TELEGRAM_BOT_TOKEN"
```

Allowed shape: `/^[A-Za-z_][A-Za-z0-9_]*(\.[A-Za-z_][A-Za-z0-9_]*)*$/`.

When `binding` is **omitted**, the field is **interpolation-only** —
the value is captured for `{{id}}` substitution inside `defaults` but
is never persisted on the agent itself. That's how `company_name` and
`agent_persona` in the example above work: their values weave into
`personality` and `extra_instructions`, but no `agent.company_name`
column ever exists.

### `materialize` — turning a stored secret into a usable credential

Only valid on `type: secret` fields whose `binding` starts with
`settings.secrets.`. Tells the launcher how to convert the stored
secret value into the form the skill actually expects — a credential
file on disk, or a one-shot auth command — on every agent boot and
config reload.

Two shapes:

**`file`** — write the secret to a path:

```jsonc
{
  "id": "notion_api_key",
  "type": "secret",
  "binding": "settings.secrets.NOTION_API_KEY",
  "materialize": {
    "type": "file",
    "path": "~/.config/notion/api_key",
    "mode": "0600",
    "template": "{{value}}"
  }
}
```

**`command`** — pipe the secret into a one-shot setup command:

```jsonc
{
  "id": "github_token",
  "type": "secret",
  "binding": "settings.secrets.GITHUB_TOKEN",
  "materialize": {
    "type": "command",
    "run": ["gh", "auth", "login", "--hostname", "github.com",
            "--git-protocol", "https", "--with-token"],
    "stdin": "{{value}}",
    "timeout_ms": 15000
  }
}
```

`{{value}}` expands to the secret string in `template`, `stdin`, and
each `run` token. The launcher re-runs every `materialize` spec on
`reload-config`, so secret rotations take effect without restarting
the container.

> **`materialize` vs `install.sh`.** Use `materialize` for
> credential-shape conversion (auth files, login commands). Use
> `install.sh` for binary/runtime setup (`apt-get install gh`) that
> must happen before any materialize command can run. Mixing the two
> is a smell: a `materialize.command` that calls a binary not yet on
> `PATH` will fail silently.

### `owned_by` — who renders the field

| Value | Meaning |
|---|---|
| `"graft"` (default) | The GRAFT renders this field. Requires `placement`. |
| `"wizard:<id>"` | The native install wizard already has an input with this id (e.g. `wizard:name`). Merge validation and defaults from this entry, but do **not** render a duplicate. |
| `"channel:<slug>"` | The channel section renders this field (e.g. `channel:telegram` for the bot token). Do not render in the GRAFT section. |

`channel:` is how a GRAFT can pre-populate or customise a channel's
secret form (label, help text, default) without owning the rendering.

### `placement` — where the field shows up

Required when `owned_by == "graft"`. Tells the wizard which step,
section, and position to drop the field into.

| Key | Notes |
|---|---|
| `step` | The wizard step id (e.g. `"identity"`). If the step is not declared in `steps[]`, the wizard creates a synthetic step for the GRAFT. |
| `section` | Optional section id within the step. **Only valid when `step` points to a step you declared in `steps[]`** — native wizard steps don't expose section ids. The validator returns `422` if you pair `section` with a native step; use `after` / `before` / `order` to position fields inside native steps instead. |
| `after` / `before` | Anchor field id. Mutually exclusive. |
| `order` | Tiebreaker integer (lower = earlier). |

### `validation` — per-field constraints

| Key | Notes |
|---|---|
| `min_length` / `max_length` | Integer bounds on the resolved string. |
| `pattern` | Regex string (no delimiters). |
| `enum` | Whitelist of scalar values. |
| `backend` | Laravel rule string evaluated server-side (e.g. `"required\|string\|email"`). The CLI / UI compose `min_length`, `max_length`, `pattern` into the same shape under the hood. |

The backend re-runs every rule before accepting the request, returning
`422` with field-level error messages on failure. Don't rely on
client-side validation alone.

### `visible_when` / `required_when` — conditional rendering

Show or require a field based on another field's value.

```jsonc
{
  "id": "telegram_username",
  "type": "text",
  "label": "Telegram username",
  "owned_by": "graft",
  "placement": { "step": "channels", "after": "enable_telegram" },
  "visible_when":  { "field": "enable_telegram", "equals": true },
  "required_when": { "field": "enable_telegram", "equals": true }
}
```

Supported operators: `equals`, `not_equals`, `contains`, `not_contains`.

---

## `steps[]` — wizard layout

Optional. Declares extra steps & sections the GRAFT contributes to the
install wizard. Steps are **pure layout**: omitting them works fine,
the wizard will create a synthetic GRAFT step that holds every
`owned_by: "graft"` field.

```jsonc
{
  "id":    "identity",      // unique step id
  "label": "Identity",       // user-facing step name
  "after": "basics",         // optional: insert after this step id
  "before": "advanced",      // optional: mutually exclusive with `after`

  "sections": [
    {
      "id":       "basics",
      "title":    "Personal information",
      "subtitle": "Tell us about your company",
      "order":    0
    }
  ]
}
```

Validation:

- `id` and `label` are required.
- `after` and `before` are mutually exclusive.
- `sections[].id` and `sections[].title` are required; `subtitle` and
  `order` are optional.

Fields target a step via `placement.step`. Targeting an undeclared
step is allowed — the wizard creates a synthetic step on the fly.

---

## Variable interpolation — `{{token}}` rules

The same interpolation engine runs server-side and client-side, so
the live preview in the install wizard matches the final agent
definition exactly.

- Token shape: `{{` + optional whitespace + `field_id` + optional whitespace + `}}`. The `field_id` must match `/^[A-Za-z_][A-Za-z0-9_]*$/`.
- The id MUST be declared in `fields[]`. Otherwise `422` at validate / push time.
- Substitution is **single-pass**. `{{a}}` resolving to a value containing `{{b}}` is **not** followed.
- Substitution applies to **string leaves only**. Arrays of strings are walked recursively; numbers, booleans, and arrays of non-strings pass through unchanged.
- Missing or empty values **leave the literal token in place** so unfilled gaps are obvious in the resolved agent definition.
- If a value is a non-scalar (array / object), the literal token is preserved — JSON never leaks into a string.

```jsonc
// defaults
{ "personality": "Hello {{company_name}}! I work for {{agent_persona}}." }

// graft_overrides
{ "company_name": "Acme", "agent_persona": "Sam" }

// resolved
{ "personality": "Hello Acme! I work for Sam." }
```

---

## How the install wizard maps schema → UI

1. **Defaults seed the form.** Each field's `default` (or its declared
   value inside `defaults` for bound fields) pre-fills the input.
2. **`fields[]` become inputs.** `type` selects the React control;
   `label`, `placeholder`, `help`, `required` decorate it. Validation
   errors from the backend's 422 response surface under the field.
3. **`owned_by` decides who renders.** `graft` → GRAFT section.
   `wizard:<id>` → merge into a native wizard input. `channel:<slug>`
   → defer to that channel's secret form.
4. **`placement` decides where.** `step` + `section` + `after` /
   `before` + `order` resolve to a position inside the wizard
   hierarchy. Undeclared steps create a synthetic step automatically.
5. **`{{token}}`s preview live.** As the user types, the wizard
   re-interpolates `defaults` so the personality / instructions panel
   shows the resolved prompt.
6. **Submit triggers the same merge server-side.** The frontend
   resolution is just preview — the backend re-runs the
   interpolator and `GraftResolverService::merge()` as the source of
   truth.

---

## Built-in wizard ids

The install wizard ships with a fixed set of steps, native inputs, and
channel slots that GRAFT authors can target by id.

### Default steps (in render order)

| `step` id | Label | What it covers |
|---|---|---|
| `identity` | Identity | Agent name, vibe, personality prose |
| `personality` | Knowledge (RAG) | Seed knowledge entries |
| `model` | LLM & capabilities | Model choice, thinking level, included tools |
| `channels` | Communication channels | Per-channel toggles + secret forms |
| `_review` | (auto) | Final review screen — synthetic, do not target |

Sections inside these steps are not addressable by id — if you set
`placement.section` while pointing `placement.step` at a native step,
the validator rejects with `422`. Use `placement.after` / `before` /
`order` to position fields inside native steps. Sections **are**
addressable when the step is one you declared yourself in `steps[]`
(see [`steps[]` — wizard layout](#steps--wizard-layout)). If you target
an undeclared step id, the wizard creates a synthetic step labelled
**"Template setup"** at the front of the flow, holding every
unanchored `owned_by: "graft"` field.

### Native field ids (for `owned_by: "wizard:<id>"`)

These are the inputs the wizard renders itself. Targeting one merges
your validation / default into the existing input instead of rendering
a duplicate.

| `owned_by` | Step | Type | Notes |
|---|---|---|---|
| `wizard:name` | identity | text | |
| `wizard:vibe` | identity | textarea | Renders as a Markdown editor (compact, ~160 px) |
| `wizard:personality` | identity | textarea | Renders as a Markdown editor (tall, ~280 px); maps to `SOUL.md` |
| `wizard:knowledge_seed` | personality | textarea-list | |
| `wizard:model` | model | select | |
| `wizard:thinking` | model | radio-cards | |
| `wizard:channels` | channels | toggle-list | |

### Channel slugs (for `owned_by: "channel:<slug>"`)

| `owned_by` | Where it renders |
|---|---|
| `channel:telegram` | Channels step → Telegram secret form |

A field flagged with `channel:<slug>` is rendered by that channel's
own form (not by the GRAFT section). Use this to pre-label, pre-fill,
or constrain a channel secret without owning its layout.

> **`chat` is not a channel.** The web-chat transport is always on and
> has no secret form. `channel:chat` is not a valid value and will be
> silently ignored by the wizard. Only `telegram` is a supported
> channel slug today.

---

## Adding new fields & sections — the workflow

### Add a new user input

1. Pick a snake_case `id`. It is the key in `graft_overrides` and the
   token used inside `defaults`.
2. Choose the `type` and add `options` if needed.
3. Decide whether it persists on the agent (`binding`) or only feeds
   interpolation (no `binding`).
4. Decide who renders it (`owned_by`). For new GRAFT-owned inputs add
   a `placement` block.
5. Add `validation` constraints. At minimum, set `required` and a
   `max_length` for free-text fields.
6. If the value is supposed to appear inside the personality / vibe /
   instructions, add `{{your_id}}` to the relevant string in `defaults`.

### Add a new wizard section

1. Add a `steps[]` entry (or extend an existing one with a new
   `sections[]` entry).
2. Reference the new `step` / `section` id from the relevant fields'
   `placement` blocks.
3. Use `order` to control the in-section order; `after` / `before` are
   handy for inserting next to a known anchor field.

### Verify

```bash
graft validate --framework openclaw -i ./graft
```

The CLI POSTs the envelope to `/api/v1/grafts/validate`, which is the
authoritative validator. All the rules described in this document are
enforced there. A `200` means the schema is well-formed; `422`
returns a per-field `errors` map you can act on.

---

## Common validation failures

| Error | Cause |
|---|---|
| `schema_version must be exactly 2` | Top-level key wrong / missing. |
| `fields[N].id` is duplicated | Two fields share an id. |
| `fields[N].type must be one of: …` | Unknown / typo'd type. |
| `fields[N].options is required for select / toggle` | Missing `options` on a choice-style field. |
| `fields[N].binding has invalid shape` | Not a dot-separated identifier path. |
| `fields[N].owned_by must start with graft \| wizard: \| channel:` | Unknown ownership prefix. |
| `fields[N].placement may set after OR before, not both` | Both anchors set. |
| `fields[N].placement.section requires step to point to a step declared in this schema's steps[]` | You set `section` while pointing `step` at a native wizard step. Drop `section` and use `after` / `before` / `order`, or declare your own step in `steps[]`. |
| `defaults references undeclared variables: <id>` | A `{{token}}` in `defaults` has no matching field. |
| `defaults.channels[N] is not a permitted GRAFT channel` | Today only `telegram` is whitelisted. |
| Field-cap exceeded (e.g. `personality > 10000 chars`) | Hits the `*FrameworkSchema` per-framework caps. |

---

## Bundle layout the schema lives in

`graft.tar.gz` (produced by `graft pack` / `graft push`):

```
graft.tar.gz
├── metadata.json          ← marketplace manifest (slug, version, name, …)
├── schema.json            ← the document described in this guide
├── README.md              ← user-facing description
├── TOOLS.md               ← optional: injected into the agent's TOOLS.md
├── install.sh             ← optional: first-apply binary setup hook
└── skills/                ← only present if the GRAFT bundles ≥1 skill
    ├── <name>.tar.gz
    ├── <name>.manifest.json
    └── …
```

### `TOOLS.md` — shipping tool notes with your GRAFT

If your workspace has a top-level `TOOLS.md`, `graft init` copies it
into the scaffold and `graft push` includes it in the bundle. On first
boot the launcher **injects it as a managed block** into the new
agent's own `TOOLS.md`:

```
<!-- BEGIN MANAGED:graft:tools -->
… your TOOLS.md content …
<!-- END MANAGED:graft:tools -->
```

The agent's notes outside that block are preserved across re-applies,
so the agent can extend `TOOLS.md` without losing your contribution.

Use `TOOLS.md` to document which CLIs the GRAFT expects (`gh`, `jq`,
…), which env vars back which command, or any host/account/path the
skills assume. Skip it if you have nothing to say — it is optional.

### Scaffold sidecars (`personality.md`, `vibe.md`, `extra_instructions.md`)

`graft init` creates **empty stub files** for each prose field — it
does NOT pre-fill them with the workspace content. You must write the
template text yourself (and drop any `{{placeholder}}` tokens for the
user to fill in). The stubs start with an HTML comment that reminds
you which workspace file each one maps to; delete it before pushing.

| Scaffold file | Maps to workspace | `defaults` key |
|---|---|---|
| `personality.md` | `SOUL.md` | `personality` |
| `vibe.md` | `IDENTITY.md` | `vibe` |
| `extra_instructions.md` | `AGENTS.md` | `settings.extra_instructions` |

Skills and `TOOLS.md` are copied verbatim from the workspace;
`install.sh` is also copied if present.

---

## What happens at first boot (the full apply sequence)

When a user creates an agent from your GRAFT, the agent's launcher
runs the apply sequence **once**, in this order:

1. **Schema is merged.** `defaults` (with all `{{token}}` substitutions)
   are written into the agent definition. Skills are copied to
   `workspace/skills/`.

2. **`TOOLS.md` is injected** (if the bundle contains one). The launcher
   inserts it as a managed block inside the agent's own `TOOLS.md`,
   preserving anything the agent has written outside that block.

3. **`install.sh` runs** (if present). The script receives a working
   directory pointing at the extracted bundle and runs as root. This is
   where you install system binaries the skills depend on. The entire
   apply fails (and is retried on next boot) if the script exits non-zero.

4. **`materialize` specs are persisted** from every `secret` field that
   declares one. The specs are stored to disk; they run now and on every
   `reload-config` from this point forward.

5. **The `.graft-applied` marker is written.** All future boots skip
   steps 1–4. Delete the marker to force a full re-apply (e.g. for GRAFT
   upgrades — this is not yet a user-facing feature, but good to know for
   debugging).

The apply sequence is **transparent to the user** — it happens in the
background while the agent container starts. The user sees the agent
status flip from *pending* to *running* when the sequence completes.

---

## Optional: shipping an `install.sh` hook

If your bundled skill needs a CLI binary on `PATH` (think `gh`, `aws`,
`terraform`), drop an executable `install.sh` at the root of your
workspace before `graft push`. The CLI ships it inside the bundle and
the launcher runs it **once**, on the first agent boot, before any
chat traffic.

```sh
#!/bin/sh
# install.sh — make `gh` available so the bundled `github` skill works.
set -e
command -v gh >/dev/null && exit 0

apt-get update
apt-get install -y --no-install-recommends gh
# Persist across container restarts (see launcher AGENTS.md).
echo "gh" >> "/mnt/efs/agents/$AGENT_ID/.deps/apt-packages.txt"
```

Contract you can rely on:

- The script runs as **root** in an isolated container — `apt-get
  install -y` works without `sudo`.
- The working directory is the bundle's extract dir, which contains
  `metadata.json`, `schema.json`, your `skills/`, and any other
  top-level files you shipped.
- stdout streams to launcher logs at `info`, stderr at `warn`.
- Hard timeout: **5 minutes**. Past that, SIGKILL → apply fails →
  next boot retries.
- A non-zero exit aborts the apply (the `.graft-applied` marker is
  not written). Use `set -e` and exit cleanly.
- The hook MUST be idempotent. Re-installs of the same agent (and
  future re-apply tooling) will invoke it again.

When **NOT** to use `install.sh`:

- Per-user secrets → declare a `secret` field in `schema.json` with a
  `materialize` block (see the secrets section above). `materialize`
  re-runs on every boot/reload to follow rotations; `install.sh` runs
  only once, so it can't carry credentials cleanly.
- Pure prompt / personality changes → those belong in `SOUL.md` /
  `IDENTITY.md` / `AGENTS.md`, which the launcher already reapplies
  through managed slots.

If you're unsure whether a tool deserves a hook at all: skip it. The
agent has full root inside its container and can `apt-get install …`
on demand the first time it needs the missing CLI. `install.sh` is the
zero-touch optimisation for that path, not a requirement.

---

## See also

- [grafts.md](../api-reference/grafts.md) — request/response API for apply,
  validate, push, and assets.
- [`@guayaba/graft-cli` README](https://www.npmjs.com/package/@guayaba/graft-cli) —
  the toolchain that produces and uploads bundles.
- [introduction.md](../api-reference/introduction.md) — auth & API basics.

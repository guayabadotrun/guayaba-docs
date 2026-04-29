# AGENTS.md — guayaba-docs

## What this is

The **user-facing public documentation** for the Guayaba platform, structured for GitBook sync. This is the canonical source for end-user docs — API reference, getting started guides, and GRAFT authoring. It is NOT the internal team documentation (that lives in `gene-seed/internal/`).

## Repo structure

| Path | Contents |
|---|---|
| `README.md` | Landing page (what is Guayaba, links to sections) |
| `SUMMARY.md` | GitBook table of contents — edit this when adding pages |
| `getting-started/quick-start.md` | Create first agent, connect Telegram |
| `getting-started/concepts.md` | Agents, sessions, GRAFTs, credits, API keys explained |
| `api-reference/introduction.md` | Base URL, quick start curls, OpenAPI link |
| `api-reference/authentication.md` | Key types (master/agent), scopes, subscription rules |
| `api-reference/agents.md` | Agent CRUD, runtime, sessions, archives, channels, files, catalogs |
| `api-reference/chat.md` | Chat endpoint: sync + SSE streaming, sessions, tool calls |
| `api-reference/grafts.md` | Marketplace API + authoring (validate, push, assets) |
| `api-reference/billing.md` | Credits, costs, subscription |
| `api-reference/errors.md` | All HTTP error codes with examples |
| `guides/authoring-grafts.md` | Full schema v2 reference: fields, types, placement, validation |

## How to contribute

1. **Edit existing content** — edit the file directly and commit to `master`.
2. **Add a new page** — create the `.md` file, then add it to `SUMMARY.md` in the right position (GitBook uses SUMMARY.md as its TOC).
3. **Keep internal docs out** — architecture decisions, inter-service contracts, and technical debt go in `gene-seed/internal/`, not here.

## Key rules

- **English only.** Every word in this repo must be in English.
- **User-facing tone.** Write for developers using the API, not for platform maintainers. Avoid internal project names or ECS/VPC implementation details.
- **Schema examples must be valid.** The canonical GRAFT channel is `["telegram"]` — never include `"chat"` in `defaults.channels[]`. Field ids use `snake_case` (e.g. `"id": "company_name"`, not `"key": ...`).
- **Keep in sync with convolution-api.** The source of truth for API contracts is `gene-seed/internal/contracts/inter-service-api.md` and the convolution-api source. When the backend changes, update docs here.
- **Do not duplicate gene-seed/internal/.** If something is only relevant to maintainers/agents, it belongs in gene-seed.

## Source of truth for API contracts

`gene-seed/internal/contracts/inter-service-api.md` — verified against source code.
Public API OpenAPI spec: `convolution-api/docs/features/openapi-v1.yaml`.

# AGENTS.md — guayaba-docs

## Quick Reference

**What this is:** User-facing public documentation for the Guayaba platform, structured for GitBook sync.

**Critical rules:**
- **English only.** Every word in this repo must be in English.
- **Keep internal docs out.** Architecture decisions, contracts, and debt go in `gene-seed/internal/`, not here.
- **Schema examples must be valid.** The canonical GRAFT channel is `["telegram"]` — never include `"chat"` in `defaults.channels[]`.
- **Keep in sync with `convolution-api`.** The source of truth for API contracts is `gene-seed/internal/contracts/` and the backend source.
- **Update `SUMMARY.md` when adding pages.** GitBook uses it as its table of contents.

> Platform map: `gene-seed/AGENTS.md`

---

## Full Reference

### Repo structure

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
| `api-reference/manager-api.md` | Dashboard API (Sanctum-authenticated observability endpoints) |
| `guides/authoring-grafts.md` | Full schema v2 reference: fields, types, placement, validation |
| `guides/workflow-agent-piece.md` | Using the Guayaba workflow piece to control agents from a workflow |

### How to contribute

1. **Edit existing content** — edit the file directly and commit to `master`.
2. **Add a new page** — create the `.md` file, then add it to `SUMMARY.md` in the right position.
3. **Keep internal docs out** — architecture decisions, inter-service contracts, and technical debt go in `gene-seed/internal/`, not here.

### Source of truth for API contracts

`gene-seed/internal/contracts/inter-service-api.md` — verified against source code.
Public API OpenAPI spec: `convolution-api/docs/features/openapi-v1.yaml`.

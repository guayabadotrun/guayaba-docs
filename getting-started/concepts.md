# Core Concepts

A brief reference for the main ideas you'll encounter across Guayaba.

---

## Agents

An agent is the central unit. It has:

- **Identity** — a name, personality, and vibe (tone/style).
- **Knowledge** — a set of seed entries the agent always has in context.
- **Model** — the LLM provider, model name, and thinking level.
- **Channels** — external communication surfaces such as Telegram. Web chat is built into the runtime and is not a GRAFT channel.
- **Tools / Skills** — capabilities like web search, code execution, or custom API calls.

Agents run in isolated containers and can be started, stopped, paused, and reloaded independently.

The dashboard includes a **Resources** tab on each agent and an **Observability** section for monitoring everything available to your account. In production, infrastructure graphs come from AWS Container Insights. In local or non-AWS environments, infrastructure graphs may be unavailable, but LLM usage can still be shown from runtime usage events.

The Hobby plan includes 1 hour of combined agent running time per UTC day. Runtime limits use an effective counter: closed sessions that have already been saved plus live time from agents that are currently starting or running.

### Agent status

| Status | Meaning |
|---|---|
| `created` | Exists but has never been started |
| `starting` | Provisioning or booting |
| `running` | Live and accepting messages |
| `paused` | Container alive, not processing messages |
| `stopped` | Container shut down |
| `failed` | Crashed or failed to start |

---

## Sessions

A session is a conversation thread. Each chat message belongs to a session. Sessions let the agent remember context across multiple turns.

- Sessions created via the API have an `api:` prefix.
- Sessions created via the web UI have a `s-` prefix.
- You can list, rename, archive, and delete sessions from the dashboard or via the API.

---

## GRAFTs

A GRAFT is a reusable agent template. It bundles a pre-configured agent definition (personality, model, channels, skills) with a declarative schema that defines the inputs a user needs to supply at install time — things like a company name, a bot token, or a custom instruction.

When you create an agent from a GRAFT:
1. The GRAFT's defaults fill in any fields you don't override.
2. Any `{{placeholder}}` tokens in the personality or instructions are replaced with the values you provided.
3. Skills bundled in the GRAFT are installed in the agent's container.

GRAFTs live in the marketplace. You can browse them in the dashboard or access them via the API.

**Authors**: see [Authoring GRAFTs](../guides/authoring-grafts.md) for how to build and publish your own.

---

## Credits

Credits are the billing unit for resource-intensive operations. Base operation credits are checked before execution; if your balance is insufficient, the request is rejected with `402`. Model-dependent LLM usage can be billed later after runtime usage data is processed.

| Operation | Cost |
|---|---|
| Create agent | 10 credits |
| Update agent | 10 credits |
| Start agent | 5 credits |
| Reload config | 1 credit |
| Chat message | 1 credit (base) + LLM usage |
| Stop / Pause / Delete | Free |
| All read operations | Free |

Credits come from your paid subscription allowance (renewed each billing period) plus any top-up purchases. New accounts receive a one-time welcome grant of **2,000 credits** on the Hobby plan. LLM usage credits are charged asynchronously after provider cost data is ready, so a very recent chat can appear in token charts before its cost is reflected in your balance. Check your balance at any time from **Settings → Billing** or via `GET /api/v1/billing/credits`.

---

## API Keys

Guayaba has two key types:

### Master Key (`g_master_…`)

One active master key for your backend account context. It has full access to every available agent and endpoint in that context. Required for creating agents, managing billing, and creating agent keys. Revoking a master key cascades to all related agent keys.

### Agent Key (`g_agent_…`)

One per agent. Scoped to a single agent. You choose the permissions when you create it:

| Scope | What it allows |
|---|---|
| `agent:read` | View agent details (always included) |
| `agent:manage` | Start, stop, pause, reload; manage sessions and archives |
| `channels` | Manage Telegram pairing |
| `files` | Upload files to the agent |
| `chat` | Send messages via the API |

Use agent keys to give third-party systems or automations the minimum access they need.

> Keys are shown **once** at creation. Store them securely — lost keys can only be regenerated (which revokes the old one).

---

## Frameworks

A framework is the runtime environment that hosts your agent. It defines what tools are available, what model parameters are accepted, which external channels can be connected, and which runtime capabilities are supported, such as chat, sessions, files, logs, reload, and GRAFT export.

Currently, **OpenClaw** is the only available framework. It supports Telegram, web chat, sessions, workspace files, logs, config reload, and a built-in set of tools (web search, code execution, file access, and more).

---

## Skills

Skills are packaged tool sets that extend what your agent can do. A skill is a tarball containing tool definitions, supporting code, and optional binary dependencies. Skills are installed in the agent's container at boot.

You can write your own skills and bundle them in a GRAFT, or install them manually on an agent via the dashboard.

# Quick Start

Get a working AI agent up and running in under 5 minutes.

## 1. Create an account

Go to [app.guayaba.run](https://app.guayaba.run) and sign up. Your account is automatically enrolled on the **Hobby** plan — no credit card required. You'll receive 2,000 welcome credits to get started.

Hobby includes 1 hour of combined agent running time per UTC day. You can see your remaining daily runtime in the dashboard.

## 2. Create your first agent

From the dashboard, click **New Agent**. You'll go through a short wizard:

1. **Choose a framework** — Select **OpenClaw** (the default, and the only framework currently available).
2. **Pick a starting point** — You can start from a blank agent or select a GRAFT from the marketplace. GRAFTs are pre-configured templates; a blank agent gives you a clean slate.
3. **Identity** — Give your agent a name and describe its personality and vibe.
4. **Knowledge** — Optionally add seed knowledge entries (facts, policies, context your agent should always remember).
5. **Model** — Choose the LLM and thinking level.
6. **Channels** — Enable Telegram if you want to connect a bot (you'll need a bot token from [@BotFather](https://t.me/BotFather)).
7. **Review and create.**

## 3. Start the agent

On the agent detail page, click **Start**. It takes a moment to provision. The status moves from `created` → `starting` → `running`.

**Cost**: 5 credits.

## 4. Chat with your agent

Once the agent is `running`, open the **Chat** tab. Type a message and press Enter.

You can also chat programmatically using the API:

```bash
# First, create an API key in Settings → API Keys
curl -X POST https://api.guayaba.run/api/v1/agents/{id}/chat \
  -H "Authorization: Bearer g_master_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello! What can you help me with?"}'
```

## 5. Connect Telegram (optional)

If you enabled the Telegram channel during setup:

1. Open a Telegram chat with your bot.
2. Send `/start` — the bot will reply with a pairing code.
3. Go to the agent's **Channels** tab in the dashboard and approve the pairing code.

Your Telegram users can now chat with the agent directly.

## What's next?

- **[Core Concepts](concepts.md)** — Understand agents, sessions, GRAFTs, and credits.
- **[Observability](observability.md)** — Track runtime, resource metrics, and LLM usage.
- **[API Reference](../api-reference/introduction.md)** — Automate everything programmatically.
- **[Authoring GRAFTs](../guides/authoring-grafts.md)** — Package and publish your agent as a reusable template.

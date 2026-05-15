# Guayaba AI Agent — Workflow Piece

The **Guayaba AI Agent** workflow node lets you interact with your agents from inside a workflow. You can send messages, start and stop agents, and trigger a workflow automatically when an agent boots — all without any additional authentication.

---

## Actions

### Send Message

Sends a chat message to a running agent and returns the agent's reply.

**Inputs:**

| Field | Type | Description |
|---|---|---|
| Agent | Dropdown | The agent to message. The agent must be in **running** status — start it from the Agents dashboard before executing this workflow, or add a **Start Agent** step earlier in the workflow. |
| Message | Text | The message to send. Supports `{{token}}` references to previous step outputs. |
| Session Key (optional) | Text | An existing session key to continue a conversation. If omitted, a new session is created each run. |

**Output:**

```json
{
  "sessionKey": "api:uuid",
  "message": { "role": "assistant", "content": "The agent's reply." }
}
```

**Error — agent not running:**

If the agent is stopped or paused, the step returns a 409 error with a message like:

> Agent "My Agent" is not available: Agent is not running. Start it from the Agents dashboard before running this workflow.

Add a **Start Agent** step before **Send Message** to handle this automatically.

---

### Start Agent

Starts a stopped or failed agent. The step **waits** for the agent to become ready before continuing — agent boot typically takes 3–5 minutes.

**How it works:**

The step fires the boot request and then pauses the workflow. The platform checks the agent's status approximately every minute and resumes the workflow automatically when the agent is running. The workflow's **Run** button shows a **Waiting…** spinner while this is in progress.

If the agent is not running after **10 minutes**, the workflow run is marked as failed.

**Inputs:**

| Field | Type | Description |
|---|---|---|
| Agent | Dropdown | The agent to start. |

**Output:**

```json
{ "agentId": "uuid", "status": "running" }
```

---

### Stop Agent

Stops a running agent. Safe to call if the agent is already stopped — the step is idempotent.

**Inputs:**

| Field | Type | Description |
|---|---|---|
| Agent | Dropdown | The agent to stop. |

**Output:**

```json
{ "agentId": "uuid", "status": "stopped" }
```

---

### Get Agent / List Agents

Read-only helpers for fetching agent metadata. Useful when you need agent details downstream in the workflow.

---

## Trigger: On Agent Start

Fires when a Guayaba agent transitions to **running** status (i.e. after a successful boot). Use this trigger to run automated tasks the moment an agent becomes available — for example, posting a notification to Slack or logging the event.

**Configuration:**

| Field | Type | Description |
|---|---|---|
| Agent | Dropdown | The agent to watch. |

**Payload available in subsequent steps:**

```json
{
  "agent_id": "uuid",
  "agent_name": "My Agent",
  "started_at": "2026-05-15T10:00:00+00:00"
}
```

**Note:** The trigger fires at most once per boot. If a run triggered by **On Agent Start** is still in progress when the agent boots again, the new event is dropped (concurrency guard).

---

## Delay workflow node and longer waits

The **Delay** workflow node (`delay-for-action`, `delay-until-action`) now supports delays of any duration — not just delays under 10 seconds. The workflow pauses and resumes automatically via the same scheduler-based mechanism as **Start Agent**. Minimum granularity is approximately 1 minute.

---

## Authentication

No configuration is required. The Guayaba AI Agent workflow node authenticates automatically using your account credentials — you will never see an API key or connection prompt when using this workflow node.

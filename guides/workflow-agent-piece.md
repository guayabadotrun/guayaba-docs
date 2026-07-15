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
| Session Key (optional) | Text | An existing session key to continue a conversation. Must match the format `api:<identifier>` (alphanumeric and hyphens only, max 255 characters). If omitted, a new session is created each run. |

**Output:**

```json
{
  "sessionKey": "api:uuid",
  "content": "The agent's reply."
}
```

**Error — agent not running:**

If the agent is stopped or paused, the step returns a 409 error with a message like:

> Agent "My Agent" is not available: Agent is not running. Start it from the Agents dashboard before running this workflow.

Add a **Start Agent** step before **Send Message** to handle this automatically.

---

### Start Agent

Starts a stopped or failed agent. The step **waits** for the agent to become ready before continuing — typical boot times range from a couple of minutes (warm path) to several minutes for cold containers or first-boot GRAFT applies. The hard upper bound is 10 minutes (see below).

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

Stops a running agent. Safe to call if the agent is already `stopping` or `stopped` — the step is idempotent.

**Inputs:**

| Field | Type | Description |
|---|---|---|
| Agent | Dropdown | The agent to stop. |

**Output:**

```json
{
  "agentId": "uuid",
  "previousStatus": "running",
  "status": "stopping"
}
```

The immediate response `status` is `stopping` (the container is being drained). The agent settles to `stopped` a few seconds later once the platform finalizes the shutdown. For an already-stopped or failed agent, `status` echoes the current terminal state.

---

### Get Agent

Returns a single `{ id, name, status }` record for the selected agent. (Internally it calls the same agent-list endpoint and filters by id; it does NOT return personality, model, settings, or other agent definition fields.)

**Inputs:**

| Field | Type | Description |
|---|---|---|
| Agent | Dropdown | The agent to look up. |

**Output:**

```json
{ "id": "uuid", "name": "My Agent", "status": "running" }
```

### List Agents

Returns every agent visible to your account.

**Output:**

```json
{
  "agents": [
    { "id": "uuid", "name": "My Agent", "status": "running" }
  ]
}
```

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

> The workflow editor's test/sample data adds an extra `status: "running"` field for convenience, but real **On Agent Start** webhook deliveries only include the three fields above. Do **not** reference `{{trigger.status}}` from downstream steps — it will be `undefined` at runtime.

**Note:** The trigger fires at most once per boot. If a run triggered by **On Agent Start** is still in progress when the agent boots again, the new event is dropped (concurrency guard).

---

## Delay workflow node and longer waits

The **Delay** workflow node (`delay-for-action`, `delay-until-action`) now supports delays of any duration — not just delays under 10 seconds. The workflow pauses and resumes automatically via the same scheduler-based mechanism as **Start Agent**. Minimum granularity is approximately 1 minute.

---

## Authentication

No configuration is required. The Guayaba AI Agent workflow node authenticates automatically using your account credentials — you will never see an API key or connection prompt when using this workflow node.

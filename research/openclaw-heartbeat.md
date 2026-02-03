# OpenClaw Heartbeat — Research Summary

## What is OpenClaw?

OpenClaw (formerly Moltbot/Clawdbot) is an open-source, self-hosted AI assistant that connects to messaging platforms (Telegram, Discord, Slack, Signal, WhatsApp, iMessage). Its creator is Peter Steinberger. The key differentiator from typical AI assistants: OpenClaw can act **proactively**, not just reactively.

---

## The Heartbeat Concept

The Heartbeat is a mechanism that lets an OpenClaw agent **wake itself up on a schedule**, run a turn in the main session, and decide whether anything needs the user's attention. It transforms the agent from a passive responder into something closer to an autonomous monitor.

**Core loop:**

1. Timer fires (default every 30 minutes).
2. Agent receives a system prompt telling it to read the workspace context (including `HEARTBEAT.md` if present).
3. Agent evaluates the situation.
4. If nothing needs attention → replies with `HEARTBEAT_OK` (silently suppressed).
5. If something is urgent → replies with alert text (delivered to the user).

---

## How It Actually Works — Session Architecture

### The gateway is a persistent daemon

OpenClaw runs a gateway process (background service) that stays alive 24/7. This gateway maintains long-lived sessions. Each agent has **one main session**, identified by a `sessionKey` like `agent:main:default`.

Each session has three components:

1. **A transcript** — a JSONL file on disk (`~/.openclaw/agents/<agentId>/sessions/`) recording every message, tool call, and response.
2. **A queue** — incoming messages wait here if the agent is already busy with a turn.
3. **An in-memory context** — the transcript loaded into memory, trimmed of old tool results, sent to the LLM on each turn.

### Normal message flow

```
User sends "check my email"
    → gateway routes it to sessionKey agent:main:default
    → message enters the queue
    → no run active → starts an agent turn
    → gateway loads the transcript (prior conversation history)
    → sends it all to the LLM as context + the new message
    → LLM responds, may call tools, responds again
    → turn ends, transcript updated on disk
```

### Heartbeat message flow

```
30 minutes pass, heartbeat-runner.ts timer fires
    → constructs a synthetic "user message" (the heartbeat prompt)
    → enqueues it into the SAME queue for agent:main:default
    → if a run is already active → SKIP, retry next tick
    → if queue is free → start an agent turn (identical to a normal message)
    → gateway loads the SAME transcript (full conversation history)
    → sends to LLM: [all prior context] + [heartbeat prompt]
    → LLM sees everything — earlier requests, pending tasks, prior tool results
    → LLM replies HEARTBEAT_OK or an alert
    → turn ends, gateway RESTORES the previous updatedAt timestamp
```

The heartbeat is not a separate process or a webhook. It is literally another turn in the same conversation, scheduled by the gateway's internal timer.

### Why the full session context matters

The heartbeat prompt says "read HEARTBEAT.md and check if anything needs attention." But the LLM doesn't see that in isolation. It sees the entire conversation:

```
[system prompt]
[user]: deploy the fix to staging and let me know when it's done
[assistant]: deploying now... done, deployed to staging at 2:14pm
[user]: great, I'm heading to lunch. ping me if anything breaks
[assistant]: will do
... 45 minutes pass ...
[user]: <heartbeat prompt: read HEARTBEAT.md, check if anything needs attention>
```

The model can now reason: "The user deployed to staging and asked me to watch for breakage. Let me check the status endpoint." It has the full conversational context to understand what "anything" means.

### The updatedAt rollback trick

Sessions have an `updatedAt` field that controls idle expiry. After a heartbeat turn, the gateway **rolls back** `updatedAt` to its previous value. This prevents a heartbeat running every 30 minutes from keeping a session alive indefinitely just by acking `HEARTBEAT_OK`. Without this, quiet sessions would never expire.

### Queue modes

If a message arrives while the agent is mid-turn, the queue supports multiple modes:

- **interrupt** — override the current run
- **steer** — merge into the current run
- **followup** — wait for a separate turn after the current one
- **collect** — accumulate for batch processing

Heartbeat messages respect this queue — if a user message is being processed, the heartbeat simply skips and retries later.

---

## The `HEARTBEAT_OK` Protocol

The response contract between the agent and the gateway:

| Position in reply | Behavior |
|---|---|
| Start or end | Token stripped; message dropped if remaining content ≤ `ackMaxChars` (default 300) |
| Middle of reply | Not treated specially |
| Alert (no token) | Delivered to the user as-is |

Outside heartbeat runs, stray `HEARTBEAT_OK` tokens are logged and stripped.

---

## `HEARTBEAT.md` — The Checklist File

An optional file in the agent's workspace that acts as a stable, repeatable checklist the agent reads every heartbeat cycle. Example:

```markdown
# Heartbeat checklist
- Quick scan: anything urgent in inboxes?
- If daytime, lightweight check-in if nothing pending
- For blocked tasks, note what's missing
```

If the file is missing or effectively empty (only headers/whitespace), the heartbeat run is **skipped entirely** to save API tokens.

---

## Configuration

### Minimal config

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last",
      },
    },
  },
}
```

### Full options

| Key | Purpose | Default |
|---|---|---|
| `every` | Interval (duration string) | `30m` (1h for Anthropic OAuth) |
| `model` | Model override for heartbeat runs | inherits from agent |
| `target` | Delivery destination (`last`, `none`, channel ID) | `last` |
| `to` | Recipient override (e.g. E.164 phone number) | — |
| `ackMaxChars` | Suppression threshold for OK replies | 300 |
| `activeHours` | Timezone-aware window (`start`/`end`) | — |
| `includeReasoning` | Deliver a separate `Reasoning:` message | false |
| `session` | Which session to run in | `main` |

### Per-agent heartbeats

When any agent in `agents.list[]` has a `heartbeat` block, only those agents run heartbeats:

```json5
{
  agents: {
    list: [
      { id: "main" },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567"
        },
      },
    ],
  },
}
```

---

## Channel Visibility Controls

Three flags control what gets delivered per-channel or per-account:

- **`showOk`** — send the OK acknowledgment
- **`showAlerts`** — send alert content
- **`useIndicator`** — emit UI status events

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  telegram:
    heartbeat:
      showOk: true
```

If all three are false, the heartbeat run is skipped entirely (no model call, no cost).

---

## Internals — No Custom Agent Loop, It Wraps Pi

OpenClaw does **not** implement its own agent loop. It wraps **Pi**, a minimal coding agent by Mario Zechner. Pi provides four core tools (Read, Write, Edit, Bash) and a simple agentic loop. OpenClaw adds the gateway layer on top — channels, sessions, routing, heartbeat scheduling — but the actual LLM reasoning loop is Pi's.

### The full execution chain for a heartbeat

```
heartbeat-runner.ts (timer fires)
    → agent RPC (same entry point as any user message)
        → resolve-route.ts (find agent + sessionKey)
            → command-queue.ts (lane serialization — skip if busy)
                → runEmbeddedPiAgent (load context, build Pi session)
                    → Pi's agentic loop (LLM inference + tool calls)
                → response handling (HEARTBEAT_OK suppression, delivery)
            → updatedAt rollback
```

### Step by step

**1. Gateway startup** — `startGatewayServer` (`src/gateway/server.impl.ts`) initializes NodeRegistry, ChannelManager, cron, and heartbeat. The heartbeat runner starts its interval timer.

**2. Timer fires** — `heartbeat-runner.ts` calls `runHeartbeatOnce()`. This constructs a synthetic user message (the heartbeat prompt) and submits it to the gateway's agent RPC — the same RPC that handles normal user messages from Telegram/Slack/etc.

**3. Route resolution** — `src/routing/resolve-route.ts` resolves the target agent and `sessionKey` using the binding hierarchy: `peer → parent peer → guild → team → account → channel → default`. For heartbeats, this resolves to the agent's main session key (e.g., `agent:main:default`).

**4. Lane queue serialization** — The message enters the command queue (`src/process/command-queue.ts`). This provides session lane + global lane concurrency control. Each session has its own lane, serial by default (one run at a time). If a run is already active on this session lane, the heartbeat is **skipped** and retried next tick. This is why heartbeats never interrupt conversations.

**5. Session resolution** — The `agent` RPC validates params, resolves the session (`sessionKey` → `sessionId`), persists metadata, and returns `{ runId, acceptedAt }`. Workspace is created if needed. A session write lock is acquired.

**6. `runEmbeddedPiAgent`** — The core execution function (`src/agents/pi-embedded-runner/run.ts`). It resolves model + auth profile (possibly using a cheaper model if `heartbeat.model` is set), loads the session transcript (JSONL) into memory as context, trims old tool results, injects bootstrap/context files into the system prompt, and builds a Pi session. The LLM receives: `[system prompt] + [full conversation history] + [heartbeat prompt as latest user message]`.

**7. Pi's agentic loop** — Pi takes over. The LLM generates a response, possibly calls tools, results get backfilled, loop continues until the model stops or hits the timeout (600s default). For a typical heartbeat, the model reads `HEARTBEAT.md`, evaluates, and replies `HEARTBEAT_OK` — one turn, no tool calls, done.

**8. Response handling** — If the reply starts/ends with `HEARTBEAT_OK` and content is ≤ `ackMaxChars`, it's suppressed. If it's an alert, it's routed to the target channel. The `updatedAt` timestamp is rolled back.

### What Pi is

Pi is a minimal agent that emphasizes simplicity. Its distinguishing features:

- Only four core tools: Read, Write, Edit, Bash
- The shortest system prompt of any agent
- Self-extension philosophy — instead of downloading plugins, you ask the agent to extend itself
- Sessions operate as branching trees, not linear progressions
- No MCP integration by design — agents build their own capabilities

OpenClaw wraps Pi via `runEmbeddedPiAgent` and adds everything else: multi-channel I/O, heartbeat scheduling, lane queue serialization, session persistence, and the `HEARTBEAT_OK` protocol.

### Key insight

The heartbeat message is **indistinguishable from a normal user message** once it enters the queue. The only special handling is at the edges: the timer that creates it (input), and the response handler that knows about `HEARTBEAT_OK` (output). Everything in between — routing, queue, session loading, Pi's loop — is identical to processing a Telegram or Slack message.

---

## Heartbeat vs. Cron

This is the most important architectural distinction in OpenClaw's scheduling model.

| | Heartbeat | Cron |
|---|---|---|
| **Session** | Runs inside the agent's main session | Fires as an isolated system event with a fresh session |
| **Context** | Sees the full conversation transcript | Sees only bootstrap files + the cron prompt |
| **Use case** | Lightweight monitoring that requires conversational awareness | Scheduled tasks that don't need prior context |
| **Cost** | Expensive — loads full transcript every cycle | Cheaper — minimal context per run |
| **State** | Does NOT keep session alive (updatedAt rolled back) | Independent session lifecycle |

A cron job that runs `MORNING_BRIEF` starts from scratch each time. A heartbeat that checks "anything need attention?" can reason about what you were doing 2 hours ago because it has the full transcript.

The tradeoff: heartbeats are inherently more expensive. Every tick loads the full session context into the LLM call.

---

## Cost Optimization

Each heartbeat is a full agent turn consuming tokens. The gateway trims old tool results from the in-memory context before LLM calls (without rewriting the JSONL transcript on disk), but large sessions still cost significantly per heartbeat.

Strategies:

- Use a **cheaper model** for heartbeat runs (e.g., via OpenRouter Auto Model routing).
- Keep `HEARTBEAT.md` minimal — every line adds tokens.
- Set `target: "none"` for internal-only state updates.
- Widen the interval or use `activeHours` to restrict to waking hours.
- If all visibility flags (`showOk`, `showAlerts`, `useIndicator`) are false, the heartbeat is skipped entirely — no model call, no cost.

---

## Known Issues

1. **Heartbeat prompt leaks into cron events** ([#3589](https://github.com/openclaw/openclaw/issues/3589)) — The heartbeat prompt gets appended to ALL system events, not just heartbeat events. This causes the agent to reply `HEARTBEAT_OK` to custom cron jobs (e.g., `MORNING_BRIEF`), ignoring their actual purpose. Root cause is in `src/infra/heartbeat-runner.ts`. A fix PR (#3580) has been opened.

2. **Heartbeat stops after context compression** ([#2935](https://github.com/openclaw/openclaw/issues/2935)) — After context compression, heartbeat polling stops with a "No session found" error until a gateway restart. The session reference breaks during compression.

3. **Token burn from context dragging** ([#1594](https://github.com/openclaw/openclaw/issues/1594)) — Large tool outputs stored in the session transcript cause cost spikes because every heartbeat (and every normal message) drags that context forward. Isolated cron sessions avoid this since they start with a clean slate each time.

4. **Per-bot heartbeats not supported** ([#752](https://github.com/openclaw/openclaw/issues/752)) — Heartbeats are tied to the default/primary bot's session and outbound account. Non-default bots don't get their own heartbeat schedule or sender identity.

---

## Observability

The gateway exports OpenTelemetry metrics for heartbeat monitoring:

- `openclaw.queue.lane.enqueue` — queue enqueue events + depth
- `openclaw.queue.lane.dequeue` — queue dequeue events + wait time
- `openclaw.session.state` — session state transitions
- `openclaw.session.stuck` — session stuck warnings
- `diagnostic.heartbeat` — aggregate counters for webhooks, queue, and session health

---

## Sources

- [Heartbeat — OpenClaw Docs](https://docs.openclaw.ai/gateway/heartbeat)
- [Agent Loop — OpenClaw Docs](https://docs.openclaw.ai/concepts/agent-loop)
- [Session Management — OpenClaw Docs](https://docs.openclaw.ai/concepts/session)
- [Messages — OpenClaw Docs](https://docs.openclaw.ai/concepts/messages)
- [Pi: The Minimal Agent Within OpenClaw — Armin Ronacher](https://lucumr.pocoo.org/2026/1/31/pi/)
- [OpenClaw Source Code Review — Moely](https://www.moely.ai/resources/openclaw-framework-source-code-review)
- [OpenClaw Architecture Guide — Vertu](https://vertu.com/ai-tools/openclaw-clawdbot-architecture-engineering-reliable-and-controllable-ai-agents/)
- [OpenClaw GitHub Repository](https://github.com/openclaw/openclaw)
- [Issue #3589 — Heartbeat prompt leaking into cron events](https://github.com/openclaw/openclaw/issues/3589)
- [Issue #2935 — Heartbeat stops after context compression](https://github.com/openclaw/openclaw/issues/2935)
- [Issue #1594 — Token burn from context dragging](https://github.com/openclaw/openclaw/issues/1594)
- [Issue #752 — Per-bot heartbeats](https://github.com/openclaw/openclaw/issues/752)
- [PR #5219 — Message queue visibility](https://github.com/openclaw/openclaw/pull/5219)
- [OpenClaw on VentureBeat](https://venturebeat.com/security/openclaw-agentic-ai-security-risk-ciso-guide)
- [OpenClaw on TechCrunch](https://techcrunch.com/2026/01/30/openclaws-ai-assistants-are-now-building-their-own-social-network/)
- [OpenRouter — OpenClaw Integration](https://openrouter.ai/docs/guides/guides/openclaw-integration)

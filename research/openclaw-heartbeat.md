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

## The `HEARTBEAT_OK` Protocol

This is the response contract between the agent and the gateway:

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

## Cost Optimization

Each heartbeat is a full agent turn consuming tokens. Strategies:

- Use a **cheaper model** for heartbeat runs (e.g., via OpenRouter Auto Model routing).
- Keep `HEARTBEAT.md` minimal.
- Set `target: "none"` for internal-only state updates.
- Widen the interval or use `activeHours` to restrict to waking hours.

---

## Key Architectural Distinction: Heartbeat vs. Cron

- **Heartbeat** runs inside the agent's main session, preserving conversational context.
- **Cron** fires system events that can trigger independent agent actions.
- The heartbeat is designed for lightweight monitoring; cron is for scheduled tasks.

---

## Known Issues

1. **Heartbeat prompt leaks into cron events** ([#3589](https://github.com/openclaw/openclaw/issues/3589)) — The heartbeat prompt gets appended to ALL system events, not just heartbeat events. This causes the agent to reply `HEARTBEAT_OK` to custom cron jobs (e.g., `MORNING_BRIEF`), ignoring their actual purpose. Root cause is in `src/infra/heartbeat-runner.ts`. A fix PR (#3580) has been opened.

2. **Heartbeat stops after context compression** ([#2935](https://github.com/openclaw/openclaw/issues/2935)) — After context compression, heartbeat polling stops with a "No session found" error until a gateway restart.

---

## Sources

- [Heartbeat — OpenClaw Docs](https://docs.openclaw.ai/gateway/heartbeat)
- [OpenClaw GitHub Repository](https://github.com/openclaw/openclaw)
- [Issue #3589 — Heartbeat prompt leaking into cron events](https://github.com/openclaw/openclaw/issues/3589)
- [Issue #2935 — Heartbeat stops after context compression](https://github.com/openclaw/openclaw/issues/2935)
- [Issue #752 — Per-bot heartbeats](https://github.com/openclaw/openclaw/issues/752)
- [OpenClaw on VentureBeat](https://venturebeat.com/security/openclaw-agentic-ai-security-risk-ciso-guide)
- [OpenClaw on TechCrunch](https://techcrunch.com/2026/01/30/openclaws-ai-assistants-are-now-building-their-own-social-network/)
- [OpenRouter — OpenClaw Integration](https://openrouter.ai/docs/guides/guides/openclaw-integration)

---
name: shipyard-foundation
description: >-
  Use this when a domain Linear agent (Bagsy, a stranger’s bot, etc.) needs the
  Shipyard factory contract — lane model, triage hooks, layer map, handoff
  shapes, secrets checklist, failure modes, and dry-run — before using
  Tailscale, webhook, Herdr, or Linear Agent plugins.
---

# Shipyard foundation

Shared **contract** for domain agents that use Shipyard to build software.
This skill does **not** install Tailscale, verify HMAC, talk OAuth, or drive
Herdr sockets — those live in later plugins. You are the factory; Shipyard is
capability.

## Way of working

- **Domain agents are the factory.** Bagsy (and peers) run build/ship themselves via Shipyard skills. Do not require a separate Shippy Linear agent as a choke point.
- **Shippy** is product owner of the Shipyard repo (Trammy ↔ Tramworks), not a mandatory runtime orchestrator in every workspace.
- **Parallel by default.** Many lanes at once. Limits are capacity (Herdr health, rate limits, triage priority), not a single global queue.
- **Webhook never auto-Herdr.** Receive → verify → ACK → persist context → triage. Dispatch coding only after an explicit triage decision.
- **Secrets** stay in env vars, mode-600 files, or plugin variables — never in skill/plugin bodies, prompts on argv, or git.

## Lane model

One **lane** =

1. One inbound ask (Linear delegate/mention, or an equivalent handoff)
2. One **Linear Agent session** (thought ACK, responses, elicitation)
3. Optionally one **Herdr agent / worktree** when triage chooses `dispatch coding`

Rules:

- Map `ask → session → (optional) Herdr agent`. Do not share a dirty worktree across lanes.
- Two concurrent asks ⇒ two sessions (and up to two Herdr agents). No global single-job lock.
- If Herdr capacity is exhausted, fail closed on that lane: comment in Linear, do not steal another lane’s agent.

## Triage hooks

After context is persisted, choose exactly one:

| Decision | Meaning |
|----------|---------|
| `handle` | Resolve in Linear / chat without coding agents |
| `ask human` | Need Jordan (or workspace lead) — elicitation / question |
| `dispatch coding` | Enqueue to a named Herdr agent; then wait/report |

Policy (when to choose which) is owned by the **installing domain agent**.
This skill only defines the decision points and that `dispatch coding` is never
implied by webhook receipt alone.

## Layer map and install order

| Order | Plugin id | Promise (interface only) |
|-------|-----------|---------------------------|
| 0 | `shipyard-skeleton` / this skill | Contracts, dry-run, compose stubs |
| **NEXT** | `herdr-coding` | List/enqueue/status for named Herdr agents; parallel lanes |
| 1 | `tailscale-onboarding` | Host on tailnet; MagicDNS; ping/status |
| 2 | `tailscale-webhooks` | Funnel path → localhost verifier; HMAC + timestamp + dedupe; fast 200 |
| 4 | `linear-agent` | App user, Agent Sessions, thought ~10s, activities; uses webhooks |
| 5 | `shipyard` | Compose: triage → optional Herdr → Linear status |

**Implementation priority (locked):** `0 → herdr-coding → 1 → 2 → 4 → 5`.
Herdr is pulled forward so domain agents can use it to code the rest of Shipyard.
Numeric milestone labels in Linear may still read 1–5; follow this priority.

## Handoff shapes

### Prompt / context file (coordinator → triage)

Write a file the agent reads; do not paste secrets into it.

```text
issue_id: BAG-123
session_id: <linear-agent-session-id>
title: ...
body: ...
url: https://linear.app/...
requested_by: <app-or-user>
delivery_id: <webhook-delivery-id>
received_at: <iso8601>
```

### Herdr enqueue payload (after `dispatch coding`)

Allowlisted fields only. **No secrets on argv.**

```json
{
  "session": "<herdr-session-name>",
  "agent": "<named-agent>",
  "prompt_path": "/absolute/path/to/prompt.md",
  "issue_id": "BAG-123",
  "requested_by": "bagsy"
}
```

Prompt file at `prompt_path` holds the task text. Coding agents report back to
the domain agent; the domain agent owns Linear writes for that lane.

## Secrets checklist (names only)

Document presence; never commit values.

- `LINEAR_WEBHOOK_SECRET` — HMAC for webhook verifier
- Linear app / OAuth token for the domain agent’s app user (name per Linear setup docs)
- Herdr/SSH credentials for the coding host (key paths / agent sockets — not in git)
- Optional: Funnel / Tailscale auth keys if your onboarding flow uses them

Prefer plugin variables or env injected at runtime.

## Failure modes

| Failure | Response |
|---------|----------|
| HMAC / signature fail | Reject; do not disable verify; rotate secret if needed |
| Slow ACK (>~10s thought) | Keep receiver thin; persist async; fix path — Linear looks dead |
| Funnel / ingress down | Comment blocker; Gus/IT for lab; no auto-fallback to public HTTP |
| Herdr unreachable / agent unknown | Fail closed; Linear comment; no ad-hoc SSH paste of payloads |
| Capacity exhausted | Queue or ask human; do not pre-empt other lanes |
| Accidental auto-dispatch | Treat as incident; confirm dispatch stays manual/`noop` on webhook |

## Dry-run

Without real credentials, a domain agent should still be able to walk:

1. Fake or recorded session `created` event
2. Thought ACK stub within ~10s
3. Persist prompt/context file
4. Triage decision point (`handle` / `ask human` / `dispatch coding`)
5. Optional Herdr enqueue **stub** (log payload; no shell)
6. Linear status/activity **stub**
7. Second concurrent lane stub mid-flight without serializing into one queue

Say clearly in the run output what still needs real credentials.

## Non-goals

- Auto-dispatch Herdr (or SSH) on every webhook / mention / assign
- Generic Funnel cookbook for dashboards/UIs
- Forcing coding onto the Bot VM when Apple/signing needs a Mac
- Notify hop / secondary wake channels (out of v1)
- Secrets in plugin bodies or prompt argv
- Replacing domain agents with a mandatory Shippy runtime factory bot

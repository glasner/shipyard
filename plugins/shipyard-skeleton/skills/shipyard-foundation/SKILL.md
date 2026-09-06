---
name: shipyard-foundation
description: >-
  Use this when starting Shipyard work or when a domain Linear agent needs the
  factory basics — what Shipyard is, who runs it, and what not to do yet.
---

# Shipyard foundation

Shipyard is a **plugin/skill stack**. Domain Linear agents (e.g. Bagsy) **are the factory** — they use these skills to build. Shippy owns this repo; you do not need a separate Shippy runtime agent.

## Right now

Stack is mostly stubs. Prefer the smallest next slice over inventing the whole factory.

**Build order:** foundation (this) → **`herdr-coding`** → Tailscale onboarding → webhooks → Linear Agent → composed `shipyard`.

## Hard rules

- Never auto-dispatch Herdr (or SSH) from a webhook alone
- No secrets in plugin/skill bodies — env, files, or plugin variables only
- Prefer parallel lanes later (one ask → one session → optional one Herdr agent); do not invent a global single-job lock

## Not in this skill yet

Payload schemas, full layer contracts, dry-run scripts, and failure playbooks land with the plugins that need them. Do not pad this file ahead of that.

# Shipyard

Public Grok Bot / Cursor **plugin monorepo**: skills that let domain Linear agents (e.g. Bagsy) turn issues into shipped work.

**Way of working:** Shipyard is capability. Domain agents *are* the factory — they install/use these plugins to build their thing. Shippy is product owner of this repo, not a mandatory runtime agent.

## Plugins

| Id | Milestone | Role |
|----|-----------|------|
| `shipyard-skeleton` | 0 | Living compose skeleton |
| `tailscale-onboarding` | 1 | Tailnet join + MagicDNS |
| `tailscale-webhooks` | 2 | Signed Funnel webhook pipe |
| `herdr-coding` | 3 | Herdr orchestration (parallel lanes) |
| `linear-agent` | 4 | Linear Agent surface |
| `shipyard` | 5 | Composed triage → optional Herdr → status |

Distribute as **separate marketplace plugins by id** from this monorepo (`.cursor-plugin/marketplace.json`). Dogfood as local skills until published.

## Non-goals

- No auto-dispatch to Herdr from Linear webhooks
- No generic Funnel cookbook — webhooks-specific only
- Coding host stays where Apple/signing needs it (Mac)
- Notify hop out of v1
- Secrets never in plugin bodies

## Linear

https://linear.app/bagsy/project/shipyard-14f3c7ff3f56

## Status

Milestone 0 scaffold only — plugin bodies intentionally stubby until each milestone ships.

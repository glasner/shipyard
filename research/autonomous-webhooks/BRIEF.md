# Deep research brief: autonomous webhooks → agent-on-demand

Jordan (Bagsy/Shipyard) is sidequesting: is there a **standalone business** for hosted “autonomous webhooks” that skip Tailscale Funnel/homelab edge and instead:

1. Expose public HTTPS webhook ingress
2. Verify signatures (HMAC etc.), timestamp + dedupe, fast HTTP 200
3. **Spin up or wake an agent process on demand** with delivery context (not merely relay to a URL)

Contrast with Shipyard’s current path (Tailscale Funnel → localhost verifier → domain agent triage → optional Herdr).

## Your job

Deep market + product research. Write `REPORT.md` in this directory. Use the web. Cite sources with links. Be opinionated at the end.

## Cover

### 1. Category definition
Name the space. Adjacent buckets: webhook relays/gateways, event buses, serverless HTTP, agent platforms, “wake a worker.”

### 2. Competitive landscape (table)
For each notable player: what they do, wake-agent vs URL-forward, auth/HMAC, multi-tenant, pricing signal if public, agent-native or not.

Look at least at: Svix, Hookdeck, ngrok, Cloudflare Workers/Queues, Inngest, Trigger.dev, Temporal, Modal, Railway/Render cron+webhooks, Zapier/Make, GitHub Apps webhooks, Linear Agent Sessions, Cursor/cloud agents if relevant, any “agent webhook” startups or OSS (e.g. Telegram bot webhooks, ccgram/herdr notes, OpenAI Assistants webhooks if any).

### 3. Jobs to be done
Who pays? Indie hackers with bots, product teams with Linear/GitHub agents, AI app builders, homelab vs SaaS.

### 4. Wedge analysis
Is “webhook → spawn/resume agent process” differentiated enough vs “better webhook relay”? What must you own (runtime? sandbox? identity? queues?)?

### 5. Risks
Platform absorption (Linear/GitHub/Cursor), abuse/cold-start SLOs, secret custody, multi-tenant isolation.

### 6. Recommendation
Go / no-go / beachhead-only as ingress product feeding Shipyard skills. 3 sharp next validation steps (talk to customers, build stub, etc.).

## Constraints

- Research and write only. Do **not** modify other repos (especially ~/code/bagsy).
- Prefer primary docs and recent posts (2024–2026).
- Keep REPORT.md scannable: exec summary up top, then sections, then sources.

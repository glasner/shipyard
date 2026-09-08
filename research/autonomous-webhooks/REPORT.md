# Autonomous webhooks → agent-on-demand: market & product research

**For:** Jordan (Bagsy / Shipyard)
**Date:** 2026-09-08
**Scope:** Is there a standalone business in hosted webhook ingress that *spins up or wakes an agent* rather than forwarding to a URL?

---

## Executive summary

**Verdict: no-go as a standalone business. Beachhead-only — build it as Shipyard's hosted ingress mode, don't spin it out.**

The three-part thesis in the brief (public HTTPS ingress → verify/dedupe/fast-200 → wake an agent with delivery context) is correct as an *architecture*. It is not available as a *business*, because every one of the three parts has already been claimed, and the part you'd own is the cheapest one.

Five findings drove this:

1. **The exact product exists, more than once, and it is mostly free.** [AgentsRoom](https://agentsroom.dev/features/webhook-triggers) ships hosted ingress → per-provider HMAC → payload filter → burst limit → replay → *starts a real agent on your own machine*, holds events a week while you're offline, and locks so one event never spawns two agents. That is the brief, feature for feature, and it's free (BYO model provider). [Gobii](https://gobii.ai/pricing/) is MIT-licensed OSS plus a hosted tier at $50/mo where "inbound webhooks wake agents" is the headline primitive.
2. **The incumbents already colonized the agent runtimes.** Svix shipped [official OpenClaw and Hermes plugins](https://www.svix.com/blog/receive-webhooks-in-openclaw-and-hermes/) on 2026-07-28. Hookdeck shipped [plugins for both](https://hookdeck.com/blog/hookdeck-for-hermes-agent) plus a [six-pattern taxonomy](https://hookdeck.com/webhooks/guides/webhooks-ai-agents-integration-patterns) of exactly this category. They are not asleep on "agent-native"; they wrote the category page.
3. **The runtimes are absorbing it natively.** OpenClaw already exposes `POST /hooks/wake`. Hermes merged HMAC + idempotency + rate-limiting + direct-delivery webhooks in [PR #12473](https://github.com/NousResearch/hermes-agent/pull/12473) (2026-04-19). Anthropic ships [`routines/{id}/fire`](https://platform.claude.com/docs/en/api/claude-code/routines-fire). Cloudflare Agents [hibernate and wake on an inbound request](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/) as a platform primitive.
4. **The economics don't clear.** Svix and Hookdeck raised ~$10.35M and ~$2.67M respectively across the whole category's history. Free tiers are 50k and 10k events/month. Cloudflare Tunnel is free and unmetered. Your buyer's current spend on this problem is $0, and the two best-funded competitors already outrank you on every SEO term you'd need.
5. **The genuinely hard, unowned problem is next door, not here.** It is not "deliver the webhook." It is *"an internet-reachable, attacker-influenced payload just became instructions to an agent with my credentials and my shell."* 40,000+ OpenClaw instances were found exposed, ~63% remotely exploitable ([CVE-2026-25253](https://www.proarch.com/blog/threats-vulnerabilities/openclaw-rce-vulnerability-cve-2026-25253)), alongside 800+ malicious skills in its registry. That is a security product for a 389k-star install base, and it is the only version of this sidequest I'd fund.

**What to do instead:** keep the ingress as a Shipyard capability (the hosted counterpart to `tailscale-webhooks`), and spend the sidequest energy on validating the security framing — *hardened ingress + injection quarantine for self-hosted agents* — before writing a line of relay code.

---

## 1. Category definition

There is no settled name. Hookdeck calls the general layer an **Event Gateway**; the agent-specific slice has no accepted label. The most accurate name for what the brief describes is:

> **Agent ingress** — the layer that terminates untrusted public HTTP, proves provenance, and converts an external event into an *agent invocation* (a new session, or a message into a live one) with the payload as context.

It sits at the intersection of five existing buckets, and the reason it feels like a category is that it borrows one job from each:

| Bucket | Canonical players | What it contributes | What it lacks |
|---|---|---|---|
| **Webhook relays / gateways** | Svix Ingest, Hookdeck Event Gateway, Convoy, Hook0 | Provider signature catalog, dedupe, retries, replay, event log | No compute. Terminates at a URL you must already be serving. |
| **Tunnels / edge ingress** | ngrok, Cloudflare Tunnel, Tailscale Funnel, Localtonet | Public HTTPS without port-forwarding | No verification, no queue, no durability. Node must be up. |
| **Event buses / queues** | Cloudflare Queues, EventBridge, Kafka | Durability, backpressure, fan-out | No provenance, no agent semantics. |
| **Serverless HTTP / "wake a worker"** | Cloudflare Workers + Durable Objects, Fly Machines autostart, Modal, Lambda, Cloud Run | Scale-to-zero, wake-on-request | Generic compute. No webhook semantics, no agent session model. |
| **Agent platforms** | Anthropic Managed Agents, Cloudflare Agents, Vercel eve, Gobii, OpenClaw, Hermes | The session/state model and the actual run | Ingress is an afterthought: weak or absent verification, no dedupe, no queue. |

**The honest framing:** "autonomous webhooks" is not a new category. It is the *adapter* between bucket 1 and bucket 5 — and both ends are actively growing into the middle.

One structural detail worth naming, because it's the single most important product fact in this whole space: **the winning implementations mostly don't push.** Svix's OpenClaw/Hermes plugins have the agent **poll Svix every 5s**. Hookdeck's plugins deliver through the **Hookdeck CLI's outbound WebSocket tunnel**. WebhookAgent uses **heartbeat polling with a break condition**. Nobody with a shipping product is pushing inbound to a homelab, because outbound-only is what makes it safe and NAT-agnostic. This inverts the brief's premise: the valuable asset is the *durable buffer the agent pulls from*, not the *public HTTPS ingress*. Ingress is table stakes; the buffer is the product; and the buffer is what Svix gives away 50,000/month of.

---

## 2. Competitive landscape

Legend for **Wake?**: **Wake** = starts or resumes an agent process/session · **Forward** = delivers to something you're already running · **Pull** = buffers, agent polls out.

### A. Webhook gateways / relays — own verification and durability, sell no compute

| Player | What it does | Wake vs forward | Auth / HMAC | Multi-tenant | Pricing signal | Agent-native? |
|---|---|---|---|---|---|---|
| **[Svix Ingest](https://www.svix.com/ingest/)** | Receiving gateway: provisioned endpoints, verification, JS transforms, fan-out, polling endpoint | Forward / **Pull** | Built-in for Stripe, GitHub, Shopify, Slack, HubSpot etc. | Yes (built for it — embeddable customer portals) | Free 50k msg/mo; Pro **$490/mo**; $0.0001/msg overage; retries & filtered msgs free ([pricing](https://www.svix.com/pricing/)) | **Yes, explicitly** — [official OpenClaw + Hermes plugins](https://www.svix.com/blog/receive-webhooks-in-openclaw-and-hermes/) shipped 2026-07-28 |
| **[Hookdeck Event Gateway](https://hookdeck.com/event-gateway)** | Ingest → verify → queue → filter → transform → route → deliver, with full event log and CLI tunnel | Forward | HMAC, Basic, Bearer, API key; **160+ preconfigured source types** | Yes | Dev $0/10k events; Team **$39/mo**; Growth **$499/mo**; overage $3.00/100k up to 5M ([pricing](https://hookdeck.com/pricing)) | **Yes, aggressively** — [Hermes plugin](https://hookdeck.com/blog/hookdeck-for-hermes-agent), [OpenClaw guide](https://hookdeck.com/webhooks/platforms/using-hookdeck-with-openclaw-reliable-webhooks-for-your-ai-agent), [Linear agent guide](https://hookdeck.com/webhooks/platforms/how-to-build-linear-agents-with-hookdeck-cli), [agent skills](https://github.com/hookdeck/agent-skills), CLI MCP server |
| **[Convoy](https://www.svix.com/alternatives/convoy/)** | OSS send+receive gateway, retries, replay, JS transforms | Forward | Yes | Yes | Free (OSS). **Company no longer active**; measured uptime <99.0% | No |
| **[Hook0](https://www.hook0.com/)** | OSS/EU webhook server | Forward | Yes | Yes | From €59/mo | No |
| **[Pipedream](https://pipedream.com/docs/pricing)** | Event sources + hosted Node workflows + Connect (managed OAuth, 10k+ tools, MCP) | Forward (runs your code) | Per-source | Yes (Connect is built for end-user tenancy) | Free 100 credits; Basic $29; Advanced $49; **Connect $99/mo** | Partly — Connect/MCP is agent-facing tooling, not agent ingress |

### B. Tunnels — the actual status quo you're displacing

| Player | What it does | Wake vs forward | Auth / HMAC | Pricing signal | Notes |
|---|---|---|---|---|---|
| **[Tailscale Funnel](https://tailscale.com/docs/features/tailscale-funnel)** | Public HTTPS to a tailnet node via relays | Forward | None (you verify) | Included with Tailscale | **Ports 443/8443/10000 only**, TLS only, needs MagicDNS + HTTPS certs + `funnel` nodeAttr, **non-configurable bandwidth limits, still beta**. No queue — node down = event lost. |
| **[ngrok](https://ngrok.com)** | Tunnels + traffic policy, webhook verification module | Forward | **Yes — webhook verification for major providers**, plus OAuth/IP allowlist | Free: 3 endpoints, 1GB, 20k req/mo; Personal ~$8/mo; Pro ~$49/mo | Closest thing to a paid incumbent for "verified ingress to my machine." Same failure mode: no durability. |
| **Cloudflare Tunnel** | Outbound-only tunnel, stable custom domain | Forward | None | **Free, no bandwidth cap, tunnels don't expire** | The price anchor that ruins this market. |

### C. Durable execution / job runners — own the run, not the ingress

| Player | What it does | Wake vs forward | Auth / HMAC | Pricing signal | Agent-native? |
|---|---|---|---|---|---|
| **[Inngest](https://www.inngest.com/pricing)** | Durable functions, event-triggered, AgentKit + `step.ai`, `useAgent` streaming hook | **Wake** (invokes your function) | **Weak** — per-source transform script *you* write; no curated provider catalog | Hobby free 50k executions; **Pro $99/mo** 1M executions; events $0.50/M | Yes (AgentKit), but ingress is DIY |
| **[Trigger.dev](https://trigger.dev/pricing)** | No-timeout tasks, Realtime streams, **Waitpoints** (human-in-the-loop), webhook callbacks that pause runs | **Wake** | You implement | Free $0 / Hobby $10 / Pro $50 + **$0.000025/run** + $0.0000338/s (small-1x) | Yes — positions as "fully-managed AI agents and workflows" |
| **[Temporal](https://temporal.io/solutions/ai)** | Durable execution; workflows wait on signals for hours/days at no compute cost | **Wake/resume** via signal | You implement | ~$200/mo production entry | Yes — and [$300M Series D at $5B, Feb 2026](https://temporal.io/) says durable execution is now table stakes |
| **Cloudflare Workers + Queues** | Edge HTTP + durable queue | **Wake** (isolate spin-up) | You implement | Workers Paid $5/mo → 10M req, then $0.30/M; [Queues $0.40/M ops](https://developers.cloudflare.com/queues/platform/pricing/), free tier 10k ops/day | Substrate, not product |

### D. Agent runtimes with *native* ingress — the absorbers

| Player | Native ingress | Wake? | Auth / HMAC | Pricing signal | Notes |
|---|---|---|---|---|---|
| **[OpenClaw](https://docs.openclaw.ai/gateway/security)** (389k★) | `POST /hooks/wake`, `/hooks/agent`, `/hooks/<name>` | **Wake — literally named `/hooks/wake`** | Single shared token; **no per-provider signature verification**; disabled by default | OSS; managed hosting $2.99–$49/mo across a whole cottage industry | Docs say **never expose unauthenticated**; recommends Tailscale Serve / SSH / reverse proxy. No queue, no dedupe, no dashboard. |
| **[Hermes Agent](https://github.com/NousResearch/hermes-agent)** (243k★, MIT) | Webhook routes with templates + delivery targets | **Wake** (spawns session) or direct-deliver | **HMAC (GitHub/GitLab/generic) + idempotency + per-route rate limits — merged [PR #12473](https://github.com/NousResearch/hermes-agent/pull/12473), 2026-04-19** | Free OSS | Also has an open RFC ([#491](https://github.com/NousResearch/hermes-agent/issues/491)) to make the whole hook system bidirectional. They are building your product in-tree. |
| **[Gobii](https://gobii.ai/pricing/)** | Inbound webhooks, inbound email (IMAP idle), agent-to-agent messages, schedules — one durable Celery/Redis loop | **Wake** — "inbound webhooks wake agents" | Yes | **OSS (MIT) free self-host; Pro $50/mo 1k tasks + $0.10/task; Scale $250/mo 10k + $0.04** | The closest thing to a priced version of the brief. |
| **[Cloudflare Agents](https://developers.cloudflare.com/agents/)** | Durable Object per agent | **Wake — hibernates when idle, wakes on HTTP/WS/alarm/email** | You implement | Workers pricing; ~free when idle | `keepAlive()` (Mar 2026) and Project Think fibers (Apr 2026) added long-run durability. This is the reference architecture. |
| **[Vercel eve](https://vercel.com/blog/introducing-eve)** (2026-06-17) | Channel adapters: HTTP, Slack, Discord, Teams, Telegram, Twilio, GitHub, Linear | **Wake** | **Vercel Connect owns the OAuth app and verifies inbound** — you set no `LINEAR_WEBHOOK_SECRET`, no `GITHUB_WEBHOOK_SECRET`, no `SLACK_SIGNING_SECRET` ([docs](https://vercel.com/docs/connect/frameworks/eve)) | Vercel platform pricing | **This is the secret-custody play, already shipped.** Connect swaps the provider's native signature for a Vercel OIDC token. |
| **n8n** (160k★) | Webhook node + AI Agent builder | **Wake** | Per-node | OSS self-host free; cloud per-execution | The homelab default. Enormous distribution. |

### E. Vendor-owned agent ingress — the platform-absorption risk, in production

| Player | Ingress | Wake? | Auth | Gaps you could theoretically sell into |
|---|---|---|---|---|
| **[Anthropic `routines/{id}/fire`](https://platform.claude.com/docs/en/api/claude-code/routines-fire)** | HTTP POST starts a Claude Code cloud session | **Wake** | Per-routine bearer `sk-ant-oat01-…`, scoped to one routine, no read access | **No HMAC** (can't be pointed at Stripe/GitHub directly — wrong auth model). **No idempotency key — "if a webhook caller retries, the endpoint creates multiple sessions."** Body is freeform `text` ≤65,536 chars, *not parsed*. Daily run caps by plan; 429 + `Retry-After`. Experimental beta header. |
| **[Anthropic Managed Agents](https://platform.claude.com/docs/en/managed-agents/sessions)** (GA 2026-04-08) | Outbound webhooks: `session.status_*`, `session.thread_created`, `vault_credential.refresh_failed` | Outbound only | `whsec_` secret, `X-Webhook-Signature`, 5-min replay window | **At-least-once, ordering not guaranteed, thin payloads (fetch-back required), endpoint auto-disables after ~20 consecutive failures.** Billing: tokens + **$0.08/session-hour**. |
| **[Linear Agent Sessions](https://linear.app/developers/agent-interaction)** | `AgentSessionEvent` webhooks (`created`, `prompted`) | **Wake** | `Linear-Signature`: bare hex HMAC-SHA256 over raw body + `webhookTimestamp` (~1 min replay window) | **5s to ack, 10s to emit first activity or the session is marked unresponsive** — while a real agent run takes minutes. This impedance mismatch is the sharpest technical gap in the whole space. |
| **GitHub Agent HQ** | Third-party coding agents (Claude, Codex, Cognition, xAI) run inside GitHub; hooks customize execution | **Wake** | GitHub App / `X-Hub-Signature-256` | GitHub is becoming the ingress *and* the runtime for the highest-value use case (code). |
| **[Cursor Cloud Agents API](https://cursor.com/docs/cloud-agent/api/endpoints)** | Programmatic launch; Slack/GitHub/schedule/webhook triggers | **Wake** | Basic or Bearer | Same absorption pattern. |
| **[OpenAI webhooks](https://platform.openai.com/docs/guides/webhooks)** | `response.completed` etc. for background/batch/deep-research | Outbound | `client.webhooks.unwrap()` signature verify | Confirms every model vendor now emits webhooks — growing the *demand* for a gateway, and growing Svix/Hookdeck's TAM more than yours. |

### F. Direct "webhook → agent" products — your actual competitors

| Player | What it does | Wake? | Auth / HMAC | Multi-tenant | Pricing | Read |
|---|---|---|---|---|---|---|
| **[AgentsRoom](https://agentsroom.dev/features/webhook-triggers)** | Hosted trigger URL + signing secret → verify → payload filter → burst limit → **starts a real agent on your machine** with `{{event.*}}` prompt variables; last-call replay; **queues up to a week if you're offline**; cross-machine lock so one event ≠ two agents | **Wake (local)** | **Per-provider: `X-Hub-Signature-256`, `X-Slack-Signature`, `X-Gitlab-Token`, raw-body HMAC for Linear/Sentry/generic. Unsigned rejected.** | Per-account, agent runs on your hardware | **Free** (BYO Claude/Codex/Antigravity). 3,567 downloads/30d, 106 countries | **This is the brief, shipped, for $0.** Desktop app, not a service — that's the only seam. |
| **[Gobii](https://docs.gobii.ai/developers/webhooks)** | OSS + hosted; unified schedule + event queue; webhooks, email, agent-to-agent all wake agents | **Wake** | Yes | Yes (hosted) | Free self-host; **$50/mo Pro** | The priced version. Also the design Hermes is copying. |
| **WebhookAgent** | Webhook queue + heartbeat polling with break-conditions for agents; "no server, no tunnel, no lost events" | **Pull** | Provider webhooks | Yes | 100 free events/mo | Small; site is now mostly an SEO content hub — treat the product claims as unverified. |
| **Macha / MindStudio / Zapier Agents / Make** | No-code agents with custom webhook triggers | **Wake** | Platform-managed | Yes | Zapier Agents **$50/mo / 1,500 activities**; Make per-operation | The non-developer end. Zapier's $50/1,500 is the ARPU ceiling reference for "event wakes an agent." |

---

## 3. Jobs to be done

Four buyers, ranked by how much they'd actually pay.

**① The self-hosted agent operator (indie / homelab) — biggest population, worst monetization.**
Runs OpenClaw or Hermes on a Mac mini, a Hetzner box, or a laptop. Wants GitHub/Linear/Stripe/Sentry to reach it. Today: Tailscale Funnel, Cloudflare Tunnel, ngrok free, or Telegram polling because it needs no public URL at all.
*JTBD:* "Let the outside world start my agent without putting my machine on the internet."
*Willingness to pay:* **~$0–10/mo.** They chose self-hosting to avoid a bill. Their existing tools are free and their agent framework is free. This is the population (389k + 243k + 160k stars) and it is not the revenue.

**② The product team shipping a Linear/GitHub agent — real budget, but shrinking need.**
Building a domain agent that acts as a Linear or GitHub App. Needs to ack in 5s, work for 10 minutes, and post activities back as the app identity.
*JTBD:* "Make the platform's session contract work when my agent is slow and my process restarts."
*Willingness to pay:* **$50–500/mo, gladly.** But this is exactly who Vercel Connect + eve already serves for free-with-platform, and who Hookdeck already sells to at $39–499. Shipyard is *in* this segment — as a customer of the capability, not a vendor of it.

**③ The AI app builder with async model callbacks — real pain, wrong shape.**
OpenAI/Anthropic/Gemini emit completion webhooks; MCP servers return async results. At-least-once, unordered, thin payloads, auto-disabling endpoints.
*JTBD:* "Don't lose or double-process my model callbacks."
*Willingness to pay:* **$39–499/mo.** This is Hookdeck's home turf and they've already published the taxonomy. Nothing here needs an agent runtime — it needs a gateway. You'd be the fourth-best gateway.

**④ The security-conscious team running agents on owned hardware — smallest, richest, unserved.**
Compliance or data-residency reasons to keep the agent local, but needs external events. Cares that a Sentry payload can't talk the agent into `rm -rf` or exfiltrating a token.
*JTBD:* "Give my local agent an internet-facing trigger that is provably not an attack surface."
*Willingness to pay:* **$100–1,000/mo**, because the alternative is a security review. **Nobody is selling this.** See §6.

**The uncomfortable pattern:** willingness-to-pay is inversely correlated with the count of people who want it, and the one segment with both is the one where the value is *security*, not *delivery*.

---

## 4. Wedge analysis

### Is "wake an agent" differentiated from "better webhook relay"?

**Technically: yes, and the difference is real.** A relay's contract ends at a 2xx from your server. An agent invocation has properties a relay has no concept of:

- **The ack window and the work window are different by three orders of magnitude.** Linear wants 5s; the agent needs 10 minutes. Every agent platform has this problem and no relay solves it, because a relay's job is to get a 200, not to represent a session that's still thinking.
- **Duplicate suppression must be semantic, not transport-level.** Anthropic's `/fire` is the proof: it has *no idempotency key*, so a retry from any webhook source spawns a second billed session. Dedupe by `event.id` is not enough — you need "is there already a live session for this issue?"
- **Payload → prompt is a lossy, security-relevant transform.** AgentsRoom's `{{event.title}}` / `{{event.author}}` mapping is the honest version. Anthropic's `/fire` takes freeform `text` and explicitly does not parse it. Somebody has to own that mapping, and getting it wrong is how you get prompt injection.
- **The target may not exist yet.** Waking a hibernating Durable Object, resuming a suspended Fly Machine, or holding an event for a laptop that's closed is categorically different from POSTing a URL.

**Commercially: no, and that's what matters.** Every one of those four properties has already been implemented by someone shipping:

| Property | Already owned by |
|---|---|
| Ack/work split | Hookdeck (queue + async), Trigger.dev (waitpoints), Linear's own SDK guidance |
| Semantic dedupe | AgentsRoom (cross-machine event lock), Hermes (in-tree idempotency, Apr 2026) |
| Payload → prompt mapping | AgentsRoom (`{{event.*}}`), Hermes (route templates), Gobii |
| Wake a non-existent target | Cloudflare Agents (DO hibernate/wake), Fly autostart, AgentsRoom (week-long offline queue) |

Differentiation that everyone has is positioning, not a moat.

### What would you actually have to own?

Ranked by defensibility, which is roughly inverse to how fun they are to build:

| Layer | Own it? | Why |
|---|---|---|
| **Ingress + HMAC catalog** | ❌ No | Pure commodity. Hookdeck ships **160+ preconfigured source types**. You cannot out-catalog that, and it's free at your volumes anyway. |
| **Durable buffer / replay** | ❌ No | Svix gives away 50k/mo, Hookdeck 10k/mo, Cloudflare Queues 10k/day. Rent it. |
| **The runtime / sandbox** | ❌ **Emphatically no** | Cold-start is a solved, brutally competitive race: Blaxel ~25ms resume, Daytona sub-90ms, E2B 300–500ms, Fly suspend-resume few-hundred ms, Modal 2–4s. You cannot compete and you don't need to — Shipyard's whole premise is the agent runs on hardware the user already owns. |
| **Identity / secret custody** | ⚠️ Maybe, but late | Holding the GitHub App and Linear OAuth app so the user never touches a webhook secret is genuinely valuable — and **Vercel Connect already shipped it** for eve, swapping provider signatures for Vercel OIDC. Doing this well means being an OAuth provider with multi-tenant token vaulting. That's a company, and it's Vercel's now. |
| **Queues** | ❌ No | Rent. |
| **The untrusted-payload → agent-action boundary** | ✅ **Yes — this is the only unclaimed layer** | Nobody sells it. See §5 and §6. |

**The wedge, stated plainly:** "webhook → spawn/resume agent" is *differentiated enough to describe* and *not differentiated enough to defend*. You would be building an adapter whose two endpoints are both racing to eliminate it — gateways adding agent plugins from one side, runtimes adding native HMAC ingress from the other. Hermes went from zero to HMAC + idempotency + rate limits + templates in one merged PR in April. That's the clock speed you'd be competing against, on a free product.

---

## 5. Risks

**① Platform absorption — already happening, not a future risk.**
This is not "Linear might build this someday." Hermes merged it (2026-04). Svix shipped runtime plugins (2026-07). Vercel Connect took secret custody (2026-08 docs). Anthropic ships `/fire`. GitHub Agent HQ is becoming the ingress *and* the runtime for coding agents. A standalone product here has a shrinking gap on both sides simultaneously, which is the worst shape a wedge can have.

**② Abuse and cold-start SLOs.**
A public URL that *starts a compute job* is a denial-of-wallet target in a way a relay never is. Anthropic's `/fire` retry-creates-a-session behavior shows the failure mode — every duplicate is a billed session. Mitigations are known (AgentsRoom's burst limit, "at most one run per window"), but they mean your SLO is now "agent started within N seconds," which depends on a runtime you don't control. If the agent is on a user's closed laptop, your SLO is *unbounded* — AgentsRoom's honest answer is "queued up to a week." Selling a latency promise you can't keep is worse than not selling one.

**③ Secret custody.**
To verify Stripe/GitHub/Linear signatures you hold their signing secrets — for every tenant. That's a credential vault with the blast radius of "read every event flowing through every customer's business." It demands SOC 2, key rotation, encryption at rest, and an incident plan, before the first paying customer. Hookdeck advertises SOC 2 on its **free** tier. That's the floor, and it's expensive to reach for a sidequest.

**④ Multi-tenant isolation.**
If you ever host the runtime, you're running untrusted agent code with shell access. That means Firecracker-class microVM isolation (E2B, Blaxel, Daytona all use it) plus egress policy. It's a serious infrastructure company. **Staying BYO-runtime avoids this entirely — which is another argument for beachhead-only.**

**⑤ Prompt injection — the underpriced one.**
The moment a webhook payload becomes prompt text, every field is attacker-influenced. A Sentry error title, a GitHub PR body, a Linear comment — all user-writable, all landing in an agent with your tokens and your shell. Google observed a **32% increase** in malicious injection payloads in web content between Nov 2025 and Feb 2026; vendor testing reports success rates as high as 84% against agentic systems (treat that specific figure as vendor-reported). OpenClaw's own docs default `allowUnsafeExternalContent` to off and recommend routing untrusted content through read-only reader agents. **The industry has priced this at zero and it is the highest-severity property of the whole architecture.**

**⑥ The market's own security record is a liability *and* the opportunity.**
OpenClaw: 25,000 stars in a day, then 40,000+ exposed instances, ~63% remotely exploitable, ~12,000 RCE-able, CVE-2026-25253 (localhost-trust WebSocket token exfiltration, patched 2026.1.29), plus 800+ malicious skills (~20% of the registry) in the ClawHavoc campaign. Any product touching this install base inherits reputational exposure — and simultaneously, that record is the single best argument for why the security-framed version of this product should exist.

---

## 6. Recommendation

### Go / no-go: **No-go as a standalone business. Beachhead-only.**

Build hosted ingress as a **Shipyard capability** — the managed counterpart to the `tailscale-webhooks` plugin, for domain agents whose operators don't have or don't want a tailnet. Keep it in the monorepo. Keep it a feature. Don't name it, don't price it, don't spin it out.

**The three reasons, compressed:**

1. **The product exists and is free.** AgentsRoom ships the brief verbatim at $0. Gobii ships it as MIT OSS with a $50/mo hosted tier. You would be launching into a market where the reference implementation is free and the priced version is open source.
2. **Both flanks are closing.** Gateways added agent plugins (Svix, Hookdeck — July 2026). Runtimes added native verified ingress (Hermes — April 2026). The adapter you'd sell is being squeezed from both directions by parties with more distribution.
3. **The winning shape isn't yours.** Every shipping competitor buffers and lets the agent *pull* (Svix 5s poll, Hookdeck CLI WebSocket, WebhookAgent heartbeat). The valuable asset is the durable buffer, not the public HTTPS ingress — and the buffer is exactly the thing Svix and Hookdeck give away at your entire addressable volume.

### But: there is one real business next door, and it isn't webhooks

The unclaimed layer is **§4's last row**. Not "deliver the event" but **"make an internet-triggered agent on hardware you own defensible."** Concretely: verified ingress *plus* payload quarantine — structured field extraction instead of raw-JSON-into-prompt, injection scanning on attacker-writable fields, capability scoping per trigger (this webhook may only wake *this* agent with *these* tools), egress policy, and an audit trail of what the payload asked for versus what the agent did.

The market is 389k-star OpenClaw + 243k-star Hermes, whose own docs say *never expose this*, whose install base has 40k+ internet-exposed instances and a CVSS-8.8 RCE in its history, and where the current answer is a free tunnel and hope. That's a security product for people who have already demonstrated they have the problem, sold into a runtime whose maintainers are shipping the *delivery* half for free — which is fine, because delivery was never the defensible half.

That framing is a genuinely different business from "autonomous webhooks," and it's the one I'd test.

### Three sharp validation steps

**1. Falsify the willingness-to-pay, in two weeks, by talking to the population that already chose a competitor.**
Not "would you use this" — that always yields yes. Go to AgentsRoom's Discord/download cohort and the OpenClaw + Hermes communities and ask three questions with a right answer: *(a)* What is currently reaching your agent from the outside, and how did you wire it? *(b)* Have you ever had a duplicate or lost agent run from it — what did it cost? *(c)* You're using a free tool for this; what would have to be true for you to pay $15/mo? **Kill criterion:** if fewer than 1 in 5 can name a concrete duplicate/lost-run incident with a cost attached, the delivery product is dead and only the security framing survives. Also ask the security question cold — *"has an agent of yours ever acted on something in a webhook payload that it shouldn't have?"* — and watch whether the energy in the conversation changes. That signal is the whole decision.

**2. Build the stub as the Shipyard `tailscale-webhooks` hosted mode — one week, zero new repos.**
A single verified-ingress endpoint that terminates `Linear-Signature` (bare hex HMAC-SHA256 over raw body, `webhookTimestamp` inside ~60s), acks in under 5s, dedupes by `(issue_id, action)` rather than delivery ID, and drives the Shipyard Linear agent. Instrument three numbers and nothing else: **time-to-ack**, **time-to-first-activity** (against Linear's 10s deadline), and **duplicate-run rate**. This is on Shipyard's locked build order anyway — sequencing it *fourth* as planned, but with these three metrics wired in, converts a milestone into an experiment for free. **Decision output:** if hitting 5s/10s reliably turns out to be genuinely hard, that difficulty *is* the product and step 3 gets more interesting. If it's a weekend, it's a feature and the sidequest closes.

**3. Probe the absorption clock with a two-hour spike, before building anything.**
Wire Shipyard's Linear agent to Anthropic's `routines/{id}/fire` and try to break it deliberately: fire the same routine twice (confirm two billed sessions — the docs say there's no idempotency key), hit the daily cap and inspect the 429 `Retry-After`, and try pointing a raw GitHub webhook at it (it will fail — wrong auth model, `text` is freeform and unparsed). Then do the same against Vercel Connect + eve's Linear channel, where the platform holds the app and the secret. **What you learn:** whether the gap between "provider HMAC POST" and "vendor agent trigger" is a durable seam or a six-month one. If Anthropic ships idempotency keys and HMAC ingress on `/fire` — a small, obvious change — the delivery product evaporates overnight. Knowing how load-bearing that single missing feature is should gate any further investment.

---

## Sources

**Webhook gateways / relays**
- Svix Ingest — https://www.svix.com/ingest/ · Pricing — https://www.svix.com/pricing/
- Svix, "Receive webhooks in OpenClaw and Hermes" (2026-07-28) — https://www.svix.com/blog/receive-webhooks-in-openclaw-and-hermes/
- Svix funding (a16z, Feb 2023) — https://www.svix.com/blog/new-round-of-funding-led-by-a16z/
- Hookdeck Event Gateway — https://hookdeck.com/event-gateway · Pricing — https://hookdeck.com/pricing
- Hookdeck, "How Developers Connect Webhooks to AI Agents, MCP Servers, and LLM Tools" — https://hookdeck.com/webhooks/guides/webhooks-ai-agents-integration-patterns
- Hookdeck for Hermes Agent — https://hookdeck.com/blog/hookdeck-for-hermes-agent
- Hookdeck with OpenClaw — https://hookdeck.com/webhooks/platforms/using-hookdeck-with-openclaw-reliable-webhooks-for-your-ai-agent
- Hookdeck, building Linear agents with the CLI — https://hookdeck.com/webhooks/platforms/how-to-build-linear-agents-with-hookdeck-cli
- Hookdeck agent skills — https://github.com/hookdeck/agent-skills
- Hookdeck, "Anthropic shipped webhooks for Claude Managed Agents" — https://hookdeck.com/blog/anthropic-managed-agent-webhooks
- Convoy status / OSS comparison — https://www.svix.com/alternatives/convoy/
- Pipedream pricing — https://pipedream.com/docs/pricing

**Tunnels / ingress**
- Tailscale Funnel — https://tailscale.com/docs/features/tailscale-funnel · Examples — https://tailscale.com/docs/reference/examples/funnel
- Hookdeck, Cloudflare Tunnel alternatives for local webhook dev — https://hookdeck.com/webhooks/platforms/cloudflare-tunnel-alternatives-for-local-webhook-development
- ngrok pricing overview — https://www.g2.com/products/ngrok/pricing

**Durable execution / runtimes**
- Inngest pricing — https://www.inngest.com/pricing · AI/AgentKit — https://www.inngest.com/ai
- Inngest webhook transforms — https://www.inngest.com/docs/platform/webhooks
- Trigger.dev pricing — https://trigger.dev/pricing · Product — https://trigger.dev/product
- Temporal for AI — https://temporal.io/solutions/ai
- Cloudflare Agents docs — https://developers.cloudflare.com/agents/ · Long-running agents — https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/
- Cloudflare Queues pricing — https://developers.cloudflare.com/queues/platform/pricing/ · Workers pricing — https://developers.cloudflare.com/workers/platform/pricing/
- Fly.io autostop/autostart — https://fly.io/docs/launch/autostop-autostart/ · Suspend/resume — https://fly.io/docs/reference/suspend-resume/
- Modal cold start — https://modal.com/docs/guide/cold-start
- Agent sandbox cold-start/pricing comparison — https://www.marktechpost.com/2026/08/27/best-agent-sandboxes-2026-cold-start-pricing-network-policy/
- Blaxel sub-second sandbox startup — https://blaxel.ai/blog/sub-second-sandbox-startup-2026

**Vendor agent ingress**
- Anthropic, trigger a routine through the API (`/fire`) — https://platform.claude.com/docs/en/api/claude-code/routines-fire
- Anthropic Managed Agents sessions — https://platform.claude.com/docs/en/managed-agents/sessions
- Claude Code routines guide — https://betterstack.com/community/guides/ai/claude-code-routines/
- Linear, developing the agent interaction — https://linear.app/developers/agent-interaction · Agents getting started — https://linear.app/developers/agents
- Linear webhook signature specifics — https://hookdeck.com/webhooks/platforms/guide-to-linear-webhooks-features-and-best-practices
- OpenAI webhooks — https://platform.openai.com/docs/guides/webhooks
- Cursor Cloud Agents API — https://cursor.com/docs/cloud-agent/api/endpoints · TypeScript SDK — https://cursor.com/blog/typescript-sdk
- GitHub, about third-party coding agents — https://docs.github.com/en/copilot/concepts/agents/about-third-party-coding-agents
- Vercel, introducing eve (2026-06-17) — https://vercel.com/blog/introducing-eve
- Vercel Connect + eve (credential/webhook custody) — https://vercel.com/docs/connect/frameworks/eve
- eve Linear channel — https://eve.dev/docs/channels/linear

**Direct competitors**
- AgentsRoom webhook triggers — https://agentsroom.dev/features/webhook-triggers
- Gobii — https://gobii.ai/ · Pricing — https://gobii.ai/pricing/ · Webhooks docs — https://docs.gobii.ai/developers/webhooks · Platform (MIT) — https://github.com/gobii-ai/gobii-platform
- WebhookAgent — https://webhookagent.com (product claims unverified; site is primarily an SEO content hub)
- Zapier Agents pricing/limits — https://www.usecarly.com/blog/zapier-agents-alternatives/

**Agent runtimes & their install base**
- OpenClaw — https://github.com/openclaw/openclaw (389,197★ as of 2026-09-08, GitHub API) · Gateway security — https://docs.openclaw.ai/gateway/security · Tailscale — https://docs.openclaw.ai/gateway/tailscale
- Hermes Agent — https://github.com/NousResearch/hermes-agent (243,237★, MIT, created 2025-07-22, GitHub API 2026-09-08)
- Hermes PR #12473, direct delivery mode (merged 2026-04-19) — https://github.com/NousResearch/hermes-agent/pull/12473
- Hermes issue #491, webhook-triggered agent sessions (open, 2026-03-06) — https://github.com/NousResearch/hermes-agent/issues/491
- n8n self-hosting / homelab adoption — https://northflank.com/blog/how-to-self-host-n8n-setup-architecture-and-pricing-guide
- OpenClaw managed hosting price range — https://blink.new/blog/openclaw-managed-hosting-comparison-2026

**Security**
- OpenClaw CVE-2026-25253 analysis — https://www.proarch.com/blog/threats-vulnerabilities/openclaw-rce-vulnerability-cve-2026-25253
- OpenClaw security crisis timeline — https://www.adminbyrequest.com/en/blogs/openclaw-went-from-viral-ai-agent-to-security-crisis-in-just-three-weeks · https://conscia.com/blog/the-openclaw-security-crisis/
- Indirect prompt injection in the wild (CSA, 2026) — https://labs.cloudsecurityalliance.org/research/csa-research-note-indirect-prompt-injection-in-the-wild-2026/
- Unit 42, web-based indirect prompt injection observed in the wild — https://unit42.paloaltonetworks.com/ai-agent-prompt-injection/
- OWASP / production agentic failures (2026-06) — https://www.helpnetsecurity.com/2026/06/11/owasp-prompt-injection-ai-security-failures/

**Note on sourcing:** star counts and repo metadata were read directly from the GitHub API on 2026-09-08. Pricing was taken from vendor pricing pages where available. A few landscape figures (ngrok tier pricing, Temporal's Series D, OpenClaw exposure counts, the 84% agentic-injection success rate) come from secondary or vendor-authored sources and are flagged inline where they carry weight in the argument.

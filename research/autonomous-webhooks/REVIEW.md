# Review: refined thesis — hardened ingress + injection quarantine

**For:** Jordan · **Date:** 2026-09-08 · **Re-reviews:** [`REPORT.md`](./REPORT.md) against [`REVIEW-BRIEF.md`](./REVIEW-BRIEF.md)

---

## TL;DR

**Partial dodge — and the split is backwards.**

The refined thesis clears the two failure modes that killed the original ([REPORT §4](./REPORT.md), §5④): you no longer race Blaxel/Daytona/E2B on cold starts, and you never touch multi-tenant isolation. It also moves you from the $0–10/mo buyer (REPORT §3①) to the $100–1,000/mo buyer (REPORT §3④). Real progress.

But three things came out of this pass:

1. **A correction to REPORT §6.** I wrote that nobody sells injection quarantine. That's wrong at the enterprise tier — the category has been rolled up: Lakera→Check Point (~$300M, Q4 2025), Prompt Security→SentinelOne, Protect AI→Palo Alto (Prisma AIRS), Robust Intelligence→Cisco AI Defense, CalypsoAI→F5, Aim→Cato, Invariant→Snyk. The accurate claim is narrower: **nobody sells it for self-hosted, no-egress, customer-owned agent runtimes.** Zenity and peers are cloud-oriented and stop at the boundary you'd start at. The gap is real; the category is not empty.
2. **The hosted/customer split in the brief is inverted.** Herdr's control API is a **local Unix socket with no auth and no network listener** ([socket API docs](https://herdr.dev/docs/socket-api/)). Anything that reaches `~/.config/herdr/herdr.sock` can call `agent.start`. That means an on-box connector is *mandatory*, and the only place capability scoping can actually be **enforced** is on-box. Hosted quarantine can inspect content; it cannot contain behaviour. **The connector is the product. The hosted ingress is the commodity front.**
3. **The thing that sells this is a demo, not a deck.** See §6, step 1.

**Verdict: narrow beachhead — build the connector, rent the ingress, and don't call it a company until §6 step 1 produces a video.**

---

## 1. Does this dodge the no-go?

| REPORT's reason for no-go | Dodged? | Why |
|---|---|---|
| Runtime/sandbox race is unwinnable (§4) | ✅ **Yes** | Customer owns compute. You never enter the cold-start race. This was the strongest structural objection and it's gone. |
| Multi-tenant isolation → serious infra company (§5④) | ✅ **Yes** | No untrusted code on your metal. Eliminated, not mitigated. |
| Buyer's willingness-to-pay is ~$0–10/mo (§3①) | ✅ **Mostly** | Reframing as security moves you to segment ④. Security budgets are real budgets. Caveat: segment ④ is also the *smallest* segment, and REPORT §3's "uncomfortable pattern" still holds. |
| Secret custody demands SOC 2 before customer #1 (§5③) | ❌ **No** | Unchanged, and slightly worse: you're now also making a *security* claim, so the bar for your own posture is higher. You still hold every tenant's provider signing secrets. |
| Ingress + HMAC catalog is commodity (§4) | ❌ **No** | Hookdeck's 160+ source types and Svix's free 50k/mo still exist. If you build this half yourself you're burning runway on the part that's free. |
| Both flanks closing (§6②) | ⚠️ **Partial** | The *delivery* flank is still closing (Hermes merged HMAC + idempotency in [PR #12473](https://github.com/NousResearch/hermes-agent/pull/12473)). The *policy* flank is not — no agent runtime ships capability scoping per trigger. You've moved to the half that isn't converging. |

**Honest score: 3 of 6 dodged, and they're the three that mattered most architecturally.** But the refined thesis walks into a new problem the original didn't have — an adjacent category with six acquisitions in two years and security majors doing the buying. That changes the shape of the outcome (acquisition, not category creation) more than it changes go/no-go.

---

## 2. What's still undifferentiated

Reusing REPORT §2 and §4 rather than re-deriving:

**Verified public ingress + per-provider HMAC.** Unchanged. [Svix Ingest](https://www.svix.com/ingest/) free 50k/mo, [Hookdeck](https://hookdeck.com/pricing) free 10k/mo with 160+ preconfigured sources. Rent this. Building it is the single easiest way to waste six weeks.

**Durable buffer + replay + dedupe.** Unchanged (REPORT §4). Also now in-tree at Hermes.

**Wake-on-event from a hosted URL to a customer machine.** [AgentsRoom](https://agentsroom.dev/features/webhook-triggers) ships this free: per-provider HMAC, filter, burst limit, week-long offline queue, cross-machine lock. The refined thesis does not beat AgentsRoom on wake. It beats it on *what happens between verify and wake* — which is the whole bet.

**Outbound-only connector to a hosted buffer.** Hookdeck CLI already does exactly this (WebSocket tunnel); Svix's OpenClaw/Hermes plugins do it via 5s polling. The transport is solved and public. Don't sell the transport.

**Content-level injection detection.** The dual-LLM / quarantined-reader pattern is publicly documented in the [OWASP prompt-injection cheat sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html). Lakera-class classifiers run sub-50ms behind an API. **Detection is a commodity you should buy.** If "quarantine" means "we score the payload for injection," you have no product.

**Secret custody.** [Vercel Connect](https://vercel.com/docs/connect/frameworks/eve) already holds the Linear/GitHub/Slack app and swaps provider signatures for OIDC. REPORT §4 called this "maybe, but late." Still late.

---

## 3. What's actually ownable, ranked

| # | Asset | Defensibility | Why |
|---|---|---|---|
| **1** | **The on-box connector as the policy enforcement point** | **High** | It is the *only* place capability scoping can be enforced — Herdr's socket has no auth, so whoever owns the socket owns the boundary. It's also the wake mechanism, so it can't be unbundled. And it lives on the customer's machine: switching cost is real, and it's the one component neither Svix nor Lakera can ship because neither has a foothold on the box. |
| **2** | **Trigger→capability binding** ("this webhook may start *only* agent X, tool allowlist Y, no shell, no egress, no skill install") + the audit trail proving what the payload asked for vs. what the agent did | **Medium-high** | No agent runtime ships this. It's the deterministic half of quarantine — it *contains* rather than *detects*, so it doesn't rot as attacks evolve. Thin as technology; the value is the schema, the defaults, and the receipt. Defensible as a standard, not as code. |
| **3** | **Semantic dedupe + the Linear 5s/10s impedance layer** | **Low-medium** | Genuinely necessary and genuinely hard to get right — but REPORT §4 established Hookdeck, Trigger.dev waitpoints, and AgentsRoom's cross-machine lock all implement versions of it. Table stakes: you must have it, you can't win on it. |
| **4** | **Structured extraction (payload → typed fields, never raw JSON → prompt)** | **Low** | AgentsRoom's `{{event.*}}` mapping and Hermes route templates already do this. Worth doing correctly; not worth claiming. |
| **5** | **Content-level injection scanning** | **None** | Buy it. See §2. |
| **6** | **Hosted ingress + HMAC catalog + queue** | **None** | Rent it. Unchanged from REPORT §4. |

**The one-line version:** you own #1 and #2 or you own nothing. Everything below the line is procurement.

---

## 4. Threat model sketch

What quarantine must actually stop, in severity order. Note which are *contained* (deterministic, ownable) vs *detected* (probabilistic, rentable).

| Threat | Concrete shape | Control | Type |
|---|---|---|---|
| **Confused deputy** ⚠️ *the real one* | Anyone on the internet opens a Linear issue / GitHub issue; the agent processes it **with the operator's full credentials and shell**. The attacker never touches your infra — they use the legitimate channel. | Bind the trigger to a capability set, not to the operator's identity. Triggered runs get a *subset* of what an interactive session gets. | **Contain** |
| **Indirect prompt injection** | Instructions hidden in issue body, PR title, commit message, Sentry error text — all attacker-writable, all landing in prompt context. | Structural isolation (untrusted text never enters the tool-holding context); classifier on high-risk fields as defence-in-depth. | Contain + detect |
| **Malicious skill / supply chain** | Triggered run installs or invokes a poisoned skill. ~800 malicious skills (~20% of registry) in the ClawHavoc campaign (REPORT §5⑥). | Pin the skill set per trigger; **deny skill install during any triggered run.** Cheap, deterministic, high value. | **Contain** |
| **Replay / duplicate runs** | Provider retry or replayed delivery spawns a second run. Anthropic's `/fire` has no idempotency key — every retry is a new billed session (REPORT §2E). | Timestamp window + dedupe on a *semantic* key (`issue_id`+`action`), not delivery ID. | **Contain** |
| **Denial-of-wallet** | Burst of events → many concurrent paid agent runs. | Burst window + concurrency cap. AgentsRoom already ships "one run per window." | **Contain** |
| **Socket squatting** | Herdr's socket has **no auth**; any local process can `agent.start`. Your connector doesn't create this hole, but it must not widen it. | Connector is the sole trusted client; filesystem perms; refuse to act as a generic proxy. | **Contain** |
| **Egress / exfiltration** | Injected agent posts a token or source to an attacker endpoint. | Egress allowlist at the connector. Hard on a general-purpose dev box — be honest about partial coverage. | Contain (partial) |

**Read:** five of seven are *containment* problems solvable with deterministic policy — which is the good news, because containment is what §3's #1 and #2 sell, and it doesn't degrade as attacks improve. Lead with contain, not detect.

---

## 5. MVP shape

### Settled by the Herdr socket API — not a design choice

[herdr.dev/docs/socket-api](https://herdr.dev/docs/socket-api/): NDJSON over a Unix domain socket at `~/.config/herdr/herdr.sock` (per-session sockets under `sessions/<name>/`), Windows named pipe. **No network listener. No auth model** — access is filesystem permissions only.

Two consequences, both decisive:

- **On-box connector is mandatory.** There is no way to reach Herdr from the internet. "Host vs on-box" is not a question.
- **Push vs pull is settled: pull.** The connector holds an outbound connection to your buffer. This matches every shipping competitor (REPORT §1: Hookdeck CLI WebSocket, Svix 5s poll) and is the only NAT/firewall-safe option.

### Smallest shippable

**Rent** (do not build): verified ingress + durable buffer. Start on a Svix or Hookdeck free tier. If the thesis works you can insource later; if it doesn't you've spent $0.

**Build:** `shipyard-connector` — one on-box binary that:
1. Holds an outbound connection to the buffer; pulls events.
2. Evaluates **trigger policy** (which agent, which tools, which repo, install-skills: never).
3. Extracts typed fields; wraps untrusted text in inert delimiters. Optional classifier call.
4. Calls Herdr `agent.start` / `agent.prompt` over the socket.
5. Subscribes via `events.subscribe`, reports state back, acks the event.
6. Writes the receipt: payload asked X, policy allowed Y, agent did Z.

### Herdr fire-API assumptions — verified, with two risks

`agent.start`, `agent.prompt`, `agent.wait`, `agent.list`, `pane.run`, `pane.send_text`, `events.subscribe`, `session.snapshot`, plus `workspace.*`/`tab.*`. The primitives the brief assumes **exist and are documented**. Named sessions are socket namespaces, so "fire a named session" maps cleanly.

Two risks worth pricing:
- **API churn.** [`herdrdev/herdr`](https://github.com/herdrdev/herdr) is ★36,452, Apache-2.0, created **2026-03-27** — five months old and pushed today. A connector against a fast-moving socket API is a treadmill. Validate before committing (§6 step 3).
- **Install base.** Herdr 36k★ vs OpenClaw 389k★ + Hermes 243k★ (REPORT §2D). Herdr-only is shippable and dogfoodable; it is not the market. That's a deliberate v1 narrowing, not a strategy.

### Explicitly not in v1
Own gateway · own HMAC catalog · own queue · trained injection classifier · OpenClaw/Hermes support · human-approval flow · web dashboard.

---

## 6. Verdict and validation

### **Narrow beachhead — yes. Standalone company — not yet, and not on this evidence.**

Build `shipyard-connector` as a Shipyard capability, on rented ingress, Herdr-only. That is justified *today* purely as infrastructure Shipyard needs — the `tailscale-webhooks` milestone has to solve this anyway, and the connector shape is strictly better than a Funnel pipe (works behind NAT, survives the box being asleep, and puts a policy point where Funnel puts a hole).

What is **not** yet justified is the security SKU. The refined thesis is more interesting than the original, but it currently rests on an argument, not evidence. The three steps below are ordered to produce evidence cheaply, and the first one is worth more than the other two combined.

**1. Build the confused-deputy demo. (2 days — do this before anything else.)**
Stand up stock Herdr + Claude Code, wire a naive webhook wake, and file a Linear issue whose body contains an injection aimed at the operator's credentials. Record the screen. Three outcomes, all informative: *it complies* → you have the artifact that sells the entire thesis, and everything downstream gets easier; *it refuses cleanly* → the model is doing your job for free and the thesis is materially weaker, learned for two days; *it partially complies* → you've found the exact boundary the product defends, which is the best possible spec. **Kill criterion:** if you cannot produce a bad outcome against a realistic setup in two days, stop and reconsider — you're selling a fear the models have already priced in.

**2. Price the fear against the incumbent quote. (1 week, gated on step 1.)**
Take the video to 5 teams running agents on owned hardware. Ask what they'd pay — then ask whether they've been quoted by Lakera/Check Point, Prisma AIRS, or Zenity. **This is the question that determines the shape of the business:** if they already have an enterprise quote, you are a line item in someone's renewal and should plan for acquisition; if they've never heard of those vendors, you have a distribution gap into a segment the majors can't reach, and that's a company. Both answers are useful; not knowing is fatal.

**3. Connector spike against the live socket. (3 days.)**
Prove `agent.start` → `events.subscribe` → ack round-trip. Measure time-to-first-activity against Linear's 10s deadline. Then diff Herdr's socket API across its two most recent releases. **Decision output:** if the API churned in either release, the connector is a maintenance treadmill on a five-month-old dependency, and v1 should target a slower-moving runtime instead.

---

## 7. Open questions for Jordan

1. **Is the buyer someone other than you?** A hardened front door for Bagsy/Shipyard is infrastructure you should build regardless. A hardened front door as a product needs a stranger to pay. Which are we validating? These have different next steps and only one of them needs step 2.
2. **Detect or contain?** If quarantine means *scoring payloads*, you're reselling Lakera and you lose. If it means *capability scoping with a receipt*, you're selling architecture and you can win. §3 and §4 assume contain — confirm that's the intent, because it changes the entire build.
3. **Herdr-only, or is Herdr the wedge into OpenClaw + Hermes?** 36k★ vs 632k★ combined. Herdr-only is dogfoodable next week and caps the market at Shipyard's own footprint.
4. **Are you willing to be a security vendor?** SOC 2, CVE intake, a disclosure policy, incident response, and the reputational exposure of the OpenClaw ecosystem (REPORT §5⑥). That's the job description, and it's a different job from shipping plugins.
5. **What happens when quarantine blocks?** Drop, hold for human approval, or run degraded (read-only agent, no shell)? The approval path is a second product — Trigger.dev waitpoints already occupy it, and HumanLayer abandoned it. Pick one before building.

---

## Sources added in this pass

Prior claims cite [`REPORT.md`](./REPORT.md) inline. New this pass:

- Herdr socket API (transport, no network listener, no auth, command set) — https://herdr.dev/docs/socket-api/ · Persistence & remote (SSH-based) — https://herdr.dev/docs/persistence-remote/
- `herdrdev/herdr` — ★36,452, Apache-2.0, created 2026-03-27 (GitHub API, 2026-09-08) — https://github.com/herdrdev/herdr
- AI-security vendor consolidation (Lakera→Check Point ~$300M Q4 2025; Prompt Security→SentinelOne; Protect AI→Palo Alto; Robust Intelligence→Cisco; CalypsoAI→F5; Aim→Cato) — https://ctaio.dev/en/ai-security/llm-firewall-tools/ · https://senthex.com/en/lakera-alternatives/
- Zenity "cloud-oriented, stops short of self-hosted, air-gapped, no-egress runtime enforcement"; Invariant→Snyk agent-scan — https://edgelabs.ai/blog/zenity-ai-agent-security
- Dual-LLM / quarantined-reader pattern as published practice — https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html

*Sourcing note: the acquisition roll-up and the Zenity gap claim come from vendor-comparison sites, not primary filings. The direction is corroborated across sources; treat specific deal values as approximate. Herdr's API surface and repo metadata are primary (docs + GitHub API).*

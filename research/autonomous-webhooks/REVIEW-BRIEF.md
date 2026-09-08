# Review brief: refined sidequest thesis

Jordan read REPORT.md and refined the idea. **Re-review** — do not redo the full landscape from scratch. Stress-test this new thesis against your prior findings.

## Refined thesis (locked intent)

Jordan does **not** want to run workloads for customers.

He wants **hardened ingress + injection quarantine for self-hosted agents**:

1. **We host:** public signed ingress, per-provider verify, dedupe/replay buffer, **injection quarantine** (untrusted webhook payload must not become raw agent instructions / shell without a boundary).
2. **Customer owns:** the machine and the session runtime (Herdr-like: named sessions/agents on *their* Mac/VPS/compute).
3. **We do:** prove the event → quarantine/structure it → wake or resume a session **on their compute** (generic “fire up a session,” Herdr-shaped).
4. **We don’t:** run the model, host the coding VM, or sell managed agent compute.

Shipyard remains skills domain agents use *after* wake; this is the **hardened front door**, not Gobii/AgentsRoom-as-SaaS runtime.

## Your job

Write `REVIEW.md` in `/Users/glasner/code/shipyard/research/autonomous-webhooks/` (same folder as REPORT.md).

### Cover

1. **Does this dodge the no-go?** Honest yes/no/partial vs REPORT’s standalone verdict.
2. **What’s still undifferentiated** vs Svix/Hookdeck pull plugins, AgentsRoom, Hermes/OpenClaw native hooks, Vercel Connect.
3. **What’s actually ownable** — quarantine UX, customer-owned Herdr-like spawn protocol, Linear 5s/10s impedance, etc. Rank by defensibility.
4. **Threat model sketch** — what quarantine must stop (prompt injection via issue body, replay, confused deputy, malicious skill install). Keep short.
5. **MVP shape** — smallest shippable: host vs on-box agent; pull vs push; Herdr session fire API assumptions.
6. **Go / no-go / narrow-beachhead** for *this* refined thesis + 3 validation steps.
7. **Open questions** for Jordan (≤5).

### Constraints

- Read REPORT.md + this brief; cite prior report where you reuse a claim.
- Research only if needed to fill a gap; prefer judgment.
- Do not touch ~/code/bagsy or other product trees.
- Keep REVIEW.md shorter than REPORT.md (aim ~scannable, not another 40KB).

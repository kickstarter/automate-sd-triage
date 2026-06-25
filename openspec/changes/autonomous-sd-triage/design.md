## Context

SD triage is currently manual EM-on-call work. CS staff open tickets on the SD board; an EM reads each one (and its linked Zendesk thread), applies the `support-dev` label and other helpful labels, sets a priority per the Quick Priority SLA reference, and assigns the ticket to an owning product/foundation team. Automation already applies due dates from priority, and any EM may override them. Once routed, it is the receiving team's job to absorb the work into their sprint.

Two facts shape this design:

1. **The downstream backstop is real and trusted.** Receiving teams already evaluate and correct triage decisions, and re-prioritizing with a justification is a normal motion. So the safety mechanism is *post-action observability + cheap correction*, not a pre-action approval gate.
2. **This must outlive its author.** The explicit goal is a system the next owner can monitor and maintain — and that keeps working if the original author leaves the company.

This is a tight team, not a large enterprise, so the design deliberately avoids heavyweight ceremony (approval queues, formal bounce-back state machines, bespoke infrastructure).

## Goals / Non-Goals

**Goals:**

- Eliminate the EM-on-call triage time sink by performing priority, labeling, and routing autonomously.
- Be **survivable**: the whole system is one readable org repo + a few Jira-native rules; policy is human-editable config; identity is externalized so a bot account swaps in trivially.
- Be **monitorable**: every action is self-explaining (audit comment), the override rate is measurable, and a digest summarizes activity.
- Match the team's existing norm: change priority/due date *with a written justification*.

**Non-Goals:**

- Resolving or investigating tickets (the receiving team owns the fix; pattern-detection/root-cause work is separate).
- A pre-action human approval gate (the backstop is downstream by design).
- Sub-minute reaction time / webhooks (SLAs are in business days/weeks).
- Re-triaging tickets already handled by a human or the agent.
- Building a general-purpose triage platform; this targets the SD board specifically.

## Decisions

### D1. Hosting: GitHub Actions cron in an org repo

Run the agent as a scheduled GitHub Actions workflow in the org-owned `automate-sd-triage` repo.

- *Why:* Lowest bus factor that still allows real judgment. The entire system is one repo any engineer can read; the schedule is a yaml cron; run history and logs are free in the Actions tab; secrets live in org/repo settings (rotatable). Nothing runs on a laptop or a personal cloud.
- *Alternatives considered:* (a) **Standalone service (Lambda/Cloud Run)** — more infra and code for the team to own, higher bus factor. (b) **A hosted long-running worker** — needs uptime ownership. (c) **Pure Jira-native automation** — cannot do the priority/routing judgment; CS-set priority is known-unreliable. GitHub Actions wins on survivability-per-unit-complexity.

### D2. Trigger: polling, not webhooks

A cron poll (~every 20 min) runs the intake JQL and processes new tickets.

- *Why:* The most urgent SLA is "end of next business day," so triage latency of minutes is immaterial. Polling needs no inbound endpoint to host, secure, or let rot — strictly simpler and more survivable than webhooks.
- *Alternative:* Jira webhooks for near-real-time reaction — rejected as unnecessary complexity for this latency profile.

### D3. Split mechanical vs. judgment work

Mechanical steps (due-date from priority; any board/field move that can be expressed as a rule) stay in **Jira-native automation**, owned by Jira admins. The agent owns only judgment: priority, labels, routing.

- *Why:* Keeps the lowest-bus-factor work where the org already maintains it, and keeps the agent's surface area small.

### D4. Routing: deterministic table first, LLM fallback

Routing is a `routing-table.yml` lookup (component / keyword / signal → team) covering the common ~80%. Only genuinely ambiguous tickets fall back to LLM reasoning over the ticket plus the team directory.

- *Why:* Most routing is effectively a lookup; rules give near-zero error and the table is the artifact the next owner maintains. The LLM handles only the tail, where judgment is actually required.
- *Alternative:* LLM-routes everything — rejected as needlessly non-deterministic and harder to audit/maintain for the easy majority.

### D5. Full autonomy with a self-documenting audit comment

The agent writes changes directly (no approval gate) and posts a comment: e.g., *"Triaged → High, routed to Payments, because: payments component, 40+ backers affected, no workaround, per SLA guide §X."*

- *Why:* The backstop is downstream and trusted; the audit comment mirrors the existing human norm of justifying changes, so receiving teams already know how to read and respond to it. No new bureaucracy is introduced.

### D6. Credential indirection

All identity (Jira user, API token, base URL, Claude key) is read from secrets/config. No code path hardcodes or assumes the actor's name; no logic does "assign to me" or "skip tickets I reported."

- *Why:* Start running as Luke today, swap to a dedicated bot account later with a secrets edit and zero code change — the single most important move for survivability given no bot account is available yet.

### D7. Policy as human-editable config

Priority definitions stay in Confluence (read at runtime, single source of truth). Routing lives in `routing-table.yml`. A `README` explains the system end-to-end.

- *Why:* The next owner *edits config and reads a guide* rather than reverse-engineering code.

### D8. Idempotency via the `support-dev` label

The intake query excludes tickets already carrying `support-dev`; applying that label is the agent's commit marker.

- *Why:* Simple, visible, and uses an existing field — no separate state store to maintain. A human who triages first (adds the label) is naturally skipped.

### D9. Shadow mode before trusting it

A dry-run mode logs the decision it *would* make (and would-be audit comment) without writing to Jira, for an initial measurement window.

- *Why:* Lets us read the override/accuracy rate before leaning on the downstream backstop — cheap insurance, especially since SLAs are forgiving.

## Risks / Trade-offs

- **Distributed correction toil could exceed centralized triage toil if accuracy is low** → Run shadow mode first; gate go-live on an acceptable would-be override rate; treat override rate as the ongoing health metric.
- **Priority errors burn SLA clock (due date auto-applies); routing errors are cheaply reversible** → Hold a higher confidence bar on priority than routing; on low-priority-confidence, the agent still acts but flags uncertainty prominently in the audit comment for fast human correction.
- **Customer PII flows to the Claude API** → Require an explicit, documented data-handling decision (and any redaction) before go-live; record it in the README.
- **Running as a personal account looks wrong / breaks on departure** → D6 credential indirection makes the bot swap a secrets-only change; documented as a one-step runbook.
- **Confluence priority guide changes or moves** → Agent reads it at runtime and fails safe (skips priority change with a flagged comment) if it cannot fetch the guide, rather than guessing.
- **Ambiguous/no-clear-team tickets in full-auto** → Defined fallback: best-guess route with an explicit low-confidence flag in the comment (and/or a configurable holding team), never silent drop.
- **Cron overlap or partial failures** → Idempotent by design (D8); a run that fails mid-ticket is safe to re-run because already-labeled tickets are skipped.

## Migration Plan

1. Stand up the repo skeleton, config (`routing-table.yml`), secrets (Luke's creds), and the cron workflow disabled / shadow-only.
2. Run in **shadow mode** for an agreed window; review logged decisions and would-be override rate with the team.
3. Tune routing table and priority prompting from shadow findings.
4. Flip to **live** (writes enabled). Monitor override rate + weekly digest.
5. When a bot account is available, swap secrets (one-step runbook). No code change.

**Rollback:** a single config flag returns the system to shadow mode (or disable the workflow) — no customer-facing teardown required.

## Open Questions

- What is the exact intake JQL behind quick filters 1230 + 1264, and is `support-dev`-absent a sufficient "untriaged" predicate?
- What does "move into a team's space" mean mechanically in this Jira — a Team field, a component, a different project/board? (Determines the actuation write.)
- Data-handling: is sending ticket content (incl. PII) to the Claude API approved as-is, or is redaction required?
- Is there a sanctioned default/holding team for no-clear-owner tickets, or should those flag for a human?
- Confidence thresholds: what would-be override rate in shadow mode is "good enough" to flip live?

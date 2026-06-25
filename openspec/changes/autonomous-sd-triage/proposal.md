## Why

Support Dev (SD) triage — reading each customer-escalated ticket, setting an engineering-realistic priority, applying labels, and routing it to the owning team — currently lands on whichever engineering manager is on call. It is a recurring, judgment-heavy time sink, and the knowledge of *how* to triage lives in one person's head. We want to automate the triage layer so it runs unattended, survives any individual leaving, and is easy for the next owner to monitor and adjust.

The downstream safety net already exists and is trusted: receiving teams evaluate and correct triage decisions as a normal motion, and re-prioritizing / adjusting due dates with a written justification is already routine on this team. That makes full automation viable — the agent simply performs the same accepted human motion, earlier and unprompted.

## What Changes

- **New org repo `automate-sd-triage`** holds the entire system as readable, version-controlled config + code (no servers, no laptop scripts).
- **Scheduled poll (GitHub Actions cron, ~every 20 min)** detects new, untriaged intake tickets on the SD board. No webhook endpoint — SLAs are measured in business days/weeks, so triage latency is irrelevant.
- **Autonomous triage agent** reads each ticket (plus linked Zendesk context), then:
  - sets priority per the Quick Priority SLA reference (Confluence is source of truth),
  - applies the `support-dev` label plus other helpful labels,
  - routes to the owning team via a deterministic `routing-table.yml` lookup, falling back to LLM judgment only for the ambiguous tail,
  - writes the changes to Jira and posts an **audit comment** stating what changed and why (mirroring the team's existing justification norm).
- **Mechanical glue stays Jira-native** where possible (due-date application on priority; any board/field move), owned by Jira admins.
- **Credential indirection** from day one: identity lives entirely in secrets/config. Starts running as Luke's account; swapping to a dedicated bot account later is a secrets edit with zero code change. No logic ever assumes the actor's identity.
- **Observability built in**: a shadow (dry-run) mode for an initial measurement period, an audit trail on every action, override-rate tracking (the system's health metric), and a weekly digest.

## Capabilities

### New Capabilities

- `ticket-intake`: Detect new, untriaged SD intake tickets on a schedule, idempotently (the `support-dev` label marks a ticket as already processed).
- `triage-decisioning`: Produce the triage decision for a ticket — priority (per the SLA guide), labels, and owning team (deterministic routing table first, LLM fallback for ambiguity) — with a confidence assessment and a written rationale.
- `triage-actuation`: Apply a triage decision to Jira (priority, labels, team/move) and post the audit comment, acting through externalized credentials with no hardcoded identity.
- `triage-observability`: Shadow/dry-run mode, per-action audit trail, override-rate tracking, and a periodic digest so the system is monitorable and maintainable by a future owner.

### Modified Capabilities

<!-- None. This is a greenfield repo with no existing specs. -->

## Impact

- **New repo**: `automate-sd-triage` (org-owned). This `llm-tech-plans` repo remains personal drafting space only.
- **External systems**: Jira (read intake + write triage), Confluence (read priority guide), Zendesk context (via Jira links), Claude API (decisioning).
- **Secrets**: Jira credentials, Claude API key — stored as GitHub org/repo secrets, rotatable, never in code.
- **Jira config**: existing due-date automation retained; may add/keep Jira-native rules for mechanical moves.
- **Data handling**: customer ticket content (incl. PII) is sent to the Claude API during decisioning — requires an explicit, documented go-ahead and a redaction decision before go-live.

---
name: priority-audit
description: Fetches one or more batches of CHECK support-dev Jira tickets, then assesses whether the current priority assigned to each ticket is accurate — cross-referencing the space's Confluence priority guide and engineering factors like scale, severity, workaround availability, and current resource constraints. Outputs a structured prioritization report with recommended changes and reasoning. Use this skill whenever an engineering manager wants to review, audit, re-triage, or sanity-check the CHECK queue — even if they just say "show me what's open" or "pull the backlog." Also triggers when asked to fetch any set of CHECK tickets for review.
compatibility:
  mcp_servers:
    - Atlassian (Jira + Confluence)
    - Looker
---

# Priority Audit Skill

Pulls support-dev tickets from a space and gives the EM an honest, engineering-realistic assessment
of whether the priorities are right. Read `CLAUDE.md` for the always-on role and safety invariants and
`references/spaces.md` for space resolution.

---

## Step 1: Resolve the space and scope

Default space is **CHECK**; if the EM names another ("audit the FOO space"), resolve that row from
`references/spaces.md`.

Then confirm scope. If the EM was specific ("pull all open Highest/High"), proceed. Otherwise ask one
focused question and default to all open tickets in the space if none is given:

> "Which tickets should I pull? e.g. all open, a priority level, due this week, or a component/label?"

---

## Step 2: Fetch tickets from Jira

Build JQL from the resolved space row: lead with `project = <primary_key>` and the row's
`scope_filter`. On an **unexpected-empty** result, retry once widening to the row's `legacy_keys`
(`project in (CHECK, CHKT) …`) and tell the EM the fallback fired.

Example patterns (CHECK):

```
# All open
project = CHECK AND (labels = support-dev OR parent = CHECK-105) AND statusCategory != Done ORDER BY priority ASC, created DESC

# Open Highest + High
project = CHECK AND (labels = support-dev OR parent = CHECK-105) AND priority in (Highest, High) AND statusCategory != Done ORDER BY created DESC

# Overdue
project = CHECK AND (labels = support-dev OR parent = CHECK-105) AND statusCategory != Done AND due < now() ORDER BY due ASC
```

Request: `summary`, `description`, `priority`, `status`, `created`, `updated`, `due`, `labels`,
`components`, `reporter`, `assignee`, the latest 2–3 `comment`s (CS context), and `issuelinks`
(Zendesk). Read the Zendesk context where present. If >30 tickets, fetch in batches of 30 and tell the
EM the total.

---

## Step 3: Pull the priority guide

Read the space row's `priority_guide` Confluence URL and hold its definitions for Step 4. If the row
has **no** guide, or it can't be retrieved, use the fallback framework below and note that the
internal guide wasn't used.

**Fallback priority framework (only if no guide available):**

| Label | Criteria |
|-------|----------|
| Highest | Platform-wide outage or data loss; affects a large share of active users; no workaround |
| High | Significant user-facing failure; a notable cohort; limited/difficult workaround |
| Medium | Degraded experience for a subset; workaround exists; no data loss |
| Low | Minor UX issue or edge case; easy workaround; little impact beyond inconvenience |

---

## Step 4: Assess each ticket

Evaluate priority alignment across:

- **Scale** — how many users? Isolated (1–5), cohort (dozens–hundreds), or systemic (thousands+)?
  When the description is vague, **query Looker** for a scale estimate (see Step 4a) and cite it.
- **Severity** — data loss/corruption? A core flow blocked (backing, payouts, project edit)? Or
  degraded-but-functional?
- **Workaround** — does one exist, and is it reasonable for the user or does it need CS/eng?
- **Time sensitivity** — campaign ending, payout window, contractual deadline? Is the due date
  realistic given bandwidth?
- **Context quality** — enough to act on (repro, ids, errors, Zendesk link)? Flag thin tickets as
  **Needs triage info** regardless of priority.

EM-stated resource constraints override default criteria ("no Highest capacity this week") — factor
them in and note it in the report.

### 4a. Looker scale signal (read-only)

Use the Looker MCP to ground affected-user/scale numbers rather than guessing from CS notes — e.g.
`rosie_errored_pledges_over_time` (counts by project/backer) for PLOT-flavored issues, or `cs.model`
(Zendesk ↔ backings/projects) for how many backers/tickets cluster on a project. Looker is read-only;
it can't change anything.

### 4b. Per-ticket verdict

Output one of: ✅ **confirmed** · ⬆️ **upgrade** · ⬇️ **downgrade** · ⚠️ **needs triage info** — each
with reasoning. Any change recommendation must say why.

---

## Step 5: Produce the report

```
### CHECK Priority Audit
[Date] · [N] tickets · [scope]

#### 🔴 Highest (N)
| Ticket | Summary | Current | Recommended | Note |
|--------|---------|---------|-------------|------|
| CHECK-123 | Payment failure on checkout | Highest | ✅ Highest | 200+ backers, no workaround (Looker) |
| CHECK-456 | Creator dashboard 500 | High | ⬆️ Highest | Full block on project editing for cohort |

#### 🟠 High (N)  · 🟡 Medium (N)  · ⚪ Low/Lowest (N)
…

#### ⚠️ Needs triage info (N)
List tickets missing key context; name specifically what's missing.

#### Summary
- N upgrades · N downgrades · N confirmed · N need info

#### Notes for the EM
Cross-cutting observations (e.g. "3 of the High tickets share an error pattern — worth a root-cause
ticket"; offer to run `pattern-detection`).
```

---

## Step 6: Offer follow-ons

1. **Update priorities in Jira** — apply recommended changes. **Confirm before any write** (per
   `CLAUDE.md`); never change a priority without explicit approval.
2. **Run `pattern-detection`** on a visible cluster.
3. **Draft a triage summary** for standup/async.

---

## Notes

- Never update priorities without explicit confirmation.
- When confidence is low (vague description, no counts), say so — don't fabricate a recommendation.
- A retrieved guide overrides the fallback framework.

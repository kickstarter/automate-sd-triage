---
name: pattern-detection
description: Analyzes a set of CHECK support-dev Jira tickets to surface recurring patterns, common failure modes, and likely systemic issues. Produces a draft Jira ticket for root-cause investigation — with well-specified acceptance criteria, linked symptom tickets, and no over-reaching assumptions about the fix. Use this skill whenever an engineering manager wants to identify patterns in CHECK tickets, group related issues, draft a root-cause or investigation ticket for a senior engineer, or hand off a cluster of symptoms as a structured engineering task. Triggers on phrases like "are these related?", "write up a root cause ticket", "group these by issue type", "I want to hand this off", or "these feel systemic."
compatibility:
  mcp_servers:
    - Atlassian (Jira + Confluence)
    - Looker
---

# Pattern Detection Skill

Finds signal in a set of support-dev tickets and turns it into a well-formed investigation ticket a
senior engineer can run with — without over-specifying the solution. Read `CLAUDE.md` for the role and
safety invariants, `references/spaces.md` for space resolution, and
`references/investigation-ac-guide.md` when unsure how to frame acceptance criteria.

---

## Step 1: Gather the ticket set

Sources:

- **Explicit list** — "look at CHECK-101, CHECK-115, CHECK-132" → use those.
- **From a prior `priority-audit`** — if a cluster was just flagged, pull those ids.
- **Fresh query** — if none given, ask: "Which tickets should I analyze? A recent batch, a component,
  or a label?" Then query the resolved space (default CHECK) using the same registry +
  `project = <primary_key>` / CHECK-CHKT fallback strategy as `priority-audit`.

Useful JQL (CHECK):

```
project = "CHECK" AND (labels = support-dev OR parent = "CHECK-105") AND labels = "payment-failures" AND statusCategory != Done
project = "CHECK" AND (labels = support-dev OR parent = "CHECK-105") AND created >= -14d AND statusCategory != Done ORDER BY created DESC
```

Fetch full descriptions, all comments, linked Zendesk issues, labels, components, and reporter notes.

---

## Step 2: Analyze for patterns

- **Symptom similarity** — same user-facing error/broken flow, error messages, UI states?
- **User/cohort overlap** — same users, project types, account attributes? Clustered on a campaign,
  creator tier, or backer behavior? **Confirm cohort overlap in Looker `cs.model`** (Zendesk ↔
  backings/projects/users) rather than inferring it from descriptions.
- **Timing clusters** — created in a tight window (deploy/data event)? Periodic (month-end, payment
  windows)?
- **System/component overlap** — same component, endpoint, integration? CS notes pointing to one area?
- **Absent root cause** — individually ambiguous but collectively suggestive; same workaround across
  tickets (consistent failure mode)?

Looker is read-only — use it to ground cohort/scale signal, not to change anything. **If Looker can't
be reached, proceed without it:** infer cohort overlap from ticket descriptions and CS notes, and mark
it as unconfirmed. Never fabricate cohort/scale figures.

---

## Step 3: Form and confirm hypotheses

Identify 1–3 candidate patterns. For each: **name**, **symptom ticket ids**, **observed signals**,
**confidence** (High/Medium/Low), and **what's unknown**. Present before drafting:

> "I found 2 possible patterns in these 12 tickets — here's what I'm seeing. Does this match your
> read?"

Let the EM confirm, redirect, or merge before you draft anything.

---

## Step 4: Draft the investigation ticket

Scope it for a senior engineer to **investigate, not implement**.

**Summary:** `[CHECK Investigation] <short description of the observed failure pattern>`

**Description:**

```
## Background
[2–3 sentences: when noticed, how identified — e.g. "Generated from a cluster of 6 support tickets
identified during triage on [date], all describing a similar checkout failure."]

## Observed Symptoms
[Observations, not conclusions.]
- Users report [behavior X] when [action Y]
- Appears to affect [cohort/condition] per CS notes / Looker
- CS workaround involves [Z], suggesting the issue is [observable layer]
- Symptom tickets span [range], clustering around [date/event if any]

## What We Don't Know Yet
[The heart of the ticket — open questions for the investigator.]
- Deterministic or intermittent? Repro conditions?
- Exact failure point? (client, API, payment processor, our backend?)
- Logs/traces for affected requests?
- A regression? If so, what changed around [date]?
- How many distinct users are actually affected? (CS tickets may underrepresent — cross-check Looker)

## Acceptance Criteria
[ ] Root cause identified and documented here
[ ] Affected user count confirmed (or estimate with methodology)
[ ] Decision made: fix / workaround / defer — with justification
[ ] If a fix is warranted: follow-on implementation ticket created and linked
[ ] Symptom tickets updated with findings
[ ] [Optional] Monitoring/alerting gap identified

## Related Tickets
[Issue links to all symptom tickets]

## Notes for Investigator
[Only if genuinely useful: recent deploys in the area, flaky tests, CS contacts with the most context.]
```

Acceptance criteria describe what must be **learned or decided**, never how to implement the fix. See
`references/investigation-ac-guide.md`.

---

## Step 5: Dedupe, then create on confirmation

Before creating, **search the space for an existing open investigation/root-cause ticket** in the
same area. If one exists, recommend linking to it instead of creating a duplicate.

On confirmation (per `CLAUDE.md`, confirm before any Jira write):
1. Create the ticket (project = resolved space; issue type `Task` or `Investigation`; label
   `investigation`).
2. Link all symptom tickets with `relates to`.
3. Optionally comment on each symptom ticket — but at scale (10+), **confirm before posting** at all:
   > "Grouped into a root-cause investigation: [LINK]. No action needed here until it concludes."
4. Return the new ticket URL.

---

## Step 6: Offer follow-ons

1. **Assign** the ticket. 2. **Draft a handoff note** for the engineer. 3. **Adjust symptom-ticket
priorities** if the pattern implies it. 4. **Re-run on a wider set** if this was a subset.

---

## Notes

- **Don't prescribe the fix.** The investigator may find a cause unlike the symptoms.
- **Be honest about confidence.** A Medium pattern is worth investigating — say Medium.
- **CS context is data.** Quote CS notes when relevant.
- **Symptom comments are optional and noisy at scale.** Always confirm.

---
name: check-remediation
description: Fetches open CHECK support-dev Jira tickets due on a given date, classifies each by failure mode, and suggests concrete manual remediations — mapping symptoms to rosie/kickstarter console support-task service objects and the team's Guru runbooks. Accepts a due date; defaults to today's UTC date. Use this skill whenever an engineering manager wants to work the CHECK backlog for a due date: "what can we remediate that's due today/this week", "suggest fixes for the tickets due June 25", "how would I fix these PLOT/PM tickets", or after a priority-audit when they want to act on a cluster. Prepares verified console snippets (never executes) and can post them as labeled Jira comments.
compatibility:
  mcp_servers:
    - Atlassian (Jira)
    - Guru (Support team bot)
    - Looker
---

# Remediation Suggester Skill

Turns a due-date slice of a space's backlog into actionable, safety-checked **manual** remediations.
For each ticket it classifies the failure mode, maps it to a known fix (a rosie or kickstarter
support-task service object and/or a Guru runbook), and writes the exact steps with the right ids — or
flags honestly when no console remediation exists.

Read `CLAUDE.md` for the role and safety invariants, `references/spaces.md` for space resolution, and
**always read [`references/remediation-catalog.md`](references/remediation-catalog.md) before proposing
anything** — it holds the two-system support-task catalog, the Guru index, the symptom→remediation map,
the ID conventions, the Looker diagnosis paths, the pre-flight, the prototype-branch workflow, and the
blast-radius lesson. The catalog is the source of truth for the symptom→remediation **mapping** and
conventions; the **living `main` of the kickstarter/rosie GitHub repos is the source of truth for which
tasks actually exist** — its task tables are curated examples, so verify availability against live
`main` at runtime. This file is the procedure.

---

## Step 1: Resolve the space and due date

- **Space:** default CHECK; if the EM names another, resolve its row from `references/spaces.md`.
- **Remediation gate:** if the resolved row's `remediation` is not enabled (no catalog for that space),
  **decline and explain** — do not improvise a fix.
- **Due date:** if the EM gave one ("due June 25", "due 2026-06-25", "today", "this week"), use it. If
  **none**, default to **today's UTC date**, computed explicitly:
  ```bash
  date -u +%F   # e.g. 2026-06-30
  ```
  State the resolved date in your first response so the EM can correct it. "This week"/"this sprint" →
  a date range; otherwise a single `duedate`.

---

## Step 2: Fetch the tickets

Build JQL from the space row (CHECK shown), leading with `primary_key` + `scope_filter`, with the
CHECK/CHKT legacy fallback on unexpected-empty (announce when used):

```
project = "CHECK" AND (labels = support-dev OR parent = "CHECK-105") AND status != Done AND duedate = "2026-06-25" ORDER BY duedate ASC
```

Request: `summary`, `description`, `status`, `priority`, `labels`, `duedate`, `parent`, `assignee`,
`comment` (CS/Zendesk context). Read the Zendesk context where present. If the result is large and the
MCP writes it to a file, extract fields with `jq` rather than re-reading the whole payload.

---

## Step 3: Classify each ticket

Pull ids/links from the description (pledge/order/project/user admin URLs, Stripe links, ZD ticket) and
map the symptom to a failure class using the catalog's **symptom → remediation map**. Note the `system`
(rosie | kickstarter) the class routes to. When the class is ambiguous, say so and lead with a
diagnostic dry-run, not a state change.

---

## Step 4: Diagnose in Looker, then resolve ids

- **Diagnose read-only in Looker** (catalog "Diagnosis via Looker"): increment/pledge/PI state, and
  affected scope. Do not open a production console to read. **If Looker can't be reached, proceed
  without it:** fall back to the ticket's embedded diagnosis/CS context and human-run console reads,
  and flag that Looker scope-confirmation was skipped. The authoritative just-before-write check is
  in-console regardless, so this degrades diagnosis breadth, not write safety. Never fabricate state.
- **Resolve ids for the routed system** (catalog ID conventions): kickstarter console → kickstarter
  `Backing.find(<admin-URL id>)`; rosie console → `Pledge.find_by(client_id: 1, foreign_key: <backing
  id>)` then `pledge.id`. Per-increment commands also need the `PaymentIncrement` id from the pledge
  inspect page. Label which id is which.

---

## Step 5: Route to the correct system and console

From the catalog's `system` tag, open the named console — **never infer it**:

- **rosie** → `cd rosie && ksr console production`; rosie pledge ids; confirm the task's branch
  (PLOT-sync/refund tasks are on `demo/stripe-sync-plan-apply`, not `main`).
- **kickstarter** → kickstarter production console; kickstarter ids.

Running a kickstarter task in the rosie console (or passing the wrong system's id) is the
highest-blast-radius mistake — both are production.

**Verify the task exists** against the living `main` of the relevant GitHub repo before relying on it
(e.g. `gh api`/`gh search code`, or `git fetch && git ls-tree origin/main -- lib/support_tasks/`). If
it isn't on `main`, treat it as a prototype: link the working branch and follow Step 7.

---

## Step 6: Write the remediation per ticket

Produce:

1. **Classification + confidence** (high / medium / needs-diagnosis) and the routed `system`.
2. **Pre-flight** — the catalog's standard checks (capabilities, Stripe PI status confirmed
   in-console, dry-run).
3. **Remediation steps** — exact console snippets, dry-run before live, ids resolved and labeled,
   preferred service-object path first with the Guru manual equivalent as cross-check.
4. **Code link** — link the support task at its branch/path. If the fix would modify an on-`main` task
   or warrant a **new** task file, prepare a **prototype on a working branch** (see Step 7).
5. **Verify-after** — what to check to confirm success.
6. **Citation** — the Guru card / support-task file backing the approach.

If no console remediation exists (code fix, suspended capabilities, creator action needed), say so
plainly and name who it routes to. Never invent a fix.

---

## Step 7: Prototype working branch (when code changes are implied)

Per the catalog's prototype-branch workflow:

- **Existing task, as-is** → link it at its current branch/path; no new branch.
- **Modifies an on-`main` task, or a new task file makes sense** → open a **working branch** in the
  relevant repo (rosie or kickstarter, per the system tag) with the prototype implementation so it can
  be examined and tested with one-off remediations. **Confirm before pushing** the branch to origin
  (pushing is outward-facing); then capture the link.

Prototype code is for review/testing — it is never executed in production by this skill.

---

## Step 8: Safety rules

- **Dry-run first, always.** Read the report before any live run.
- **Never run anything live yourself.** This skill prepares verified-correct commands for a human to
  run in a production console; it does not execute production remediations.
- **Capabilities suspended → stop, tag T&S.**
- **Trace admin actions before claiming blast radius** (catalog blast-radius lesson). Grep the
  `engines/pledge_redemption/` engine (form field → controller → service) and confirm scope; don't
  speculate.
- **Looker is read-only and lags prod** — authoritative pre-write checks happen in-console/Stripe.

---

## Step 9: Output and follow-ons

Default output: a concise per-ticket briefing (classification, system, snippet, code link, citation)
for the resolved due date.

Offer, after presenting:

1. **Post to Jira** — drop the remediation as a comment on the relevant card(s). The comment's first
   line is exactly `Experimental: Suggested Remediation.`, it carries the runnable code blocks **and**
   any prototype-branch link(s), and it is **idempotent** (check for an existing such comment; update
   or skip rather than duplicate). **Confirm before posting.**
2. **Emit a multipage remediation doc** — an indexed folder (`README.md` + one page per ticket) like
   `support_dev/<date>-remediations/`, with prev/index/next links and a metadata table per ticket.
3. **Hand off** — if a cluster looks systemic, offer to run `pattern-detection` to draft a root-cause
   ticket.

---

## Notes

- This is judgment work: classification, system routing, and id resolution drive everything. When
  confidence is low, lead with diagnosis, not a state change.
- A scheduled routine could pre-stage a daily "due-today" briefing on top of this skill — but the skill
  is the reusable primitive; build the routine only once the skill is trusted, and keep it to the
  read/prepare path (no unattended posting or execution).

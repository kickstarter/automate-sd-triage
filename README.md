# automate-sd-triage

An attempt to automate as much customer support triage as we safely can.

This repo packages a Claude **CHECK support-dev triage** toolkit: a role (`CLAUDE.md`), a space
registry (`references/spaces.md`), and three skills under `.claude/skills/`:

| Skill | What it does |
|-------|--------------|
| `priority-audit` | Re-triage / sanity-check the CHECK queue; assess whether priorities are engineering-realistic. |
| `pattern-detection` | Find clusters across tickets; draft a root-cause investigation ticket for handoff. |
| `check-remediation` | Work a due-date slice of the backlog; prepare manual, safety-checked remediations. |

Everything reads from Jira/Confluence/Guru/Looker and **prepares** work for a human — it never executes
production remediations. See `CLAUDE.md` for the full safety posture.

---

## Setup

For internal Kickstarter teammates. Assumes Kickstarter SSO and standard access; if a connector below
is missing, request it through the usual access process.

### Prerequisites

- **Claude Code** (this repo is driven through it).
- **CHECK Jira access** (or access to the space you'll run against).
- **Clone all four supporting repos** as standard setup — they back the remediation snippets, ID
  resolution, and Looker reference:
  - `kickstarter` — monolith; `lib/support_tasks/` (backings, pledges, projects, users) + the
    `engines/pledge_redemption/` PM/orders engine.
  - `rosie` — payments; `lib/support_tasks/` (PLOT / increments / Stripe).
  - `rosie-api` — rosie API gem.
  - `looker` — LookML modeling both systems (read-only diagnosis + scope).

### 1. Clone and open

```bash
git clone <this repo>
```

Open the repo in Claude Code. The root `CLAUDE.md` (role + safety invariants) and the
`.claude/skills/` skills load automatically — **there is no skill-copying step**; the skills ship with
the repo.

### 2. Connect the MCP connectors

| Connector | Used by | Purpose |
|-----------|---------|---------|
| Atlassian (Jira + Confluence) | all skills | Fetch CHECK tickets; read the priority guide; create/link tickets; post comments |
| Guru (Support team bot) | `check-remediation` | Remediation runbooks (PLOT, pledge manager, payouts) |
| Looker | all skills | Read-only cross-system diagnosis and affected-scope confirmation |

Connect and authenticate them in Claude Code before running the skills.

### 3. Register your space (if not CHECK)

The skills are space-agnostic and resolve everything from `references/spaces.md`. To run against a
different space, add a row with its `primary_key`, `legacy_keys`, `scope_filter`, optional
`priority_guide`, and whether `remediation` applies. A space without a remediation catalog should set
`remediation` disabled — `check-remediation` will decline rather than improvise.

### 4. Safety posture (read before acting)

- Diagnosis and scope reads go through **Looker (read-only)**.
- Remediations are **prepared, never executed** — production console writes are **human-only**,
  dry-run first.
- Every Jira write (priority change, comment, ticket creation) is **confirmed** first.
- Suspended Stripe capabilities → stop and route to Trust & Safety.

Full invariants are in `CLAUDE.md`.

### 5. First prompts to validate the install

- **Priority audit:** "Audit the open priorities on the CHECK board."
- **Pattern detection:** "Any patterns worth investigating in the last 2 weeks of CHECK tickets?"
- **Remediation:** "What can we remediate that's due today?" (defaults to today's UTC date)

---

## Layout

```
automate-sd-triage/
├── CLAUDE.md                       role, TTR/priority defs, safety invariants, skill-routing map
├── references/
│   └── spaces.md                   space registry (default CHECK; add a row per team)
└── .claude/skills/
    ├── priority-audit/SKILL.md
    ├── pattern-detection/
    │   ├── SKILL.md
    │   └── references/investigation-ac-guide.md
    └── check-remediation/
        ├── SKILL.md
        └── references/remediation-catalog.md
```

# Space Registry

The triage skills (`priority-audit`, `pattern-detection`, `check-remediation`) are space-agnostic.
This registry is the single place that holds what varies per space. Default space is **CHECK**.

To onboard another team: add a row. To point a skill at a space, the EM names it ("audit the FOO
space") and the skill resolves the row below.

## Columns

| Field | Meaning |
|---|---|
| `space` | Friendly name the EM types (e.g. "CHECK") |
| `primary_key` | Jira project key used in JQL |
| `legacy_keys` | Old keys to fall back to on an unexpected-empty result (announce when used) |
| `scope_filter` | JQL clause that scopes the space's support-dev tickets |
| `priority_guide` | Primary Confluence priority guide for `priority-audit` (optional — blank ⇒ skip to fallback) |
| `priority_guide_fallback` | Secondary Confluence guide if the primary is missing/unavailable; if it too is blank, use the built-in framework |
| `remediation` | Whether `check-remediation` applies (requires a remediation catalog for the space) |

## Query strategy

Lead with `project = <primary_key>` plus the `scope_filter`. If that returns **zero** tickets when a
result was expected, retry once widening to the legacy keys (`project in (<primary_key>, <legacy_keys>)`)
and tell the EM the fallback fired. The goal is to drop legacy keys once they're fully retired.

**Reserved words:** `CHECK` is a reserved JQL word — always quote project keys: `project = "CHECK"`,
`project in ("CHECK", "CHKT")`. Quote issue keys in clauses too (`parent = "CHECK-105"`).

## Registry

### CHECK

| Field | Value |
|---|---|
| `space` | CHECK |
| `primary_key` | `CHECK` |
| `legacy_keys` | `CHKT` *(legacy — the space was migrated to CHECK shortly after creation; keep only as an unexpected-empty fallback)* |
| `scope_filter` | `labels = support-dev OR parent = "CHECK-105"` |
| `priority_guide` | https://kickstarter.atlassian.net/wiki/spaces/Checkout/pages/4488036353/SD+Ticket+Priorities+Framework+for+Checkout |
| `priority_guide_fallback` | https://kickstarter.atlassian.net/wiki/spaces/EN/pages/4649877514/Quick+Priority+SLA+Reference |
| `remediation` | enabled (catalog: `.claude/skills/check-remediation/references/remediation-catalog.md`) |

Example JQL (CHECK, due a given date):

```
project = "CHECK" AND (labels = support-dev OR parent = "CHECK-105") AND status != Done AND duedate = "2026-06-25" ORDER BY duedate ASC
```

<!-- Add new spaces below. A space without a remediation catalog should set remediation = disabled;
     check-remediation will decline rather than improvise. -->

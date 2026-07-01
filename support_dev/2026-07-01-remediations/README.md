# CHECK remediation briefing — 2026-07-01

Due-date slice of the CHECK support-dev backlog (`duedate = "2026-07-01"`, defaulted to today's UTC
date — no date was given), classified and mapped to concrete remediations per the `check-remediation`
skill.

**Looker was unauthenticated for this run.** Scope-confirmation via Looker is skipped for every ticket
below; diagnosis relies on ticket-embedded Stripe/admin context, with the authoritative just-before-write
check taken in-console immediately before any live run.

## Tickets

| Ticket | Summary | Priority | Classification | System | Confidence |
|---|---|---|---|---|---|
| [CHECK-306](check-306.md) | PLOT installment errored, 2 successful Stripe charges | High | State drift (corrected 2026-07-01 — was misclassified as duplicate charge) | rosie | High |
| [CHECK-304](check-304.md) | PLOT installment not attempted, still errored | High | Collection never ran | rosie | Medium |
| [CHECK-303](check-303.md) | PLOT installments errored, successful in Stripe (2 backers) | High | State drift | rosie | High |
| [CHECK-250](check-250.md) | Blank dialog fixing errored PLOT payment | Medium | UI symptom, state varies | rosie | Needs-diagnosis |
| [CHECK-236](check-236.md) | German translation edit | Low | Copy/i18n — no remediation | — | N/A |
| [CHECK-145](check-145.md) | Retry 3rd incremental payment | Lowest | Already resolved by policy | — | N/A |

## Cluster note

CHECK-304 and CHECK-250 share project 5269345. CHECK-303 and CHECK-306's drift/duplicate symptoms have
prior instances per CS's own triage comments (CHECK-256, CHECK-251). Candidate for a `pattern-detection`
pass — tracked separately, not run as part of this briefing.

## Catalog note

While verifying the catalog's task list against live `main`, found `SupportTasks::RefundDuplicateCharge`
(rosie `main`) — previously uncataloged. Added to
[`remediation-catalog.md`](../../.claude/skills/check-remediation/references/remediation-catalog.md).
No test coverage existed for the three project/deposit/ksr refund tasks as of this date; tests were
added and are up for review on `demo/plot-refund-tasks-tests` in rosie
([draft PR #4129](https://github.com/kickstarter/rosie/pull/4129)).

**Correction (2026-07-01):** CHECK-306 was originally misclassified here as a duplicate charge
(`RefundDuplicateCharge`). The EM corrected it on the ticket — it's state drift
(`SyncIncrementalPledgeToStripe`), same as CHECK-303. See [check-306.md](check-306.md) for the
corrected writeup and what went wrong with the original read.

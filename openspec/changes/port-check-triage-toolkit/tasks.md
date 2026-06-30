## 1. Shared context & registry

- [x] 1.1 Create root `CLAUDE.md` adapted from the source `ROLE.md`: assistant identity, TTR/priority definitions, the cross-cutting safety invariants (per `specs/triage-safety`), and a human-readable skill-routing map. Reference `references/spaces.md` for space resolution.
- [x] 1.2 Create `references/spaces.md` space registry with the documented columns (`space`, `primary_key`, `legacy_keys`, `scope_filter`, `priority_guide`, `remediation`) and the CHECK row (primary `CHECK`, legacy `CHKT`, scope `labels = support-dev OR parent = CHECK-105`, remediation enabled).

## 2. Port the read/triage skills

- [x] 2.1 Port `priority-audit` to `.claude/skills/priority-audit/SKILL.md`: resolve scope, query via the registry + CHECK/CHKT fallback strategy, use the space priority guide with built-in fallback, assess with Looker-grounded scale signal, produce the structured report, confirm before any priority write.
- [x] 2.2 Port `pattern-detection` to `.claude/skills/pattern-detection/SKILL.md` and bundle `references/investigation-ac-guide.md`: gather set, analyze with Looker `cs.model` cohort signal, confirm hypotheses before drafting, draft investigate-not-implement tickets, dedupe existing investigations, create/link only on confirmation.

## 3. Remediation catalog (two systems)

- [x] 3.1 Rebuild `.claude/skills/check-remediation/references/remediation-catalog.md`: tag every service object and symptom→remediation row with its `system` (rosie | kickstarter) and console command; record each rosie task's actual branch (PLOT tasks on `demo/stripe-sync-plan-apply`; `reactivate` + general payments on `main`) and instruct runtime branch/path confirmation.
- [x] 3.2 Enumerate the relevant kickstarter support tasks against `kickstarter/lib/support_tasks/` (e.g. `resync_backing_with_pledge`, `fix_orphaned_backing`, `drop_errored_pledges`, `sync_refund_checkout`), correcting the prior mis-attribution of `ResyncBackingWithPledge`.
- [x] 3.3 Update the catalog's ID-conventions, pre-flight, and blast-radius sections to the "which id in which console" model and the Looker-first read / in-console just-before-write check.

## 4. Port the remediation skill

- [x] 4.1 Port `check-remediation` to `.claude/skills/check-remediation/SKILL.md`: resolve due date (default today UTC), fetch the registry slice, gate on the `remediation` flag, classify by failure mode, diagnose in Looker, route to the correct system/console, produce per-ticket remediation with citation, decline honestly when no console fix exists, and post the `Experimental: Suggested Remediation.` comment (confirmed + idempotent).
- [x] 4.2 Add the prototype-working-branch workflow to `check-remediation` and the catalog: when a fix would modify an on-`main` task or warrant a new task file, open a working branch with the prototype (rosie or kickstarter per system tag), confirm before pushing to origin, and include the branch link plus runnable code blocks in the Jira comment; when an existing task is used as-is, link it at its current branch/path with no new branch.

## 5. Onboarding README

- [x] 5.1 Rewrite `README.md` for multi-machine onboarding (internal teammates; clone all four repos as standard setup): prerequisites, MCP connectors to authenticate (Atlassian, Guru, Looker), auto-loading skills, registering a space in `references/spaces.md`, the human-in-the-loop safety posture, and first-run prompts.

## 6. Verify

- [x] 6.1 Confirm the skills are discoverable and self-describe correctly (frontmatter `name`/`description`), CLAUDE.md loads the role + safety, and `references/spaces.md` resolves the CHECK row.
- [x] 6.2 Trace each skill's read paths to ensure diagnosis/scope go through Looker and any production-console step is prepared-for-a-human only (no live execution), per `specs/triage-safety`.

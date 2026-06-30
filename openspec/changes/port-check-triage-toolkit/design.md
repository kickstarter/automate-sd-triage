## Context

The CHECK support-triage toolkit exists today as a claude.ai-oriented package in `llm-tech-plans`
(`claude-skills/support-dev-triage/`): a `ROLE.md`, three skills (`priority-audit`,
`pattern-detection`, `remediation-suggester`), and two reference docs (`investigation-ac-guide`,
`remediation-catalog`). It is prompt-engineering, not code — its "implementation" is markdown that
steers Claude through Jira/Confluence/Guru via MCP and prepares (never executes) production console
remediations.

This repo (`automate-sd-triage`) already hosts skills in the Claude convention
(`.claude/skills/<name>/SKILL.md` with bundled `references/`), mirrored into `.gemini/` and
`.github/` for the OpenSpec tooling. The design problem is therefore mostly *structural and
contractual*: where artifacts live, how the role binds, how the board key and space are
parameterized, and how the new Jira write-back reconciles with the safety posture. No services,
schemas, or application code are introduced.

Constraints carried from the proposal:
- Real board is **CHECK**; `CHKT` is a legacy key kept only as an unexpected-empty fallback.
- The toolkit must be **shareable to other spaces** via a convenient interface.
- `check-remediation` now **posts prepared snippets as Jira comments**, prefixed
  `Experimental: Suggested Remediation.`
- Remediations draw on **two** support-task libraries — rosie `lib/support_tasks/` and kickstarter
  `lib/support_tasks/` (~37 tasks: backings, pledges, projects, users) — each run in its **own**
  production console. Branch reality is mixed: rosie's PLOT/increment-sync/refund tasks
  (`SyncIncrementalPledgeToStripe`, the increment refunds) live only on the working branch
  `demo/stripe-sync-plan-apply`; `reactivate_pledge_from_pledged_backing` and the general payments
  tasks are on `main`.
- Remediations link to code, not just inline snippets. When a fix would modify an on-`main` support
  task or warrant a new task file, a working branch holding the prototype is opened and linked.
- **Looker** is a connected, read-only warehouse modeling **both** systems (`rosie_*` views:
  pledges, payment_increments, payment_intents, funds_captures, errored_pledges_over_time; `ksr_*`
  views: backings, PLOT configs; `cs.model`: Zendesk ↔ backings/projects/users). It is the safe way
  to read across systems without a production console.
- Safety ("prepared, never executed") is a first-class requirement, not role tone.

## Goals / Non-Goals

**Goals:**
- Faithfully port the role + 3 skills + 2 reference docs into this repo as version-controlled
  artifacts, with the draft-era assumptions corrected.
- Unify all three skills onto the CHECK board, with a documented CHKT legacy fallback.
- Provide a single, low-friction mechanism to point any skill at a space other than CHECK.
- Establish the safety invariants as a single source of truth that every skill binds to.
- Define the `check-remediation` Jira comment write-back: when it writes, how it is labeled, and how
  it stays reconcilable with prepared-not-executed.
- Make Looker the default read-only path for cross-system diagnosis and scope confirmation, keeping
  operators out of production consoles for routine triage.

**Non-Goals:**
- Building the scheduled / unattended "due-today" routine (explicit proposal non-goal).
- Executing production remediations of any kind.
- Authoring remediation catalogs for non-CHECK spaces (only CHECK has the rosie/Guru mapping).
- Adding application code, services, data stores, or standing up new infrastructure. (Using
  already-connected MCP servers — Atlassian, Guru, Looker — is in scope; provisioning new ones is not.)
- Mirroring the triage skills into `.gemini/`/`.github/` (see Open Questions).

## Decisions

### 1. Repo layout: skills under `.claude/skills/`, role in `CLAUDE.md`

Port each skill to `.claude/skills/<name>/SKILL.md` with its `references/` bundled alongside,
matching the existing `openspec-*` skills. The role (`ROLE.md`) becomes the repo's root `CLAUDE.md`,
which is always in context for a session here.

- *Why:* skills self-trigger via their frontmatter `description`; the role's value that isn't
  trigger-routing — assistant identity, TTR/priority definitions, cross-cutting safety — is exactly
  the "always-on" context `CLAUDE.md` is for.
- *`.claude/` only:* the triage skills are authored canonically under `.claude/` and are **not**
  mirrored into `.gemini/`/`.github/` (unlike the OpenSpec skills). The operator runs this in Claude;
  mirroring is added later only if another tool actually needs it.
- *Alternative considered:* a custom agent under `.claude/agents/`. Rejected for a single-role repo —
  it adds an invocation seam without benefit. Revisit if a second, distinct role appears.
- *Consequence:* the role's per-skill "trigger when the EM says…" lists become partly redundant with
  skill `description`s. Keep a short routing map in `CLAUDE.md` for human readability, but skill
  descriptions are the source of truth for triggering.

### 2. Space parameterization: a space registry + a default

Introduce a small **space registry** in a standalone `references/spaces.md`, keyed by space, with one
row per space. Default is CHECK; pointing a skill elsewhere is "use space FOO" in natural language,
and the skill resolves the row to build its queries.

Registry columns:

| Field | Purpose |
|---|---|
| `space` | Friendly name the EM types ("CHECK", "FOO") |
| `primary_key` | Jira project key for JQL (`CHECK`) |
| `legacy_keys` | Fallback keys for unexpected-empty (`CHKT`) |
| `scope_filter` | The `labels = support-dev OR parent = CHECK-105` style clause |
| `priority_guide` | Confluence URL for `priority-audit` — **optional**; when blank, `priority-audit` uses its built-in fallback framework |
| `remediation` | Whether `check-remediation` applies (CHECK only, today) |

- *Why a registry over a free-form argument:* it captures everything that varies per space (key,
  fallback, scope clause, priority guide, remediation applicability) in one place, so onboarding a
  new team is "add a row," and skills stay space-agnostic.
- *Why a standalone `references/spaces.md`:* the "bring others along" goal implies the table grows;
  keeping it out of `CLAUDE.md` keeps that file focused on role + safety, and lets a space row be
  added without touching always-in-context prose. Skills reference it by relative path.
- *Alternative considered:* inline the table in `CLAUDE.md`. Rejected — fine for one space, noisy as
  spaces accumulate, and mixes config with role.

### 3. CHECK/CHKT query strategy: lead with CHECK, fall back on empty

Skills query `project = CHECK …` first. On an **unexpected empty** result, retry once widening to
the registry's `legacy_keys` (`project in (CHECK, CHKT) …`) and announce that the fallback was used.

- *Why:* CHKT is legacy and being retired; we want queries that will keep working when CHKT is gone,
  without silently missing tickets in the interim.
- *Mitigation against masking real empties:* fallback fires only when the primary returns zero and a
  result was expected, and it is always surfaced to the EM — never silent.

### 4. `check-remediation` write-back: comment, labeled, confirmed, idempotent

`check-remediation` may post its prepared snippet to the relevant CHECK card(s) as a Jira **comment**
whose first line is exactly `Experimental: Suggested Remediation.`

Rules:
- **Confirm before posting** by default — show the EM the comment and the target card first.
- The `Experimental: Suggested Remediation.` prefix is **mandatory** and non-removable; it signals the
  content is unvetted and machine-prepared.
- **Idempotency:** before posting, check the card for an existing `Experimental: Suggested
  Remediation.` comment; update/replace or skip rather than stacking duplicates on re-runs.
- The comment carries only **prepared** snippets (dry-run first, IDs resolved). It never carries or
  triggers a live production action.

- *Reconciling with prepared-not-executed:* the invariant governs **production console writes**, which
  remain human-only. A Jira comment is a low-blast-radius, reversible, clearly-labeled annotation —
  an allowed write, gated by confirmation. The design draws this line explicitly so the two are not
  in tension.
- *Unattended posting is deferred:* confirm-before-post is the rule for this change. An auto-post mode
  (no per-card confirmation) belongs to the future automation work, not here; the gate that would
  replace human confirmation is itself an unresolved design question for that later effort.

### 5. Safety as a single source of truth, bound by every skill

The cross-cutting invariants — prepared-not-executed, dry-run-first, confirm-before-any-Jira-write,
blast-radius tracing before claiming scope, suspended-capabilities → stop/route-to-T&S, and correct
rosie↔kickstarter ID resolution — live in one authoritative section of `CLAUDE.md`. Each skill cites
it rather than restating it. Remediation-specific safety (ID conventions, the blast-radius lesson)
stays in `remediation-catalog.md`, which the safety section links to.

- *Why one source:* prevents drift between skills and keeps the "what keeps automation safe" answer in
  one reviewable place.
- *rosie branch:* record each task's actual branch in the catalog (PLOT tasks on
  `demo/stripe-sync-plan-apply`; `reactivate` + general payments on `main`) rather than assuming
  `main`, and keep the instruction to confirm the path at runtime if a `require_relative` is missing.
  See Decision 8 for the prototype-branch workflow.

### 6. Remediation spans two support-task systems; route by system, not by guess

Remediations run against **two** support-task libraries, each in its own production console:

| System | Library | Console | Operates on | Example tasks |
|---|---|---|---|---|
| rosie | `rosie/lib/support_tasks/` (`main`) | `cd rosie && ksr console production` | rosie `Pledge` ids, increments, Stripe | `SyncIncrementalPledgeToStripe`, `RefundPaymentIncrementFromKsr` |
| kickstarter | `kickstarter/lib/support_tasks/` | kickstarter Rails console (production) | kickstarter `Backing`/`Pledge`/project/user ids | `ResyncBackingWithPledge`, `FixOrphanedBacking`, `DropErroredPledges`, `SyncRefundCheckout` |

The `remediation-catalog` is the routing authority: **every** catalog entry and every
symptom→remediation row is tagged with its `system` (rosie | kickstarter) and the console command to
open. The skill picks the console from the tag — it never infers which system a task belongs to.

- *Why:* the catalog today documents only rosie tasks and even mis-files `ResyncBackingWithPledge`
  (actually a kickstarter task) as rosie. Tagging by system removes the ambiguity and makes the
  console choice explicit and auditable, which matters because both consoles are production.
- *ID model, now consistent with the systems:* kickstarter console operates on the kickstarter ids in
  admin URLs (`Backing.find(<id>)`); rosie console operates on rosie `Pledge` ids resolved via
  `Pledge.find_by(client_id: 1, foreign_key: <kickstarter backing id>)`. The existing ID-conventions
  section becomes "which id in which console," not a special case.
- *Catalog coverage:* kickstarter's library (~37 tasks) covers CHECK failure classes the catalog has no
  entries for yet (`drop_errored_pledges`, `fix_orphaned_backing`, `sync_refund_checkout`). At
  implementation time the catalog is regenerated against the actual `lib/support_tasks/` listings of
  *both* repos rather than carried over rosie-only.
- *Alternative considered:* keep remediation rosie-only and treat kickstarter as id-resolution only.
  Rejected — it's already false (CHECK-277 writes in both consoles) and would route real fixes to the
  wrong system.

### 7. Looker is the default read/diagnosis layer; consoles are for writes only

Reading across systems — diagnosing a failure class, resolving rosie↔kickstarter records, and
confirming affected scope — goes through **Looker (read-only) by default**. A production console is
opened only when a state-changing remediation is actually run, and that remains human-only and
dry-run-first.

This reshapes the remediation flow:

```
classify ──▶ diagnose in Looker ──▶ confirm scope in Looker ──▶ prepare snippet ──▶ [human] console write
            (rosie_payment_increments,   (errored_pledges_over_time,    (system-tagged,
             pledges, payment_intents)     cs.model cohort/counts)        dry-run first)
```

- *Why:* most of what the catalog does in a production console today is **reading** (pledge state,
  increment state, Stripe PI status, "how many backers affected"). Moving reads to Looker removes the
  majority of production-console exposure and directly satisfies `triage-safety`'s "confirm affected
  count before claiming scope" requirement with a read-only source.
- *Cross-cutting:* the same layer enriches `priority-audit` (real affected-user counts, not just CS
  notes) and `pattern-detection` (Zendesk↔backing/project cohort overlap via `cs.model`).
- *Critical caveat — freshness:* Looker is a synced warehouse (a `fivetran` model is present) and
  lags production. Use it for triage, scope, cohort analysis, and candidate diagnosis — **not** for
  the authoritative just-before-write check. The final "is this Stripe PI actually `succeeded` right
  now?" confirmation stays in-console/Stripe, immediately before the dry-run.
- *Alternative considered:* keep all reads in the production console (status quo). Rejected — it puts
  an operator in a production console for routine triage, which is exactly the exposure we want to
  remove.
- *Graceful degradation (Looker is not a hard dependency):* if the Looker connector can't be reached,
  the skills proceed without it — falling back to ticket/CS context and human-run console reads, and
  explicitly flagging that Looker scope-confirmation was skipped (so scale/scope claims are caveated).
  They never fabricate the figures Looker would have given. This degrades diagnosis breadth, not write
  safety — the authoritative just-before-write check is in-console regardless. (Observed live: the
  connector required interactive auth and was unavailable during a demo run.)

### 8. Remediations are backed by prototype working branches, linked from the Jira comment

A prepared remediation is not only an inline console snippet. When a fix would **modify an existing
support task that is on `main`**, or when it **would make sense as a new support-task file**,
`check-remediation` opens a **working branch** in the relevant repo (rosie or kickstarter, per the
system tag) holding the prototype implementation, so the team can examine and test it with one-off
remediations. The `Experimental: Suggested Remediation.` Jira comment then carries **both**:

- **link(s)** to the prototype branch/code, and
- the **actual code blocks** to run — which may include defining a new class inline for a single
  console session (the `require_relative`/paste-into-console pattern the existing
  `demo/stripe-sync-plan-apply` branch already demonstrates).

- *Why:* it makes a remediation reviewable as real code (diff-able, testable) rather than a snippet in
  a comment, and gives a clear path from "one-off console fix" to "promote the task to `main`."
- *When a working branch is NOT needed:* the fix uses an existing task as-is (link to it at its current
  branch/path; inline the call). A branch is opened only for a modification-to-`main` or a new task.
- *Git-write boundary (safety):* creating a local working branch and committing prototype code is an
  allowed, reversible, clearly-scoped write. **Pushing** that branch to origin (needed to produce a
  shareable link) is outward-facing and is **confirmed before pushing**, consistent with
  confirm-before-write. The prototype is for examination/testing; production runs remain human-only and
  dry-run-first.
- *Branch reality, recorded not hidden:* the catalog records each task's actual branch. rosie's PLOT
  tasks are on `demo/stripe-sync-plan-apply` today; that branch IS the prototype branch for them. The
  catalog instructs confirming the branch/path at runtime before relying on a `require_relative`.

### 9. The living `main` repos are the source of truth for available tasks; the catalog is examples

The authoritative answer to "which support tasks exist" is the **living `main` of the kickstarter and
rosie GitHub repos** — verified at runtime (`gh`/`git ls-tree origin/main -- lib/support_tasks/`), not a
static list. The `remediation-catalog`'s task tables are **curated examples**: the common remediations
and the symptom→task mapping, useful for prompting, but neither exhaustive nor guaranteed current.

- *Why:* a hardcoded inventory goes stale the moment a task is added, renamed, or promoted from a
  branch. Reading the live repo keeps the toolkit correct as the codebases evolve, and uses the
  examples only to steer — not to constrain.
- *Interaction with Decision 8:* "available" means on `main`. A task not on `main` (the PLOT tasks
  today) is a prototype on a working branch by definition — exactly the case Decision 8 handles.
- *What the catalog still owns:* the symptom→remediation **mapping**, the ID/console conventions, the
  Looker diagnosis paths, the pre-flight, and the blast-radius lesson — judgment and conventions, not
  the inventory.
- *Alternative considered:* treat the catalog as the authoritative task list and regenerate it each
  run. Rejected — it duplicates what the repo already is and drifts between regenerations.

## Risks / Trade-offs

- **Jira comment spam or wrong snippet posted** → confirm-before-post by default; mandatory
  `Experimental:` prefix; idempotency check to avoid duplicates; only prepared (dry-run-verified)
  snippets are eligible.
- **Space parameterization leaks CHECK-only logic to spaces without rosie/Guru** → the registry's
  `remediation` flag gates `check-remediation`; on a space without a catalog it declines and explains,
  rather than improvising a fix.
- **CHKT fallback masks a genuine empty queue** → fallback is narrow (primary empty + result
  expected) and always announced; slated for removal once CHKT is fully retired.
- **Safety guidance drifts between `CLAUDE.md` and skills** → single source of truth + skills cite,
  not copy.
- **rosie `main` assumption goes stale if service objects move/rename** → catalog instructs runtime
  confirmation of the path/branch before relying on a `require_relative`.
- **Wrong console / wrong-system routing** (e.g. running a kickstarter task in the rosie console, or
  passing a kickstarter id to a rosie task) → every catalog entry is `system`-tagged with its console;
  the skill opens the console from the tag and labels which id is which; both being production makes
  this the highest-blast-radius mistake, so it is dry-run-gated and human-run like any other.
- **Stale Looker data drives a wrong write** (warehouse lags production; a pledge "looks errored" in
  Looker but was already fixed) → Looker is for triage/scope/diagnosis only; the authoritative
  just-before-write state check happens in-console/Stripe immediately before the dry-run, never from
  Looker.
- **Prototype working branches proliferate or leak into production** → branches are clearly named as
  experimental prototypes, pushing to origin is confirm-gated, and they hold prototype code for review
  — not a deployment path. Promotion to `main` is normal PR review, out of this toolkit's scope.
- **Role routing duplicated between `CLAUDE.md` and skill descriptions drifts** → skill `description`s
  are authoritative for triggering; the `CLAUDE.md` map is explicitly "human reference only."

## Migration Plan

Greenfield repo, single current operator (the author) — low migration risk.

1. Add root `CLAUDE.md`: role identity, TTR/priority definitions, safety invariants, human-readable
   routing map.
2. Add `references/spaces.md` with the space registry (CHECK row + the column schema above).
3. Port the three skills to `.claude/skills/<name>/` (`.claude/` only, not mirrored) with `references/`
   bundled; reconcile all JQL to the registry + CHECK/CHKT strategy.
4. Update `remediation-catalog.md`: rosie branch → `main`; system-tag every entry (rosie | kickstarter)
   with its console; regenerate against both repos' `lib/support_tasks/`; wire the Jira comment
   write-back rules.
5. Verify each skill triggers and resolves the CHECK row end to end (Looker-first read paths).
6. Rewrite `README.md` for multi-machine onboarding (prerequisites, MCP connectors to authenticate,
   auto-loading skills, registering a space in `references/spaces.md`, safety posture, first-run
   prompts). Written last so it documents what actually shipped, not what was planned.
   - **Audience: internal Kickstarter teammates** — assume KSR SSO and standard access; no external
     access-request walkthrough needed (a one-line "request access if a connector is missing" is enough).
   - **Prerequisite: clone all four repos** (kickstarter, rosie, rosie-api, looker) as standard setup,
     not conditional on whether the user expects to run remediations.

**Rollback:** delete the added files / `git revert`; nothing external is mutated by the port itself.
**Behavioral migration:** the only person using the old `SD`-board skills is the author; switch to the
CHECK space registry.

## Resolved Questions

- **Multi-tool mirroring** → `.claude/` only; triage skills are not mirrored into `.gemini/`/`.github/`.
  (Decision 1.)
- **Registry home** → standalone `references/spaces.md`, not inline in `CLAUDE.md`. (Decision 2.)
- **Priority guide for non-CHECK spaces** → not required; the `priority_guide` registry column is
  optional and `priority-audit` falls back to its built-in framework when blank. (Decision 2.)
- **Unattended posting** → deferred to the future automation work; confirm-before-post stands for this
  change. The confirmation-replacement gate is an open question for that later effort, not this one.
  (Decision 4.)

## Open Questions

- None blocking. The only deferred item is the unattended-posting gate, which belongs to the
  out-of-scope automation phase.

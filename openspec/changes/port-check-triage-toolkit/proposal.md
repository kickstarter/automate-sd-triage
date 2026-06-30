## Why

The CHECK support-triage toolkit currently lives as a loose package of Claude skills in the
`llm-tech-plans` repo (`claude-skills/support-dev-triage/`): a role definition, three skills, and
two reference docs. It was built and proven against real tickets (see the June 25 2026 remediation
run), but it sits outside the repo whose entire purpose is to automate this work, and it carries
placeholder assumptions from its draft origins. To grow it into actual automation — the goal of
`automate-sd-triage` — the trusted manual primitives need to live here first, with their board,
connector, and safety assumptions corrected and made explicit. This change establishes that
foundation; it does not build the automation on top of it yet.

## What Changes

- **Adopt the full triage toolkit into this repo** — the `support-dev-triage` role, all three skills
  (`priority-audit`, `pattern-detection`, `remediation-suggester`), and both reference docs
  (`investigation-ac-guide`, `remediation-catalog`) become first-class, version-controlled artifacts
  here rather than copy-paste install instructions for claude.ai projects.
- **Reconcile the board key to CHECK/CHKT.** `priority-audit` and `pattern-detection` currently query
  the placeholder `project = SD`; the real board — and the one `remediation-suggester` already uses —
  is `project = CHKT` with `CHECK-*` issue keys (`labels = support-dev OR parent = CHECK-105`). All
  three skills are unified onto the actual board. Note that the `CHKT` key is legacy--we migrated the whole space to `CHECK` shortly after creating it. Eventually we should be able to remove `CHKT` from queries, but it's worth adding when we unexpectedly see empty results.
- **Make this easy to share with other spaces than CHECK.** Offer an option to pass a different space via a convenient interface, so that as I iterate on this I can bring others along to try it out.
- **Onboard other users via the repo README.** Replace the stub `README.md` with setup instructions for another person to run the toolkit on their own machine: prerequisites (Claude Code; the kickstarter/rosie/rosie-api/looker repos cloned for remediation work; CHECK Jira access), which MCP connectors to connect and authenticate (Atlassian, Guru, Looker), how the skills load automatically once the repo is open in Claude Code (no copy step — `.claude/` ships with the repo), how to register their own space in `references/spaces.md`, the human-in-the-loop safety posture, and first prompts to validate the install.
- **Promote the safety posture from role "tone" to a first-class, testable requirement.** Remediations
  are *prepared, never executed*; dry-run before live; trace blast radius before claiming
  project-wide scope; suspended Stripe capabilities → stop and route to T&S; resolve rosie vs
  kickstarter IDs correctly. These become explicit guarantees, not prose guidance.
- **Give the role a real home.** `ROLE.md` (assistant identity, TTR/priority definitions, skill-
  routing triggers, behavior guidelines) is adapted into repo-native context/config rather than a
  prompt pasted into a claude.ai project.
- **Carry over the connector contract.** Document the required MCP servers per skill — Atlassian
  (Jira + Confluence), Guru, and Looker (read-only, cross-system data) — so the repo states its
  runtime dependencies explicitly.
- **Lay the groundwork for automation without building it.** Structure the skills as the reusable,
  trusted primitives that a future scheduled "due-today" routine can wrap. The routine itself is a
  **non-goal** for this change — per the toolkit's own guidance, automation is built on top only once
  the skills are trusted in this repo.

## Capabilities

### New Capabilities
- `priority-audit`: Fetch a scoped batch of CHECK tickets and assess whether each ticket's priority
  is engineering-realistic, cross-referencing the Confluence priority guide and factors like scale,
  severity, workaround availability, and resource constraints; output a structured re-triage report.
- `pattern-detection`: Analyze a set of CHECK tickets for recurring symptoms, cohort overlap, timing
  clusters, and shared components, then draft a well-scoped root-cause **investigation** ticket
  (observed symptoms, open questions, acceptance criteria) without prescribing the fix.
- `check-remediation`: Take a due-date slice of the CHECK backlog, classify each ticket by failure
  mode, and produce concrete, ID-correct manual remediation steps mapped to rosie support-task
  service objects and Guru runbooks — emitting prepared console snippets for a human to run. Post these console snippets as comments to the card(s) they pertain to, with a line at the top of the comment reading "Experimental: Suggested Remediation."
- `triage-safety`: Cross-cutting safety guarantees governing the above — prepared-not-executed,
  dry-run-first, blast-radius tracing, T&S routing for suspended capabilities, and correct
  rosie/kickstarter ID resolution. The invariant that keeps "automation" safe.

### Modified Capabilities
<!-- None. openspec/specs/ is empty; this is the repo's first set of capabilities. -->

## Impact

- **New repo artifacts**: triage skills under `.claude/skills/` (mirroring the existing
  `openspec-*` skill layout), their `references/` docs, repo-native role/context (e.g. a
  `CLAUDE.md` or agent definition), and a rewritten `README.md` with multi-machine setup
  instructions.
- **Runtime dependencies**: Atlassian MCP (Jira + Confluence) for all skills; Guru MCP for
  `check-remediation`; Looker MCP (read-only) for cross-system diagnosis and scope confirmation. No
  application code or services are added in this change.
- **External systems referenced (read/prepare only)**: the CHECK Jira space, unless an alternative space is specified, the Confluence priority guide, Guru runbooks, Looker (read-only, models both rosie and kickstarter data) for diagnosis and scope, and — for remediation snippets — the rosie and kickstarter production consoles and their `lib/support_tasks/` service objects (rosie on branch `main`). This change writes none of these; it produces prepared steps for a human.
- **Behavioral correction**: anyone currently invoking the `SD`-board skills must move to the
  CHECK/CHKT board. This is the only breaking change to existing behavior, and it corrects a known
  placeholder rather than altering intended function.

## ADDED Requirements

### Requirement: Remediations are prepared, never executed

The toolkit SHALL prepare verified remediation steps for a human to run and SHALL NOT execute any
state-changing production console action itself. This invariant governs production console writes
across every skill.

#### Scenario: A state-changing remediation is requested

- **WHEN** a skill produces a remediation that changes production data (e.g. flipping an increment
  to `collected`, re-driving collection, resyncing a backing)
- **THEN** it outputs the exact console snippet for a human to run and does not run it

#### Scenario: The operator asks the assistant to run the fix

- **WHEN** the operator asks the assistant to execute a prepared production remediation directly
- **THEN** the assistant declines, restates that production writes are human-only, and hands over the
  verified snippet

### Requirement: Dry-run precedes any live run

Any prepared state-changing snippet SHALL present the `dry_run: true` (or equivalent read-only
diagnostic) form before the `dry_run: false` form, with an instruction to read the dry-run report
before running live.

#### Scenario: Preparing a sync or re-drive

- **WHEN** a remediation snippet mutates state
- **THEN** the dry-run form appears first and the live form is explicitly gated behind reviewing the
  dry-run output

### Requirement: Reads go through Looker; consoles are for writes

Cross-system diagnosis and scope confirmation SHALL use Looker (read-only) by default. A production
console SHALL be opened only for the human-run write. Because Looker lags production, the
authoritative just-before-write state check SHALL be performed in-console (or against Stripe), never
from Looker.

#### Scenario: Diagnosing and confirming scope

- **WHEN** a skill needs ticket-underlying data (pledge/increment/payment-intent state, affected
  counts, cohort overlap)
- **THEN** it queries Looker rather than opening a production console

#### Scenario: Final state check before a write

- **WHEN** a prepared remediation depends on current production state (e.g. "the Stripe PI is
  `succeeded`")
- **THEN** that authoritative check is taken in-console/Stripe immediately before the dry-run, not
  from Looker

### Requirement: Confirm before any Jira write

The toolkit SHALL obtain explicit operator confirmation before writing to Jira — including changing
priorities, posting comments, creating tickets, or linking issues.

#### Scenario: A skill is ready to write to Jira

- **WHEN** a skill is about to change a priority, post a comment, create a ticket, or add a link
- **THEN** it shows the exact change and target and waits for confirmation before writing

### Requirement: Suspended Stripe capabilities halt remediation

When a backer's failure is caused by suspended Stripe capabilities, the toolkit SHALL stop, classify
it as not a Support Dev fix, and route it to Trust & Safety rather than preparing a console
remediation.

#### Scenario: Capabilities found suspended during pre-flight

- **WHEN** pre-flight shows the project's Stripe Connect capabilities are suspended
- **THEN** the skill stops, states this is a T&S matter, and prepares no state change

### Requirement: Correct rosie / kickstarter ID resolution

The toolkit SHALL treat ids in `kickstarter.com` admin URLs as kickstarter ids, resolve a rosie
pledge via `Pledge.find_by(client_id: 1, foreign_key: <kickstarter backing id>)`, and label which id
is which in every snippet. It SHALL NOT pass a kickstarter id where a rosie primary key is expected.

#### Scenario: A snippet needs a rosie pledge id from an admin URL

- **WHEN** a remediation needs a rosie pledge but the operator has a kickstarter admin-URL backing id
- **THEN** the snippet resolves the rosie pledge through the `foreign_key` bridge and labels each id
  by system

### Requirement: Trace blast radius before claiming scope

The toolkit SHALL trace the actual write path (form field → controller → service) and confirm the
real scope before asserting that an admin action or remediation changed something project-wide,
rather than speculating from a label or name.

#### Scenario: Asked whether an admin action changed something project-wide

- **WHEN** the operator asks whether an admin action had project-wide effect
- **THEN** the assistant traces the relevant code path and reports the confirmed scope of the writes
  before answering

### Requirement: State confidence and never fabricate

The toolkit SHALL state its confidence and flag missing context. When information is insufficient to
recommend safely, it SHALL say so and lead with diagnosis rather than inventing a recommendation or
a fix.

#### Scenario: A ticket lacks the context needed to act

- **WHEN** a ticket is missing reproduction steps, ids, user counts, or links needed to assess
- **THEN** the assistant flags exactly what is missing and does not fabricate a recommendation

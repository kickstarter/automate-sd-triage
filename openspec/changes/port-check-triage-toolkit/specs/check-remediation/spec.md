## ADDED Requirements

### Requirement: Resolve the due date explicitly

`check-remediation` SHALL operate on a due date. When the operator gives a date (absolute,
"today", "this week") it SHALL use it; when none is given it SHALL default to today's UTC date,
compute it explicitly, and state the resolved date so the operator can correct it.

#### Scenario: No date given

- **WHEN** the operator asks what can be remediated without a date
- **THEN** the skill resolves today's UTC date, states it, and proceeds

#### Scenario: Explicit date or range

- **WHEN** the operator says "due June 25" or "this week"
- **THEN** the skill uses that single date or date range

### Requirement: Fetch the due-date slice from the resolved space

`check-remediation` SHALL query the resolved space registry row (default CHECK) for open tickets at
the resolved due date, leading with the `primary_key` and `scope_filter` and applying the same
CHECK/CHKT legacy fallback (announced when used) as the other skills.

#### Scenario: Due-today slice on CHECK

- **WHEN** the resolved date is today and the space is CHECK
- **THEN** the skill queries open CHECK tickets due that date using the registry row

### Requirement: Only operate on spaces flagged for remediation

`check-remediation` SHALL run only for spaces whose registry row marks `remediation` as applicable.
For a space without a remediation mapping it SHALL decline and explain, rather than improvising a fix.

#### Scenario: Remediation requested on an unsupported space

- **WHEN** the operator asks to remediate a space whose registry row has `remediation` disabled
- **THEN** the skill declines, explains there is no remediation catalog for that space, and stops

### Requirement: Classify each ticket by failure mode

For each ticket `check-remediation` SHALL extract the relevant ids/links and map the symptom to a
failure class using the remediation catalog's symptom→remediation map (e.g. PLOT state drift, PLOT
collection never ran, suspended capabilities, PM order issue, backing↔pledge drift, code-fix-not-data).
When the class is ambiguous it SHALL say so and lead with a diagnostic dry-run rather than a state
change.

#### Scenario: Clear failure class

- **WHEN** a ticket shows an increment `errored` while Stripe shows the charge succeeded
- **THEN** the skill classifies it as state drift and proposes the sync path

#### Scenario: Ambiguous failure class

- **WHEN** the symptom does not map cleanly to a known class
- **THEN** the skill leads with a diagnostic dry-run and withholds a state-change recommendation

### Requirement: Diagnose in Looker, confirm authoritative state in-console

`check-remediation` SHALL perform diagnosis and scope reads via Looker (read-only). Any pre-write
state assertion the remediation depends on SHALL be confirmed in-console/Stripe immediately before
the dry-run, never from Looker, consistent with `triage-safety`.

#### Scenario: Diagnosing PLOT state

- **WHEN** the skill needs pledge/increment/payment-intent state to classify and scope
- **THEN** it reads that from Looker rather than a production console

#### Scenario: Confirming a sync precondition

- **WHEN** a sync depends on the Stripe PI being `succeeded`
- **THEN** the prepared steps confirm the PI status in-console/Stripe right before the dry-run

### Requirement: Route each remediation to the correct system and console

`check-remediation` SHALL determine, from the remediation catalog's `system` tag, whether a task runs
in the rosie or the kickstarter console, and SHALL open the console named by the tag. It SHALL NOT
infer the system or pass an id of the wrong system to a task.

#### Scenario: A kickstarter-system task

- **WHEN** the chosen remediation is a kickstarter support task (e.g. `ResyncBackingWithPledge`)
- **THEN** the prepared steps open the kickstarter console and use kickstarter ids

#### Scenario: A rosie-system task

- **WHEN** the chosen remediation is a rosie support task (e.g. `SyncIncrementalPledgeToStripe`)
- **THEN** the prepared steps open the rosie console and use the rosie pledge id

### Requirement: Produce a per-ticket remediation with citation

For each ticket `check-remediation` SHALL produce: a classification with confidence; the standard
pre-flight checks; remediation steps with exact console snippets (dry-run before live, ids resolved
and labeled); a verify-after check; and a citation to the backing Guru card or support-task file.

#### Scenario: A remediable ticket

- **WHEN** a ticket maps to a known remediation
- **THEN** the output includes classification, confidence, pre-flight, dry-run-first snippet,
  verify-after, and a citation

### Requirement: Decline honestly when no console remediation exists

When a ticket is a code fix, requires a creator action, or is a T&S matter, `check-remediation` SHALL
state plainly that there is no console remediation and name who it routes to, rather than inventing a
fix.

#### Scenario: Email-template rendering bug

- **WHEN** a ticket is a code/template rendering bug
- **THEN** the skill states there is no console remediation and routes it to engineering

### Requirement: Post prepared remediations to Jira as labeled, confirmed, idempotent comments

`check-remediation` SHALL, when it posts a prepared remediation to a card, post it as a Jira comment
whose first line is exactly `Experimental: Suggested Remediation.`, only after operator confirmation,
with only prepared (dry-run-first, id-resolved) snippets, and idempotently — checking for an existing
such comment and updating or skipping rather than stacking duplicates.

#### Scenario: Posting a suggestion

- **WHEN** the operator approves posting a prepared remediation to a card
- **THEN** the skill posts a comment beginning `Experimental: Suggested Remediation.` containing the
  prepared snippet

#### Scenario: Re-run on an already-commented card

- **WHEN** the card already has an `Experimental: Suggested Remediation.` comment
- **THEN** the skill updates or skips rather than posting a duplicate

#### Scenario: No confirmation given

- **WHEN** the operator has not confirmed posting
- **THEN** the skill prepares the comment but does not post it

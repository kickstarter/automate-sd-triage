## ADDED Requirements

### Requirement: Resolve audit scope before querying

`priority-audit` SHALL determine which tickets to fetch before querying. When the operator has
already specified a scope it SHALL proceed; otherwise it SHALL ask one focused scoping question and
default to all open tickets in the resolved space if none is given.

#### Scenario: Operator gives an explicit scope

- **WHEN** the operator says "pull all open P1s and P2s"
- **THEN** the skill proceeds without further questions using that scope

#### Scenario: No scope specified

- **WHEN** the operator asks to audit the queue without naming a scope
- **THEN** the skill asks one focused scoping question and, absent an answer, defaults to all open
  tickets in the resolved space

### Requirement: Query the resolved space with the CHECK/CHKT strategy

`priority-audit` SHALL build its JQL from the space registry row (default CHECK), leading with the
`primary_key` and the row's `scope_filter`. On an unexpected-empty result it SHALL retry once
widening to the row's `legacy_keys` (e.g. `project in (CHECK, CHKT)`) and SHALL announce when the
fallback was used.

#### Scenario: Default space audit

- **WHEN** no alternate space is named
- **THEN** the skill queries the CHECK row's `primary_key` and `scope_filter`

#### Scenario: Primary query unexpectedly returns nothing

- **WHEN** the primary-key query returns zero tickets but results were expected
- **THEN** the skill retries including the registry `legacy_keys` and tells the operator the fallback
  fired

#### Scenario: Alternate space requested

- **WHEN** the operator says "audit the FOO space"
- **THEN** the skill resolves the FOO registry row and uses its keys, scope filter, and priority guide

### Requirement: Use the space's priority guide, with a fallback framework

`priority-audit` SHALL fetch and apply the space's Confluence priority guide when the registry row
provides one. When the row's `priority_guide` is blank or the guide cannot be retrieved, it SHALL use
the built-in fallback priority framework and note that the internal guide was not used.

#### Scenario: Priority guide available

- **WHEN** the registry row has a `priority_guide` URL
- **THEN** the skill reads it and assesses against its definitions

#### Scenario: No priority guide

- **WHEN** the row has no `priority_guide` or it cannot be retrieved
- **THEN** the skill applies the built-in fallback framework and says so in the report

### Requirement: Assess each ticket against engineering factors

`priority-audit` SHALL evaluate each ticket for scale, severity, workaround availability, time
sensitivity, and context quality, and SHALL use Looker (read-only) to ground affected-user/scale
signal rather than relying only on CS notes. Operator-stated resource constraints SHALL override the
default criteria and be noted in the report.

#### Scenario: Estimating blast radius

- **WHEN** a ticket's affected-user count is unclear from its description
- **THEN** the skill queries Looker for a scale estimate and cites it

#### Scenario: Operator declares limited capacity

- **WHEN** the operator says there is no P1 capacity this week
- **THEN** the skill factors that into recommendations and records it in the report

### Requirement: Produce a structured re-triage report

`priority-audit` SHALL output a structured report that, for each ticket, gives one of: priority
confirmed, recommend upgrade, recommend downgrade, or needs-triage-info — each with reasoning.
Recommendations to change priority SHALL include the reason.

#### Scenario: Mixed audit results

- **WHEN** the assessment is complete
- **THEN** the report groups tickets and marks each with a confirm/upgrade/downgrade/needs-info verdict
  plus reasoning, and summarizes the recommended changes

### Requirement: Apply priority changes only on confirmation

`priority-audit` SHALL NOT change any ticket priority in Jira without explicit operator confirmation,
consistent with the confirm-before-Jira-write guarantee in `triage-safety`.

#### Scenario: Operator accepts a recommended change

- **WHEN** the operator approves a recommended priority change
- **THEN** the skill applies it in Jira; absent approval it leaves the ticket unchanged

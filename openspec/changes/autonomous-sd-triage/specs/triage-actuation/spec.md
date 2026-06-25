## ADDED Requirements

### Requirement: Apply triage decision to Jira

The system SHALL write the decided priority and labels (including `support-dev`) to the ticket, then move the ticket to the owning team's project. Labels and field changes SHALL be applied before the move so that a failed move leaves the ticket marked and retry-safe.

#### Scenario: Successful actuation

- **WHEN** a triage decision is finalized in live mode
- **THEN** the system sets the ticket's priority and labels (including `support-dev`), and then moves the ticket to the owning team's project

#### Scenario: Cross-project move fails

- **WHEN** the priority/labels are applied but the cross-project move fails (e.g., issue-type or field incompatibility in the target project)
- **THEN** the system leaves the ticket labeled `support-dev` (so it is skipped on re-run), records the failure in the audit trail, and surfaces it for human completion

#### Scenario: Mechanical steps deferred to Jira-native automation

- **WHEN** a step (e.g., due-date from priority) is handled by existing Jira automation
- **THEN** the system does not duplicate that step and relies on the Jira-native rule

### Requirement: Moved ticket lands in the receiving team's TODO column

After a cross-project move, the system SHALL ensure the ticket's status is the receiving team's TODO-equivalent (the start-of-board column), regardless of what status the move operation defaults to. The TODO-equivalent status SHALL be configurable per target project (Jira workflows name it differently), with a sane default.

#### Scenario: Move lands the ticket in TODO

- **WHEN** a ticket is moved into the owning team's project
- **THEN** its status is the receiving team's configured TODO-equivalent column

#### Scenario: Move defaults to a non-TODO status

- **WHEN** the move operation places the ticket in a status other than the receiving team's TODO-equivalent
- **THEN** the system explicitly transitions the ticket to the configured TODO-equivalent status

#### Scenario: Target TODO status not reachable

- **WHEN** the ticket cannot be transitioned to the configured TODO-equivalent status (e.g., no valid workflow transition)
- **THEN** the system records the failure in the audit trail and surfaces the ticket for human completion, leaving it labeled `support-dev` so it is not re-triaged

### Requirement: Supported issue types and type mapping on move

The system SHALL triage SD Bug, Task, and Query issue types. On the move it SHALL preserve Bug and Task types and SHALL convert Query to Task on the receiving board. It assumes target projects' Bug/Task types contain at least the same fields as the corresponding SD source (a superset) and SHALL NOT attempt field remapping in this pass.

#### Scenario: Bug or Task ticket moved with type preserved

- **WHEN** a Bug or Task ticket is moved to a target project
- **THEN** the system moves it keeping the same issue type, without remapping fields, relying on the move-failure safety net only if the field-superset assumption is violated

#### Scenario: Query ticket converted to Task on move

- **WHEN** a Query ticket is moved to a target project
- **THEN** the system moves it and sets its issue type to Task on the receiving board

#### Scenario: Unsupported issue type encountered

- **WHEN** an intake ticket is not a Bug, Task, or Query
- **THEN** the system does not move it autonomously and surfaces it for human handling

### Requirement: Low-confidence review label

The system SHALL apply a `needs-review` label when a triage decision is low-confidence, so such tickets are filterable for batch review.

#### Scenario: Low-confidence triage labeled

- **WHEN** a ticket is triaged with low confidence in priority and/or routing
- **THEN** the system applies the `needs-review` label in addition to posting the low-confidence rationale in the audit comment

### Requirement: Self-documenting audit comment

The system SHALL post a comment on each triaged ticket stating what was changed and why, mirroring the team's existing norm of justifying triage changes.

#### Scenario: Audit comment posted

- **WHEN** the system actuates a triage decision
- **THEN** it posts a comment naming the priority set, the team routed to, and the rationale (including any low-confidence flag)

### Requirement: Externalized identity (credential indirection)

The system SHALL read all actor identity and credentials (Jira user, API token, base URL, Claude API key) from secrets/configuration, and SHALL NOT hardcode or behaviorally assume the actor's identity.

#### Scenario: Swap to bot account

- **WHEN** the credentials are changed from a personal account to a bot account
- **THEN** the system operates as the new account with no code change

#### Scenario: No identity-dependent logic

- **WHEN** the system processes tickets
- **THEN** its behavior does not depend on who the configured actor is (e.g., it does not skip tickets reported by the actor or self-assign to the actor)

### Requirement: Safe write semantics

The system SHALL ensure that writing actions are consistent with idempotent processing, so re-runs do not produce duplicate or conflicting changes.

#### Scenario: No duplicate comments on re-run

- **WHEN** a ticket has already been actuated (carries `support-dev`)
- **THEN** a subsequent run does not re-apply changes or post a duplicate audit comment

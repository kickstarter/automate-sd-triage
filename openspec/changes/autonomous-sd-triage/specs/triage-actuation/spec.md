## ADDED Requirements

### Requirement: Apply triage decision to Jira

The system SHALL write the decided priority, labels (including `support-dev`), and team assignment/move to the ticket in Jira.

#### Scenario: Successful actuation

- **WHEN** a triage decision is finalized in live mode
- **THEN** the system sets the ticket's priority, applies the labels, and assigns/moves it to the owning team

#### Scenario: Mechanical steps deferred to Jira-native automation

- **WHEN** a step (e.g., due-date from priority) is handled by existing Jira automation
- **THEN** the system does not duplicate that step and relies on the Jira-native rule

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

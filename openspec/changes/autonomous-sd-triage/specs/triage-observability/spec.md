## ADDED Requirements

### Requirement: Shadow (dry-run) mode

The system SHALL support a mode in which it computes and logs the triage decision (including the would-be audit comment) without writing any change to Jira, controlled by configuration.

#### Scenario: Shadow run takes no Jira writes

- **WHEN** the system runs in shadow mode
- **THEN** it logs the decision it would make for each ticket and performs no writes to Jira

#### Scenario: Toggle to live

- **WHEN** a maintainer changes the mode configuration from shadow to live
- **THEN** subsequent runs actuate decisions, with no code change required

### Requirement: Per-run audit trail

The system SHALL record, for each run, the tickets considered, the decisions made (or would-be decisions in shadow mode), and the outcome of each write, retrievable by a maintainer.

#### Scenario: Reviewable run output

- **WHEN** a maintainer inspects a completed run
- **THEN** they can see which tickets were processed, the decisions and rationales, and whether each write succeeded or failed

### Requirement: Override-rate tracking

The system SHALL make it possible to measure the rate at which receiving teams subsequently change the agent's triage decisions (priority, labels, or team), as the primary health metric.

#### Scenario: Override signal captured

- **WHEN** a triaged ticket's priority or team is later changed by a human
- **THEN** that change is observable as an override for health-metric reporting

### Requirement: Periodic digest

The system SHALL produce a periodic (e.g., weekly) summary of triage activity and health for the owner and team.

#### Scenario: Weekly digest generated

- **WHEN** the digest period elapses
- **THEN** the system produces a summary of tickets triaged, routing distribution, low-confidence flags, and override rate

### Requirement: Maintainability for a future owner

The system SHALL be documented and configured such that a new owner can operate, monitor, pause, and adjust it without the original author.

#### Scenario: New owner pauses the system

- **WHEN** a new owner needs to stop autonomous actions
- **THEN** they can return the system to shadow mode or disable it via a single documented configuration change

#### Scenario: New owner adjusts policy

- **WHEN** routing or priority policy needs updating
- **THEN** the owner edits the routing table or the referenced Confluence guide, per the README, without modifying code

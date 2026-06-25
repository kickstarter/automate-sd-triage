## ADDED Requirements

### Requirement: Live operation with an available pause mode

The system SHALL actuate live by default, and SHALL support a shadow (dry-run) mode — controlled by configuration — in which it computes and logs the triage decision (including the would-be audit comment) without writing any change to Jira, usable for testing or as a pause switch.

#### Scenario: Shadow run takes no Jira writes

- **WHEN** the system runs in shadow mode
- **THEN** it logs the decision it would make for each ticket and performs no writes to Jira

#### Scenario: Pause via configuration

- **WHEN** a maintainer switches the mode configuration from live to shadow
- **THEN** subsequent runs stop writing to Jira, with no code change required

### Requirement: Per-run audit trail

The system SHALL record, for each run, the tickets considered, the decisions made (or would-be decisions in shadow mode), and the outcome of each write, retrievable by a maintainer.

#### Scenario: Reviewable run output

- **WHEN** a maintainer inspects a completed run
- **THEN** they can see which tickets were processed, the decisions and rationales, and whether each write succeeded or failed

### Requirement: Weekly override-audit job and override logfile

The system SHALL provide a separate, less-frequent (e.g., weekly) job that scans tickets the agent has triaged for subsequent human changes to priority, labels, or team, and appends each detected override to a dedicated override logfile committed in the repository. This is the primary health metric and runs independently of the main triage schedule.

#### Scenario: Override detected and logged

- **WHEN** the weekly override-audit job finds that a triaged ticket's priority, labels, or team was later changed by a human
- **THEN** it appends a record of that override to the override logfile in the repo

#### Scenario: Override-audit runs on its own schedule

- **WHEN** the override-audit job is configured
- **THEN** it runs on a less-frequent cadence than the main triage job and does not block or slow the main job

#### Scenario: Override trend is reviewable in version control

- **WHEN** a maintainer reviews the override logfile
- **THEN** the override history and trend are visible through the repository's version history without additional infrastructure

### Requirement: Batch review of low-confidence triages

The system SHALL make low-confidence triages reviewable as a batch via the `needs-review` label.

#### Scenario: Low-confidence tickets swept

- **WHEN** a maintainer queries Jira for the `needs-review` label
- **THEN** they retrieve the set of low-confidence triages for batch review

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

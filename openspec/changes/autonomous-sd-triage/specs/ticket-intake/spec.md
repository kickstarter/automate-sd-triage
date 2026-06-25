## ADDED Requirements

### Requirement: Scheduled detection of untriaged intake tickets

The system SHALL, on a recurring schedule, query the SD board for new intake tickets that have not yet been triaged, and pass each to the decisioning stage exactly once.

#### Scenario: New untriaged ticket detected

- **WHEN** the scheduled run executes and a ticket in `project = SD` exists without the `support-dev` label
- **THEN** the system selects that ticket for triage

#### Scenario: Already-triaged ticket skipped

- **WHEN** a ticket already carries the `support-dev` label (applied by the agent or a human)
- **THEN** the system excludes it from the intake set and does not re-triage it

#### Scenario: Run produces no new tickets

- **WHEN** the scheduled run finds no untriaged intake tickets
- **THEN** the system completes the run successfully without taking any triage action

### Requirement: Idempotent and re-runnable processing

The system SHALL be safe to run repeatedly and SHALL NOT double-process a ticket, even if a previous run failed partway through.

#### Scenario: Re-run after partial failure

- **WHEN** a prior run failed after labeling some tickets but before processing others
- **THEN** the next run skips the already-labeled tickets and processes only the remaining untriaged ones

### Requirement: Configurable intake scope

The system SHALL define the intake query (board, filter criteria) in version-controlled configuration so the next owner can adjust it without code changes.

#### Scenario: Owner updates intake criteria

- **WHEN** a maintainer edits the intake query configuration
- **THEN** subsequent runs use the updated criteria without any code modification

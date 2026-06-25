## ADDED Requirements

### Requirement: Read ticket and linked context

The system SHALL gather the ticket's content and its linked customer context (Zendesk via Jira links, CS notes, comments) before making a triage decision.

#### Scenario: Ticket with linked Zendesk context

- **WHEN** a ticket has a linked Zendesk thread and CS notes
- **THEN** the system incorporates that context into its decision inputs

#### Scenario: Sparse ticket

- **WHEN** a ticket lacks key context (no reproduction steps, no user count, no links)
- **THEN** the system proceeds but records the missing context as a reason for lower decision confidence

### Requirement: Priority decision per the SLA guide

The system SHALL set ticket priority according to the Quick Priority SLA reference, treating Confluence as the source of truth, and SHALL NOT rely on the CS-provided priority as authoritative.

#### Scenario: Priority determined from guide

- **WHEN** the system makes a priority decision
- **THEN** it applies the criteria from the current Confluence priority guide fetched at runtime

#### Scenario: Priority guide unavailable

- **WHEN** the system cannot fetch the priority guide
- **THEN** it does not guess a priority change and instead flags the ticket for human attention with the reason recorded

#### Scenario: Low priority confidence

- **WHEN** the available information is insufficient to confidently set priority
- **THEN** the system makes a best-effort decision and prominently flags the low confidence in its rationale for fast human correction

### Requirement: Label decision

The system SHALL determine the `support-dev` label and any additional helpful labels for the ticket.

#### Scenario: Helpful labels applied

- **WHEN** a ticket clearly concerns an identifiable area or signal
- **THEN** the system includes the `support-dev` label plus relevant additional labels in the decision

### Requirement: Team routing decision

The system SHALL determine the owning team using a deterministic routing table first, and SHALL fall back to LLM judgment over the ticket and team directory only when the table does not yield a confident match.

#### Scenario: Deterministic match

- **WHEN** the ticket matches a routing-table entry (component/keyword/signal)
- **THEN** the system routes to the mapped team without invoking LLM routing judgment

#### Scenario: Ambiguous routing falls back to judgment

- **WHEN** no routing-table entry confidently matches
- **THEN** the system reasons over the ticket content and team directory to choose the most likely owning team

#### Scenario: No clear owner

- **WHEN** neither the table nor judgment yields a confident team
- **THEN** the system applies the configured fallback (best-guess with a low-confidence flag, or the designated holding team) and never silently drops the ticket

### Requirement: Written rationale and confidence

For each ticket, the system SHALL produce a written rationale and a confidence assessment covering the priority, label, and routing decisions.

#### Scenario: Decision carries rationale

- **WHEN** the system finishes deciding a ticket
- **THEN** it emits a human-readable justification (e.g., affected scale, severity, workaround, component) and a confidence level usable by the actuation and observability stages

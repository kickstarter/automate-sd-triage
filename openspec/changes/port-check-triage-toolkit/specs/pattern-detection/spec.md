## ADDED Requirements

### Requirement: Gather the ticket set to analyze

`pattern-detection` SHALL determine which tickets to analyze from one of: an explicit list from the
operator, a cluster flagged by a prior `priority-audit`, or a fresh query against the resolved space.
When no source is given it SHALL ask which set to analyze. Fresh queries SHALL follow the same space
registry and CHECK/CHKT strategy as `priority-audit`.

#### Scenario: Explicit ticket list

- **WHEN** the operator names specific tickets (e.g. "look at CHECK-101, CHECK-115, CHECK-132")
- **THEN** the skill analyzes exactly those tickets

#### Scenario: No set provided

- **WHEN** the operator asks to find patterns without naming a set
- **THEN** the skill asks which set to analyze (recent batch, component, or label)

### Requirement: Analyze for cross-ticket patterns

`pattern-detection` SHALL examine the set for symptom similarity, user/cohort overlap, timing
clusters, and shared component/system signals. It SHALL use Looker `cs.model` (Zendesk ↔
backings/projects/users) to ground cohort overlap rather than inferring it from descriptions alone.

#### Scenario: Detecting a shared cohort

- **WHEN** several tickets may share affected backers, projects, or account attributes
- **THEN** the skill checks Looker for cohort overlap and reports what it found

### Requirement: Form and confirm hypotheses before drafting

`pattern-detection` SHALL present 1–3 candidate patterns — each with a name, the symptom ticket ids,
the observed signals, a confidence level, and what is still unknown — and SHALL get operator
confirmation before drafting any ticket.

#### Scenario: Presenting candidate patterns

- **WHEN** analysis surfaces possible patterns
- **THEN** the skill presents them with confidence and unknowns and asks the operator to confirm,
  redirect, or merge before drafting

### Requirement: Draft investigation tickets that investigate, not implement

For each confirmed pattern `pattern-detection` SHALL draft an investigation ticket containing
background, observed symptoms (as observations, not conclusions), a "what we don't know yet" section,
and acceptance criteria scoped to reaching a decision. Acceptance criteria SHALL NOT prescribe a
specific fix.

#### Scenario: Drafting the ticket

- **WHEN** a pattern is confirmed
- **THEN** the draft documents symptoms and open questions and its acceptance criteria define what
  must be learned or decided, not how to implement a fix

### Requirement: Avoid duplicate investigation tickets

Before creating an investigation ticket `pattern-detection` SHALL search the space for an existing
open investigation or root-cause ticket in the same area and SHALL recommend linking to it rather
than creating a duplicate when one exists.

#### Scenario: A matching investigation already exists

- **WHEN** an open investigation ticket already covers the same area
- **THEN** the skill recommends linking to it instead of creating a new one

### Requirement: Create and link only on confirmation

`pattern-detection` SHALL create the investigation ticket and link the symptom tickets only after
operator confirmation, consistent with `triage-safety`. Bulk comments on symptom tickets SHALL be
confirmed before posting.

#### Scenario: Operator approves creation

- **WHEN** the operator approves the draft
- **THEN** the skill creates the ticket, links the symptom tickets as related, and returns the new
  ticket URL

#### Scenario: Commenting on many symptom tickets

- **WHEN** the pattern spans 10+ symptom tickets and commenting on each is proposed
- **THEN** the skill confirms with the operator before posting comments at scale

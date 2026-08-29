# Job Postings Specification

## Purpose

Manage job postings linked to approved job descriptions, supporting finite and unlimited vacancy types, posting lifecycle (draft, open, on hold, closed), and closure rules for finite vs evergreen postings.

## Requirements

### Requirement: Posting linked to approved JD

A job posting MUST reference an approved job description. Postings MUST NOT be opened against draft or in-review JDs.

#### Scenario: Open posting

- **GIVEN** a posting in `draft` linked to an `approved` JD
- **WHEN** `POST /api/v1/postings/:id/open` is called
- **THEN** status MUST change to `open`

#### Scenario: Reject open on unapproved JD

- **GIVEN** the linked JD is not `approved`
- **WHEN** open is attempted
- **THEN** the system MUST return HTTP 400
- **AND** MUST NOT change posting status

### Requirement: Vacancy types

Postings MUST support `finite` (with `vacancy_count >= 1`) and `unlimited` vacancy types.

#### Scenario: Finite posting display

- **GIVEN** a finite posting with vacancy_count 5 and filled_count 2
- **WHEN** displayed in the UI
- **THEN** badge MUST show `2/5` filled

#### Scenario: Unlimited posting display

- **GIVEN** an unlimited posting
- **WHEN** displayed in the UI
- **THEN** badge MUST indicate unlimited capacity (e.g. `∞`)

### Requirement: Posting lifecycle

Postings MUST support statuses: `draft`, `open`, `on_hold`, `closed`.

#### Scenario: Hold posting

- **GIVEN** an `open` posting
- **WHEN** hold is applied
- **THEN** status MUST change to `on_hold`
- **AND** a lifecycle event MUST be recorded

### Requirement: Auto-close finite postings

When a finite posting's `filled_count` reaches `vacancy_count`, the system MUST automatically close the posting with reason `vacancy_filled`.

#### Scenario: Last vacancy filled

- **GIVEN** a finite posting with vacancy_count 3 and filled_count 2
- **WHEN** a third offer is accepted (incrementing filled_count to 3)
- **THEN** posting status MUST change to `closed`
- **AND** `closed_reason` MUST be `vacancy_filled`
- **AND** a lifecycle event MUST be recorded

### Requirement: Manual evergreen controls

Unlimited (evergreen) postings MUST NOT auto-close. Closure MUST require explicit admin/recruiter action.

#### Scenario: Close evergreen posting

- **GIVEN** an unlimited posting in `open` status
- **WHEN** recruiter calls close with a reason
- **THEN** status MUST change to `closed`
- **AND** `closed_reason` MUST reflect the manual reason

### Requirement: Scheduled reconciliation

A scheduled job MUST reconcile stale finite postings and record lifecycle events.

#### Scenario: Daily reconcile

- **GIVEN** Cloud Scheduler publishes to `posting-reconcile` topic daily
- **WHEN** worker processes the message
- **THEN** finite postings where `filled_count >= vacancy_count` and status is still `open` MUST be closed

# Priority Selection Specification

## Purpose

Allow hiring teams to select up to five priority candidates per finite vacancy posting, with explicit rank order 1–5. Unlimited postings MUST NOT expose priority selection.

## Requirements

### Requirement: Finite posting only

Priority selection MUST only be available when `vacancy_type = finite`.

#### Scenario: Unlimited posting hides priority

- **GIVEN** an unlimited posting
- **WHEN** posting detail renders
- **THEN** priority selection UI MUST NOT appear

### Requirement: Maximum five selections

A finite posting MUST NOT have more than five priority selections. Rank order MUST be unique values 1–5.

#### Scenario: Set five priorities

- **GIVEN** a finite posting with five linked candidates
- **WHEN** recruiter calls `PUT /api/v1/postings/:id/priority-selections` with five unique ranked candidates
- **THEN** all five `priority_selections` rows MUST be saved
- **AND** candidate stages MUST update to `priority_selected`

#### Scenario: Reject sixth selection

- **GIVEN** five priority selections already exist
- **WHEN** a sixth candidate is added
- **THEN** the system MUST return HTTP 400
- **AND** MUST NOT modify existing selections

#### Scenario: Duplicate rank rejected

- **GIVEN** a priority update with two candidates at rank 1
- **WHEN** the API is called
- **THEN** the system MUST return HTTP 400

### Requirement: Drag-and-drop ranking UI

Posting detail MUST provide drag-and-drop ranking for ranks 1–5 when posting is finite.

#### Scenario: Reorder priorities

- **GIVEN** three priority candidates ranked 1, 2, 3
- **WHEN** recruiter drags rank 3 to position 1
- **THEN** ranks MUST reorder accordingly on save

### Requirement: Floating panel integration

Priority selection MUST support the floating action panel for cross-page candidate picking.

#### Scenario: Pick from another page

- **GIVEN** recruiter is building priority list via floating panel
- **WHEN** they navigate to candidates list and add selections
- **THEN** panel MUST retain selections up to the limit of five

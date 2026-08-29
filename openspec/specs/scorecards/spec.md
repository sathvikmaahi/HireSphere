# Scorecards Specification

## Purpose

Provide a Scorecard Center restricted to interviewed candidates, with AI-assisted scorecard drafts and mandatory human review and approval before scores affect hiring decisions.

## Requirements

### Requirement: Interviewed-only gate

Scorecards MUST only be created or viewed for candidates who have at least one completed interview round for the posting.

#### Scenario: Block unscored candidate

- **GIVEN** a candidate with no completed interview rounds
- **WHEN** scorecard list or create is attempted
- **THEN** the system MUST return HTTP 403 or show empty state explaining the gate
- **AND** MUST NOT create a scorecard

#### Scenario: Allow interviewed candidate

- **GIVEN** a candidate with a completed interview round
- **WHEN** recruiter opens scorecards
- **THEN** scorecard creation MUST be available

### Requirement: Scorecard lifecycle

Scorecards MUST support statuses: `draft`, `ai_generated`, `in_review`, `approved`.

#### Scenario: Manual draft

- **GIVEN** an interviewed candidate
- **WHEN** interviewer creates a scorecard manually
- **THEN** status MUST be `draft`

### Requirement: AI-assisted scorecard generation

The system SHALL generate scorecard drafts from interview notes and JD via `POST /api/v1/scorecards/:id/ai-generate`.

#### Scenario: AI generate

- **GIVEN** a scorecard in `draft` with linked completed interview notes
- **WHEN** AI generate is triggered
- **THEN** Vertex AI MUST be called with active `scorecard_draft` prompt
- **AND** status MUST change to `ai_generated`
- **AND** `ai_run_id` MUST be recorded
- **AND** output MUST be presented for human edit, not auto-approved

### Requirement: Human approval

Scorecards MUST require approval by a user with `scorecards:approve` before status becomes `approved`.

#### Scenario: Approve scorecard

- **GIVEN** a scorecard in `in_review`
- **WHEN** hiring manager approves
- **THEN** status MUST change to `approved`
- **AND** `approved_by` and `approved_at` MUST be recorded
- **AND** candidate stage MUST advance toward offer eligibility

### Requirement: Scorecard UI

The `/scorecards` route MUST list only eligible candidates. AI drafts MUST show side-by-side comparison of AI output and editable fields.

#### Scenario: AI draft review

- **GIVEN** an AI-generated scorecard
- **WHEN** reviewer opens it
- **THEN** AI draft and editable fields MUST be visible side-by-side
- **AND** AI run metadata (model, prompt version) MUST be displayed

# Job Descriptions Specification

## Purpose

Provide a Job Description Workspace where recruiters draft job descriptions with optional AI assistance and route them through human review and approval before they can be linked to job postings.

## Requirements

### Requirement: JD lifecycle states

A job description MUST progress through states: `draft` → `in_review` → `approved` → `archived`.

#### Scenario: Submit for review

- **GIVEN** a JD in `draft` status
- **WHEN** author calls `POST /api/v1/job-descriptions/:id/submit-review`
- **THEN** status MUST change to `in_review`
- **AND** approvers MUST be notified (email stub acceptable in v1)

#### Scenario: Approve JD

- **GIVEN** a JD in `in_review` status
- **WHEN** a user with `job_descriptions:approve` calls approve
- **THEN** status MUST change to `approved`
- **AND** `approved_by` and `approved_at` MUST be recorded
- **AND** content MUST become read-only

### Requirement: Structured JD content

JD content MUST be stored as structured JSON with sections: summary, responsibilities, requirements, and nice-to-have.

#### Scenario: Create draft

- **GIVEN** a recruiter creates a new JD
- **WHEN** they save
- **THEN** a `draft` record MUST be created with version 1
- **AND** MUST be editable

### Requirement: Versioning

Approved JD edits MUST create a new draft version forked from the approved parent, preserving history.

#### Scenario: New version from approved JD

- **GIVEN** an approved JD
- **WHEN** author creates a new version
- **THEN** a new `draft` MUST be created with `parent_version_id` pointing to the approved version
- **AND** version number MUST increment

### Requirement: AI-assisted drafting

The system SHALL offer AI-assisted drafting via `POST /api/v1/job-descriptions/:id/ai-draft` using governed prompts and Vertex AI.

#### Scenario: AI draft generation

- **GIVEN** a draft JD with minimal input (title, bullets)
- **WHEN** user requests AI draft
- **THEN** the system MUST call Vertex AI with the active `jd_draft` prompt version
- **AND** MUST log the run in `ai_runs`
- **AND** MUST present generated content for human editing before submission
- **AND** MUST NOT auto-approve AI output

### Requirement: JD workspace UI

The `/job-descriptions` route MUST use browsable grid for listing and detail split for editing.

#### Scenario: List view badges

- **GIVEN** JDs in various states
- **WHEN** the list renders
- **THEN** each card MUST show status badge using design-spec semantic colors

# Responsible AI Specification

## Purpose

Govern AI usage across HireSphere through versioned prompt registry, pinned model versions on every AI run, human feedback on AI outputs, and admin controls for prompt activation and retirement.

## Requirements

### Requirement: Prompt registry

All AI features MUST resolve prompts from `prompt_registry` by name. Only one `active` version per prompt name MUST exist at a time.

#### Scenario: Activate prompt version

- **GIVEN** prompt `jd_draft` version 2 in `draft` status
- **WHEN** admin activates version 2
- **THEN** version 2 MUST become `active`
- **AND** prior active version MUST be `retired`
- **AND** subsequent AI calls MUST use version 2

### Requirement: Registered prompts

The system MUST register at minimum: `jd_draft`, `candidate_rank`, `fitment_summary`, `gap_summary`, `interview_questions`, `scorecard_draft`.

#### Scenario: Unknown prompt name

- **GIVEN** an AI endpoint requests prompt `unknown_prompt`
- **WHEN** no active version exists
- **THEN** the system MUST return HTTP 500 with clear error
- **AND** MUST NOT call Vertex AI

### Requirement: Model version pinning

Every `ai_runs` record MUST store `model_version` string identifying the Vertex AI model used.

#### Scenario: AI run logging

- **GIVEN** any AI invocation
- **WHEN** the run starts
- **THEN** `ai_runs` MUST record `model_version`, `prompt_version_id`, `input_hash`, and timestamps
- **AND** completion MUST record token usage when available

### Requirement: Output feedback

Users MUST provide feedback on AI-generated content via `POST /api/v1/ai-runs/:id/feedback`.

#### Scenario: Thumbs down with comment

- **GIVEN** a displayed AI-generated fitment summary
- **WHEN** user submits negative feedback with optional comment
- **THEN** feedback MUST be stored linked to `ai_run_id`
- **AND** MUST be visible in AI governance dashboard

### Requirement: AI governance admin UI

Admins MUST manage prompts from `/admin/ai-governance` with version list, diff viewer, and activate/retire actions.

#### Scenario: Prompt diff

- **GIVEN** two versions of `candidate_rank`
- **WHEN** admin views diff
- **THEN** template changes MUST be displayed side-by-side or inline diff

### Requirement: No auto-approval of AI output

AI-generated content MUST NOT bypass human review gates defined in JD, scorecard, and ranking features.

#### Scenario: AI JD draft

- **GIVEN** AI generates JD content
- **WHEN** generation completes
- **THEN** content MUST remain in `draft` status
- **AND** MUST require explicit human submit and approve actions

### Requirement: Safety settings

Vertex AI calls MUST use configured safety settings appropriate for HR/recruitment content.

#### Scenario: Blocked AI response

- **GIVEN** Vertex AI returns a blocked or empty response due to safety filters
- **WHEN** the run completes
- **THEN** `ai_runs.status` MUST be `failed`
- **AND** user MUST see a non-technical retry message
- **AND** failure MUST be logged in AI run logs

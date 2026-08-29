# AI Candidate Ranking Specification

## Purpose

Provide AI-assisted candidate ranking per job posting, including fitment summaries, gap summaries, and generated interview questions. All AI runs MUST be logged with model and prompt versions for governance.

## Requirements

### Requirement: Batch ranking per posting

The system SHALL rank all candidates linked to a posting via `POST /api/v1/postings/:id/rank`, processed asynchronously by the worker.

#### Scenario: Trigger ranking

- **GIVEN** a posting with linked candidates who have parsed resumes
- **WHEN** recruiter triggers rank
- **THEN** an `ai_runs` record MUST be created with status `queued`
- **AND** a message MUST be published to `ai-ranking` Pub/Sub topic
- **AND** API MUST return the `aiRunId` for polling

### Requirement: Ranking output

For each candidate, the system MUST store rank score, fitment summary, gap summary, and interview questions JSON.

#### Scenario: Ranking complete

- **GIVEN** ranking job completes successfully
- **WHEN** results are stored
- **THEN** each candidate MUST have a `ranking_results` row linked to the `ai_run_id`
- **AND** results MUST be sortable by `rank_score` descending

### Requirement: Governed prompts and models

Ranking MUST use the active `candidate_rank` prompt from `prompt_registry` and record `model_version` on the AI run.

#### Scenario: Prompt version pinned

- **GIVEN** active prompt `candidate_rank` version 3
- **WHEN** ranking executes
- **THEN** `ai_runs.prompt_version_id` MUST reference version 3
- **AND** `ai_runs.model_version` MUST record the Vertex AI model used

### Requirement: Re-rank invalidates display order

Re-ranking MUST create a new AI run while preserving prior runs in history.

#### Scenario: Re-rank posting

- **GIVEN** existing ranking results for a posting
- **WHEN** recruiter triggers re-rank
- **THEN** a new `ai_run` MUST be created
- **AND** UI MUST display results from the latest completed run by default
- **AND** prior runs MUST remain queryable in AI run logs

### Requirement: Rankings UI

Posting detail MUST include a Rankings tab sorted by score with expandable fitment, gaps, and interview questions.

#### Scenario: View ranking detail

- **GIVEN** ranking results exist
- **WHEN** recruiter expands a candidate row
- **THEN** fitment summary, gap summary, and generated questions MUST be displayed
- **AND** AI run metadata (model, prompt version) MUST be visible

# Interview Console Specification

## Purpose

Provide an Interview Console for structured interview notes across multiple rounds, with note versioning and AI-readable storage format for downstream scorecard generation.

## Requirements

### Requirement: Multiple interview rounds

Each candidate-posting pair MUST support multiple `interview_rounds` with round number and interviewer.

#### Scenario: Create round

- **GIVEN** a scheduled interview
- **WHEN** interviewer opens the console
- **THEN** a round MUST exist or be creatable with incrementing `round_number`

### Requirement: Structured notes

Interview notes MUST be stored as structured JSON (competencies, strengths, concerns, recommendation) plus markdown preview.

#### Scenario: Save structured note

- **GIVEN** an interviewer edits notes in the console
- **WHEN** they save
- **THEN** `content_structured_json` MUST persist the structured fields
- **AND** `content_markdown` MUST store a rendered preview

### Requirement: Note versioning

Edits MUST append new note versions; prior versions MUST NOT be overwritten or deleted.

#### Scenario: Edit existing note

- **GIVEN** a round with note version 1
- **WHEN** interviewer saves changes
- **THEN** version 2 MUST be created
- **AND** version 1 MUST remain readable in version history

### Requirement: AI-readable note format

Each note version MUST include `ai_readable_json` — a normalized flat structure for AI scorecard input.

#### Scenario: AI-readable export

- **GIVEN** a completed note version
- **WHEN** AI scorecard generation is requested
- **THEN** `ai_readable_json` MUST be available without re-parsing markdown

### Requirement: Round completion

Completing a round MUST update candidate stage to `interviewed` and enable scorecard creation.

#### Scenario: Complete round

- **GIVEN** required note fields are filled
- **WHEN** interviewer marks round complete
- **THEN** round status MUST be `completed`
- **AND** candidate stage MUST be `interviewed`

### Requirement: Console UI

The interview console MUST support round selector tabs, structured form, markdown preview, and version history sidebar.

#### Scenario: Version history

- **GIVEN** multiple note versions exist
- **WHEN** user opens version history
- **THEN** older versions MUST be read-only
- **AND** current version MUST be editable

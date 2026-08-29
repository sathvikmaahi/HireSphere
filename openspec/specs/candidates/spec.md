# Candidates Specification

## Purpose

Maintain a canonical candidate database with deduplication, resume version history, candidate-to-posting links, and an append-only history feed for all stage changes and notable events.

## Requirements

### Requirement: Candidate record

The system SHALL store candidates with display name, canonical email, canonical phone, and dedup key.

#### Scenario: Create candidate

- **GIVEN** a recruiter creates a candidate
- **WHEN** required fields are provided
- **THEN** a candidate record MUST be created
- **AND** dedup check MUST run before save

### Requirement: Deduplication

The system MUST detect likely duplicates by exact email, normalized phone, and optional fuzzy name match.

#### Scenario: Duplicate email detected

- **GIVEN** an existing candidate with email `jane@example.com`
- **WHEN** a new candidate is created with the same email
- **THEN** the system MUST warn the recruiter
- **AND** MUST offer merge or proceed explicitly

#### Scenario: Dedup key

- **GIVEN** email and phone are known
- **WHEN** dedup key is computed
- **THEN** it MUST be `sha256(lower(email) | normalized_phone)`

### Requirement: Resume versions

Each candidate MAY have multiple resume versions; exactly one MUST be marked `is_current` at a time.

#### Scenario: New upload becomes current

- **GIVEN** a candidate with an existing current resume version
- **WHEN** a new resume is successfully parsed
- **THEN** the new version MUST be marked `is_current`
- **AND** the prior version MUST remain in history

### Requirement: Candidate-posting links

Linking a candidate to a posting MUST create a unique `candidate_posting_links` row with source and stage.

#### Scenario: Link candidate to posting

- **GIVEN** a candidate and an open posting
- **WHEN** they are linked
- **THEN** a link MUST be created with `stage = applied` by default
- **AND** duplicate links for the same candidate+posting MUST be rejected

### Requirement: Candidate history

All material candidate events MUST append to `candidate_history` without deletion or overwrite.

#### Scenario: Stage change recorded

- **GIVEN** a candidate's stage changes on a posting
- **WHEN** the update succeeds
- **THEN** a history event MUST be appended with actor, event type, and payload

### Requirement: Candidates UI

The `/candidates` route MUST provide search, filter chips, browsable grid, and detail split with resume viewer, versions timeline, posting links, and history feed.

#### Scenario: Candidate detail

- **GIVEN** a recruiter opens a candidate detail page
- **WHEN** the page renders
- **THEN** it MUST show current resume, version history, linked postings, and chronological history

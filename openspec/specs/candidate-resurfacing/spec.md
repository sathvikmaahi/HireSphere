# Candidate Resurfacing Specification

## Purpose

Surface existing candidates who may fit a new job posting, including a priority lane for candidates who were previously selected but not offered a role.

## Requirements

### Requirement: Resurface suggestions endpoint

The system SHALL provide `GET /api/v1/postings/:id/resurface-candidates` returning suggested existing candidates.

#### Scenario: Skill overlap match

- **GIVEN** existing candidates with parsed resume data
- **WHEN** resurface is requested for a posting with an approved JD
- **THEN** the system MUST return candidates ranked by relevance (skills/title overlap or AI match)
- **AND** MUST exclude candidates already linked to the posting

### Requirement: Priority lane for selected-not-offered

Candidates with `stage = selected` on a prior posting and no accepted offer MUST appear in a distinct priority lane.

#### Scenario: Priority lane candidate

- **GIVEN** a candidate was shortlisted/selected on posting A but never received an accepted offer
- **WHEN** resurface runs for new posting B
- **THEN** the candidate MUST appear in the priority lane section
- **AND** MUST carry a distinct warning badge (design-spec Warning color)

### Requirement: One-click add to posting

Recruiters MUST add resurfaced candidates to a posting with one action, creating a link with `source = resurface`.

#### Scenario: Add resurfaced candidate

- **GIVEN** a suggested candidate in the resurface panel
- **WHEN** recruiter clicks add
- **THEN** a `candidate_posting_links` row MUST be created with `source = resurface`
- **AND** a candidate history event MUST be recorded

### Requirement: Resurface UI

Posting detail MUST include a "Suggested candidates" panel with standard suggestions and a separate priority lane section.

#### Scenario: Empty suggestions

- **GIVEN** no matching candidates exist
- **WHEN** the panel renders
- **THEN** an empty state MUST explain that no matches were found
- **AND** MUST NOT show errors

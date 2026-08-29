# Shortlisting Specification

## Purpose

Enable recruiters to shortlist candidates for a posting with mandatory reason capture. Shortlisting MUST update candidate stage and append history; removal MUST also be audited.

## Requirements

### Requirement: Mandatory shortlist reason

Adding a candidate to the shortlist MUST require a reason of at least 10 characters.

#### Scenario: Shortlist with reason

- **GIVEN** a candidate linked to a posting
- **WHEN** recruiter calls `POST /api/v1/postings/:id/shortlist` with `{ candidateId, reason }` where reason length >= 10
- **THEN** a `shortlist_entries` row MUST be created
- **AND** candidate stage MUST update to `shortlisted`
- **AND** history MUST record the reason and actor

#### Scenario: Reject missing reason

- **GIVEN** a shortlist request without reason or with reason shorter than 10 characters
- **WHEN** the API is called
- **THEN** the system MUST return HTTP 400
- **AND** MUST NOT create a shortlist entry

### Requirement: Shortlist removal

Recruiters MUST be able to remove a candidate from the shortlist.

#### Scenario: Remove from shortlist

- **GIVEN** a shortlisted candidate
- **WHEN** `DELETE /api/v1/postings/:id/shortlist/:candidateId` is called
- **THEN** the shortlist entry MUST be removed
- **AND** stage MUST revert appropriately
- **AND** an audit/history event MUST be recorded

### Requirement: Shortlists nav and count badge

The sidebar MUST include a Shortlists nav item with a live count badge (capped at `99+`).

#### Scenario: Badge update

- **GIVEN** shortlist count changes
- **WHEN** the sidebar renders
- **THEN** the badge MUST reflect the current count

### Requirement: Floating panel multi-select

Shortlisting MUST integrate with the floating action panel for cross-page multi-select with a shared reason.

#### Scenario: Multi-select shortlist

- **GIVEN** recruiter selects multiple candidates via floating panel
- **WHEN** they submit a shared reason
- **THEN** all selected candidates MUST be shortlisted with the same reason
- **AND** panel MUST remain until dismissed or cleared

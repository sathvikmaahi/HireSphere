# Offers and Onboarding Specification

## Purpose

Track offer lifecycle and onboarding progress for hired candidates. Capture optional Hubble Employee ID as an external reference field — not for authentication.

## Requirements

### Requirement: Offer lifecycle

Offers MUST support statuses: `draft`, `extended`, `accepted`, `declined`, `withdrawn`.

#### Scenario: Extend offer

- **GIVEN** a priority-selected or approved candidate
- **WHEN** recruiter creates and extends an offer
- **THEN** status MUST be `extended`
- **AND** `offer_date` MUST be recorded

#### Scenario: Accept offer

- **GIVEN** an extended offer
- **WHEN** candidate acceptance is recorded
- **THEN** status MUST change to `accepted`
- **AND** `accepted_at` MUST be recorded
- **AND** posting `filled_count` MUST increment by 1

### Requirement: Hubble ID capture

The system SHALL allow capture of an optional `hubble_id` string on offers labeled "Hubble Employee ID" for external HR system reference.

#### Scenario: Record Hubble ID

- **GIVEN** an accepted offer
- **WHEN** recruiter enters Hubble Employee ID via `PATCH /api/v1/offers/:id/hubble-id`
- **THEN** `hubble_id` MUST be stored on the offer record
- **AND** MUST NOT be used for login or authentication

#### Scenario: Hubble ID is not auth

- **GIVEN** any offer with `hubble_id` set
- **WHEN** a user attempts to authenticate
- **THEN** the system MUST NOT accept `hubble_id` as a credential
- **AND** login MUST remain username/password only

### Requirement: Onboarding checklist

Offers MUST include a configurable `onboarding_checklist_json` with step completion timestamps.

#### Scenario: Complete onboarding step

- **GIVEN** an accepted offer with onboarding checklist
- **WHEN** recruiter marks a step complete
- **THEN** step completion timestamp MUST be recorded
- **AND** `onboarding_status` MUST reflect overall progress

### Requirement: Offers UI

The `/offers` route MUST show offers by status (grid or pipeline board) with links to candidate and posting.

#### Scenario: Offer detail

- **GIVEN** an offer in `extended` status
- **WHEN** detail page opens
- **THEN** it MUST show offer status, Hubble ID field, onboarding checklist, and navigation to candidate/posting

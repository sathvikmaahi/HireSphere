# Dashboards and Audit Specification

## Purpose

Provide operational dashboards, exportable reports, immutable audit logs, AI run logs, and permission-gated export controls with PII redaction based on role.

## Requirements

### Requirement: Role-based dashboards

The system SHALL provide dashboards tailored to recruiter, hiring manager, and admin roles.

#### Scenario: Recruiter dashboard

- **GIVEN** a user with recruiter role
- **WHEN** they open the dashboard
- **THEN** it MUST show open postings count, pipeline funnel, and overdue interview tasks

#### Scenario: Hiring manager dashboard

- **GIVEN** a hiring manager
- **WHEN** they open the dashboard
- **THEN** it MUST show pending JD approvals, scorecards awaiting review, and priority selections

#### Scenario: Admin dashboard

- **GIVEN** an admin
- **WHEN** they open the dashboard
- **THEN** it MUST show user activity summary, AI usage metrics, and storage utilization

### Requirement: Reports and async export

Reports MUST be exportable via `POST /api/v1/exports` with async generation stored in GCS.

#### Scenario: Request CSV export

- **GIVEN** a user with `reports:export` permission
- **WHEN** they request a posting funnel report export
- **THEN** a `report_exports` row MUST be created with status `queued`
- **AND** worker MUST generate CSV to GCS
- **AND** user MUST receive a signed download URL when complete

### Requirement: Audit log

All material user actions MUST append to immutable `audit_events` with actor, action, resource, before/after payload, IP, and user agent.

#### Scenario: Search audit log

- **GIVEN** an admin on `/admin/audit`
- **WHEN** they filter by actor and date range
- **THEN** matching audit events MUST be returned paginated
- **AND** events MUST NOT be editable or deletable via API

### Requirement: AI run logs

All AI invocations MUST be viewable at `/admin/ai-runs` with status, model version, prompt version, token usage, and linked entity.

#### Scenario: View failed AI run

- **GIVEN** an AI run with status `failed`
- **WHEN** admin opens the run detail
- **THEN** error context and input hash MUST be visible
- **AND** linked feature entity (JD, ranking, scorecard) MUST be navigable

### Requirement: Export PII redaction

Exports MUST redact PII fields based on the requester's role.

#### Scenario: Viewer role export

- **GIVEN** a user with `viewer` role and export permission
- **WHEN** they export candidate data
- **THEN** phone numbers and full email addresses MUST be redacted or omitted per policy
- **AND** recruiters with full `candidates:view` MUST receive unredacted exports

### Requirement: Export retention

Export files in GCS MUST expire after 90 days per bucket lifecycle policy.

#### Scenario: Expired export

- **GIVEN** an export older than 90 days
- **WHEN** lifecycle job runs
- **THEN** the GCS object MUST be deleted
- **AND** `report_exports` metadata MAY remain for audit

# Interview Scheduling Specification

## Purpose

Provide a task-based interview scheduling workflow where recruiters assign scheduling tasks, set due dates, and mark interviews as scheduled with datetime and interviewer assignment.

## Requirements

### Requirement: Interview tasks

The system SHALL create `interview_tasks` linked to a candidate and posting with assignee, due date, and status.

#### Scenario: Create scheduling task

- **GIVEN** a shortlisted candidate
- **WHEN** recruiter creates an interview task assigned to a coordinator
- **THEN** task status MUST be `pending`
- **AND** task MUST appear in the assignee's queue

### Requirement: Schedule interview

Assignees MUST schedule interviews via `POST /api/v1/interview-tasks/:id/schedule` with datetime and interviewer.

#### Scenario: Mark scheduled

- **GIVEN** a pending interview task
- **WHEN** schedule is submitted with valid datetime
- **THEN** task status MUST change to `scheduled`
- **AND** `scheduled_at` MUST be recorded
- **AND** candidate stage MUST update to `interview_scheduled`
- **AND** assignee MUST receive notification (email stub acceptable in v1)

### Requirement: Task queue UI

The `/interviews` route MUST show a task queue filterable by mine, overdue, and unscheduled.

#### Scenario: Overdue task highlight

- **GIVEN** a task past its due date and still `pending`
- **WHEN** displayed in the queue
- **THEN** it MUST be visually distinguished (warning status styling)

### Requirement: Task completion and cancellation

Tasks MUST support completion and cancellation with audit trail.

#### Scenario: Cancel task

- **GIVEN** a scheduled task
- **WHEN** it is cancelled with reason
- **THEN** status MUST change to `cancelled`
- **AND** history MUST record the cancellation

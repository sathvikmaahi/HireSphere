## Purpose

The task and notification substrate: assignable work items with ownership and aging, and delivery of workflow alerts to internal recipients through in-app and email channels. Candidate-facing communication is explicitly outside this capability.

## ADDED Requirements

### Requirement: Task model

The system SHALL support tasks carrying a type, subject reference, assignee, owner, creation timestamp, due or target timestamp, status, and completion attribution.

#### Scenario: Task assignment

- **WHEN** a workflow event creates a task for a user
- **THEN** the task appears in that user's task list with its type, subject, and target date

#### Scenario: Task completion

- **WHEN** an assignee completes a task
- **THEN** the task records who completed it and when, and no longer appears as outstanding

#### Scenario: Reassignment

- **WHEN** a task is reassigned by an authorized user
- **THEN** the task moves to the new assignee and the reassignment is audited

### Requirement: Task visibility follows permissions

A user SHALL see only tasks they are permitted to see, and a task SHALL NOT disclose subject detail the recipient is not authorized to read.

#### Scenario: Task referencing an unreadable subject

- **WHEN** a user holds a task whose subject record they lack permission to read
- **THEN** the task is presented without the restricted subject detail
- **AND** opening the subject is denied

### Requirement: Aging thresholds

Task and workflow aging thresholds SHALL be configurable without code changes.

#### Scenario: Threshold change

- **WHEN** an administrator changes an aging threshold
- **THEN** subsequent aging calculations and any aging indicators use the new value without a deployment

### Requirement: Notification delivery channels

The system SHALL deliver notifications through an in-app channel and an internal email channel.

#### Scenario: Workflow event notification

- **WHEN** a workflow event configured to notify occurs
- **THEN** the recipient receives an in-app notification
- **AND** an email is sent to the recipient's corporate address where the notification type is configured for email

#### Scenario: Marking read

- **WHEN** a user reads an in-app notification
- **THEN** it is marked read for that user only

### Requirement: Internal recipients only

Notifications SHALL be addressed only to internal users. The system SHALL NOT send email to candidates in this scope.

#### Scenario: Attempt to notify an external address

- **WHEN** a notification is requested for a recipient who is not an internal user
- **THEN** the send is refused and the refusal is recorded

### Requirement: Configurable templates

Notification templates SHALL be configurable without code changes.

#### Scenario: Template edited

- **WHEN** an administrator edits a notification template
- **THEN** subsequently sent notifications of that type use the edited template

### Requirement: Retry with backoff and failure visibility

Notification delivery SHALL retry transient failures with backoff, and delivery status SHALL be visible to authorized users.

#### Scenario: Transient email failure

- **WHEN** the email service is temporarily unavailable
- **THEN** delivery is retried with backoff and the notification remains pending rather than being silently dropped

#### Scenario: Permanent failure

- **WHEN** delivery fails permanently
- **THEN** the failure and its reason are recorded and visible to authorized users

### Requirement: Notification outages do not block work

A notification or email service outage SHALL NOT prevent users from viewing records or completing workflow actions.

#### Scenario: Email service down during a workflow action

- **WHEN** a user completes a workflow action while the email service is unavailable
- **THEN** the workflow action succeeds
- **AND** the resulting notification is queued for retry

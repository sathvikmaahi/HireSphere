## Purpose

The task and notification substrate: assignable work items with ownership and aging, and
delivery of workflow alerts to internal recipients through in-app and internal email channels.
Candidate-facing communication is explicitly outside this capability. Owned by `TS-BL-005`.

## ADDED Requirements

### Requirement: Task model

The system SHALL support tasks carrying a type, subject reference, assignee, owner, creation
timestamp, due or target timestamp, status, and completion attribution.

*Source: reference spec §8, Notification Service — tasks, reminders, workflow alerts.*

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

A user SHALL see only tasks they are permitted to see, and a task SHALL NOT disclose subject
detail the recipient is not authorized to read.

*A notification is a delivery surface, and a delivery surface that quotes a record's contents
is a read path. Without this, a task about a candidate would leak candidate detail to a
recipient with no permission on that candidate — which would make the Interviewer and
Administrator scope boundaries decorative.*

#### Scenario: Task referencing an unreadable subject

- **WHEN** a user holds a task whose subject record they lack permission to read
- **THEN** the task is presented without the restricted subject detail
- **AND** opening the subject is denied

### Requirement: Notifications carry references, not personal data

Notification and task payloads SHALL carry references to records rather than copies of personal
data, and SHALL be resolved against the recipient's permissions at read time.

*Source: `design.md` D6's reasoning applied to this surface — audit records carry references
precisely so the audit trail does not become a personal-data back door. A notification store
that copied candidate detail at send time would reopen exactly that hole, and would also serve
stale values after the record changed.*

#### Scenario: Notification about a candidate record

- **WHEN** a notification concerning a candidate record is stored
- **THEN** it holds a reference to that record rather than a copy of its personal data

#### Scenario: Referenced record changes after send

- **WHEN** a referenced record changes after the notification was sent
- **THEN** reading the notification resolves the current value, subject to the reader's
  permissions

### Requirement: Notification delivery channels

The system SHALL deliver notifications through an in-app channel and an internal email channel.

*Source: reference spec §24, Email Notification Service.*

#### Scenario: Workflow event notification

- **WHEN** a workflow event configured to notify occurs
- **THEN** the recipient receives an in-app notification
- **AND** an email is sent to the recipient's corporate address where the notification type is
  configured for email

#### Scenario: Marking read

- **WHEN** a user reads an in-app notification
- **THEN** it is marked read for that user only

### Requirement: Internal recipients only

Notifications SHALL be addressed only to internal users, refused at the sender rather than
filtered at the template. The system SHALL NOT send email to candidates in this scope.

*Source: `design.md` D16 and `C-04`, which defer candidate-facing communication pending legal
sign-off (`OD-005`). "We simply will not send to candidates" is not a control; a sender-side
guard means the deferral cannot be violated by a misconfigured template. Candidate-facing
delivery is `resurfacing-and-communications`' `TS-BL-078`, and remains gated on `OD-005`.*

#### Scenario: Attempt to notify an external address

- **WHEN** a notification is requested for a recipient who does not resolve to an internal user
  record
- **THEN** the send is refused and the refusal is recorded

#### Scenario: Template configured with an external address

- **WHEN** a notification template is configured to reach an external address
- **THEN** the sender refuses the send regardless of the template's configuration

### Requirement: Delivery payload contract

Notifications SHALL expose a stable payload carrying at minimum a severity, a title, a body,
the originating event reference, and an optional action reference, so that a rendering surface
can present a notification without knowing which feature produced it.

*This is the contract `design-system`'s toast component (`TS-BL-012`) renders against — that
item depends on this one specifically because a toast needs a real payload shape rather than a
visual mock. Severity maps onto the status-surface triplets in `reference/design-spec.md` §1.5;
this capability defines the semantics, and the design system owns their appearance.*

#### Scenario: Rendering surface consumes a notification

- **WHEN** a rendering surface receives a notification
- **THEN** it can present severity, title, body, and any action without feature-specific
  knowledge

#### Scenario: New notification type introduced

- **WHEN** a feature introduces a new notification type
- **THEN** existing rendering surfaces present it without modification

### Requirement: Aging thresholds

Task and workflow aging thresholds SHALL be configurable without code changes.

#### Scenario: Threshold change

- **WHEN** an administrator changes an aging threshold
- **THEN** subsequent aging calculations and any aging indicators use the new value without a
  deployment

### Requirement: Configurable templates

Notification templates SHALL be configurable without code changes.

#### Scenario: Template edited

- **WHEN** an administrator edits a notification template
- **THEN** subsequently sent notifications of that type use the edited template

### Requirement: Delivery dispatches through the shared async substrate

Notification delivery SHALL be dispatched as background work through the shared asynchronous
orchestration pattern, declaring its retry policy per channel like any other job type. It SHALL
NOT define its own dispatch, retry, or dead-letter mechanism. Delivery status SHALL be visible
to authorized users.

*Source: reference spec §25, which lists Notifications among the background job types carrying a
retry-with-backoff policy, in the same table as resume parsing and ranking.
`platform/async-orchestration` requires one dispatch pattern and fails the test suite on
background work started outside it; notification delivery is that capability's most obvious
consumer, so an independent retry mechanism here would be the first exception to a rule written
to have none. Both specs otherwise state retry-with-backoff, status visibility, and
permanent-failure recording separately — the same three requirements twice. See `design.md` D9.*

#### Scenario: Transient email failure

- **WHEN** the email service is temporarily unavailable
- **THEN** delivery is retried under the channel's declared retry policy and the notification
  remains pending rather than being silently dropped

#### Scenario: Permanent failure

- **WHEN** delivery fails past its declared retry policy
- **THEN** the failure and its reason are recorded and visible to authorized users

#### Scenario: In-app delivery only

- **WHEN** a notification is delivered through the in-app channel alone
- **THEN** it is written synchronously and requires no background dispatch

### Requirement: Notification outages do not block work

A notification or email service outage SHALL NOT prevent users from viewing records or
completing workflow actions.

*Source: `DEP-008`, `G-12` graceful degradation.*

#### Scenario: Email service down during a workflow action

- **WHEN** a user completes a workflow action while the email service is unavailable
- **THEN** the workflow action succeeds
- **AND** the resulting notification is queued for retry

## Purpose

The reusable pattern by which work that must not block an interactive request is dispatched,
executed, observed, and retried — established once here so that resume parsing, embedding
generation, ranking, scorecard drafting, notification delivery, and report export each inherit
it rather than inventing their own. Owned by `TS-BL-006`.

## ADDED Requirements

### Requirement: Long work does not block interactive requests

Triggering long-running work SHALL return promptly with a job identifier, and the work SHALL
execute outside the interactive request.

*Source: `DEP-007`, `NFR-002`.*

#### Scenario: Long job submitted

- **WHEN** a client triggers a long-running operation
- **THEN** the response returns a job identifier and the interactive request completes promptly

#### Scenario: Screen triggering background work

- **WHEN** a screen triggers a long-running background job
- **THEN** the screen itself still returns within the interactive performance threshold, with
  job progress reported asynchronously

### Requirement: Job status is retrievable

Job status SHALL be retrievable by job identifier, reporting at minimum whether the job is
pending, running, succeeded, or failed, and carrying a failure reason where it failed.

*Source: `NFR-002` — asynchronous work runs with progress status.*

#### Scenario: Status queried

- **WHEN** a client queries a job identifier
- **THEN** the current state is returned

#### Scenario: Failed job

- **WHEN** a job has failed
- **THEN** its status reports the failure and preserves the failure detail for investigation

### Requirement: One dispatch pattern, not one per feature

Background work SHALL be dispatched through a single shared pattern: an event published to a
topic, delivered to a dispatcher, which selects and starts the appropriate job. A feature SHALL
introduce new background work by registering against that pattern, not by building its own
dispatch mechanism.

*Source: `exploration-notes.md` D.9's `TS-BL-006`, and D09's supersession block of 2026-08-25.
The mechanism is Pub/Sub → Eventarc → dispatcher → Cloud Workflows → Cloud Run Job, consumed
from the landing zone's existing Terraform modules rather than built from primitives — the same
relationship this repository has with the shared pipeline template. This supersedes the "Cloud
Tasks or Celery" row in D09's settled-stack table, which was derived from the backend language
choice rather than decided on merits; see `design.md` D6.*

*Notification delivery is in scope as a consumer of this pattern and is **not** an exception —
see `platform/notifications`' delivery requirement, which declares its retry policy here rather
than implementing its own. It is called out by name because it is the one consumer with enough
retry, status, and failure-handling machinery of its own to look like it could stand alone; that
question was resolved deliberately (`design.md` D9) and should not be reopened without
revisiting this requirement. The in-app channel is a synchronous write and is outside this
requirement's scope, since it is not background work.*

#### Scenario: New background job type added

- **WHEN** a feature adds a new type of background work
- **THEN** it registers against the shared dispatch pattern with no new dispatch mechanism

#### Scenario: Feature builds its own dispatch

- **WHEN** a code path starts background work outside the shared pattern
- **THEN** the test suite fails

### Requirement: Independent worker scaling

Background workers SHALL be scalable independently of API services.

*Source: `DEP-006`.*

#### Scenario: Background load rises

- **WHEN** background work load rises
- **THEN** worker capacity scales without scaling the interactive API service

### Requirement: Correlation survives the boundary

The correlation identifier of the request that triggered background work SHALL be carried
through the event, the dispatcher, and the job, so that the whole chain is traceable from one
identifier.

*Source: reference spec §26 Audit Correlation; `NFR-007`. This is what makes the observability
requirement in `platform/delivery-foundation` true across an asynchronous boundary rather than
only within a request.*

#### Scenario: Tracing across the boundary

- **WHEN** an investigator holds the correlation identifier of the triggering request
- **THEN** the dispatched event, the job execution, and its logs can all be located

#### Scenario: Job produces a transition

- **WHEN** a background job performs a state transition
- **THEN** the transition record carries the correlation identifier of the triggering event

### Requirement: Retry semantics are declared per job type

Each job type SHALL declare its retry policy, and retries SHALL NOT produce duplicate effects.
Work that cannot be safely retried SHALL be declared as such rather than retried by default.

*Source: reference spec §25's per-job retry policies and `NFR-006` — failed calls preserve
failure details and support safe retry where applicable. "Where applicable" is the operative
clause: retrying a job that already had an effect is worse than not retrying it.*

#### Scenario: Transient failure

- **WHEN** a job fails transiently and its type declares retry
- **THEN** it is retried according to its declared policy

#### Scenario: Duplicate delivery

- **WHEN** the same event is delivered more than once
- **THEN** the effect occurs once

#### Scenario: Unsafe retry

- **WHEN** a job type declares its work unsafe to retry
- **THEN** a failure is recorded for human action rather than retried automatically

### Requirement: Exhausted work is visible, never silently dropped

Work that fails past its retry policy SHALL be retained with its failure reason and made
visible to authorized users, rather than discarded.

*A queue that silently drops work is indistinguishable from a queue with nothing in it — the
same class of failure as Sprint 0's green-pipeline defects, where success was reported for work
that never happened.*

#### Scenario: Retries exhausted

- **WHEN** a job exhausts its retry policy
- **THEN** it is retained with its failure reason and surfaced to authorized users

#### Scenario: Backlog growth

- **WHEN** pending work accumulates beyond the configured threshold
- **THEN** an alert is raised

### Requirement: Background actors are attributed and permission-bound

A background job SHALL execute as a named service account, and SHALL be subject to the same
permission evaluation as any other actor.

*Source: `WF-004`; `design.md` D1 — the AI Service Account is evaluated by the same rules as
every other actor, which is why the evaluator covers background actors and not only endpoints.*

#### Scenario: Job acts on a resource

- **WHEN** a background job performs an action on a resource
- **THEN** the action is attributed to the job's service account

#### Scenario: Job lacks permission

- **WHEN** a background job attempts an action its service account does not hold permission for
- **THEN** the action is denied

### Requirement: Downstream outage degrades gracefully

Unavailability of an external dependency SHALL NOT prevent users from viewing existing records
or performing actions that do not require it.

*Source: `DEP-008`, `NFR-004`, `G-12`.*

#### Scenario: External dependency unavailable

- **WHEN** an external dependency a background job needs is unavailable
- **THEN** interactive read and non-dependent workflow actions continue to function
- **AND** the affected work is queued or recorded as failed rather than blocking the user

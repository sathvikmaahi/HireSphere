## Purpose

`§25`'s existing Report Export job type, built on the platform's established dispatch chain: an export
that carries exactly what the screen that offered it carries, under permission, with a classification
label and its own audit event. Owned by `TS-BL-071`, consumed by every operational reporting surface.

## ADDED Requirements

### Requirement: Report Export is registered as the existing job type, not a new one

Report export SHALL be registered as a job type against the platform's async dispatch registry. It
SHALL NOT implement its own queue, retry loop, scheduler, or job-status store, and SHALL report status
from the substrate's job-status record.

*Source: `reference/spec.md` `§25`, whose eleven rows already contain "Report Export | User action |
Retry export build only when safe. | Export file and audit event." `platform-core`'s
`platform/async-orchestration` and its task 6.4, which exists so "a new job type is a registration
rather than an infrastructure change"; its task 5.9 sets the precedent of reading status from the
substrate rather than a second store. `§25` stays at eleven — the discipline
`decision-and-offers`' D11 applied when declining to add a twelfth row. `design.md` D9.*

#### Scenario: Export requested

- **WHEN** an authorized user requests a report export
- **THEN** a job identifier is returned promptly and the export builds outside the interactive request

#### Scenario: Status retrieved

- **WHEN** the export's status is requested by job identifier
- **THEN** it is returned from the dispatch substrate's job-status record

#### Scenario: Second mechanism sought

- **WHEN** this capability is inspected for its own queue, retry loop or job-status store
- **THEN** none exists

#### Scenario: Job-type count

- **WHEN** the registered job types are enumerated against `§25`
- **THEN** report export is the `§25` row it implements and no additional `§25` row is invented

### Requirement: An export build is retry-safe only until its artifact is published

The export job type SHALL declare its retry policy such that a build is retried only where retrying
cannot produce a second artifact under the same export identity or deliver a partially written one.
Once an artifact is published, failure SHALL be recorded for human action rather than retried.

*Source: `§25`'s "Retry export build only when safe" turned into a rule the dispatcher can act on.
`platform-core`'s task 6.8, which provides for job types "declared unsafe to retry, which record a
failure for human action instead", and its task 6.9's idempotency at the effect boundary — needed
because Pub/Sub delivery is at-least-once. `NFR-006` — failed calls preserve failure details and
support safe retry where applicable. `design.md` D9.*

#### Scenario: Failure before publication

- **WHEN** an export build fails before its artifact is published
- **THEN** it is retried under the declared policy

#### Scenario: Failure after publication

- **WHEN** an export fails after its artifact is published
- **THEN** it is not retried and the failure is recorded for human action

#### Scenario: Duplicate event delivery

- **WHEN** the same export request is delivered to the dispatcher twice
- **THEN** exactly one artifact exists for that export identity

#### Scenario: Partial artifact

- **WHEN** an export build is interrupted mid-write
- **THEN** no partially written artifact is delivered to a requester

### Requirement: An export carries the same column set as the view that offered it

An export SHALL be generated from the same reporting projection, under the same filters, as the
reporting view from which it was requested. Its column set SHALL equal that view's column set.

*Source: `design.md` D3, vector 3 — this is the vector where a screen that redacts and a file that does
not would breach `D16` without any code looking wrong.
`access-control/audit-review`'s `Export honors the same content constraint` scenario applies the same
rule to audit exports. `reporting/read-model`'s projection requirement is what makes the guarantee
cheap.*

#### Scenario: Column parity

- **WHEN** an export and its originating view are generated for one filter set
- **THEN** their column sets are equal, and a test asserts it

#### Scenario: Personal data in an export

- **WHEN** an export of a candidate-derived report is inspected
- **THEN** it contains no candidate name, contact field or resume content, exactly as the view does not

#### Scenario: Filter applied

- **WHEN** an export is requested from a filtered view
- **THEN** the exported content reflects those filters and not a wider set

### Requirement: Export requires Export permission and is denied without it

Report export SHALL require the Export action permission on the reporting surface being exported,
resolved by the permission evaluator server-side. Roles seeded without Export SHALL be denied.

*Source: `PRV-007` — "Data exports shall be permission-controlled and auditable"; `§9.3`'s Export
action flag as one of the nine; `UI-008`, which makes table export conditional on permission;
`AUTHZ-004`. `TS-BL-022` task 5.6 seeds Interviewer and Hiring Panel Member with no Export.
`design-system`'s `TS-BL-009` task 3.5 gates the table's export control on a caller-supplied decision
and performs no evaluation itself, so the decision must be made here.*

#### Scenario: Export without permission

- **WHEN** a user without Export permission requests an export
- **THEN** the request is denied

#### Scenario: Control offered

- **WHEN** a reporting view is rendered for a user holding Export
- **THEN** the table's export control is offered, the decision having been supplied by this capability
  rather than evaluated by the table

#### Scenario: Interviewer requests an export

- **WHEN** a user holding only the Interviewer role requests an export
- **THEN** the request is denied

### Requirement: Every export carries a classification label and generates an audit event

An export artifact SHALL carry a data classification label. Each export SHALL generate an audit record
naming who exported what, when, and under which filter.

*Source: `SEC-012` — "Report exports shall include data classification labels and export audit
events"; `§20.2` item 16 — "Report and audit exports" among the events the system shall audit;
`PRV-007`. `platform/audit-trail`'s reference-and-diff-only rule applies to the audit record itself, so
the record names the filter without reproducing exported content.*

#### Scenario: Label present

- **WHEN** an export artifact is produced
- **THEN** it carries a data classification label

#### Scenario: Audit event written

- **WHEN** an export completes
- **THEN** an audit record names the actor, the report, the time and the filter applied

#### Scenario: Audit record content

- **WHEN** an export's audit record is read
- **THEN** it identifies the export by reference and contains no candidate personal data

#### Scenario: Failed export

- **WHEN** an export fails
- **THEN** the attempt is audited and no artifact is delivered

### Requirement: Export size is bounded by configured limits

Export volume SHALL be bounded by a configured limit. A request exceeding it SHALL be refused with the
limit named, rather than truncated silently.

*Source: `§29` item 13 — "Export limits" among the values configurable without code changes;
`talentsphere-wave-1-foundation`'s `platform/delivery-foundation` spec, which names export limits among
the runtime-configurable set. `ERR-001`'s requirement that user-facing errors be actionable. Silent
truncation would present a partial export as complete, which is the "silent success" failure class
Sprint 0's handover identifies as the dangerous one.*

#### Scenario: Within the limit

- **WHEN** a requested export falls within the configured limit
- **THEN** it builds and is delivered complete

#### Scenario: Exceeding the limit

- **WHEN** a requested export would exceed the configured limit
- **THEN** it is refused, naming the limit and the requested size, and no partial artifact is delivered

#### Scenario: Limit changed

- **WHEN** an authorized administrator changes the export limit
- **THEN** the change is audited with a mandatory reason and applies to subsequent requests

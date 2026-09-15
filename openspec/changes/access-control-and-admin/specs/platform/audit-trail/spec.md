## Purpose

The immutable evidence trail behind every material change in TalentSphere: what an audit record
contains, what it may never contain, and why it is append-only at the database rather than by
application convention. Records must be readable by compliance reviewers without exposing candidate
personal data, which is a constraint on what they may contain rather than a filter applied when
displaying them. Owned by `TS-BL-020`.

## ADDED Requirements

### Requirement: Audit on every material write

Every material write action SHALL produce an audit record, written in the same transaction as the
change. A failure to write the audit record SHALL fail the operation.

*Source: `config.yaml`'s standing architectural rule; `SEC-010`. Inherited from
`talentsphere-wave-1-foundation`'s `platform/audit-trail`; path preserved per `design.md` D2.*

#### Scenario: Material write through the API

- **WHEN** any endpoint performs a material write
- **THEN** an audit record is written in the same transaction as the change

#### Scenario: Audit write fails

- **WHEN** the audit record for a material write cannot be written
- **THEN** the operation fails rather than silently proceeding, and the change does not take effect

#### Scenario: Read operations

- **WHEN** a user views a record without changing it
- **THEN** no audit record is required, except for the separately audited categories of export and
  elevated access

### Requirement: Audit record content

Each audit record SHALL capture the acting user or service account, the action, the target type and
identifier, previous and new values where applicable, a reason where required, a request
correlation identifier, source address where available, and an immutable timestamp.

*Source: `reference/spec.md` §12.2 `audit_logs`; §26 Audit Correlation. The field set matches the
`AuditEvent` shape `platform-core` already ships, which this substrate accepts unchanged
(`design.md` D8).*

#### Scenario: Human actor

- **WHEN** a user changes a record
- **THEN** the audit record identifies the user, the action, the target, and both previous and new
  values

#### Scenario: Service account actor

- **WHEN** a background task performs a material write
- **THEN** the audit record attributes the action to the service account and correlates it to the
  triggering event

#### Scenario: Actor without a user identity

- **WHEN** an event has no human actor, such as a failed authentication attempt by an unknown
  identity
- **THEN** the record attributes the acting subsystem as a service account actor and records the
  attempted identity as the target

### Requirement: The existing audit interface is honored without modification

The durable writer SHALL be installed behind the audit interface that already exists, and SHALL
accept the event vocabulary already emitted through it. No emitting surface SHALL require
modification, and the interface SHALL NOT be redefined.

*Source: `platform-core`'s `design.md` D8, which built the workflow engine's transaction-coupled
audit write against this promise, and its task 4.7. `identity-and-access`'s `TS-BL-017`
authentication vocabulary and the runtime-configuration change path are the events already in
flight. `design.md` D8.*

#### Scenario: Existing emitters unchanged

- **WHEN** the durable writer is deployed
- **THEN** events previously recorded to the structured log are recorded to the audit store
- **AND** no emitting surface is modified

#### Scenario: Authentication vocabulary accepted

- **WHEN** the authentication event vocabulary — login success, login failure, logout, forced
  revocation, activation-state change — is emitted
- **THEN** each event is recorded without alteration to its actor, target, or reason category

#### Scenario: Correlation identifier preserved

- **WHEN** an event carrying the request's correlation identifier is recorded
- **THEN** the stored record carries the same identifier

### Requirement: Audit records contain no candidate personal data

Audit records SHALL reference subjects by identifier and event type and SHALL NOT contain candidate
names, contact details, resume text, or other candidate personal data in their stored values.
Redaction SHALL be applied when the record is written, not when it is displayed.

*Source: `D16` — Administrators are configuration-only and the Auditor role reads this trail; if
records held personal data both boundaries would be decorative and this table would be the back
door. `design.md` D6.*

#### Scenario: Compliance reviewer reads the trail

- **WHEN** an Auditor reads audit records relating to candidate activity
- **THEN** the records identify candidates by identifier and describe the event
- **AND** no candidate personal data is present in the returned payload

#### Scenario: A write whose values are sensitive

- **WHEN** a material write changes a field holding personal data
- **THEN** the audit record captures that the field changed, with the sensitive values redacted or
  referenced rather than stored verbatim

#### Scenario: An unclassified field

- **WHEN** a value is recorded for a field not declared in the sensitivity classification
- **THEN** it is treated as sensitive and referenced rather than stored verbatim

#### Scenario: Redaction is not a display filter

- **WHEN** stored audit records are read directly from the audit store, bypassing any application
  surface
- **THEN** no candidate personal data is present in them

### Requirement: Mandatory reasons on sensitive changes

Where a reason is required by the acting workflow, the audit record SHALL carry it, and the
underlying action SHALL be rejected when the reason is absent.

*Source: `AUTHZ-007`, `ADM-007`, and `reference/spec.md` §12.2 `audit_logs.reason` — "mandatory for
sensitive changes".*

#### Scenario: Sensitive change without a reason

- **WHEN** a user performs an action designated as requiring a reason and supplies none
- **THEN** the action is rejected with a field-level validation error and no audit record is written

#### Scenario: Permission change carries its reason

- **WHEN** a permission assignment changes
- **THEN** the audit record carries the actor, the subject, the previous value, the new value, the
  reason, and the affected page and action

### Requirement: Append-only enforced by the database, not by the application

Audit records SHALL be immutable. No application surface SHALL offer update or delete of an audit
record, and the runtime database identity SHALL hold insert and select rights only on the audit
store — not update, delete, or truncate.

*Source: `SEC-011`, `RET-004`. This is the half of a split whose other half `platform-core` already
built: its `design.md` D3 and `platform/database-access` separate the runtime identity from the
migration identity precisely because this narrowing "is decorative if the same identity can alter
the table". `design.md` D7.*

#### Scenario: Attempted modification through the application

- **WHEN** any user, including an administrator, attempts to modify or delete an audit record
  through the application
- **THEN** the attempt is rejected

#### Scenario: Attempted modification at the database

- **WHEN** the runtime identity attempts to update, delete, or truncate a row in the audit store
- **THEN** the database refuses the operation

#### Scenario: Narrowing verified against a real instance

- **WHEN** the append-only narrowing is verified
- **THEN** the verification runs against a real managed database instance with the real grants, not
  only against a local database where the developer holds superuser rights

### Requirement: Retention and disposal are recorded and separately privileged

Audit records SHALL be retained according to a configured retention policy. Disposal SHALL be
performed by the schema-owning identity rather than the runtime identity, and each disposal SHALL
itself be recorded.

*Source: `RET-004` — immutable retention rules defined by Company policy. `design.md` D7: disposal
is the one deletion the design must account for, and routing it away from the runtime identity is
what keeps the append-only grant absolute rather than acquiring an exception a defect could reach.*

#### Scenario: Retention period elapses

- **WHEN** audit records pass the configured retention period and are disposed of
- **THEN** the disposal is itself recorded, identifying what was disposed of and under which policy

#### Scenario: Runtime identity cannot dispose

- **WHEN** the runtime identity attempts to remove audit records for retention purposes
- **THEN** the database refuses the operation

#### Scenario: Retention period is configurable without deployment

- **WHEN** the retention period is changed through the audited configuration path
- **THEN** the new period applies without a deployment

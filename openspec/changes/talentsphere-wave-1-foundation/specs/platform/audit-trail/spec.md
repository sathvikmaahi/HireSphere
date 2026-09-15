## Purpose

The immutable evidence trail behind every material change in TalentSphere. Audit records must be readable by compliance reviewers without exposing candidate personal data, which is a constraint on what the records may contain rather than a filter applied when displaying them.

## ADDED Requirements

### Requirement: Audit on every material write

Every material write action SHALL produce an audit record.

#### Scenario: Material write through the API

- **WHEN** any endpoint performs a material write
- **THEN** an audit record is written in the same transaction as the change
- **AND** a failure to write the audit record fails the operation rather than silently proceeding

#### Scenario: Read operations

- **WHEN** a user views a record without changing it
- **THEN** no audit record is required, except for the separately audited categories of export and elevated access

### Requirement: Audit record content

Each audit record SHALL capture the acting user or service account, the action, the target type and identifier, previous and new values where applicable, a reason where required, a request correlation identifier, source address where available, and an immutable timestamp.

#### Scenario: Human actor

- **WHEN** a user changes a record
- **THEN** the audit record identifies the user, the action, the target, and both previous and new values

#### Scenario: Service account actor

- **WHEN** a background task performs a material write
- **THEN** the audit record attributes the action to the service account and correlates it to the triggering event

### Requirement: Audit records contain no candidate personal data

Audit records SHALL reference subjects by identifier and event type and SHALL NOT contain candidate names, contact details, resume text, or other candidate personal data in their stored values.

#### Scenario: Compliance reviewer reads the trail

- **WHEN** an Auditor reads audit records relating to candidate activity
- **THEN** the records identify candidates by identifier and describe the event
- **AND** no candidate personal data is present in the returned payload

#### Scenario: A write whose values are sensitive

- **WHEN** a material write changes a field holding personal data
- **THEN** the audit record captures that the field changed, with sensitive values redacted or referenced rather than stored verbatim

### Requirement: Mandatory reasons on sensitive changes

Where a reason is required by the acting workflow, the audit record SHALL carry it, and the underlying action SHALL be rejected when the reason is absent.

#### Scenario: Sensitive change without a reason

- **WHEN** a user performs an action designated as requiring a reason and supplies none
- **THEN** the action is rejected with a field-level validation error and no audit record is written

### Requirement: Immutability

Audit records SHALL be immutable to standard users. No application surface SHALL offer update or delete of an audit record.

#### Scenario: Attempted modification

- **WHEN** any user, including an administrator, attempts to modify or delete an audit record through the application
- **THEN** the attempt is rejected

### Requirement: Audit search

Authorized users SHALL be able to search audit records by actor, target type, target identifier, action, module, and time range.

#### Scenario: Investigating a target

- **WHEN** an Auditor searches for all records affecting one target identifier
- **THEN** the matching records are returned in chronological order with their correlation identifiers

#### Scenario: Unauthorized search

- **WHEN** a user without audit View permission calls the audit search endpoint
- **THEN** the request is denied

### Requirement: Audited, permission-controlled export

Audit and report exports SHALL require Export permission, SHALL carry a data classification label, and SHALL themselves generate an audit event.

#### Scenario: Export produces its own trail

- **WHEN** an authorized user exports audit records
- **THEN** the export is labeled with its data classification
- **AND** an audit record captures who exported what, when, and under which filter

### Requirement: Correlation with AI activity

Audit records and AI run records SHALL share correlation identifiers so a single investigation can span both.

#### Scenario: Tracing an AI-triggered change

- **WHEN** an investigator holds the correlation identifier from an audit record produced by a service-account action
- **THEN** the corresponding AI run record can be retrieved using that identifier

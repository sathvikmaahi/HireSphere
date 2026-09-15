## Purpose

Reading the audit trail: how an Auditor or other authorized investigator searches it, follows one
correlation identifier across audit and AI activity, and exports it under permission with a
classification label and its own audit event. This capability answers "what happened to this
record", not "how are we doing" — reporting is a separate concern. Owned by `TS-BL-021`.

## ADDED Requirements

### Requirement: Audit search

Authorized users SHALL be able to search audit records by actor, target type, target identifier,
action, module, and time range, with results returned in chronological order.

*Source: `reference/spec.md` §14.2 Audit Center. Inherited from
`talentsphere-wave-1-foundation`'s `platform/audit-trail`; relocated here because D.9 makes the
query surface an independently deployable item (`design.md` D2).*

#### Scenario: Investigating a target

- **WHEN** an Auditor searches for all records affecting one target identifier
- **THEN** the matching records are returned in chronological order with their correlation
  identifiers

#### Scenario: Combining filters

- **WHEN** a search combines actor, action, and a time range
- **THEN** only records matching every supplied filter are returned

#### Scenario: Search does not mutate

- **WHEN** any search is performed
- **THEN** no audit record is created, modified, or removed by the search itself

### Requirement: Audit review is permission-gated

Access to the audit search surface and its endpoints SHALL require an audit View permission
resolved by the permission evaluator. A caller without it SHALL be denied, whether calling the
endpoint directly or loading the screen.

*Source: `AUTHZ-003`, `AUTHZ-004`; `C-02`'s Auditor role, which is read-only on audit and AI run
logs. `design.md` D13 records why this item depends on `TS-BL-018` in addition to `TS-BL-020`,
which D.9's table does not show: an Auditor-role-gated surface cannot be built or tested without an
evaluator.*

#### Scenario: Unauthorized search

- **WHEN** a user without audit View permission calls the audit search endpoint
- **THEN** the request is denied

#### Scenario: Auditor is read-only

- **WHEN** the Auditor role is evaluated for Create, Edit, Delete, Approve, or Assign on any
  recruitment resource
- **THEN** access is denied

### Requirement: Reading the trail discloses no candidate personal data

The audit review surface SHALL present records as they are stored, and SHALL NOT resolve
identifiers into candidate names, contact details, or resume content on the reader's behalf.

*Source: `D16`. The no-personal-data property is a constraint on the stored record
(`platform/audit-trail`); this requirement is the corresponding constraint on the reader — a review
surface that helpfully resolved identifiers would reopen the back door the storage constraint
closes.*

#### Scenario: Reader sees identifiers

- **WHEN** an Auditor views records referencing a candidate
- **THEN** the candidate appears as an identifier and the event is described
- **AND** no name, contact detail, or resume content is rendered or returned

#### Scenario: Resolving a reference requires its own permission

- **WHEN** a reader follows a reference from an audit record to the referenced record
- **THEN** access to that record is evaluated on its own terms and denied without the required
  permission

### Requirement: Correlation with AI activity

Audit records and AI run records SHALL share correlation identifiers so that a single investigation
can span both, and the review surface SHALL support retrieving the corresponding records by
correlation identifier.

*Source: `reference/spec.md` §26 Audit Correlation. `ai-platform-governance`'s `TS-BL-028` owns the
AI run record; this requirement fixes the identifier they share.*

#### Scenario: Tracing an AI-triggered change

- **WHEN** an investigator holds the correlation identifier from an audit record produced by a
  service-account action
- **THEN** the corresponding AI run record can be retrieved using that identifier

#### Scenario: Correlation spans an asynchronous boundary

- **WHEN** a background job performs a material write triggered by an interactive request
- **THEN** the audit records from both are retrievable under the same correlation identifier

### Requirement: Audited, permission-controlled export

Audit exports SHALL require Export permission, SHALL carry a data classification label, and SHALL
themselves generate an audit event recording who exported what, when, and under which filter.

*Source: `SEC-012`, `PRV-007`. Sprint 0's own handover flagged that "audited, permission-gated
export" has nothing to gate against until the evaluator exists — which is why this item depends on
`TS-BL-018` (`design.md` D13).*

#### Scenario: Export without permission

- **WHEN** a user without Export permission requests an export of audit records
- **THEN** the request is denied

#### Scenario: Export produces its own trail

- **WHEN** an authorized user exports audit records
- **THEN** the export is labeled with its data classification
- **AND** an audit record captures who exported what, when, and under which filter

#### Scenario: Export honors the same content constraint

- **WHEN** audit records are exported
- **THEN** the exported content contains no candidate personal data, exactly as the stored records
  do

### Requirement: Review surface performance is bounded by indexed access paths

The search filters this capability exposes SHALL be supported by indexed access paths, so that
audit volume growth does not degrade an investigation into a full scan.

*Source: `NFR-005` — the data model must support growth in audit records — and `NFR-001`'s
three-second bound on common authenticated screens. Audit volume grows monotonically and is never
pruned except by retention, which makes this the one screen whose worst case arrives with time
rather than with load.*

#### Scenario: Search under grown volume

- **WHEN** a filtered search is performed against a substantially populated audit store
- **THEN** it resolves through an indexed access path rather than a full scan

#### Scenario: Unindexed filter combination

- **WHEN** a filter combination the capability exposes has no supporting index
- **THEN** the test suite fails rather than the query silently degrading

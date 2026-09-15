## Purpose

The evidence an elevation leaves behind: not only that it was granted, but what was actually done
under it, and that it ended. Without this, break-glass answers "who was allowed to look" and never
"what they looked at" — which is the weak after-the-fact control `D16` rejected. Owned by
`TS-BL-026`.

## ADDED Requirements

### Requirement: The elevation lifecycle is fully recorded

Every stage of an elevation — request, grant, revocation, and expiry — SHALL produce its own audit
record. Expiry SHALL be recorded as an event rather than inferred from a timestamp having passed.

*Source: `D16` — "explicit logged elevation"; `SEC-010`. `design.md` D10: an expiry that is only a
column value leaves an investigator reconstructing when access actually ended, which is the
question they came to answer.*

#### Scenario: Elevation granted

- **WHEN** an elevation is granted
- **THEN** an audit record captures the requesting administrator, the granted permissions, the
  reason, the duration, and the notified recipients

#### Scenario: Elevation expires

- **WHEN** an elevation's duration elapses
- **THEN** an audit record captures the expiry as its own event

#### Scenario: Elevation revoked early

- **WHEN** an administrator ends an active elevation before its expiry
- **THEN** an audit record captures the revocation, the acting administrator, and the time

#### Scenario: Elevation request refused

- **WHEN** an elevation request is rejected
- **THEN** the refusal is recorded, with no elevation created

### Requirement: Access performed under an elevation is attributed to it

Every access to candidate personal data performed while an elevation is active SHALL be recorded
and attributed to that elevation, including read access.

*Source: `platform/audit-trail`'s rule that reads require no audit record **except** the separately
audited categories of export and elevated access — this is one of those two exceptions. `D16`'s
rejection of "audit log as the only control" turns on knowing what was accessed, not merely who was
permitted.*

#### Scenario: Elevated read

- **WHEN** an elevated administrator views candidate personal data
- **THEN** an audit record captures the access, the record accessed by identifier, and the
  elevation it was performed under

#### Scenario: Unelevated reads are not audited

- **WHEN** an ordinary user views a record they are permitted to see
- **THEN** no audit record is produced for the read

#### Scenario: Access after expiry

- **WHEN** the same administrator accesses candidate personal data after the elevation has ended
- **THEN** access is denied and no access is attributed to the expired elevation

### Requirement: One correlation identifier spans an elevation and its accesses

An elevation and every access performed under it SHALL share a correlation identifier, so a single
investigation retrieves the grant, the accesses, and the ending together.

*Source: `reference/spec.md` §26 Audit Correlation; `design.md` D10.*

#### Scenario: Investigating an elevation

- **WHEN** an investigator holds an elevation's correlation identifier
- **THEN** the grant, every access performed under it, and its ending are retrievable together

#### Scenario: Reaching the elevation from an access

- **WHEN** an investigator holds an audit record of an elevated access
- **THEN** the elevation that authorized it is retrievable from that record

### Requirement: Elevation records carry no candidate personal data

Records of elevated access SHALL identify accessed records by identifier and SHALL NOT contain the
candidate personal data that was viewed.

*Source: `D16`'s central consequence and `platform/audit-trail`'s content constraint. Recording
elevated access in order to protect candidate data, in a form that copies that data into a table
the Auditor reads, would defeat the purpose exactly.*

#### Scenario: Elevated access record inspected

- **WHEN** an Auditor reads the record of an elevated access
- **THEN** it names the record accessed by identifier and the elevation
- **AND** it contains none of the personal data that was viewed

### Requirement: Elevated activity is reviewable as a set

An authorized reviewer SHALL be able to retrieve elevations by administrator, by subject, by reason,
and by time range, together with the accesses performed under each.

*Source: `C-02`'s Auditor role, read-only on audit records; `reference/spec.md` §14.2's Audit
Center. An elevation trail that can only be read one correlation identifier at a time cannot answer
"has this administrator been elevating unusually often", which is the question a periodic review
asks.*

#### Scenario: Reviewing one administrator's elevations

- **WHEN** an Auditor retrieves elevations for one administrator over a time range
- **THEN** each elevation is returned with its reason, duration, ending, and the accesses performed
  under it

#### Scenario: Unauthorized review

- **WHEN** a user without audit View permission requests the elevation review surface
- **THEN** the request is denied

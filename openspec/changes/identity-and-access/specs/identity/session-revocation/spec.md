## Purpose

Ends a session before it would expire on its own, immediately and for every subsequent request —
whether an administrator revokes it deliberately or a deactivation revokes it as a consequence.
This is the answer to the leaver problem: a departed employee holding a live session is an audit
finding, not an inconvenience. Owned by `TS-BL-015`.

## ADDED Requirements

### Requirement: Forced session revocation

An Application Administrator SHALL be able to revoke another user's active sessions. A revoked
session SHALL become invalid immediately for every subsequent request, with no permissible caching
window.

*Source: `G-11`, adopting `AUTH-006`/`AUTH-007`. `config.yaml`'s standing rule states the same
property from the authorization side — uncached, so revocation bites on the next request.*

#### Scenario: Administrator revokes an active session

- **WHEN** an Application Administrator revokes a user's session
- **THEN** the next request presenting that session is rejected as unauthenticated
- **AND** the revocation is recorded in the audit trail with actor, subject, and timestamp

#### Scenario: All sessions for one user

- **WHEN** a user holding several concurrent sessions is revoked
- **THEN** every one of that user's sessions is invalidated, not only the most recent

#### Scenario: No delay is permitted

- **WHEN** a request presenting a revoked session arrives immediately after revocation
- **THEN** it is rejected
- **AND** no cache, token lifetime, or replication interval is permitted to delay the rejection

#### Scenario: Revoking an already-ended session

- **WHEN** revocation targets a session that has already expired or been logged out
- **THEN** the operation succeeds without error and the session remains invalid

### Requirement: Revocation is restricted and the restriction is declared

The revocation surface SHALL declare the permission it demands, and SHALL deny in the absence of a
decision from the permission evaluator. A user SHALL NOT be able to revoke another user's sessions
without that permission.

*Source: `AUTHZ-001` deny-by-default, `AUTHZ-003` server-side enforcement on every endpoint,
`SEC-010` admin actions require strong authorization. The evaluator is `TS-BL-018` in
`access-control-and-admin`; per `design.md` D5 and D7 the requirement is declared here and
enforced when that arrives, rather than approximated with a role check.*

#### Scenario: Permission requirement declared

- **WHEN** the revocation endpoint is enumerated
- **THEN** it is classified as permission-gated and names the permission it demands

#### Scenario: No evaluator installed

- **WHEN** revocation is requested while no permission evaluator is installed
- **THEN** the request is denied rather than allowed

#### Scenario: Self-revocation is logout

- **WHEN** a user ends their own session
- **THEN** it is treated as logout and requires no administrative permission

### Requirement: Deactivation revokes as a consequence, not as a separate step

Deactivating a user SHALL revoke that user's active sessions as part of the deactivation, without
waiting for expiry and without requiring a separate administrative action.

*Source: `G-11` and `D08`'s leaver problem — a terminated recruiter with a live session. Coupling
the two means the audit finding cannot be produced by an administrator who simply forgot the
second step.*

#### Scenario: Deactivated user's sessions end

- **WHEN** a user is deactivated
- **THEN** all of that user's active sessions are revoked and subsequent requests are rejected

#### Scenario: Deactivation and revocation are one operation

- **WHEN** deactivation succeeds
- **THEN** revocation of that user's sessions has succeeded with it
- **AND** neither outcome can be observed without the other

#### Scenario: Re-activation issues nothing

- **WHEN** a previously deactivated user is activated again
- **THEN** no previously revoked session is restored
- **AND** the user must authenticate afresh

### Requirement: A revoked session is distinguishable from an expired one in the record

The stored outcome of a session SHALL distinguish revocation from ordinary expiry and from
logout. The response to the holder of a revoked session SHALL NOT disclose which occurred.

*Source: `AUTH-004` session status; `SEC-015` secure error handling. An investigator needs to
know a session was cut short; the person holding it does not need to be told they were
individually targeted.*

#### Scenario: Investigator examines a session

- **WHEN** an ended session is examined in the record
- **THEN** its status identifies whether it expired, was logged out, or was revoked

#### Scenario: Holder of a revoked session

- **WHEN** the holder of a revoked session makes a request
- **THEN** the response reports that authentication is required
- **AND** it does not disclose that the session was revoked rather than expired

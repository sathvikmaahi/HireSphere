## Purpose

Decides whether an authenticated enterprise identity is admitted into TalentSphere at all,
independently of whether Hubble authenticated it. Covers the first login of an unknown identity,
administrative activation and deactivation, the access-denied path, and the invariant that a
departing user's work outlives their access. Owned by `TS-BL-016`.

## ADDED Requirements

### Requirement: Activation required for access

Only activated and authorized users SHALL access TalentSphere. An authenticated Hubble user who is
not activated SHALL receive an access-denied response and SHALL NOT receive application data.

*Source: `AUTH-005`, `BR-001`.*

#### Scenario: Authenticated but not activated

- **WHEN** a user authenticates successfully through Hubble but has no activated TalentSphere user
  record
- **THEN** the user is shown an access-denied screen
- **AND** no recruitment, candidate, posting, or configuration data is returned by any endpoint

#### Scenario: Activated with no permissions

- **WHEN** an activated user holds no role grants and no direct grants
- **THEN** the user is admitted to the application shell but every page and action is denied
- **AND** the denial is explainable through the permission explanation surface

#### Scenario: Deactivated user attempts to sign in

- **WHEN** a deactivated user authenticates successfully through Hubble
- **THEN** access is denied and no session is established

### Requirement: First login of an unknown identity creates a pending record

An identity that Hubble authenticates and that has no local user record SHALL be recorded in a
pending, unactivated state carrying no permissions, and SHALL be visible to administrators for
activation.

*Source: `ADM-001`; `D08`'s open question "JIT user creation on first login, or Admin
pre-provisioning?" — resolved as both, because pre-provisioning alone leaves a legitimate new
employee with nothing an administrator can find and activate.*

#### Scenario: First login of an unknown identity

- **WHEN** an identity that Hubble authenticates has no local user record
- **THEN** the system records the identity in a pending, unactivated state carrying no permissions
- **AND** the pending record is visible to administrators for activation

#### Scenario: Pending record grants nothing

- **WHEN** an identity holding only a pending record makes any request
- **THEN** access is denied and no application data is returned

#### Scenario: Repeated first logins

- **WHEN** the same unknown identity authenticates repeatedly
- **THEN** one pending record exists for it, refreshed rather than duplicated

### Requirement: Activation state transitions

A local user SHALL hold exactly one status among active, inactive, pending, and deactivated. Every
status change SHALL be audited with actor, previous value, new value, timestamp, and reason.

*Source: `reference/spec.md` §12.2 `users.status`; `ADM-001`; `AUTHZ-007` — permission and access
changes audited with previous and new value.*

#### Scenario: Activation

- **WHEN** an Application Administrator activates a pending user
- **THEN** the user status becomes active and an audit record is written

#### Scenario: Deactivation requires a reason

- **WHEN** an administrator deactivates a user without supplying a reason
- **THEN** the request is rejected with a field-level validation error

#### Scenario: Transition audited with both values

- **WHEN** any user status change succeeds
- **THEN** the audit record carries the previous status, the new status, the acting administrator,
  the reason, and the request correlation identifier

#### Scenario: Failed audit fails the transition

- **WHEN** the audit record for a status change cannot be written
- **THEN** the status change does not take effect

### Requirement: Deactivation preserves owned work and historical attribution

Deactivating a user SHALL NOT delete, orphan, or reassign records the user owned or authored, and
SHALL NOT alter the actor recorded on historical activity. Owned work SHALL remain reassignable to
another user.

*Source: `G-11`'s pairing with ownership reassignment and `D08`'s departing recruiter who owned
twelve open postings. The reassignment **workflow** is `TS-BL-080` in `access-control-and-admin`,
recorded in `exploration-notes.md` D.10.1 after this feature found it missing from the backlog;
this requirement is the invariant that workflow must satisfy and that nothing may violate in the
meantime.*

#### Scenario: Owned work survives deactivation

- **WHEN** a user who owns assigned work items is deactivated
- **THEN** those items remain intact and remain reassignable to another user
- **AND** the original actor attribution on historical records is preserved unchanged

#### Scenario: No silent re-attribution

- **WHEN** a deactivated user's owned work is examined
- **THEN** it still names that user as its owner until an explicit reassignment occurs

#### Scenario: Deletion refused

- **WHEN** deletion of a user record holding owned or authored work is attempted
- **THEN** it is refused in favour of deactivation

### Requirement: Access-denied surface discloses nothing

The access-denied response SHALL NOT disclose recruitment data, role names, user lists, or system
configuration, and SHALL provide the user a route to request access.

*Source: `AUTH-005`, `SEC-015`. The correlation identifier is present because
`platform/api-ingress` assigns one at the edge to every request, including those rejected before
reaching application code — it is what lets a support case be traced without the screen revealing
anything.*

#### Scenario: Unactivated user inspects the response

- **WHEN** an unactivated user receives the access-denied screen and inspects the underlying
  response
- **THEN** the payload contains only the denial, a correlation identifier, and contact guidance

#### Scenario: Denial does not distinguish causes

- **WHEN** access is denied to an identity that is unknown, pending, or deactivated
- **THEN** the response does not reveal which of those applies

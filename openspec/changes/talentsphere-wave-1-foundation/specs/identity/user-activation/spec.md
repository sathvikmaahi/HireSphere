## Purpose

Decides whether an authenticated enterprise identity is admitted into TalentSphere at all, independently of whether Hubble authenticated it. Covers local user records, activation states, the access-denied path, and leaver handling.

## ADDED Requirements

### Requirement: Activation required for access

Only activated and authorized users SHALL access TalentSphere. An authenticated Hubble user who is not activated SHALL receive an access-denied response and SHALL NOT receive application data.

#### Scenario: Authenticated but not activated

- **WHEN** a user authenticates successfully through Hubble but has no activated TalentSphere user record
- **THEN** the user is shown an access-denied screen
- **AND** no recruitment, candidate, posting, or configuration data is returned by any endpoint

#### Scenario: Activated with no permissions

- **WHEN** an activated user holds no role grants and no direct grants
- **THEN** the user is admitted to the application shell but every page and action is denied
- **AND** the denial is explainable through the permission explanation surface

### Requirement: Local user records

The system SHALL maintain a local user record for each known Hubble identity, created either by administrative action or on first authentication, and SHALL treat local records as the authority for access rather than the Hubble directory.

#### Scenario: Administrator pre-provisions a user

- **WHEN** an Application Administrator creates a local user for a known Hubble identity and activates it
- **THEN** that user is admitted on next login with the roles assigned

#### Scenario: First login of an unknown identity

- **WHEN** an identity that Hubble authenticates has no local user record
- **THEN** the system records the identity in a pending, unactivated state carrying no permissions
- **AND** the pending record is visible to administrators for activation

### Requirement: Activation state transitions

A local user SHALL hold exactly one status among active, inactive, pending, and deactivated. Every status change SHALL be audited with actor, previous value, new value, timestamp, and reason.

#### Scenario: Activation

- **WHEN** an Application Administrator activates a pending user
- **THEN** the user status becomes active and an audit record is written

#### Scenario: Deactivation requires a reason

- **WHEN** an administrator deactivates a user without supplying a reason
- **THEN** the request is rejected with a field-level validation error

### Requirement: Leaver handling and ownership continuity

Deactivating a user SHALL immediately revoke that user's active sessions and SHALL NOT delete or orphan records the user owned or authored.

#### Scenario: Deactivated user's sessions end

- **WHEN** a user is deactivated
- **THEN** all of that user's active sessions are revoked and subsequent requests are rejected

#### Scenario: Owned work survives deactivation

- **WHEN** a user who owns assigned work items is deactivated
- **THEN** those items remain intact and remain reassignable to another user
- **AND** the original actor attribution on historical records is preserved unchanged

### Requirement: Multiple roles per user

A user SHALL be assignable to more than one role. Effective permissions SHALL be the union of all grants from all assigned roles, subject to explicit denials and to the most restrictive applicable data scope.

#### Scenario: A manager who also interviews

- **WHEN** one user holds both the Practice Manager and Interviewer roles
- **THEN** the user receives the union of both roles' action grants
- **AND** where the two roles imply different data scopes for the same resource, the more restrictive scope applies

### Requirement: Practice as a data attribute

A user record SHALL carry an optional practice attribute, used for filtering and reporting only. The practice attribute SHALL NOT participate in any authorization decision.

#### Scenario: Filtering by practice

- **WHEN** an administrator filters users or the permission matrix by practice
- **THEN** only users carrying that practice value are listed

#### Scenario: Practice never authorizes

- **WHEN** two users differing only in their practice attribute are evaluated for the same page and action
- **THEN** both receive the same verdict

#### Scenario: Practice is optional

- **WHEN** a user record carries no practice value
- **THEN** the user is unaffected in access terms and appears only under an unfiltered or unassigned-practice view

### Requirement: Access-denied surface discloses nothing

The access-denied response SHALL NOT disclose recruitment data, role names, user lists, or system configuration, and SHALL provide the user a route to request access.

#### Scenario: Unactivated user inspects the response

- **WHEN** an unactivated user receives the access-denied screen and inspects the underlying response
- **THEN** the payload contains only the denial, a correlation identifier, and contact guidance

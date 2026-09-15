## Purpose

Authenticates interactive users against the enterprise Hubble identity provider and manages the local TalentSphere session lifecycle, including administrative revocation. Authentication is external; authorization is decided locally and separately.

## ADDED Requirements

### Requirement: Hubble-backed authentication

The system SHALL authenticate all interactive users through the Hubble REST Login API. The system SHALL NOT provide any alternative interactive credential path, and SHALL NOT accept locally stored passwords.

#### Scenario: Valid enterprise credentials

- **WHEN** a user submits credentials that Hubble confirms as valid
- **THEN** the system creates or refreshes a local TalentSphere session mapped to the returned Hubble identity
- **AND** the session is returned to the client with its expiry timestamp

#### Scenario: Invalid credentials

- **WHEN** Hubble rejects the submitted credentials
- **THEN** the system establishes no session
- **AND** the response distinguishes authentication failure from authorization failure without revealing whether the identity exists

#### Scenario: Identity provider unavailable

- **WHEN** the Hubble API times out or returns an error
- **THEN** the system reports an authentication service failure to the user
- **AND** the system does not fall back to any local or cached credential check

### Requirement: No credential retention

The system SHALL NOT store Hubble passwords or any reusable Hubble credential, in any store, log, or cache.

#### Scenario: Post-login inspection

- **WHEN** a login succeeds
- **THEN** no Hubble password or reusable Hubble credential is present in the database, object storage, application logs, or session record

### Requirement: Session content

Every established session SHALL carry the TalentSphere user ID, Hubble user identifier, display name, email, assigned roles, resolved page permissions, login timestamp, expiry timestamp, and session status.

#### Scenario: Session introspection

- **WHEN** an authenticated client requests its own session context
- **THEN** the response contains the user identity, roles, resolved permissions, and session state
- **AND** the response contains no other user's identity or permission data

### Requirement: Session lifecycle

The system SHALL support logout, session expiration, and session renewal. Session expiration duration SHALL be configurable without code changes.

#### Scenario: Renewal before expiry

- **WHEN** a client refreshes a session that has not expired and has not been revoked
- **THEN** the session expiry is extended and the same session identity is retained

#### Scenario: Expired session

- **WHEN** a client makes a request with a session past its expiry timestamp
- **THEN** the request is rejected as unauthenticated
- **AND** no application data is returned

#### Scenario: Logout

- **WHEN** a user logs out
- **THEN** the session becomes invalid for all subsequent requests

### Requirement: Forced session revocation

An Application Administrator SHALL be able to revoke another user's active sessions. A revoked session SHALL become invalid immediately for every subsequent request, with no permissible caching window.

#### Scenario: Administrator revokes an active session

- **WHEN** an Application Administrator revokes a user's session
- **THEN** the next request presenting that session is rejected as unauthenticated
- **AND** the revocation is recorded in the audit trail with actor, subject, and timestamp

#### Scenario: Leaver with a live session

- **WHEN** a user is deactivated while holding an unexpired session
- **THEN** that session is revoked as part of deactivation without waiting for expiry

### Requirement: Identity mapping

The system SHALL map each Hubble identity to exactly one local user record, keyed on the Hubble user identifier rather than on email.

#### Scenario: Returning known identity

- **WHEN** a user whose Hubble identifier already maps to a local user authenticates
- **THEN** the session is bound to that existing local user
- **AND** display name and email claims received from Hubble are refreshed on the local record

#### Scenario: Email changes upstream

- **WHEN** an authenticated identity presents a different email than the local record holds, with an unchanged Hubble identifier
- **THEN** the system updates the email on the existing local user rather than creating a second user

### Requirement: Authentication rate limiting

The system SHALL enforce rate limits on authentication attempts.

#### Scenario: Repeated failed attempts

- **WHEN** authentication attempts for the same identity or source exceed the configured threshold
- **THEN** further attempts are throttled
- **AND** the throttling event is recorded

### Requirement: Authentication logging

The system SHALL log authentication successes and failures, including metadata sufficient for investigation, without exposing credential details.

#### Scenario: Failure logging

- **WHEN** an authentication attempt fails
- **THEN** a record is written capturing timestamp, outcome, reason category, and request correlation ID
- **AND** the record contains no submitted password or token value

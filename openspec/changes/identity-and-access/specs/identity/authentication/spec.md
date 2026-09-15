## Purpose

Authenticates interactive users against the enterprise Hubble identity provider, behind a single
adapter so that the unconfirmed Hubble contract (`OD-001`) is one replaceable seam rather than a
dependency spread through the application. Authentication is external and decides only *who is
speaking*; whether that person is admitted, and what they may do, are decided locally and
elsewhere. Owned by `TS-BL-013`.

## ADDED Requirements

### Requirement: Hubble-backed authentication

The system SHALL authenticate all interactive users through the Hubble REST Login API. The system
SHALL NOT provide any alternative interactive credential path, and SHALL NOT accept locally stored
passwords.

*Source: `AUTH-001`, `G-002`. Authentication external, authorization internal, is the split
`D08` fixes for the whole product.*

#### Scenario: Valid enterprise credentials

- **WHEN** a user submits credentials that Hubble confirms as valid
- **THEN** the system creates or refreshes a local TalentSphere session mapped to the returned
  Hubble identity
- **AND** the session is returned to the client with its expiry timestamp

#### Scenario: Invalid credentials

- **WHEN** Hubble rejects the submitted credentials
- **THEN** the system establishes no session
- **AND** the response distinguishes authentication failure from authorization failure without
  revealing whether the identity exists

#### Scenario: No local credential path exists

- **WHEN** the authentication surfaces are enumerated
- **THEN** Hubble is the only path by which an interactive session can be established
- **AND** no endpoint accepts a locally stored password

### Requirement: Identity provider isolated behind a replaceable adapter

All Hubble interaction SHALL be confined to a single adapter that accepts credentials and returns
normalized internal identity claims. No other part of the system SHALL depend on the shape of a
Hubble response.

*Source: inherited `design.md` D8 from `talentsphere-wave-1-foundation`, assigned to this feature
by `platform-core`'s D11. `AUTH-009` requires the Hubble timeout, refresh behaviour, token shape
and identity payload to be confirmed before implementation is finalized; the adapter is what
absorbs that confirmation.*

#### Scenario: Downstream consumers see normalized claims

- **WHEN** any component other than the adapter handles an authenticated identity
- **THEN** it receives normalized internal claims
- **AND** no Hubble-specific response structure is present outside the adapter

#### Scenario: Unknown or missing claim

- **WHEN** the identity provider returns a claim the system does not recognize, or omits one it
  expects
- **THEN** the adapter maps the situation explicitly rather than substituting a silent default
- **AND** an omitted required claim fails authentication rather than producing a partial identity

#### Scenario: Recorded contract belief

- **WHEN** the adapter is built while `OD-001` remains unconfirmed
- **THEN** a contract test records the response shape currently believed correct
- **AND** a substitute provider satisfying that contract serves Local and Dev

### Requirement: No credential retention

The system SHALL NOT store Hubble passwords or any reusable Hubble credential, in any store, log,
or cache.

*Source: `AUTH-002`, `SEC-009`.*

#### Scenario: Post-login inspection

- **WHEN** a login succeeds
- **THEN** no Hubble password or reusable Hubble credential is present in the database, object
  storage, application logs, or session record

#### Scenario: Credential in an error path

- **WHEN** authentication fails and the failure is reported and logged
- **THEN** neither the submitted credential nor any provider token value appears in the response
  or in any log record

### Requirement: Identity mapping

The system SHALL map each Hubble identity to exactly one local user record, keyed on the Hubble
user identifier rather than on email.

*Source: `AUTH-003`; inherited D8 — email is the claim most likely to change or be absent. That
argues against mapping **on** email; it does not argue against email being unique, which
`reference/spec.md` §12.2 requires and `identity/user-model` enforces.*

#### Scenario: Returning known identity

- **WHEN** a user whose Hubble identifier already maps to a local user authenticates
- **THEN** the session is bound to that existing local user
- **AND** display name and email claims received from Hubble are refreshed on the local record

#### Scenario: Email changes upstream

- **WHEN** an authenticated identity presents a different email than the local record holds, with
  an unchanged Hubble identifier
- **THEN** the system updates the email on the existing local user rather than creating a second
  user

#### Scenario: Refreshed email collides with another user

- **WHEN** an authenticated identity presents an email address that another local user record
  already holds
- **THEN** the session is established, because the Hubble identifier confirms who authenticated
- **AND** the email on the local record is left unchanged rather than violating uniqueness
- **AND** the collision is recorded for administrative attention

### Requirement: Behavior when the identity provider is unavailable

When Hubble cannot be reached or does not answer within the configured timeout, the system SHALL
report an authentication service failure and SHALL NOT fall back to any local or cached credential
check.

*Source: `AUTH-009`, `NFR-004`/`G-12`. Graceful degradation covers AI, OCR, notification and
vector outages; it deliberately does not extend to the identity provider, because degrading
authentication means admitting someone unverified.*

#### Scenario: Identity provider timeout

- **WHEN** the Hubble API times out or returns an error
- **THEN** the system reports an authentication service failure to the user
- **AND** the system does not fall back to any local or cached credential check
- **AND** no session is established

#### Scenario: Existing sessions during an outage

- **WHEN** the identity provider is unavailable and a user holds an unexpired, unrevoked session
- **THEN** that session continues to be accepted
- **AND** only the establishment of new sessions is affected

### Requirement: Authentication rate limiting

The system SHALL enforce rate limits on authentication attempts, per identity and per source. The
threshold SHALL be changeable without a deployment.

*Source: `SEC-013`. The threshold key `auth.rate_limit_attempts_per_minute` already exists in the
runtime configuration registry; this requirement consumes it rather than introducing a second
mechanism.*

#### Scenario: Repeated failed attempts

- **WHEN** authentication attempts for the same identity or source exceed the configured
  threshold
- **THEN** further attempts are throttled
- **AND** the throttling event is recorded

#### Scenario: Threshold changed at runtime

- **WHEN** an administrator changes the authentication rate-limit threshold
- **THEN** the new threshold takes effect without a redeployment
- **AND** the change is audited with a mandatory reason

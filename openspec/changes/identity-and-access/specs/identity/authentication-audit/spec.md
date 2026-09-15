## Purpose

Turns authentication activity into audit events — the first events the product's audit trail is
built to accept. Defines what a login, a failed login, a logout and a revocation record, who is
named as the actor when nobody is authenticated, and what such a record may never contain. Owned
by `TS-BL-017`.

## ADDED Requirements

### Requirement: Authentication events are emitted as audit events

Successful authentication, failed authentication, logout, forced revocation, and user status
changes SHALL each emit an audit event through the system's single audit interface. No
authentication surface SHALL write to an audit store by any other path.

*Source: `AUTH-008`, `SEC-010`; `config.yaml`'s rule that every material write produces an audit
record. The audit interface already exists (`AuditEvent` and the sink protocol, defined in Sprint 0
so that the event shape would not be defined twice); this capability adds the authentication
vocabulary to it rather than a second mechanism.*

#### Scenario: Successful login

- **WHEN** a user authenticates successfully
- **THEN** an audit event is emitted naming the user as actor, with outcome, timestamp, source
  address and correlation identifier

#### Scenario: Logout and revocation

- **WHEN** a session ends by logout or by forced revocation
- **THEN** an audit event is emitted distinguishing which occurred
- **AND** a revocation event names both the acting administrator and the affected user

#### Scenario: One path only

- **WHEN** the writes this capability makes to audit storage are enumerated
- **THEN** all of them pass through the single audit interface

### Requirement: Failed authentication is attributed to the system, not the stranger

A failed authentication event SHALL name the authentication subsystem as its actor and the
attempted identity as its target. It SHALL NOT be recorded without an actor.

*Source: `AUTH-008`; `C-02`'s AI Service Account precedent, where a non-human actor is modelled as
a first-class actor rather than left null. An audit event requires an actor by construction, and a
failed login has no authenticated user — recording the system as the actor is also the accurate
reading, since the system is what acted.*

#### Scenario: Failure by an unknown identity

- **WHEN** authentication fails for an identity with no local user record
- **THEN** an audit event is emitted with the authentication subsystem as actor and the attempted
  identity as target

#### Scenario: Failure by a known user

- **WHEN** authentication fails for an identity that maps to a known local user
- **THEN** the event names that user as the target and remains attributed to the subsystem as
  actor

#### Scenario: Reason category recorded

- **WHEN** an authentication failure is recorded
- **THEN** it carries a reason category distinguishing rejected credentials, throttling, an
  unavailable identity provider, and denied access for an unactivated identity

### Requirement: Authentication records carry no credential material

An authentication audit event or log record SHALL NOT contain a submitted password, a provider
token, a session credential, or a raw identity-provider response body.

*Source: `AUTH-008` — log failures "without exposing sensitive credential details"; `AUTH-002`,
`SEC-015`, `SEC-009`. Redaction is applied at write time, not as a display filter.*

#### Scenario: Failure logging

- **WHEN** an authentication attempt fails
- **THEN** a record is written capturing timestamp, outcome, reason category, and request
  correlation ID
- **AND** the record contains no submitted password or token value

#### Scenario: Provider response not retained

- **WHEN** the identity provider returns an error body
- **THEN** the recorded event carries a reason category rather than the body itself

### Requirement: Authentication events survive the arrival of the durable trail

Authentication events SHALL be emitted against the audit interface from the first release,
regardless of whether the durable audit store exists yet. When the durable writer is installed, no
emitting surface SHALL require modification, and records SHALL become durable without any change
to what is emitted.

*Source: `D.9`'s dependency direction — `TS-BL-020`, the durable audit write substrate, depends on
this item, because the login event is the first event that substrate is built to accept. The
interim state is the same one the runtime-configuration path already occupies: events reach the
structured log and are explicitly marked as not durable.*

#### Scenario: Before the durable writer exists

- **WHEN** an authentication event is emitted and no durable audit store is installed
- **THEN** the event is recorded through the structured log and marked as not durable

#### Scenario: After the durable writer is installed

- **WHEN** the durable audit writer is installed
- **THEN** authentication events are persisted to the audit store
- **AND** no authentication surface required a change to make that happen

#### Scenario: Correlation preserved across the change

- **WHEN** an authentication event is recorded in either state
- **THEN** it carries the request correlation identifier assigned at the edge

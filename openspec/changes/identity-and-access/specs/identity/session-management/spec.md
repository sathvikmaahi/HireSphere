## Purpose

Issues and maintains the local TalentSphere session that every authenticated request is judged
against — its content, its transport, its renewal and its end. Sessions are held as records the
system can reach and invalidate, rather than as self-validating tokens, because immediate
revocation is a requirement and not an optimization. Owned by `TS-BL-014`.

## ADDED Requirements

### Requirement: Session content

Every established session SHALL carry the TalentSphere user ID, Hubble user identifier, display
name, email, assigned roles, resolved page permissions, login timestamp, expiry timestamp, and
session status.

*Source: `AUTH-004`. Resolved permissions are obtained through the interface
`identity/user-model` declares, and are an empty set until the evaluator (`TS-BL-018`) exists —
the field is present in the contract from the first release so that consumers are not rewritten
when it fills.*

#### Scenario: Session introspection

- **WHEN** an authenticated client requests its own session context
- **THEN** the response contains the user identity, roles, resolved permissions, and session state
- **AND** the response contains no other user's identity or permission data

#### Scenario: Introspection before the evaluator exists

- **WHEN** a client requests its own session context and no permission evaluator is installed
- **THEN** the permissions field is present and empty rather than absent

#### Scenario: Unauthenticated introspection

- **WHEN** session context is requested without a valid session
- **THEN** the request is rejected as unauthenticated and no user data is returned

### Requirement: Sessions are revocable records, not self-validating tokens

Session validity SHALL be determined by consulting stored session state on each request. The
system SHALL NOT accept any credential that remains valid on its own contents after the session it
represents has been revoked.

*Source: `AUTH-007`, and `config.yaml`'s standing rule that authorization is uncached so that
revocation bites on the next request. `design.md` D6 records the alternative rejected.*

#### Scenario: Session state consulted per request

- **WHEN** an authenticated request is made
- **THEN** the stored state of the presented session determines whether it is accepted

#### Scenario: Credential outliving its session

- **WHEN** the credentials the system issues are examined
- **THEN** none can be validated without consulting stored session state

### Requirement: Session transport

The session credential SHALL be transmitted in a form not readable by page scripts, sent only over
encrypted connections, and restricted from cross-site submission.

*Source: `SEC-001`, `SEC-007`; `platform/api-ingress`'s TLS-only edge. No public routes exist
(`project.md`), so the exposed surface is a signed-in staff application.*

#### Scenario: Script access attempted

- **WHEN** page script attempts to read the session credential
- **THEN** it is not accessible

#### Scenario: Plaintext transmission attempted

- **WHEN** a request carrying a session credential is made without encryption
- **THEN** it is refused or redirected to an encrypted connection

### Requirement: Session lifecycle

The system SHALL support logout, session expiration, and session renewal. Session expiration
duration SHALL be configurable without code changes.

*Source: `AUTH-006`. The key `session.expiration_minutes` already exists in the runtime
configuration registry with a provisional default; this requirement consumes it.*

#### Scenario: Renewal before expiry

- **WHEN** a client refreshes a session that has not expired and has not been revoked
- **THEN** the session expiry is extended and the same session identity is retained

#### Scenario: Expired session

- **WHEN** a client makes a request with a session past its expiry timestamp
- **THEN** the request is rejected as unauthenticated
- **AND** no application data is returned

#### Scenario: Renewal of an expired session

- **WHEN** a client attempts to renew a session that has already expired
- **THEN** renewal is refused and a new authentication is required

#### Scenario: Logout

- **WHEN** a user logs out
- **THEN** the session becomes invalid for all subsequent requests

#### Scenario: Expiry duration changed at runtime

- **WHEN** an administrator changes the configured session expiration duration
- **THEN** sessions established afterwards use the new duration without a redeployment
- **AND** the change is audited with a mandatory reason

### Requirement: Authentication state is required by default on every route

Every route the system exposes SHALL declare whether it requires a session and what permission it
demands. A route that declares nothing SHALL deny.

*Source: `API-001`, `AUTHZ-003`, `AUTHZ-004`, and `platform/api-ingress`'s "Every exposed endpoint
is classified" requirement, whose residual Sprint 0 gap this capability must not widen. The
permission mechanism itself belongs to `TS-BL-018`; the declaration obligation is met here.*

#### Scenario: Sign-in is the only unauthenticated route

- **WHEN** the routes this capability exposes are enumerated
- **THEN** only the login route is reachable without a session
- **AND** every other route requires one

#### Scenario: Route without a declaration

- **WHEN** a route is exposed carrying no authentication or permission declaration
- **THEN** requests to it are denied

#### Scenario: Direct call bypassing the interface

- **WHEN** an authenticated route is called directly rather than through the user interface
- **THEN** the same session and declaration checks apply

### Requirement: One session context per request, resolved once

An authenticated request SHALL resolve its session context once and SHALL present the same user
identity, roles and permissions to every component handling that request.

*Source: `config.yaml` — verdict and explanation come from the same evaluation pass, never
reconstructed separately. A request that resolved identity twice could disagree with itself
across a revocation.*

#### Scenario: Consistent identity within a request

- **WHEN** several components handle one authenticated request
- **THEN** each sees the same user identity, roles and resolved permissions

#### Scenario: Revocation between requests

- **WHEN** a session is revoked while a request is in flight and a further request follows
- **THEN** the in-flight request completes under the context it resolved
- **AND** the following request is rejected

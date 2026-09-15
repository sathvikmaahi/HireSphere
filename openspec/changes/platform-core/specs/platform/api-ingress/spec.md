## Purpose

The single controlled entry point into TalentSphere: every request arrives through the gateway,
direct invocation of the backing service is refused, and the routes the gateway exposes are
generated from the infrastructure definitions rather than maintained by hand. Owned by
`TS-BL-003`.

## ADDED Requirements

### Requirement: One ingress, and no path around it

Every external request SHALL reach the application through the API gateway. The backing service
SHALL grant invocation rights to the gateway's identity and to no other caller, so that a
direct call to the service address is refused rather than merely discouraged.

*Source: reference spec §5 — the frontend reaches the API backend and nothing else. Enforcing
this at the invocation grant rather than by convention means there is no code path to audit.*

#### Scenario: Request through the gateway

- **WHEN** a client sends a request to the gateway address
- **THEN** the request reaches the application

#### Scenario: Direct call to the backing service

- **WHEN** a caller other than the gateway invokes the backing service address directly
- **THEN** the invocation is refused

#### Scenario: Health verification path

- **WHEN** the pipeline verifies a deployment is serving traffic
- **THEN** it verifies through the gateway rather than around it

### Requirement: Ingress contract generated from infrastructure

The routes and backend bindings the gateway exposes SHALL be generated from the infrastructure
definitions, so that the deployed ingress configuration cannot diverge from the definitions
that produced it.

#### Scenario: Route added

- **WHEN** a route is added to the infrastructure definitions
- **THEN** the deployed gateway exposes it after the next apply, with no separate hand-edited
  configuration step

#### Scenario: Hand-edited ingress configuration

- **WHEN** the deployed ingress configuration is changed outside the definitions
- **THEN** a plan run reports the difference

### Requirement: Transport security at the edge

The gateway SHALL accept only TLS-encrypted connections.

#### Scenario: Plaintext request to the gateway

- **WHEN** a client attempts a non-TLS connection to the gateway
- **THEN** the connection is refused or redirected to TLS

### Requirement: Ingress-level request correlation

Every request arriving through the gateway SHALL carry a correlation identifier by the time it
reaches the application, generated at the edge when the client did not supply one, and echoed
on the response.

*Source: reference spec §26 Audit Correlation. Assigning the identifier at the edge means every
request has one, including those that fail before reaching application code.*

#### Scenario: Client supplies no correlation identifier

- **WHEN** a request arrives without a correlation identifier
- **THEN** one is generated and carried through the request
- **AND** it is present on the response

#### Scenario: Request fails at the edge

- **WHEN** a request is rejected before reaching application code
- **THEN** the rejection is still traceable by correlation identifier

### Requirement: Every exposed endpoint is classified

Every route the gateway exposes SHALL be classified as to whether it requires authentication
and what permission it demands. An endpoint that declares nothing SHALL be treated as denying
everything.

*The declarative permission mechanism itself is owned by `access-control-and-admin`
(`TS-BL-018`). This requirement is the ingress-side obligation that no route escapes
classification when that mechanism arrives.*

*Residual scope as of Sprint 0: the health, build-information, and two diagnostic routes carry
no declarative permission requirement, because the mechanism does not yet exist. Diagnostics
are restricted to non-production environments in the meantime, and the eventual endpoint
enumeration must classify these four rather than skip them.*

#### Scenario: Endpoint enumeration

- **WHEN** the exposed routes are enumerated
- **THEN** each is classified as public, authenticated, or permission-gated, with none
  unclassified

#### Scenario: Endpoint declares no permission

- **WHEN** an authenticated route declares no required permission
- **THEN** access is denied rather than allowed by default

#### Scenario: Diagnostic route in production

- **WHEN** a diagnostic route is requested in a production environment
- **THEN** it is unavailable

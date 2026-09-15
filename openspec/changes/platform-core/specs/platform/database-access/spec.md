## Purpose

How TalentSphere's services reach their relational database: privately, without a stored
credential, and under identities separated so that least-privilege grants on append-only tables
are enforceable rather than decorative. Owned by `TS-BL-002`.

## ADDED Requirements

### Requirement: The database has no password

Services SHALL authenticate to the database as identity-managed users holding short-lived
tokens. No database password SHALL exist in source, in infrastructure state, in the secret
manager, or on a developer machine.

*Source: `design.md` D18, added during Sprint 0. Sprint 0 first shipped a generated password in
Secret Manager, which satisfied `DEP-003` — the secret was in an approved manager and injected
at runtime — but created a credential that exists, sits in state, and must be rotated. Removing
the credential is the stronger form of the same requirement.*

#### Scenario: Credential search across all stores

- **WHEN** source, infrastructure state, and the secret manager are searched for a database
  password
- **THEN** none exists

#### Scenario: Token expiry

- **WHEN** a service's database token expires
- **THEN** the service obtains a new short-lived token without operator action

#### Scenario: Secret management remains required elsewhere

- **WHEN** a service needs an external credential that is not the database
- **THEN** it is still injected at runtime from the secret manager

### Requirement: Separated runtime and migration identities

Two distinct database identities SHALL exist. The runtime identity SHALL hold data-manipulation
rights only and SHALL NOT hold schema-definition rights. A separate migration identity SHALL
own the schema and hold schema-definition rights.

*Source: `design.md` D18, cross-referencing D7. D7 makes audit and AI run tables append-only by
granting the runtime `INSERT` and `SELECT` only — a guarantee that is decorative if the same
identity can alter the table. The append-only requirement itself is owned by
`access-control-and-admin`'s audit capability; this requirement is what makes it enforceable.*

#### Scenario: Runtime attempts schema change

- **WHEN** the runtime identity attempts to alter or drop a table
- **THEN** the database refuses the operation

#### Scenario: Append-only narrowing holds

- **WHEN** the runtime identity attempts to update or delete a row in a table granted
  `INSERT` and `SELECT` only
- **THEN** the database refuses the operation

#### Scenario: Migration identity performs schema change

- **WHEN** the migration identity applies a versioned migration
- **THEN** the schema change succeeds

### Requirement: Private network reachability only

The database instance SHALL have no public address and SHALL be reachable only from within the
private network. Connections SHALL be encrypted.

#### Scenario: Connection attempted from outside the private network

- **WHEN** a client outside the private network attempts to connect
- **THEN** the connection fails because no publicly routable address exists

#### Scenario: Unencrypted connection attempted

- **WHEN** a client attempts an unencrypted connection
- **THEN** the connection is refused

### Requirement: One source of connection configuration

The connection URL SHALL be derived in exactly one place, shared by every consumer including
the application, the migration runner, and any background job.

*Source: Sprint 0's split-brain defect — the session module was updated for IAM authentication
and the migration environment module was not. Both were internally correct, so nothing caught
it; migrations kept using the password form and reached a loopback address inside a job.*

#### Scenario: Two consumers compared

- **WHEN** the application's connection configuration and the migration runner's connection
  configuration are compared
- **THEN** they resolve from the same source and agree

#### Scenario: A consumer derives its own URL

- **WHEN** a code path constructs a connection URL independently of the shared source
- **THEN** the test suite fails

### Requirement: Local development parity without identity management

The connection layer SHALL support both password authentication for the Local environment and
identity authentication for deployed environments, selected by configuration rather than by
branching at call sites.

*A developer's local database has no identity management; parity is achieved by configuration,
not by a second code path.*

#### Scenario: Local environment

- **WHEN** the application runs in the Local environment
- **THEN** it connects with local credentials and no identity-management dependency

#### Scenario: Deployed environment

- **WHEN** the application runs in Development or above
- **THEN** it connects with identity authentication and no password

### Requirement: Least-privilege grants are provisioned, not assumed

The grants that narrow each identity's rights SHALL be applied as a controlled, repeatable
step, and SHALL be verified against a real instance rather than only against a local database.

*Residual scope as of Sprint 0: schema ownership is granted by hand through a console, because
no automated identity holds the rights to reassign it. Re-provisioning Development, or standing
up UAT, repeats that step. Automating it requires an administrative identity the pipeline can
assume, which does not yet exist. Recorded as a known issue, not as a satisfied requirement.*

#### Scenario: Environment re-provisioned

- **WHEN** an environment is re-provisioned
- **THEN** the least-privilege grants are re-applied
- **AND** any step still requiring manual action is named in the release's known issues

#### Scenario: Grant verified against the real instance

- **WHEN** a grant narrowing an identity's rights is verified
- **THEN** the verification runs against a real managed instance, not only against a local
  database where the developer is a superuser

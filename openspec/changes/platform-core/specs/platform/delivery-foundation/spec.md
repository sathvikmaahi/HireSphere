## Purpose

The deployment, configuration, and observability baseline every other TalentSphere capability
depends on: infrastructure as code, environment topology, secret handling, migrations, CI
quality gates, release traceability, and the application-layer signals that make the platform
supportable. Owned by `TS-BL-001`.

## ADDED Requirements

### Requirement: Infrastructure as code

Platform infrastructure SHALL be provisioned from version-controlled infrastructure-as-code,
with no manually created production-path resources. Where a provisioning step cannot yet be
automated, it SHALL be recorded as a known issue rather than left undocumented.

*Source: `DEP-001`.*

#### Scenario: Environment rebuilt from source

- **WHEN** an environment is provisioned from the infrastructure definitions
- **THEN** the resulting resources match the definitions
- **AND** any step still requiring manual action is named in the release's known issues

#### Scenario: Drift is detectable

- **WHEN** infrastructure is changed outside the definitions
- **THEN** a plan run reports the difference rather than silently accepting it

### Requirement: Ownership boundary between platform and workload

Project-scoped resources — the project itself, enabled APIs, network attachment, the artifact
repository, the CI identity and its federation binding — SHALL be owned by the shared platform
repository. This repository SHALL own only what runs inside the project. Neither SHALL declare
a resource the other owns.

*Source: `sprint-0-outcome.md` §3; the landing zone was adopted mid-Sprint-0 after this
repository had duplicated its project vending.*

#### Scenario: Workload definitions applied

- **WHEN** the workload definitions are applied
- **THEN** no project-scoped resource is created, modified, or destroyed by them

#### Scenario: State separation

- **WHEN** the workload pipeline identity loads state for a plan
- **THEN** the platform stack's state is not loaded into that plan

### Requirement: Environment topology

The system SHALL define Local, Development, UAT, and Production environments. Environments
that cannot currently be provisioned SHALL still exist as validated infrastructure code that
is never applied, and SHALL be structurally identical to provisioned environments,
parameterized by variables only.

*Source: `design.md` D15 as amended 2026-08-21 — QA was dropped outright, because the GCP
organization has no `fld-qa` and no QA network host, so QA infrastructure would have been
unapplyable in principle rather than merely deferred.*

#### Scenario: Code-only environments validate

- **WHEN** the infrastructure definitions for UAT and Production are validated
- **THEN** validation passes without provisioning any billable resource

#### Scenario: Environments differ only by variables

- **WHEN** two environments' definitions are compared
- **THEN** they differ only in variable values, not in structure

#### Scenario: Data policy per environment

- **WHEN** a non-production environment is used
- **THEN** it holds only synthetic, anonymized, or masked data

### Requirement: Configuration without hardcoded secrets

Each deployable service SHALL take environment-specific configuration, and secrets SHALL be
stored only in an approved secret manager and injected at runtime.

*Source: `DEP-002`, `DEP-003`.*

#### Scenario: Secret in source control

- **WHEN** the CI pipeline scans a change that introduces a credential into source or
  configuration files
- **THEN** the pipeline fails the change

#### Scenario: Runtime secret retrieval

- **WHEN** a service starts
- **THEN** it obtains its secrets from the secret manager rather than from committed
  configuration

### Requirement: Versioned migrations

Database migrations SHALL be versioned, reversible, peer-reviewed, and executed through
controlled deployment steps. Schema SHALL be created only by migrations, in every environment
including Local. A failed migration SHALL fail the deployment rather than leaving services
running against an unmigrated schema.

*Source: `DEP-004`. The failure-ordering clause comes from Sprint 0, where the migration step
could not reach a private-IP database from shared runners and had to be reworked.*

#### Scenario: Migration in the pipeline

- **WHEN** a change includes a schema migration
- **THEN** the pipeline applies it to a non-production environment as part of validation

#### Scenario: Migration fails during deployment

- **WHEN** a migration fails
- **THEN** the deployment fails
- **AND** no service is left serving traffic against the unmigrated schema

#### Scenario: Unreviewed migration

- **WHEN** a migration reaches the deployment stage without peer review
- **THEN** the deployment is blocked

### Requirement: Deployed artifact provenance

A deployed service SHALL report the commit and image digest it is actually running, and the
pipeline SHALL verify that the running service reports the digest the pipeline just built by
comparing the two sources rather than trusting deployment success.

*Source: Sprint 0's most expensive defect — `ignore_changes` on the service image meant three
days of green pipelines deployed nothing while environment variables updated, so the service
reported a fresh commit while running the first image ever built.*

#### Scenario: Deployed artifact inspected

- **WHEN** a deployed artifact is inspected
- **THEN** it identifies the commit and image digest it was built from

#### Scenario: Deployment silently kept a stale image

- **WHEN** the digest reported by the running service does not match the digest the pipeline
  built
- **THEN** the pipeline fails
- **AND** the mismatch is reported rather than the deployment being recorded as successful

### Requirement: CI quality gates

The CI pipeline SHALL run linting, type checks, unit tests, dependency checks, and security
scanning, and SHALL block merge on failure. A gate whose activation is conditional SHALL fail
the pipeline when its condition cannot be satisfied, rather than skipping silently.

*Source: `NFR-010`. The non-silent-skip clause comes from Sprint 0, where the shared template's
`test:backend` and `security:dependency-scan` were gated on a file this project does not use,
so both would have skipped with 115 tests behind them.*

#### Scenario: Failing gate

- **WHEN** a change fails any required gate
- **THEN** the pipeline blocks the merge

#### Scenario: Gate condition unmet

- **WHEN** a required gate's activation condition is not satisfied
- **THEN** the pipeline fails rather than reporting success with the gate skipped

### Requirement: Shared pipeline template consumption

Deployment machinery SHALL be consumed from the shared pipeline template by pinned version
reference, never by a floating reference, and any local override of a template-provided job
SHALL be explicit.

#### Scenario: Template updated upstream

- **WHEN** the shared template publishes a new version
- **THEN** this project's pipeline behavior is unchanged until its pinned reference is
  deliberately advanced

### Requirement: Release recoverability

Deployments SHALL support rollback or forward-fix through the same controlled pipeline, and
releases SHALL publish notes covering changes, migrations, and known issues. A pipeline stage
that publishes release information SHALL be reachable regardless of the state of any manual
approval gate.

*Source: `DEP-005`. The reachability clause comes from Sprint 0, where release notes sat behind
a blocking manual gate in an earlier stage and could therefore never run.*

#### Scenario: Failed deployment

- **WHEN** a deployment fails validation after release
- **THEN** the previous version can be restored or a forward fix applied through the same
  controlled pipeline

#### Scenario: Release notes published

- **WHEN** a release is deployed
- **THEN** release notes are published identifying the changes, any migrations included, and
  known issues

#### Scenario: Approval gate pending

- **WHEN** a manual approval gate has not been actioned
- **THEN** any artifact an approver needs in order to decide is still retrievable

### Requirement: Feature flags

AI features, external integrations, and phased module rollout SHALL be controllable by feature
flags. Flags SHALL be declared in a closed registry; resolving an undeclared flag SHALL raise
rather than resolve to a default.

#### Scenario: Disabling a capability

- **WHEN** a capability's feature flag is turned off
- **THEN** the capability's entry points are unavailable and the rest of the application
  continues to operate

#### Scenario: Undeclared flag

- **WHEN** code resolves a flag that the registry does not declare
- **THEN** the resolution raises rather than silently returning false

### Requirement: Runtime configurability

The following SHALL be configurable without code changes: session expiration, AI model
provider, model name and version per prompt family, prompt template active version, aging
thresholds, notification templates, export limits, and role and permission matrix values.
Every runtime configuration change SHALL be audited with a mandatory reason, and no unaudited
setter SHALL exist.

*Deployment settings and runtime configuration are distinct: a settings change is a redeploy;
a runtime configuration change is not.*

#### Scenario: Model swapped for one family

- **WHEN** an administrator changes the configured model for a single prompt family
- **THEN** subsequent runs of that family use the new model and other families are unaffected

#### Scenario: Configuration changed without a reason

- **WHEN** a runtime configuration change is submitted with no reason
- **THEN** the change is rejected and no value is altered

### Requirement: Observability signals

The system SHALL emit structured logs carrying a request correlation identifier, metrics, and
distributed traces spanning API requests, workflow actions, database operations, background
jobs, and external integrations. Correlation identifiers SHALL survive into background work.
Alerts SHALL be configured for authentication failure spikes, job failure spikes, queue
backlog, and integration downtime.

*Source: `NFR-007`, reference spec §26.*

#### Scenario: Tracing a request

- **WHEN** an investigator holds a request correlation identifier
- **THEN** the corresponding logs, trace, and any resulting background job can be located

#### Scenario: Structured log content

- **WHEN** a request is served
- **THEN** its log entry carries request identifier, endpoint, action, status, duration, and
  error code where applicable

#### Scenario: Sensitive value in a log field

- **WHEN** a log record would carry a secret, token, or credential-shaped value
- **THEN** the value is redacted before the record is emitted

### Requirement: Secure error handling

User-facing and API errors SHALL be actionable and free of stack traces, secrets, tokens, and
raw prompt content, and API errors SHALL carry a consistent error code, message, trace
identifier, correlation identifier, and field validation detail.

*Source: `SEC` error-handling requirements and `NFR-011`.*

#### Scenario: Unhandled server failure

- **WHEN** an unexpected server error occurs
- **THEN** the client receives a generic message with a trace identifier
- **AND** no stack trace, secret, or prompt content appears in the response

#### Scenario: Rejected input echoed back

- **WHEN** input validation fails
- **THEN** the response names the field and the rule violated
- **AND** does not echo the rejected value

### Requirement: Transport and storage protection

The platform SHALL encrypt data in transit using TLS and SHALL encrypt sensitive data at rest.

#### Scenario: Plaintext transport

- **WHEN** a client attempts a non-TLS connection
- **THEN** the connection is refused or redirected to TLS

### Requirement: Interactive performance baseline

Common authenticated screens SHALL return initial usable data within three seconds under
expected load, excluding long-running AI and parsing jobs. The baseline SHALL be measured
rather than assumed.

*Source: `NFR-001`.*

#### Scenario: Common screen under expected load

- **WHEN** a common authenticated screen is loaded under expected load
- **THEN** initial usable data is returned within three seconds

#### Scenario: Baseline recorded

- **WHEN** a feature's screens are complete
- **THEN** measured load times for each are recorded against the threshold, so a later
  regression is detectable

### Requirement: Frontend delivery

The built frontend SHALL be served to authenticated users over TLS, and SHALL obtain its
environment-specific API address at runtime rather than having it compiled in, so that one
built artifact can be promoted between environments unchanged.

*Residual scope as of Sprint 0: the storage bucket exists and the pipeline syncs into it, but
no edge serves it publicly yet.*

#### Scenario: Artifact promoted between environments

- **WHEN** a frontend artifact built for one environment is promoted to another
- **THEN** it addresses the target environment's API without being rebuilt

#### Scenario: Frontend requested over plaintext

- **WHEN** the frontend is requested without TLS
- **THEN** the request is refused or redirected to TLS

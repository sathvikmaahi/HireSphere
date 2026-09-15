## Purpose

The deployment, configuration, and observability baseline every other capability depends on: infrastructure as code, environment topology, secret handling, migrations, CI quality gates, and the operational signals that make the platform supportable.

## ADDED Requirements

### Requirement: Infrastructure as code

Platform infrastructure SHALL be provisioned from version-controlled infrastructure-as-code, with no manually created production-path resources.

#### Scenario: Environment rebuilt from source

- **WHEN** an environment is provisioned from the infrastructure definitions
- **THEN** the resulting resources match the definitions with no manual configuration step required

#### Scenario: Drift is detectable

- **WHEN** infrastructure is changed outside the definitions
- **THEN** a plan run reports the difference rather than silently accepting it

### Requirement: Environment topology

The system SHALL define Local, Development, UAT, and Production environments. Environments that cannot currently be provisioned SHALL still exist as validated infrastructure code that is never applied.

#### Scenario: Code-only environments validate

- **WHEN** the infrastructure definitions for UAT and Production are validated
- **THEN** validation passes without provisioning any billable resource

#### Scenario: Data policy per environment

- **WHEN** a non-production environment is used
- **THEN** it holds only synthetic, anonymized, or masked data

### Requirement: Configuration without hardcoded secrets

Each deployable service SHALL take environment-specific configuration, and secrets SHALL be stored only in an approved secret manager and injected at runtime.

#### Scenario: Secret in source control

- **WHEN** the CI pipeline scans a change that introduces a credential into source or configuration files
- **THEN** the pipeline fails the change

#### Scenario: Runtime secret retrieval

- **WHEN** a service starts
- **THEN** it obtains its secrets from the secret manager rather than from committed configuration

### Requirement: Versioned migrations

Database migrations SHALL be versioned, peer-reviewed, executed through controlled deployment steps, and applied to a non-production environment before production.

#### Scenario: Migration in the pipeline

- **WHEN** a change includes a schema migration
- **THEN** the pipeline applies it to a non-production environment as part of validation

#### Scenario: Unreviewed migration

- **WHEN** a migration reaches the deployment stage without peer review
- **THEN** the deployment is blocked

### Requirement: Release recoverability

Deployments SHALL support rollback or forward-fix, and releases SHALL publish notes covering changes, migrations, and known issues.

#### Scenario: Failed deployment

- **WHEN** a deployment fails validation after release
- **THEN** the previous version can be restored or a forward fix applied through the same controlled pipeline

#### Scenario: Release notes published

- **WHEN** a release is deployed
- **THEN** release notes are published identifying the changes, any migrations included, and known issues

### Requirement: Independent worker scaling and non-blocking jobs

Background workers SHALL be scalable independently of API services, and long-running parsing or AI jobs SHALL NOT block interactive requests.

#### Scenario: Long job submitted

- **WHEN** a client triggers a long-running operation
- **THEN** the response returns a job identifier and the interactive request completes promptly
- **AND** job status is retrievable from a status endpoint

### Requirement: CI quality gates

The CI pipeline SHALL run linting, type checks, unit tests, dependency checks, and security scanning, and SHALL block merge on failure. Changes SHALL require peer review before merge, and build artifacts SHALL be traceable to a commit.

#### Scenario: Failing tests

- **WHEN** a change fails any required gate
- **THEN** the pipeline blocks the merge

#### Scenario: Artifact traceability

- **WHEN** a deployed artifact is inspected
- **THEN** it identifies the commit it was built from

### Requirement: Feature flags

AI features, external integrations, and phased module rollout SHALL be controllable by feature flags.

#### Scenario: Disabling an AI feature

- **WHEN** an AI feature flag is turned off
- **THEN** the feature's entry points are unavailable and the rest of the application continues to operate

### Requirement: Observability signals

The system SHALL emit structured logs carrying a request correlation identifier, metrics, and distributed traces spanning API requests, workflow actions, database operations, background jobs, and external integrations. Alerts SHALL be configured for authentication failure spikes, job failure spikes, queue backlog, and integration downtime.

#### Scenario: Tracing a request

- **WHEN** an investigator holds a request correlation identifier
- **THEN** the corresponding logs, trace, and any resulting background job can be located

#### Scenario: Structured log content

- **WHEN** a request is served
- **THEN** its log entry carries request identifier, endpoint, action, status, duration, and error code where applicable

### Requirement: Runtime configurability

The following SHALL be configurable without code changes: session expiration, AI model provider, model name and version per prompt family, prompt template active version, aging thresholds, notification templates, export limits, and role and permission matrix values.

#### Scenario: Model swapped for one family

- **WHEN** an administrator changes the configured model for a single prompt family
- **THEN** subsequent runs of that family use the new model and other families are unaffected

### Requirement: Transport and storage protection

The platform SHALL encrypt data in transit using TLS and SHALL encrypt sensitive data at rest.

#### Scenario: Plaintext transport

- **WHEN** a client attempts a non-TLS connection
- **THEN** the connection is refused or redirected to TLS

### Requirement: Interactive performance baseline

Common authenticated screens SHALL return initial usable data within three seconds under expected load, excluding long-running AI and parsing jobs. The baseline SHALL be measured rather than assumed.

#### Scenario: Common screen under expected load

- **WHEN** a common authenticated screen is loaded under expected load
- **THEN** initial usable data is returned within three seconds

#### Scenario: Long-running job excluded

- **WHEN** a screen triggers a long-running background job
- **THEN** the screen itself still returns within the threshold, with job progress reported asynchronously

#### Scenario: Baseline recorded

- **WHEN** the wave's screens are complete
- **THEN** measured load times for each are recorded against the threshold, so a later regression is detectable

### Requirement: Secure error handling

User-facing and API errors SHALL be actionable and free of stack traces, secrets, tokens, and raw prompt content, and API errors SHALL carry a consistent error code, message, trace identifier, and field validation detail.

#### Scenario: Unhandled server failure

- **WHEN** an unexpected server error occurs
- **THEN** the client receives a generic message with a trace identifier
- **AND** no stack trace, secret, or prompt content appears in the response

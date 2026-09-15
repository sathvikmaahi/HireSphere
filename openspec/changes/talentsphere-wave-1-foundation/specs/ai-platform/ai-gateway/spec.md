## Purpose

The single governed path through which every AI call in TalentSphere is made: per-family model configuration, safety filtering, rate limiting, untrusted-content isolation, and degradation behavior when the provider is unavailable.

## ADDED Requirements

### Requirement: All AI calls route through the gateway

Every AI invocation SHALL be issued through the AI gateway. No feature SHALL call a model provider directly.

#### Scenario: Feature requests an AI capability

- **WHEN** any feature requires model output
- **THEN** the request passes through the gateway, which resolves the prompt template, model configuration, and logging before invoking the provider

#### Scenario: Provider credentials are not feature-accessible

- **WHEN** feature code is inspected
- **THEN** model provider credentials are reachable only by the gateway

### Requirement: Per-family model configuration

The gateway SHALL resolve model provider, model name, and model version per prompt family from configuration.

#### Scenario: Two families on different models

- **WHEN** two prompt families are configured with different models
- **THEN** each family's runs use its own configured model, and each run records which was used

#### Scenario: Unconfigured family

- **WHEN** a run is requested for a family with no model configuration
- **THEN** the run fails with a configuration error and is logged as failed rather than falling back to an arbitrary model

### Requirement: Untrusted content isolation

The gateway SHALL separate trusted system instructions from untrusted document-derived content, SHALL sanitize document-derived content before including it in a prompt, and SHALL restrict tool or action capability when processing candidate documents.

#### Scenario: Document content carrying instructions

- **WHEN** document-derived content contains text that reads as an instruction
- **THEN** it is passed as delimited data and does not alter the system instruction
- **AND** the run completes without the injected instruction taking effect

#### Scenario: Tool capability during document processing

- **WHEN** a run processes candidate document content
- **THEN** no tool or action capability beyond producing the contracted output is available to the model

### Requirement: Personal data minimization

The gateway SHALL exclude candidate contact information from AI input unless a controlled exception is approved and logged, and SHALL provide protected-attribute redaction for inputs used in scoring.

#### Scenario: Contact fields excluded

- **WHEN** a run is prepared from a candidate record
- **THEN** contact fields are omitted from the prompt payload
- **AND** the run log records that exclusion applied

#### Scenario: Approved exception

- **WHEN** an approved exception permits contact information in a specific run type
- **THEN** the inclusion is recorded on the run so it can be audited

### Requirement: Advisory-only invocation

The gateway SHALL NOT be capable of causing a state transition. Model output SHALL be persisted as advisory insight or draft content only, and SHALL never itself reject, shortlist, select, offer, hire, onboard, or close.

#### Scenario: Output cannot transition state

- **WHEN** a run completes with output that recommends a decision
- **THEN** no state transition occurs as a result
- **AND** any state change requires a separate, permission-checked action by an accountable user

### Requirement: Rate limiting and consumption bounds

The gateway SHALL enforce rate limits and bounded consumption per caller and per family.

#### Scenario: Caller exceeds its rate limit

- **WHEN** a caller exceeds the configured rate limit
- **THEN** further runs are rejected or queued rather than issued to the provider

#### Scenario: Oversized input

- **WHEN** an input exceeds the configured size bound for its family
- **THEN** the run is rejected with a clear error before provider invocation

### Requirement: Graceful degradation

An AI provider outage SHALL NOT prevent users from viewing existing records or performing non-AI workflow actions.

#### Scenario: Provider unavailable

- **WHEN** the AI provider is unavailable
- **THEN** AI-triggering controls report the capability as temporarily unavailable
- **AND** all non-AI pages, records, and workflow actions remain fully usable

#### Scenario: Partial batch failure

- **WHEN** a batch of runs partially fails
- **THEN** per-item success and failure are reported rather than the whole batch being reported as failed

### Requirement: Failure preservation and safe retry

Failed runs SHALL preserve failure detail and SHALL support retry where the operation is safe to repeat, without corrupting workflow state.

#### Scenario: Transient provider failure

- **WHEN** a run fails transiently
- **THEN** the failure detail is preserved on the run record and the run is retryable

#### Scenario: Failure during a workflow action

- **WHEN** an AI run fails while a user is mid-workflow
- **THEN** the workflow state is unchanged and the user is told the AI step failed

### Requirement: Asynchronous execution

AI runs SHALL execute asynchronously, returning a job identifier with a retrievable status.

#### Scenario: Run triggered

- **WHEN** an authorized user triggers an AI run
- **THEN** the response returns a job identifier immediately
- **AND** progress and completion are retrievable from a status endpoint

### Requirement: Run AI permission enforced at the gateway

The gateway SHALL verify that the requesting actor holds the Run AI permission for the relevant page before invoking a provider.

#### Scenario: Caller lacks Run AI

- **WHEN** a user without Run AI permission triggers a run
- **THEN** the request is denied and no provider invocation occurs

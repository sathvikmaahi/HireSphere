## Purpose

The single governed path through which every AI call in TalentSphere is made: per-family model
configuration, permission verification, untrusted-content isolation, personal-data minimization,
rate limiting, asynchronous execution, and degradation behavior when the provider is unavailable.
It is also the only place in the product that knows which model provider is in use.

## ADDED Requirements

### Requirement: All AI calls route through the gateway

Every AI invocation SHALL be issued through the AI gateway. No feature SHALL call a model provider
directly.

*Source: `AI-003`, `domain-model.md`'s "AI platform" section, and inherited `D9` — the gateway is
the *only* egress to a model provider. Every governance guarantee in this feature is enforced at
one chokepoint, so a second egress would not weaken the guarantees, it would remove them.*

#### Scenario: Feature requests an AI capability

- **WHEN** any feature requires model output
- **THEN** the request passes through the gateway, which resolves the prompt template, model
  configuration, and logging before invoking the provider

#### Scenario: Provider credentials are not feature-accessible

- **WHEN** feature code is inspected
- **THEN** model provider credentials are reachable only by the gateway

#### Scenario: A second egress is detectable

- **WHEN** the codebase is enumerated for outbound calls to a model provider
- **THEN** exactly one call site exists, and a test fails if a second appears

### Requirement: Provider independence

The gateway SHALL address the model provider through a single adapter boundary. Substituting the
provider SHALL require no change to any calling feature, any prompt template contract, or the
shape of any run record. The gateway SHALL operate against a deterministic stub provider in
environments where no real provider is configured.

*Source: `OD-003`, which is unresolved and owned by AI Engineering; `config.yaml`'s standing rule
that "unconfirmed external contracts sit behind adapters: Hubble login (`OD-001`) and the AI
provider (`OD-003`)". This mirrors `identity-and-access`'s treatment of the unconfirmed Hubble
contract, where the mock is a first-class artifact rather than a test fixture, because everything
downstream keeps depending on it until the real contract lands. Note that `D09` settled **Vertex
AI Vector Search for retrieval**; it did not choose a generation provider, and no decision
anywhere in `exploration-notes.md` does.*

#### Scenario: Provider substituted

- **WHEN** the configured provider adapter is replaced
- **THEN** no calling feature, prompt contract, or run record schema changes
- **AND** existing run records remain interpretable, because each names the provider it used

#### Scenario: No provider configured

- **WHEN** an environment has no real provider configured
- **THEN** the gateway serves runs through the stub provider
- **AND** the run record identifies the stub as the provider used, rather than recording a real
  provider name

#### Scenario: Provider identity is never assumed

- **WHEN** a run record is written
- **THEN** provider, model name, and model version are read from the resolved configuration rather
  than defaulted in code

### Requirement: Uniform request and response envelope

The gateway SHALL accept every invocation in one envelope — prompt family, typed input payload,
and calling context — and SHALL return one envelope: a run reference, a status, and either
contract-satisfying output or failure detail. The envelope SHALL NOT vary by family.

*Source: inherited `D9`'s central-validation argument, extended to the request side. Six items
across four not-yet-proposed features will call this gateway; a per-family request shape would
make the gateway's contract as inconsistent as its callers, and would have to be renegotiated by
each new consumer. `design.md` D3 records the concrete future call shape the envelope was
designed against.*

#### Scenario: Two families, one envelope

- **WHEN** two different prompt families are invoked
- **THEN** both requests and both responses use the same envelope structure, differing only in the
  typed payload and the family named

#### Scenario: Caller receives a run reference regardless of outcome

- **WHEN** an invocation succeeds, fails validation, or fails at the provider
- **THEN** the response carries a run reference in every case, so the caller can retrieve the run
  record for any outcome

### Requirement: Per-family model configuration

The gateway SHALL resolve model provider, model name, and model version per prompt family from
configuration.

*Source: `AI-002`, `AI-004`; `reference/spec.md` §36's mitigation for AI cost and latency, which
names "model selection by task" — that is only possible if configuration is per family.*

#### Scenario: Two families on different models

- **WHEN** two prompt families are configured with different models
- **THEN** each family's runs use its own configured model, and each run records which was used

#### Scenario: Unconfigured family

- **WHEN** a run is requested for a family with no model configuration
- **THEN** the run fails with a configuration error and is logged as failed rather than falling
  back to an arbitrary model

#### Scenario: Model configuration changes without a deployment

- **WHEN** the configured model for a family is changed through the audited configuration path
- **THEN** subsequent runs of that family use it, and the change is audited with previous and new
  value

### Requirement: Untrusted content isolation

The gateway SHALL separate trusted system instructions from untrusted document-derived content,
SHALL sanitize document-derived content before including it in a prompt, and SHALL restrict tool
or action capability when processing candidate documents.

*Source: `AI-012`, `AI-013`, `AI-014`; `reference/spec.md` §36 names prompt injection from resumes
as a top risk. Resumes are attacker-controllable content submitted by the person the output
assesses, which is the strongest injection incentive in the product.*

#### Scenario: Document content carrying instructions

- **WHEN** document-derived content contains text that reads as an instruction
- **THEN** it is passed as delimited data and does not alter the system instruction
- **AND** the run completes without the injected instruction taking effect

#### Scenario: Tool capability during document processing

- **WHEN** a run processes candidate document content
- **THEN** no tool or action capability beyond producing the contracted output is available to the
  model

### Requirement: Personal data minimization

The gateway SHALL exclude candidate contact information from AI input unless a controlled
exception is approved and logged, and SHALL provide protected-attribute redaction for inputs used
in scoring.

*Source: `AI-005`, `AI-006`; §16.4's prohibited ranking signals. Redaction at the gateway rather
than at each caller means a new consumer inherits it rather than re-implementing it, which is the
same argument `platform-core` D4 makes about two independently-correct implementations of one
rule.*

#### Scenario: Contact fields excluded

- **WHEN** a run is prepared from a candidate record
- **THEN** contact fields are omitted from the prompt payload
- **AND** the run log records that exclusion applied

#### Scenario: Approved exception

- **WHEN** an approved exception permits contact information in a specific run type
- **THEN** the inclusion is recorded on the run so it can be audited

#### Scenario: Protected attributes in scoring input

- **WHEN** input destined for a scoring family contains a protected or sensitive attribute
- **THEN** the attribute is redacted before the prompt is composed

### Requirement: Rate limiting and consumption bounds

The gateway SHALL enforce rate limits and bounded consumption per caller and per family.

*Source: `reference/spec.md` §36's excessive-cost-and-latency risk, whose mitigation names rate
limits and batch controls. `D07` sets the volume expectation low — fewer than 5,000 candidates and
30 postings — which makes an unbounded batch a plausible accident rather than a load concern.*

#### Scenario: Caller exceeds its rate limit

- **WHEN** a caller exceeds the configured rate limit
- **THEN** further runs are rejected or queued rather than issued to the provider

#### Scenario: Oversized input

- **WHEN** an input exceeds the configured size bound for its family
- **THEN** the run is rejected with a clear error before provider invocation

### Requirement: Graceful degradation

An AI provider outage SHALL NOT prevent users from viewing existing records or performing non-AI
workflow actions.

*Source: `G-12`; `NFR`-level graceful degradation. This is the operational face of advisory-only:
if the product stopped working when the AI did, the AI would be load-bearing in fact whatever the
permission model said.*

#### Scenario: Provider unavailable

- **WHEN** the AI provider is unavailable
- **THEN** AI-triggering controls report the capability as temporarily unavailable
- **AND** all non-AI pages, records, and workflow actions remain fully usable

#### Scenario: Partial batch failure

- **WHEN** a batch of runs partially fails
- **THEN** per-item success and failure are reported rather than the whole batch being reported as
  failed

### Requirement: Failure preservation and safe retry

Failed runs SHALL preserve failure detail and SHALL support retry where the operation is safe to
repeat, without corrupting workflow state.

*Source: `AI-002`'s status and failure fields; `platform-core`'s async substrate requires
idempotency at the effect boundary because Pub/Sub delivers at least once, so a retried AI run
must not produce a second insight record for the same request.*

#### Scenario: Transient provider failure

- **WHEN** a run fails transiently
- **THEN** the failure detail is preserved on the run record and the run is retryable

#### Scenario: Failure during a workflow action

- **WHEN** an AI run fails while a user is mid-workflow
- **THEN** the workflow state is unchanged and the user is told the AI step failed

#### Scenario: Duplicate delivery of the same run request

- **WHEN** the same run request is delivered more than once by the dispatch substrate
- **THEN** exactly one run is issued to the provider and exactly one output record results

### Requirement: Asynchronous execution

AI runs SHALL execute asynchronously, returning a job identifier with a retrievable status.

*Source: `reference/spec.md` §25, which lists AI ranking among the background job types;
`platform-core`'s `TS-BL-006` requires one dispatch pattern, with a scenario that fails the suite
on background work started outside it. AI runs register as a job type against that pattern rather
than growing a second queue.*

#### Scenario: Run triggered

- **WHEN** an authorized user triggers an AI run
- **THEN** the response returns a job identifier immediately
- **AND** progress and completion are retrievable from a status endpoint

#### Scenario: Dispatch uses the shared pattern

- **WHEN** an AI run is dispatched for background execution
- **THEN** it is dispatched as a registered job type through the platform's single dispatch
  pattern, not through a queue owned by this feature

### Requirement: Run AI permission enforced at the gateway

The gateway SHALL verify that the requesting actor holds the Run AI permission for the relevant
page before invoking a provider.

*Source: `AUTHZ-003`, §9.3's nine action flags, and `C-02`'s note that a distinct `Run AI` action
is what makes triggering AI separately controllable from viewing its output.
`access-control-and-admin`'s `TS-BL-018` supplies the evaluator; this requirement is the gateway
consuming it, not a second evaluation.*

#### Scenario: Caller lacks Run AI

- **WHEN** a user without Run AI permission triggers a run
- **THEN** the request is denied and no provider invocation occurs

#### Scenario: Verdict comes from the evaluator

- **WHEN** the gateway checks Run AI
- **THEN** the verdict is produced by the central permission evaluator rather than by logic local
  to the gateway

#### Scenario: Background run under the service account

- **WHEN** a background AI run executes under the AI Service Account
- **THEN** its permission is evaluated by the same evaluator, on the same terms as a human caller

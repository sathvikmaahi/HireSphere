## Purpose

The interpretive half of hybrid parsing: skills, seniority, role summaries and domain experience,
produced by the one governed model egress from a reference to a resume version rather than from
resume content, with every claim labeled, evidenced and bounded. Owned by `TS-BL-044`.

## ADDED Requirements

### Requirement: Enrichment is invoked through the gateway in the fixed envelope, carrying a resume version reference

Enrichment SHALL be requested through the AI gateway using the uniform request envelope, with the
input payload carrying a resume version reference as identifier-and-version. The request SHALL NOT
carry resume content.

*Source: `ai-platform-governance`'s `ai-platform/ai-gateway` uniform-envelope requirement and its
`design.md` D3, which fixed the envelope against a different family and generalized it to this one by
name: "`resume_extraction` sends a resume version reference." `design.md` D3 here.*

#### Scenario: Enrichment requested

- **WHEN** enrichment is requested for a parsed resume
- **THEN** the request carries the family, the calling context, and a resume version reference, in
  the same envelope every other family uses

#### Scenario: Request payload inspected

- **WHEN** an enrichment request payload is inspected
- **THEN** it contains no resume text, no extracted contact field, and no candidate personal data

#### Scenario: Reference resolves to a specific version

- **WHEN** a later resume version exists for the same candidate
- **THEN** the reference recorded on a completed enrichment still resolves to the version it was run
  against

### Requirement: No provider is called from this capability, and no second dispatch path is built

Every model invocation SHALL go through the gateway. This capability SHALL NOT contain an outbound
call to a model provider, and SHALL NOT register its own background job type for the enrichment call.

*Source: `config.yaml`'s rule that the gateway is the only egress; `ai-platform-governance`
`design.md` D10 — "AI runs register as a job type; they do not get a queue" — under which every
gateway call already dispatches asynchronously inside the gateway. `design.md` D2.*

#### Scenario: Outbound provider call

- **WHEN** this capability's code is inspected for outbound calls to a model provider
- **THEN** none exists

#### Scenario: Job types enumerated

- **WHEN** the job types this feature registers are enumerated
- **THEN** the enrichment call is not among them, because the gateway dispatches it

#### Scenario: Enrichment runs asynchronously

- **WHEN** enrichment is requested
- **THEN** it executes asynchronously with a retrievable status, without this capability
  implementing dispatch

### Requirement: The prompt input excludes candidate contact information

The content the resolved reference supplies to the prompt SHALL exclude the structured contact fields
— email, phone and name — that deterministic extraction already recovered. The controlled exception
permitting candidate contact information in a prompt SHALL NOT be invoked by this family.

*Source: `PRV-001`, `PRV-002`, `CAN-008`, `AI-006`; `design.md` D3 point 2 and D10. Enrichment
produces skills, seniority, role summaries and domain experience, none of which requires a contact
field, and a model that can see a name can produce a claim keyed to it.*

#### Scenario: Resolved prompt input inspected

- **WHEN** the content passed to the prompt for this family is inspected
- **THEN** the structured email, phone and name fields are absent

#### Scenario: Controlled exception invoked

- **WHEN** this family attempts to invoke the controlled exception for candidate contact information
- **THEN** it is refused, and the refusal is asserted rather than left to configuration

### Requirement: Resume content is marked untrusted at the boundary

The input this capability supplies SHALL be marked as untrusted document-derived content, so that the
gateway's separation of trusted instructions from untrusted content applies to it.

*Source: `AI-012`, `AI-013`, `AI-014`, `SEC-008`, `SEC-014`; `D18` — resume text "must be delimited
and treated as data, never as instructions"; `reference/spec.md` §36's prompt-injection risk. The
isolation mechanism is the gateway's; what this capability owes is marking the input so it applies.*

#### Scenario: Input marked

- **WHEN** enrichment input is handed to the gateway
- **THEN** it is marked as untrusted document-derived content

#### Scenario: Instruction-shaped resume content

- **WHEN** a resume contains text shaped as an instruction to the model
- **THEN** the output contract is still enforced, the run is recorded, and no instruction in the
  document alters the family, the caller's permissions, or the output contract

### Requirement: Every enriched claim carries a source label and an evidence reference

Each claim in enrichment output SHALL carry a source label drawn from the closed evidence-source
vocabulary, and a reference to the resume section it was derived from. The label for this family SHALL
be the resume source value.

*Source: `G-02`, `AI-007`; `ai-platform-governance`'s `ai-platform/evidence-labeling` capability,
whose vocabulary this consumes; `DOC-008`'s traceable sections, which are what a reference points at.
`design.md` D4 — this is the first family producing claims about a candidate, so the contract clause
`ai-platform-governance` D3 deliberately did not exercise applies here.*

#### Scenario: Claim inspected

- **WHEN** an enriched skill or seniority claim is inspected
- **THEN** it carries a source label and a resolvable reference to the section it came from

#### Scenario: Claim with no evidence reference

- **WHEN** output contains a claim with no evidence reference
- **THEN** the output is rejected as a contract violation and recorded as a failed run

#### Scenario: Label outside the vocabulary

- **WHEN** output carries a source label outside the closed vocabulary, or a label other than the
  resume source value for this family
- **THEN** the output is rejected as a contract violation

### Requirement: Missing evidence produces an insufficiency marker, not a value

Where the resume does not support a field, the output SHALL carry an insufficiency marker for that
field rather than an inferred value.

*Source: `G-03`, `AI-008`, `DOC-009`.*

#### Scenario: Resume with no stated seniority

- **WHEN** a resume contains no evidence of seniority
- **THEN** the output marks seniority insufficient rather than inferring one

#### Scenario: Insufficiency distinguishable from absence

- **WHEN** an enriched record is read
- **THEN** a field marked insufficient is distinguishable from a field that was never requested

### Requirement: Every free-text enrichment field carries a declared bound

Each free-text field in this family's output contract SHALL declare a machine-checkable length bound.
Output exceeding a bound SHALL be recorded as a failed run and SHALL NOT be persisted as content.
Bounds SHALL be held in audited configuration with machine-readable provenance marking them
provisional.

*Source: `AGENTS.md`'s standing product-quality bar; `S.5`; `ai-platform-governance` `design.md` D7,
whose three enforcement points this consumes rather than reimplements.*

#### Scenario: Over-long role summary

- **WHEN** the model returns a role summary exceeding its declared bound
- **THEN** the run is recorded as failed with the violation preserved and no content is persisted

#### Scenario: Field with no bound

- **WHEN** this family's contract declares a free-text field with no bound
- **THEN** registration is refused

#### Scenario: Bound changed

- **WHEN** an authorized administrator changes a bound
- **THEN** the change is an audited configuration change with no deployment

### Requirement: Enrichment output is advisory, marked, and approves nothing

Enrichment output SHALL persist only as advisory insight carrying its AI-generated marking until a
recorded human approval, SHALL advance no workflow state, and SHALL set no field that gates a
decision.

*Source: `AI-001`, `AI-010`, `UI-004`, `project.md`'s central rule;
`ai-platform-governance`'s `ai-platform/advisory-only` capability, which this consumes.*

#### Scenario: Enrichment completes

- **WHEN** enrichment completes successfully
- **THEN** its output is stored as advisory insight marked AI-generated, and no candidate,
  application or posting state has changed

#### Scenario: Human approves enrichment

- **WHEN** an authorized human approves enriched content
- **THEN** the marking clears for that version only, recording actor and timestamp

#### Scenario: Enrichment attempts a transition

- **WHEN** a persistence path from enrichment output to a workflow-governed state field is sought
- **THEN** none exists

### Requirement: Enrichment failure leaves the candidate intact and is retryable

Where enrichment fails, the candidate and its resume SHALL remain valid and usable, the failure SHALL
be recorded with its typed kind, and retry SHALL be available. A run reference SHALL be returned on
every outcome including failure.

*Source: `D18` — "LLM enrichment fails → the candidate still exists, just unenriched → **retryable**,
no blocking"; `ERR-004`, `NFR-006`; the gateway's failure-preservation requirement.*

#### Scenario: Provider failure

- **WHEN** enrichment fails because the provider is unavailable
- **THEN** the candidate and resume remain valid, the failure kind is reported, and retry is offered

#### Scenario: Run reference on failure

- **WHEN** enrichment fails for any reason
- **THEN** the response carries a run reference so the run can be retrieved

#### Scenario: Unenriched candidate

- **WHEN** a candidate has never been enriched
- **THEN** it is a valid readable candidate whose unenriched state is explicit rather than
  indistinguishable from an empty enrichment

### Requirement: Enrichment requires the Run AI action and degrades gracefully

Requesting enrichment SHALL require the Run AI action on the intake surface, distinct from view and
edit. With the model provider unavailable, every non-AI intake action SHALL remain fully usable.

*Source: §9.3, `C-02`, `AI-005`; `G-12`, `NFR-004`, `DEP-008`.*

#### Scenario: Actor without Run AI

- **WHEN** a user lacking the Run AI action views the intake surface
- **THEN** the enrichment control is absent, and a request submitted directly is refused server-side

#### Scenario: Provider unavailable

- **WHEN** the model provider is unavailable
- **THEN** upload, validation, deterministic extraction, manual entry, identity resolution and merge
  all continue to work, and the enrichment control reports temporary unavailability

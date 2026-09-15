## Purpose

AI-assisted drafting of a job description: the `job_description_generation` prompt content and output
contract, invoked through the governed AI gateway using the envelope that feature already fixed, with
output that is marked as AI-generated until a human approves it and that can never approve itself.
Owned by `TS-BL-034`.

## ADDED Requirements

### Requirement: Generation is invoked through the AI gateway

Job description generation SHALL be invoked through the AI gateway. This capability SHALL NOT call a
model provider, resolve a model, or write a run record itself.

*Source: `AI-003`; `ai-platform-governance`'s gateway requirement that every AI invocation is issued
through it and no feature calls a provider directly. This capability is the product's first real
gateway consumer.*

#### Scenario: Practice Manager runs AI on a draft

- **WHEN** an authorized Practice Manager triggers AI drafting from the JD Workspace
- **THEN** the request is issued through the gateway
- **AND** a run record exists before the provider is invoked

#### Scenario: No second egress

- **WHEN** this capability's code is enumerated for outbound calls to a model provider
- **THEN** none exists

### Requirement: The fixed request envelope, with a reference rather than content

The request SHALL carry the prompt family, the calling context — actor, page, action and correlation
identifier — and a typed input of a job description draft reference in identifier-and-version form
plus optional bounded notes. The response SHALL carry a run reference, a status, and either
contract-satisfying output or typed failure detail. The request SHALL NOT carry a copy of the draft's
content.

*Source: `ai-platform-governance` `design.md` D3, which fixed this envelope against this exact item
and recorded that references rather than content is what makes the run log's reference-only
requirement natural rather than a redaction step. The payload inside the envelope is this feature's,
which D3 states explicitly. See `design.md` D3.*

#### Scenario: Draft addressed by reference

- **WHEN** a generation request is composed
- **THEN** it names the draft as an identifier and version
- **AND** no job description content appears in the request envelope

#### Scenario: Run reference returned on every outcome

- **WHEN** a generation succeeds, fails contract validation, or fails at the provider
- **THEN** the response carries a run reference in every case

#### Scenario: Notes exceed their bound

- **WHEN** notes longer than the declared bound are supplied
- **THEN** the request is rejected before invocation with a field-level error

#### Scenario: Typed failure reaches the user

- **WHEN** a run fails
- **THEN** the failure kind — validation, provider, or configuration — is reported to the user
- **AND** the draft is unchanged

### Requirement: The output contract returns structured content, not a document

The `job_description_generation` contract SHALL return the structured job description object.
Markdown SHALL be rendered from that object by the application rather than authored by the model.
Every free-text field in the output SHALL be validated against the same bound the schema declares for
it.

*Source: §16.3 requires "Markdown plus structured JSON summary"; `ai-platform-governance` `design.md`
D3 named the tension that a single bound over a whole document is either vacuous or impossible, and
resolved the form of the answer without a JD schema to resolve it against. `design.md` D4 resolves it
here, and records that this answer is specific to a document whose sections are fixed by `JOB-002`.*

#### Scenario: Output exceeds a field bound

- **WHEN** model output exceeds the bound declared for one of its free-text fields
- **THEN** the run is recorded as a failed run with the contract violation preserved
- **AND** no content is written to the draft

#### Scenario: Markdown is derived

- **WHEN** markdown for a generated job description is produced
- **THEN** it is rendered from the validated structured object
- **AND** its content cannot differ from the fields that were validated

#### Scenario: Output satisfies the schema

- **WHEN** a run succeeds
- **THEN** the output validates against the same structured schema a human-typed draft validates
  against

### Requirement: Generated content is marked and never self-approving

Generated content SHALL be marked as AI-generated and SHALL retain that marking until a human
approval is recorded with actor and timestamp. Generation SHALL NOT advance a job description's
approval state.

*Source: `JOB-005`, `AI-001`, `UI-004`, `AI-010`; `project.md`'s central rule. `ai-platform-governance`'s
advisory-only capability establishes structurally that gateway output holds no transition capability;
this requirement is the JD-side statement of what that means here.*

#### Scenario: Draft after generation

- **WHEN** generation completes successfully
- **THEN** the draft remains in its pre-generation approval state
- **AND** its content is labeled AI-generated in every surface that renders it

#### Scenario: Human edits generated content

- **WHEN** a Practice Manager edits generated content
- **THEN** the edited content is retained separately from the generated content
- **AND** the marking reflects AI-assisted rather than discarding the provenance

#### Scenario: Marking survives until approval

- **WHEN** an approval is recorded by a human with actor and timestamp
- **THEN** the marking is cleared for that version only

### Requirement: Run AI permission and its own action flag

Triggering generation SHALL require the Run AI action on the JD Workspace page. Holding view or edit
permission SHALL NOT confer it.

*Source: §9.3's nine action flags; `C-02`'s note that a distinct Run AI action is what makes
triggering AI separately controllable from viewing its output; `ai-platform-governance`'s requirement
that the gateway verifies Run AI before invoking a provider.*

#### Scenario: Editor without Run AI

- **WHEN** a user who may edit a job description but does not hold Run AI triggers generation
- **THEN** the request is denied and no provider invocation occurs

#### Scenario: Control absent without the permission

- **WHEN** the JD Workspace renders for a user without Run AI
- **THEN** the generation control is absent

### Requirement: Generation degrades without blocking drafting

An AI provider outage SHALL NOT prevent creating, editing, saving, submitting or approving a job
description.

*Source: `G-12`; `ai-platform-governance`'s graceful-degradation requirement. Manual drafting is
`TS-BL-033` and exists independently — an outage returns the workspace to the state it shipped in.*

#### Scenario: Provider unavailable

- **WHEN** the AI provider is unavailable
- **THEN** the generation control reports the capability as temporarily unavailable
- **AND** every non-AI job description action remains fully usable

### Requirement: The template version is promoted, never edited in place

The prompt content SHALL be introduced as a new version of the already-registered
`job_description_generation` family, activated through the audited configuration path. It SHALL NOT
modify an existing template version.

*Source: `ai-platform-governance`'s prompt-registry requirement that a content change creates a new
version rather than modifying one, and its promotion gate refusing activation in production without a
recorded passing corpus run. `TS-BL-030` registered this family with stub text on purpose; Phase 2 is
where two of the ten are authored. See `design.md` D3.*

#### Scenario: New version created

- **WHEN** the authored prompt content is introduced
- **THEN** a new template version is created and the prior version remains retrievable

#### Scenario: Activation without a passing corpus run

- **WHEN** activation of the new version in production is attempted with no passing corpus run
  recorded for it
- **THEN** the registry refuses activation

#### Scenario: A past run still names its version

- **WHEN** a run recorded against an earlier template version is inspected after the active version
  has moved on
- **THEN** it names the exact version it used

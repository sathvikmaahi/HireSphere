## Purpose

The versioned registry of prompt templates and their output contracts, covering all ten prompt
families the product will use. Each family's contract fixes not only the output's shape but its
length, so that AI content in TalentSphere is concise and precise by construction rather than by
instruction.

## ADDED Requirements

### Requirement: Ten registered prompt families

The registry SHALL contain template records for at least the following families:
`job_description_generation`, `job_posting_generation`, `resume_extraction`, `candidate_ranking`,
`fitment_summary`, `gap_summary`, `interview_questions`, `interview_note_summary`,
`scorecard_generation`, and `resurfacing`.

*Source: `reference/spec.md` §16.3; `AI-003`. The ten families are the registry's inventory, not
the product's four human-facing AI touchpoints (`D11`) — several families serve one touchpoint.*

#### Scenario: Registry inventory

- **WHEN** the registry is listed
- **THEN** all ten families are present, each with at least one version and a declared output
  contract

#### Scenario: Unregistered family

- **WHEN** a run is requested for a family not present in the registry
- **THEN** the run is rejected

### Requirement: Template versioning

Every template SHALL be versioned. A change to template content SHALL create a new version rather
than modifying an existing one, and the active version per family SHALL be configurable.

*Source: `AI-003`, `AI-004`, `ENG-008`. The same fork-on-edit rule `D12` applies to approved job
descriptions, for the same reason: editing in place would silently invalidate every output already
computed against the prior text.*

#### Scenario: Template edited

- **WHEN** a template's content is changed
- **THEN** a new version is created and prior versions remain retrievable

#### Scenario: Active version switched

- **WHEN** an administrator changes the active version for a family
- **THEN** subsequent runs of that family use the new version, and the change is audited with
  previous and new value

#### Scenario: Historical run remains interpretable

- **WHEN** a past run is inspected after the active version has moved on
- **THEN** the exact template version used by that run is still retrievable

### Requirement: Enforced output contracts

Each family SHALL declare a machine-checkable output contract. Output that violates the contract
SHALL be rejected, recorded as a failed run, and SHALL NOT be persisted as usable content.

*Source: `reference/spec.md` §16.3's required output formats; inherited `D9` — if each feature
validated its own output, contract enforcement would be as inconsistent as the features, and a
contract violation would become a malformed insight row instead of a recorded failure.*

#### Scenario: Malformed output

- **WHEN** a provider returns output that does not satisfy the family's contract
- **THEN** the run is marked failed with the validation detail preserved
- **AND** no insight or draft record is created from that output

#### Scenario: Contract satisfied

- **WHEN** output satisfies the contract
- **THEN** it is persisted with a reference to the template version that produced it

### Requirement: Conciseness constraints

Every family's output contract SHALL carry an explicit, machine-checkable conciseness bound — a
maximum sentence, item, or character count — on **each** free-text field, enforced by the same
validation that enforces shape. No family SHALL permit unbounded long-form prose. A family whose
contract declares a free-text field with no bound SHALL be rejected by the registry.

*Source: `S.5`, which identifies the exact gap: §16.3 "specifies an **output shape** for all ten
prompt families … but never a **length**. Nothing in scope so far stops a model from writing three
paragraphs where two sentences would serve." `AGENTS.md`'s standing product-quality bar requires
"an explicit, machine-checkable length bound (sentence count, word cap), not just a JSON shape."
Inherited `D11` — asking a model politely to be brief is not enforcement.*

#### Scenario: Output exceeds its bound

- **WHEN** output satisfies the structural shape but exceeds the declared conciseness bound for a
  field
- **THEN** the output is treated as contract-violating and rejected

#### Scenario: Every family bounded

- **WHEN** the registry is inspected
- **THEN** each of the ten families declares a conciseness bound on each free-text field of its
  contract

#### Scenario: Registering an unbounded field

- **WHEN** a family is registered whose contract declares a free-text field carrying no bound
- **THEN** registration is refused

### Requirement: Provisional bounds are recorded as provisional

Each declared conciseness bound SHALL carry machine-readable provenance recording whether its value
is a confirmed product decision or a provisional default. Changing a bound SHALL be an audited
configuration change, not a code change.

*Source: `S.5`'s explicit statement that the exact per-family numbers are "genuinely open, not yet
specified … a concrete number wasn't given and shouldn't be invented here". A bound that ships
provisionally and is labelled as such is honest and enforceable; an unlabelled one hardens into an
accidental decision, and an absent one is not a contract at all.*

#### Scenario: Bound provenance is visible

- **WHEN** the registry is inspected
- **THEN** each bound states whether its value is confirmed or provisional

#### Scenario: Owner confirms a number

- **WHEN** an owner decision fixes a family's bound
- **THEN** it is applied through the audited configuration path and its provenance changes to
  confirmed, with no deployment required

### Requirement: Evidence and insufficiency in contracts

Contracts for families that make claims about a candidate SHALL require evidence references for
each claim and SHALL permit an explicit insufficiency result instead of an unsupported answer.

*Source: `AI-007`, `AI-008`, `G-02`, `G-03`. `G-03` notes insufficiency "attacks the failure mode
most likely to erode trust in ranking". The evidence-reference model these clauses are written
against is `ai-platform/evidence-labeling`'s; this requirement is the contract clause, not the
substrate.*

#### Scenario: Claim without evidence

- **WHEN** output asserts a claim about a candidate with no evidence reference attached
- **THEN** the output violates the contract and is rejected

#### Scenario: Evidence is missing from the inputs

- **WHEN** the inputs do not support a conclusion
- **THEN** the contract accepts an insufficiency result identifying what was missing, rather than
  requiring a guess

### Requirement: Source labelling in contracts

Contracts for families producing candidate insight SHALL require each claim to label its evidence
source using the source vocabulary defined by the evidence-labeling capability.

*Source: `G-02`, `UI-005`, `BR-008`. The vocabulary is defined once, in
`ai-platform/evidence-labeling`, and referenced here rather than restated, so a family cannot
invent a source value the substrate and the UI do not recognize.*

#### Scenario: Unlabelled claim

- **WHEN** output includes a claim carrying no source label
- **THEN** the output is rejected as contract-violating

#### Scenario: Label outside the vocabulary

- **WHEN** output labels a claim with a source value not in the defined vocabulary
- **THEN** the output is rejected as contract-violating

### Requirement: Promotion gate

A template version SHALL NOT become the active version in a production environment until it has
passed the required evaluation and security test corpus for its family.

*Source: `AI-015` — templates tested against adverse examples **before production release** — and
`ENG-008`, which requires a recorded prompt and model version review for changes affecting ranking
behavior. The corpus itself and the CI enforcement are `ai-platform/ai-evaluation`'s; this
requirement is the registry refusing an activation that has no passing run recorded against it.*

#### Scenario: Untested version promotion attempt

- **WHEN** an administrator attempts to activate a template version with no passing corpus run
  recorded
- **THEN** the activation is refused

#### Scenario: Ranking behavior change

- **WHEN** a change would alter ranking behavior through a prompt or model version
- **THEN** promotion requires a recorded prompt and model version review

### Requirement: No family is invoked by a user-facing feature in this scope

Registered families SHALL be executable for evaluation and smoke purposes in this scope, while no
user-facing feature invokes a family until its owning feature ships.

*Source: inherited `D12` — the families are registered, contracted and tested here, but their
prompt text is a stub, because "writing ten real prompts without the features that use them,
against schemas that [Phase 2] will refine, guarantees rework", while the contract and the tests
are exactly what must exist before the first real call.*

#### Scenario: Registry without consumers

- **WHEN** this feature is deployed
- **THEN** each family can be exercised through the evaluation harness and an authorized diagnostic
  path
- **AND** no end-user screen triggers a family

#### Scenario: Stub text satisfies its own contract

- **WHEN** a family's stub template is executed against the stub provider
- **THEN** the resulting output satisfies that family's contract, including its conciseness bound

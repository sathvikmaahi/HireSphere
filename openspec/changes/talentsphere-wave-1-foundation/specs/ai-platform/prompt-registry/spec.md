## Purpose

The versioned registry of prompt templates and their output contracts, covering all ten prompt families the product will use. Each family's contract fixes not only the output shape but its length, so that AI content in the product stays concise and precise by construction.

## ADDED Requirements

### Requirement: Ten registered prompt families

The registry SHALL contain template records for at least the following families: job_description_generation, job_posting_generation, resume_extraction, candidate_ranking, fitment_summary, gap_summary, interview_questions, interview_note_summary, scorecard_generation, and resurfacing.

#### Scenario: Registry inventory

- **WHEN** the registry is listed
- **THEN** all ten families are present, each with at least one version and a declared output contract

#### Scenario: Unregistered family

- **WHEN** a run is requested for a family not present in the registry
- **THEN** the run is rejected

### Requirement: Template versioning

Every template SHALL be versioned. A change to template content SHALL create a new version rather than modifying an existing one, and the active version per family SHALL be configurable.

#### Scenario: Template edited

- **WHEN** a template's content is changed
- **THEN** a new version is created and prior versions remain retrievable

#### Scenario: Active version switched

- **WHEN** an administrator changes the active version for a family
- **THEN** subsequent runs of that family use the new version, and the change is audited with previous and new value

#### Scenario: Historical run remains interpretable

- **WHEN** a past run is inspected after the active version has moved on
- **THEN** the exact template version used by that run is still retrievable

### Requirement: Enforced output contracts

Each family SHALL declare a machine-checkable output contract. Output that violates the contract SHALL be rejected, recorded as a failed run, and SHALL NOT be persisted as usable content.

#### Scenario: Malformed output

- **WHEN** a provider returns output that does not satisfy the family's contract
- **THEN** the run is marked failed with the validation detail preserved
- **AND** no insight or draft record is created from that output

#### Scenario: Contract satisfied

- **WHEN** output satisfies the contract
- **THEN** it is persisted with a reference to the template version that produced it

### Requirement: Conciseness constraints

Every family's output contract SHALL carry an explicit, machine-checkable conciseness bound — such as a maximum sentence, item, or character count per field — enforced structurally by the contract rather than stated as style guidance. No family SHALL permit unbounded long-form prose.

#### Scenario: Output exceeds its bound

- **WHEN** output satisfies the structural shape but exceeds the declared conciseness bound for a field
- **THEN** the output is treated as contract-violating and rejected

#### Scenario: Every family bounded

- **WHEN** the registry is inspected
- **THEN** each of the ten families declares a conciseness bound on each free-text field of its contract

### Requirement: Evidence and insufficiency in contracts

Contracts for families that make claims about a candidate SHALL require evidence references for each claim and SHALL permit an explicit insufficiency result instead of an unsupported answer.

#### Scenario: Claim without evidence

- **WHEN** output asserts a claim about a candidate with no evidence reference attached
- **THEN** the output violates the contract and is rejected

#### Scenario: Evidence is missing from the inputs

- **WHEN** the inputs do not support a conclusion
- **THEN** the contract accepts an insufficiency result identifying what was missing, rather than requiring a guess

### Requirement: Source labelling in contracts

Contracts for families producing candidate insight SHALL require each claim to label its evidence source as resume, interview note, scorecard, or human decision.

#### Scenario: Unlabelled claim

- **WHEN** output includes a claim carrying no source label
- **THEN** the output is rejected as contract-violating

### Requirement: Promotion gate

A template version SHALL NOT become the active version in a production environment until it has passed the required evaluation and security test corpus for its family.

#### Scenario: Untested version promotion attempt

- **WHEN** an administrator attempts to activate a template version with no passing corpus run recorded
- **THEN** the activation is refused

#### Scenario: Ranking behavior change

- **WHEN** a change would alter ranking behavior through a prompt or model version
- **THEN** promotion requires a recorded prompt and model version review

### Requirement: No family is invoked by a user-facing feature in this scope

Registered families SHALL be executable for evaluation and smoke purposes in this scope, while no user-facing feature invokes a family until its owning feature ships.

#### Scenario: Registry without consumers

- **WHEN** this wave is deployed
- **THEN** each family can be exercised through the evaluation harness and an authorized diagnostic path
- **AND** no end-user screen triggers a family

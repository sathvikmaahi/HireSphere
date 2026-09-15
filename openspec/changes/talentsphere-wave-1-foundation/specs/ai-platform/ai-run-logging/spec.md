## Purpose

The permanent record of every AI invocation and of every human decision that diverges from one. This history cannot be reconstructed after the fact, which is why it exists before the first AI feature ships.

## ADDED Requirements

### Requirement: Every AI run is logged

Every AI run SHALL record model provider, model name, model version, prompt template identifier and version, input object references, output object references, token usage where available, safety flags, status, start and completion timestamps, and failure detail where applicable.

#### Scenario: Successful run

- **WHEN** an AI run completes successfully
- **THEN** a run record exists carrying provider, model, model version, template version, input and output references, and timing

#### Scenario: Failed run

- **WHEN** an AI run fails
- **THEN** the run record is retained with status failed and the failure detail preserved

#### Scenario: Logging is not optional

- **WHEN** a run cannot be logged
- **THEN** the run is not issued to the provider

### Requirement: Reference-only inputs

Run records SHALL store references to source objects rather than raw sensitive content. Resume text, candidate contact details, and other personal data SHALL NOT be stored in the run record.

#### Scenario: Administrator reads AI run logs

- **WHEN** a user with AI run log access reads a run record for a candidate-related run
- **THEN** the record identifies the source objects by reference
- **AND** no resume text or candidate contact data is present in the payload

#### Scenario: Reconstructing a run's inputs

- **WHEN** an investigator needs the actual inputs behind a run
- **THEN** the references resolve to the source records, and reading those records requires permission on them

### Requirement: Safety flags

Safety and policy filter results SHALL be recorded on the run.

#### Scenario: Safety filter triggers

- **WHEN** a safety or policy filter flags a run
- **THEN** the flag and its category are recorded on the run record and the run is retrievable by that flag

### Requirement: Reproducibility tuple

Each run SHALL be reconstructible from the recorded combination of input references, prompt template version, and model version.

#### Scenario: Explaining a past output

- **WHEN** a past AI output is questioned
- **THEN** the run record identifies which input versions, template version, and model version produced it

### Requirement: AI content labelled until approved

Content produced by an AI run SHALL be marked as AI-generated or AI-assisted, and SHALL retain that marking until a human user approves it.

#### Scenario: Unapproved AI content

- **WHEN** AI-produced content is stored or displayed before human approval
- **THEN** it carries an AI-generated or AI-assisted marker

#### Scenario: Approved content

- **WHEN** an authorized human approves AI-produced content
- **THEN** the approval is recorded with actor and timestamp, and the content is no longer presented as unreviewed

### Requirement: Human override records

The system SHALL provide a first-class override record for any human decision that departs from AI output. An override SHALL require a reason and SHALL NOT delete or overwrite the original AI output.

#### Scenario: Override recorded

- **WHEN** a user overrides AI output
- **THEN** an override record captures the actor, timestamp, reason, the original AI output reference, and the human value
- **AND** the original AI output remains retrievable

#### Scenario: Override without a reason

- **WHEN** an override is submitted with no reason
- **THEN** it is rejected

#### Scenario: Divergence is measurable

- **WHEN** override records are queried over a period
- **THEN** the rate at which humans diverged from AI output can be computed from the stored records alone

### Requirement: Output feedback

The system SHALL provide a mechanism for users to flag AI output as inaccurate, incomplete, biased, unsafe, or not useful, and SHALL retain feedback linked to the run.

#### Scenario: Flagging output

- **WHEN** a user flags an AI output with a category and optional comment
- **THEN** the feedback is stored against the run and is retrievable for governance review

#### Scenario: Feedback does not mutate output

- **WHEN** feedback is submitted
- **THEN** the original output is unchanged

### Requirement: Run log access and search

AI run records SHALL be searchable by authorized users, filterable by family, template version, model version, status, safety flag, and time range, and SHALL be readable by the Auditor role without granting access to candidate personal data.

#### Scenario: Auditor reviews AI activity

- **WHEN** an Auditor searches AI run records
- **THEN** matching records are returned with governance metadata and no candidate personal data

#### Scenario: Unauthorized access

- **WHEN** a user without AI run log permission calls the run log endpoint
- **THEN** the request is denied

### Requirement: Retention

AI run records SHALL be retained long enough to support auditability, debugging, and model governance review, according to configured policy.

#### Scenario: Retention policy applied

- **WHEN** the retention policy is configured
- **THEN** run records are retained for at least that period and their disposal is itself recorded

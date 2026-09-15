## Purpose

The maintained test corpus and CI harness that must pass before any prompt template reaches production. It is the primary control against hallucinated claims, prompt injection, protected-attribute leakage, and unbounded AI prose.

## ADDED Requirements

### Requirement: Required AI evaluation coverage

The evaluation corpus SHALL cover, for each prompt family that consumes candidate or interview content: low-information inputs, adversarial inputs, conflicting evidence, protected-attribute redaction, insufficiency outputs, and evidence citation.

#### Scenario: Low-information input

- **WHEN** a family is evaluated against an input carrying almost no usable content
- **THEN** the output returns an insufficiency result rather than fabricated detail

#### Scenario: Adversarial input

- **WHEN** a family is evaluated against an input crafted to inflate its own assessment or to issue instructions
- **THEN** the injected instruction does not take effect and the output remains within its contract

#### Scenario: Conflicting evidence

- **WHEN** a family is evaluated against inputs that contradict one another
- **THEN** the output reports the conflict rather than silently selecting one side

#### Scenario: Protected attributes

- **WHEN** an input contains protected or sensitive attributes
- **THEN** those attributes are absent from the output and from any scoring rationale

#### Scenario: Evidence citation

- **WHEN** output asserts a claim about a candidate
- **THEN** the claim carries a resolvable evidence reference

### Requirement: Required security coverage

The security test suite SHALL cover broken access control, file upload attacks, prompt injection samples, sensitive data exposure, and export permission checks.

#### Scenario: Broken access control probe

- **WHEN** the suite calls protected endpoints without or with insufficient permission
- **THEN** every call is denied and the suite fails if any succeeds

#### Scenario: Sensitive data exposure probe

- **WHEN** the suite inspects responses, logs, and run records for personal data that should be excluded
- **THEN** the suite fails if any is present

#### Scenario: Export permission probe

- **WHEN** the suite attempts an export without Export permission
- **THEN** the attempt is denied

### Requirement: Conciseness compliance testing

The corpus SHALL verify that each family's output respects its declared conciseness bounds.

#### Scenario: Verbose output detected

- **WHEN** a family produces output exceeding its declared bound on any evaluated case
- **THEN** the evaluation fails for that family

### Requirement: CI harness gates promotion

The evaluation and security suites SHALL run in CI, and a failing suite SHALL block both merge and template promotion.

#### Scenario: Failing corpus blocks merge

- **WHEN** a change causes any required evaluation or security case to fail
- **THEN** the pipeline blocks the merge

#### Scenario: Promotion without a passing run

- **WHEN** promotion of a template version is attempted with no passing corpus run recorded for it
- **THEN** promotion is refused

### Requirement: Results attributable to versions

Evaluation results SHALL be recorded against the specific prompt template version and model version that produced them.

#### Scenario: Comparing two template versions

- **WHEN** two template versions have been evaluated
- **THEN** their results are separately retrievable and comparable by version

#### Scenario: Model changed under a fixed template

- **WHEN** the configured model changes with the template unchanged
- **THEN** the corpus is re-run and results are recorded against the new model version

### Requirement: Corpus is versioned and maintained

The corpus SHALL be version-controlled, and cases SHALL be added as new failure modes are found in use.

#### Scenario: New failure mode found in use

- **WHEN** a real AI output failure is identified through output feedback or governance review
- **THEN** a corresponding case is added to the corpus so the failure is detected on future runs

### Requirement: Fairness measurement remains out of scope

The harness SHALL NOT collect or process demographic or protected-class data for fairness measurement. Protected-attribute testing SHALL verify redaction and exclusion only.

#### Scenario: Attempted demographic analysis

- **WHEN** an evaluation case would require demographic or protected-class data to compute an outcome distribution
- **THEN** the case is not implemented, and the requirement is recorded as pending legal and compliance approval

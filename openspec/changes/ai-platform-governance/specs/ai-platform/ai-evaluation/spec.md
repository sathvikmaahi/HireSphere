## Purpose

The maintained test corpus and CI harness that must pass before any prompt template reaches
production. It is the primary control against hallucinated claims, prompt injection,
protected-attribute leakage, and unbounded AI prose — and it is what makes every other contract in
this feature enforced rather than merely declared.

## ADDED Requirements

### Requirement: Required AI evaluation coverage

The evaluation corpus SHALL cover, for each prompt family that consumes candidate or interview
content: low-information inputs, adversarial inputs, conflicting evidence, protected-attribute
redaction, insufficiency outputs, and evidence citation.

*Source: `reference/spec.md` §27.1's AI Evaluation Tests row, adopted in full by `C-09`, which
reverses the earlier non-goal that an offline eval harness is a separate initiative. `AI-015` makes
adverse-example testing a precondition of production release rather than a later hardening step.*

#### Scenario: Low-information input

- **WHEN** a family is evaluated against an input carrying almost no usable content
- **THEN** the output returns an insufficiency result rather than fabricated detail

#### Scenario: Adversarial input

- **WHEN** a family is evaluated against an input crafted to inflate its own assessment or to issue
  instructions
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

### Requirement: Every registered family is covered

Every family present in the prompt registry SHALL have corpus coverage for each category that
applies to it. Registering a family without coverage SHALL fail the suite rather than pass by
absence.

*Source: `C-09`'s adoption of §27.1 across **all ten** prompt families, and `AGENTS.md`'s standing
bar that conciseness be machine-checkable rather than documented. A corpus that only covers the
families someone remembered to add cases for reports green for the ones nobody did — the same
silent-success failure class `platform-core` D5 was written against, where a gate that skips reads
identically to a gate that passes.*

#### Scenario: New family registered without cases

- **WHEN** a family is added to the registry with no corpus cases for an applicable category
- **THEN** the suite fails, naming the family and the missing category

#### Scenario: Coverage is enumerated, not sampled

- **WHEN** the suite reports coverage
- **THEN** it reports it per family and per category, so an uncovered combination is visible rather
  than inferred

### Requirement: Conciseness compliance testing

The corpus SHALL verify that each family's output respects its declared conciseness bounds, and a
family producing output beyond its own declared bound SHALL fail evaluation.

*Source: `S.5`, which directs that the harness "should test for length/conciseness compliance as
one of its checks, alongside the adversarial, low-information, and protected-attribute tests
already planned". This is the requirement that makes `S.5` structural: a template that violates its
own bound fails here, rather than being described as verbose in a review comment.*

#### Scenario: Verbose output detected

- **WHEN** a family produces output exceeding its declared bound on any evaluated case
- **THEN** the evaluation fails for that family

#### Scenario: Bound present but never exercised

- **WHEN** a family declares a conciseness bound with no case that tests it
- **THEN** the suite fails, because a bound no case exercises is not an enforced bound

### Requirement: Required security coverage

The security test suite SHALL cover broken access control, file upload attacks, prompt injection
samples, sensitive data exposure, and export permission checks.

*Source: `reference/spec.md` §27.1's Security Tests row, adopted alongside the evaluation coverage
by `C-09`. Prompt injection appears in both suites deliberately: the evaluation suite asks whether
the output stayed within contract, the security suite asks whether anything else happened.*

#### Scenario: Broken access control probe

- **WHEN** the suite calls protected endpoints without or with insufficient permission
- **THEN** every call is denied and the suite fails if any succeeds

#### Scenario: Sensitive data exposure probe

- **WHEN** the suite inspects responses, logs, and run records for personal data that should be
  excluded
- **THEN** the suite fails if any is present

#### Scenario: Export permission probe

- **WHEN** the suite attempts an export without Export permission
- **THEN** the attempt is denied

### Requirement: CI harness gates promotion

The evaluation and security suites SHALL run in CI, and a failing suite SHALL block both merge and
template promotion.

*Source: `C-09`'s cost note — "a maintained test corpus and CI integration touching … every
AI-bearing slice thereafter" — and `AI-015`'s before-production-release wording, which is a gate
rather than a practice. `platform-core` D5's selection rule applies: a conditional gate must fail
when its condition is unmet rather than skip.*

#### Scenario: Failing corpus blocks merge

- **WHEN** a change causes any required evaluation or security case to fail
- **THEN** the pipeline blocks the merge

#### Scenario: Promotion without a passing run

- **WHEN** promotion of a template version is attempted with no passing corpus run recorded for it
- **THEN** promotion is refused

#### Scenario: Suite unable to run

- **WHEN** the suite cannot execute — no provider, no corpus, or a misconfiguration
- **THEN** the gate fails rather than skipping

### Requirement: Results attributable to versions

Evaluation results SHALL be recorded against the specific prompt template version and model version
that produced them.

*Source: `ENG-008`, `AI-004`, `G-07`. The registry's promotion gate refuses activation without a
passing run recorded **for that version**, which is only meaningful if results are attributable to
one.*

#### Scenario: Comparing two template versions

- **WHEN** two template versions have been evaluated
- **THEN** their results are separately retrievable and comparable by version

#### Scenario: Model changed under a fixed template

- **WHEN** the configured model changes with the template unchanged
- **THEN** the corpus is re-run and results are recorded against the new model version

#### Scenario: Provider changed under a fixed template and model

- **WHEN** the configured provider adapter changes
- **THEN** the corpus is re-run, because a passing result attributed to one provider is not
  evidence about another

### Requirement: Corpus is versioned and maintained

The corpus SHALL be version-controlled, and cases SHALL be added as new failure modes are found in
use.

*Source: `C-09`'s middle-path resolution, which accepts the cost of a maintained corpus rather than
a one-time test set. Output feedback (`AI-011`) is the intake path: a real failure a user flags
becomes a case, which is what stops the corpus decaying into the failure modes anticipated before
the product had users.*

#### Scenario: New failure mode found in use

- **WHEN** a real AI output failure is identified through output feedback or governance review
- **THEN** a corresponding case is added to the corpus so the failure is detected on future runs

### Requirement: Fairness measurement remains out of scope

The harness SHALL NOT collect or process demographic or protected-class data for fairness
measurement. Protected-attribute testing SHALL verify redaction and exclusion only.

*Source: `PRV-006` and `OD-010`, which forbid collecting demographic or protected-class data
without legal approval. `C-09` is explicit that only the eval-harness non-goal was reversed and the
bias-audit non-goal survives — the reason is not that fairness is unimportant but that collecting
the data it needs is itself restricted.*

#### Scenario: Attempted demographic analysis

- **WHEN** an evaluation case would require demographic or protected-class data to compute an
  outcome distribution
- **THEN** the case is not implemented, and the requirement is recorded as pending legal and
  compliance approval

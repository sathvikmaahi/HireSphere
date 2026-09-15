## Purpose

The Scorecard Center's drafting stage: only interviewed candidates are eligible, the draft separates
resume evidence from interview evidence, and it records exactly which note versions it was drawn from.
Owned by `TS-BL-060`.

## ADDED Requirements

### Requirement: The Scorecard Center lists interviewed candidates only

The Scorecard Center SHALL list only candidates with at least one completed interview round for the
posting. A candidate with no completed round SHALL NOT appear in the list.

*Source: `SCR-002`, `BR-011`, §14.2's Scorecard Center screen description ("interviewed candidates
only"), §13.2's `GET /api/postings/{postingId}/scorecards/eligible`.*

#### Scenario: Interviewed candidate

- **WHEN** a candidate has a completed round for the posting
- **THEN** they appear in the eligible list

#### Scenario: Shortlisted but not interviewed

- **WHEN** a candidate is shortlisted with no completed round
- **THEN** they do not appear in the eligible list

#### Scenario: Round scheduled but not completed

- **WHEN** a candidate has a scheduled round that is not complete
- **THEN** they do not appear in the eligible list

### Requirement: Generation is refused for a candidate with no completed interview

Scorecard generation SHALL be refused for an Application with no completed interview round, including
when requested directly against the API.

*Source: `SCR-001` ("at least one completed interview **for the posting**"), `AI-009`, `BR-010`.
`C-07` records that carry-forward does not relax this: a candidate carrying prior evidence still
completes one confirmatory round for the new posting.*

#### Scenario: Generation for an uninterviewed candidate

- **WHEN** generation is requested for an Application with no completed round
- **THEN** it is refused and no draft is created

#### Scenario: Direct API request

- **WHEN** the same request is submitted directly against the API rather than through the list
- **THEN** it is refused on the same rule

#### Scenario: Carried evidence does not substitute

- **WHEN** an Application holds evidence carried from a prior Application but has no completed round
  of its own
- **THEN** generation is still refused

### Requirement: The draft records the note versions it was generated from

A generated scorecard SHALL record the set of interview note versions it was drawn from. The set SHALL
be recorded at generation time and SHALL remain readable for the life of the scorecard.

*Source: `D24`'s one consequence that survives its own reversal — "the approved scorecard must record
which **note version** it was drafted from" — which `C-08` makes more necessary rather than less,
since without a lock the recorded set is the only thing that makes drift detectable.
`interview/scorecard-approval` consumes this set to scope supersession. `design.md` D9.*

#### Scenario: Draft generated

- **WHEN** a draft is generated
- **THEN** it records the note version identifiers it consumed

#### Scenario: Multiple rounds

- **WHEN** a draft is generated from notes across several rounds
- **THEN** the recorded set names a version for each note consumed

#### Scenario: Set readable later

- **WHEN** a scorecard is read after later note versions exist
- **THEN** the recorded set still names the versions the draft was generated from

### Requirement: Resume-derived and interview-derived evidence are stored separately

A scorecard SHALL store resume-derived evidence and interview-derived evidence as separate bodies.
Output that merges them SHALL be rejected as a contract violation. The separation SHALL exist in the
stored artifact, not only in its presentation.

*Source: `SCR-004`, §12.2's separate `resume_evidence_json` and `interview_evidence_json`, `UI-005`.
`C-05`'s reversal of `D11` rests on `SCR-004`/`SCR-005` being explicitly not averaging, which is a
property of the stored record. `design.md` D11 records why a render-time separation leaves every later
reader — carry-forward, resurfacing, audit — with one merged body.*

#### Scenario: Draft inspected

- **WHEN** a generated draft is inspected in storage
- **THEN** resume-derived and interview-derived evidence are held separately

#### Scenario: Merged output

- **WHEN** generated output merges the two evidence bodies
- **THEN** it is rejected as a contract violation and recorded as a failed run

#### Scenario: Display-time separation sought

- **WHEN** the codebase is inspected for a presentation-layer split relied upon for this separation
- **THEN** none is relied upon

### Requirement: Every dimension carries an evidence source and a confidence level

Each scorecard dimension SHALL carry an evidence source label from the platform's closed
evidence-source vocabulary and a confidence level. A dimension lacking either SHALL be rejected as a
contract violation. This capability SHALL derive no source value of its own.

*Source: `SCR-005`, `G-02`, `AI-007`, `BR-008`; `ai-platform-governance`'s
`ai-platform/evidence-labeling` closed vocabulary, whose whole value is that exactly one component
defines source values, and `design-system`'s label component, which "renders the source value it is
given without interpreting or deriving it". `design.md` D11.*

#### Scenario: Dimension inspected

- **WHEN** a dimension of a generated draft is inspected
- **THEN** it carries a source label from the closed vocabulary and a confidence level

#### Scenario: Missing confidence

- **WHEN** generated output supplies a dimension with no confidence level
- **THEN** it is rejected as a contract violation

#### Scenario: Source value derived locally

- **WHEN** this capability is inspected for its own definition of a source value
- **THEN** none exists

### Requirement: Insufficient evidence is marked, never inferred

Where the evidence does not support a dimension, the scorecard SHALL carry an insufficiency marker
naming what evidence was absent, rather than an inferred value. An insufficiency SHALL be
distinguishable from a dimension evaluated as weak and from a dimension not evaluated.

*Source: `G-03`, `AI-008`; and `matching-and-ranking`'s `matching/ranking-insufficiency`, which
established this vocabulary and its three-state distinction for ranking output. Reused verbatim rather
than re-termed, on that feature's `design.md` D10 reasoning: two vocabularies for one concept would
make "insufficient" mean subtly different things on two screens describing the same candidate.*

#### Scenario: Unsupported dimension

- **WHEN** the evidence does not support a dimension
- **THEN** an insufficiency marker is recorded naming the absent evidence

#### Scenario: Three states distinguishable

- **WHEN** an insufficient dimension, a dimension evaluated as weak and a dimension not evaluated are
  compared
- **THEN** the three are distinguishable

#### Scenario: Inferred value

- **WHEN** generated output assigns a dimension a value with no evidence reference
- **THEN** it is rejected as a contract violation

### Requirement: Generation runs through the gateway on references and registers one job type

Scorecard generation SHALL be invoked through the AI gateway in its fixed envelope as a new version of
the already-registered `scorecard_generation` family, carrying resume-version and note-version
references rather than content. It SHALL register the Scorecard Generation background job type against
the shared dispatch pattern and SHALL register no job type for the gateway call itself. Every
free-text field of its output contract SHALL declare a length bound, and output exceeding one SHALL be
rejected as a contract violation recorded as a failed run.

*Source: `ai-platform-governance`'s `ai-platform/ai-gateway` and its `design.md` D3, D7 and D10;
`AI-003`; §16.3's `scorecard_generation` required output — "strict JSON with dimensions, evidence,
confidence, recommendation"; §25's Scorecard Generation row; `S.5` and `AGENTS.md`'s standing
conciseness bar. Bounds ship provisional and machine-readably labelled as such (`design.md` Open
Questions).*

#### Scenario: Envelope contents

- **WHEN** a generation request is inspected
- **THEN** it carries resume-version and note-version references and no resume text, no note text and
  no candidate personal data

#### Scenario: One job type

- **WHEN** this capability's registered job types are enumerated
- **THEN** exactly one is present, and none corresponds to a gateway call

#### Scenario: Duplicate delivery

- **WHEN** the same generation request is delivered twice
- **THEN** one run is issued and one draft exists

#### Scenario: Provider call sought

- **WHEN** this capability is inspected for an outbound call to a model provider
- **THEN** none exists

### Requirement: Untrusted document-derived content is marked at the boundary

Resume-derived and note-derived content reaching a prompt SHALL be marked as untrusted
document-derived content. Text shaped as an instruction SHALL alter neither the prompt family, the
caller's permissions, the dimensions evaluated, nor the output contract.

*Source: `AI-012`, `AI-013`, `AI-014`, `SEC-014`; `D18` names ranking as the larger injection surface,
and scorecard generation is the second surface where resume text reaches a model. Interview notes are
a new untrusted surface this feature introduces: they are written by internal users, but they quote
candidates.*

#### Scenario: Instruction-shaped resume text

- **WHEN** a resume contains text shaped as an instruction to raise its own evaluation
- **THEN** the family, permissions, dimensions and output contract are unchanged

#### Scenario: Instruction-shaped note text

- **WHEN** a note quotes candidate text shaped as an instruction
- **THEN** the same guarantees hold

### Requirement: A generated scorecard is advisory until approved

A generated scorecard SHALL be created in Draft status, SHALL carry the AI-generated marking, and
SHALL NOT be treated as an evaluation of record. Generation SHALL cause no Application state
transition.

*Source: `SCR-006`, `AI-001`, `AI-010`, `BR-007`, `UI-004`, `project.md`'s governing rule.
The approval gate itself is `interview/scorecard-approval`.*

#### Scenario: Draft created

- **WHEN** a scorecard is generated
- **THEN** it is in Draft status and carries the AI-generated marking

#### Scenario: Transition from generation

- **WHEN** generation completes
- **THEN** no Application state transition occurs

#### Scenario: Marking cleared

- **WHEN** the AI-generated marking is inspected on a Draft scorecard
- **THEN** it is present, and no path in this capability clears it

### Requirement: Generation failure degrades without blocking the pipeline

When the model provider or generation path is unavailable, existing scorecards, interview notes,
rounds and every non-AI action SHALL remain fully usable, and the failure SHALL be visible with a
retrievable run reference rather than producing an empty or partial draft presented as complete.

*Source: `G-12`, `NFR-004`, `ERR-007`'s partial-failure reporting, `NFR-002`'s asynchronous execution
with progress status. `matching-and-ranking` carries the equivalent requirement for ranking, including
the specific failure of proceeding on an empty evidence set.*

#### Scenario: Provider unavailable

- **WHEN** generation is requested while the provider is unavailable
- **THEN** the request reports temporary unavailability, existing scorecards and notes remain usable,
  and no partial draft is created

#### Scenario: Failure traceable

- **WHEN** a generation attempt fails
- **THEN** a run reference is retrievable for the failure

#### Scenario: Progress retrievable

- **WHEN** generation is in progress
- **THEN** its status is retrievable by identifier

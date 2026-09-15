## Purpose

The record that makes a score explicable: the full reproducibility tuple, versioned so that
re-ranking preserves the board it replaced, with an explicit representation of a ranking whose
inputs have moved on underneath it. Owned by `TS-BL-051`.

## ADDED Requirements

### Requirement: A ranking entry records the full reproducibility tuple

Every ranking entry SHALL record the resume version, the job description version, the prompt
template version and the model version it was produced from, together with the AI run reference. A
ranking entry SHALL NOT be created with any term of that tuple absent.

*Source: `domain-model.md`'s ranking provenance — `RankingScore = f(resume_version, jd_version,
prompt_template_version, model_version)` — and its statement that without the full tuple "a score
can't be explained or reproduced once the JD or the model changes underneath it"; `AI-002`,
`RANK-002`'s AI run version column; `ai-platform-governance`'s reproducibility-tuple requirement,
which this is the first real consumer of.*

#### Scenario: Entry created

- **WHEN** a ranking entry is persisted
- **THEN** it carries resume version, job description version, prompt template version, model
  version and a run reference

#### Scenario: Incomplete tuple

- **WHEN** a ranking entry is submitted with any tuple term missing
- **THEN** it is rejected rather than stored with a null term

#### Scenario: Explaining a past score

- **WHEN** a past score is questioned
- **THEN** the four versions that produced it are readable from the entry, and each resolves to the
  content it names

### Requirement: Regenerating a ranking creates a new version and preserves prior versions

Re-ranking a posting SHALL create a new ranking version. Prior ranking versions and their entries
SHALL be preserved and remain readable. No ranking entry SHALL be updated in place with a new score.

*Source: `G-07`, `RANK-005` — "regenerated rankings shall create a new version while preserving
prior versions"; `G-07`'s own reasoning that "the reproducibility tuple identified *what* to record;
this adds versioning around it, so a re-rank after a JD edit or model change does not destroy the
earlier board."*

#### Scenario: Re-rank

- **WHEN** a posting is re-ranked
- **THEN** a new ranking version is created and the previous version's entries remain readable
  unchanged

#### Scenario: Prior board retrievable

- **WHEN** an earlier ranking version is requested
- **THEN** its full entry set is returned as it stood, with its own tuple values

#### Scenario: In-place score update attempted

- **WHEN** a write against an existing ranking entry's score is attempted
- **THEN** it is refused

### Requirement: Exactly one ranking version is active per posting, and the application's columns point at it

Each posting SHALL have at most one active ranking version. The application record's rank and match
score SHALL reflect the active version and SHALL be derived from it rather than written
independently.

*Source: §12.2's `candidate_posting_applications.latest_rank` ("active ranking version output") and
`latest_match_score`; `candidate-intake`'s `TS-BL-046`, which created both columns and deliberately
left them unwritten for this feature. `design.md` D4 records why the versioned record is the source
and these two columns are a pointer.*

#### Scenario: New version activated

- **WHEN** a new ranking version is activated
- **THEN** the applications' rank and match score reflect it, and the prior version's values remain
  readable on that version's entries

#### Scenario: Independent write attempted

- **WHEN** a code path writes the application's rank or match score without an active ranking
  version to derive it from
- **THEN** the test suite fails

#### Scenario: Posting never ranked

- **WHEN** a posting has never been ranked
- **THEN** its applications carry no rank and no match score, and that state is distinguishable from
  a score of zero

### Requirement: A ranking entry whose inputs have moved on is marked stale, not recomputed

Where the resume version an application is pinned to, or a ranking-relevant posting input, differs
from the version recorded on the active ranking entry, that entry SHALL be marked stale. Staleness
SHALL NOT delete the entry, alter its score, or trigger an automatic re-rank.

*Source: `G-07`'s preservation rule; `candidate-intake`'s `candidate/resume-versioning` capability,
which records that "whether a re-point forces a re-rank is `matching-and-ranking`'s to decide under
`G-07`", and its `TS-BL-048` requirement that a re-point "emit[s] no state transition and recomput[es]
no ranking"; §25's Candidate Ranking triggers, which name posting open, manual trigger and
requirement change — and not a resume change. `design.md` D5 gives the full trigger table.*

#### Scenario: Resume re-pointed

- **WHEN** an application is explicitly re-pointed to a different resume version
- **THEN** its active ranking entry is marked stale, its score is unchanged, and no re-rank occurs

#### Scenario: Stale entry readable

- **WHEN** a stale ranking entry is read
- **THEN** its score, its tuple and its staleness are all visible, and the version it was computed
  against is named

#### Scenario: Candidate merge

- **WHEN** a candidate merge leaves an application holding a ranking computed against a superseded
  input
- **THEN** the entry is marked stale and no ranking is deleted, merged or recomputed

#### Scenario: Staleness cleared

- **WHEN** the posting is re-ranked
- **THEN** the new version's entries are not stale and the prior stale entries remain readable

### Requirement: Ranking output is advisory, marked, and sets no governed state

A ranking entry SHALL persist only as advisory insight carrying its AI-generated marking until a
recorded human approval, SHALL advance no workflow state, and SHALL set no field that gates a
decision.

*Source: `AI-001`, `AI-010`, `BR-007`, `RANK-004`, `project.md`'s central rule;
`ai-platform-governance`'s `ai-platform/advisory-only` capability, which this consumes rather than
re-enforces. `mandatory_criteria_status` is the one adjacent field a ranking run touches, and
`G-01`'s asymmetry governs it — see `matching/ranking-engine`.*

#### Scenario: Ranking completes

- **WHEN** a ranking version completes
- **THEN** its entries are stored as advisory insight marked AI-generated, and no application,
  candidate or posting state has changed

#### Scenario: Transition path sought

- **WHEN** a persistence path from a ranking entry to a workflow-governed state field is sought
- **THEN** none exists

#### Scenario: Human approves a ranking entry

- **WHEN** an authorized human approves a ranking entry's content
- **THEN** the marking clears for that entry only, recording actor and timestamp

### Requirement: Ranking versions are retained and their disposal is recorded

Ranking versions SHALL be retained according to configured retention policy, and their disposal
SHALL itself be recorded.

*Source: `G-07`'s preservation requirement combined with `RET`-level policy, whose interval is
deferred to company policy under `OD-004`; the same mechanism-specified, interval-deferred treatment
`ai-platform-governance` applied to run-log retention. `D22` caps configurable windows by the
retention period, so the two cannot be set inconsistently.*

#### Scenario: Retention applied

- **WHEN** the retention policy is configured
- **THEN** ranking versions are retained for at least that period and their disposal is recorded

#### Scenario: Disposal does not orphan a run

- **WHEN** a ranking version is disposed of
- **THEN** the AI run records it referenced remain retrievable on their own retention terms

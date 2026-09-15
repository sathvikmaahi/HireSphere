## Purpose

Resume versions that never change and never disappear, one active version per candidate, and an
application that stays pinned to the version it was submitted with — so that a ranking score computed
against a resume stays explicable after a newer resume arrives. Owned by `TS-BL-048`.

## ADDED Requirements

### Requirement: Every upload preserves a new version and overwrites none

Each accepted resume upload for a candidate SHALL create the next sequential version, and SHALL NOT
alter or replace any prior version. A version's content, hash and storage reference SHALL be immutable
once created.

*Source: `DOC-010` — "every resume upload shall preserve a new resume version and shall not overwrite
prior resume versions"; `RET-002`.*

#### Scenario: Second resume for a candidate

- **WHEN** a second resume is uploaded for an existing candidate
- **THEN** a new version is created with the next sequential number, and version one is unchanged

#### Scenario: Write against an existing version

- **WHEN** a write is attempted against an existing version's content, hash or storage reference
- **THEN** it is rejected

### Requirement: An identical file under the same candidate creates no new version

Where an uploaded file's hash matches a resume version already belonging to the same candidate, the
upload SHALL link to that existing version rather than creating a new one.

*Source: `G-04` — "same hash under the same candidate means the same resume: link to the existing
record, do not create a new version"; `D23`'s table. `design.md` D8 reconciles this against
`DOC-010`'s literal wording: `DOC-010`'s purpose is that an upload never *overwrites* a prior version,
and linking to an identical existing version overwrites nothing.*

#### Scenario: Same file uploaded twice for one candidate

- **WHEN** the identical file is uploaded a second time for the same candidate
- **THEN** it links to the existing version, the version count does not increase, and the second
  upload is recorded as having occurred

#### Scenario: Same content, different file

- **WHEN** a file with different bytes but similar content is uploaded for the same candidate
- **THEN** a new version is created, since the signal is document-level and not content-level

### Requirement: One version is active per candidate

Exactly one resume version per candidate SHALL be marked active at a time, and the active version
SHALL be what a new submission and the candidate profile default to.

*Source: §12.2's `candidate_resumes.is_active_version`; `RET-002`'s active-version indicator.*

#### Scenario: New version uploaded

- **WHEN** a new version is created for a candidate
- **THEN** it becomes the active version and the prior version ceases to be active

#### Scenario: Active version count

- **WHEN** a candidate's resume versions are inspected
- **THEN** exactly one is marked active

#### Scenario: Active version changed by a human

- **WHEN** a permitted user makes an earlier version active again
- **THEN** the change is recorded with actor, reason and timestamp

### Requirement: An application stays pinned to the version it was submitted with

An application SHALL record the resume version it was submitted with, and that reference SHALL NOT
change when a newer version is uploaded for the candidate.

*Source: `domain-model.md`'s `resume_version_ref` on `Application` and `RankingScore =
f(resume_version, jd_version, prompt_template_version, model_version)`, without which "a score can't
be explained or reproduced once the JD or the model changes underneath it"; §12.2's
`candidate_posting_applications.current_resume_id`. `design.md` D8 — the mirror of `hiring-postings`
D9's pinning of `jd_version`, for the other term of the same tuple.*

#### Scenario: Newer resume uploaded for a candidate with a live application

- **WHEN** a candidate with an existing application uploads a newer resume
- **THEN** the application still references the version it was submitted with, and the newer version
  is available but not attached

#### Scenario: Score remains explicable

- **WHEN** a ranking computed against version one is read after version two exists
- **THEN** the version it was computed against is identifiable and its content is retrievable

#### Scenario: Silent re-point attempted

- **WHEN** a code path re-points an application's resume version without an explicit human action
- **THEN** the test suite fails

### Requirement: Re-pointing an application is an explicit, audited human action

Attaching a different resume version to an existing application SHALL be a permission-gated human
action recording actor, reason, previous version and new version. It SHALL emit no state transition and
SHALL NOT itself recompute a ranking.

*Source: `WF-005`'s mandatory-reason pattern; `config.yaml`'s rule that no surface writes a state field
directly. `design.md` D8 — whether a re-point forces a re-rank is `matching-and-ranking`'s to decide,
since `G-07` gives it versioned rankings that preserve prior boards.*

#### Scenario: Recruiter re-points an application

- **WHEN** a permitted user attaches a newer resume version to an application with a reason
- **THEN** the change is recorded with both versions and the actor, and no stage transition occurs

#### Scenario: Re-point with no reason

- **WHEN** a re-point is submitted with no reason
- **THEN** it is rejected with a field-level error

#### Scenario: Prior ranking after a re-point

- **WHEN** an application is re-pointed
- **THEN** the ranking computed against the prior version is preserved rather than deleted, and the
  fact that it was computed against a superseded version is visible

### Requirement: Superseded versions remain retrievable indefinitely

A resume version SHALL remain retrievable after later versions exist, after the application that
referenced it is closed, and after the posting it was submitted against is closed, subject to retention
policy.

*Source: `RET-002`, `RET-003`, `WF-007`, `BR-020`;
`ai-platform-governance`'s evidence-labeling requirement that a claim on a versioned record names the
version, which is only satisfiable if the named version stays resolvable.*

#### Scenario: Version referenced after supersession

- **WHEN** an AI run's input reference names version one and version three now exists
- **THEN** the reference resolves to version one's content

#### Scenario: Posting closed

- **WHEN** the posting a resume version was submitted against is closed
- **THEN** the version remains retrievable to authorized users

### Requirement: Resume version listing carries its provenance

A candidate's resume versions SHALL be listable with version number, source posting, uploader,
timestamp, active indicator, scan state, parsing state and confidence.

*Source: `RET-002`; §13.2's resume-version listing endpoint; `UI-003`'s requirement that a surface show
state and next action.*

#### Scenario: Versions listed

- **WHEN** an authorized user lists a candidate's resume versions
- **THEN** each entry carries its number, source posting, uploader, timestamp, active indicator and
  processing state

#### Scenario: Listing without permission

- **WHEN** a user without permission requests a candidate's resume versions
- **THEN** the request is denied rather than returning an empty list

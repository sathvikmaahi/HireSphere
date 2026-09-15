## Purpose

Immutability of approved job description versions, forking on edit, comparison of AI-generated
against human-edited content, and the pinning that keeps a live posting's ranking scores meaningful.
Owned by `TS-BL-036`.

## ADDED Requirements

### Requirement: An approved version is immutable

A job description version in the approved state SHALL NOT be modified. Any edit SHALL create a new
version in draft, carrying the next sequential version number.

*Source: `D12`'s retained consequence — "**Approved JDs fork on edit.** Editing an approved JD creates
a new version" — explicitly carried forward by `C-12`'s resolution. `TS-BL-033` creates version rows;
this requirement is what makes them immutable after approval (`design.md` D11).*

#### Scenario: Edit of an approved version

- **WHEN** an approved version is edited
- **THEN** a new version is created in draft with the next version number
- **AND** the approved version's content is unchanged

#### Scenario: Direct modification attempted

- **WHEN** a write is attempted against an approved version's content
- **THEN** it is rejected

#### Scenario: The new version needs its own approval

- **WHEN** a forked version is created from an approved one
- **THEN** it starts in draft and requires its own approval before any posting can link to it

### Requirement: A live posting stays pinned to the version it was published against

A posting's job description version reference SHALL be settable while the posting is in draft and
SHALL be immutable from the point the posting opens. Approving a later job description version SHALL
NOT change what an open posting references.

*Source: `D12`, retained by `C-12` — "live postings stay pinned to the version they were published
against." The reason is `domain-model.md`'s ranking tuple: `RankingScore = f(resume_version,
jd_version, prompt_template_version, model_version)`, without which "a score can't be explained or
reproduced once the JD or the model changes underneath it." `D12` states the failure directly —
edit-in-place "would silently invalidate every ranking score computed against that JD." See
`design.md` D9.*

#### Scenario: Later version approved while a posting is open

- **WHEN** a new version of a job description is approved while a posting referencing an earlier
  version is open
- **THEN** the open posting continues to reference the earlier version

#### Scenario: Re-pointing an open posting

- **WHEN** a change to an open posting's job description version reference is attempted
- **THEN** it is rejected

#### Scenario: Re-pointing a draft posting

- **WHEN** a posting still in draft is re-pointed to a different approved version
- **THEN** the change is accepted

### Requirement: Version history is retrievable and comparable

Version history SHALL be retrievable for the life of the job description, and SHALL support
comparison of AI-generated content against human-edited content for any version.

*Source: `JOB-006`; §13.2's version-listing endpoint. `design.md` D9 records why comparison is
load-bearing rather than a convenience — fork-on-edit makes a one-word correction a new version
requiring a new approval, and an approver needs to see exactly what changed to approve it cheaply.*

#### Scenario: Listing versions

- **WHEN** an authorized user lists a job description's versions
- **THEN** every version is returned with its number, state, author and approval metadata

#### Scenario: Comparing generated against edited

- **WHEN** a version carrying both AI-generated and human-edited content is compared
- **THEN** the differences between them are shown

#### Scenario: Comparing two versions

- **WHEN** two versions of one job description are compared
- **THEN** the differences between them are shown field by field

#### Scenario: Superseded version remains readable

- **WHEN** a version has been superseded and the posting that pinned it has closed
- **THEN** the version remains retrievable to authorized users

### Requirement: A version reference resolves to the content the reference was made against

A reference naming a job description version SHALL resolve to that version's content, including after
later versions exist.

*Source: `G-07`'s ranking-version adoption and `ai-platform-governance`'s evidence-labeling
requirement that "a claim resting on a versioned record names the version it was made against, since
`D12` forks job descriptions on edit" — an unversioned reference "would resolve to content the claim
was never made about."*

#### Scenario: Reference resolved after the job description has moved on

- **WHEN** a stored reference to version *n* is resolved after version *n+1* is approved
- **THEN** version *n*'s content is returned

#### Scenario: AI run reproducibility

- **WHEN** an AI run's input reference is resolved
- **THEN** it returns the exact version the run consumed

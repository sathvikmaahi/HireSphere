## Purpose

The job description record and the surface a Practice Manager drafts it in: a typed structured-input
schema covering every mandatory field, a declared length bound on every free-text section, and a
versioned draft from the first save. Owned by `TS-BL-033`.

## ADDED Requirements

### Requirement: Practice Managers create job descriptions

Authorized Practice Managers SHALL create job description records. The Recruiter role SHALL NOT hold
create or edit permission on job descriptions.

*Source: `JOB-001`; `C-12` as resolved, which removed the Recruiter from the JD flow per §8's listing
of JD Workspace users as Practice Managers and Recruitment Managers. `D12` originally had the
Recruiter drafting; that reading is superseded.*

#### Scenario: Practice Manager creates a job description

- **WHEN** an authorized Practice Manager submits a new job description
- **THEN** the record is created with the actor as creator
- **AND** an audit record is written in the same transaction

#### Scenario: Recruiter attempts to create a job description

- **WHEN** a user holding only the Recruiter role attempts to create or edit a job description
- **THEN** the request is denied by the permission evaluator
- **AND** the denial is a server-side decision, not only a hidden control

#### Scenario: Permission verdict comes from the central evaluator

- **WHEN** any job description surface checks whether an actor may act
- **THEN** the verdict is produced by the central permission evaluator rather than by logic local to
  this capability

### Requirement: Mandatory structured fields

A job description SHALL carry a typed structured-input schema. The schema SHALL cover job title,
practice, hiring manager, experience range, urgency, role summary, responsibilities, required skills,
preferred skills, and qualifications. Submission SHALL be rejected with field-level errors when a
mandatory field is absent or fails its type.

*Source: `JOB-002`; `design.md` D1, which fixes the schema as typed and validated rather than
freeform, because §16.4's permitted ranking signals are keyed to required-skill, preferred-skill and
experience fields that a freeform payload cannot supply.*

#### Scenario: Submission missing a mandatory field

- **WHEN** a job description is submitted with a mandatory field absent
- **THEN** it is rejected with a field-level error naming the field
- **AND** the errors also appear as a summary

#### Scenario: Experience range inverted

- **WHEN** a job description declares a maximum experience lower than its minimum
- **THEN** it is rejected with a field-level validation error

#### Scenario: Unknown field submitted

- **WHEN** a payload carries a field the schema does not declare
- **THEN** it is rejected rather than stored, so the structured input cannot accumulate undeclared
  content

### Requirement: Work location, work mode and employment type are JD-level defaults

The job description SHALL carry work location, work mode and employment type as **defaults that seed
a posting**. The posting's own values SHALL be authoritative for that posting, and a posting whose
values differ from its job description SHALL NOT be treated as inconsistent.

*Source: `JOB-002` lists these on the job description while §12.2 stores them on `job_postings`;
`domain-model.md` records one JD version to N postings, so two postings from one JD can legitimately
differ. See `design.md` D2.*

#### Scenario: Posting seeded from a job description

- **WHEN** a posting is created from an approved job description version
- **THEN** its work location, work mode and employment type are pre-filled from that version

#### Scenario: Posting diverges from its job description

- **WHEN** a posting's work location is changed to differ from its job description's
- **THEN** the change is accepted and the posting's value governs that posting

### Requirement: Vacancy mode is not a job description field

The job description SHALL NOT carry a vacancy count or an evergreen flag. Vacancy mode SHALL exist
only on the posting.

*Source: a deliberate, recorded divergence from `JOB-002`'s literal wording, in favour of §12.2's own
table layout, `JOB-003`/`JOB-004`'s posting-scoped phrasing, and `vacancy_slots` hanging off
`job_postings`. A count on a record reusable by N postings has no defined meaning. See `design.md`
D2.*

#### Scenario: Vacancy field submitted on a job description

- **WHEN** a job description payload carries a vacancy count or evergreen flag
- **THEN** it is rejected as an undeclared field

### Requirement: Every free-text field carries a declared bound

Every free-text field in the schema SHALL declare a maximum length as a sentence, item or character
count. Content exceeding a field's bound SHALL be rejected with a field-level validation error,
whether it was typed by a human or produced by a model. No free-text field SHALL exist without a
bound.

*Source: `S.5`, whose stated concern — an over-long paragraph in compact, information-dense type — is
produced identically by human and model authorship; `AGENTS.md`'s standing bar requiring a
machine-checkable bound rather than a style note; `ai-platform-governance` `design.md` D3, which
establishes that the bound is per free-text field rather than per output. See `design.md` D1.*

#### Scenario: Human-typed content exceeds a bound

- **WHEN** a Practice Manager types a role summary longer than its declared bound
- **THEN** the submission is rejected with a field-level error stating the bound

#### Scenario: List exceeds its item count

- **WHEN** more responsibilities are submitted than the field's declared item bound permits
- **THEN** the submission is rejected

#### Scenario: A field without a bound cannot ship

- **WHEN** the schema declares a free-text field with no bound
- **THEN** the test suite fails

#### Scenario: Bounds are configuration, not code

- **WHEN** a declared bound is changed through the audited configuration path
- **THEN** subsequent validation uses the new value with no deployment
- **AND** the change is audited with previous and new value

#### Scenario: Provisional bounds are marked as provisional

- **WHEN** a bound's provenance is inspected
- **THEN** it states whether the value is a confirmed product decision or a provisional default

### Requirement: A draft is a versioned record from the first save

Saving a job description SHALL produce a version record carrying a sequential version number.
Structured input, generated content and human-edited content SHALL be held on the version, not on
the parent record.

*Source: §12.2's `job_description_versions`; `ai-platform-governance` `design.md` D3, whose fixed
envelope takes `jd_draft_ref: <id@version>` — AI drafting cannot address a draft that is not already
a version. `design.md` D3 and D11 record this as a boundary refinement: the version row is created
here, while `TS-BL-036` owns immutability, fork-on-edit and comparison.*

#### Scenario: First save

- **WHEN** a new job description is saved for the first time
- **THEN** a version record is created with version number 1
- **AND** the job description references it as its current version

#### Scenario: Draft is addressable by version

- **WHEN** a draft is referenced by another capability
- **THEN** it is addressable as an identifier and version together, not as an identifier alone

### Requirement: JD Workspace surface

The JD Workspace SHALL present the structured job form, the current version's content, and the
actions available to the viewing actor. It SHALL show workflow state, owner, next action, blockers
where any exist, and last-updated time. Actions the actor cannot perform SHALL be absent rather than
disabled.

*Source: §14.2's Job Description Workspace row; `UI-003`; `design-system`'s app-shell requirement
that permission-aware affordances default to absent. The surface is assembled from existing form
controls, page templates and the dense data table — this capability introduces no new component.*

#### Scenario: Viewer without edit permission

- **WHEN** a user who may view but not edit opens the JD Workspace
- **THEN** editing affordances are absent from the rendering
- **AND** the API rejects an edit submitted directly, independently of the UI

#### Scenario: Validation errors surfaced

- **WHEN** a submission fails validation
- **THEN** errors appear at field level and in a summary

#### Scenario: State and next action visible

- **WHEN** a job description is displayed
- **THEN** its state, owner, next action and last-updated time are shown

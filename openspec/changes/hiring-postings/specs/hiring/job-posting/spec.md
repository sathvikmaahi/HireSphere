## Purpose

The job posting record: exactly one approved job description version, a vacancy mode of finite-with-a-
count or evergreen, ownership and recruiter and panel assignment, and the posting metadata that
scoping, filtering and reporting read. Owned by `TS-BL-037`.

## ADDED Requirements

### Requirement: A posting links to exactly one approved job description version

A posting SHALL reference exactly one job description version, and that version SHALL be in the
approved state at the time the reference is set.

*Source: `JOB-007`; `domain-model.md`'s spine, where one JD version fans out to N postings.*

#### Scenario: Posting created from an approved version

- **WHEN** a posting is created referencing an approved job description version
- **THEN** the posting is created with that reference

#### Scenario: Posting created from an unapproved version

- **WHEN** a posting is created referencing a draft or submitted version
- **THEN** the request is rejected

#### Scenario: Several postings from one version

- **WHEN** two postings are created from the same approved version
- **THEN** both are created, each with its own metadata

### Requirement: Vacancy mode is finite with a count, or evergreen

A posting SHALL carry a vacancy type of finite or evergreen. A finite posting SHALL require a
positive integer vacancy count. An evergreen posting SHALL carry no vacancy count and SHALL create no
fixed vacancy slots.

*Source: `JOB-003`, `JOB-004`; `D14`, which chose "evergreen is finite with n = unlimited" — one state
machine, one flag, one code path — over a rolling cap and over a separate object. This requirement
introduces the flag; nothing here reads it. Evergreen's lifecycle behavior is `TS-BL-070`. See
`design.md` D5.*

#### Scenario: Finite posting with no count

- **WHEN** a finite posting is created with no vacancy count
- **THEN** it is rejected with a field-level validation error

#### Scenario: Finite posting with a non-positive count

- **WHEN** a finite posting is created with a vacancy count of zero or less
- **THEN** it is rejected

#### Scenario: Evergreen posting

- **WHEN** an evergreen posting is created
- **THEN** it is accepted with no vacancy count
- **AND** no fixed vacancy slots are created for it

#### Scenario: Vacancy count supplied on an evergreen posting

- **WHEN** an evergreen posting is submitted carrying a vacancy count
- **THEN** it is rejected rather than the count being silently ignored

### Requirement: Vacancy mode is fixed once the posting opens

Vacancy type and vacancy count SHALL be editable while a posting is in draft or pending approval, and
SHALL be immutable from the point the posting opens.

*Source: `D14` left this open in its own words — "can a posting change type mid-life? Probably should
be blocked; needs an explicit rule" — and this is the capability that introduces the field.
`design.md` D12 records why immutability rather than a conditional override: finite-to-evergreen
orphans slots that `JOB-004` says evergreen cannot have, and evergreen-to-finite requires choosing a
count retroactively against a selection set `D14` records as unbounded.*

#### Scenario: Type changed while in draft

- **WHEN** a posting in draft is changed from finite to evergreen
- **THEN** the change is accepted

#### Scenario: Type changed after opening

- **WHEN** a change of vacancy type is attempted on an open posting
- **THEN** it is rejected

#### Scenario: Count raised after opening

- **WHEN** an increase to vacancy count is attempted on an open finite posting
- **THEN** it is rejected

### Requirement: Posting metadata

A posting SHALL carry posting title, practice, work location, work mode, employment type, priority
and target start date. Work location, work mode and employment type SHALL be seeded from the
referenced job description version and SHALL be authoritative for the posting once set.

*Source: §12.2's `job_postings` columns; `R.4`'s reconciliation note enumerating `work_mode`,
`employment_type`, `priority` and `target_start_date` as "posting metadata fields not previously
enumerated"; `design.md` D2 for why these are posting-authoritative rather than JD-authoritative.*

#### Scenario: Metadata seeded on creation

- **WHEN** a posting is created from an approved job description version
- **THEN** work location, work mode and employment type are pre-filled from that version

#### Scenario: Posting metadata edited

- **WHEN** a posting's priority or target start date is changed
- **THEN** the change is accepted and audited

### Requirement: Practice is a data attribute, not a permission boundary

A posting SHALL carry a practice value usable for filtering, routing and reporting. Practice SHALL
NOT be evaluated as an authorization boundary.

*Source: `D02` as resolved — visibility is governed by the matrix with an assignment-scoped predicate
for Interviewers and Hiring Panel Members, not by practice; `glossary.md` states the same. `D02`'s own
recommendation was to model Practice as a data attribute anyway, as "cheap insurance."*

#### Scenario: Filtering by practice

- **WHEN** a user filters postings by practice
- **THEN** postings are filtered on that attribute

#### Scenario: Practice does not restrict access

- **WHEN** an authorized user views a posting belonging to a different practice
- **THEN** access is decided by the permission evaluator alone, with no practice-based restriction

### Requirement: The practice vocabulary is audited configuration

The practice value list SHALL be held in the platform's audited runtime configuration, not compiled
into the application. A posting's practice SHALL be selected from that list and SHALL NOT be free
text. Adding, retiring or renaming a practice SHALL be audited with previous value, new value and a
mandatory reason. No unaudited setter SHALL exist for it.

*Source: `ENG-010`'s seed scripts for lifecycle values, §29's principle that operational values
change without code changes, and `platform-core`'s audited runtime-configuration registry — the
shape `interview-pipeline`'s `interview/shortlisting` already uses for disposition reason codes.
`design.md` D13 records why this rather than a reference table: free text fragments every group-by,
and a rename splits a practice's own history invisibly. Answers a question this feature deferred to
`insight-and-reporting`, whose `design.md` D12 supplied the reasoning.*

#### Scenario: Practice selected rather than typed

- **WHEN** a posting is created with a practice value not present in the configured list
- **THEN** it is rejected

#### Scenario: Vocabulary changed

- **WHEN** an authorized administrator retires a practice
- **THEN** the change is audited with previous and new value and a mandatory reason

#### Scenario: Unaudited setter sought

- **WHEN** the codebase is inspected for a path that changes the practice list without an audit
  record
- **THEN** none exists

#### Scenario: Retired practice on historical postings

- **WHEN** a practice is retired after postings have been recorded against it
- **THEN** those postings remain readable with the practice they were recorded under

### Requirement: Ownership, recruiter assignment and interview panel

A posting SHALL carry an owner, a set of assigned recruiters, and an interview panel. Assignment
SHALL require the Assign action, and the assigned recruiter set SHALL be the value the
assigned-postings scope predicate evaluates.

*Source: §12.2's `owner_id`, `recruiter_ids[]` and `interview_panel_ids[]`; §9.3's Assign action;
`access-control-and-admin`'s authorization requirement, which scopes write access on posting-bound
resources to "postings the user is assigned to" and cites `job_postings.recruiter_ids[]` by name.
`C-03` as resolved — writes scoped by posting assignment, reads governed by the matrix.*

#### Scenario: Scope predicate evaluated against real assignments

- **WHEN** a Recruiter attempts a write against a posting they are not assigned to
- **THEN** the write is denied by the assigned-postings scope predicate

#### Scenario: Reads are not assignment-scoped for Recruiters

- **WHEN** a Recruiter views a posting they are not assigned to
- **THEN** the read is permitted, because read scope is governed by the matrix rather than by
  assignment

#### Scenario: Assignment requires the Assign action

- **WHEN** a user without the Assign action attempts to add a recruiter to a posting
- **THEN** the request is denied

#### Scenario: Interviewer scoped to their own assignments

- **WHEN** a user holding only the Interviewer or Hiring Panel Member role reads postings
- **THEN** results are scoped to postings they are assigned to

### Requirement: Every posting write is audited

Creating or updating a posting SHALL write an audit record in the same transaction, carrying actor,
action, target, previous and new value, and correlation identifier. A failed audit write SHALL fail
the operation.

*Source: `domain-model.md`'s access-and-governance section; `config.yaml`'s standing rule that every
material write produces an audit record in the same transaction and a failed audit write fails the
operation; `D6`'s references-and-diffs-only rule.*

#### Scenario: Posting updated

- **WHEN** a posting field is changed
- **THEN** an audit record captures previous and new value in the same transaction

#### Scenario: Audit write fails

- **WHEN** the audit record for a posting write cannot be written
- **THEN** the write does not take effect

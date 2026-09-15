## Purpose

Interview notes are immutable by versioning rather than by freezing: every edit to a submitted note
creates a new version, forever, with no lock and no author-only rule — and the AI interview summary
drawn from those versions. Owned by `TS-BL-059`.

## ADDED Requirements

### Requirement: Every edit to a submitted note creates a new version

A submitted interview note SHALL NOT be silently edited. Any edit SHALL create a new version, and
every prior version SHALL remain readable with its own author, content and timestamp.

*Source: `INT-008`, `BR-017`, and `C-08`'s resolution of 2026-08-14, which **amends `D24`**:
"Immutability is achieved by **versioning, not freezing**: every edit to a submitted note creates a
new version (`INT-008`, `BR-017`), forever." §28 item 12 names note edit versions as an auditable
event. `design.md` D8 tabulates the three separate reversals `C-08` makes.*

#### Scenario: Note edited after submission

- **WHEN** a submitted note is edited
- **THEN** a new version is created and the prior version remains readable unchanged

#### Scenario: In-place update sought

- **WHEN** this capability is inspected for a path that overwrites a submitted note's content
- **THEN** none exists

#### Scenario: Current version identified

- **WHEN** a note with several versions is read
- **THEN** exactly one version is marked current and the others remain retrievable

### Requirement: Notes never lock, and editability is not a state

A note SHALL remain editable by a permitted actor for the life of the record. Approval of a scorecard,
completion of a round, or an Application reaching a terminal state SHALL NOT make a note uneditable.
No lock flag, freeze timestamp or approval-derived edit condition SHALL exist.

*Source: `C-08` — "No lock on scorecard approval, and no hard-coded author restriction" — which
reverses `D24`'s "all notes for a round freeze permanently once that round's scorecard is approved"
and its append-only-addendum consequence. `design.md` D8. The consequence of a post-approval edit is
supersession of the affected scorecard (`interview/scorecard-approval`), which is an effect, not a
prohibition.*

#### Scenario: Edit after scorecard approval

- **WHEN** a permitted actor edits a note whose Application has an approved scorecard
- **THEN** the edit succeeds and creates a new version

#### Scenario: Edit after a terminal state

- **WHEN** a permitted actor edits a note on an Application in a terminal state
- **THEN** the edit succeeds and creates a new version

#### Scenario: Lock mechanism sought

- **WHEN** this capability is inspected for a lock flag, freeze timestamp or approval-derived edit
  condition
- **THEN** none exists

#### Scenario: Addendum-only path sought

- **WHEN** this capability is inspected for an append-only addendum path replacing editing
- **THEN** none exists

### Requirement: Who may edit a note is decided by the permission matrix, not by authorship

Whether an actor may edit a note SHALL be determined by the `Edit` action on the Interview Console
page as evaluated by the central permission evaluator. No authorship comparison in code SHALL decide
it. Granting or denying that action through the matrix SHALL change who may edit with no code change.

*Source: `C-08` — "**Who may edit** is now a **permission-matrix question**, consistent with `C-02`
and `C-03` — the `Edit` action flag on the Interview Console page decides it, per role" — reversing
`D24`'s "only the authoring panelist may edit" and its "Recruiters and PMs are read-only on notes,
always". `AUTHZ-001` deny-by-default, `AUTHZ-005` direct denial overrides role grants, `ADM-005`'s
grant/deny/unset cells. `access-control-and-admin`'s proposal names this capability's note-edit matrix
control as a consumer of the evaluator. `design.md` D8.*

#### Scenario: Seeded matrix

- **WHEN** an actor holding the seeded permissions requests a note edit
- **THEN** the outcome follows the matrix verdict rather than whether they authored the note

#### Scenario: Non-author granted the action

- **WHEN** an administrator grants the note `Edit` action to a non-authoring role
- **THEN** that actor may edit with no code change, and the grant is audited

#### Scenario: Direct denial

- **WHEN** an actor holds the action by role but a direct denial is recorded against them
- **THEN** the edit is denied

#### Scenario: Authorship comparison in code

- **WHEN** this capability is inspected for a comparison of the actor against the note's author to
  decide editability
- **THEN** none exists

### Requirement: The note Edit action is seeded to the Interviewer role only

The permission matrix SHALL be seeded granting the Interview Console `Edit` action to the Interviewer
role and to no other role. The seeded value SHALL be a recorded, auditable configuration state.

*Source: `C-08` — "Recommended seeding: grant `Edit` on notes to Interviewer only" — and
`exploration-notes.md`'s open item "Seed `Edit` on interview notes to Interviewer only", now a matrix
decision. `Interaction B` records that seeded matrix values are a decision rather than a default and
must be reviewable rather than a silent script effect. `AUTHZ-008`'s least-privilege default for
interviewers and panel members.*

#### Scenario: Seeded state

- **WHEN** the seeded matrix is inspected
- **THEN** the Interview Console `Edit` action is granted to Interviewer and to no other role

#### Scenario: Seeding is recorded

- **WHEN** the seeded value is applied
- **THEN** it exists as a recorded, auditable configuration state rather than an unrecorded script
  effect

### Requirement: Every version records its own author and the note records its submitter

Each note version SHALL record the identity that created that version. The note SHALL retain the
identity that originally submitted it. Both SHALL remain readable on every version.

*Source: `INT-007`'s interviewer identity and timestamps, §12.2's `interview_notes.submitted_by`,
`BR-018`. `design.md` D8 records why this is the protection that survives `C-08`'s removal of the
author-only rule: an edit by a non-author must be visible as one rather than absorbed into the
original.*

#### Scenario: Non-author edit

- **WHEN** a permitted non-author edits a note
- **THEN** the new version records that actor as its author and the note still names its original
  submitter

#### Scenario: Version history read

- **WHEN** a note's version history is read
- **THEN** each version names who created it and when

### Requirement: A new note version is announced so dependent evaluations can respond

Creating a new note version SHALL publish a note-version-created event carrying the note reference, the
new version, the round and the Application, with the correlation identifier.

*Source: `C-08`'s recorded open consequence — "a note can change *after* the consolidated scorecard
drawn from it was approved, so an approved evaluation can silently stop matching its evidence" — whose
resolution is consumed by `interview/scorecard-approval`. `design.md` D3 decides that mechanism; this
requirement is the signal it depends on. §25's Interview Summary job type is triggered by the same
submission and edit path.*

#### Scenario: Version created

- **WHEN** a new note version is created
- **THEN** a note-version-created event is published naming the note, version, round and Application

#### Scenario: Event undelivered

- **WHEN** the event is not delivered
- **THEN** the version still exists and the note is still readable

#### Scenario: Duplicate delivery

- **WHEN** the same note-version-created event is delivered twice
- **THEN** its downstream effects occur once

### Requirement: An interview summary is generated only after a note is submitted

An AI interview summary SHALL be generated only once at least one note has been submitted for the
round. It SHALL reference the note versions it was drawn from. It SHALL be marked AI-generated and
SHALL require human review before being treated as anything other than advisory.

*Source: `INT-009`; §16.1's Interview Summary — trigger "after note submission", input "submitted
notes", output "interview-sourced summary", human gate "human review required"; §16.3's
`interview_note_summary` family, required output "structured summary with note references"; §25's
Interview Summary job type; `AI-001`, `UI-004`. `design.md` D1 records that this family had no owning
item in `D.10` and why it lands on this item rather than on the console.*

#### Scenario: No notes submitted

- **WHEN** a summary is requested for a round with no submitted note
- **THEN** it is refused

#### Scenario: Summary generated

- **WHEN** a summary is generated
- **THEN** it names the note versions it was drawn from and carries the AI-generated marking

#### Scenario: Note edited after a summary exists

- **WHEN** a new note version is created after a summary was generated
- **THEN** the summary is identifiable as drawn from a superseded note version rather than presented
  as current

### Requirement: Summary generation runs through the gateway on references and is bounded

Interview summary generation SHALL be invoked through the AI gateway in its fixed envelope as a new
version of the already-registered `interview_note_summary` family, carrying note references rather
than note content. It SHALL register the Interview Summary background job type against the shared
dispatch pattern. Every free-text field of its output contract SHALL declare a length bound; output
exceeding a bound SHALL be rejected as a contract violation and recorded as a failed run.

*Source: `ai-platform-governance`'s `ai-platform/ai-gateway` and its `design.md` D3, D7 and D10;
`AI-003`; §25's Interview Summary row; `S.5` and `AGENTS.md`'s standing bar requiring an explicit,
machine-checkable length bound. `design.md` Open Questions records that the concrete bounds are
provisional, per `S.5`'s instruction that the numbers were never given and should not be invented.*

#### Scenario: Envelope contents

- **WHEN** a summary request is inspected
- **THEN** it carries note references and no note text and no candidate personal data

#### Scenario: Bound exceeded

- **WHEN** generated output exceeds a declared bound
- **THEN** it is rejected as a contract violation, a failed run is recorded, and no content is
  persisted

#### Scenario: Bound absent

- **WHEN** the output contract is inspected
- **THEN** every free-text field declares a bound, and the bound's provisional provenance is
  machine-readable

#### Scenario: Provider call sought

- **WHEN** this capability is inspected for an outbound call to a model provider
- **THEN** none exists

### Requirement: Summary claims are interview-sourced and evidence-referenced

Every claim in an interview summary SHALL carry a source label from the platform's closed
evidence-source vocabulary and at least one reference to the note version it rests on. A claim with
neither SHALL be rejected. This capability SHALL derive no source value of its own.

*Source: `G-02`, `AI-007`, `BR-008`, `UI-005`; `ai-platform-governance`'s `ai-platform/evidence-labeling`
closed vocabulary, of which this is the first producer of `interview_note`-labelled output
(`design.md` D11).*

#### Scenario: Labelled claims

- **WHEN** a summary's claims are inspected
- **THEN** each carries a source label from the closed vocabulary and a note-version reference

#### Scenario: Unsupported claim

- **WHEN** generated output contains a claim with no evidence reference
- **THEN** it is rejected as a contract violation

#### Scenario: Source value derived locally

- **WHEN** this capability is inspected for its own definition of a source value
- **THEN** none exists

### Requirement: Note versions are retained and their disposal is recorded

Note versions SHALL be retained under the configured retention policy, and disposal SHALL itself be
recorded. Derived data drawn from a note version SHALL be discoverable from it.

*Source: `RET-006`, `RET-007` ("derived data such as summaries, embeddings, and scorecards shall be
linked to source data for deletion or re-indexing workflows"), `OD-004`, which owns the interval — it
is not invented here.*

#### Scenario: Derived data discoverable

- **WHEN** a note version is inspected
- **THEN** the summaries, embeddings and scorecards derived from it are discoverable from it

#### Scenario: Disposal recorded

- **WHEN** a note version is disposed of under the retention policy
- **THEN** the disposal is recorded and the records derived from it are reachable for the same
  treatment

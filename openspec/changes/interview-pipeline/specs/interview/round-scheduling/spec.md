## Purpose

Interview rounds against an Application: how many, of what type, with whom, and the assignment that
every downstream interview surface is scoped against. Owned by `TS-BL-057`.

## ADDED Requirements

### Requirement: An Application may have multiple interview rounds

An Application SHALL support many interview rounds against the same posting, each carrying its own
sequential round number, interview type, assigned interviewer, scheduled window and status.

*Source: `INT-001`, `INT-002`, `BR-009` ("a candidate can have many resumes, many posting
associations, and many interview rounds"), §12.2's `interview_rounds`. `D11` as amended by `C-05`
keeps evaluation per round in the notes while consolidating the scorecard.*

#### Scenario: Second round scheduled

- **WHEN** a second round is scheduled for an Application that already has one
- **THEN** it is created with the next sequential round number and both remain readable

#### Scenario: Round record contents

- **WHEN** a round is inspected
- **THEN** it carries interview type, assigned interviewer, scheduled start and end, status and its
  question-set reference

#### Scenario: Round on a different posting

- **WHEN** the same candidate has an Application against a second posting
- **THEN** rounds on the two Applications are independent and are not shared

### Requirement: Interview types come from a fixed enumeration

An interview round's type SHALL be one of the enumerated types. A type outside the enumeration SHALL
be rejected rather than stored as free text.

*Source: §12.2's `interview_rounds.interview_type` — technical, managerial, culture, panel, final,
other. `C-06` closed the "note templates per interview type" open item: `INT-006`'s note fields are
one fixed set across all types, so a type carries no template of its own. `design.md` Open Questions
records that extending the enumeration is deferrable for exactly that reason.*

#### Scenario: Enumerated type

- **WHEN** a round is created with an enumerated interview type
- **THEN** it is accepted

#### Scenario: Unenumerated type

- **WHEN** a round is created with a type outside the enumeration
- **THEN** it is rejected

#### Scenario: Note fields per type

- **WHEN** rounds of two different interview types are compared
- **THEN** both use the same fixed structured note field set

### Requirement: Scheduling a non-shortlisted candidate is blocked unless an override is recorded

An interview round SHALL NOT be created for an Application that is not shortlisted, unless an
authorized override is recorded with an actor and a reason. The override SHALL be audited.

*Source: `SHL-005`. The override is a recorded exception rather than a silent bypass, consistent with
`G-06`'s human-override-as-a-first-class-record and `BR-016`'s mandatory reason.*

#### Scenario: Non-shortlisted candidate

- **WHEN** a round is requested for an Application that has not been shortlisted
- **THEN** it is blocked

#### Scenario: Authorized override

- **WHEN** an actor holding the override permission schedules with a reason
- **THEN** the round is created and the override is recorded with actor, reason and timestamp

#### Scenario: Override without a reason

- **WHEN** an override is attempted with no reason
- **THEN** it is rejected

#### Scenario: Unauthorized override

- **WHEN** an actor without the override permission attempts one
- **THEN** it is denied server-side

### Requirement: Assignment defines the scope every interview surface is evaluated against

Assigning an interviewer to a round SHALL be the fact that grants that actor access to the round and
its candidate context. Assignment scope SHALL be applied as a query predicate, and a request for an
unassigned round SHALL be denied rather than returned empty.

*Source: `INT-003` — interviewers see only assigned interviews unless broader permission is granted —
`D01`'s assignment-scoped role as retained by `C-02`, `D02` as resolved by `C-03`, and
`access-control-and-admin`'s requirement that scope is applied as a query filter rather than by
discarding fetched rows. `matching-and-ranking`'s `matching/ai-context-visibility` applies the same
scope to its projection and depends on this assignment existing.*

#### Scenario: Assigned round

- **WHEN** an assigned interviewer lists their interviews
- **THEN** the round appears

#### Scenario: Unassigned round

- **WHEN** that interviewer requests a round they are not assigned to
- **THEN** the request is denied

#### Scenario: Denial distinguishable from emptiness

- **WHEN** a denied request is compared with a permitted request that matched nothing
- **THEN** the two responses are distinguishable

#### Scenario: Scope applied as a predicate

- **WHEN** a listing path is inspected
- **THEN** the scope is applied as a query predicate rather than by filtering fetched rows

### Requirement: Round scheduling publishes a round-created event with no subscriber of its own

Creating a round SHALL publish a round-created event through the platform's shared dispatch pattern,
carrying the correlation identifier. This capability SHALL register no subscriber for it.

*Source: §25's Question Generation job type, triggered by "shortlist or interview setup" — the
subscriber is `interview/interview-console` (`TS-BL-058`). `design.md` D1 records why generation lands
with the surface that renders it rather than here, and why publishing now avoids editing the
scheduling path later. The same producer-before-consumer seam `hiring-postings` used for
`posting.opened`.*

#### Scenario: Round created

- **WHEN** a round is created
- **THEN** a round-created event is published carrying the correlation identifier

#### Scenario: No subscriber registered here

- **WHEN** this capability's registered subscribers are enumerated
- **THEN** none subscribes to the round-created event

#### Scenario: Event undelivered

- **WHEN** the event is not delivered
- **THEN** the round still exists, is still viewable and is still conductable

### Requirement: Panelist assignment notifies internally and produces no calendar entry

Assigning an interviewer SHALL notify them through internal delivery. This capability SHALL create no
calendar event and send no message to the candidate.

*Source: `D04` (internal email early, the rest deferred), `D06`/`C-04` — calendar integration deferred,
with the explicitly accepted gap that "panelists receive an assignment email but no calendar entry;
they add the meeting themselves. Candidates are contacted manually, outside the system, until `OD-005`
resolves." `TS-BL-079` owns calendar integration; `TS-BL-078` owns candidate-facing communication.*

#### Scenario: Interviewer assigned

- **WHEN** an interviewer is assigned to a round
- **THEN** they receive an internal notification naming the candidate, posting, type and scheduled
  window

#### Scenario: Calendar integration sought

- **WHEN** this capability is inspected for a calendar API call
- **THEN** none exists

#### Scenario: Candidate messaging sought

- **WHEN** the messages produced by scheduling are enumerated
- **THEN** none is addressed to the candidate

### Requirement: Round status transitions are workflow-governed and reason-bearing where negative

A round's status SHALL move through the enumerated values by workflow transition. Cancellation and
no-show SHALL require a reason. A round's status change SHALL NOT be written directly.

*Source: §12.2's `interview_rounds.status` — scheduled, completed, cancelled, rescheduled, no_show —
`WF-001`, `WF-005`, `BR-018` (all status transitions timestamped and attributed).*

#### Scenario: Round cancelled

- **WHEN** a round is cancelled
- **THEN** a reason is required and the transition records actor, timestamp and reason

#### Scenario: Round rescheduled

- **WHEN** a round is rescheduled
- **THEN** the prior scheduled window remains readable on the transition history

#### Scenario: Direct status write

- **WHEN** this capability is inspected for a direct write to a round's status
- **THEN** none exists

### Requirement: Completion requires submitted notes

A round SHALL NOT be marked complete until at least one structured note has been submitted for it.

*Source: §13.2's `POST /api/interviews/{interviewId}/complete` — "mark interview complete when notes
are submitted"; `SCR-001`/`BR-010`, which make a **completed** round the precondition for a scorecard,
so a round completable without notes would make a scorecard drawable from no evidence.*

#### Scenario: Completion with notes

- **WHEN** completion is requested for a round with at least one submitted note
- **THEN** the round is marked complete

#### Scenario: Completion without notes

- **WHEN** completion is requested for a round with no submitted note
- **THEN** it is rejected and the round's status is unchanged

### Requirement: The first scheduled round advances the posting into interviewing

Scheduling the first interview round on a posting SHALL advance the posting's state from screening to
interviewing. The advance SHALL occur only where the posting is in screening, SHALL be attributed to
the actor who scheduled the round, and SHALL be a no-op for every subsequent round.

*Source: §11.1's `Screening → Interviewing` transition, which `hiring-postings`' `TS-BL-038` declares
and no feature invoked — `decision-and-offers` `design.md` D9a records the investigation and assigns
this advance to the item that owns its triggering event; `design.md` D10a adopts it. `WF-003`'s actor
attribution and `WF-004`'s correlation. The advance is attributed to a human actor inside that
human's action, which is what `hiring-postings`' requirement that no transition fires without an
authenticated actor demands. The posting's state is `job_postings.status`, distinct from the
Application states in §11.2 and from the round's own status.*

#### Scenario: First round on a posting

- **WHEN** the first interview round is scheduled on a posting in screening
- **THEN** the posting advances to interviewing, attributed to the scheduling actor

#### Scenario: Second round

- **WHEN** a further round is scheduled on the same posting
- **THEN** the posting's state is unchanged and no error is raised

#### Scenario: Posting already past interviewing

- **WHEN** a round is scheduled on a posting already in scorecard review or later
- **THEN** the scheduling succeeds and the posting is not moved backwards

#### Scenario: Round scheduled under an override

- **WHEN** the first round is scheduled through `SHL-005`'s authorized override, so the posting never
  entered screening
- **THEN** no advance is attempted and the posting's state is unchanged

#### Scenario: Advance without an actor

- **WHEN** the codebase is inspected for a posting advance that fires with no authenticated actor
- **THEN** none exists

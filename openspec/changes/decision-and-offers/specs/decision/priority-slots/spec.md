## Purpose

The selection arithmetic — up to five ranked selections per vacancy on a finite posting, the vacancy
slots those selections are ultimately promoted into, and the remainder of the Application lifecycle
from priority selection through to recruited. Owned by `TS-BL-064`.

## ADDED Requirements

### Requirement: Selections and vacancy slots are separate records

A finite posting SHALL carry up to `5 × n` priority selection records, each keyed to an Application,
and exactly `n` vacancy slot records, each keyed to the posting, where `n` is the posting's vacancy
count. A selection record SHALL NOT reference a vacancy slot. A vacancy slot SHALL reference at most
one Application, and only once that Application's offer has been accepted.

*Source: §12.2's two separate tables — `priority_selections` (application-keyed, carrying
`priority_rank`, `selection_tag`, `reason`) and `vacancy_slots` (posting-keyed, carrying
`slot_number`, `status`, `recruited_candidate_application_id`). `BR-012` caps the first; `G-13`'s
`open → reserved → filled` machine runs on the second. `SEL-002` scopes rank uniqueness to the
posting rather than to a vacancy, which is why a selection is not made against a particular vacancy.
`design.md` D2 records why merging the two satisfies `BR-012` and silently fails `CLS-001`.*

#### Scenario: Three-vacancy posting

- **WHEN** a finite posting with three vacancies has a full slate
- **THEN** fifteen selection records and three vacancy slot records exist

#### Scenario: Slot reference on a selection sought

- **WHEN** a selection record is inspected for a vacancy slot reference
- **THEN** none exists

#### Scenario: Two candidates joined

- **WHEN** two candidates have been recruited against a three-vacancy posting
- **THEN** two of the three vacancy slots are filled and the third is open

### Requirement: Vacancy slots are created by this capability, not by posting creation

Vacancy slot records SHALL be created for a finite posting by this capability. Posting creation SHALL
NOT create them.

*Source: `hiring-postings`' `TS-BL-037` task 5.1 — "**Do not create `vacancy_slots`** — slots are
reserved and filled by `TS-BL-064` and the offer path". `JOB-004` states an evergreen posting creates
no fixed slots.*

#### Scenario: Finite posting opened

- **WHEN** a finite posting reaches a state where selection is possible
- **THEN** one vacancy slot record exists per vacancy

#### Scenario: Evergreen posting

- **WHEN** an evergreen posting is inspected for vacancy slots
- **THEN** none exists

### Requirement: The selection cap is five times the effective vacancy count

The number of active selections on a finite posting SHALL NOT exceed five times the number of
non-cancelled vacancy slots. A selection that would exceed the cap SHALL be rejected, naming the cap
and the current count.

*Source: `SEL-001`, `BR-012`, `D20`'s "5 slots per vacancy". The count of non-cancelled vacancy slots
rather than `job_postings.vacancy_count` — the two are equal for every posting where no vacancy has
been cancelled, and `design.md` D3 records why the literal reading makes a posting with a cancelled
vacancy permanently uncloseable, since `hiring-postings` made the count immutable once a posting
opens.*

#### Scenario: Cap reached

- **WHEN** a sixteenth selection is attempted on a three-vacancy posting with fifteen active
  selections
- **THEN** it is rejected, naming the cap and the current count

#### Scenario: Vacancy cancelled

- **WHEN** one vacancy slot on a three-vacancy posting is cancelled
- **THEN** the cap becomes ten

#### Scenario: Removed selection does not consume the cap

- **WHEN** a selection is removed and a new one is made
- **THEN** the new selection is accepted, the removed one not counting against the cap

### Requirement: Evergreen selection is uncapped and unranked

Selection on an evergreen posting SHALL carry no cap and SHALL NOT require a priority rank. Rank
uniqueness SHALL remain enforced wherever a rank is present.

*Source: `D14`'s "**No 5-cap on evergreen** — selection is unbounded", `D20`'s "Evergreen postings
have no slots at all: selection there is a flat unranked list", and `SEL-002`, which is satisfied
where no rank is supplied.*

#### Scenario: Many evergreen selections

- **WHEN** more than five candidates are selected on an evergreen posting with no vacancy count
- **THEN** every selection is accepted

#### Scenario: Rank on an evergreen selection

- **WHEN** an evergreen selection is created with no priority rank
- **THEN** it is accepted

### Requirement: Priority rank is unique within the posting slate

Where a rank is supplied, it SHALL be unique among the posting's active selections. A rank collision
SHALL be rejected rather than resolved.

*Source: `SEL-002`. `design.md` D4 records why ties are not permitted: a tie makes "who gets the next
offer" undefined at the moment an offer is extended.*

#### Scenario: Duplicate rank

- **WHEN** a selection is created with a rank already held by an active selection on that posting
- **THEN** it is rejected

#### Scenario: Rank freed by removal

- **WHEN** a selection holding rank 3 is removed and a new selection claims rank 3
- **THEN** it is accepted

### Requirement: One candidate cannot hold two active selections on one posting

An Application SHALL hold at most one active selection on a posting, and a candidate SHALL hold at
most one active selection on a posting across their Applications to it.

*Source: `SEL-003`. `D23a`'s stated concern — "the same person can occupy **two of the five priority
slots** ... it corrupts the cap arithmetic" — is why the rule is evaluated on the candidate and not
only on the Application. `design.md` D10 records that this rule cannot prevent the unmerged-duplicate
case, which is what the duplicate warning below exists for.*

#### Scenario: Same Application selected twice

- **WHEN** a second active selection is attempted for an Application that already holds one
- **THEN** it is rejected

#### Scenario: Two Applications, one candidate, one posting

- **WHEN** a selection is attempted for a candidate who already holds an active selection on that
  posting through a different Application
- **THEN** it is rejected

### Requirement: Displacement and insertion re-sequence ranks as an audited reorder

Inserting a selection at an occupied rank, or removing one, SHALL re-sequence the affected ranks as a
single recorded operation carrying the actor, the prior and resulting order, and a mandatory reason.
Ranks SHALL NOT be renumbered without such a record.

*Source: `D20`'s "displacement allowed with a mandatory reason and audit entry" and "Displacement and
insertion require **re-sequencing ranks** — a reorder operation that must itself be audited, not a
silent renumber". `BR-018`.*

#### Scenario: Insertion at an occupied rank

- **WHEN** a candidate is inserted at rank 2 of a five-selection slate with a reason
- **THEN** the ranks below re-sequence and one reorder record is written carrying the prior and
  resulting order

#### Scenario: Reorder without a reason

- **WHEN** a re-sequencing operation is attempted with no reason
- **THEN** it is rejected

#### Scenario: Silent renumber sought

- **WHEN** the codebase is inspected for a path that changes a rank without writing a reorder record
- **THEN** none exists

### Requirement: Cancelling a vacancy evicts selections beyond the reduced cap

Cancelling a vacancy slot SHALL reduce the cap and SHALL evict selections beyond it, each eviction
carrying a mandatory reason and tagging the selection as selected-not-offered, with the remaining
ranks re-sequenced as an audited reorder.

*Source: `D20`'s "**Vacancy count reduction** (3 → 1) shrinks 15 slots to 5; the 10 evicted candidates
require a reason and flow into the **priority lane** — which is consistent, since 'selected but not
offered' is exactly what they are." `design.md` D3 records that the count-reduction trigger is
unreachable because `hiring-postings` made vacancy count immutable once a posting opens, and that
vacancy-slot cancellation is the mechanism that survives. `SEL-004`, `SEL-005`.*

#### Scenario: Vacancy cancelled with a full slate

- **WHEN** one vacancy on a three-vacancy posting with fifteen active selections is cancelled
- **THEN** five selections are evicted with reasons, tagged selected-not-offered, and the remaining
  ranks re-sequence

#### Scenario: Eviction without a reason

- **WHEN** a vacancy cancellation is attempted that would evict selections and no reason is supplied
- **THEN** it is rejected

#### Scenario: A filled vacancy is cancelled

- **WHEN** cancellation is attempted on a vacancy slot that is filled
- **THEN** it is rejected

### Requirement: Vacancy slots move through open, reserved, filled and cancelled

A vacancy slot SHALL move from open to reserved when an offer is accepted, and from reserved to
filled when the Application reaches recruited. It SHALL return to open where a reserved or filled
slot is released. No other transition SHALL be defined.

*Source: `G-13`'s explicit machine — `open ──accept──▶ reserved ──onboard + Hubble ID──▶ filled` — and
§12.2's `vacancy_slots.status` enum. The return to open is `G-13`'s renege path: "slot returns to
`open`; posting was never closed", and after `Filled` it pairs with the reopen transition in
`decision/posting-closure`.*

#### Scenario: Offer accepted

- **WHEN** an offer for a selected candidate is accepted
- **THEN** an open vacancy slot becomes reserved and references that Application

#### Scenario: Renege before closure

- **WHEN** a reserved slot's candidate reneges
- **THEN** the slot returns to open and the posting's state is unchanged

#### Scenario: Undefined transition

- **WHEN** a transition from filled directly to cancelled is attempted
- **THEN** it is rejected

### Requirement: The remainder of the Application lifecycle is registered here

This capability SHALL register the Application state machine's priority-selected-onward states,
their permitted transitions, the permission each demands, and a reason requirement on every negative
and terminal transition. The states SHALL be priority selected, offer in progress, offer accepted,
onboarding complete, recruited, selected not offered, offer declined and offer withdrawn. No other
capability in this feature SHALL declare a state or transition.

*Source: §11.2's lifecycle past `ScorecardReady`. `interview-pipeline` `design.md` D10 declared these
states "as reachable but not owned" and recorded that "their transitions are registered by
`decision-and-offers`". `WF-001` requires transitions to run through the workflow service; `WF-005`
requires a reason on terminal and negative states; `platform-core`'s workflow engine takes
declarative registrations and contains no domain logic. `design.md` D9 records why one declaration in
one item rather than four items each appending.*

#### Scenario: Machine registered

- **WHEN** this capability is deployed
- **THEN** the priority-selected-onward transitions are registered with the workflow service
- **AND** registering them required no change to the workflow framework

#### Scenario: Forward path complete

- **WHEN** an Application at scorecard-ready is inspected for available transitions
- **THEN** priority selected is among them

#### Scenario: Reason required on a negative transition

- **WHEN** a transition to selected not offered, offer declined or offer withdrawn is attempted with
  no reason
- **THEN** it is rejected

#### Scenario: A second declaration sought

- **WHEN** the other capabilities in this feature are inspected for state or transition declarations
- **THEN** none exists, and each invokes transitions instead

#### Scenario: Direct state write

- **WHEN** a code path writes an Application's status without going through the workflow service
- **THEN** the test suite fails

### Requirement: The first committed selection advances the posting into its selection state

Committing the first priority selection on a posting SHALL advance the posting's state from scorecard
review to selection. The advance SHALL occur only where the posting is in scorecard review, SHALL be
attributed to the actor who committed the selection, and SHALL be a no-op for every subsequent
selection.

*Source: §11.1's `ScorecardReview → Selection` transition, which `hiring-postings`' `TS-BL-038`
declares and no feature invokes — `design.md` D9a records the investigation and assigns this advance
to the item that owns its triggering event. `WF-003`'s actor attribution and `WF-004`'s
correlation. The advance is attributed to a human actor inside that human's action, which is what
`hiring-postings`' requirement that no transition fires without an authenticated actor demands.*

#### Scenario: First selection on a posting

- **WHEN** the first priority selection is committed on a posting in scorecard review
- **THEN** the posting advances to selection, attributed to the committing actor

#### Scenario: Second selection

- **WHEN** a second selection is committed on the same posting
- **THEN** the posting's state is unchanged

#### Scenario: Posting already past selection

- **WHEN** a selection is committed on a posting already in offer or onboarding
- **THEN** the selection succeeds and the posting is not moved backwards

#### Scenario: Posting not yet in scorecard review

- **WHEN** a selection is committed on a posting in an earlier state
- **THEN** the selection succeeds and no advance is attempted

### Requirement: Possible duplicates are surfaced on the selection surface as a warning

Where two Applications on one posting are flagged as possible duplicates, the flag SHALL be visible
where selection happens. It SHALL NOT block selection and SHALL NOT merge anything.

*Source: `D23a`'s posting-scoped check, surfaced as "a **non-blocking warning on the ranked list**
('possible duplicate of #7'), actioned by the Recruiter, never auto-merged at this stage — consistent
with the advisory-only principle". `design.md` D10 records that `SEL-003` cannot catch this case,
because it evaluates on candidate identity and two unmerged duplicate records disagree on exactly
that.*

#### Scenario: Flagged pair on the slate

- **WHEN** a possible-duplicate pair appears among a posting's selectable candidates
- **THEN** the flag is visible at the point of selection

#### Scenario: Selecting a flagged candidate

- **WHEN** a candidate carrying a duplicate flag is selected
- **THEN** the selection succeeds and the flag remains visible

#### Scenario: Auto-merge sought

- **WHEN** this capability is inspected for a path that merges candidates or Applications
- **THEN** none exists

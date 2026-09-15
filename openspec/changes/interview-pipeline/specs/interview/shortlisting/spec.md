## Purpose

The first place a human decides about a candidate: shortlist or reject, always with an accountable
actor and a recorded reason, executed as a workflow transition rather than a field write. Owned by
`TS-BL-056`.

## ADDED Requirements

### Requirement: Every disposition requires a reason

Marking a candidate shortlisted, removing them from a shortlist, or recording any negative
disposition on an Application SHALL require a reason. The reason SHALL be selected from a maintained
code list, with optional free text alongside it. A disposition submitted with no reason SHALL be
rejected and SHALL produce no state change.

*Source: `SHL-002` (reason on shortlist), `SHL-004` (reason on removal, history preserved), and
`WF-005` — transitions into terminal or negative states require a reason — which already covers every
`NotSelected` transition in §11.2 independently. The broader "reason on every disposition, from an
Admin-maintained code list plus optional free text" is a recommendation from `exploration-notes.md`'s
**"Recommendations made, not yet formally decided"** table and is **treated as adopted, not cited as
settled**; `design.md` D2 records the grounds and separates the settled half (the requirement itself,
carried by `WF-005`) from the unratified half (the coded vocabulary). `domain-model.md`'s spine
already renders this node as "Shortlist (reason required on every disposition)".*

#### Scenario: Shortlist without a reason

- **WHEN** a shortlist action is submitted with no reason
- **THEN** it is rejected with a field-level validation error and the Application's state is unchanged

#### Scenario: Rejection without a reason

- **WHEN** a candidate is dispositioned to a negative outcome with no reason
- **THEN** it is rejected and no state change occurs

#### Scenario: Coded reason with free text

- **WHEN** a disposition is submitted with a code from the maintained list and additional free text
- **THEN** both are stored on the transition record and appear in the audit trail

#### Scenario: Reason outside the maintained list

- **WHEN** a disposition supplies a reason code that the maintained list does not contain
- **THEN** it is rejected rather than stored as free text under an unknown code

### Requirement: The disposition reason vocabulary is audited configuration

The disposition reason code list SHALL be held in the platform's audited runtime configuration, not
compiled into the application. Adding, retiring or renaming a code SHALL be audited with previous
value, new value and a mandatory reason. No unaudited setter SHALL exist for it.

*Source: `ENG-010`'s seed scripts for lifecycle values, §29's principle that operational values change
without code changes, and `platform-core`'s audited runtime-configuration registry. `design.md` D2
records this as the specific property that makes the unratified half of the recommendation cheap to
reverse: rejecting the coded list costs a configuration change, not a revert.*

#### Scenario: Code list changed

- **WHEN** an authorized administrator retires a disposition code
- **THEN** the change is audited with previous and new value and a mandatory reason

#### Scenario: Unaudited setter sought

- **WHEN** the codebase is inspected for a path that changes the code list without an audit record
- **THEN** none exists

#### Scenario: Retired code on historical records

- **WHEN** a code is retired after dispositions have been recorded against it
- **THEN** those dispositions remain readable with the code they were recorded under

### Requirement: Dispositions are workflow transitions, never direct state writes

Every shortlist, removal and negative disposition SHALL be executed through the workflow service
against the registered Application state machine. No surface SHALL write an Application's `status`,
`shortlist_reason` or `final_outcome` field directly.

*Source: `WF-001`, `config.yaml`'s standing rule, and `platform-core`'s `platform/workflow-engine`
requirement whose scenario fails the suite on a direct state write. `matching-and-ranking`'s ranking
board carries the mirror requirement — it "decides nothing about a candidate's progress" — naming this
capability as where the decision belongs. `design.md` D10.*

#### Scenario: Direct state write attempted

- **WHEN** this capability is inspected for a write to an Application's state fields outside the
  workflow service
- **THEN** none exists

#### Scenario: Transition not permitted from the current state

- **WHEN** a shortlist is requested for an Application in a state the machine does not permit it from
- **THEN** the request is rejected and the response names the current state and the transitions
  available from it

#### Scenario: Attribution

- **WHEN** a disposition succeeds
- **THEN** the transition record carries the actor, prior state, new state, reason, timestamp and
  correlation identifier

### Requirement: The Application state machine is registered with the workflow service

This capability SHALL register the Application state machine declaratively, defining §11.2's states,
its permitted transitions, the permission each transition requires, and which transitions require a
reason. States owned by later features SHALL be declared as reachable so that a rejection can name
the transitions available from the current state.

*Source: `platform-core`'s `platform/workflow-engine` requirement "Declarative state machine
registration", whose scenario asserts the framework ships with no Application machine defined and
which names this feature as a registrant; §11.2's candidate-posting lifecycle; `WF-002`'s requirement
that a rejection explains itself. `design.md` D10 records why registration lands on this item rather
than being appended by each item that uses a transition.*

#### Scenario: Machine registered

- **WHEN** the workflow service is inspected after this capability is deployed
- **THEN** an Application state machine is registered covering §11.2's states, transitions,
  per-transition permissions and reason requirements

#### Scenario: Forward states declared

- **WHEN** an Application reaches the last state this feature owns
- **THEN** the transitions a later feature owns are named as available rather than the state
  appearing terminal

#### Scenario: Domain logic in the framework

- **WHEN** the workflow framework is inspected for Application-specific transition logic
- **THEN** none exists, and the behaviour comes from this registration

### Requirement: Removal from a shortlist preserves history

Removing a candidate from a shortlist SHALL preserve the prior shortlist record, its reason and its
actor. The removal SHALL be recorded as its own transition with its own reason and actor.

*Source: `SHL-004`, `BR-020` (candidate history preserved even when not selected), `WF-006`.*

#### Scenario: Candidate removed from shortlist

- **WHEN** a shortlisted candidate is removed with a reason
- **THEN** the original shortlist record and its reason remain readable
- **AND** the removal is recorded with its own actor, reason and timestamp

#### Scenario: Re-shortlisted after removal

- **WHEN** a previously removed candidate is shortlisted again
- **THEN** all three events remain individually readable in order

### Requirement: Shortlisting creates a scheduling task

A successful shortlist SHALL create a scheduling task for the recruiters responsible for the posting,
and the task SHALL be discoverable by them without a manual hand-off.

*Source: `SHL-003`. Delivery of the accompanying notification uses `platform-core`'s internal
notification engine (`D04` keeps internal delivery in early scope); no candidate-facing communication
is sent (`OD-005`, `C-04`).*

#### Scenario: Shortlist succeeds

- **WHEN** a candidate is shortlisted for a posting
- **THEN** a scheduling task is created for that posting's responsible recruiters

#### Scenario: Notification delivery unavailable

- **WHEN** notification delivery is unavailable at the time of shortlisting
- **THEN** the disposition still succeeds and the task still exists

#### Scenario: Candidate-facing message

- **WHEN** the messages produced by a shortlist are enumerated
- **THEN** none is addressed to the candidate

### Requirement: Disposition is permission-gated and evaluated server-side

Shortlisting, removal and negative disposition SHALL each require the permission the corresponding
transition declares, taken from the central permission evaluator. Controls SHALL be absent for actors
who lack the permission, and a directly submitted request SHALL be refused server-side.

*Source: `SHL-001` (authorized Practice Managers), `AUTHZ-003`, `AUTHZ-004` — a direct API call fails
even when the interface hides the control — `UI-002`, `SEC-005`, `C-02`'s nine roles and `C-03`'s
configurable read scope.*

#### Scenario: Actor without the permission

- **WHEN** an actor without the shortlist permission views a candidate
- **THEN** the shortlist control is absent rather than present and disabled

#### Scenario: Direct submission

- **WHEN** that actor submits a shortlist request directly against the API
- **THEN** it is refused server-side

#### Scenario: Denial distinguishable from an empty result

- **WHEN** a denied request is compared with a permitted request that matched nothing
- **THEN** the two responses are distinguishable

### Requirement: AI output cannot produce a disposition

No disposition SHALL be reachable from AI output alone. A recommendation, ranking or draft SHALL
require an authenticated actor holding the transition's permission before any state change occurs.

*Source: `AI-010`, `BR-007`, `project.md`'s governing rule, and `ai-platform-governance`'s
advisory-only capability. Stated here because this is the first capability in the product that
**has** the transitions the advisory-only guarantee is about; upstream it was guaranteed by there
being nothing to trigger.*

#### Scenario: Advisory output recommending rejection

- **WHEN** AI output recommends that a candidate not proceed
- **THEN** no state change occurs until an authorized human requests one with a reason

#### Scenario: AI Service Account attempts a disposition

- **WHEN** the AI Service Account requests a disposition transition
- **THEN** it is denied by the permission evaluator on the same terms as any other actor

#### Scenario: Path from AI output to a state field

- **WHEN** the codebase is inspected for a path from an AI output record to an Application state
  write
- **THEN** none exists

### Requirement: Every disposition is audited in the same transaction

Each disposition SHALL write an audit record in the same transaction as the state change, carrying
actor, action, target reference, previous and new value, reason and correlation identifier. A failed
audit write SHALL fail the disposition. The record SHALL carry references, never candidate personal
data.

*Source: `config.yaml`'s standing audit rule, `D6`'s references-not-content constraint, §28 item 11
(shortlist and shortlist removal are named auditable events), `access-control-and-admin`'s durable
audit writer.*

#### Scenario: Disposition recorded

- **WHEN** a disposition succeeds
- **THEN** an audit record exists carrying actor, action, target reference, previous and new value,
  reason and correlation identifier

#### Scenario: Audit write fails

- **WHEN** the audit record cannot be written
- **THEN** the disposition does not take effect

#### Scenario: Personal data in the audit record

- **WHEN** a disposition audit record is inspected
- **THEN** it contains references and values, and no candidate personal data

### Requirement: The first shortlist advances the posting into screening

Shortlisting the first candidate on a posting SHALL advance the posting's state from open to
screening. The advance SHALL occur only where the posting is in open, SHALL be attributed to the
actor who shortlisted, and SHALL be a no-op for every subsequent shortlist.

*Source: §11.1's `Open → Screening` transition, which `hiring-postings`' `TS-BL-038` declares and no
feature invoked — `decision-and-offers` `design.md` D9a records the investigation and assigns this
advance to the item that owns its triggering event; `design.md` D10a adopts it. `WF-003`'s actor
attribution and `WF-004`'s correlation. The advance is attributed to a human actor inside that
human's action, which is what `hiring-postings`' requirement that no transition fires without an
authenticated actor demands. The posting's state is `job_postings.status`, distinct from the
Application states in §11.2.*

#### Scenario: First shortlist on a posting

- **WHEN** the first candidate is shortlisted on a posting in open
- **THEN** the posting advances to screening, attributed to the shortlisting actor

#### Scenario: Second shortlist

- **WHEN** a second candidate is shortlisted on the same posting
- **THEN** the posting's state is unchanged and no error is raised

#### Scenario: Posting already past screening

- **WHEN** a candidate is shortlisted on a posting already in interviewing or later
- **THEN** the shortlist succeeds and the posting is not moved backwards

#### Scenario: Advance without an actor

- **WHEN** the codebase is inspected for a posting advance that fires with no authenticated actor
- **THEN** none exists

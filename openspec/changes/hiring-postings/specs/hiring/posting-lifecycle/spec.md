## Purpose

The posting state machine — registered declaratively against the platform's workflow engine, which
ships with none — together with the open-validation checklist that decides whether a posting may
open, and the posting-opened event that downstream matching subscribes to. Owned by `TS-BL-038`.

## ADDED Requirements

### Requirement: One state machine, registered declaratively

The posting lifecycle SHALL be registered as a declarative state machine defining states, permitted
transitions, reason requirements, and the permission each transition demands. The states SHALL be
draft, pending approval, open, screening, interviewing, scorecard review, selection, offer,
onboarding, filled, paused, cancelled and closed. A single machine SHALL govern both finite and
evergreen postings.

*Source: §11.1's posting lifecycle; `platform-core`'s workflow-engine requirement that domain features
register machines declaratively and that the framework contains no domain-specific transition logic —
this is the first machine registered in the product. `D14`'s "one state machine, one flag, one code
path" is why there is one machine and not two, and why there is no vacancy-type guard set; see
`design.md` D5.*

#### Scenario: Machine registered

- **WHEN** this capability is deployed
- **THEN** the posting state machine is registered with the workflow service
- **AND** registering it required no change to the workflow framework

#### Scenario: One machine covers both vacancy types

- **WHEN** an evergreen posting and a finite posting are transitioned
- **THEN** both are governed by the same registered machine

#### Scenario: Transition not defined from the current state

- **WHEN** a transition is requested that the machine does not define from the posting's current state
- **THEN** the request is rejected, naming the current state and the transitions available from it

#### Scenario: No surface writes the state directly

- **WHEN** a code path writes a posting's state without going through the workflow service
- **THEN** the test suite fails

### Requirement: No transition is automatic

Every transition this capability registers SHALL be initiated by an authenticated actor holding the
transition's declared permission. No transition SHALL fire on a timer, a threshold, a data condition,
or a configuration change.

*Source: `D14`'s split between the two vacancy types is entirely about automation — "finite =
automatic per rules plus manual; evergreen = manual only" — so a machine with no automation is
already correct for evergreen and leaves the finite automation to `TS-BL-069`/`TS-BL-070`, which is
where closure rules and evergreen lifecycle live. `platform-core`'s requirement that no caller
transitions by producing advisory output is the same principle from the AI side. See `design.md` D5.*

#### Scenario: Filled is reachable but not automatic

- **WHEN** this capability is deployed
- **THEN** filled is a registered state with no automatic transition into it

#### Scenario: Closed is reachable but not automatic

- **WHEN** this capability is deployed
- **THEN** closed is a registered state with no automatic transition into it

#### Scenario: An automatic transition cannot be added silently

- **WHEN** a transition is registered that fires without an authenticated actor
- **THEN** the test suite fails

### Requirement: Pending approval is a readiness gate, not a second content approval

The transition from draft to pending approval SHALL be the owner declaring the posting ready. The
transition from pending approval to open SHALL succeed only when open validation passes, and SHALL
otherwise be rejected naming what is unsatisfied. This capability SHALL NOT introduce a human approver
for posting content.

*Source: §11.1 defines the state; `JOB-009` defines what open validation checks; `C-12` placed the
content-approval gate on the job description, so "approval status" in `JOB-009`'s check list means the
linked job description version's approval status. `design.md` D6 records why the state is retained
rather than collapsed into validation-on-transition: §11.1 defines pending approval to cancelled, so
postings must actually be able to sit there, and `UI-003` requires blockers to be renderable.*

#### Scenario: Posting submitted as ready

- **WHEN** the owner transitions a draft posting to pending approval
- **THEN** the transition succeeds and the posting's blockers are shown

#### Scenario: Open attempted with an unsatisfied checklist

- **WHEN** opening is attempted while an open-validation item is unsatisfied
- **THEN** the transition is rejected and the response names each unsatisfied item

#### Scenario: No second approver exists

- **WHEN** the posting machine's transitions are enumerated
- **THEN** no transition requires a human approval of posting content

#### Scenario: Posting cancelled while pending approval

- **WHEN** a posting in pending approval is cancelled with a reason
- **THEN** the transition succeeds and the reason is recorded

### Requirement: Open validation checklist

Opening a posting SHALL verify required fields, vacancy mode, the linked job description version's
approval status, compliance text status, an owner, at least one assigned recruiter, and panel
configuration where required. Validation SHALL report every unsatisfied item, not the first.

*Source: `JOB-009`; `NFR-011`'s requirement that blocked actions are explained. Reporting all
unsatisfied items rather than the first is what makes the state renderable as a blocker list rather
than a sequence of retries.*

#### Scenario: All checks pass

- **WHEN** opening is requested and every checklist item is satisfied
- **THEN** the posting opens with an opened timestamp recorded

#### Scenario: Several items unsatisfied

- **WHEN** opening is requested with more than one checklist item unsatisfied
- **THEN** every unsatisfied item is reported

#### Scenario: Job description version not approved

- **WHEN** opening is requested and the linked job description version is not approved
- **THEN** the transition is rejected

### Requirement: Pause, reopen, cancel and close

An open posting SHALL be pausable and a paused posting SHALL be reopenable. Cancellation SHALL be
available from draft, pending approval, open, screening, interviewing, scorecard review, selection,
offer and onboarding. Closure SHALL be available from open and paused. Transitions into cancelled and
closed SHALL require a reason.

*Source: §11.1's transition set; `WF-005`'s reason requirement on terminal and negative states.
`D14`'s "manual pause/close only" for evergreen is satisfied by these transitions existing and
nothing automatic existing alongside them.*

#### Scenario: Pause and reopen

- **WHEN** an open posting is paused and later reopened
- **THEN** both transitions succeed and both are recorded

#### Scenario: Cancellation without a reason

- **WHEN** cancellation is requested with no reason
- **THEN** it is rejected with a field-level validation error and no state change occurs

#### Scenario: Closure from paused

- **WHEN** a paused posting is closed with a reason
- **THEN** the transition succeeds

### Requirement: History survives terminal states

A cancelled or closed posting SHALL retain its full transition history and SHALL remain viewable to
authorized users, continuing to inform candidate history and analytics.

*Source: `WF-006`, `WF-007`, `RET-003`; `platform-core`'s history-preservation requirement.*

#### Scenario: Cancelled posting inspected

- **WHEN** a cancelled posting is opened by an authorized user
- **THEN** its full transition history is readable

#### Scenario: Closed posting remains in analytics scope

- **WHEN** a posting is closed
- **THEN** it remains viewable and its records remain available to reporting

### Requirement: Opening a posting emits an event with no handler registered here

Opening a posting SHALL publish a posting-opened event through the platform's single dispatch pattern.
This capability SHALL register no subscriber. Failure to publish SHALL NOT roll back or block the
open transition.

*Source: `JOB-010`; §25, which lists Candidate Ranking and Existing Candidate Resurfacing as separate
job types both triggered by posting open. Both consumers are unbuilt — `matching-and-ranking`'s
`TS-BL-052` and `resurfacing-and-communications`' `TS-BL-075`. `design.md` D7 records why the trigger
is emitted here and why the transition must not depend on delivery: `G-12`'s degradation reasoning,
and that nothing requires candidate matching to be load-bearing for opening a posting.*

#### Scenario: Event published on open

- **WHEN** a posting opens
- **THEN** a posting-opened event is published through the shared dispatch pattern
- **AND** it carries the correlation identifier of the request that opened the posting

#### Scenario: No subscriber registered

- **WHEN** this capability is deployed
- **THEN** no subscriber to the posting-opened event is registered by it

#### Scenario: Publication fails

- **WHEN** publication of the posting-opened event fails
- **THEN** the posting remains open and the failure is visible rather than silently dropped

#### Scenario: Duplicate delivery

- **WHEN** the posting-opened event is delivered more than once
- **THEN** the contract requires the consumer to be idempotent, and the event carries what a consumer
  needs to detect a repeat

### Requirement: Every transition is permission-gated and attributed

Each registered transition SHALL declare the permission it requires, and SHALL be denied by the
permission evaluator before any state change when the actor does not hold it. Every transition SHALL
record actor, prior state, new state, reason where required, timestamp, correlation identifier and
originating module.

*Source: `WF-003`, `WF-004`, `AUTHZ-003`; `platform-core`'s per-transition permission and transition
record requirements. The evaluator is `access-control-and-admin`'s; this capability declares
requirements and consumes verdicts rather than evaluating anything.*

#### Scenario: Actor lacks the transition's permission

- **WHEN** an actor without a transition's declared permission requests it
- **THEN** it is denied before any state change

#### Scenario: Transition recorded

- **WHEN** a transition succeeds
- **THEN** actor, prior state, new state, timestamp, correlation identifier and source module are
  recorded, and the audit record is written

#### Scenario: Unauthenticated transition attempt

- **WHEN** a transition is requested with no authenticated actor
- **THEN** it is rejected and no state change occurs

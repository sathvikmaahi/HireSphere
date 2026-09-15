## Purpose

The Offer and Onboarding Tracker — four independent status tracks per selected candidate, carrying no
compensation amount and sending no message, covering the acceptance-to-joining gap where candidates
are most often lost. Owned by `TS-BL-067`.

## ADDED Requirements

### Requirement: One offer and onboarding record per selected Application

An offer and onboarding record SHALL exist for at most one Application and SHALL be creatable only for
an Application holding an active priority selection.

*Source: §12.2's `offer_onboarding_records.candidate_application_id`; `OFF-001` scopes the tracker to
"priority-selected candidates". §11.2 routes `PrioritySelected → OfferInProgress`, so selection is the
precondition for an offer.*

#### Scenario: Record created for a selected candidate

- **WHEN** an offer record is created for an Application holding an active selection
- **THEN** it is accepted

#### Scenario: Record for an unselected candidate

- **WHEN** an offer record is created for an Application with no active selection
- **THEN** it is rejected

#### Scenario: Second record

- **WHEN** a second offer record is created for an Application that already has one
- **THEN** it is rejected

### Requirement: Four status tracks move independently

Salary status, offer status, joining status and onboarding status SHALL each be recorded and updated
independently. No update to one SHALL imply or cascade an update to another.

*Source: §12.2's four separate enum columns rather than one status field. `G-14` added `joining_status`
specifically to cover "the acceptance-to-joining gap where candidates are most often lost", which only
exists as a distinct concern because acceptance and joining are tracked separately. `design.md` D5.*

#### Scenario: Salary finalized without an offer

- **WHEN** salary status is set to finalized while offer status is not started
- **THEN** it is accepted

#### Scenario: Cascade sought

- **WHEN** the codebase is inspected for a path that changes one track as a side effect of another
- **THEN** none exists

### Requirement: No compensation amount is stored

The tracker SHALL record salary status only. No field SHALL hold a salary figure, rate, band, currency
amount or equivalent.

*Source: `D19`'s decision — "status tracker only" — and its consequence: "**No compensation data**,
which means no extra access tier on top of `D16`. A meaningful simplification." Confirmed by §12.2,
whose `offer_onboarding_records` carries `salary_status` as an enum and no amount column, and by the
recommended non-goal "Offer letter generation, e-signature, comp amounts, and approval chains" which
the reference schema independently confirms. `design.md` D5 records why `OFF-001`'s "salary
finalization" is a status rather than a capability.*

#### Scenario: Amount field sought

- **WHEN** the record's fields are enumerated
- **THEN** none holds a compensation amount

#### Scenario: Amount supplied

- **WHEN** an update carrying a salary figure is submitted
- **THEN** it is rejected rather than the figure being stored in a free-text field

### Requirement: Salary and offer fields are permission-gated

Editing salary status and offer workflow fields SHALL be authorized through the platform's permission
evaluator. Authorization SHALL NOT be decided by comparing role names.

*Source: `OFF-002` — "Only authorized recruitment users shall edit salary and offer workflow fields";
`AUTHZ-003`'s server-side authorization on every endpoint; `C-02`'s nine roles and `ADM-005`'s
grant/deny/unset cells make the matrix the decision surface. `access-control-and-admin`'s evaluator is
consumed.*

#### Scenario: Unauthorized edit

- **WHEN** a user without the required permission updates offer status
- **THEN** it is rejected and the attempt is audited

#### Scenario: Role comparison sought

- **WHEN** this capability is inspected for a hard-coded role comparison in an authorization path
- **THEN** none exists

### Requirement: Terminal and negative updates require a reason, and a blocked track requires a blocker reason

Any update setting a track to a blocked, declined, withdrawn or no-show value SHALL require a reason.
Where any track is blocked, a blocker reason SHALL be present on the record.

*Source: `OFF-008` — "Terminal or negative status updates shall require a reason" — and `G-14`'s
`blocker_reason` "(required when blocked)". `WF-005` requires a reason on terminal and negative
transitions. The reason is held on the record as well as in the audit trail because `CLS-003`'s
checklist must render it.*

#### Scenario: Blocked without a reason

- **WHEN** onboarding status is set to blocked with no reason
- **THEN** it is rejected

#### Scenario: Declined offer

- **WHEN** offer status is set to declined with a reason
- **THEN** it is accepted and the reason is retained on the record

#### Scenario: Blocker reason readable

- **WHEN** a record with a blocked track is read
- **THEN** the blocker reason is present

### Requirement: Offer status drives the Application's state and the vacancy slot

Setting offer status to extended, accepted, declined or withdrawn SHALL invoke the corresponding
Application transition through the workflow service. Acceptance SHALL reserve a vacancy slot; a
decline, a withdrawal or a post-acceptance renege SHALL release it.

*Source: §11.2's `OfferInProgress → OfferAccepted | OfferDeclined | OfferWithdrawn`; `G-13`'s slot
machine — `open ──accept──▶ reserved` — and its renege path, "slot returns to `open`; posting was never
closed". `WF-001` requires transitions through the workflow service. Transitions are declared by
`decision/priority-slots` and invoked here.*

#### Scenario: Offer accepted

- **WHEN** offer status is set to accepted
- **THEN** the Application transitions to offer accepted and an open vacancy slot becomes reserved

#### Scenario: Offer declined after acceptance

- **WHEN** an accepted offer is subsequently reneged
- **THEN** the reserved slot returns to open and a reason is required

#### Scenario: Transition declared here

- **WHEN** this capability is inspected for a state or transition declaration
- **THEN** none exists

### Requirement: The first extended and first accepted offer advance the posting's state

Setting the first offer on a posting to extended SHALL advance the posting from selection to offer.
Setting the first offer on a posting to accepted SHALL advance the posting from offer to onboarding.
Each advance SHALL occur only where the posting is in the immediately preceding state, SHALL be
attributed to the actor whose update triggered it, and SHALL be a no-op for every subsequent offer.

*Source: §11.1's `Selection → Offer` and `Offer → Onboarding` transitions, which `hiring-postings`'
`TS-BL-038` declares and no feature invokes. `design.md` D9a records that `onboarding` was declared
and undriven, which left fulfilment closure guarding an unreachable state, and assigns these two
advances to the item that owns their triggering events. `WF-003`, `WF-004`. The posting's state is
`job_postings.status`, distinct from the Application states in §11.2 — `design.md` D9 records why the
two must not be conflated.*

#### Scenario: First offer extended

- **WHEN** the first offer on a posting in selection is set to extended
- **THEN** the posting advances to offer, attributed to the updating actor

#### Scenario: First offer accepted

- **WHEN** the first offer on a posting in offer is set to accepted
- **THEN** the posting advances to onboarding, attributed to the updating actor

#### Scenario: Fifth offer extended

- **WHEN** a further offer is extended on a posting already in offer or onboarding
- **THEN** the posting's state is unchanged and is not moved backwards

#### Scenario: Acceptance on a posting not in offer

- **WHEN** an offer is accepted on a posting whose state is not offer
- **THEN** the acceptance succeeds and no advance is attempted

#### Scenario: Advance without an actor

- **WHEN** the codebase is inspected for a posting advance that fires with no authenticated actor
- **THEN** none exists

### Requirement: No candidate-facing communication is sent

The tracker SHALL record the status of communication with a candidate and SHALL NOT send any message
to a candidate.

*Source: `OFF-001` names "candidate communication status"; `OD-005` blocks candidate-facing
communication pending legal sign-off and `C-04` defers the capability to
`resurfacing-and-communications`' `TS-BL-078`. `C-04` records the resulting manual channel as an
accepted gap. `design.md` D5.*

#### Scenario: Communication recorded

- **WHEN** a recruiter records that an offer was communicated
- **THEN** the status is stored

#### Scenario: Outbound candidate message sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no candidate-addressed message, template or delivery path

### Requirement: Every material update is audited

Every status change on the record SHALL be recorded through the platform's audit writer with the actor,
the field, the prior and new value, the reason where one is required, and a correlation identifier.

*Source: `API-003`'s audit on every material write; `BR-018`'s timestamped and attributed transitions;
`D6`'s audit design — references only, never raw personal data.*

#### Scenario: Status change audited

- **WHEN** joining status changes
- **THEN** an audit record exists carrying actor, field, prior and new value

#### Scenario: Personal data in an audit record

- **WHEN** an audit record written by this capability is inspected
- **THEN** it carries references rather than candidate personal data

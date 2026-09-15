## Purpose

Fulfilment closure for a finite posting — the closure checklist, the test that decides whether closing
as filled is available, the cascade that closure performs, and the audited reopen path for a renege
discovered after the posting reached filled. Owned by `TS-BL-069`.

## ADDED Requirements

### Requirement: Closing as filled is an actor-initiated action whose availability is computed

Closing a finite posting as filled SHALL be performed by an authenticated actor holding the required
permission. It SHALL NOT fire on a timer, a threshold, a data condition or a configuration change. The
availability of the action SHALL be computed from the fulfilment condition.

*Source: `CLS-002` and `CLS-003` state the rule as availability — "Close as Filled **shall be disabled
when**..." — and §13.2 exposes it as a route, `POST /api/postings/{postingId}/close-filled`, "Close
finite posting as filled after validation". `hiring-postings`' `hiring/posting-lifecycle` spec asserts
that every transition it registers is actor-initiated; this capability adds a transition with an actor
rather than contradicting that assertion. `project.md`'s rule that every material hiring decision rests
with an accountable human. `WF-001`, `WF-003`. `design.md` D7, which also records why the recommended
`FILLING` state is not built: §11.1's onboarding state already is it, as `G-13` states. The state this
transition leaves from is reachable only once the pipeline advances are driven — three by this feature
and three by `interview-pipeline` — which `design.md` D9a establishes and Open Questions tracks.*

#### Scenario: Condition met

- **WHEN** the fulfilment condition holds and an authorized actor closes the posting as filled
- **THEN** it succeeds and the posting reaches filled

#### Scenario: Condition not met

- **WHEN** closing as filled is attempted while the condition does not hold
- **THEN** it is rejected, naming what is unsatisfied

#### Scenario: Automatic close sought

- **WHEN** the codebase is inspected for a path that closes a posting with no authenticated actor
- **THEN** none exists

#### Scenario: A filling state sought

- **WHEN** the posting states are enumerated
- **THEN** there is no state between onboarding and filled

### Requirement: The fulfilment condition is recruited candidates equal to the effective vacancy count

The condition SHALL be that the number of Applications on the posting that have reached recruited —
each with onboarding complete and a present, unique Hubble ID — equals the number of non-cancelled
vacancy slots.

*Source: `CLS-001` and `BR-013` — a finite posting closes as filled only when recruited candidates with
onboarding complete and valid Hubble IDs equal the vacancy count. `G-13`, which supersedes the earlier
working assumption of closing on offers accepted equalling vacancy count, and whose comparison table
records the cost of that option as "phantom fills". The effective count is non-cancelled vacancy slots
rather than the immutable declared count — `design.md` D3 records that the literal reading makes a
posting with a cancelled vacancy permanently uncloseable with no operator remedy.*

#### Scenario: All vacancies recruited

- **WHEN** three of three non-cancelled vacancy slots are filled by recruited candidates
- **THEN** the condition holds

#### Scenario: Accepted but not joined

- **WHEN** three candidates have accepted offers and none has completed onboarding
- **THEN** the condition does not hold

#### Scenario: One vacancy cancelled

- **WHEN** one vacancy of three is cancelled and the other two are filled by recruited candidates
- **THEN** the condition holds

### Requirement: Closure is blocked while a selected candidate sits in an unresolved state

Closing as filled SHALL be blocked where the fulfilment condition does not hold and any selected
candidate remains in an unresolved salary, offer or onboarding state.

*Source: `CLS-003` — "Close as Filled shall be disabled when selected candidates remain in unresolved
salary, offer, or onboarding states and vacancy count has not been satisfied." `design.md` D7 records
that this is a condition about candidates who are **not** counted toward fulfilment, which is one
reason an automatic rule keyed on fulfilment alone would be wrong.*

#### Scenario: Unresolved selected candidate

- **WHEN** the fulfilment condition does not hold and a selected candidate's offer is in progress
- **THEN** closing as filled is blocked, naming that candidate's unresolved state

#### Scenario: Resolved and fulfilled

- **WHEN** the fulfilment condition holds and no selected candidate is in an unresolved state
- **THEN** closing as filled is available

### Requirement: The closure checklist and the closure guard share one predicate

A closure checklist SHALL be readable for a finite posting, showing missing Hubble IDs and blocking
workflow states. The checklist and the guard on the closure action SHALL be computed from the same
predicate.

*Source: `CLS-002` and `CLS-004` — "Closure checklist shall show missing Hubble IDs and blocking
workflow states." `UI-003`'s requirement that every page shows workflow state, owner, next action and
blockers. `design.md` D7 records why one predicate with two readers: a checklist reading "ready" while
the transition rejects the close is then structurally impossible rather than merely tested.*

#### Scenario: Missing Hubble ID

- **WHEN** a candidate has completed onboarding with no Hubble ID entered
- **THEN** the checklist names that candidate's missing identifier

#### Scenario: Checklist and guard agree

- **WHEN** the checklist reports the posting ready to close
- **THEN** the closure action succeeds

#### Scenario: A second predicate sought

- **WHEN** the codebase is inspected for a fulfilment computation used by only one of the two
- **THEN** none exists

### Requirement: Closure cascades to interview tasks, panelists, slot holders and the bench

Closing a finite posting as filled SHALL cancel that posting's pending interview tasks, notify the
affected panelists, tag remaining active selections as selected not offered while retaining their
selection reasons, and transition the posting's remaining non-terminal Applications to a terminal state
with a reason naming the closure. Each cascade effect SHALL be audited and attributed.

*Source: the "Closure semantics" recommendation in `exploration-notes.md`'s **Recommendations made, not
yet formally decided** table — "Closure must **cascade**: cancel pending interview tasks, notify
panelists, move slot holders to the priority lane, move bench to the resurfacing pool." `design.md` D1
names that status class explicitly and records why the cascade is treated as adopted: `WF-006` and
`WF-007` independently require a closed posting to retain history and remain informative, which is
unachievable with applications left in a live stage; `SEL-005` independently requires selected-not-
offered candidates to retain their reason and gain resurfacing precedence. `WF-004` attributes
system-generated transitions to a service account; `WF-005` requires the reason.*

#### Scenario: Pending interviews at closure

- **WHEN** a posting closes as filled with interview tasks still pending
- **THEN** those tasks are cancelled and their panelists are notified

#### Scenario: Unoffered slot holder

- **WHEN** a posting closes as filled with an active selection that never received an offer
- **THEN** that selection is tagged selected not offered with its original reason retained

#### Scenario: Bench application

- **WHEN** a posting closes as filled with candidates still at earlier stages
- **THEN** each is transitioned to a terminal state with a reason naming the closure

#### Scenario: History after closure

- **WHEN** a closed posting is read
- **THEN** its candidate and workflow history remain readable

#### Scenario: Cascade attribution

- **WHEN** a cascade transition is inspected
- **THEN** it is attributed to a service account and correlated to the closure

### Requirement: Closure publishes an event and registers no subscriber

Closure SHALL publish a posting-closed event through the platform's dispatch pattern, carrying the
posting reference, the closure kind, and references to the Applications the cascade terminated. This
capability SHALL register no subscriber and SHALL NOT build a resurfacing pool, a suggestion record or
a precedence ordering. Closure SHALL succeed whether or not publication succeeds.

*Source: the cascade's fourth consequence — "move bench to the resurfacing pool" — whose pool is a
`MatchSuggestion` set that `domain-model.md` describes as "lighter than `Application`, promoted into a
real `Application` only when a human acts on it", owned by `resurfacing-and-communications`'
`TS-BL-075`, which `D.10` makes depend on `TS-BL-069` for exactly this. `TS-BL-075` is the intended
consumer. The same shape as `hiring-postings`' unregistered `posting.opened` event. `G-12`'s
degradation reasoning is why closure does not depend on delivery. `design.md` D8.*

#### Scenario: Event published

- **WHEN** a posting closes as filled
- **THEN** a posting-closed event is published carrying the posting and the terminated Applications

#### Scenario: Subscriber sought

- **WHEN** this capability is inspected for a handler of the event it publishes
- **THEN** none exists

#### Scenario: Publication fails

- **WHEN** publication fails
- **THEN** the posting remains closed and the failure is recorded

#### Scenario: Resurfacing pool sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no suggestion pool, resurfacing engine or precedence ordering

### Requirement: A renege discovered after filled is handled by an audited reopen

A posting that has reached filled SHALL be reopenable by an authorized actor with a mandatory reason,
returning the posting to open, returning the affected vacancy slot to open, and clearing the closure
timestamp. The reopen SHALL be audited.

*Source: `G-13`'s explicit requirement — "The audited **reopen** path remains necessary for reneges
discovered after a posting does reach `Filled`" — and its comparison table, which records that in an IT
services context with notice periods and counter-offers "the renege row is not hypothetical". §11.1
defines no transition out of filled at all, so `design.md` D9 records this as an addition to the
posting machine, and records why it returns to open with one unguarded edge rather than a conditional
return to onboarding.*

#### Scenario: Renege after filled

- **WHEN** an authorized actor reopens a filled posting with a reason
- **THEN** the posting returns to open, the affected slot returns to open and the closure timestamp is
  cleared

#### Scenario: Reopen without a reason

- **WHEN** a reopen is attempted with no reason
- **THEN** it is rejected

#### Scenario: Reopen audited

- **WHEN** a reopen completes
- **THEN** an audit record names the actor, the posting, the affected slot and the reason

### Requirement: A reopen recomputes the posting's pipeline state in one step

On reopening a filled posting, the posting SHALL be advanced in a single recorded step to the furthest
pipeline state its live Applications justify, attributed to the reopening actor. The reopened posting
SHALL NOT be left in open where its Applications already justify a later state.

*Source: `design.md` D9a. The reopen returns the posting to open, and §11.1's pipeline advances are
each triggered by a first-occurrence event — so a reopen whose remaining vacancy is filled by a
candidate who has already accepted has no event left to fire and would strand the posting in open
permanently. An earlier draft of `design.md` D9 assumed the path back up was "re-traversable" without
establishing that anything traverses it; this requirement is what makes the reopen safe instead of
that assumption.*

#### Scenario: Reopen with an already-accepted candidate remaining

- **WHEN** a filled posting is reopened and a remaining candidate has already accepted an offer
- **THEN** the posting is advanced to onboarding rather than left in open

#### Scenario: Reopen with no live Applications past shortlisting

- **WHEN** a filled posting is reopened and no Application has progressed past shortlisting
- **THEN** the posting settles at the state those Applications justify and no further

#### Scenario: Recomputation attributed

- **WHEN** the recomputation moves the posting
- **THEN** the move is recorded and attributed to the reopening actor

#### Scenario: Recomputation repeated

- **WHEN** the recomputation is applied again with no Application having changed
- **THEN** the posting's state is unchanged

### Requirement: This capability closes as filled and does not perform the posting's manual close

Fulfilment closure SHALL end in the filled state. The manual close ending in the closed state SHALL
remain the posting lifecycle's own action and SHALL NOT be duplicated here.

*Source: §11.1 carries `Filled` and `Closed` as distinct states, with `Open → Closed` and
`Paused → Closed` registered by `hiring-postings`; §13.2 carries both
`POST /api/job-postings/{id}/close` and `POST /api/postings/{postingId}/close-filled`. `WF-007`'s
requirement that closed postings continue to inform candidate history and analytics depends on why a
posting ended being answerable. `design.md` D9.*

#### Scenario: Fulfilment closure

- **WHEN** a posting is closed through this capability's action
- **THEN** it reaches filled and not closed

#### Scenario: Manual close duplicated

- **WHEN** this capability is inspected for a manual close path
- **THEN** none exists

## Purpose

The separation-of-duties gate on job descriptions: the Practice Manager drafts and submits, the
Recruitment Manager approves or requests changes, the drafter never approves their own work, and the
fallback when no Recruitment Manager is assigned is another Practice Manager. Owned by `TS-BL-035`.

## ADDED Requirements

### Requirement: The Recruitment Manager approves; the Practice Manager drafts

A job description version SHALL move from draft through submitted to approved. Submission SHALL be
performed by an authorized Practice Manager; approval SHALL be performed by a Recruitment Manager.

*Source: `C-12` as resolved 2026-08-14 — `PM drafts (AI-assisted) → SUBMITTED → RM approves / requests
changes → APPROVED` — which **amends** `D12`. `D12`'s body still reads "Practice Manager approves"; that
reading is superseded and `C-12` governs. The Recruitment Manager gains a real function beyond
dashboards, and the person who owns the headcount is not the person who signs off on it.*

#### Scenario: Practice Manager submits for approval

- **WHEN** an authorized Practice Manager submits a draft version
- **THEN** the version moves to submitted and appears in the Recruitment Manager's review queue

#### Scenario: Recruitment Manager approves

- **WHEN** a Recruitment Manager approves a submitted version
- **THEN** the version becomes approved with approver and approval time recorded
- **AND** an audit record is written in the same transaction

#### Scenario: Practice Manager attempts to approve

- **WHEN** a user holding only the Practice Manager role attempts to approve a submitted version
- **THEN** the request is denied

### Requirement: The drafter never approves their own job description

Approval by the actor who created or last edited the version SHALL be refused, regardless of the
roles that actor holds.

*Source: `C-12`'s fallback clause — approval falls to another Practice Manager, "**never the
drafter**" — and `D12`'s rejection of self-approval on the grounds that it "would make 'human
approval' a formality." A person holding both Practice Manager and Recruitment Manager roles is the
case this requirement exists for.*

#### Scenario: Actor holds both roles

- **WHEN** the actor who drafted a version also holds the Recruitment Manager role and attempts to
  approve it
- **THEN** the request is denied on the drafter identity, not on the role

#### Scenario: A different Recruitment Manager approves

- **WHEN** a Recruitment Manager who did not draft or edit the version approves it
- **THEN** the approval succeeds

### Requirement: Fallback approver when no Recruitment Manager is assigned

Where no Recruitment Manager is available to approve, approval SHALL be permitted by another Practice
Manager who is not the drafter. No configuration SHALL permit the drafter to be the fallback.

*Source: `C-12`'s "**Fallback required**" clause. `D12` named the bottleneck as something that "needs
designing, not discovering" — this is the part of it `C-12` decided. A general delegation or
out-of-office model is not in scope; see `design.md` Non-Goals.*

#### Scenario: Fallback approval

- **WHEN** no Recruitment Manager is assigned and a Practice Manager other than the drafter approves
- **THEN** the approval succeeds and records that the fallback path was used

#### Scenario: Fallback cannot reach the drafter

- **WHEN** the fallback path is evaluated and the only available Practice Manager is the drafter
- **THEN** approval is refused and the blocker is reported, rather than the drafter being permitted

### Requirement: Request-changes loop with comments

A reviewer SHALL be able to request changes instead of approving, with a mandatory reason. The
version SHALL return to draft, remain editable by the drafter, and retain the review comments against
it.

*Source: `D12`'s consequence — "a review queue with an approve / request-changes loop, plus comments";
`WF-005`'s reason requirement on negative transitions, applied through the workflow service.*

#### Scenario: Changes requested

- **WHEN** a reviewer requests changes with a reason
- **THEN** the version returns to draft with the reason recorded and visible to the drafter

#### Scenario: Changes requested with no reason

- **WHEN** a request-changes action is submitted with no reason
- **THEN** it is rejected with a field-level validation error and no state change occurs

#### Scenario: Comment history retained

- **WHEN** a version is resubmitted after changes
- **THEN** prior review comments remain retrievable against it

### Requirement: The review queue is a work surface, not a notification

Submitted versions awaiting approval SHALL be listed for the actors permitted to approve them,
showing state, drafter, submission time and waiting time. Notification of a submission SHALL be
delivered through the platform's internal notification engine.

*Source: `D12`'s "new UI surface: a review queue"; `C-12`'s change that the queue belongs to the
Recruitment Manager rather than the Practice Manager; `D04`'s internal-delivery-only scope. The queue
uses `design-system`'s dense data table; no new component is introduced.*

#### Scenario: Reviewer opens the queue

- **WHEN** a Recruitment Manager opens the review queue
- **THEN** submitted versions awaiting their approval are listed with drafter, submission time and
  waiting time

#### Scenario: Notification on submission

- **WHEN** a version is submitted
- **THEN** an internal notification is delivered to the permitted approvers

#### Scenario: Queue respects permission

- **WHEN** a user without approval permission requests the queue
- **THEN** the request is denied rather than returning an empty list

### Requirement: Approval state transitions run through the workflow service

Every approval-state change on a job description version SHALL be executed through the workflow
service. No surface SHALL write the approval state directly.

*Source: `WF-001`, `WF-003`; `platform-core`'s workflow-engine requirement that no surface writes a
workflow-governed state field directly, and that transitions record actor, prior state, new state,
reason where required, timestamp, correlation identifier and originating module.*

#### Scenario: Direct state write attempted

- **WHEN** a code path writes a job description approval state without going through the workflow
  service
- **THEN** the test suite fails

#### Scenario: Invalid transition

- **WHEN** approval is requested on a version that is in draft rather than submitted
- **THEN** the request is rejected, naming the current state and the transitions available from it

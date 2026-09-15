## Purpose

The generic state transition framework through which every domain state change in TalentSphere
is executed, so that validation, reason enforcement, and actor attribution exist once rather
than being reimplemented per feature. This capability provides the framework only; individual
domain state machines are defined by the features that own them. Owned by `TS-BL-004`.

## ADDED Requirements

### Requirement: All transitions execute through the workflow service

Every state transition SHALL be executed through the workflow service. No surface SHALL write a
state field directly.

*Source: `WF-001`, `NFR-009`.*

#### Scenario: Transition requested through an API

- **WHEN** a client requests a state change on a workflow-governed resource
- **THEN** the workflow service evaluates and applies the transition, recording it

#### Scenario: Business rules are not duplicated in the client

- **WHEN** a transition that the UI would prevent is requested directly against the API
- **THEN** the workflow service rejects it on the same rule the UI applied

#### Scenario: Direct state write attempted

- **WHEN** a code path writes a workflow-governed state field without going through the service
- **THEN** the test suite fails

### Requirement: Invalid transitions are blocked and explained

Invalid transitions SHALL be blocked, and the rejection SHALL explain to the user why the
transition is not permitted from the current state.

*Source: `WF-002`, `NFR-011`.*

#### Scenario: Transition not defined from the current state

- **WHEN** a transition is requested that the resource's state machine does not define from its
  current state
- **THEN** the request is rejected
- **AND** the response names the current state and the transitions available from it

#### Scenario: Unknown transition name

- **WHEN** a transition name that no registered state machine defines is requested
- **THEN** the request is rejected rather than treated as a no-op

### Requirement: Transition records

Every transition SHALL record the actor, prior state, new state, reason where required,
timestamp, correlation identifier, and originating module.

*Source: `WF-003`, reference spec §26 Audit Correlation.*

#### Scenario: Successful transition

- **WHEN** a transition succeeds
- **THEN** a transition record captures actor, prior state, new state, timestamp, correlation
  identifier, and source module
- **AND** the corresponding audit record is written

#### Scenario: Audit write fails

- **WHEN** the audit record for a transition cannot be written
- **THEN** the transition does not take effect

### Requirement: Service account attribution

System-generated transitions SHALL be attributed to a service account and correlated to the
triggering workflow event, never to an arbitrary human user.

*Source: `WF-004`.*

#### Scenario: Background job transitions a resource

- **WHEN** a background job performs a transition
- **THEN** the transition record names the service account as actor and carries the identifier
  of the triggering event

### Requirement: Reason enforcement on terminal and negative states

Transitions into terminal or negative states SHALL require a reason. A state machine definition
SHALL be able to mark any transition as reason-required.

*Source: `WF-005`.*

#### Scenario: Negative transition without a reason

- **WHEN** a transition into a state marked reason-required is requested with no reason
- **THEN** the request is rejected with a field-level validation error and no state change
  occurs

#### Scenario: Reason recorded

- **WHEN** a reason-required transition succeeds
- **THEN** the reason is stored on the transition record and appears in the audit trail

### Requirement: History preservation

Transition history SHALL be retained for the life of the resource, including after the resource
reaches a cancelled, closed, or otherwise terminal state.

*Source: `WF-006`, `WF-007`.*

#### Scenario: Terminal resource remains inspectable

- **WHEN** a resource reaches a terminal state
- **THEN** its full transition history remains readable to authorized users

### Requirement: Declarative state machine registration

Domain features SHALL register their state machines declaratively, defining states, permitted
transitions, reason requirements, and the permission required per transition. The framework
SHALL NOT contain domain-specific transition logic.

*Registering a machine is how a feature such as `hiring-postings` or `interview-pipeline`
obtains transition behavior; the framework itself ships with none.*

#### Scenario: Framework ships without domain machines

- **WHEN** this capability is deployed
- **THEN** the workflow service is operational with no posting, Application, offer, or closure
  state machine defined
- **AND** registering a new state machine requires no change to the framework itself

#### Scenario: Permission enforced per transition

- **WHEN** an actor without the permission a transition declares attempts that transition
- **THEN** the request is denied by the permission evaluator before any state change

### Requirement: No transition capability without an authenticated permitted actor

The workflow service SHALL accept a transition only from an authenticated actor holding the
transition's declared permission. No caller SHALL be able to trigger a transition by virtue of
producing advisory output.

*Source: `AI-010`, `BR-007`, and `design.md` D10 — advisory-only is the product's central
promise, and it is enforced structurally rather than by review. This requirement is the
workflow-side half of that guarantee; the gateway-side half, that the AI gateway holds no
transition capability at all, is owned by `ai-platform-governance` (`TS-BL-029`).*

#### Scenario: Unauthenticated transition attempt

- **WHEN** a transition is requested with no authenticated actor
- **THEN** the request is rejected and no state change occurs

#### Scenario: Advisory output does not transition

- **WHEN** advisory output is produced that recommends a state change
- **THEN** no state change occurs until an authenticated actor holding the transition's
  permission requests it

### Requirement: Concurrent transition safety

Concurrent transition attempts on the same resource SHALL NOT produce a lost update or an
inconsistent state.

#### Scenario: Two simultaneous transitions

- **WHEN** two clients request different transitions on the same resource at the same time
- **THEN** exactly one succeeds
- **AND** the other is rejected with the resource's current state reported back

## Purpose

The absence of a capability, made checkable. No AI output can reject, shortlist, select, offer,
hire, onboard, or close a candidate — not because a policy forbids it, but because no code path
exists by which it could. This capability is the AI-side half of a guarantee whose workflow-side
and authorization-side halves are already specified by `platform-core` and
`access-control-and-admin`.

## ADDED Requirements

### Requirement: The gateway holds no transition capability

The AI gateway SHALL NOT be capable of causing a state transition. It SHALL hold no permission
that constitutes a human accountability decision, and SHALL expose no interface, direct or
indirect, through which a run's completion causes a workflow-governed state change.

*Source: `AI-010`, `BR-007`, `project.md`'s one rule that governs everything else — "enforced
structurally (the AI has no capability to trigger those transitions), not by convention or code
review"; inherited `D10`. Removing the capability makes the guarantee cheap to verify: there is no
code path to audit.*

#### Scenario: Output cannot transition state

- **WHEN** a run completes with output that recommends a decision
- **THEN** no state transition occurs as a result
- **AND** any state change requires a separate, permission-checked action by an accountable user

#### Scenario: Gateway holds no approval permission

- **WHEN** the gateway's own actor identity is evaluated for the Approve action on any page
- **THEN** access is denied, and the action cannot be granted to it

#### Scenario: Gateway never assumes a human identity

- **WHEN** the gateway persists output produced on behalf of a user's request
- **THEN** the output is attributed to the AI Service Account and the requesting user's identity is
  recorded as the requester, never as the actor of a decision

### Requirement: AI output is persisted only as advisory insight or draft content

Model output SHALL be persisted only as advisory insight or draft content. No AI output SHALL be
written to a workflow-governed state field, and no persistence path SHALL exist from a run's output
to such a field.

*Source: `AI-010`; `platform-core`'s workflow-engine requirement that the transition service is the
only path that writes a workflow-governed state field. That requirement closes the door from the
workflow side for every caller; this one establishes that AI output has no destination on the
other side of it.*

#### Scenario: Output lands in an advisory record

- **WHEN** a run produces valid contracted output
- **THEN** it is persisted as an insight or draft record carrying its AI marking

#### Scenario: No write to a governed state field

- **WHEN** persistence paths from run output are enumerated
- **THEN** none writes a field governed by a registered state machine

### Requirement: The guarantee is verified from every side, not assumed from one

The advisory-only guarantee SHALL be verified by an enumeration that asserts all of its enforced
halves together: that the workflow service accepts transitions only from an authenticated actor
holding the transition's declared permission, that the AI Service Account cannot hold Approve, and
that the gateway holds no transition capability. A failure of any one SHALL fail the verification.

*Source: `platform-core`'s `TS-BL-004` task 4.10 states the obligation directly — "assert no caller
can trigger a transition by producing advisory output — the workflow half of the advisory-only
guarantee that `ai-platform-governance`'s `TS-BL-029` completes."
`access-control-and-admin`'s service-account requirement carries a third half and names the other
two. Three partial assertions in three features, each passing its own suite, is precisely the
defect shape `platform-core` D4 identifies: independently-correct implementations that no test
compares. This requirement is the comparison.*

#### Scenario: All three halves asserted together

- **WHEN** the advisory-only verification runs
- **THEN** it asserts the workflow-side, authorization-side, and gateway-side guarantees in one
  pass and reports which failed

#### Scenario: One half regresses

- **WHEN** any single half is weakened — a transition accepted without an authenticated permitted
  actor, Approve becoming grantable to the AI Service Account, or a transition capability
  appearing on the gateway
- **THEN** the verification fails

#### Scenario: Advisory output recommending a decision

- **WHEN** advisory output is produced that recommends rejecting, shortlisting, selecting,
  offering, hiring, onboarding, or closing a candidate
- **THEN** no such change occurs until an authenticated actor holding the relevant permission
  performs it

### Requirement: Human decision is recorded as its own act

A decision taken after AI output SHALL be recorded as the human actor's decision, carrying that
actor and the reason its surface requires, and SHALL NOT be recorded as a confirmation of the AI
output.

*Source: `BR-016`, `SHL-002`, `SHL-004` — reason required on every disposition — and `D05`'s
accepted anchoring trade-off, whose mitigation only works if the human's act is a first-class
record rather than an acknowledgement of the model's. `G-06`'s override record covers divergence;
this covers agreement, which is the case that would otherwise leave no trace of a human having
decided anything.*

#### Scenario: Human agrees with AI output

- **WHEN** a user takes the action AI output recommended
- **THEN** the record names the human actor as the decision-maker and carries their reason
- **AND** the AI output is referenced as input, not as authority

#### Scenario: Decision without a preceding run

- **WHEN** a user takes a decision with no AI output involved
- **THEN** the record has the same shape, with no AI reference

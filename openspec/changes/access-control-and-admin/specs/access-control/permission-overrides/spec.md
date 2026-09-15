## Purpose

Direct per-user grants and denials that operate independently of role membership: the exception
mechanism for the cases a role cannot express, always carrying a reason, always audited, and always
losing to nothing except themselves. Owned by `TS-BL-024`.

## ADDED Requirements

### Requirement: Per-user overrides independent of role membership

The system SHALL support direct per-user grants and denials that apply regardless of which roles
the user holds.

*Source: `AUTHZ-002` — access may be granted "by role, direct assignment, or approved exception";
`reference/spec.md` §12.2 `user_permission_overrides`. Inherited from
`talentsphere-wave-1-foundation`'s `access-control/authorization`; relocated to its own capability
because D.9 makes the override flow an independently deployable item (`design.md` D2).*

#### Scenario: Grant with no supporting role

- **WHEN** a user holds no role granting an action and an administrator creates a direct grant for
  it
- **THEN** the user may perform the action

#### Scenario: Override survives role change

- **WHEN** a user's roles are reassigned
- **THEN** existing direct overrides continue to apply until explicitly removed

#### Scenario: Override removed

- **WHEN** a direct override is removed
- **THEN** the user's verdict reverts to what their roles alone produce, on the next request

### Requirement: Every override carries a mandatory reason

Every override SHALL require a reason. An override submitted without one SHALL be rejected and no
override SHALL be created.

*Source: `reference/spec.md` §12.2 `user_permission_overrides.reason` — "Mandatory";
`AUTHZ-007`. An exception nobody had to justify is indistinguishable from a mistake six months
later.*

#### Scenario: Override without a reason

- **WHEN** an administrator creates a user-level override with no reason supplied
- **THEN** the request is rejected with a field-level validation error and no override is created

#### Scenario: Reason is retrievable

- **WHEN** an existing override is inspected
- **THEN** its reason, its creator, and its creation time are present

### Requirement: A direct denial is absolute

A direct user-level denial SHALL override every grant that would otherwise apply, from any role and
from any direct grant.

*Source: `AUTHZ-005`, `ADM-006`. This is the precedence rule the evaluator implements
(`access-control/authorization`); stated here as the property the override flow must not allow an
administrator to defeat by any ordering of edits.*

#### Scenario: Denial and grant for the same permission

- **WHEN** a user holds both a direct grant and a direct denial for the same page and action
- **THEN** access is denied

#### Scenario: Denial cannot be superseded by adding a role

- **WHEN** a user under a direct denial is assigned a role granting that same action
- **THEN** access remains denied

### Requirement: Every override change is audited with both values

Creating, changing, or removing an override SHALL produce an audit record carrying the acting
administrator, the subject user, the previous value, the new value, the timestamp, the reason, and
the affected page and action.

*Source: `AUTHZ-007`, `ADM-007`. The audit substrate that stores it is `platform/audit-trail`.*

#### Scenario: Override created

- **WHEN** an administrator creates an override
- **THEN** an audit record captures the previous state as unset, the new value, the subject, the
  reason, and the affected page and action

#### Scenario: Override removed

- **WHEN** an administrator removes an override
- **THEN** an audit record captures the previous value and the new state as unset

#### Scenario: Failed audit fails the change

- **WHEN** the audit record for an override change cannot be written
- **THEN** the override change does not take effect

### Requirement: Overrides take effect immediately

An override change SHALL affect the subject user's next request, without requiring the subject to
re-authenticate and without any interval during which a stale verdict is served.

*Source: `AUTHZ-007`'s immediate effect; `config.yaml`'s uncached-evaluator rule. Restated here
because the override is the mechanism an administrator reaches for during an incident, which is
exactly when a delay is least acceptable.*

#### Scenario: Denial applied to an active session

- **WHEN** an administrator creates a direct denial for a user holding an active session
- **THEN** that user's next request for the affected action is denied

#### Scenario: Grant applied to an active session

- **WHEN** an administrator creates a direct grant for a user holding an active session
- **THEN** that user's next request for the affected action succeeds

### Requirement: An override cannot grant what is structurally prohibited

An override SHALL NOT be able to grant a permission the model prohibits, including Approve to a
service account.

*Source: `AI-010`, `BR-007`, `C-02`'s AI Service Account. The advisory-only guarantee is structural
rather than configurational (`project.md`); an override path that could grant Approve to the AI
Service Account would make it configurational.*

#### Scenario: Approve to a service account through an override

- **WHEN** an administrator creates a direct grant of Approve for a service account
- **THEN** the request is rejected

#### Scenario: Prohibitions enumerated

- **WHEN** the structurally prohibited assignments are enumerated
- **THEN** each is rejected by both the matrix path and the override path

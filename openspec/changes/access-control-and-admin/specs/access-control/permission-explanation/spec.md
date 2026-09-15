## Purpose

The administrator-facing account of any authorization verdict — why a particular user can or cannot
perform a particular action on a particular page — produced by the same evaluation pass that
produced the verdict, so the two can never disagree. Owned by `TS-BL-019`.

## ADDED Requirements

### Requirement: Permission explanation for any subject, page, and action

The system SHALL expose to authorized administrators an explanation of any permission verdict,
identifying the contributing role grants, direct grants, and explicit denials, and naming the
deciding factor.

*Source: `AUTHZ-006`, `ADM-006`. Inherited from `talentsphere-wave-1-foundation`'s
`access-control/authorization`; relocated to its own capability because D.9 makes the explanation
endpoint an independently deployable item (`design.md` D2).*

#### Scenario: Explaining a denial

- **WHEN** an administrator asks why a specific user cannot access a specific page and action
- **THEN** the response identifies the deciding factor, including which denial overrode which
  grant, and which roles were consulted

#### Scenario: Explaining a grant

- **WHEN** an administrator asks why a user does have access
- **THEN** the response names the role or direct assignment that granted it

#### Scenario: Explaining an absence

- **WHEN** the verdict is a denial because no grant of any kind applies
- **THEN** the response states that no applicable grant exists, distinguishing it from an explicit
  denial

### Requirement: Explanation and verdict come from one evaluation pass

The explanation SHALL be produced by the same evaluation that produces the verdict, and SHALL NOT
be reconstructed by a separate code path.

*Source: `design.md` D1, inherited from `talentsphere-wave-1-foundation`'s D1. An explanation
derived separately will eventually disagree with the decision, at which point it is worse than no
explanation — it sends an administrator to change the wrong cell.*

#### Scenario: Explanation agrees with the verdict

- **WHEN** any explanation is produced
- **THEN** the verdict it explains is the verdict the same request would receive

#### Scenario: No second derivation path

- **WHEN** the code paths producing authorization explanations are enumerated
- **THEN** each originates from an evaluation that also produced a verdict

### Requirement: Explanation is itself permission-gated and discloses nothing extra

Access to the explanation surface SHALL require an administrative permission. The explanation
SHALL disclose only permission configuration, and SHALL NOT disclose candidate, posting, or other
recruitment data about the subject or the resource.

*Source: `AUTHZ-006` scopes the explanation to administrators; `ADM-008` and `D16` — an
administrator is configuration-only and the explanation surface must not become the exception.*

#### Scenario: Non-administrator requests an explanation

- **WHEN** a user without the administrative permission calls the explanation endpoint
- **THEN** the request is denied and no configuration data is returned

#### Scenario: Explanation payload inspected

- **WHEN** an explanation for a candidate-data page is returned
- **THEN** it names roles, grants, denials and scopes only
- **AND** it contains no candidate personal data

### Requirement: Users can see their own effective permissions

A user SHALL be able to retrieve their own effective permissions without holding an administrative
permission, and SHALL NOT be able to retrieve another user's.

*Source: `AUTH-004` requires the session to carry resolved page permissions, and
`identity-and-access`'s `GET /api/auth/me` returns "the caller's own identity, roles, permissions
and session state and never another user's". This requirement states the boundary that
introspection stops at self.*

#### Scenario: Own permissions

- **WHEN** a user requests their own effective permissions
- **THEN** the response reflects the same verdicts the evaluator would produce for their requests

#### Scenario: Another user's permissions

- **WHEN** a user without the administrative permission requests another user's permissions or
  explanation
- **THEN** the request is denied

### Requirement: Scope is explained, not only the flag

Where a verdict is decided or narrowed by a data scope predicate, the explanation SHALL name the
predicate and its effect.

*Source: `C-03`'s scoped writes and configurable reads. A "granted" verdict that a scope
nevertheless narrows is the case an administrator most often misreads, because the matrix cell
shows a grant.*

#### Scenario: Granted flag, narrowed by scope

- **WHEN** a user holds a granted action whose scope excludes the resource in question
- **THEN** the explanation states that the action is granted and that the scope predicate excluded
  the resource

#### Scenario: Scope named in a positive verdict

- **WHEN** an explanation reports a grant carrying a scope predicate
- **THEN** the predicate is named alongside the grant

## Purpose

The single authorization model for TalentSphere: how role grants, direct grants, explicit denials
and data scopes combine into one deny-by-default verdict, and how that verdict is enforced on every
page, endpoint and background actor. There is exactly one evaluator, and this capability is it.
Owned by `TS-BL-018`.

## ADDED Requirements

### Requirement: Deny-by-default authorization

The system SHALL deny every page and action unless access is granted by role, by direct assignment,
or by an approved exception.

*Source: `AUTHZ-001`, `AUTHZ-002`. Inherited from `talentsphere-wave-1-foundation`'s
`access-control/authorization`; path preserved per `design.md` D2.*

#### Scenario: No applicable grant

- **WHEN** a user requests a page or action for which no role grant, direct grant, or exception
  exists
- **THEN** access is denied

#### Scenario: Newly introduced page

- **WHEN** a new page or action is added to the permission catalog
- **THEN** it is denied to every role until explicitly granted

### Requirement: Nine action flags

The permission model SHALL support at least the action flags View, Create, Edit, Delete, Approve,
Run AI, Export, Assign, and Administer, evaluated per page.

*Source: `reference/spec.md` §9.3; `C-02`'s resolution, which notes that making `Run AI` a distinct
flag is what makes "who may trigger AI" itself controllable.*

#### Scenario: Run AI is independently controllable

- **WHEN** a user holds View on a page but not Run AI
- **THEN** the user may read the page and is denied any action that triggers an AI task

#### Scenario: Approve is independently controllable

- **WHEN** a user holds Edit but not Approve on a page
- **THEN** the user may modify content and is denied the approval action

### Requirement: Grant, deny, and unset assignment states

Each permission assignment SHALL be expressible as grant, deny, or unset, where unset means "no
opinion" and contributes nothing to the verdict.

*Source: `ADM-005`.*

#### Scenario: Unset does not grant

- **WHEN** a role holds unset for an action and no other grant applies
- **THEN** access is denied

#### Scenario: Unset does not deny

- **WHEN** one of a user's roles holds unset for an action and another role holds grant
- **THEN** access is granted

### Requirement: Explicit denial overrides grants

An explicit denial SHALL override any role-based or direct grant, at both role and user level.

*Source: `AUTHZ-005`, `ADM-006`.*

#### Scenario: Direct denial beats role grant

- **WHEN** a user's role grants an action and a direct user-level denial exists for that same action
- **THEN** access is denied

#### Scenario: Denial in one of several roles

- **WHEN** a user holds two roles, one granting and one explicitly denying the same action
- **THEN** access is denied

### Requirement: One evaluator, consulted by every actor

Every page, every API endpoint, and every background actor including the AI Service Account SHALL
obtain its verdict from the same evaluator. No access decision SHALL be derived anywhere else,
including from role assignments directly.

*Source: `config.yaml`'s standing architectural rule; `NFR-009`, which requires business rules
centralized in workflow and authorization services rather than duplicated. `design.md` D1 records
why decorators alone are insufficient — they cover endpoints and cannot reach a background job.*

#### Scenario: Decision paths enumerated

- **WHEN** the access decisions made anywhere in the system are enumerated
- **THEN** each resolves through the single evaluator
- **AND** none is derived from role assignments directly

#### Scenario: Background actor evaluated identically

- **WHEN** a background job runs under a service account and attempts an action
- **THEN** its verdict is produced by the same evaluator, under the same rules, as an interactive
  request

### Requirement: Server-side enforcement on every surface

Every page and every API endpoint SHALL enforce authorization server-side. A direct API call SHALL
fail when the caller lacks the required permission, even when the corresponding UI control is
hidden.

*Source: `AUTHZ-003`, `AUTHZ-004`, `SEC-005`, `UI-002`.*

#### Scenario: Hidden control called directly

- **WHEN** a user without Edit permission calls the update endpoint directly, bypassing the UI
- **THEN** the request is rejected with an authorization error
- **AND** no partial write occurs

#### Scenario: Administrative surface is not exempt

- **WHEN** an administrative endpoint is called
- **THEN** it is evaluated by the same evaluator as any other endpoint, with no administrator
  bypass path

### Requirement: Every exposed endpoint declares a permission requirement

Every exposed endpoint SHALL carry a declarative classification as public, authenticated, or
permission-gated. An endpoint declaring nothing SHALL deny every request. The set of exposed
endpoints SHALL be enumerated automatically, and an unclassified endpoint SHALL fail the build
rather than deploy.

*Source: `AUTHZ-003`; `platform-core`'s `platform/api-ingress` "Every exposed endpoint is
classified", which names this capability as the owner of the mechanism. The automated enumeration
is the "assert a negative" gate shape `platform-core`'s D5 identifies as the only kind that caught
real defects.*

#### Scenario: Endpoint declares no permission

- **WHEN** an authenticated route declares no required permission
- **THEN** access is denied rather than allowed by default

#### Scenario: Unclassified endpoint introduced

- **WHEN** an endpoint is added without a classification
- **THEN** the enumeration check fails and the change does not deploy

#### Scenario: Pre-existing routes are classified, not exempted

- **WHEN** the endpoint enumeration runs
- **THEN** the health and build-information routes are classified as public
- **AND** each diagnostic route is classified as permission-gated on an administrative grant, in
  addition to remaining unavailable in a production environment

### Requirement: Data scope predicates carried on grants

Beyond page and action flags, each grant SHALL carry a data scope predicate. Interviewer and Hiring
Panel Member access SHALL be scoped to that user's own assignments. Write access to posting-bound
resources SHALL be scoped to postings the user is assigned to or authorized to manage, while read
scope SHALL be governed by configuration rather than hard-coded.

*Source: `C-03` as resolved — writes scoped by posting assignment (`BR-005`,
`job_postings.recruiter_ids[]`), reads governed by the matrix with a least-privilege default
(`AUTHZ-008`, `CAN-006`). `design.md` D4.*

#### Scenario: Assignment-scoped predicate is intrinsic

- **WHEN** the Interviewer role's grants are inspected
- **THEN** each carries an assignment-scoped predicate, so the role cannot resolve to all
  interviews regardless of page grants

#### Scenario: Write on an unassigned posting

- **WHEN** a user with a posting-scoped write grant attempts a write against a posting they are not
  assigned to
- **THEN** the request is denied even though the page-level action is granted

#### Scenario: Read scope is configuration, not code

- **WHEN** an administrator changes the configured read scope for a role
- **THEN** the evaluator's read verdicts change accordingly with no code deployment

### Requirement: Scope is applied as a query filter, not by discarding fetched rows

For list operations the evaluator SHALL return a scope filter that the query applies, rather than
the caller retrieving records and removing those the user may not see.

*Source: `design.md` D4, inherited from `talentsphere-wave-1-foundation`'s D2. Post-filtering leaks
counts and breaks pagination — the user learns how many records exist that they cannot see.*

#### Scenario: Result counts reflect scope

- **WHEN** a scoped user lists a resource
- **THEN** the reported total and the pagination reflect only records within scope

#### Scenario: List path without a filter

- **WHEN** a list operation is performed without applying the evaluator's scope filter
- **THEN** the test suite fails

### Requirement: Authorization decisions are never stored as resolved verdicts

A resolved authorization verdict SHALL NOT be persisted into any derived record, index, or cached
artifact. Where stored data must support an authorization decision, it SHALL carry the inputs to
that decision rather than its outcome.

*Source: `exploration-notes.md` Interaction A. `C-01`'s adoption of `VEC-003` plus `C-03`'s
configurable reads means a matrix change would invalidate any verdict baked into a vector index,
forcing a `VEC-005` re-index on every permission edit. Evaluating at query time removes that
coupling; `matching-and-ranking`'s `TS-BL-050` is the consumer.*

#### Scenario: Permission change without re-indexing

- **WHEN** an administrator changes the permission matrix
- **THEN** no derived record or index requires rebuilding for authorization to be correct

#### Scenario: Stored authorization metadata inspected

- **WHEN** authorization metadata stored alongside a derived record is inspected
- **THEN** it contains the identifiers and attributes an authorization decision is made from
- **AND** it contains no resolved allow or deny outcome

### Requirement: Service account authorization

A service account SHALL be authorized as a first-class actor with controlled service permissions,
and SHALL NOT hold a permission that constitutes a human accountability decision.

*Source: `C-02`'s AI Service Account as a first-class actor, `WF-004`, `AI-010`, `BR-007`. The
Approve prohibition is the authorization half of the advisory-only guarantee; `platform-core`'s
workflow engine and `ai-platform-governance`'s gateway carry the other halves.*

#### Scenario: Service account cannot approve

- **WHEN** the AI Service Account is evaluated for the Approve action on any page
- **THEN** access is denied
- **AND** the action cannot be granted to it through the matrix

#### Scenario: Service account acts on background work

- **WHEN** a background AI task runs under the service account
- **THEN** its verdicts are produced by the same evaluator as human requests

### Requirement: Immediate effect, with no cached verdict

A permission, override, or role change SHALL take effect for the affected user's subsequent
requests without requiring re-authentication, and SHALL NOT be served from a stale cached verdict.

*Source: `AUTHZ-007`, `ADM-007`, and `config.yaml`'s "uncached, so revocation bites on the next
request". `identity-and-access`'s D6 already pays a per-request session read to hold the same
property for sessions; caching verdicts would hand it straight back on the same request path.*

#### Scenario: Access removed mid-session

- **WHEN** an administrator revokes a permission from a user holding an active session
- **THEN** that user's next request for the affected action is denied

#### Scenario: Access granted mid-session

- **WHEN** an administrator grants a permission to a user holding an active session
- **THEN** that user's next request for the affected action succeeds without re-authentication

### Requirement: Resolved permissions are supplied through the declared resolution interface

The evaluator SHALL supply a user's resolved permissions through the permission-resolution
interface that already exists, without requiring any change to a consumer of that interface.

*Source: `identity-and-access`'s `identity/user-model` — "Permission resolution is consumed, never
computed here" — which declares the interface and ships a resolver returning no grants, with a
scenario stating that when the evaluator becomes available no consumer requires modification.
`design.md` D3.*

#### Scenario: Evaluator installed behind the existing interface

- **WHEN** the evaluator is deployed
- **THEN** resolved permissions come from it
- **AND** no consumer of the resolution interface is modified

#### Scenario: Session content unchanged in shape

- **WHEN** a session's resolved permissions are inspected before and after the evaluator is
  installed
- **THEN** the shape of the response is unchanged and only the resolved values differ

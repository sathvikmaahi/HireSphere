## Purpose

The product-wide catalog of pages and actions that the permission matrix is a matrix *of*, and the
seeded matrix values for the nine roles — recorded as an auditable product decision rather than a
silent script effect, because those values determine whether a headline feature works on day one.
Owned by `TS-BL-022`.

## ADDED Requirements

### Requirement: The page and action catalog covers the whole product

The system SHALL seed a permission catalog entry for every screen in the product's screen
inventory, across every applicable action flag, including screens whose routes do not yet exist.
Where a single screen's capabilities are required by two roles whose seeded postures differ, that
screen SHALL seed as more than one catalog entry, so that each posture is expressible on a key of
its own.

*Source: `reference/spec.md` §14.2's screen inventory and §9.3's action flags; `design.md` D5,
inherited from `talentsphere-wave-1-foundation`'s D3. A complete catalog makes the matrix
reviewable as one artifact and lets an administrator pre-configure access ahead of a feature
landing; seeding per phase would make the matrix a moving target and force the seeded-values
decision to recur four more times.*

*The splitting rule was added after `insight-and-reporting`'s `design.md` D4 found that §14.2's
single `Reports` screen spans both operational dashboards and AI audit. A page key is the unit both
the matrix and `user_permission_overrides` act on, so one key cannot hold two postures and no
override can rescue it — a per-user deny lands on the whole key.*

#### Scenario: Catalog enumerated

- **WHEN** the permission catalog is enumerated
- **THEN** every screen in the product's screen inventory is present with its applicable actions

#### Scenario: A screen spanning two postures

- **WHEN** the catalog entries for the reporting surfaces are enumerated
- **THEN** operational reporting and governance reporting are present as separate entries
- **AND** a grant on either confers no access to the other

#### Scenario: Unrouted page grants nothing

- **WHEN** a catalog entry exists for a page whose route has not been built
- **THEN** it is denied to every role until explicitly granted, and grants nothing to anyone

#### Scenario: Catalog entry added later

- **WHEN** a new page or action is introduced
- **THEN** it enters the catalog denied to every role

### Requirement: Roles are seeded with grants, not created here

The seed SHALL attach permission assignments to the nine existing role records and SHALL NOT
create, rename, or alter those records.

*Source: `identity-and-access`'s `design.md` D4 and its `identity/user-model` capability, which own
`roles`, `user_roles` and `users` and seed the nine roles as identity. The split is at the join
table: this capability owns `role_permissions` and what a role may do; the role's own existence and
name are not its to change (`design.md` D2, D3).*

#### Scenario: Role identity untouched

- **WHEN** the seed runs
- **THEN** no role record is created, renamed, or removed
- **AND** permission assignments referencing the nine roles are created

#### Scenario: Seeded roles are protected from deletion

- **WHEN** a user attempts to delete a seeded system role
- **THEN** the request is rejected
- **AND** the role's permission assignments remain editable

### Requirement: Seeded matrix values are a recorded product decision

The seeded matrix values SHALL be recorded as an auditable configuration state with a stated
rationale, and SHALL NOT be applied as a silent script effect.

*Source: `exploration-notes.md` Interaction B — "the seeded matrix is a decision, not a default":
with `AUTHZ-008` applied literally to reads, a Recruiter cannot see candidates outside their
assigned postings and cross-pool resurfacing silently returns nothing. `design.md` D5 records the
posture and flags it for owner sign-off.*

#### Scenario: Seeded posture is auditable

- **WHEN** the seed applies matrix values
- **THEN** the resulting configuration state is recorded in the audit trail as a configuration
  decision, identifying what was set

#### Scenario: Changing a seeded value is configuration

- **WHEN** an administrator changes a seeded matrix value after deployment
- **THEN** it changes as ordinary configuration with its own audit record, requiring no deployment

### Requirement: Seeded read scope is open within role for the recruiting roles

The seed SHALL grant Recruiter, Practice Manager, and Recruitment Manager read access across the
candidate and posting pools, as a deliberate widening of the least-privilege default.

*Source: `C-03` as resolved — `CAN-006` defers read scope to the matrix, and `AUTHZ-008` makes the
default restrictive; Interaction B shows that shipping the default literally breaks resurfacing
quietly. `design.md` D5: a closed default fails invisibly (an empty queue looks like "no matches"),
an open one fails visibly (someone sees a candidate they did not expect to, and says so).*

#### Scenario: Cross-pool resurfacing functions on first deployment

- **WHEN** a freshly seeded Recruiter views candidates outside their assigned postings
- **THEN** those candidates are readable, so resurfacing has a pool to draw from

#### Scenario: Reads open, writes still scoped

- **WHEN** a freshly seeded Recruiter attempts a write against a posting they are not assigned to
- **THEN** the write is denied despite the open read scope

### Requirement: Seeded least-privilege postures for the remaining roles

The seed SHALL grant Interviewer and Hiring Panel Member assignment-scoped access with no Export
and no Approve; the Auditor read-only access to audit and AI run records with no Create, Edit,
Delete, Approve, or Assign anywhere; Application and System Administrators configuration access
with no View on candidate personal data; and a service account only the actions its background work
requires. The Auditor SHALL additionally be granted View on the governance-reporting entry, and
SHALL be seeded with no assignment at all on the operational-reporting entry.

*Source: `AUTHZ-008`; `C-02`'s role definitions; `D16`'s configuration-only Administrator;
`design.md` D5.*

*The Auditor's operational-reporting cell is left unset rather than seeded as an explicit denial,
deliberately. `AUTHZ-005`'s deny-wins precedence crosses roles, so an explicit denial on the Auditor
role would also strip operational reporting from a user who holds Practice Manager alongside it —
silently, and indistinguishably from an ordinary denial. Deny-by-default already produces the
required outcome for an Auditor-only user. The property that matters is the resulting posture, not
the cell state (`design.md` D5).*

#### Scenario: Freshly seeded interviewer

- **WHEN** a user is granted only the Interviewer role with no additional configuration
- **THEN** the user receives no administrative, export, or approval capability
- **AND** their access is scoped to their own assignments

#### Scenario: Freshly seeded administrator

- **WHEN** an Application Administrator's seeded grants are inspected
- **THEN** they include no View on candidate personal data

#### Scenario: Freshly seeded auditor

- **WHEN** the Auditor's seeded grants are inspected
- **THEN** they include read access to audit and AI run records and no Create, Edit, Delete,
  Approve, or Assign on any resource

#### Scenario: Auditor reaches governance reporting only

- **WHEN** a user holding only the Auditor role requests each reporting surface
- **THEN** governance reporting is readable
- **AND** operational reporting is denied

#### Scenario: Auditor who also holds a recruiting role

- **WHEN** a user holds both the Auditor role and the Practice Manager role
- **THEN** operational reporting remains readable through the Practice Manager grant
- **AND** the Auditor role contributes no denial that removes it

#### Scenario: Approve is unassignable to a service account

- **WHEN** an administrator attempts to grant Approve to the AI Service Account through the matrix
- **THEN** the assignment is rejected

### Requirement: Seeding is idempotent and produces a usable environment

The seed SHALL run against an empty database to produce the catalog, the seeded matrix values, and
at least one usable administrative user, and SHALL run again against an already-seeded database
without duplicating anything.

*Source: `ENG-010` seed scripts; `talentsphere-wave-1-foundation`'s Migration Plan, which requires
idempotent seeds run as a deployment step so a fresh environment reaches a usable state.*

#### Scenario: Fresh environment

- **WHEN** the seed runs against an empty database
- **THEN** the page catalog, the seeded matrix values, and at least one usable administrative user
  exist

#### Scenario: Re-running the seed

- **WHEN** the seed runs against an already-seeded database
- **THEN** it completes without duplicating catalog entries, permission assignments, or users

#### Scenario: Re-running does not revert administrator changes

- **WHEN** the seed runs after an administrator has changed a seeded matrix value
- **THEN** the administrator's value is preserved rather than reset to the seeded one

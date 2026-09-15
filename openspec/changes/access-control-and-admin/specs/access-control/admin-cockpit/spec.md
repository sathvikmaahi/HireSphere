## Purpose

The administrative surface for local access control: administering users and roles, editing the
page and action permission matrix, and reading the explanation for any verdict — all inside the
authenticated application shell, and visible to nobody who is not an administrator. Owned by
`TS-BL-023`.

## ADDED Requirements

### Requirement: Admin Cockpit restricted to administrators

Only Application Administrators and System Administrators SHALL access Admin Cockpit pages and
APIs. Non-administrators SHALL NOT view Admin Cockpit routes or endpoints.

*Source: `ADM-008`. Inherited from `talentsphere-wave-1-foundation`'s
`access-control/admin-cockpit`; path preserved per `design.md` D2. Enforced by the evaluator, not
by a separate administrative check — `design.md` D1's no-second-path rule applies to this surface
like any other.*

#### Scenario: Non-admin navigates to an admin route

- **WHEN** a Recruiter requests an Admin Cockpit page or its underlying endpoint
- **THEN** access is denied and no configuration data is returned

#### Scenario: Admin navigation visibility

- **WHEN** a non-administrator loads the application shell
- **THEN** no Admin Cockpit navigation entry is present

#### Scenario: Hidden entry is not the control

- **WHEN** a non-administrator requests an Admin Cockpit endpoint directly, with no navigation
  entry having been rendered
- **THEN** the server refuses the request

### Requirement: The Cockpit supplies navigation and affordance decisions; the shell evaluates nothing

The Cockpit SHALL supply the application shell with an already-filtered navigation list and with
explicit affirmative decisions for permission-governed affordances. It SHALL NOT rely on the shell
to evaluate permissions.

*Source: `design-system`'s `design-system/app-shell` — the sidebar "renders the navigation items it
is given and SHALL NOT evaluate permissions itself", and a permission-governed affordance renders
only on an affirmative decision, nothing otherwise. `UI-002` keeps the server authoritative.
`design.md` D12.*

#### Scenario: Navigation reflects verdicts

- **WHEN** the Cockpit supplies the shell's navigation list
- **THEN** it excludes pages the signed-in user cannot view, having asked the evaluator

#### Scenario: Affordance with no decision

- **WHEN** a Cockpit screen renders a permission-governed control without supplying a decision
- **THEN** the control is not rendered

### Requirement: User administration

Admin Cockpit SHALL allow creation, activation, deactivation, and update of local users once a
Hubble identity is known, and SHALL allow role assignment. Every such change SHALL be audited.

*Source: `ADM-001`, `ADM-002`. The user record and its activation states are
`identity-and-access`'s `identity/user-activation` and `identity/user-model`; this capability is
the administrative surface that drives them.*

#### Scenario: Activating a pending user

- **WHEN** an administrator activates a pending user and assigns one or more roles
- **THEN** the user's status becomes active, the roles are assigned, and both changes are audited

#### Scenario: Filtering the user list

- **WHEN** an administrator filters users by status or role
- **THEN** only matching users are listed

#### Scenario: Deactivation from the Cockpit

- **WHEN** an administrator deactivates a user, supplying a reason
- **THEN** the deactivation is applied and audited, and the user's sessions are revoked

### Requirement: Role administration

Admin Cockpit SHALL allow role creation, update, deactivation, and permission assignment, and SHALL
reject a duplicate role name with a field-level validation error.

*Source: `ADM-002`. The nine seeded roles remain protected from deletion while their permission
assignments stay editable (`access-control/permission-seed`).*

#### Scenario: Creating a role

- **WHEN** an administrator creates a role with a unique name
- **THEN** the role is created with no permissions granted

#### Scenario: Duplicate role name

- **WHEN** an administrator creates a role whose name already exists
- **THEN** the request is rejected with a field-level validation error

### Requirement: Permission matrix surface

The matrix view SHALL display users or roles as rows and modules, pages, or actions as columns,
with cells settable to grant, deny, or unset, and SHALL support filtering by user, role, module,
practice, and active status.

*Source: `ADM-003`, `ADM-004`, `ADM-005`. The dense-data-table pattern this screen is built on is
`design-system`'s `design-system/data-table`, consumed here rather than reinvented.*

#### Scenario: Switching row subject

- **WHEN** an administrator switches the matrix from roles to users
- **THEN** rows become users and each cell reflects that user's effective assignment

#### Scenario: Bulk update

- **WHEN** an administrator submits multiple cell changes as one update
- **THEN** all changes are applied atomically and each produces its own audit record

#### Scenario: Partial failure in a bulk update

- **WHEN** one cell change in a submitted batch is invalid
- **THEN** no change in the batch is applied

#### Scenario: Denial is visually distinct

- **WHEN** a cell holds an explicit denial that overrides a role grant
- **THEN** the matrix presents that cell as a denial rather than as merely ungranted

### Requirement: Explanation panel

Admin Cockpit SHALL present the permission explanation for any selected subject, page, and action,
showing role grants, direct grants, and explicit denials.

*Source: `ADM-006`, `AUTHZ-006`. The explanation itself is
`access-control/permission-explanation`; this requirement places it in the administrator's
workflow, where an unexpected verdict is actually encountered.*

#### Scenario: Inspecting an unexpected denial

- **WHEN** an administrator selects a user, page, and action where the user expected access
- **THEN** the panel names the denial or absent grant responsible for the verdict

#### Scenario: Explanation reachable from the matrix cell

- **WHEN** an administrator selects a matrix cell
- **THEN** the explanation for that subject, page, and action is available without leaving the
  matrix

### Requirement: Administrative table behavior

Administrative listings SHALL support search, filtering, sorting, and pagination, and SHALL support
export only where the requesting user holds Export permission.

*Source: `UI-008`, `SEC-012`, `PRV-007`.*

#### Scenario: Export without permission

- **WHEN** a user without Export permission requests an export of an administrative listing
- **THEN** the request is denied and no export control was rendered

#### Scenario: Export with permission

- **WHEN** a user with Export permission exports a listing
- **THEN** the export is produced, carries its data classification label, and the export event is
  audited

### Requirement: The Cockpit renders inside the authenticated shell

Admin Cockpit screens SHALL render inside the authenticated application shell, using its page
header, page template, and navigation contracts rather than defining a parallel layout.

*Source: `design-system`'s `design-system/app-shell`; `AGENTS.md`'s standing quality bar — reuse
the shared shell rather than inventing a new one. D.9 records that this item's dependency is the
Authenticated Shell specifically, a correction made when the old decomposition's dependency did not
survive translation to the finer grain (`design.md` D12).*

#### Scenario: Cockpit page structure

- **WHEN** an Admin Cockpit page renders
- **THEN** it uses the shell's page header pattern and one of the shell's page templates

#### Scenario: No parallel layout

- **WHEN** Admin Cockpit screens are inspected
- **THEN** none defines its own shell, sidebar, or raw color or spacing value

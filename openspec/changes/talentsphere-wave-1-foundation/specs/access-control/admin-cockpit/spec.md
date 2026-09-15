## Purpose

The administrative surface for local access control: activating users, administering roles, editing the page and action permission matrix, explaining verdicts, and seeding the system so it is testable from first deployment.

## ADDED Requirements

### Requirement: Admin Cockpit restricted to administrators

Only Application Administrators and System Administrators SHALL access Admin Cockpit pages and APIs. Non-admin users SHALL NOT view Admin Cockpit routes or endpoints.

#### Scenario: Non-admin navigates to an admin route

- **WHEN** a Recruiter requests an Admin Cockpit page or its underlying endpoint
- **THEN** access is denied and no configuration data is returned

#### Scenario: Admin navigation visibility

- **WHEN** a non-admin user loads the application shell
- **THEN** Admin Cockpit navigation entries are not shown

### Requirement: User administration

Admin Cockpit SHALL allow creation, activation, deactivation, and update of local users once a Hubble identity is known.

#### Scenario: Activating a pending user

- **WHEN** an administrator activates a pending user and assigns one or more roles
- **THEN** the user's status becomes active, the roles are assigned, and both changes are audited

#### Scenario: Filtering the user list

- **WHEN** an administrator filters users by status or role
- **THEN** only matching users are listed

### Requirement: Role administration

Admin Cockpit SHALL allow role creation, update, deactivation, and permission assignment.

#### Scenario: Creating a role

- **WHEN** an administrator creates a role with a unique name
- **THEN** the role is created with no permissions granted

#### Scenario: Duplicate role name

- **WHEN** an administrator creates a role whose name already exists
- **THEN** the request is rejected with a field-level validation error

### Requirement: Permission matrix surface

The matrix view SHALL display users or roles as rows and modules, pages, or actions as columns, with cells settable to grant, deny, or unset. The matrix SHALL support filtering by user, role, module, practice, and active status.

#### Scenario: Switching row subject

- **WHEN** an administrator switches the matrix from roles to users
- **THEN** rows become users and each cell reflects that user's effective assignment

#### Scenario: Bulk update

- **WHEN** an administrator submits multiple cell changes as one update
- **THEN** all changes are applied atomically and each produces its own audit record

#### Scenario: Denial is visually distinct

- **WHEN** a cell holds an explicit denial that overrides a role grant
- **THEN** the matrix presents that cell as a denial rather than as merely ungranted

### Requirement: Explanation panel

Admin Cockpit SHALL present the permission explanation for any selected subject, page, and action, showing role grants, direct grants, and explicit denials.

#### Scenario: Inspecting an unexpected denial

- **WHEN** an administrator selects a user, page, and action where the user expected access
- **THEN** the panel names the denial or absent grant responsible for the verdict

### Requirement: Administrators are configuration-only on candidate data

Administrator roles SHALL NOT hold View permission on candidate personal data by default. Access to candidate personal data by an administrator SHALL require an explicit, time-boxed, audited elevation.

#### Scenario: Administrator opens a candidate record

- **WHEN** an Application Administrator requests candidate personal data without an active elevation
- **THEN** access is denied

#### Scenario: Break-glass elevation

- **WHEN** an administrator requests elevation, supplying a typed reason
- **THEN** the elevation is granted for a bounded duration, recorded in the audit trail, and notified to the configured recipients
- **AND** the elevation expires automatically without administrator action

#### Scenario: Elevation without a reason

- **WHEN** an administrator requests elevation with no reason supplied
- **THEN** the request is rejected

### Requirement: Seed data

The system SHALL provide seed scripts creating the nine roles, the complete page and action permission catalog, initial administrative users, and the seeded matrix values.

#### Scenario: Fresh environment

- **WHEN** the seed scripts run against an empty database
- **THEN** the nine roles, the page catalog, and at least one usable administrative user exist
- **AND** the seeded matrix values are recorded as an auditable configuration state rather than applied silently

#### Scenario: Re-running seeds

- **WHEN** the seed scripts run against an already-seeded database
- **THEN** they complete without duplicating roles, permissions, or users

### Requirement: Administrative table behavior

Administrative listings SHALL support search, filtering, sorting, and pagination, and SHALL support export only where the requesting user holds Export permission.

#### Scenario: Export without permission

- **WHEN** a user without Export permission requests an export of an administrative listing
- **THEN** the request is denied

#### Scenario: Export with permission

- **WHEN** a user with Export permission exports a listing
- **THEN** the export is produced and the export event is audited

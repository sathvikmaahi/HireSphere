# Admin RBAC Specification

## Purpose

Provide an Admin Cockpit for managing users, roles, permissions, and a page/action authorization matrix. All authorization MUST be enforced server-side; UI hiding is defense-in-depth only.

## Requirements

### Requirement: Role-based access control

The system SHALL implement RBAC where users are assigned roles, roles are granted permissions (`resource:action`), and API handlers enforce permissions.

#### Scenario: Permission denied

- **GIVEN** a user lacks `job_postings:close` permission
- **WHEN** they call `POST /api/v1/postings/:id/close`
- **THEN** the system MUST return HTTP 403
- **AND** MUST NOT modify the posting

### Requirement: Default system roles

The system MUST ship seeded roles: `admin`, `recruiter`, `hiring_manager`, `interviewer`, and `viewer`.

#### Scenario: Admin full access

- **GIVEN** a user has the `admin` role
- **WHEN** they access admin routes
- **THEN** they MUST have `admin:access` and implicit permission to all resources unless explicitly restricted by policy

### Requirement: Page action matrix

The system SHALL maintain a page/action matrix mapping UI pages and actions to required permissions.

#### Scenario: Matrix update

- **GIVEN** an admin edits the matrix via `PUT /api/v1/permissions/matrix`
- **WHEN** the save succeeds
- **THEN** subsequent authorization checks MUST use the updated mapping
- **AND** an audit event MUST record before/after state

### Requirement: Admin users management UI

Admins MUST manage users from `/admin/users` using the browsable grid + detail drawer pattern.

#### Scenario: Create user

- **GIVEN** an admin on the users page
- **WHEN** they create a user with username, email, and initial password
- **THEN** the user MUST be created with status `pending`
- **AND** MUST appear in the user list

### Requirement: Admin roles and permissions UI

Admins MUST manage roles and permission assignments from `/admin/roles` and `/admin/permissions`.

#### Scenario: Assign role to user

- **GIVEN** an admin edits a user
- **WHEN** they assign the `recruiter` role
- **THEN** the user MUST gain all permissions linked to that role on next token refresh or login

### Requirement: Admin nav gating

The sidebar Admin nav item MUST only appear for users with `admin:access` permission.

#### Scenario: Non-admin sidebar

- **GIVEN** a user without `admin:access`
- **WHEN** they view the sidebar
- **THEN** the Admin nav item MUST NOT be visible
- **AND** direct navigation to `/admin/*` MUST redirect or show forbidden state

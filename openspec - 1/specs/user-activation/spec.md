# User Activation Specification

## Purpose

Ensure only activated users can access HireSphere. User lifecycle states (pending, active, suspended, deactivated) MUST be enforced at the API layer and reflected in the admin UI.

## Requirements

### Requirement: User status enum

Every user MUST have a status of `pending`, `active`, `suspended`, or `deactivated`.

#### Scenario: New user default

- **GIVEN** an admin creates a user account
- **WHEN** the account is saved
- **THEN** status MUST default to `pending`
- **AND** the user MUST NOT be able to access protected routes until activated

### Requirement: API access enforcement

All protected API endpoints MUST reject requests from users whose status is not `active`.

#### Scenario: Suspended user API call

- **GIVEN** a user with status `suspended` holds a valid JWT
- **WHEN** they call any protected endpoint
- **THEN** the system MUST return HTTP 403 with reason `USER_NOT_ACTIVE`
- **AND** MUST write an audit event

### Requirement: Activation by admin

Admins MUST be able to activate users via `POST /api/v1/users/:id/activate`.

#### Scenario: Admin activates pending user

- **GIVEN** a user with status `pending`
- **WHEN** an admin with `admin:access` calls the activate endpoint
- **THEN** status MUST change to `active`
- **AND** `activated_at` and `activated_by` MUST be recorded
- **AND** an audit event MUST be written

### Requirement: Suspend and deactivate

Admins MUST be able to suspend or deactivate users, immediately revoking effective access.

#### Scenario: Admin suspends active user

- **GIVEN** a user with status `active`
- **WHEN** an admin calls suspend
- **THEN** status MUST change to `suspended`
- **AND** subsequent API calls MUST return 403 even if JWT has not expired

### Requirement: Optional invite activation

The system MAY support activation via time-limited activation tokens for invite-link onboarding.

#### Scenario: Valid activation token

- **GIVEN** a pending user receives an activation link with a valid unexpired token
- **WHEN** they complete activation
- **THEN** status MUST change to `active`
- **AND** the token MUST be marked used and MUST NOT be reusable

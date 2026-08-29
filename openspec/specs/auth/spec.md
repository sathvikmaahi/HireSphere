# Authentication Specification

## Purpose

Provide REST API authentication using username and password. Hubble SSO and external identity providers are out of scope. All API access except login and health checks MUST require a valid access token.

## Requirements

### Requirement: REST login

The system SHALL authenticate users via `POST /api/v1/auth/login` with `{ username, password }` and return access and refresh tokens on success.

#### Scenario: Successful login

- **GIVEN** a user with status `active` and valid credentials
- **WHEN** they submit username and password to the login endpoint
- **THEN** the system MUST return HTTP 200 with `accessToken`, `refreshToken`, and a user object
- **AND** MUST update `last_login_at` in the database
- **AND** MUST write an audit event

#### Scenario: Invalid credentials

- **GIVEN** incorrect username or password
- **WHEN** login is attempted
- **THEN** the system MUST return HTTP 401
- **AND** MUST NOT reveal whether the username exists

#### Scenario: Non-active user blocked

- **GIVEN** a user with status `pending`, `suspended`, or `deactivated`
- **WHEN** they submit valid credentials
- **THEN** the system MUST return HTTP 403 with a machine-readable reason code
- **AND** MUST NOT issue tokens

### Requirement: Password storage

Passwords MUST be hashed with bcrypt (cost factor 12). Plain-text passwords MUST NOT be stored or logged.

#### Scenario: User creation

- **GIVEN** an admin creates a user
- **WHEN** a password is set
- **THEN** only the bcrypt hash MUST be persisted

### Requirement: JWT access and refresh tokens

Access tokens MUST be short-lived (15 minutes). Refresh tokens MUST be long-lived (7 days) and rotated on use.

#### Scenario: Token refresh

- **GIVEN** a valid refresh token
- **WHEN** `POST /api/v1/auth/refresh` is called
- **THEN** the system MUST return a new access token and a new refresh token
- **AND** MUST invalidate the previous refresh token

#### Scenario: Expired access token

- **GIVEN** an expired access token
- **WHEN** a protected endpoint is called without refresh
- **THEN** the system MUST return HTTP 401

### Requirement: Rate limiting on login

The login endpoint MUST be rate-limited to mitigate brute-force attacks.

#### Scenario: Excessive login attempts

- **GIVEN** more than 5 failed login attempts from the same IP within 15 minutes
- **WHEN** another attempt is made
- **THEN** the system MUST return HTTP 429
- **AND** MUST log the event

### Requirement: Login UI on bare shell

The web application SHALL provide `/login` on the bare/public shell with username and password fields.

#### Scenario: Login form

- **GIVEN** an unauthenticated user visits `/login`
- **WHEN** the page renders
- **THEN** it MUST show username and password inputs and a submit button
- **AND** MUST display validation and error states without exposing internal details
- **AND** MUST NOT offer Hubble or OAuth login options

### Requirement: Logout

The system SHALL support `POST /api/v1/auth/logout` to invalidate the current refresh token.

#### Scenario: User signs out

- **GIVEN** an authenticated user
- **WHEN** they sign out
- **THEN** refresh token MUST be invalidated server-side
- **AND** client MUST clear stored tokens
- **AND** user MUST be redirected to `/login`

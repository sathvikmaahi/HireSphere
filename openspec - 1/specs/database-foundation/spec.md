# Database Foundation Specification

## Purpose

Provide the Drizzle ORM setup, identity and audit schema, migration pipeline, and seed data so auth and RBAC feature branches can build on a consistent PostgreSQL foundation in local, staging, and production environments.

Branch: `infra/database-foundation`. MUST merge after `infra/monorepo-scaffold`. Column-level detail lives in `docs/database-schema.md`.

## Requirements

### Requirement: Drizzle configuration

Database schema and migrations MUST live in `packages/db/` with Drizzle Kit configured for PostgreSQL 15.

#### Scenario: Generate migration

- **GIVEN** schema changes in `packages/db/schema/`
- **WHEN** `pnpm db:generate` is run
- **THEN** a new SQL migration MUST be written to `packages/db/migrations/`
- **AND** MUST be idempotent when applied via `pnpm db:migrate`

### Requirement: Phase 0 identity and audit tables

The initial migration (`0001_identity_audit.sql`) MUST create: `users`, `roles`, `permissions`, `role_permissions`, `user_roles`, `page_action_matrix`, `activation_tokens`, and `audit_events` with enums and constraints per `docs/database-schema.md`.

#### Scenario: Fresh migrate

- **GIVEN** an empty PostgreSQL database
- **WHEN** `pnpm db:migrate` runs
- **THEN** all Phase 0 tables MUST exist
- **AND** `audit_events` MUST be append-only (no UPDATE/DELETE triggers or application enforcement documented in migration comments)

### Requirement: Seed data

A seed script MUST create default roles and a bootstrap admin user for local and staging environments.

#### Scenario: Seed after migrate

- **GIVEN** migrations have been applied
- **WHEN** `pnpm db:seed` runs
- **THEN** roles MUST include at minimum: `admin`, `recruiter`, `hiring_manager`, `interviewer`, `viewer`
- **AND** a seed admin user MUST exist with documented default credentials for local dev only

### Requirement: Connection helper

`packages/db` MUST expose a connection helper that supports direct TCP (local Docker Postgres) and Cloud SQL connector (Cloud Run / migrate job).

#### Scenario: Local connection

- **GIVEN** `DATABASE_URL` points to `localhost:5432`
- **WHEN** the API starts
- **THEN** it MUST connect without the Cloud SQL connector

#### Scenario: Cloud Run connection

- **GIVEN** the API runs on Cloud Run with VPC connector configured
- **WHEN** it connects to Cloud SQL private IP
- **THEN** SSL MUST be required (`sslmode=verify-full` or equivalent)

### Requirement: Health check database probe

After this branch merges, `GET /api/v1/health` MUST return HTTP 200 only when the database connection succeeds.

#### Scenario: Database unreachable

- **GIVEN** the API is running but Postgres is down
- **WHEN** `/api/v1/health` is called
- **THEN** response status MUST NOT be 200
- **AND** response body MUST indicate database unreachable

### Requirement: Cloud Run migrate job

Migrations MUST run via the `hiresphere-migrate` Cloud Run Job using the same migration artifacts as local `pnpm db:migrate`.

#### Scenario: Staging deploy

- **GIVEN** a new migration is merged to `main`
- **WHEN** Cloud Build runs the migrate job before API deploy
- **THEN** staging database schema MUST match `packages/db/migrations/`
- **AND** API deploy MUST NOT proceed if migrate fails

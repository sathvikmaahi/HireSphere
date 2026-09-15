# Monorepo Scaffold Specification

## Purpose

Establish the HireSphere monorepo layout, local development environment, container build targets, and platform adapters so application feature branches can ship against a runnable empty app locally and a deployable skeleton in GCP staging.

Branch: `infra/monorepo-scaffold`. MUST merge after `infra/gcp-terraform`.

## Requirements

### Requirement: pnpm workspace layout

The repository SHALL use pnpm workspaces with at minimum: `apps/web`, `apps/api`, `services/worker`, `packages/db`, `packages/shared`, `packages/ai-prompts`.

#### Scenario: Install from root

- **GIVEN** a clean clone on branch `infra/monorepo-scaffold`
- **WHEN** an engineer runs `pnpm install` at the repository root
- **THEN** all workspace packages MUST resolve dependencies
- **AND** each package MUST expose `dev`, `build`, and `lint` scripts

### Requirement: Local development stack

Local development MUST be supported via `docker-compose.yml` with PostgreSQL 15 and a Pub/Sub emulator.

#### Scenario: Start local dependencies

- **GIVEN** Docker is running
- **WHEN** `docker compose up postgres` is executed
- **THEN** Postgres MUST accept connections on port 5432
- **AND** the API MUST connect using `DATABASE_URL` from `.env.local`

#### Scenario: API dev server

- **GIVEN** local Postgres is running
- **WHEN** `pnpm dev` is run in `apps/api`
- **THEN** the API MUST listen on a documented port (default 3000)
- **AND** `GET /api/v1/health` MUST return HTTP 200

### Requirement: Container images

Dockerfiles MUST exist for API, worker, and migrate services under `docker/`.

#### Scenario: Cloud Build image build

- **GIVEN** the scaffold branch is merged
- **WHEN** Cloud Build runs `docker build` for each Dockerfile
- **THEN** images MUST build without application feature code beyond health checks
- **AND** MUST be pushable to Artifact Registry

### Requirement: Environment configuration

A root `.env.example` MUST document all required environment variables. Secrets MUST NOT be committed.

#### Scenario: New developer setup

- **GIVEN** `.env.example` exists
- **WHEN** a developer copies it to `.env.local` and fills values
- **THEN** API and worker MUST start without undocumented env vars
- **AND** variables MUST include at minimum: `DATABASE_URL`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `GCS_BUCKET_RESUMES`, `GCP_PROJECT_ID`

### Requirement: Platform adapters

`packages/shared` MUST define interfaces and GCP implementations for `StorageProvider`, `JobQueue`, and `AIProvider`.

#### Scenario: Adapter swap for tests

- **GIVEN** feature code imports adapters from `packages/shared`
- **WHEN** a unit test provides in-memory implementations
- **THEN** API handlers MUST NOT import GCS, Pub/Sub, or Vertex SDKs directly

### Requirement: Health endpoint

The API MUST expose `GET /api/v1/health` returning HTTP 200 when the process is running. Database connectivity checks MAY be added in `infra/database-foundation`.

#### Scenario: Load balancer probe

- **GIVEN** the API is deployed to Cloud Run
- **WHEN** the load balancer hits `/api/v1/health`
- **THEN** response status MUST be 200

# Infrastructure Specification

## Purpose

Define how HireSphere is provisioned and deployed on Google Cloud Platform using a single Terraform branch (`infra/gcp-terraform`). All GCP resources — networking, database, compute, storage, messaging, load balancing, secrets, CI/CD, and observability — MUST be codified in `infra/terraform/` and applied per environment (dev, staging, prod).

## Requirements

### Requirement: Single infrastructure branch

All GCP infrastructure SHALL be implemented on branch `infra/gcp-terraform`. The branch MUST NOT be split across multiple infra branches.

#### Scenario: Infrastructure PR scope

- **GIVEN** a pull request targeting `main`
- **WHEN** the PR modifies Terraform, Dockerfiles, or `cloudbuild.yaml`
- **THEN** the branch name MUST be `infra/gcp-terraform` (or a short-lived fork rebased from it)
- **AND** the PR MUST NOT mix application feature code

### Requirement: Terraform module layout

The repository SHALL organize Terraform as reusable modules under `infra/terraform/modules/` and environment roots under `infra/terraform/environments/{dev,staging,prod}/`.

#### Scenario: Environment apply

- **GIVEN** an engineer runs `terraform apply` from `infra/terraform/environments/staging`
- **WHEN** variables are set in `terraform.tfvars`
- **THEN** all modules (networking, iam, secrets, cloud-sql, storage, pubsub, cloud-run, load-balancer, observability, cicd) MUST be composed in that environment's `main.tf`
- **AND** remote state MUST be stored in a GCS backend bucket unique to that environment

### Requirement: Private database networking

Cloud SQL MUST use private IP only. The instance MUST NOT expose a public IPv4 endpoint.

#### Scenario: Database connectivity from API

- **GIVEN** Cloud Run API is deployed
- **WHEN** the API connects to PostgreSQL
- **THEN** connection MUST traverse Serverless VPC Access connector to Cloud SQL private IP
- **AND** SSL MUST be required

### Requirement: Static frontend and API routing

The HTTPS load balancer SHALL serve the React SPA from GCS + Cloud CDN and route `/api/v1/*` to the Cloud Run API service.

#### Scenario: Browser requests

- **GIVEN** DNS points to the load balancer
- **WHEN** a user requests `https://{domain}/candidates`
- **THEN** the SPA MUST be served from the web bucket (with `index.html` fallback for client routing)
- **WHEN** a user requests `https://{domain}/api/v1/health`
- **THEN** the request MUST be forwarded to Cloud Run API

### Requirement: Async job processing

Background work (resume scan/parse, AI batch ranking, report exports) SHALL be published to Pub/Sub topics and consumed by the Cloud Run worker service.

#### Scenario: Resume upload triggers worker

- **GIVEN** a resume PDF is accepted by the API
- **WHEN** the file is stored temporarily
- **THEN** the API MUST publish a message to the `resume-processing` topic
- **AND** the worker MUST process the message without a publicly invokable worker URL

### Requirement: Secrets management

Application secrets (database password, JWT keys, third-party API keys) MUST be stored in Secret Manager and injected into Cloud Run at deploy time. Secrets MUST NOT be committed to git.

#### Scenario: Secret access

- **GIVEN** Cloud Run API starts
- **WHEN** it needs `JWT_SECRET`
- **THEN** the value MUST be read from Secret Manager via the API service account
- **AND** MUST NOT appear in Terraform state as plain text except where Terraform generates the initial DB password

### Requirement: CI/CD pipeline

Cloud Build SHALL build container images, run database migrations via Cloud Run Job, deploy API and worker services, and sync the SPA build to GCS on merge to `main` (staging) and on version tags (prod with manual approval).

#### Scenario: Staging deploy on merge

- **GIVEN** a commit is merged to `main`
- **WHEN** Cloud Build trigger fires
- **THEN** images MUST be pushed to Artifact Registry
- **AND** migrate job MUST run before API deploy
- **AND** staging environment MUST be updated

### Requirement: Health check endpoint

The API Cloud Run service SHALL expose `GET /api/v1/health` returning HTTP 200 when the service and database connection are healthy.

#### Scenario: Load balancer health

- **GIVEN** infrastructure is applied
- **WHEN** uptime check hits `/api/v1/health`
- **THEN** response status MUST be 200
- **AND** response body MUST indicate database reachability

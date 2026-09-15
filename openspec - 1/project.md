# HireSphere — Project Context

## Overview

HireSphere is an end-to-end recruitment platform: job descriptions, postings, candidate intake, AI-assisted ranking, interview workflows, scorecards, offers, and onboarding. It is deployed on **Google Cloud Platform** and follows the visual and layout specifications in `design-spec.md` and `layout-spec.md`.

## Goals

- Give recruiters and hiring managers a single workspace from JD drafting through hire.
- Enforce role-based access, auditability, and responsible AI governance.
- Support finite and unlimited vacancy postings with distinct closure rules.
- Capture structured interview data and AI-readable notes for downstream scorecards.

## Non-Goals (Explicit Exclusions)

- **Hubble SSO / Hubble login** — authentication is username + password via REST API only.
- **Miracle branding** — use HireSphere branding and design tokens only.
- Hubble appears only as an optional **external employee ID** field on offers/onboarding (`hubble_id`), not as an identity provider.

## Repository Layout

```
HireSphere/
├── apps/web/              # React SPA → GCS + Cloud CDN
├── apps/api/              # REST API (Fastify) → Cloud Run
├── services/worker/       # Pub/Sub consumer → Cloud Run
├── packages/
│   ├── db/                # Drizzle schema, migrations, seeds
│   ├── shared/            # Types, Zod schemas, platform adapters
│   └── ai-prompts/        # Versioned prompt templates
├── infra/terraform/       # GCP IaC (modules + environments)
├── docker/                # api, worker, migrate Dockerfiles
└── docs/                  # Planning docs (architecture, schema, CI/CD)
```

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS v4 → GCS + Cloud CDN |
| Edge | HTTPS Load Balancer, Cloud CDN (SPA + `/api/v1/*` routing) |
| API | Node.js (Fastify) on Cloud Run |
| Workers | Cloud Run + Pub/Sub (resume pipeline, AI batch, exports) |
| Scheduler | Cloud Scheduler → Pub/Sub (`posting-reconcile`, cleanup) |
| Migrations | Cloud Run Job (`hiresphere-migrate`) on deploy |
| Auth | JWT (access + refresh), bcrypt passwords |
| Database | Cloud SQL PostgreSQL 15 (private IP), Drizzle ORM |
| Storage | GCS (resumes, exports, SPA assets); signed URLs |
| AI | Vertex AI (Gemini) |
| OCR | Document AI (low text-density PDFs) |
| Malware scan | TBD — ClamAV on Cloud Run or vendor API |
| Email | TBD — SendGrid or Mailgun (interview notifications) |
| Secrets | Secret Manager → Cloud Run env |
| Images | Artifact Registry |
| IaC | Terraform (`infra/terraform/`) |
| CI/CD | Cloud Build (+ GitHub trigger) |
| Observability | Cloud Logging, Monitoring, Error Reporting, Cloud Trace |

Platform adapters (`StorageProvider`, `JobQueue`, `AIProvider`) live in `packages/shared` so feature code stays deployment-agnostic. See `docs/gcp-infrastructure.md` §14.

## Environments

| Environment | GCP Project | Deploy trigger | Detail |
|---|---|---|---|
| dev | `hiresphere-dev` | Manual `terraform apply` | [gcp-infrastructure.md](../docs/gcp-infrastructure.md) §3 |
| staging | `hiresphere-staging` | Merge to `main` | |
| prod | `hiresphere-prod` | Tag `v*` + manual approval | |
| local | — | `docker compose up` (Postgres) | [gcp-infrastructure.md](../docs/gcp-infrastructure.md) §9 |

## Cross-Cutting

- **API:** `/api/v1`; auth required except login and health. Full surface in `docs/implementation-plan.md` §5.
- **Testing:** unit (middleware, state machines), integration (auth, resume pipeline, AI mocks), E2E Playwright. See `docs/implementation-plan.md` §9.
- **Local dev:** `docker compose` for Postgres; Cloud SQL Auth Proxy for remote DB. API/worker run on host.
- **Security:** RBAC on every endpoint; signed GCS URLs; rate limit on login (app + Cloud Armor in prod).
- **Observability:** structured logs with `requestId` / `trace`; DLQ alert on `hiresphere-dlq`. See `docs/architecture.md` §8.

## Open Decisions

| Topic | Options | Blocking |
|---|---|---|
| Malware scan | ClamAV on Cloud Run vs vendor API | `feat/resume-upload-pipeline` |
| Email provider | SendGrid vs Mailgun | `feat/interview-scheduling` |
| Refresh token transport | Response body vs httpOnly cookie | `feat/auth-rest-login` |
| AI sync vs async | Inline Vertex for small requests vs always Pub/Sub | `feat/ai-candidate-ranking` |

## Branch Strategy

| Prefix | Purpose | Examples |
|---|---|---|
| `infra/` | Infrastructure and platform scaffolding | `infra/gcp-terraform`, `infra/monorepo-scaffold` |
| `feat/` | Product features | `feat/auth-rest-login`, `feat/candidate-database` |

**Canonical merge order:** see [feature-branches.md](../docs/feature-branches.md) and [implementation-plan.md](../docs/implementation-plan.md) §10.

Infra first: `infra/gcp-terraform` → `infra/monorepo-scaffold` → `infra/database-foundation` → `infra/design-system-app-shell`.

Features 1–19 follow in dependency order. **`feat/responsible-ai-governance` merges twice:** prompt registry early (before JD/ranking features) and full governance UI last.

## Spec Index

| Spec | Branch | Description |
|---|---|---|
| [infrastructure](./specs/infrastructure/spec.md) | `infra/gcp-terraform` | GCP Terraform, Cloud Run, Cloud SQL, GCS, Pub/Sub, LB/CDN |
| [monorepo-scaffold](./specs/monorepo-scaffold/spec.md) | `infra/monorepo-scaffold` | pnpm workspaces, docker-compose, Dockerfiles, platform adapters |
| [database-foundation](./specs/database-foundation/spec.md) | `infra/database-foundation` | Drizzle, Phase 0 schema, migrations, seeds, migrate job |
| [app-shell](./specs/app-shell/spec.md) | `infra/design-system-app-shell` | Sidebar shell, design tokens, page templates |
| [auth](./specs/auth/spec.md) | `feat/auth-rest-login` | REST username/password login |
| [user-activation](./specs/user-activation/spec.md) | `feat/user-activation-access` | User activation and access enforcement |
| [admin-rbac](./specs/admin-rbac/spec.md) | `feat/admin-cockpit-rbac` | Users, roles, permissions, page/action matrix |
| [job-descriptions](./specs/job-descriptions/spec.md) | `feat/job-description-workspace` | JD workspace, AI draft, human approval |
| [job-postings](./specs/job-postings/spec.md) | `feat/job-posting-management`, `feat/posting-closure-rules` | Postings, vacancies, closure rules |
| [resume-upload](./specs/resume-upload/spec.md) | `feat/resume-upload-pipeline` | PDF upload, scan, parse, OCR, storage |
| [candidates](./specs/candidates/spec.md) | `feat/candidate-database` | Dedup, versions, links, history |
| [ai-ranking](./specs/ai-ranking/spec.md) | `feat/ai-candidate-ranking` | Ranking, fitment, gaps, interview questions |
| [candidate-resurfacing](./specs/candidate-resurfacing/spec.md) | `feat/candidate-resurfacing` | Existing-candidate match, priority lane |
| [shortlisting](./specs/shortlisting/spec.md) | `feat/shortlisting-workflow` | Mandatory reason capture |
| [interview-scheduling](./specs/interview-scheduling/spec.md) | `feat/interview-scheduling` | Scheduling task workflow |
| [interview-console](./specs/interview-console/spec.md) | `feat/interview-console` | Notes, rounds, versioning |
| [scorecards](./specs/scorecards/spec.md) | `feat/scorecard-center`, `feat/ai-assisted-scorecards` | Scorecards, AI assist, approval |
| [priority-selection](./specs/priority-selection/spec.md) | `feat/priority-selection` | Up to 5 picks per finite vacancy |
| [offers-onboarding](./specs/offers-onboarding/spec.md) | `feat/offer-onboarding-tracker` | Offers, onboarding, Hubble ID capture |
| [dashboards-audit](./specs/dashboards-audit/spec.md) | `feat/dashboards-reports-audit` | Dashboards, reports, audit, exports |
| [responsible-ai](./specs/responsible-ai/spec.md) | `feat/responsible-ai-governance` (partial + full) | Prompt registry early; governance UI last |

## Related Documents

- `docs/implementation-playbook.md` — **phase-wise implementation guide with copy-paste prompts**
- `docs/feature-branches.md` — branch index and merge order
- `docs/implementation-plan.md` — sequencing, API surface, per-branch deliverables
- `docs/gcp-infrastructure.md` — Terraform layout and GCP service map
- `docs/database-schema.md` — PostgreSQL schema
- `docs/architecture.md` — system diagrams and data flows
- `layout-spec.md`, `design-spec.md` — UI specifications

# HireSphere — Project Context

## Overview

HireSphere is an end-to-end, AI-assisted recruitment platform: job descriptions, postings,
candidate intake, vector-backed ranking, interview workflows, scorecards, offers, onboarding,
insight, and communications. Deployed on **Google Cloud Platform**. Visual system:
`design-spec.md` and `layout-spec.md`.

**Product architecture (feature domains):** [docs/product-architecture.md](../docs/product-architecture.md)

## Goals

- Single workspace from JD drafting through hire for recruiters and hiring managers.
- HireSphere-owned identity: branded login, activation, RBAC, and audit — **not** external SSO.
- Enforce responsible AI: advisory-only, versioned prompts, logged runs, human approval.
- Support finite and unlimited vacancy postings with distinct closure rules.
- Resurface strong candidates across postings; keep internal communications governed.

## Non-Goals (Explicit Exclusions)

- **Hubble SSO / Hubble login** — authentication is username + password via HireSphere REST API only.
- **Miracle branding** — HireSphere design tokens and assets only.
- Hubble appears only as an optional **external employee ID** on offers/onboarding (`hubble_id`).
- Candidate-facing email and public candidate portal are **gated / out of v1 core** unless product unlocks them.

## Governing rule

**AI recommends. A human decides, always.** No code path lets AI alone reject, shortlist, select,
offer, hire, or close a candidate.

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
└── docs/                  # Planning docs
```

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS v4 → GCS + Cloud CDN |
| Edge | HTTPS Load Balancer, Cloud CDN |
| API | Node.js (Fastify) on Cloud Run |
| Workers | Cloud Run + Pub/Sub |
| Scheduler | Cloud Scheduler → Pub/Sub |
| Migrations | Cloud Run Job (`hiresphere-migrate`) |
| Auth | **HireSphere JWT** (access + refresh), bcrypt — custom login page |
| Database | Cloud SQL PostgreSQL 15 (private IP), Drizzle ORM |
| Storage | GCS (resumes, exports, SPA); signed URLs |
| AI | Vertex AI (Gemini) via governed gateway + `prompt_registry` |
| Retrieval | Vertex AI Vector Search (embeddings; query-time auth filters) |
| OCR | Document AI (low text-density PDFs) |
| Malware scan | TBD — ClamAV on Cloud Run or vendor API |
| Email | TBD — SendGrid or Mailgun (**internal** recipients in v1) |
| Secrets | Secret Manager → Cloud Run env |
| IaC | Terraform (`infra/terraform/`) |
| CI/CD | Cloud Build (+ GitHub trigger) |

## Product feature domains

| Domain | What it delivers | Primary branches / specs |
|---|---|---|
| **Identity & access** | Custom login, activation, RBAC, admin | `auth`, `user-activation`, `admin-rbac` |
| **AI platform governance** | Gateway, prompts, AIRun, advisory-only | `responsible-ai` |
| **Design system** | Tokens, shell, tables, forms | `app-shell` |
| **Hiring & postings** | JD + postings + closure | `job-descriptions`, `job-postings` |
| **Candidate intake** | Upload, scan, parse, dedup, versions | `resume-upload`, `candidates` |
| **Matching & ranking** | Embeddings, vector search, ranking board | `ai-ranking` |
| **Interview pipeline** | Shortlist, schedule, notes, scorecards | `shortlisting`, `interview-*`, `scorecards` |
| **Decision & offers** | Priority, offers, onboarding, close | `priority-selection`, `offers-onboarding` |
| **Insight & reporting** | Dashboards, aging, audit, exports | `dashboards-audit` |
| **Communications** | Internal notify, resurfacing, priority lane | `candidate-resurfacing` |

Full detail: [docs/product-architecture.md](../docs/product-architecture.md)

## Environments

| Environment | GCP Project | Deploy trigger |
|---|---|---|
| dev | `hiresphere-dev` | Manual `terraform apply` |
| staging | `hiresphere-staging` | Merge to `main` |
| prod | `hiresphere-prod` | Tag `v*` + manual approval |
| local | — | `docker compose up` |

## Cross-Cutting

- **API:** `/api/v1`; auth required except login and health.
- **Testing:** unit, integration, Playwright E2E — see `docs/implementation-plan.md` §9.
- **Security:** RBAC on every endpoint; signed GCS URLs; login rate limit (+ Cloud Armor in prod).
- **Observability:** structured logs; DLQ alert on `hiresphere-dlq`.

## Open Decisions

| Topic | Options | Blocking |
|---|---|---|
| Malware scan | ClamAV vs vendor API | Candidate intake |
| Email provider | SendGrid vs Mailgun | Communications / scheduling |
| Refresh token transport | Response body vs httpOnly cookie | Identity & access |
| AI sync vs async | Inline Vertex vs always Pub/Sub | Matching & ranking |
| Vector index sizing | Dev vs prod Vertex AI Vector Search tiers | Matching & ranking |

## Branch Strategy

| Prefix | Purpose |
|---|---|
| `infra/` | Platform scaffolding |
| `feat/` | Product features |

**Canonical merge order:** [feature-branches.md](../docs/feature-branches.md) ·
[implementation-playbook.md](../docs/implementation-playbook.md)

Infra first, then Identity → AI governance (partial) → Design system already in infra →
Hiring → Intake → Matching → Interview → Decision → Insight ∥ Communications.
`feat/responsible-ai-governance` merges **twice** (registry early, full UI late).

## Spec Index

| Spec | Branch | Description |
|---|---|---|
| [infrastructure](./specs/infrastructure/spec.md) | `infra/gcp-terraform` | GCP Terraform, Cloud Run, SQL, GCS, Pub/Sub, LB/CDN |
| [monorepo-scaffold](./specs/monorepo-scaffold/spec.md) | `infra/monorepo-scaffold` | pnpm workspaces, adapters, health |
| [database-foundation](./specs/database-foundation/spec.md) | `infra/database-foundation` | Drizzle, Phase 0 schema, migrate job |
| [app-shell](./specs/app-shell/spec.md) | `infra/design-system-app-shell` | Design system / shell |
| [auth](./specs/auth/spec.md) | `feat/auth-rest-login` | Custom REST login (not Hubble) |
| [user-activation](./specs/user-activation/spec.md) | `feat/user-activation-access` | Activation enforcement |
| [admin-rbac](./specs/admin-rbac/spec.md) | `feat/admin-cockpit-rbac` | Admin RBAC |
| [job-descriptions](./specs/job-descriptions/spec.md) | `feat/job-description-workspace` | JD workspace |
| [job-postings](./specs/job-postings/spec.md) | `feat/job-posting-management`, `feat/posting-closure-rules` | Postings + closure |
| [resume-upload](./specs/resume-upload/spec.md) | `feat/resume-upload-pipeline` | Resume pipeline |
| [candidates](./specs/candidates/spec.md) | `feat/candidate-database` | Candidate identity / Applications |
| [ai-ranking](./specs/ai-ranking/spec.md) | `feat/ai-candidate-ranking` | Matching & ranking |
| [candidate-resurfacing](./specs/candidate-resurfacing/spec.md) | `feat/candidate-resurfacing` | Communications / resurfacing |
| [shortlisting](./specs/shortlisting/spec.md) | `feat/shortlisting-workflow` | Shortlist |
| [interview-scheduling](./specs/interview-scheduling/spec.md) | `feat/interview-scheduling` | Scheduling + notify |
| [interview-console](./specs/interview-console/spec.md) | `feat/interview-console` | Notes / rounds |
| [scorecards](./specs/scorecards/spec.md) | `feat/scorecard-center`, `feat/ai-assisted-scorecards` | Scorecards |
| [priority-selection](./specs/priority-selection/spec.md) | `feat/priority-selection` | Priority picks |
| [offers-onboarding](./specs/offers-onboarding/spec.md) | `feat/offer-onboarding-tracker` | Offers / onboarding |
| [dashboards-audit](./specs/dashboards-audit/spec.md) | `feat/dashboards-reports-audit` | Insight & reporting |
| [responsible-ai](./specs/responsible-ai/spec.md) | `feat/responsible-ai-governance` | AI platform governance |

## Related Documents

- `docs/product-architecture.md` — **feature-domain architecture**
- `docs/implementation-playbook.md` — phase-wise prompts
- `docs/feature-branches.md` — branch index
- `docs/implementation-plan.md` — API surface, deliverables
- `docs/gcp-infrastructure.md` — Terraform / GCP
- `docs/database-schema.md` — PostgreSQL schema
- `docs/architecture.md` — system diagrams
- `share/hiresphere-overview.html` — investor / stakeholder deck
- `layout-spec.md`, `design-spec.md` — UI specifications

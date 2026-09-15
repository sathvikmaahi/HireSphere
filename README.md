# HireSphere

End-to-end, AI-assisted recruitment platform: identity, governed AI, design system, hiring &
postings, candidate intake, matching & ranking, interview pipeline, decisions & offers, insight,
and communications. Deployed on **Google Cloud Platform**.

**Current status:** Planning and specification phase. Application code lands via infra and feature
branches. See [docs/product-architecture.md](docs/product-architecture.md).

---

## What it does

| Domain | Capabilities |
|---|---|
| **Identity & access** | Custom branded login (not Hubble SSO), activation, RBAC, admin cockpit |
| **AI platform governance** | Prompt registry, AIRun logging, advisory-only AI, feedback |
| **Design system** | Tokens, app shell, dense tables, page templates |
| **Hiring & postings** | AI-assisted JD drafting, finite/unlimited postings, closure rules |
| **Candidate intake** | Secure resume upload, scan, parse/OCR, dedup, versions |
| **Matching & ranking** | Embeddings, Vector Search retrieval, fitment/gaps, ranking board |
| **Interview pipeline** | Shortlisting, scheduling, structured notes, AI scorecards |
| **Decision & offers** | Priority picks, offers, onboarding, posting closure |
| **Insight & reporting** | Dashboards, aging, audit, AI agreement, exports |
| **Communications** | Internal notifications, candidate resurfacing, priority lane |

**Investor / stakeholder overview:** [share/hiresphere-overview.html](share/hiresphere-overview.html)

**Auth:** HireSphere-owned username + password (JWT + bcrypt). Hubble SSO is out of scope;
`hubble_id` is only an optional onboarding field.

---

## Repository layout

```
HireSphere/
├── docs/                          # Planning and implementation guides
│   ├── implementation-playbook.md # Phase-wise build guide with copy-paste AI prompts
│   ├── product-architecture.md    # Ten product feature domains
│   ├── implementation-plan.md     # Sequencing, API surface, per-branch deliverables
│   ├── feature-branches.md          # Branch index and merge order
│   ├── architecture.md              # System context, data flows, security boundaries
│   ├── database-schema.md           # PostgreSQL schema (Drizzle migrations)
│   └── gcp-infrastructure.md        # Terraform modules, GCP service map, environments
│
├── openspec/                      # Behavioral specs (OpenSpec format)
│   ├── project.md                   # Project context, tech stack, domain → spec index
│   ├── README.md                    # OpenSpec usage
│   └── specs/                       # Infra + product feature specs
│
├── share/                         # Externally shareable artifacts
│   └── hiresphere-overview.html     # Investor / stakeholder overview
│
├── infra/
│   └── terraform/
│       └── modules/
│           └── cloud-sql/           # Partial Terraform (full layout in gcp-infrastructure.md)
│
├── design-spec.md                 # Visual design tokens (color, type, elevation)
├── layout-spec.md                 # App shell, navigation, page templates, breakpoints
├── LICENSE
├── main.py                        # Placeholder (application stack is Node.js — see below)
└── pyproject.toml
```

### Planned application layout (not yet in repo)

Created by `infra/monorepo-scaffold` and feature branches:

```
apps/web/              # React 19 + Vite + Tailwind v4 → GCS + Cloud CDN
apps/api/              # Fastify REST API → Cloud Run
services/worker/       # Pub/Sub consumer → Cloud Run
packages/db/           # Drizzle schema, migrations, seeds
packages/shared/       # Types, Zod schemas, platform adapters
packages/ai-prompts/   # Versioned prompt templates
docker/                # api, worker, migrate Dockerfiles
cloudbuild.yaml
```

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS v4 |
| API | Node.js (Fastify) on Cloud Run |
| Workers | Cloud Run + Pub/Sub |
| Auth | HireSphere JWT + bcrypt (custom login — not Hubble SSO) |
| Database | Cloud SQL PostgreSQL 15, Drizzle ORM |
| Storage | Google Cloud Storage (resumes, exports, SPA assets) |
| AI | Vertex AI (Gemini) + governed prompt registry |
| Retrieval | Vertex AI Vector Search |
| OCR | Document AI |
| IaC | Terraform (`infra/terraform/`) |
| CI/CD | Cloud Build |

---

## Getting started

### 1. Read the docs (recommended order)

1. [openspec/project.md](openspec/project.md) — goals, non-goals, spec index
2. [docs/architecture.md](docs/architecture.md) — system diagrams and data flows
3. [docs/implementation-playbook.md](docs/implementation-playbook.md) — **start building here** (branch-by-branch prompts)
4. [docs/feature-branches.md](docs/feature-branches.md) — quick branch reference
5. [design-spec.md](design-spec.md) + [layout-spec.md](layout-spec.md) — UI implementation

### 2. Implement branch by branch

```bash
git checkout main && git pull
git checkout -b infra/gcp-terraform   # or the next branch in the playbook
# Follow the prompt and acceptance checks in docs/implementation-playbook.md
```

**Merge order:** infra branches first, then features 1–19. Full roadmap in [docs/implementation-playbook.md](docs/implementation-playbook.md#master-roadmap).

| Phase | Branches |
|---|---|
| Foundation | `infra/gcp-terraform` → `infra/monorepo-scaffold` → `infra/database-foundation` → `infra/design-system-app-shell` |
| Access | `feat/auth-rest-login` → `feat/user-activation-access` → `feat/admin-cockpit-rbac` |
| Jobs | `feat/job-description-workspace` → `feat/job-posting-management` |
| Candidates | `feat/resume-upload-pipeline` → `feat/candidate-database` → ranking, resurfacing |
| Pipeline | shortlisting → interviews → scorecards → priority selection |
| Close & observe | offers → closure rules → dashboards/audit → AI governance UI |

### 3. OpenSpec (optional)

```bash
npm install -g @fission-ai/openspec@latest
openspec init
openspec validate --strict
```

See [openspec/README.md](openspec/README.md).

### 4. Local development (after monorepo scaffold lands)

```bash
docker compose up          # Postgres (+ Pub/Sub emulator)
pnpm install
pnpm db:migrate && pnpm db:seed
pnpm dev                   # web + api (exact scripts TBD in monorepo-scaffold)
```

Details: [docs/gcp-infrastructure.md](docs/gcp-infrastructure.md) §9 (local dev).

---

## Environments

| Environment | GCP Project | Deploy trigger |
|---|---|---|
| dev | `hiresphere-dev` | Manual `terraform apply` |
| staging | `hiresphere-staging` | Merge to `main` |
| prod | `hiresphere-prod` | Tag `v*` + manual approval |
| local | — | `docker compose up` |

---

## Scope exclusions

- **Hubble SSO** — auth is username + password via REST API only
- **Miracle branding** — HireSphere design tokens and assets only
- `hubble_id` on offers/onboarding is an optional external employee ID field, not an identity provider

---

## Open decisions

Resolve before the blocking step (see [openspec/project.md](openspec/project.md)):

| Topic | Options | Blocks |
|---|---|---|
| Malware scan | ClamAV on Cloud Run vs vendor API | `feat/resume-upload-pipeline` |
| Email provider | SendGrid vs Mailgun | `feat/interview-scheduling` |
| Refresh token transport | Response body vs httpOnly cookie | `feat/auth-rest-login` |
| AI sync vs async | Inline Vertex vs always Pub/Sub | `feat/ai-candidate-ranking` |

---

## Documentation index

| Document | Purpose |
|---|---|
| [docs/implementation-playbook.md](docs/implementation-playbook.md) | Phase-wise implementation with copy-paste prompts |
| [docs/product-architecture.md](docs/product-architecture.md) | Ten product feature domains |
| [docs/implementation-plan.md](docs/implementation-plan.md) | API surface, schema overview, per-branch deliverables |
| [docs/feature-branches.md](docs/feature-branches.md) | Branch naming and merge order |
| [docs/architecture.md](docs/architecture.md) | System context, auth flow, pipeline state machine |
| [docs/database-schema.md](docs/database-schema.md) | PostgreSQL tables and enums |
| [docs/gcp-infrastructure.md](docs/gcp-infrastructure.md) | Terraform layout, GCP services, CI/CD |
| [openspec/project.md](openspec/project.md) | Canonical project context and spec index |
| [openspec/specs/](openspec/specs/) | Behavioral requirements per feature |
| [design-spec.md](design-spec.md) | Color, typography, components |
| [layout-spec.md](layout-spec.md) | App shell, sidebar, grids, breakpoints |
| [share/hiresphere-overview.html](share/hiresphere-overview.html) | Investor / stakeholder overview |

---

## License

See [LICENSE](LICENSE).

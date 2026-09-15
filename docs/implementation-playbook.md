# HireSphere — Implementation Playbook

Phase-wise, feature-wise implementation guide for building HireSphere branch by branch. **Use this document to prompt AI assistants** — each section includes a copy-paste prompt, deliverables, and acceptance checks.

**Related:** [product-architecture.md](./product-architecture.md) · [project.md](../openspec/project.md) · [implementation-plan.md](./implementation-plan.md) · [gcp-infrastructure.md](./gcp-infrastructure.md) · [database-schema.md](./database-schema.md) · [feature-branches.md](./feature-branches.md)

### Product domains (architecture map)

| Domain | Playbook steps |
|---|---|
| Design system | 0.4 |
| Identity & access *(custom login, not Hubble)* | 1.1–1.3 |
| AI platform governance | 1.4 (partial) + 5.4 (full) |
| Hiring & postings | 2.1–2.2 + 5.2 |
| Candidate intake | 3.1–3.2 |
| Matching & ranking | 3.3 |
| Interview pipeline | 4.1–4.5 |
| Decision & offers | 4.6 + 5.1–5.2 |
| Insight & reporting | 5.3 |
| Communications | 3.4 (+ notify in 4.2) |

---

## How to use this playbook

### Workflow per branch

```bash
git checkout main && git pull
git checkout -b <branch-name>
# implement using the prompt below
# run acceptance checks
# open PR → main
```

### Prompting rules

1. **One branch at a time** — never mix infra and feature code in one PR.
2. **Attach context** — paste the **Global context** block + the **branch prompt** for that step.
3. **Point to specs** — each step lists its OpenSpec file; ask the agent to read it first.
4. **Verify before merge** — run the acceptance checklist at the end of each step.
5. **Migrations are additive** — never edit shipped migration files; add new ones only.

### Global context (paste at the start of every prompt)

```
Project: HireSphere — GCP-hosted recruitment platform.
Stack: React 19 + Vite + Tailwind v4 (SPA → GCS/CDN), Fastify API on Cloud Run,
Cloud Run worker + Pub/Sub, Cloud SQL PostgreSQL 15 + Drizzle, Vertex AI Gemini,
Vertex AI Vector Search for retrieval.
API base: /api/v1. Auth: HireSphere JWT + bcrypt custom login — NOT Hubble SSO.
No Miracle branding. AI is advisory-only; humans decide all hiring transitions.
Domains: identity, AI governance, design system, hiring/postings, candidate intake,
matching/ranking, interview pipeline, decision/offers, insight, communications.
Design: layout-spec.md + design-spec.md. See docs/product-architecture.md.
Monorepo: apps/web, apps/api, services/worker, packages/{db,shared,ai-prompts}.
```

---

## Master roadmap

| Step | Phase | Branch | Depends on | OpenSpec |
|---|---|---|---|---|
| 0.1 | Foundation | `infra/gcp-terraform` | GCP projects | [infrastructure](../openspec/specs/infrastructure/spec.md) |
| 0.2 | Foundation | `infra/monorepo-scaffold` | 0.1 | [monorepo-scaffold](../openspec/specs/monorepo-scaffold/spec.md) |
| 0.3 | Foundation | `infra/database-foundation` | 0.2 | [database-foundation](../openspec/specs/database-foundation/spec.md) |
| 0.4 | Foundation | `infra/design-system-app-shell` | 0.3 | [app-shell](../openspec/specs/app-shell/spec.md) |
| 1.1 | Access | `feat/auth-rest-login` | 0.4 | [auth](../openspec/specs/auth/spec.md) |
| 1.2 | Access | `feat/user-activation-access` | 1.1 | [user-activation](../openspec/specs/user-activation/spec.md) |
| 1.3 | Access | `feat/admin-cockpit-rbac` | 1.2 | [admin-rbac](../openspec/specs/admin-rbac/spec.md) |
| 1.4 | Access | `feat/responsible-ai-governance` **(partial)** | 1.3 | [responsible-ai](../openspec/specs/responsible-ai/spec.md) |
| 2.1 | Jobs | `feat/job-description-workspace` | 1.4 | [job-descriptions](../openspec/specs/job-descriptions/spec.md) |
| 2.2 | Jobs | `feat/job-posting-management` | 2.1 | [job-postings](../openspec/specs/job-postings/spec.md) |
| 3.1 | Candidates | `feat/resume-upload-pipeline` | 2.2 | [resume-upload](../openspec/specs/resume-upload/spec.md) |
| 3.2 | Candidates | `feat/candidate-database` | 3.1 | [candidates](../openspec/specs/candidates/spec.md) |
| 3.3 | Candidates | `feat/ai-candidate-ranking` | 3.2 | [ai-ranking](../openspec/specs/ai-ranking/spec.md) |
| 3.4 | Candidates | `feat/candidate-resurfacing` | 3.2 | [candidate-resurfacing](../openspec/specs/candidate-resurfacing/spec.md) |
| 4.1 | Pipeline | `feat/shortlisting-workflow` | 3.2 | [shortlisting](../openspec/specs/shortlisting/spec.md) |
| 4.2 | Pipeline | `feat/interview-scheduling` | 4.1 | [interview-scheduling](../openspec/specs/interview-scheduling/spec.md) |
| 4.3 | Pipeline | `feat/interview-console` | 4.2 | [interview-console](../openspec/specs/interview-console/spec.md) |
| 4.4 | Pipeline | `feat/scorecard-center` | 4.3 | [scorecards](../openspec/specs/scorecards/spec.md) |
| 4.5 | Pipeline | `feat/ai-assisted-scorecards` | 4.4 | [scorecards](../openspec/specs/scorecards/spec.md) |
| 4.6 | Pipeline | `feat/priority-selection` | 4.5 | [priority-selection](../openspec/specs/priority-selection/spec.md) |
| 5.1 | Close | `feat/offer-onboarding-tracker` | 4.6 | [offers-onboarding](../openspec/specs/offers-onboarding/spec.md) |
| 5.2 | Close | `feat/posting-closure-rules` | 5.1 | [job-postings](../openspec/specs/job-postings/spec.md) |
| 5.3 | Observe | `feat/dashboards-reports-audit` | 1.3 | [dashboards-audit](../openspec/specs/dashboards-audit/spec.md) |
| 5.4 | Observe | `feat/responsible-ai-governance` **(full UI)** | 5.3 | [responsible-ai](../openspec/specs/responsible-ai/spec.md) |

**Parallel after 1.3:** Steps 2.x, 3.x, and 5.3 can run in parallel if teams coordinate migrations.

**Open decisions (resolve before blocked step):**

| Topic | Options | Blocks |
|---|---|---|
| Malware scan | ClamAV on Cloud Run vs vendor API | 3.1 |
| Email provider | SendGrid vs Mailgun | 4.2 |
| Refresh token transport | Response body vs httpOnly cookie | 1.1 |
| AI sync vs async | Inline Vertex vs always Pub/Sub | 3.3 |

---

# Phase 0 — Foundation

## Step 0.1 — `infra/gcp-terraform`

**Goal:** All GCP resources, Dockerfiles, and Cloud Build — no application feature code.

**Terraform commits (in order):**

1. `networking`, `iam`, `secrets`
2. `artifact-registry`, `cloud-sql`
3. `storage`, `pubsub`
4. `cloud-run` (placeholder images)
5. `load-balancer`
6. `observability`, `cicd`, `cloudbuild.yaml`

**Key files to create:**

```
infra/terraform/modules/{networking,iam,secrets,artifact-registry,cloud-sql,storage,pubsub,cloud-run,load-balancer,observability,cicd}/
infra/terraform/environments/{dev,staging,prod}/
docker/{api,worker,migrate}.Dockerfile
cloudbuild.yaml
```

**Acceptance:**

- [ ] `terraform plan` clean in dev, staging, prod
- [ ] `terraform apply` in staging succeeds
- [ ] Cloud SQL private IP only (no public endpoint)
- [ ] LB serves SPA bucket; `/api/v1/health` routes to Cloud Run
- [ ] Worker ingress internal only; Pub/Sub topics + DLQ exist
- [ ] Cloud Build trigger on merge to `main`

**Verify:**

```bash
cd infra/terraform/environments/dev && terraform init && terraform plan
terraform output   # LB IP, SQL connection name, bucket names
curl -s "https://<staging-domain>/api/v1/health"
```

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: infra/gcp-terraform

Read first:
- openspec/specs/infrastructure/spec.md
- docs/gcp-infrastructure.md (sections 5–8, 13)

Scope: Terraform modules + environments (dev/staging/prod), docker/*.Dockerfile,
cloudbuild.yaml. NO application feature code.

Create infra/terraform/ with modules in this apply order:
networking → iam → secrets → artifact-registry → cloud-sql → storage → pubsub
→ cloud-run → load-balancer → observability → cicd.

Requirements:
- Cloud SQL private IP only, SSL, VPC connector for Cloud Run
- GCS buckets: web, resumes, exports
- Pub/Sub: resume-processing, ai-ranking, report-export, posting-reconcile, DLQ
- Cloud Run: API (LB-facing), worker (internal only), migrate job
- HTTPS LB: CDN → web bucket, /api/v1/* → API
- Secrets in Secret Manager; service accounts per gcp-infrastructure.md §5

Use placeholder container images for Cloud Run until monorepo-scaffold lands.
Include terraform.tfvars.example per environment.
```

</details>

---

## Step 0.2 — `infra/monorepo-scaffold`

**Goal:** Runnable empty app locally; deployable skeleton to staging.

**Key files:**

```
package.json, pnpm-workspace.yaml
apps/web/, apps/api/, services/worker/
packages/db/, packages/shared/, packages/ai-prompts/
docker-compose.yml
.env.example
packages/shared/src/adapters/{storage,queue,ai}.ts
apps/api/src/routes/health.ts
```

**Acceptance:**

- [ ] `pnpm install` from root succeeds
- [ ] `docker compose up postgres` + `pnpm dev` in apps/api → health 200
- [ ] Platform adapters defined; feature code won't import GCS/Pub/Sub/Vertex directly
- [ ] Docker images build via Cloud Build

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: infra/monorepo-scaffold (merge after infra/gcp-terraform).

Read: openspec/specs/monorepo-scaffold/spec.md

Create pnpm monorepo:
- apps/web (Vite React stub)
- apps/api (Fastify, GET /api/v1/health)
- services/worker (Pub/Sub consumer stub)
- packages/db (empty schema placeholder)
- packages/shared (Zod, types, platform adapters)
- packages/ai-prompts (empty)

Add docker-compose.yml: Postgres 15 + Pub/Sub emulator.
Add .env.example with DATABASE_URL, JWT_SECRET, JWT_REFRESH_SECRET,
GCS_BUCKET_RESUMES, GCP_PROJECT_ID.

Implement StorageProvider (GCS), JobQueue (Pub/Sub), AIProvider (Vertex) interfaces
in packages/shared with GCP implementations.

Match Dockerfiles to infra/gcp-terraform. Ensure Cloud Build can build and deploy.
```

</details>

---

## Step 0.3 — `infra/database-foundation`

**Goal:** Drizzle migrations, Phase 0 tables, seeds, migrate job wired.

**Migration:** `0001_identity_audit.sql`

**Tables:** `users`, `roles`, `permissions`, `role_permissions`, `user_roles`, `page_action_matrix`, `activation_tokens`, `audit_events`

**Acceptance:**

- [ ] `pnpm db:migrate && pnpm db:seed` works locally
- [ ] Health endpoint returns 200 only when DB connected
- [ ] Cloud Run migrate job runs before API deploy in Cloud Build

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: infra/database-foundation (after monorepo-scaffold).

Read: openspec/specs/database-foundation/spec.md, docs/database-schema.md (Phase 0)

Add Drizzle to packages/db:
- schema/ for Phase 0 tables and enums
- migrations/0001_identity_audit.sql
- seed: roles (admin, recruiter, hiring_manager, interviewer, viewer), dev admin user
- connection helper: local TCP + Cloud SQL connector for Run

Upgrade GET /api/v1/health to fail when DB unreachable.
Wire migrate Cloud Run Job in cloudbuild.yaml (migrate before API deploy).
```

</details>

---

## Step 0.4 — `infra/design-system-app-shell`

**Goal:** UI shell, design tokens, page templates — no product features.

**Routes:** `/login` (bare shell), `/dev/shell` (demo), authenticated layout wrapper

**Acceptance:**

- [ ] Three shell states: authenticated, bare/public, presentation
- [ ] Sidebar 224/56px collapse, persisted; navy sidebar ignores theme toggle
- [ ] All design-spec CSS variables (light/dark)
- [ ] Page templates: browsable grid, detail split (280px rail), full-bleed
- [ ] Unauthenticated users redirect to `/login`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: infra/design-system-app-shell.

Read: openspec/specs/app-shell/spec.md, layout-spec.md, design-spec.md

In apps/web:
- CSS variables for all design-spec tokens; Plus Jakarta Sans + JetBrains Mono
- Components: Button (6 variants), Badge, Input, Card
- App shell: sidebar nav (placeholder items), main content, bare header
- Route guards (redirect to /login)
- Page templates: BrowsableGrid, DetailSplit, FullBleed
- Floating action panel (hidden when empty)
- /dev/shell demo route showing all shell states and breakpoints

No product features yet — shell and tokens only.
```

</details>

---

# Phase 1 — Access

## Step 1.1 — `feat/auth-rest-login` (Feature 1)

**API:** `POST /auth/login`, `POST /auth/logout`, `POST /auth/refresh`, `GET /auth/me`

**DB:** uses Phase 0 `users` table

**Tests:** login success/fail, rate limit, refresh rotation, logout invalidates refresh

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/auth-rest-login (Feature 1).

Read: openspec/specs/auth/spec.md

Backend (apps/api):
- POST /api/v1/auth/login, logout, refresh; GET /api/v1/auth/me
- bcrypt cost 12; JWT access 15min, refresh 7d with rotation
- Rate limit: 5 failed attempts / 15 min / IP → 429
- Audit event on login

Frontend (apps/web):
- /login on bare shell with error states
- Token storage (pick httpOnly cookie OR response body — document choice)
- 401 interceptor → refresh → retry

Tests for login, rate limit, refresh, logout.
```

</details>

---

## Step 1.2 — `feat/user-activation-access` (Feature 2)

**API:** `POST /users/:id/activate`, `POST /users/:id/suspend`, `POST /users/:id/deactivate`

**Guard:** middleware rejects non-`active` users → 403 + reason code

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/user-activation-access (Feature 2).

Read: openspec/specs/user-activation/spec.md

Enforce user_status (pending|active|suspended|deactivated) in auth middleware.
Add activate/suspend/deactivate endpoints (admin-only stub OK until feature 3).
Pending/suspended users get 403 on all protected routes. Write audit events.
Frontend: blocked state for non-active users.
```

</details>

---

## Step 1.3 — `feat/admin-cockpit-rbac` (Feature 3)

**API:** CRUD `/users`, `/roles`, `GET/PUT /permissions/matrix`

**Middleware:** `requirePermission('resource:action')`

**Routes:** `/admin/users`, `/admin/roles`, `/admin/permissions`

**Seed:** default page/action matrix per implementation-plan.md

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/admin-cockpit-rbac (Feature 3).

Read: openspec/specs/admin-rbac/spec.md, docs/implementation-plan.md (Feature 3 matrix)

Backend: user/role CRUD, permission middleware, page_action_matrix API.
Frontend: /admin/users (grid + drawer), /admin/roles, /admin/permissions matrix.
Sidebar Admin nav gated by admin:access. Seed default permissions and matrix.
```

</details>

---

## Step 1.4 — `feat/responsible-ai-governance` (PARTIAL — prompt registry only)

**Migration:** `0004_ai.sql` (partial) — `prompt_registry`, `ai_runs`, `ai_run_outputs`, feedback tables

**Scope this merge:** DB tables + `resolvePrompt(name)` helper + seed v1 prompts. **No admin UI yet.**

**Prompts to seed:** `jd_draft`, `candidate_rank`, `fitment_summary`, `gap_summary`, `interview_questions`, `scorecard_draft`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/responsible-ai-governance — PARTIAL merge only.

Read: openspec/specs/responsible-ai/spec.md, docs/database-schema.md (AI tables)

PARTIAL scope (no UI):
- Migration 0004_ai.sql: prompt_registry, ai_runs, ai_run_outputs, ai_run_feedback
- packages/ai-prompts: seed v1 prompt templates
- packages/shared: resolveActivePrompt(name), createAiRun(), completeAiRun()
- Seed prompts.sql with all 6 prompt names in draft/active states

Do NOT build /admin/ai-governance UI — that ships in Step 5.4.
All later AI features must use prompt_registry from this step.
```

</details>

---

# Phase 2 — Jobs

## Step 2.1 — `feat/job-description-workspace` (Feature 4)

**Migration:** `0002_jobs.sql` (JD tables)

**API:** JD CRUD, `POST /:id/ai-draft`, `POST /:id/submit-review`, `POST /:id/approve`

**Status machine:** `draft` → `in_review` → `approved` → `archived`

**Route:** `/job-descriptions`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/job-description-workspace (Feature 4).

Read: openspec/specs/job-descriptions/spec.md

Migration 0002_jobs.sql: job_descriptions with versioning, status enum.
API: CRUD, submit-review, approve (requires job_descriptions:approve).
AI draft via Vertex + prompt_registry (jd_draft); log every call to ai_runs.
Frontend: /job-descriptions grid, detail editor + metadata rail, AI draft button.
Approved JDs read-only; new version forks draft.
```

</details>

---

## Step 2.2 — `feat/job-posting-management` (Feature 5)

**Migration:** `0002_jobs.sql` (posting tables if not done in 2.1)

**API:** Posting CRUD, `POST /:id/open`, `POST /:id/hold`, `POST /:id/close`

**Route:** `/postings`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/job-posting-management (Feature 5).

Read: openspec/specs/job-postings/spec.md (lifecycle only — closure rules are Step 5.2)

Postings linked to approved JDs only. vacancy_type: finite|unlimited.
Status: draft|open|on_hold|closed. filled_count denormalized (default 0).
Frontend: /postings grid with vacancy badge (3/5 or ∞), create flow, detail view.
```

</details>

---

# Phase 3 — Candidates

## Step 3.1 — `feat/resume-upload-pipeline` (Feature 6)

**Worker:** Pub/Sub `resume-processing` consumer

**API:** `POST /candidates/:id/resumes`, `GET /resumes/:id/status`

**Pipeline:** validate → temp GCS → malware scan → parse → OCR if needed → permanent storage

> **Prerequisite:** Decide malware scan (ClamAV vs vendor API).

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/resume-upload-pipeline (Feature 6).

Read: openspec/specs/resume-upload/spec.md, docs/architecture.md (§4 pipeline)

API: multipart PDF upload max 10MB, returns 202 + resumeFileId.
Worker: consume resume-processing topic — scan BEFORE parse; delete infected files.
Use StorageProvider + JobQueue adapters. Document AI OCR path for low text density.
Frontend: upload dropzone, status polling (uploading→scanning→parsing→ready|failed).
```

</details>

---

## Step 3.2 — `feat/candidate-database` (Feature 7)

**Migration:** `0003_candidates.sql`

**API:** `POST /candidates`, `POST /dedup-check`, merge flow, history feed

**Route:** `/candidates`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/candidate-database (Feature 7).

Read: openspec/specs/candidates/spec.md

Migration 0003_candidates.sql: candidates, resume_versions, candidate_posting_links,
candidate_history (append-only). Dedup: sha256(lower(email)|normalized_phone).
Frontend: /candidates grid, detail split (resume, versions, links, history).
Dedup warning modal on create.
```

</details>

---

## Step 3.3 — `feat/ai-candidate-ranking` (Feature 8)

**Worker:** Pub/Sub `ai-ranking` topic

**API:** `POST /postings/:id/rank`, `GET /postings/:id/rankings`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/ai-candidate-ranking (Feature 8).

Read: openspec/specs/ai-ranking/spec.md

Batch rank candidates per posting via worker + Vertex (candidate_rank prompt).
Store rank_score, fitment_summary, gap_summary, interview_questions_json
in ranking_results linked to ai_runs. Pin model_version per run.
Frontend: Rankings tab on posting detail, expandable rows, re-rank button.
```

</details>

---

## Step 3.4 — `feat/candidate-resurfacing` (Feature 9)

**API:** `GET /postings/:id/resurface-candidates`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/candidate-resurfacing (Feature 9).

Read: openspec/specs/candidate-resurfacing/spec.md

Suggest existing candidates for new postings (skills/title overlap).
Priority lane: stage=selected with no accepted offer on prior postings.
Frontend: Suggested candidates panel on posting detail, one-click add with source=resurface.
```

</details>

---

# Phase 4 — Selection Pipeline

## Step 4.1 — `feat/shortlisting-workflow` (Feature 10)

**Migration:** `0005_workflow.sql` (start)

**API:** `POST /postings/:id/shortlist` (reason required min 10 chars), `DELETE .../shortlist/:candidateId`

**Route:** `/shortlists`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/shortlisting-workflow (Feature 10).

Read: openspec/specs/shortlisting/spec.md

Shortlist requires reason (min 10 chars). Updates candidate_posting_links.stage
and candidate_history. Frontend: shortlist modal, /shortlists nav with count badge.
```

</details>

---

## Step 4.2 — `feat/interview-scheduling` (Feature 11)

**API:** `GET/POST/PATCH /interview-tasks`, `POST /:id/schedule`

> **Prerequisite:** Decide email provider (SendGrid vs Mailgun) or stub notifications.

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/interview-scheduling (Feature 11).

Read: openspec/specs/interview-scheduling/spec.md

interview_tasks: assignee, due_date, status (pending|scheduled|completed|cancelled).
Schedule sets scheduled_at, notifies assignee (email stub OK for v1).
Frontend: /interviews task queue, schedule modal, link to candidate detail.
```

</details>

---

## Step 4.3 — `feat/interview-console` (Feature 12)

**API:** `GET/POST /interview-rounds/:id/notes` with versioning

**Route:** `/interviews/:roundId`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/interview-console (Feature 12).

Read: openspec/specs/interview-console/spec.md

interview_rounds per candidate+posting. interview_notes versioned (never overwrite).
content_structured_json + ai_readable_json for downstream scorecards.
Frontend: round tabs, structured form, version history sidebar.
```

</details>

---

## Step 4.4 — `feat/scorecard-center` (Feature 13)

**API:** `GET/POST /scorecards` — only interviewed candidates

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/scorecard-center (Feature 13).

Read: openspec/specs/scorecards/spec.md (center only — not AI assist)

GET /scorecards JOIN requires completed interview_rounds; 403 otherwise.
Frontend: /scorecards grid, empty state explains interview gate.
CTA from interview console to create scorecard.
```

</details>

---

## Step 4.5 — `feat/ai-assisted-scorecards` (Feature 14)

**API:** `POST /scorecards/:id/ai-generate`, `POST /:id/approve`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/ai-assisted-scorecards (Feature 14).

Read: openspec/specs/scorecards/spec.md (AI assist sections)

AI generate from ai_readable_json + JD via scorecard_draft prompt.
Status: draft→ai_generated→in_review→approved. Requires scorecards:approve.
Frontend: side-by-side AI draft vs editable fields, approve flow, AI run metadata.
```

</details>

---

## Step 4.6 — `feat/priority-selection` (Feature 15)

**API:** `PUT /postings/:id/priority-selections` (max 5, finite postings only)

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/priority-selection (Feature 15).

Read: openspec/specs/priority-selection/spec.md

PUT accepts [{candidateId, rankOrder}] max 5 unique ranks for finite postings only.
DB constraint + transaction to prevent race. Frontend: drag-and-drop rank 1–5 panel.
```

</details>

---

# Phase 5 — Close & Observe

## Step 5.1 — `feat/offer-onboarding-tracker` (Feature 16)

**Migration:** `0006_offers.sql`

**API:** Offers CRUD, `PATCH /:id/hubble-id`

**Route:** `/offers`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/offer-onboarding-tracker (Feature 16).

Read: openspec/specs/offers-onboarding/spec.md

offers: draft|extended|accepted|declined|withdrawn. hubble_id optional string (NOT auth).
onboarding_checklist_json with completion timestamps.
On accepted: increment posting.filled_count. Frontend: /offers pipeline, Hubble ID field.
```

</details>

---

## Step 5.2 — `feat/posting-closure-rules` (Feature 17)

**API:** Auto-close finite when filled_count >= vacancy_count; scheduled reconcile

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/posting-closure-rules (Feature 17).

Read: openspec/specs/job-postings/spec.md (closure sections)

Auto-close finite postings when vacancies filled. Unlimited: manual close only.
posting_lifecycle_events for every transition. Worker handles posting-reconcile topic.
Frontend: close/hold actions, closure history, approaching-limit warning.
```

</details>

---

## Step 5.3 — `feat/dashboards-reports-audit` (Feature 18)

**Migration:** `0007_reporting.sql`

**Routes:** `/dashboard`, `/admin/audit`, `/admin/ai-runs`

**API:** `GET /dashboards/:key`, `GET /audit-events`, `POST /exports`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/dashboards-reports-audit (Feature 18).

Read: openspec/specs/dashboards-audit/spec.md

Dashboards: recruiter, hiring manager, admin views.
POST /exports → async worker → GCS exports bucket. Audit log search with filters.
AI run log viewer. PII redaction on exports per role. Migration 0007_reporting.sql.
```

</details>

---

## Step 5.4 — `feat/responsible-ai-governance` (FULL UI — Feature 19)

**Scope this merge:** Admin UI, feedback widgets, prompt diff viewer — registry already exists from Step 1.4

**Route:** `/admin/ai-governance`

<details>
<summary><strong>Prompt — copy/paste</strong></summary>

```
[Paste Global context]

Implement branch: feat/responsible-ai-governance — FULL UI merge (final).

Read: openspec/specs/responsible-ai/spec.md

Build on prompt_registry from partial merge (Step 1.4):
- /admin/ai-governance: prompt list, version diff, activate/retire
- POST /ai-runs/:id/feedback — thumbs + comment widget on all AI panels
- Dashboard tile: feedback sentiment, runs by model version
Do not recreate DB tables — extend UI and feedback endpoints only.
```

</details>

---

## Recruitment pipeline (reference)

Candidate stage on a posting (`candidate_posting_links.stage`):

```
applied → shortlisted → interview_scheduled → interviewed
  → scorecard_pending → scorecard_approved → priority_selected
  → offered → hired | rejected | withdrawn
```

| Transition | Feature step |
|---|---|
| → shortlisted | 4.1 |
| → interview_scheduled | 4.2 |
| → interviewed | 4.3 |
| → scorecard_* | 4.4–4.5 |
| → priority_selected | 4.6 |
| → offered / hired | 5.1 |
| posting closed | 5.2 |

---

## Cross-cutting (apply on every feature branch)

| Concern | Rule |
|---|---|
| Auth | JWT on all endpoints except login + health |
| RBAC | `requirePermission()` on every handler |
| Validation | Zod schemas in `packages/shared` |
| Audit | Write `audit_events` on mutating actions |
| AI | Resolve prompt from registry; log `ai_runs` |
| Tests | Unit for logic; integration for API; one E2E path by Feature 10 |
| Migrations | New file per branch; never edit shipped SQL |

---

## Quick prompt template (any step)

```
[Paste Global context]

Implement branch: <branch-name> (Step <X.Y>).

Read these files first:
- openspec/specs/<spec>/spec.md
- docs/implementation-playbook.md (Step <X.Y>)
- docs/database-schema.md (if migration needed)

Constraints:
- One feature per branch; match existing monorepo patterns
- Server-side enforcement for all auth/RBAC/business rules
- Follow layout-spec.md and design-spec.md for UI
- Add tests per acceptance criteria in the playbook step

Deliver: working code + migration (if any) + tests. List what you changed.
```

---

## Document index

| Need | Read |
|---|---|
| Requirements (GIVEN/WHEN/THEN) | `openspec/specs/<domain>/spec.md` |
| Table columns | `docs/database-schema.md` |
| GCP / Terraform | `docs/gcp-infrastructure.md` |
| Architecture diagrams | `docs/architecture.md` |
| API endpoint list | `docs/implementation-plan.md` §5 |
| UI layout | `layout-spec.md`, `design-spec.md` |

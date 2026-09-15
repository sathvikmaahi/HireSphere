# HireSphere — Database Schema

PostgreSQL schema for **Cloud SQL**. ORM: Drizzle. Migrations live in `packages/db/migrations/`.

See [implementation-plan.md](./implementation-plan.md) for feature mapping and
[product-architecture.md](./product-architecture.md) for domain ownership.

**Naming note:** Product language uses **Application** (candidate × posting). In v1 schema this
is represented by `candidate_posting_links` (plus stage/history). A later migration may rename
or introduce an `applications` view/table without breaking the domain model.

### Domain additions (planned tables)

| Table / concept | Domain | Notes |
|---|---|---|
| `prompt_registry`, `ai_runs`, `ai_run_outputs`, feedback | AI platform governance | Must exist before first Vertex call |
| `vector_index_records` | Matching & ranking | Embedding metadata + traceback to relational IDs |
| `match_suggestions` | Communications | Light resurfacing records; promote to Application on human action |
| `notifications`, `notification_templates` | Communications | Internal recipients only in v1 |
| Ranking provenance columns | Matching & ranking | resume_version, jd_version, prompt_version, model_version |

---

## Enums

```sql
CREATE TYPE user_status AS ENUM ('pending', 'active', 'suspended', 'deactivated');
CREATE TYPE jd_status AS ENUM ('draft', 'in_review', 'approved', 'archived');
CREATE TYPE vacancy_type AS ENUM ('finite', 'unlimited');
CREATE TYPE posting_status AS ENUM ('draft', 'open', 'on_hold', 'closed');
CREATE TYPE scan_status AS ENUM ('pending', 'clean', 'infected', 'failed');
CREATE TYPE parse_status AS ENUM ('pending', 'parsed', 'ocr_required', 'failed');
CREATE TYPE ai_run_status AS ENUM ('queued', 'running', 'completed', 'failed');
CREATE TYPE prompt_status AS ENUM ('draft', 'active', 'retired');
CREATE TYPE offer_status AS ENUM ('draft', 'extended', 'accepted', 'declined', 'withdrawn');
CREATE TYPE interview_task_status AS ENUM ('pending', 'scheduled', 'completed', 'cancelled');
CREATE TYPE scorecard_status AS ENUM ('draft', 'ai_generated', 'in_review', 'approved');
```

---

## Phase 0 — Identity & Audit

### users

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | `gen_random_uuid()` |
| username | VARCHAR(64) UNIQUE NOT NULL | Login identifier |
| email | VARCHAR(255) UNIQUE NOT NULL | |
| password_hash | VARCHAR(255) NOT NULL | bcrypt |
| status | user_status NOT NULL DEFAULT 'pending' | |
| activated_at | TIMESTAMPTZ | |
| activated_by | UUID FK → users | |
| last_login_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| updated_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### roles / permissions / role_permissions / user_roles

Standard RBAC junction tables. `permissions` uses `(resource, action)` unique pair.

### page_action_matrix

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| page_key | VARCHAR(64) NOT NULL | e.g. `job_postings` |
| action_key | VARCHAR(64) NOT NULL | e.g. `close` |
| required_permission_id | UUID FK → permissions | |
| description | TEXT | |

UNIQUE (`page_key`, `action_key`).

### activation_tokens

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| user_id | UUID FK → users | |
| token_hash | VARCHAR(255) NOT NULL | Store hash only |
| expires_at | TIMESTAMPTZ NOT NULL | |
| used_at | TIMESTAMPTZ | |

### audit_events

Append-only. Index on `(created_at DESC)`, `(actor_id)`, `(resource_type, resource_id)`.

---

## Phase 2 — Jobs

### job_descriptions

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| title | VARCHAR(255) NOT NULL | |
| content_json | JSONB NOT NULL | Structured sections |
| status | jd_status NOT NULL DEFAULT 'draft' | |
| version | INT NOT NULL DEFAULT 1 | |
| parent_version_id | UUID FK → job_descriptions | Null for v1 |
| created_by | UUID FK → users | |
| approved_by | UUID FK → users | |
| approved_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| updated_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### job_postings

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| job_description_id | UUID FK → job_descriptions | Must be approved |
| title | VARCHAR(255) NOT NULL | |
| vacancy_type | vacancy_type NOT NULL | |
| vacancy_count | INT | Required when finite; CHECK > 0 |
| filled_count | INT NOT NULL DEFAULT 0 | |
| status | posting_status NOT NULL DEFAULT 'draft' | |
| is_evergreen | BOOLEAN NOT NULL DEFAULT false | |
| closed_at | TIMESTAMPTZ | |
| closed_reason | VARCHAR(255) | |
| created_by | UUID FK → users | |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| updated_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

CHECK: `(vacancy_type = 'finite' AND vacancy_count IS NOT NULL) OR (vacancy_type = 'unlimited' AND vacancy_count IS NULL)`.

### posting_lifecycle_events

Event sourcing for open/hold/close/auto-close transitions.

---

## Phase 3 — Candidates

### candidates

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| display_name | VARCHAR(255) NOT NULL | |
| canonical_email | VARCHAR(255) | Nullable if unknown |
| canonical_phone | VARCHAR(32) | E.164 normalized |
| dedup_key | VARCHAR(64) UNIQUE | Hash for dedup |
| status | VARCHAR(32) NOT NULL DEFAULT 'active' | |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| updated_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### candidate_identities

Alternate emails/phones/names from resumes for merge review.

### resume_files

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| candidate_id | UUID FK → candidates | |
| blob_key | VARCHAR(512) NOT NULL | GCS object key |
| sha256 | VARCHAR(64) NOT NULL | |
| mime_type | VARCHAR(128) NOT NULL | |
| size_bytes | BIGINT NOT NULL | |
| scan_status | scan_status NOT NULL DEFAULT 'pending' | |
| parse_status | parse_status NOT NULL DEFAULT 'pending' | |
| ocr_required | BOOLEAN NOT NULL DEFAULT false | |
| uploaded_by | UUID FK → users | |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### resume_versions

Links parsed output to files; `is_current` partial unique index per candidate.

### candidate_posting_links

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| candidate_id | UUID FK → candidates | |
| posting_id | UUID FK → job_postings | |
| source | VARCHAR(32) NOT NULL | upload, resurface, referral |
| stage | VARCHAR(32) NOT NULL DEFAULT 'applied' | Pipeline stage |
| linked_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

UNIQUE (`candidate_id`, `posting_id`).

### candidate_history

Append-only event log per candidate.

---

## Phase 4 — AI

### prompt_registry

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| name | VARCHAR(64) NOT NULL | e.g. `jd_draft` |
| version | INT NOT NULL | |
| template | TEXT NOT NULL | With `{{placeholders}}` |
| parameters_schema | JSONB | |
| status | prompt_status NOT NULL DEFAULT 'draft' | |
| approved_by | UUID FK → users | |
| approved_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ NOT NULL DEFAULT now() | |

UNIQUE (`name`, `version`). Partial unique: one `active` per `name`.

### ai_runs / ai_run_outputs

Every AI invocation logged. `input_hash` for dedup/debug.

### ranking_results

UNIQUE (`posting_id`, `candidate_id`, `ai_run_id`).

---

## Phase 5 — Workflow

### shortlist_entries

| Column | Type | Notes |
|---|---|---|
| reason | TEXT NOT NULL | Min length enforced in API |
| shortlisted_by | UUID FK → users | |

UNIQUE (`posting_id`, `candidate_id`).

### interview_tasks / interview_rounds / interview_notes

`interview_notes.version` increments per round; never delete, only append.

### scorecards

FK to `interview_rounds`; API enforces round `status = completed`.

### priority_selections

| Column | Type | Notes |
|---|---|---|
| posting_id | UUID FK → job_postings | |
| candidate_id | UUID FK → candidates | |
| rank_order | SMALLINT NOT NULL | 1–5 |
| selected_by | UUID FK → users | |

UNIQUE (`posting_id`, `candidate_id`). UNIQUE (`posting_id`, `rank_order`).

Trigger or API: reject 6th insert for same `posting_id` where posting is finite.

---

## Phase 6 — Offers & Closure

### offers

| Column | Type | Notes |
|---|---|---|
| hubble_id | VARCHAR(64) | External employee ID; not auth |
| onboarding_checklist_json | JSONB | |
| onboarding_status | VARCHAR(32) | |

### posting_closure_rules

Links auto-close config to postings; populated when posting opens.

---

## Phase 7 — Reporting

### report_exports

Async export jobs with blob reference and status.

---

## Migration File Plan

| Migration | Branch | Tables |
|---|---|---|
| `0001_identity_audit.sql` | infra/database-foundation | users, roles, permissions, audit |
| `0002_jobs.sql` | job-description-workspace + job-posting-management | job_descriptions, job_postings, lifecycle |
| `0003_candidates.sql` | candidate-database | candidates, resumes, links, history |
| `0004_ai.sql` | responsible-ai-governance (partial) | prompt_registry, ai_runs |
| `0005_workflow.sql` | shortlisting → scorecards | shortlist through scorecards |
| `0006_offers.sql` | offer-onboarding-tracker | offers, closure rules |
| `0007_reporting.sql` | dashboards-reports-audit | report_exports, indexes |

Each feature branch adds only its migration(s); never edit shipped migrations.

---

## Seed Data

| Seed | Contents |
|---|---|
| `roles.sql` | admin, recruiter, hiring_manager, interviewer, viewer |
| `permissions.sql` | Full permission catalog |
| `matrix.sql` | Default page/action matrix |
| `prompts.sql` | v1 prompts (draft) for all AI features |
| `dev-users.sql` | Local admin + recruiter test accounts |

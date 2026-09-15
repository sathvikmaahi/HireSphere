# Sprint 0 — Outcome and Handover

Written at the close of Sprint 0, for whoever picks this change up next. It
records what was built, what changed in the plan while building it, what is
verified versus merely written, and what Sprint 1 needs to know before it
starts.

`proposal.md`, `design.md`, the delta specs and `tasks.md` remain the
authoritative planning artifacts. This document does not restate them; it
records what implementation revealed.

**Status: Sprint 0 is 13/13. Wave 1 is 13/196 (7%).**

---

## 1. What exists now

### Dev is a real, deployable environment

The full request path works and was verified live, not inferred from pipeline
status:

```
API Gateway → Cloud Run → IAM auth → private-IP Cloud SQL → query returns data
```

| Component | State |
| --- | --- |
| Project | `prj-talentsphere-dev-dm01`, in folder `fld-dev` |
| Database | Cloud SQL POSTGRES_16, private IP `10.98.0.6`, **no public address**, `ssl_mode = ENCRYPTED_ONLY` |
| Service | Cloud Run `crs-talentsphere-api`, direct VPC egress, `PRIVATE_RANGES_ONLY` |
| Ingress | API Gateway `talentsphere-gw-dev-7g3u3m8h.uc.gateway.dev` — the only route in |
| Migrations | Cloud Run job `crj-talentsphere-migrate`, run by the pipeline with `--wait` |
| Frontend | Bucket `prj-talentsphere-dev-dm01-frontend` (no CDN/LB edge yet) |

Only Local and Dev are provisioned. UAT and Prod are validated, planned, and
never applied — see `design.md` D15.

### Application code

Roughly 2,000 lines of Python and TypeScript, 115 backend tests and 8 frontend
tests. All of it is platform-layer; **no hiring feature exists yet**, by design.

- **Error handling** — one construction site for every API error body. Tests
  assert DSNs, prompts, tokens, stack traces and rejected input values cannot
  reach a response.
- **Observability** — correlation IDs in a contextvar that survives into
  background tasks, structlog JSON with trace/span IDs, OTel traces and metrics,
  sensitive keys redacted in a processor rather than at call sites.
- **Feature flags** — declared registry; an unknown flag raises rather than
  resolving false. Resolution: test override → environment kill switch → audited
  runtime config → declared default. A disabled capability answers 404.
- **Runtime configuration** — closed registry covering every value the
  `delivery-foundation` spec requires to be changeable without deployment. No
  unaudited setter exists.
- **Audit port** — `app/audit/port.py`. Sprint 1 replaces `LoggingAuditSink`
  with the durable writer; no call site changes.
- **Database access** — IAM authentication, two identities, no password
  anywhere.

### Pipeline

Deployment machinery comes from the shared landing-zone template, pinned:

```yaml
include:
  - project: dhanush.mallu16/gitlab-ci-templates
    ref: v0.9.1
    file: /templates/gcp-cloud-run-application.yml
```

TalentSphere adds what the template lacks: ruff, mypy, bandit, gitleaks,
release notes, and a provenance check. It **overrides** the template's
`test:backend` and `security:dependency-scan`, which are gated on
`exists: backend/requirements.txt` — this project uses `pyproject.toml`, so both
would silently skip with 115 tests behind them.

---

## 2. Plan changes made during Sprint 0

Three changes to the planning artifacts. Each was recorded rather than applied
silently.

### QA was dropped (`design.md` D15, amended)

The GCP organization is built around dev/uat/prod: `fld-nonprod` contains
`fld-dev` and `fld-uat`, there is no `fld-qa`, and each environment folder has
its own `prj-network-<env>` shared-VPC host. A QA stack would have described
infrastructure with nowhere to land — validating cleanly while being unapplyable
in principle rather than merely deferred.

Environments are now **Local, Dev, UAT, Prod**. `proposal.md`, `design.md`, the
`delivery-foundation` spec and tasks 1.3/1.4 were all reconciled.

### D18 added — the database has no password

The application and migration runner authenticate to Cloud SQL as
`CLOUD_IAM_SERVICE_ACCOUNT` users with short-lived tokens. Sprint 0 first
shipped a Terraform-generated password in Secret Manager, which satisfied the
spec but created a credential to protect. IAM authentication removes it instead.

Two identities, not one: the runtime holds DML only, `talentsphere-migrate` owns
the schema and holds DDL. D7 makes the audit tables append-only by granting the
runtime `INSERT`/`SELECT` only — decorative if the same role can `ALTER TABLE`.

### Task 1.7 was reopened, then completed

Marked complete on the strength of the Alembic tooling, then reopened: the
pipeline step could not reach a private-IP database from GitLab's shared
runners. Fixed properly with the Cloud Run migration job.

---

## 3. Structural change: adopting the landing zone

**The most consequential thing that happened in Sprint 0, and the most
expensive mistake.**

Sprint 0 initially built its own project vending, CI identity and pipeline,
because project memory recorded the landing zone as inaccessible. When the repos
turned out to be reachable, all of it was migrated.

### Ownership split

- **`platform-infra` creates projects.** `app-talentsphere-dev` owns the
  project, APIs, shared-VPC attachment, subnet grant, Artifact Registry
  (`images`), the `talentsphere-ci` service account, its WIF binding, and two
  state-bucket bindings.
- **This repo owns what runs inside.** Workload Terraform at a single `TF_ROOT`
  per environment, plus a thin `.gitlab-ci.yml`.

### State

Two prefixes in `gs://prj-bootstrap-dm01-tfstate`, matching demoapp:
`app-talentsphere-dev/` (platform) and `talentsphere-dev/` (workload).

### The lesson worth carrying

Two problems were independently rediscovered that the landing zone's own
comments already documented: `SERVICE_NETWORKING_NOT_ENABLED` needing the API on
the *service* project, and `storage.objects.list` being authorised against the
bucket so a prefix condition can never grant it.

**Check for existing platform repos before writing infrastructure.**

---

## 4. Defects found, and the pattern in them

Fourteen distinct defects were found and fixed. The instructive ones:

### Silent successes — the dangerous class

Three defects produced **green pipelines while doing nothing**:

1. **Three days of deploys that deployed nothing.** `ignore_changes` on the
   Cloud Run image meant Terraform discarded every new digest and kept the first
   image ever built — while environment variables *were* updated, so the service
   reported a fresh commit while running old code. Found by comparing the
   deployed `/build` response against the registry, not from any gate.
2. **A plan artifact that collected nothing.** GitLab does not expand variables
   in `artifacts:paths`, so the plan an approver reads before authorising an
   apply was never downloadable.
3. **`release:notes` behind an unclickable gate.** Blocking manual apply jobs
   pause the pipeline at their stage, so a job in a later stage could never run.

A green job and a healthy `/health` are both true of a service running stale
code. The gates that caught real problems were the ones asserting a **negative**
or comparing **two sources of truth**: gitleaks finding a credential-shaped DSN
in this project's own test, and the digest comparison now enforced by
`verify:provenance`.

### Local environments more permissive than deployed ones

- `requests` was present in the developer virtualenv through an unrelated
  package but absent from the slim container, so the IAM token refresh failed
  only in deployment. Fixed by declaring `google-auth[requests]`.
- The grant script was verified against local PostgreSQL as a **real
  superuser**; Cloud SQL gives you `cloudsqlsuperuser`, which cannot reassign
  schema ownership without role membership.

### One split-brain bug

`db/session.py` was updated for IAM authentication and `app/migrations/env.py`
was not. Both files were internally correct, so nothing caught it — migrations
kept using the password DSN and reached `127.0.0.1` inside a Cloud Run job. The
URL selection now lives in one module, and three tests assert the two agree.

---

## 5. Carry-forward items

Also in `KNOWN_ISSUES.md`.

| Item | Consequence |
| --- | --- |
| `audit_logs` narrowing unexercised | The `INSERT`/`SELECT`-only grant is verified against local PostgreSQL but has never run against a real table. **Task 2.3 must prove it on Cloud SQL.** |
| Schema ownership granted by hand | No automated identity can reassign schema ownership, so Cloud SQL Studio was used. Re-provisioning Dev or standing up UAT repeats that step. |
| Sprint 0 endpoints declare no permission | `/health`, `/build` and two diagnostics routes. **Task 8.2's enumeration must classify them, not skip them.** |
| Audit is a port, not a trail | Runtime-config changes emit through `LoggingAuditSink`, which logs `durable: false`. Sprint 1 substitutes the real writer. |
| `npm` registry blocked on the corporate network | `registry.npmjs.org` is filtered by hostname. The committed lockfile is canonical, so CI is fine; a developer on that network cannot `npm ci`. |
| `/build` reports `ref` and `pipeline_id` as unknown | The template passes only commit and digest, which is what traceability requires. |
| No CDN/load-balancer edge for the frontend | The bucket exists and the pipeline syncs into it; nothing serves it publicly yet. |

---

## 6. What Sprint 1 needs to decide

**Sprint 1 — Schema & Audit Substrate, 11 tasks.** The first real domain code:
`users`, `roles`, `user_roles`, `audit_logs`, the session store, the audit
writer, write-time redaction, audit search, and retention.

Two decisions are better made before code than during it:

### Task 2.7 depends on a Sprint 6 capability

"Audited, permission-gated export" needs the permission evaluator, which arrives
in Sprint 6. Sprint 1 can build the export, its classification label and its
audit event, but "permission-gated" has nothing to gate against.

Same shape as the audit port: build against an interface Sprint 6 fills. Worth
deciding deliberately — and if `tasks.md` should say so, that is an artifact
update rather than something to code around.

### Task 2.3 needs Cloud SQL, not local PostgreSQL

"Prove `UPDATE` and `DELETE` fail at the database level" cannot be proven where
the developer is a superuser. It needs the real instance with the real grants —
which means the test belongs in an integration suite that runs where Dev is
reachable, not in the unit suite.

### Sequencing note

Sprint 1 is almost entirely local backend code against the Postgres in
`docker-compose.yml`. It should move considerably faster than Sprint 0, whose
cost was infrastructure round-trips rather than code.

---

## 7. Where things live

| Path | Contents |
| --- | --- |
| `backend/app/` | FastAPI application: `core/`, `api/`, `db/`, `audit/`, `features/`, `runtime_config/`, `middleware/`, `migrations/` |
| `backend/sql/01_least_privilege.sql` | Database grants. Runs as instance admin; cannot be an Alembic migration because a role cannot grant itself privileges |
| `backend/Dockerfile` | One image, two entry points — the service, and the migration job with an overridden command |
| `frontend/src/api/client.ts` | Correlation IDs, error normalisation, runtime config from `config.json` |
| `infra/modules/workload/` | Everything that runs inside the project |
| `infra/environments/{dev,uat,prod}/` | One root module each; `dev` is `TF_ROOT` |
| `docker-compose.yml` | The Local environment. Postgres on **5433**, not 5432, to avoid colliding with a native install |
| `.gitleaks.toml` | Credential scanning rules and a deliberately narrow allowlist |
| `scripts/release_notes.py` | Release notes from commits, migrations and known issues |

Reference material, outside this repo:

- `git@gitlab.com:dhanush.mallu16/platform-infra.git` — folders, networks,
  bootstrap, and `app-talentsphere-dev`
- `git@gitlab.com:dhanush.mallu16/gitlab-ci-templates.git` — the pipeline
  template, consumed by pinned tag

# Platform Core

## Why

Every other TalentSphere feature runs on infrastructure and substrate that belongs to no
feature in particular: the GCP project and its pipeline, the database and how identities
reach it, the single ingress, the engine that executes state transitions, the engine that
notifies people about them, and the pattern by which long work happens off the request path.
Eleven of the twelve features consume this and none of them own it. `platform-core` owns it.

**Why now, and why this is not a greenfield proposal.** Roughly half of this scope is already
built, deployed, and verified live in GCP Dev — Sprint 0 of `talentsphere-wave-1-foundation`
shipped 13/13 tasks against a real `API Gateway → Cloud Run → IAM auth → private-IP Cloud SQL`
request path (see that change's `sprint-0-outcome.md`). What does *not* exist is the workflow
engine, the notification engine, and the async orchestration pattern. This change formalizes
the built half at the corrected backlog granularity established in `exploration-notes.md`
[D.9](../talentsphere/exploration-notes.md), and scopes the unbuilt half honestly rather than
re-proposing shipped work as if it were new.

Per [D.5](../talentsphere/exploration-notes.md) as adjusted to feature level by
[D.8.4](../talentsphere/exploration-notes.md), `talentsphere-wave-1-foundation` **splits, it
does not get rewritten** — its real content migrates onto the finer items, and Sprint 0's
historical record is referenced, never restated as future work.

## What Changes

Six backlog items, `TS-BL-001` through `TS-BL-006`, exactly as decomposed in D.9. Three
already exist in some form; three do not exist at all.

**Already real and live in Dev — formalized here, not rebuilt:**

- **`TS-BL-001` Base infrastructure and CI/CD.** Workload Terraform at one `TF_ROOT` per
  environment, consuming the shared landing-zone pipeline template by pinned tag, with the
  quality gates that template lacks (ruff, mypy, bandit, gitleaks, release notes, provenance
  verification). Ownership is split: `platform-infra` creates the project, WIF binding, CI
  service account and Artifact Registry; this repository owns what runs inside it.
  **Widened beyond its D.9 one-line title** to also cover the application-layer platform
  baseline Sprint 0 shipped alongside it — the global error handler, OpenTelemetry
  observability with correlation-ID propagation, the declared feature-flag registry, and the
  audited runtime-configuration store. No other item in the 79-item backlog covers these, and
  D.11 permits refining item boundaries within a feature. Residual scope: the frontend bucket
  has no CDN or load-balancer edge, so nothing serves it publicly yet.
- **`TS-BL-002` Cloud SQL and IAM database authentication.** Private-IP PostgreSQL 16, no
  public address, reached by two `CLOUD_IAM_SERVICE_ACCOUNT` identities holding short-lived
  tokens — a runtime identity with DML only, and a migration identity owning the schema and
  holding DDL. **The database has no password anywhere**: not in Terraform state, not in
  Secret Manager, not on a laptop. Residual scope: schema ownership is still granted by hand
  through Cloud SQL Studio because no automated identity holds the rights to reassign it.
- **`TS-BL-003` API Gateway ingress.** The only route into the system. The Cloud Run service
  grants `run.invoker` to the gateway's identity and to nothing else, so a direct call to the
  service URL is refused.

**Genuinely unbuilt — the real new work in this change:**

- **`TS-BL-004` Workflow / state-machine engine substrate.** The generic transition framework
  every domain state change executes through: validation, reason enforcement, actor
  attribution, transition records, concurrency safety, and declarative state-machine
  registration. Ships with **zero registered machines** — posting, Application, offer, and
  closure machines belong to the features that own them.
- **`TS-BL-005` Notification engine, internal delivery only.** Tasks and notifications with
  in-app and internal email channels, retry with backoff, and a **sender-side guard that
  refuses any recipient not resolving to an internal user record** — so the deferral of
  candidate-facing communication is a control, not a convention.
- **`TS-BL-006` Async orchestration pattern.** Pub/Sub → Eventarc → dispatcher → Workflows →
  Cloud Run Job, established once as a reusable pattern so that resume parsing, embedding,
  ranking, and scorecard generation each inherit it rather than inventing their own.

**Explicitly not in this change:** no domain state machines, no candidate-facing
communication, no permission evaluator or audit writer (those are `access-control-and-admin`),
no AI gateway (`ai-platform-governance`), and no visual components — the toast that renders a
notification is `design-system`'s `TS-BL-012`, which depends on this feature's payload shape.

## Capabilities

### New Capabilities

`openspec/specs/` is currently empty — nothing has been archived or synced yet — so every
capability below is new. Paths preserve the `platform/` organization established by
`talentsphere-wave-1-foundation`, and three of them (`delivery-foundation`, `workflow-engine`,
`notifications`) reuse that change's exact existing paths rather than inventing parallel ones.

- `platform/delivery-foundation`: infrastructure as code, environment topology, secret
  handling, versioned migrations, CI quality gates, release recoverability, feature flags,
  runtime configurability, observability signals, and secure error handling. *(`TS-BL-001`)*
- `platform/database-access`: passwordless database connectivity — IAM authentication,
  separated runtime and migration identities, private networking, and the least-privilege
  grants that make append-only guarantees enforceable rather than decorative. *(`TS-BL-002`)*
- `platform/api-ingress`: single controlled ingress — every request arrives through the
  gateway, direct service invocation is refused, and the ingress contract is generated from
  infrastructure rather than maintained by hand. *(`TS-BL-003`)*
- `platform/workflow-engine`: the generic state transition framework, framework-only, with no
  domain machines. *(`TS-BL-004`)*
- `platform/notifications`: the task and internal-notification substrate. *(`TS-BL-005`)*
- `platform/async-orchestration`: the reusable event-driven pattern for work that must not
  block an interactive request, including status retrieval and retry semantics. *(`TS-BL-006`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**Overlap to resolve outside this change:** `talentsphere-wave-1-foundation` remains an active,
unarchived change whose delta specs cover `platform/delivery-foundation`,
`platform/workflow-engine` and `platform/notifications`. Both changes therefore describe the
same capability paths until that change is retired or split per D.5. That migration spans
five features and is not this change's to perform.

## Impact

**Code and infrastructure already in the repository** — `infra/modules/workload/`,
`infra/environments/{dev,uat,prod}/`, `.gitlab-ci.yml`, `backend/app/core/`,
`backend/app/db/`, `backend/app/runtime_config/`, `backend/app/middleware/`,
`backend/sql/01_least_privilege.sql`, `scripts/release_notes.py`. `TS-BL-001`–`TS-BL-003`
touch these as completion work only; `TS-BL-004`–`TS-BL-006` add new modules beside them.

**GCP resources** — Pub/Sub topics and subscriptions, Eventarc triggers, Cloud Workflows
definitions, and additional Cloud Run Jobs are new for `TS-BL-006`. A load-balancer and CDN
edge for the frontend bucket is new for `TS-BL-001`'s residual scope. Each is additive; none
alters the live request path.

**Downstream features that block on this one** — `identity-and-access` (`TS-BL-013` needs the
gateway), `access-control-and-admin` (`TS-BL-020` needs the database), `design-system`
(`TS-BL-012` needs the notification payload shape), `ai-platform-governance` (`TS-BL-029`
needs the workflow engine to have no transition capability to grant), and every Phase 2–5
feature that runs background work.

**External constraints carried, not solved here** — only Local and Dev are provisionable
(free-tier billing caps linked projects, and QA was dropped outright); the Hubble login
contract `OD-001` is unconfirmed; `registry.npmjs.org` is blocked on the corporate network.

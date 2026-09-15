## Why

TalentSphere is a greenfield, AI-assisted recruitment platform whose defining constraint is governance: every material hiring decision must rest with an accountable human, and every AI contribution must be logged, explainable, and advisory-only. Wave 1 builds the substrate that makes those guarantees possible before any hiring feature exists — identity, authorization, workflow attribution, audit, and the AI governance platform.

Sequencing this first is not preference but necessity. `AIRun` history cannot be backfilled, so the AI governance substrate must precede the first LLM call (JD generation, Wave 2). `WF-001` routes every state transition through a Workflow Service, and `NFR-009` forbids re-implementing business rules per feature — so the transition framework must exist before the domain state machines that use it. And the design-system foundation lands here deliberately, so the shared UI patterns that five later slices consume are built once, correctly, rather than reinvented ad hoc.

This change covers **Wave 1 — Foundation & Governance** only: slices **S1–S4** of the 14-slice backlog fixed in `openspec/changes/talentsphere/exploration-notes.md` (the ✔ FINAL 2026-08-20 block). Waves 2–5 are proposed separately, one at a time, when their turn comes.

## What Changes

**S1 — Identity & Access Foundation**
- Authenticate all interactive users through the Hubble REST Login API; never store Hubble credentials (`AUTH-001`, `AUTH-002`).
- Local session lifecycle: create, refresh, expire, logout, and **forced revocation** by an Application Administrator taking effect immediately on the next request (`AUTH-003`–`AUTH-007`).
- Activation gate: an authenticated Hubble user who is not activated in TalentSphere receives an access-denied screen and **no application data** (`AUTH-005`, `BR-001`).
- `users` / `roles` / `user_roles` schema, with roles **many-to-many** per user and permissions resolving to the union of grants under the most restrictive scope.
- Sign-in renders in the **Bare Shell** — the only bare-shell screen in scope for the whole product.

**S2 — Authorization & Admin Cockpit**
- Nine roles × nine action flags (View, Create, Edit, Delete, Approve, Run AI, Export, Assign, Administer) over a seeded page catalog, with matrix cells as **grant / deny / unset** (`ADM-005`).
- Deny-by-default evaluation where **explicit denial overrides role grants** (`AUTHZ-001`, `AUTHZ-005`), enforced server-side on every page and endpoint so direct API calls fail even when the UI hides the control (`AUTHZ-003`, `AUTHZ-004`).
- A **permission explanation** endpoint returning role grants, direct grants, and explicit denials behind any verdict (`AUTHZ-006`).
- Per-user permission overrides with a mandatory reason, and every permission change audited with previous and new value (`AUTHZ-007`).
- Admin Cockpit UI: user activation/deactivation, role administration, the filterable permission matrix, and the explanation panel.
- Admins are **config-only** on candidate data, with a time-boxed, audited break-glass elevation path for support cases.
- Seed scripts for roles, permissions, the page catalog, and initial admin users (`ENG-010`).
- **NEW (design-system foundation):** design tokens — color pairs, type scale, iconography, shape, elevation, spacing — plus the **Authenticated Shell** (collapsible sidebar, three shell states, page header and template patterns), and a **dense-data-table pattern** owned here, first used by the matrix UI and later consumed by the Wave 2 ranking board and Wave 4 audit logs.

**S3 — Workflow & Notification Core**
- A Workflow Service transition framework: transition validation with explained rejections, mandatory-reason enforcement on terminal and negative states, and actor attribution covering both human users and the **AI Service Account** (`WF-001`–`WF-005`, `BR-018`).
- A task model (assignment, ownership, due/aging, completion) and a notification service delivering **in-app and internal email** with retry-and-backoff.
- **Framework only — no domain state machines.** Posting, application, offer, and closure machines ship with their own features in later waves.

**S4 — AI Platform & Governance**
- An AI gateway abstraction routing every call through per-prompt-family model configuration, rate limits, and safety filters, with **graceful degradation** so an AI outage never blocks viewing records or non-AI workflow actions (`DEP-008`, `NFR-004`).
- A versioned **prompt template registry** scaffolding all ten families (`job_description_generation`, `job_posting_generation`, `resume_extraction`, `candidate_ranking`, `fitment_summary`, `gap_summary`, `interview_questions`, `interview_note_summary`, `scorecard_generation`, `resurfacing`), each with a strict output contract and — per the standing quality bar — an **explicit conciseness constraint enforced structurally in the template**, not left as style guidance.
- `ai_run_logs` recording provider, model, version, prompt template version, input/output **references rather than raw sensitive data**, token usage, safety flags, and status (`AI-002`); plus an output-feedback mechanism (`AI-011`).
- Advisory-only by construction: no code path in which AI output auto-rejects, auto-shortlists, auto-selects, or auto-closes (`AI-010`, `BR-007`).
- Prompt-injection isolation: trusted instructions separated from untrusted document content, document-derived content sanitized, tool capability restricted (`AI-012`–`AI-014`, `SEC-014`).
- The **§27.1 AI Evaluation and Security test corpus with a CI harness** — low-information and adversarial inputs, conflicting evidence, protected-attribute redaction, insufficiency outputs, evidence citation, and conciseness compliance — gating production release of any template (`AI-015`, `ENG-008`).

**Enabling scope (assumption stated explicitly)**
- A GCP delivery foundation via **Terraform + GitLab CI/CD**: repository skeleton, FastAPI/Python and React/TypeScript scaffolds, Cloud SQL PostgreSQL, secret management, versioned migrations, OpenTelemetry observability, and CI gates. Per the recorded billing constraint, **only Local and Dev are real environments**; UAT/Prod are written as Terraform code that is validated but never applied. QA is deliberately not an environment — the GCP organization is built around dev/uat/prod only, so see `design.md` D15. The backlog does not list infrastructure as a slice, but S1 cannot deploy or be tested without it, so it is carried here as Sprint 0 rather than discovered later.

**Explicit non-goals for this wave:** no job descriptions, postings, candidates, resumes, ranking, interviews, scorecards, offers, closure, resurfacing, dashboards, calendar integration, or candidate-facing communication. No prompt family is *invoked* by a user-facing feature in Wave 1 — the families are registered, contracted, and tested, and first execute in Wave 2. No candidate portal or public route of any kind. No OCR implementation, no bias-audit tooling, no vector store (Wave 2, S7).

## Capabilities

### New Capabilities

`openspec/specs/` is currently empty; every capability below is new.

- `identity/authentication`: Hubble REST login, identity mapping, session create/refresh/expire/logout, forced revocation, and authentication failure logging.
- `identity/user-activation`: local user records derived from Hubble identity, activation states, the access-denied path for unactivated users, and deactivation/leaver handling.
- `access-control/authorization`: the permission model (roles, page/action permissions, per-user overrides), deny-by-default evaluation with deny-overrides-grant, scope predicates, server-side enforcement, and permission explanation.
- `access-control/admin-cockpit`: user and role administration, the permission matrix surface with grant/deny/unset cells and filtering, break-glass elevation, and seed data for roles, permissions, pages, and initial admins.
- `platform/audit-trail`: the audit substrate — immutable, PII-free audit records for every material write, with correlation IDs and permission-controlled, audited export.
- `platform/workflow-engine`: the generic transition framework — validation, reason enforcement, actor attribution including service accounts, and transition history.
- `platform/notifications`: the task model plus in-app and internal email notification delivery with templates, retry-and-backoff, and per-user visibility.
- `platform/delivery-foundation`: infrastructure-as-code, environment topology, secret management, versioned migrations, CI/CD quality gates, feature flags, and observability signals.
- `ai-platform/ai-gateway`: the governed model gateway — per-family model configuration, rate limiting, safety filtering, prompt-injection isolation, retry semantics, and graceful degradation.
- `ai-platform/prompt-registry`: versioned prompt template records for the ten families, their output contracts, conciseness bounds, and active-version configuration.
- `ai-platform/ai-run-logging`: `ai_run_logs` capture with reference-only inputs, safety flags, status and failure detail, human-override recording primitives, and AI output feedback.
- `ai-platform/ai-evaluation`: the AI evaluation and security test corpus, its required coverage categories, and the CI harness gating template promotion.
- `design-system/foundations`: design tokens for color (light/dark pairs), typography, iconography, shape, elevation, and spacing, plus accessibility baselines.
- `design-system/app-shell`: the three shell states, sidebar behavior, page header and page templates, the dense-data-table pattern, and responsive/grid rules.

### Modified Capabilities

None — this is the first implementation change in a greenfield repository.

## Impact

- **New repository structure:** FastAPI/Python backend, React/TypeScript frontend, Terraform infrastructure, GitLab CI/CD pipelines. Fills the `context:` block of `openspec/config.yaml`, which the resolved stack decision now unblocks.
- **Database:** first migrations creating `users`, `roles`, `user_roles`, `page_permissions`, `role_permissions`, `user_permission_overrides`, `audit_logs`, `ai_run_logs`, prompt template registry tables, and the task/notification tables.
- **External dependencies:** Hubble REST Login API (contract unconfirmed — `OD-001`), an AI model provider and gateway (`OD-003`), an SMTP/email service for internal notifications, GCP (Cloud Run or equivalent, Cloud SQL, Secret Manager, Cloud Tasks), and GitLab.
- **Downstream waves:** every later slice depends on this wave's permission evaluator, workflow service, audit substrate, AI gateway, and shared UI patterns. Two shared patterns are *owned* here and consumed later: the dense data table (S8, S13) and the Authenticated Shell (all subsequent screens).
- **Decisions carried in as assumptions**, recorded in `design.md` and flagged for owner sign-off: the **seeded permission-matrix values** (a product decision, not a default — a fully closed matrix would silently break cross-pool resurfacing later), the **break-glass notification target**, the **page catalog seeded for the whole product** rather than per-wave, and the **per-family conciseness bounds**, whose principle is settled but whose exact numbers are deliberately left to this wave's prompt-design work.
- **Unaffected by design:** the `talentsphere` change remains exploration-only and permanently unproposed; this change does not add artifacts to it.

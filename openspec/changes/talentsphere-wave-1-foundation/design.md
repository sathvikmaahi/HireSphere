## Context

See `proposal.md` — Why, for motivation, and the delta specs under `specs/` for the behavior contracts. This document covers only the technical decisions Wave 1 must settle before coding.

Constraints that shape the approach:

- **Greenfield repository.** No code, no schema, no CI. Every convention this wave establishes is inherited by four later waves, so the cost of getting a pattern wrong here is multiplied.
- **Settled stack** (exploration notes, D09): FastAPI/Python, React + TypeScript, PostgreSQL on Cloud SQL, Cloud Tasks or Celery for background work, OpenTelemetry. Vertex AI Vector Search is the vector backend, but that is Wave 2 scope.
- **GCP with Terraform and GitLab CI/CD**, modeled on the prior landing-zone patterns. The billing account is capped at 5 linked projects with 3 already in use, so **only Local and Dev can be real environments**; UAT and Prod exist as validated-but-never-applied Terraform. This is a hard external constraint, not a preference.
- **Two external contracts are unconfirmed**: the Hubble login contract (`OD-001`) and the AI provider and gateway contract (`OD-003`). Wave 1 cannot wait for either, so both must sit behind adapters.
- **Build fresh.** Per the manager's 2026-08-20 direction, this is not reconciled against the existing Azure TalentSphere deployment. There is no guarantee of parity with that system, and that is accepted as intentional.
- **A design system was supplied**, product-agnostic, in `reference/design-spec.md` and `reference/layout-spec.md`. It is authoritative for visual language but says nothing about TalentSphere's screens.

## Goals / Non-Goals

**Goals:**

- One permission evaluator that every page, endpoint, and background actor consults, with no second implementation anywhere.
- Governance substrate that is *load-bearing rather than advisory*: it must be impossible to write an AI feature in Wave 2 that skips run logging or bypasses the gateway.
- Adapter boundaries around both unconfirmed external contracts, so a contract surprise costs an adapter rewrite rather than a redesign.
- Shared UI patterns built once, in a form the five later consuming slices can adopt without modification.
- A Dev environment that can actually be deployed to and tested against, from infrastructure code.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- No domain state machines. The workflow engine ships with zero registered machines; posting, application, offer, and closure machines belong to their owning features.
- No caching layer for permission verdicts in this wave. Correctness of immediate revocation outweighs latency at this user population; see the decision below.
- No multi-tenancy, no practice-based row-level security. `Practice` is carried as a data attribute for later filtering and reporting, never as a permission boundary.
- No prompt engineering of substance. Wave 1 defines each family's contract, bounds, and tests; the actual prompt text is written by the feature that first uses it.

## Decisions

### D1 — Permission evaluation is centralized, synchronous, and uncached

A single evaluator resolves `(actor, page, action, resource-scope) → verdict + explanation`. Every endpoint declares its required permission declaratively; an endpoint that declares nothing denies everything. The explanation is produced by the same evaluation pass that produces the verdict, not reconstructed afterwards by a second code path.

*Why:* `AUTHZ-006` requires an explanation that is trustworthy, and an explanation derived separately from the decision will eventually disagree with it — at which point the explanation is worse than none. Deriving both from one pass makes divergence structurally impossible.

*Uncached, deliberately.* `AUTH-007` and the immediate-effect requirement mean a revoked permission must bite on the next request. At this scale — hundreds of users, not hundreds of thousands — a per-request evaluation against indexed tables is affordable. *Alternative considered:* caching verdicts in the session with an invalidation channel. Rejected for now: the invalidation channel is exactly where this class of bug hides, and we would be buying performance we do not yet need. If profiling later demands it, the cache goes behind the evaluator interface without touching callers.

*Alternative considered:* framework-level decorators only. Rejected — decorators cover endpoints but not background actors, and the AI Service Account must be evaluated by the same rules.

### D2 — Scope predicates are declared on grants, not hard-coded per query

A grant carries an optional scope predicate (`all`, `assigned-postings`, `own-assignments`). The evaluator returns both a verdict and, for list operations, a scope filter the query layer applies. Wave 1 defines the predicate vocabulary and enforces it; the resources it filters on mostly arrive in Wave 2.

*Why:* C-03 resolved read scope to be *configurable* and write scope to be assignment-bound. Hard-coding predicates into each query would make that configuration a lie. Returning a filter rather than post-filtering results also avoids the classic pattern of fetching rows the user may not see and dropping them afterwards, which leaks counts and pagination.

*Trade-off:* the query layer must consistently apply the returned filter, which is a discipline no type system enforces. Mitigated by requiring every list endpoint to route through a helper that takes the filter as a required argument, so omission is a compile-time-visible mistake rather than a silent one.

### D3 — The page catalog is seeded for the entire product, not per wave

All screens from the reference specification's screen inventory are seeded as page/action permission rows in Wave 1, including screens whose routes do not exist yet.

*Why:* it makes the matrix complete and reviewable as one artifact, lets Admins pre-configure access ahead of a wave landing, and avoids five subsequent re-seed migrations each of which would need its own audit story. Unrouted pages are harmless: deny-by-default means an unreachable page grants nothing.

*Alternative considered:* seed per wave. Rejected — it turns the matrix into a moving target and makes the seeded-values decision (D4) recur four more times.

### D4 — Seeded matrix values are a recorded product decision, with reads open within role

Interaction B in the exploration notes is explicit that the seeded matrix determines whether a headline feature works on day one: with `AUTHZ-008` least-privilege applied literally to reads, a Recruiter cannot see candidates outside their assigned postings, and cross-pool resurfacing silently returns nothing.

The seed therefore is:

- **Writes**: assignment-scoped for Recruiters on posting-bound resources; least-privilege everywhere else.
- **Reads**: open within role for Recruiter, Practice Manager, and Recruitment Manager across the candidate and posting pools — a deliberate widening of the least-privilege default, recorded as such.
- **Interviewer and Hiring Panel Member**: `own-assignments` scope only, with no export and no approve.
- **Auditor**: read-only on audit and AI run logs; structurally no Create, Edit, Delete, Approve, or Assign anywhere.
- **Application Administrator / System Administrator**: configuration only, with no View on candidate personal data.
- **AI Service Account**: only the actions its background jobs require; Approve is unassignable.

*This is flagged for owner sign-off rather than treated as settled* — it is a policy choice about who sees whose candidates, and it belongs to the business. It is seeded this way because a closed default breaks a headline feature quietly, whereas an open-within-role default fails loudly and visibly if it is wrong. Recorded in Open Questions.

### D5 — Break-glass is a time-boxed override, not a separate access mode

Administrator access to candidate personal data is granted as an ordinary user-level permission override with an expiry, a mandatory typed reason, an audit record, and a notification. It expires on its own.

*Why:* reusing the override mechanism means break-glass inherits the audit trail, the explanation endpoint, and the evaluator for free. A parallel "elevated mode" would be a second authorization path — precisely the thing D1 exists to prevent.

*Why time-boxed rather than per-request:* per-request approval needs a second human available on demand, which real support work does not have. Expiry bounds the exposure without adding a synchronous dependency on another person.

### D6 — Audit records store references and diffs, never personal data

Audit rows carry actor, action, target type and ID, previous and new values, reason, correlation ID, and timestamp. Where a changed field holds personal data, the value is redacted or referenced rather than copied in.

*Why:* D16 made Admins config-only, and the Auditor role reads the audit trail. If audit rows contained resume text or candidate names, both boundaries would be decorative — the audit table would become the PII back door. This is a schema constraint, enforced by a redaction policy applied at write time, not a display filter.

*Trade-off:* an investigator sometimes wants the old value verbatim. They can get it by resolving the reference, which requires permission on the referenced record — which is the point.

### D7 — Insert-only audit and run-log tables, with no application write path

Audit and AI run tables are append-only from the application's perspective: no update or delete surface exists, and the application's database role is granted `INSERT` and `SELECT` only on them.

*Why:* `SEC-011` requires immutability to standard users. Enforcing it at the grant level rather than in application code means a future bug cannot violate it.

*Alternative considered:* database triggers rejecting updates. Equivalent in effect, more machinery; the grant is simpler and self-documenting.

### D8 — Hubble sits behind an authentication adapter with a contract test

All Hubble interaction is confined to one adapter exposing `authenticate(credentials) → identity claims`. Everything downstream consumes a normalized internal identity. A recorded contract test defines what we currently believe the response shape to be, and the adapter maps unknown or missing claims explicitly rather than silently.

*Why:* `OD-001` is unresolved and owned by another team. The wave cannot block on it. The identity mapping keys on the Hubble user identifier rather than email precisely because email is the claim most likely to change or be absent.

*Consequence to accept:* until the real contract lands, Dev runs against a mock. That mock is a first-class artifact, not a test fixture, because Wave 2 and 3 development will keep depending on it.

### D9 — The AI gateway is the only egress, and logging precedes invocation

The gateway resolves template and model configuration, writes the run record, invokes the provider, validates output against the family contract, then persists the output as advisory insight. Provider credentials are reachable only from the gateway.

**Logging happens before invocation, not after.** If the run record cannot be written, the provider is never called.

*Why:* `AI-002` plus the exploration notes' insistence that `AIRun` history cannot be backfilled. Logging after the fact loses exactly the runs most worth having — the ones that crashed. Ordering it first converts "we log AI calls" from a convention into an invariant.

*Why output validation lives in the gateway:* if each feature validated its own output, contract enforcement would be as inconsistent as the features. Central validation also means a contract violation is recorded as a failed run rather than becoming a malformed insight row.

*Consequence for sequencing:* the `ai_run_logs` table and the write-before-invoke check must land **in the same sprint as the gateway and ahead of it**, not in the sprint after. An invariant that holds from the second sprint onward is not an invariant. `tasks.md` Sprint 13 therefore opens with the log substrate and closes by verifying no invocation path bypasses it.

### D10 — Advisory-only is enforced structurally, not by review

The gateway has no capability to trigger a state transition, and the workflow service accepts transitions only from an authenticated actor holding the transition's declared permission — with `Approve` unassignable to the AI Service Account.

*Why:* `AI-010` and `BR-007` are the product's central promise. A promise kept by code review is a promise that breaks the first time someone is in a hurry. Removing the capability altogether makes the guarantee cheap to verify: there is no code path to audit.

### D11 — Conciseness is a contract clause, not prompt wording

Each family's output contract declares machine-checkable bounds per free-text field — maximum sentences, list items, or characters — validated by the same code that validates shape. Violations fail the run.

*Why:* S.5 identified that the reference specification fixes output *shape* but never *length*, so nothing stops three paragraphs where two sentences would serve. Asking a model politely to be brief is not enforcement. Making length part of the contract makes verbosity a validation failure that CI can see.

*Open:* the actual numbers per family. Deliberately not invented here — see Open Questions.

### D12 — Prompt families are registered and tested but not authored in this wave

Each of the ten families gets a registry record, an output contract with bounds, an evaluation corpus, and a smoke-testable path. The prompt text itself is a stub sufficient to exercise the contract.

*Why:* writing ten real prompts without the features that use them, against schemas that Wave 2 will refine, guarantees rework. But the *contract* and the *tests* are exactly what must exist before the first real call, because they are what the substrate enforces. This splits the work along the seam where value actually sits.

### D13 — Tokens are the single source of truth for visual style, generated once

Color, type, spacing, radius, elevation, and icon rules from `reference/design-spec.md` are expressed as a token set, consumed by components; no component defines a raw color or spacing value. A validation step fails the build on a raw hex value outside the token definitions and on any token missing its light or dark counterpart.

*Why:* the design spec's own rule against one-off colors is unenforceable by convention across five waves and multiple developers. A build check makes it real. The dark-pair completeness check exists because the spec is explicit that a color with only one value is incomplete.

### D14 — The dense data table is a fourth page template, not a restyled card grid

Built from existing tokens, with compact type steps, its own overflow container, and permission-gated export.

*Why:* S.3 found the supplied guide is card-oriented and offers nothing for the product's genuinely tabular screens — the permission matrix here, the ranking board in Wave 2, audit and AI-run logs in Wave 4. Wave 1 owns it because the matrix is its first real consumer, and building it against a real screen rather than speculatively is what keeps it honest.

### D15 — Only Local and Dev are provisioned; UAT and Prod are validated code

The Terraform for all four environments is written and kept in CI validation. Only Local and Dev are applied, matching the existing prototype pattern rather than the reference specification's project-per-app-per-environment model.

*Why:* the billing account allows 5 linked projects and 3 are used. This is arithmetic, not architecture. Writing the later environments as unapplied code keeps them reviewable and ready, and means raising the quota later is an `apply`, not a project.

*Risk accepted:* unapplied Terraform drifts from reality in ways validation does not catch. Mitigated by keeping the environments structurally identical, parameterized by variables only.

**Amended 2026-08-21 — QA dropped.** This decision originally named five environments including QA. Inspecting the live GCP organization showed it is built around dev, uat, and prod only: `fld-nonprod` contains `fld-dev` and `fld-uat` with no `fld-qa`, and each environment folder has its own `prj-network-<env>` shared-VPC host, so QA had neither a parent folder nor a network host. A QA stack would have been code describing infrastructure with no place to land — validating cleanly while being unapplyable in principle rather than merely deferred, which is a worse failure mode than not having it. The owner confirmed dropping QA outright rather than creating `fld-qa` to satisfy the artifact. Environments are now **Local, Dev, UAT, Prod**.

*Consequence:* if QA is wanted later it is a folder, a network project, and one more variables file — not a redesign, because the environments are structurally identical by construction.

### D16 — Notifications are internal-only, enforced at the sender

The notification service refuses any recipient that does not resolve to an internal user record.

*Why:* C-04 defers candidate-facing communication pending Legal (`OD-005`), and "we simply won't send to candidates" is not a control. A sender-side guard means the deferral cannot be violated by a misconfigured template.

### D17 — One landing route, thin at first, with dashboards deferred

Sign-in resolves to a single landing route that any activated user can reach without further page grants. It ships as an empty shell in Sprint 5 and is populated in Sprint 12 with the signed-in user's own tasks and notifications. The reference specification's Home Dashboard, with role-specific summaries and pending approvals, remains Wave 4 scope alongside the other dashboards.

*Why:* a user has to land somewhere the moment sign-in works, and the alternative — routing straight to a permissioned page — breaks for exactly the user whose access is still being configured, which is the most common state in this wave. Requiring no page grant means the landing route can never itself produce an access denial, so a misconfigured user sees an empty page and a route to request access rather than a dead end.

*Why not build the full dashboard here:* its content is postings, candidates, and approvals — none of which exist until Waves 2 and 3. Building it now would mean building placeholders and then rebuilding it.

### D18 — The database has no password; identities authenticate with IAM

*Added 2026-08-21, during Sprint 0.*

The application and the migration runner connect to Cloud SQL as
`CLOUD_IAM_SERVICE_ACCOUNT` users, authenticating with a short-lived OAuth token
rather than a stored password. `cloudsql.iam_authentication` is enabled on the
instance. Two identities exist rather than one: the runtime holds DML only, and
a separate migration identity owns the schema and holds DDL.

*Why:* Sprint 0 first shipped a `BUILT_IN` role whose password was generated by
Terraform and stored in Secret Manager. That satisfies the `Configuration
without hardcoded secrets` requirement — the secret was in an approved manager
and injected at runtime — but it creates a credential that exists, sits in
Terraform state, and has to be rotated. IAM authentication removes the
credential rather than protecting it, which is the stronger version of the same
requirement. It also matches how every other application project in this
organization already connects.

*Why two identities:* D7 makes the audit and AI run tables append-only by
granting the application role `INSERT` and `SELECT` only. That guarantee is
decorative if the same role can `ALTER TABLE`, so the role that serves requests
must not hold DDL. Splitting migration from runtime is what makes the grant
enforceable rather than advisory.

*Why now rather than as a follow-up:* exactly one module reads the connection
configuration today. From Sprint 1 the audit writer depends on it as well, so
the change gets more expensive with every sprint that passes, and it is
cheapest before the substrate that relies on it exists.

*Consequence for Local:* a developer's Postgres in `docker-compose` has no IAM,
so the connection layer supports both modes — password locally, IAM in Dev and
above — selected by configuration rather than by branching at call sites.

*What this does not change:* secret management remains wired and required. The
Hubble configuration, the AI provider credentials, and SMTP settings are all
still secrets injected at runtime; the database simply stops being one of them.

## Risks / Trade-offs

- **Hubble contract turns out materially different from what the adapter assumes** (`OD-001`) → All Hubble specifics confined to one adapter plus its contract test; the internal identity model is normalized, so a surprise costs an adapter rewrite. Filing the contract request early is free and should happen in Sprint 1.
- **Uncached permission evaluation becomes a latency problem as pages get denser** → Evaluation sits behind an interface with the verdict and explanation produced together; a cache can be introduced later without changing callers. Watch p95 on the matrix screen, which is the worst case by design.
- **The seeded matrix (D4) is wrong for the business** → It is recorded as a decision with a named rationale rather than buried in a seed script, so it can be reviewed and changed as configuration. The failure mode chosen is the visible one.
- **Wave 1 has no end-user-visible feature beyond sign-in and administration**, which makes progress hard to demonstrate for eleven-plus sprints → Sequence the milestones so S1+S2 produce a genuinely demoable outcome early: a user logs in, is blocked when unactivated, and an Admin configures their access. The first AI-visible demo is deliberately Wave 2's.
- **The AI substrate is built before any consumer exists**, risking abstractions that fit no real caller → Mitigated by exercising every family through the evaluation harness and a diagnostic path in this wave, so the substrate is executed rather than merely compiled. Residual risk accepted: Wave 2 will refine the contracts, and that is expected rather than a defect.
- **Ten evaluation corpora plus a security suite is substantial ongoing cost** → Accepted deliberately (C-09). It is the strongest available control against injection and protected-attribute leakage, and the corpus grows from real failures found in use rather than being written exhaustively up front.
- **Design-system foundation adds real schedule cost to Wave 1** → Accepted: roughly one wave-level increment in exchange for not reinventing shell, tokens, and table patterns inside S2, S8, S9, S10, and S13 separately.
- **Dataless-file recovery in the planning repo** → `openspec/config.yaml` and the `talentsphere` change marker were unreadable iCloud placeholders and were regenerated byte-identically from the CLI defaults; originals are preserved alongside as `.icloud-dataless.bak`. No planning content was lost, but the repository would benefit from being committed to git, which it currently is not.

## Migration Plan

There is no existing system to migrate from; this is initial deployment. Sequence:

1. **Repository and pipeline first.** Repo skeleton, CI gates, and Terraform for Local and Dev, before application code — so every subsequent commit is built and tested by the same pipeline that will deploy it.
2. **Schema in versioned migrations from the first table.** No hand-applied DDL in any environment, including Dev.
3. **Seed scripts are idempotent** and run as a deployment step, so a fresh environment reaches a usable state with the nine roles, the page catalog, and an initial administrator.
4. **Feature-flag the AI substrate.** The gateway and registry ship disabled by default; enabling them in Dev is a flag change, which also proves the flag mechanism before Wave 2 depends on it.
5. **Rollback** is redeploy-previous-artifact plus migration-down for any migration in the release. Because Wave 1 creates tables rather than transforming data, rollback is low-risk throughout.

## Open Questions

These are deferrable: none changes the specs, the approach, or the task breakdown, but each needs an owner before the wave closes.

- **Seeded matrix values (D4)** — needs business sign-off on read scope across the candidate pool. Defaults are seeded and auditable; changing them is configuration, not code.
- **Break-glass notification target** — currently all other Application Administrators plus the Auditor role. Security may want a different recipient.
- **Per-family conciseness bounds (D11)** — the principle is settled and enforced; the numbers belong to whoever writes each family's real prompt. Stubs carry provisional bounds so the mechanism is testable now.
- **Hubble claim set** (`OD-001`) — which claims arrive at login determines how much identity data is maintained locally and whether a user-to-Workspace-email mapping table is needed later.
- **Hubble ID validation endpoint existence** (`OD-002`) — irrelevant until Wave 3, noted so it is not rediscovered then.
- **AI provider and gateway contract** (`OD-003`) — the gateway abstraction is provider-shaped, not provider-specific; the first concrete provider can be chosen during this wave without redesign.
- **Session expiration default** — configurable per the specs; the initial value should follow the enterprise standard once confirmed.
- **Whether `innovation practice relevance` applies here or should be renamed** — a Wave 3 scorecard concern, recorded because the term appears in seeded reference data.

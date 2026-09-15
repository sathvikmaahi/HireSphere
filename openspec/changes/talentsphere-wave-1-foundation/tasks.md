# Wave 1 — Foundation & Governance · Sprint Backlog

Covers slices **S1** Identity & Access Foundation, **S2** Authorization & Admin Cockpit (including the design-system foundation), **S3** Workflow & Notification Core, and **S4** AI Platform & Governance.

**Sprint length: 1 week.** Groups below are sprints, in dependency order. Sprint 0 is the enabling infrastructure scope declared in `proposal.md`; Sprints 1–15 are slice work.

This is 15 slice sprints plus Sprint 0, marginally above the 11–14 illustrative range in the exploration notes — because the delivery foundation and the ten-family evaluation corpus are itemized here rather than assumed inside other slices. The illustrative range was explicitly a placeholder pending a real task breakdown; this is that breakdown.

Slice mapping: Sprints 1–3 → S1 · Sprints 4–10 → S2 · Sprints 11–12 → S3 · Sprints 13–15 → S4.

**Not in this wave, deliberately:** §27.1 Regression Tests are scoped to "critical workflows from posting creation to closure" — no such workflow exists until Wave 2, so regression coverage starts there rather than being a gap here.

## 1. Sprint 0 — Delivery Foundation

Goal: every later commit is built, tested, and deployable by the same pipeline. Covers `platform/delivery-foundation`.

- [x] 1.1 Create the repository skeleton: FastAPI/Python backend, React + TypeScript frontend, Terraform, and pipeline definitions, with linting and formatting configured per language
- [x] 1.2 Fill the `context:` block in `openspec/config.yaml` with the settled stack and conventions, and commit the repository to git (it is currently untracked)
- [x] 1.3 Write Terraform for all four environments — Local, Dev, UAT, Prod — parameterized by variables only, and apply Local and Dev only
- [x] 1.4 Verify UAT and Prod definitions pass `validate` and `plan` in CI without provisioning any billable resource
- [x] 1.5 Provision Cloud SQL PostgreSQL for Dev, with the application database role granted least privilege
- [x] 1.6 Wire secret management: runtime injection from the secret manager, with a CI check that fails on any credential committed to source or configuration
- [x] 1.7 Set up the migration tool with a versioned baseline migration and a pipeline step applying migrations to Dev before any deployment
- [x] 1.8 Build the GitLab CI pipeline gates: lint, type check, unit tests, dependency check, security scan, with merge blocked on failure and artifacts tagged to their commit
- [x] 1.9 Add OpenTelemetry structured logging with a request correlation ID propagated through API, background jobs, and database calls; export metrics and traces
- [x] 1.10 Implement the feature-flag mechanism and prove it by flagging one placeholder capability off and on in Dev
- [x] 1.11 Implement the global error handler producing a consistent error code, message, trace ID, and field validation detail, asserting no stack trace, secret, or prompt content ever reaches a response
- [x] 1.12 Add the runtime configuration store for the values the specs require to be changeable without deployment, with an audited change path
- [x] 1.13 Add the release-notes step to the pipeline, publishing changes, included migrations, and known issues per release

## 2. Sprint 1 — Schema & Audit Substrate

Goal: the audit trail exists before the first record it must witness. Covers `platform/audit-trail`.

- [ ] 2.1 Migrate `users`, `roles`, and `user_roles`, keyed on the Hubble user identifier and supporting many-to-many role assignment
- [ ] 2.2 Migrate `audit_logs` with actor user, actor service account, action, target type and ID, previous and new value, reason, correlation ID, source address, and immutable timestamp
- [ ] 2.3 Grant the application database role `INSERT` and `SELECT` only on `audit_logs`, and prove `UPDATE` and `DELETE` fail at the database level
- [ ] 2.4 Implement the audit writer so audit records are written in the same transaction as the change, and a failed audit write fails the operation
- [ ] 2.5 Implement the redaction policy applied at write time so no candidate personal data is stored in audit values, with references or redaction markers used instead
- [ ] 2.6 Implement audit search filterable by actor, target type, target ID, action, module, and time range, with indexes covering those paths
- [ ] 2.7 Implement audited, permission-gated export carrying a data classification label and generating its own audit event
- [ ] 2.8 Test that a material write with no audit record cannot succeed, and that a reason-required action with no reason writes nothing
- [ ] 2.9 Add the optional `practice` attribute to the user record as a filtering and reporting field only, and assert it participates in no authorization decision
- [ ] 2.10 Migrate the session store with user reference, issued and expiry timestamps, status, and revocation fields, indexed for immediate per-request validity checks
- [ ] 2.11 Implement the configured audit-log retention policy with disposal itself recorded

## 3. Sprint 2 — Hubble Authentication Adapter

Goal: real login against an isolated, replaceable contract. Covers `identity/authentication`.

- [ ] 3.1 File the Hubble contract request with the Architecture and Hubble team for `OD-001`, and record the response in this change
- [ ] 3.2 Build the Hubble adapter exposing `authenticate(credentials) → normalized identity claims`, with all Hubble specifics confined to it
- [ ] 3.3 Write the recorded contract test capturing the assumed response shape, mapping unknown or missing claims explicitly rather than silently
- [ ] 3.4 Build the Hubble mock service as a first-class Dev artifact that Waves 2 and 3 can keep developing against
- [ ] 3.5 Implement the login endpoint: successful authentication creates or refreshes a session; rejection distinguishes authentication from authorization without revealing identity existence
- [ ] 3.6 Implement provider-failure handling with timeout and retry, asserting no local or cached credential fallback exists
- [ ] 3.7 Implement identity mapping on the Hubble identifier, refreshing display name and email on the local record, and updating rather than duplicating a user whose email changed upstream
- [ ] 3.8 Implement authentication rate limiting and authentication success and failure logging with no credential values recorded
- [ ] 3.9 Test that no Hubble password or reusable credential appears in database, object storage, logs, or session records after a successful login
- [ ] 3.10 Write the integration test suite against the Hubble mock covering success, rejection, timeout, and malformed-claim responses, and wire it into CI

## 4. Sprint 3 — Sessions, Revocation & Activation

Goal: admission to the system is decided locally, and can be withdrawn instantly. Covers `identity/authentication` and `identity/user-activation`.

- [ ] 4.1 Implement the session record carrying user ID, Hubble identifier, display name, email, roles, resolved permissions, login timestamp, expiry, and status
- [ ] 4.2 Implement logout, expiry, and refresh, with session duration read from runtime configuration
- [ ] 4.3 Implement forced session revocation by an Application Administrator, taking effect on the next request with no cached-verdict window, and audited
- [ ] 4.4 Implement the session introspection endpoint returning only the caller's own identity, roles, and permissions
- [ ] 4.5 Implement local user states — active, inactive, pending, deactivated — with every transition audited with previous and new value and a mandatory reason on deactivation
- [ ] 4.6 Implement pending-record creation on first authentication of an unknown identity, carrying no permissions and visible to administrators
- [ ] 4.7 Implement the activation gate so an authenticated but unactivated user receives access-denied and no endpoint returns application data
- [ ] 4.8 Implement deactivation so it revokes active sessions immediately while leaving owned records intact, reassignable, and with historical attribution unchanged
- [ ] 4.9 Test the access-denied payload discloses only the denial, a correlation ID, and contact guidance
- [ ] 4.10 Test that a user holding two roles receives the union of grants under the most restrictive applicable scope

## 5. Sprint 4 — Design Tokens

Goal: the visual language exists as enforced tokens before any screen is built. Covers `design-system/foundations`.

- [ ] 5.1 Define the color token set from `reference/design-spec.md` as explicit light and dark pairs, including brand accent, surfaces, text, semantic, and status-surface triplets
- [ ] 5.2 Add build validation failing on any token missing its light or dark counterpart, and on any raw color value outside the token definitions
- [ ] 5.3 Define the type scale, weight scale, and the separate long-form prose treatment, including inline code and code block styling
- [ ] 5.4 Define spacing, radius, and elevation tokens, with dark-mode shadow opacity higher than light-mode
- [ ] 5.5 Set up the two typefaces with fallbacks and antialiasing, and constrain the heaviest weight to the brand lockup only
- [ ] 5.6 Build the icon set integration: outline style, color inherited from surrounding text, defined size steps, plus the separate full-color external brand-mark category
- [ ] 5.7 Build the base components from tokens: button variants and sizes, badge, disabled state as uniform opacity, and the focus ring
- [ ] 5.8 Implement theme switching with animated transition, persisted across sessions
- [ ] 5.9 Add brand assets with theme-appropriate variant swapping, fixed-height constraint, and clear-space rule
- [ ] 5.10 Verify WCAG 2.1 AA contrast for every token pair in both themes, and keyboard operability with always-visible focus on every base component

## 6. Sprint 5 — App Shell & Sign-in Screen

Goal: the first real screen, on the shell every later screen inherits. Covers `design-system/app-shell`.

- [ ] 6.1 Implement the three shell states, with the shell occupying full viewport height and only the main content column scrolling
- [ ] 6.2 Implement route-level shell selection with bare shell restricted to sign-in, and every other unauthenticated request redirected there
- [ ] 6.3 Verify no public, embeddable, or shared read-only route exists in the route table
- [ ] 6.4 Build the sidebar: two widths, smooth transition, expanded default, choice persisted across sessions, on its own fixed surface excluded from the theme toggle
- [ ] 6.5 Build navigation items with icon and label, distinct active and hover treatments, capped count badges, collapsed-state accessible names, and items hidden where the user lacks View
- [ ] 6.6 Build the sidebar utility footer: theme toggle, user menu with identity and sign-out, expanded-only brand logo
- [ ] 6.7 Implement the main content area with standard padding, bottom clearance for the floating panel region, and a per-route opt-out
- [ ] 6.8 Implement the page header pattern with title, one-line description, single primary action, and no wrapping on narrow viewports
- [ ] 6.9 Implement the browsable card grid, detail view with collapsing metadata rail, and standalone templates, with loading and empty states inside the content container
- [ ] 6.10 Implement the floating action panel region, rendering nothing when no task is in progress, with no consumer in this wave
- [ ] 6.11 Build the sign-in screen on the bare shell with its top header, error handling, and the access-denied state
- [ ] 6.12 Build the AI-generated content label and the human-review disclaimer components from the badge and status-surface patterns, with wording that makes no bias-free claim
- [ ] 6.13 Implement the field-level plus summary-level validation error presentation, and the workflow state, owner, next action, blocker, and last-updated display pattern
- [ ] 6.14 Implement the single authenticated landing route, reachable by any activated user without further page grants, rendering an empty state where the user has nothing outstanding
- [ ] 6.15 Implement deep-link preservation so a visitor sent to sign-in returns to their originally requested route rather than the landing route
- [ ] 6.16 Author the UI test suite for shell states, role-based navigation visibility, sign-in, access-denied, and theme persistence, and wire the UI suite into CI

## 7. Sprint 6 — Permission Model & Evaluator

Goal: one evaluator, one verdict, one explanation. Covers `access-control/authorization`.

- [ ] 7.1 Migrate `page_permissions`, `role_permissions`, and `user_permission_overrides` with grant, deny, and unset states and mandatory reason on overrides
- [ ] 7.2 Implement the evaluator resolving actor, page, action, and resource scope to a verdict, deny-by-default with no applicable grant
- [ ] 7.3 Implement unset semantics: contributes neither grant nor denial, and a grant in any one role wins over unset in another
- [ ] 7.4 Implement denial precedence so an explicit denial at role or user level overrides every grant
- [ ] 7.5 Produce the explanation from the same evaluation pass as the verdict, naming the deciding grant or denial and the roles consulted
- [ ] 7.6 Implement the nine action flags per page, with Run AI and Approve independently controllable
- [ ] 7.7 Implement the scope-predicate vocabulary — all, assigned-postings, own-assignments — returning a query filter for list operations rather than post-filtering results
- [ ] 7.8 Implement the mandatory list-query helper that takes the scope filter as a required argument so omission is visible at the call site
- [ ] 7.9 Implement immediate effect of permission and role changes on subsequent requests, with no stale cached verdict
- [ ] 7.10 Unit-test the evaluator against the full grant, deny, unset, and multi-role matrix, including the deny-beats-grant and unset-does-not-grant cases

## 8. Sprint 7 — Enforcement & Explanation Surface

Goal: the evaluator is unavoidable, including for direct API calls and background actors. Covers `access-control/authorization`.

- [ ] 8.1 Implement declarative per-endpoint permission requirements, with an endpoint declaring none denying all requests
- [ ] 8.2 Add a startup or CI check enumerating endpoints and failing on any that declares no permission requirement
- [ ] 8.3 Implement page-level authorization on every route, server-side, independent of the client
- [ ] 8.4 Implement the permission explanation endpoint for administrators, covering both grants and denials
- [ ] 8.5 Implement service-account authorization so background AI tasks are evaluated by the same evaluator, with Approve unassignable to the AI Service Account
- [ ] 8.6 Implement write-scope enforcement so a posting-scoped grant denies writes against unassigned postings even when the page action is granted
- [ ] 8.7 Implement configuration-driven read scope so changing a role's read scope changes verdicts with no deployment
- [ ] 8.8 Implement permission-change auditing capturing actor, subject, previous and new value, reason, and affected page and action
- [ ] 8.9 Test that every hidden UI control's underlying endpoint denies a direct call, with no partial write
- [ ] 8.10 Test that a permission revoked mid-session denies the affected action on the very next request

## 9. Sprint 8 — Seed Data & the Nine Roles

Goal: a freshly provisioned environment is usable and its access posture is a recorded decision. Covers `access-control/admin-cockpit` and `access-control/authorization`.

- [ ] 9.1 Seed the nine roles, protecting them from deletion while leaving their permission assignments editable
- [ ] 9.2 Seed the complete product-wide page and action catalog, including screens whose routes arrive in later waves
- [ ] 9.3 Seed the matrix values per the design decision: assignment-scoped writes, reads open within role for Recruiter, Practice Manager, and Recruitment Manager, own-assignments for Interviewer and Hiring Panel Member, read-only Auditor, configuration-only Administrators, and a minimal AI Service Account
- [ ] 9.4 Record the seeded posture as an auditable configuration state rather than a silent script effect, and route it to the business owner for sign-off
- [ ] 9.5 Seed the Interviewer role's grants with the assignment-scoped predicate so the role can never resolve to all interviews
- [ ] 9.6 Verify the Auditor role holds no Create, Edit, Delete, Approve, or Assign anywhere, and that Administrator roles hold no View on candidate personal data
- [ ] 9.7 Seed initial administrative users and make every seed script idempotent, verified by running it twice against the same database
- [ ] 9.8 Test a freshly seeded environment end to end: an administrator signs in, activates a user, and that user's access matches the seeded posture

## 10. Sprint 9 — Dense Data Table & Permission Matrix

Goal: the shared tabular pattern, built against its first real consumer. Covers `design-system/app-shell` and `access-control/admin-cockpit`.

- [ ] 10.1 Build the dense data table as a fourth page template from existing tokens only, using the compact type steps, introducing no new tokens
- [ ] 10.2 Implement search, filtering, sorting, and pagination in the table pattern
- [ ] 10.3 Implement horizontal overflow inside the table's own container, verifying the page body never scrolls horizontally
- [ ] 10.4 Implement export gated on Export permission, with no control rendered without it and a direct request denied
- [ ] 10.5 Build the permission matrix screen on the dense table: users or roles as rows, modules, pages, or actions as columns
- [ ] 10.6 Implement matrix cell editing across grant, deny, and unset, presenting an overriding denial as visually distinct from merely ungranted
- [ ] 10.7 Implement matrix filtering by user, role, module, practice, and active status, and row-subject switching between roles and users
- [ ] 10.8 Implement atomic bulk matrix update where every changed cell produces its own audit record
- [ ] 10.9 Build the explanation panel showing role grants, direct grants, and explicit denials for a selected subject, page, and action
- [ ] 10.10 Measure p95 render and evaluation latency for the fully populated matrix, the worst case for uncached evaluation, and record the result against the design's caching decision
- [ ] 10.11 Author UI tests for the matrix: cell state changes, denial-versus-ungranted presentation, filtering, row-subject switching, bulk update, and the explanation panel

## 11. Sprint 10 — Admin Cockpit & Break-glass

Goal: administrators can run access without touching candidate data. Covers `access-control/admin-cockpit`.

- [ ] 11.1 Restrict every Admin Cockpit page and endpoint to Application and System Administrators, and hide admin navigation from everyone else
- [ ] 11.2 Build user administration: create, activate, deactivate, update, and role assignment, each audited
- [ ] 11.3 Build user list filtering by status and role
- [ ] 11.4 Build role administration: create, update, deactivate, and permission assignment, rejecting duplicate role names with a field-level error
- [ ] 11.5 Implement break-glass as a time-boxed user-level override requiring a typed reason, audited, and expiring automatically without administrator action
- [ ] 11.6 Implement break-glass notification to the configured recipients, defaulting to other Application Administrators and the Auditor role
- [ ] 11.7 Test that an administrator without an active elevation is denied candidate personal data, and that elevation without a reason is rejected
- [ ] 11.8 Build the forced-session-revocation control into the user administration surface

## 12. Sprint 11 — Workflow Engine

Goal: the transition framework, with no domain machine inside it. Covers `platform/workflow-engine`.

- [ ] 12.1 Implement the declarative state machine registration format covering states, permitted transitions, reason requirements, and the permission each transition requires
- [ ] 12.2 Implement the transition executor as the only write path to a state field, rejecting direct state writes
- [ ] 12.3 Implement transition validation, rejecting undefined transitions with a response naming the current state and the available transitions
- [ ] 12.4 Reject unknown transition names outright rather than treating them as no-ops
- [ ] 12.5 Implement transition records capturing actor, prior state, new state, reason, timestamp, and source module, each paired with an audit record
- [ ] 12.6 Implement service-account attribution correlated to the triggering event for system-generated transitions
- [ ] 12.7 Implement reason enforcement on transitions marked reason-required, rejecting with a field-level error and no state change
- [ ] 12.8 Implement per-transition permission checks through the evaluator, before any state change
- [ ] 12.9 Implement optimistic concurrency so simultaneous transitions leave exactly one winner and report the current state to the loser
- [ ] 12.10 Verify history is retained after a resource reaches a terminal state, and that the wave ships with zero domain state machines registered
- [ ] 12.11 Test with a throwaway fixture machine that the framework needs no modification to register a new machine

## 13. Sprint 12 — Tasks & Notifications

Goal: work reaches the person who owns it. Covers `platform/notifications`.

- [ ] 13.1 Migrate and implement the task model: type, subject reference, assignee, owner, timestamps, target date, status, and completion attribution
- [ ] 13.2 Build the per-user task list with outstanding and completed views
- [ ] 13.3 Implement task reassignment by authorized users, audited
- [ ] 13.4 Implement permission-aware task presentation so a task whose subject the recipient cannot read omits the restricted detail and denies opening it
- [ ] 13.5 Implement aging calculation against runtime-configurable thresholds
- [ ] 13.6 Implement in-app notification delivery with per-user read state
- [ ] 13.7 Implement internal email delivery with configurable templates
- [ ] 13.8 Implement the sender-side internal-recipient guard, refusing and recording any recipient that does not resolve to an internal user
- [ ] 13.9 Implement retry with backoff for transient failures, and record permanent failures with their reason, visible to authorized users
- [ ] 13.10 Test that a workflow action completes successfully while the email service is down, with the notification queued for retry
- [ ] 13.11 Populate the landing route from Sprint 5 with the signed-in user's outstanding tasks and unread notifications, scoped to what they may see
- [ ] 13.12 Write the integration test suite for the notification service covering delivery, retry with backoff, permanent failure, and the internal-recipient guard

## 14. Sprint 13 — Run Log Substrate, AI Gateway & Prompt Registry

Goal: a single governed egress and ten contracted families — with the run log in place **before** the first provider call, since `design.md` D9 makes write-before-invoke an invariant and no AI run may exist unlogged.

Covers `ai-platform/ai-run-logging` (substrate), `ai-platform/ai-gateway`, and `ai-platform/prompt-registry`.

- [ ] 14.1 Migrate `ai_run_logs` with provider, model, model version, template ID and version, input and output references, token usage, safety flags, status, timings, and error detail
- [ ] 14.2 Grant the application database role `INSERT` and `SELECT` only on `ai_run_logs`
- [ ] 14.3 Implement write-before-invoke ordering, and test that a run whose log cannot be written is never issued to the provider
- [ ] 14.4 Implement the gateway as the sole AI egress, with provider credentials reachable only from it, and verify no feature can call a provider directly
- [ ] 14.5 Implement per-family model configuration resolution, failing a run with a configuration error where a family has none rather than falling back to a default model
- [ ] 14.6 Implement Run AI permission verification at the gateway before any provider invocation
- [ ] 14.7 Implement untrusted-content isolation: system instructions separated from document-derived content, content sanitized and delimited, and tool capability restricted during document processing
- [ ] 14.8 Implement personal-data minimization: contact fields excluded from prompt payloads by default, recorded on the run, with approved exceptions logged
- [ ] 14.9 Implement protected-attribute redaction for scoring inputs
- [ ] 14.10 Implement rate limiting, per-family input size bounds, and consumption caps
- [ ] 14.11 Implement asynchronous execution returning a job identifier with a retrievable status endpoint
- [ ] 14.12 Implement graceful degradation so a provider outage leaves all non-AI pages, records, and workflow actions fully usable, with AI controls reporting temporary unavailability
- [ ] 14.13 Implement per-item success and failure reporting for partial batch failures, and safe retry preserving failure detail without touching workflow state
- [ ] 14.14 Migrate and implement the prompt template registry with versioning, active-version configuration, and an audited version switch
- [ ] 14.15 Register all ten prompt families with stub prompt text, each carrying a machine-checkable output contract
- [ ] 14.16 Implement contract validation in the gateway, recording violations as failed runs and persisting no insight from invalid output
- [ ] 14.17 Add conciseness bounds to every free-text field of every family's contract, validated by the same code that validates shape, with provisional numbers recorded as such
- [ ] 14.18 Add evidence-reference, source-label, and insufficiency clauses to the contracts of every family producing candidate insight
- [ ] 14.19 Write the integration test suite for the gateway against a provider stub, covering success, contract violation, timeout, outage degradation, and rate-limit rejection
- [ ] 14.20 Verify no user-facing screen invokes any family, and that each family is exercisable through the harness and an authorized diagnostic path
- [ ] 14.21 Verify by inspection that no invocation path exists which bypasses the run log, before this sprint is accepted

## 15. Sprint 14 — Run Log Content, Overrides & Feedback

Goal: the governance record is complete and queryable. Covers `ai-platform/ai-run-logging`.

- [ ] 15.1 Implement reference-only input capture, and test that no resume text or candidate contact data can reach a run record
- [ ] 15.2 Implement safety-flag capture with retrieval by flag category
- [ ] 15.3 Verify the reproducibility tuple: input references, template version, and model version together identify what produced any past output
- [ ] 15.4 Implement AI-generated and AI-assisted marking that persists until a human approval is recorded with actor and timestamp
- [ ] 15.5 Implement the human override record: mandatory reason, original AI output preserved and retrievable, rejecting an override with no reason
- [ ] 15.6 Verify divergence between AI output and human decisions is computable from stored override records alone
- [ ] 15.7 Implement AI output feedback with categories for inaccurate, incomplete, biased, unsafe, and not useful, stored against the run and leaving the output unchanged
- [ ] 15.8 Implement run log search filterable by family, template version, model version, status, safety flag, and time range, denying callers without permission
- [ ] 15.9 Verify an Auditor can read run records without any candidate personal data in the payload, and implement retention with recorded disposal
- [ ] 15.10 Verify the gateway holds no capability to trigger a state transition, closing the advisory-only guarantee structurally

## 16. Sprint 15 — AI Evaluation & Security Harness

Goal: nothing reaches production untested. Covers `ai-platform/ai-evaluation`.

- [ ] 16.1 Build the versioned corpus structure with results recorded against specific template and model versions
- [ ] 16.2 Author low-information cases for every candidate-consuming family, asserting an insufficiency result rather than fabricated detail
- [ ] 16.3 Author adversarial and prompt-injection cases, asserting injected instructions do not take effect and output stays within contract
- [ ] 16.4 Author conflicting-evidence cases, asserting the conflict is reported rather than silently resolved
- [ ] 16.5 Author protected-attribute cases, asserting absence from both output and scoring rationale
- [ ] 16.6 Author evidence-citation cases, asserting every candidate claim carries a resolvable reference
- [ ] 16.7 Author conciseness cases per family, asserting declared bounds hold
- [ ] 16.8 Build the security suite: broken access control, file upload attacks, sensitive data exposure in responses, logs and run records, and export permission checks
- [ ] 16.9 Wire both suites into CI so failure blocks merge
- [ ] 16.10 Implement the promotion gate refusing activation of a template version with no passing corpus run recorded
- [ ] 16.11 Implement corpus re-run and result recording when the configured model changes under an unchanged template
- [ ] 16.12 Document the process for adding a corpus case from a real failure found through output feedback or governance review
- [ ] 16.13 Verify no evaluation case collects or processes demographic or protected-class data, and record fairness measurement as pending legal and compliance approval

## 17. Wave 1 Close-out

Goal: the wave is verifiably done, and Wave 2 is unblocked.

- [ ] 17.1 Run the full test suite — unit, API, integration, UI, security, and AI evaluation — against Dev and record the results
- [ ] 17.2 Verify server-side authorization on every endpoint and page by enumeration, not by sampling
- [ ] 17.3 Verify every material write produces an audit record, by enumerating write endpoints
- [ ] 17.4 Confirm the seeded matrix posture has business sign-off, or record it as an accepted open risk carried into Wave 2
- [ ] 17.5 Confirm the break-glass notification target with Security, and update the configuration to match
- [ ] 17.6 Record the Hubble contract response for `OD-001`, and reconcile the adapter and its contract test against it
- [ ] 17.7 Verify WCAG 2.1 AA on the sign-in, shell, matrix, and administration screens
- [ ] 17.8 Demo the wave outcome: a user signs in through Hubble, is blocked while unactivated, an administrator activates and configures them, and their access matches the matrix with the explanation panel accounting for it
- [ ] 17.9 Update `design.md` Open Questions with what closed and what carries forward, then run `/opsx:archive` for this change
- [ ] 17.10 Confirm the Wave 2 preconditions are met: evaluator, workflow engine, audit substrate, AI gateway, prompt registry, run logging, shell, tokens, and dense table all in place and consumable
- [ ] 17.11 Measure and record the load-time baseline for every screen this wave ships — sign-in, landing, matrix, user and role administration — against the three-second threshold, so later regressions are detectable
- [ ] 17.12 Replace GitLab's boilerplate root README with a real project README pointing at the OpenSpec change structure

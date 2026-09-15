# Platform Core — Design

## Context

See `proposal.md` — Why, for motivation, and the six delta specs under `specs/` for the
behavior contracts. This document covers only the technical decisions `platform-core` must
settle, and the reconciliation between what Sprint 0 actually built and what the corrected
backlog says this feature owns.

Constraints that shape the approach:

- **This is not a greenfield feature.** `TS-BL-001`, `TS-BL-002` and `TS-BL-003` are deployed
  and verified live in GCP Dev. Their design decisions are inherited from
  `talentsphere-wave-1-foundation`'s `design.md`, not re-derived — and where Sprint 0 corrected
  a decision while building it, the correction is what carries forward, not the original.
- **A landing zone already exists** and is authoritative for anything project-scoped. Sprint 0's
  most expensive mistake was building project vending, CI identity and a pipeline before
  checking whether shared platform repositories were reachable; they were.
- **Settled stack** (`exploration-notes.md` D09): FastAPI on Python, React + TypeScript,
  PostgreSQL on Cloud SQL, Terraform on GCP with GitLab CI/CD, OpenTelemetry.
- **Only Local and Dev are provisionable.** The free-tier billing account caps linked projects;
  UAT and Prod are validated Terraform that is never applied, and QA was dropped outright
  because the GCP organization has no `fld-qa` and no QA network host (`design.md` D15 as
  amended 2026-08-21).
- **Three features are blocked on this one before they can start**: `identity-and-access` needs
  the ingress, `access-control-and-admin` needs the database, `design-system` needs the
  notification payload shape.

## Goals / Non-Goals

**Goals:**

- One dispatch pattern for background work, established once, so eleven consuming features
  inherit it rather than each choosing a queue.
- A workflow engine that is genuinely framework-only — provably operable with zero registered
  state machines, so it cannot quietly accrete domain logic.
- Substrate decisions that survive being consumed: a notification payload a rendering surface
  can bind to without knowing which feature produced it, and a transition record shape the
  audit writer can accept without modification.
- Honest reconciliation of the built half — residual gaps named as residual, not quietly
  described as complete.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No domain state machines.** The engine ships with none. Posting, Application, offer and
  closure machines belong to the features that own them, registered declaratively.
- **No permission evaluator, no audit writer, no AI gateway.** This feature depends on the
  first two and is depended on by the third; it implements none of them.
- **No visual components.** The notification payload contract is defined here; the toast that
  renders it is `design-system`'s `TS-BL-012`.
- **No retry of the Sprint 0 argument about environments.** Local and Dev only, QA dropped.
  That is arithmetic and organizational fact, not a preference to revisit.

## Decisions

### D1 — The landing zone owns the project; this repository owns what runs inside it

`platform-infra` creates the project, enables APIs, attaches the shared VPC, grants the subnet,
creates the Artifact Registry repository, and creates the CI service account with its workload
identity federation binding. This repository owns the workload: database, service, jobs,
ingress, and frontend bucket, at a single Terraform root per environment plus a thin pipeline
definition.

*Why:* Sprint 0 built its own version of all of the above because project memory recorded the
landing-zone repositories as inaccessible. When they turned out to be reachable, all of it was
migrated. Two problems were independently rediscovered along the way that the landing zone's own
comments already documented — an API that must be enabled on the *service* project, and a
storage permission authorized against the bucket rather than the prefix, so a prefix condition
can never grant it.

*The generalizable lesson, recorded because it will recur:* **check for an existing platform
repository before writing infrastructure.**

*Alternative considered:* keep the self-built vending, since it worked. Rejected — it would
have meant this project drifting from every other application project in the organization, and
paying that divergence forever to avoid one migration.

### D2 — Two state prefixes, so the deploy identity cannot plan against the database

Platform state and workload state live under separate prefixes in the shared state bucket. The
pipeline identity that deploys revisions never loads the platform stack into a plan.

*Why:* a plan that can see a resource can destroy it. Separating state means a workload deploy
has no capability to act on the project, the network attachment, or the CI identity — the same
argument D10 makes about advisory-only AI, applied to infrastructure: remove the capability
rather than rely on nobody using it.

*Naming note, worth stating because it is counter-intuitive:* the `app-*` prefix holds the
**platform** stack that creates the application's project; the bare application-name prefix
holds the **workload**. Getting these backwards points a deploy at the wrong state.

### D3 — The database has no password; two identities, split by what they may do

Inherited from `talentsphere-wave-1-foundation`'s D18, which was itself added mid-Sprint-0 after
that change originally shipped a Terraform-generated password in Secret Manager. Runtime and
migration are separate identities: the runtime holds DML only, the migrator owns the schema and
holds DDL. Both authenticate with short-lived tokens.

*Why the split, specifically:* `access-control-and-admin`'s audit design (that change's D7)
makes audit and AI run tables append-only by granting the runtime `INSERT` and `SELECT` only.
That guarantee is decorative if the same identity can `ALTER TABLE`. **This is the
cross-reference D.5 named as a risk when it split these decisions across changes** — the
enforcement mechanism lives here in `platform-core`, the requirement it enforces lives in
`access-control-and-admin`, and neither is complete without the other. `TS-BL-020` is where the
narrowing gets proven against a real Cloud SQL table; it has never run against one, because the
table does not exist yet.

*Why removing the credential beats protecting it:* a password in an approved secret manager
satisfies `DEP-003` literally. It also creates a credential that exists, sits in Terraform
state, and has to be rotated. Identity authentication removes the thing rather than guarding it.

*Consequence for Local:* a developer's containerized Postgres has no identity management, so the
connection layer supports both modes, selected by configuration — which leads directly to D4.

### D4 — Connection configuration is derived in exactly one module

*Why:* Sprint 0's one split-brain defect. The session module was updated for identity
authentication and the migration environment module was not. **Both files were internally
correct**, so no test failed — migrations kept using the password form and reached a loopback
address inside a Cloud Run job. The fix was not "update the second file" but "make a second
file impossible": one module derives the URL, and tests assert the consumers agree.

*The pattern worth generalizing:* two independently-correct implementations of the same rule is
a defect class that unit tests structurally cannot catch, because each passes its own. The
control is a shared source plus an agreement test, not more coverage on either side.

### D5 — Gates assert negatives and compare two sources of truth

The pipeline verifies that the digest reported by the *running service* matches the digest the
pipeline just built. Credential scanning asserts that no credential-shaped value exists.
Conditional gates fail when their condition is unmet rather than skipping.

*Why:* three Sprint 0 defects produced **green pipelines while doing nothing** — three days of
deploys that deployed nothing because `ignore_changes` on the service image discarded every new
digest while environment variables still updated, so the service truthfully reported a fresh
commit while running the first image ever built; a plan artifact that collected nothing because
variables are not expanded in artifact paths, so the plan an approver reads was never
downloadable; and a release-notes job behind a blocking manual gate in an earlier stage, so it
could never run.

**A green job and a healthy readiness probe are both true of a service running stale code.** The
gates that caught real problems in Sprint 0 were the ones asserting a negative or comparing two
sources of truth. That is the selection rule for every gate added from here.

*Alternative considered:* more integration tests. Rejected as the primary control — the failures
above were not behavioral, they were *absences*, and a test suite that runs against the wrong
artifact passes.

### D6 — Async orchestration consumes the landing zone's chain, rather than choosing a queue

Background work runs on Pub/Sub → Eventarc → dispatcher → Cloud Workflows → Cloud Run Job. **The
landing zone already implements this chain as reusable Terraform modules; TalentSphere consumes
them.** The same relationship this repository already has with the shared pipeline template,
which it consumes by pinned tag rather than reproducing.

*Why this is not really an architecture choice:* it is D1's argument applied to a second domain.
The organization runs this pattern, the modules exist and work, and the alternative is
diverging from the platform every other application project uses. Sprint 0 learned what that
costs — its most expensive mistake was building bespoke project vending because the landing zone
was assumed unavailable, and two problems were rediscovered that the landing zone's own comments
already documented.

**This supersedes "Cloud Tasks, Celery, or RQ"** in `exploration-notes.md` D09's settled-stack
table — now recorded there as an explicit supersession block rather than left as a contradiction
between D09 and D.9's `TS-BL-006` row.

*Cloud Tasks, considered seriously and rejected on grounds that are not technical:* it is
genuinely the simpler option at TalentSphere's scale — one concept rather than five, native
task-name deduplication (which would satisfy this feature's idempotency requirement nearly for
free), per-queue retry configuration, and a far smaller Terraform and IAM surface for a team of
3-4 junior developers. Two arguments commonly made against it do **not** hold here and are
recorded so they are not repeated: it does not force work to run inside the API service (its
target can be a separate worker service, satisfying `DEP-006`), and its lack of fan-out is
irrelevant in an application whose producers already know their consumers. It loses for one
reason only — **the chain's complexity is already paid for, and Cloud Tasks would mean paying to
diverge.** Were TalentSphere standalone, this decision would likely go the other way.

*What TalentSphere still owns, module or not:* the dispatcher's routing table, the job-type
registry, per-type retry declarations, idempotency at the effect boundary, and the job-status
record. The modules provide transport and execution; none of those are transport concerns.

*One honest gap:* Cloud Tasks' native task-name deduplication has no direct equivalent here, and
Pub/Sub guarantees at-least-once delivery. Idempotency must therefore be implemented at the
effect boundary rather than inherited — a real cost of this choice, and the reason it is a
distinct spec requirement rather than an implementation note.

### D7 — The workflow engine is proven framework-only by shipping with zero machines

States, permitted transitions, reason requirements, and the permission each transition demands
are declared by the registering feature. The framework contains no domain vocabulary.

*Why the zero-machine test matters:* "framework-only" is the kind of claim that decays. A
deployment assertion that the service is operational with no posting, Application, offer or
closure machine defined is checkable; a code-review convention is not. `WF-001` and `NFR-009`
require centralized business rules precisely so features cannot reimplement transitions, and
the guarantee only holds if the framework stays empty.

*Concurrency:* optimistic — a transition carries the state it believes the resource to be in,
and loses if that changed. *Alternative considered:* row locking for the duration of the
transition. Rejected — transitions are short and contention is low at this user population,
and a lock held across an audit write is a lock held across a second table's I/O.

### D8 — A transition and its audit record commit together, against an interface that already exists

The transition record and its audit record are written in the same transaction; a failed audit
write fails the transition.

*Why this is cheap to build now:* Sprint 0 shipped `app/audit/port.py` as a port with a logging
sink that records `durable: false`. `access-control-and-admin`'s `TS-BL-020` substitutes the
durable writer behind it with **no call-site changes**. The workflow engine therefore writes
against the port, not against a table, and does not need `TS-BL-020` to be finished first.

*The sequencing this buys:* `TS-BL-004` depends on `TS-BL-002` (the database) and not on
`TS-BL-020` (the audit writer), which is what lets the workflow engine and the audit substrate
be built concurrently by different people. This is the same stub-and-integrate shape Sprint 0
already used once and found sound.

### D9 — Notifications carry references and refuse external recipients at the sender

Two constraints, both structural rather than procedural. Payloads hold record references,
resolved against the reader's permissions at read time. Any recipient not resolving to an
internal user record is refused by the sender.

*Why references rather than copies:* `design.md` D6's reasoning applied to a second surface. The
audit trail carries references specifically so it does not become a personal-data back door for
the config-only Administrator and the read-only Auditor. A notification store that copied
candidate detail at send time would reopen exactly that hole through a different table — and
would serve stale values after the underlying record changed.

*Why the guard is at the sender:* `C-04` defers candidate-facing communication pending legal
sign-off (`OD-005`). "We will not send to candidates" is not a control; a misconfigured template
violates it silently. A sender-side check means the deferral cannot be violated by
configuration. `resurfacing-and-communications`' `TS-BL-078` is where the gate is deliberately
opened, and it remains blocked on `OD-005` — a real blocker, not a sequencing note.

*The payload contract exists for a consumer that does not exist yet:* `design-system`'s
`TS-BL-012` depends on this item specifically because a toast needs a real payload shape rather
than a visual mock. Severity is defined here as semantics; its appearance is the design system's
status-surface triplets (`reference/design-spec.md` §1.5). Defining severity as an enumeration
rather than a color is what keeps that boundary from inverting.

*Delivery dispatches through the async substrate — it does not get its own retry mechanism.*
Notification delivery is background work, and `platform/async-orchestration` requires one
dispatch pattern with a scenario that fails the test suite on background work started outside
it. Notifications are that capability's most obvious consumer, so an independent retry loop here
would be the first exception to a rule written to have none — and would build retry-with-backoff,
status visibility, and permanent-failure recording twice inside one feature, since both specs
already state all three. `reference/spec.md` §25 agrees, listing Notifications among the
background job types alongside resume parsing and ranking.

*The cost of that, stated rather than discovered later:* it adds `TS-BL-005 → TS-BL-006`, which
makes this feature fully sequential and transitively puts the notification engine behind the
workflow engine (defensible — notifications are triggered by workflow events). The in-app
channel is unaffected: it is a synchronous write and needs no dispatch, which the spec says
explicitly so the coupling is not read as wider than it is. And the **payload contract itself
has no dispatch dependency** — it is a schema definition, so the thing `TS-BL-012` actually
needs can land well ahead of the rest of `TS-BL-005`.

### D10 — Open work on the built items is completion scope, not rediscovery

`TS-BL-001` and `TS-BL-002` are recorded as in-progress rather than done. Five sub-tasks remain
open across them, in two distinct kinds — the distinction matters, because only the first kind
is something Sprint 0 left unfinished:

**Literal Sprint 0 residuals** — built, but incompletely:

- **No edge serves the frontend** (`1.15`). The bucket exists and the pipeline syncs into it; a
  load balancer and CDN are the remaining work. This is `TS-BL-001` because no other item in the
  79-item backlog covers serving the frontend.
- **Schema ownership is granted by hand** (`2.8`). No automated identity holds the rights to
  reassign it, so a console was used. Re-provisioning Dev or standing up UAT repeats the step.
  Automating it needs an administrative identity the pipeline can assume, which does not exist —
  so this gap is *blocked on a decision*, not merely unstarted, and is called out as such.

**Obligations Sprint 0 could not discharge**, because the thing they apply to did not exist yet.
These are not leftovers; they are requirements whose subject arrives with this feature:

- **Artifact promotion is unverified** (`1.16`). The runtime config-resolution mechanism was
  built, but that one artifact promotes between environments unchanged has never been exercised
  — there is only one real environment to promote into.
- **No performance baseline exists** (`1.17`). The three-second threshold is a spec requirement,
  and Sprint 0 shipped no authenticated screen to measure, so there is nothing yet for a later
  regression to be detected against.
- **No Cloud SQL integration harness exists** (`2.9`). Grants are currently verified against a
  local database where the developer is a superuser, which is precisely where a
  least-privilege narrowing cannot be proven. `access-control-and-admin`'s `TS-BL-020` needs
  this harness to prove the `audit_logs` narrowing against a real table.

*Gaps that are not this feature's* are cross-referenced rather than absorbed: the unexercised
audit narrowing is `TS-BL-020`'s task, the unclassified Sprint 0 endpoints are
`access-control-and-admin`'s enumeration, and `/build` reporting `ref` and `pipeline_id` as
unknown is documented as deliberate — the template passes commit and digest, which is what
traceability actually requires.

### D11 — What this feature inherits from `talentsphere-wave-1-foundation`, and what it leaves behind

Per D.5 as adjusted to feature level by D.8.4, that change **splits rather than being
rewritten**. `platform-core` inherits its D15 (environment topology, as amended), D16
(internal-only notifications) and D18 (identity database authentication), and the delta specs
`platform/delivery-foundation`, `platform/workflow-engine` and `platform/notifications` — whose
capability paths are preserved exactly rather than renamed.

It leaves behind everything that belongs to another feature: D1–D7 to
`access-control-and-admin`, D8 to `identity-and-access`, D9–D12 to `ai-platform-governance`,
D13–D14 and D17 to `design-system`.

*What this change does not do:* retire or archive `talentsphere-wave-1-foundation`. Both changes
therefore describe the same three capability paths until that migration completes across all
five Phase-1 features. That is a known, accepted interim state — the alternative is one feature
unilaterally deleting content four other features still need to inherit.

## Risks / Trade-offs

- **Two active changes describe the same capability paths** → Accepted as interim. The
  wave-1-foundation change is the source being migrated *from*; `platform-core` is authoritative
  for platform-core's six items from now on. Resolved when all five Phase-1 features are
  proposed and that change is retired.
- **The async substrate is built before any consumer exists** (the same shape as the risk
  `talentsphere-wave-1-foundation` accepted for the AI substrate) → Mitigated by exercising it
  end-to-end with a real job in this feature rather than leaving it compiled but unrun. Residual
  risk accepted: Phase 2's first real consumer will refine it, and that is expected.
- **Consuming landing-zone modules couples this feature to their interface**, which is owned by
  another repository and can change → Mitigated the same way the pipeline template already is:
  consumed by pinned version, so an upstream change is adopted deliberately rather than
  arriving unannounced. `TS-BL-006`'s first task is confirming that interface and recording the
  pinned version.
- **Pub/Sub is at-least-once, so duplicate delivery is normal rather than exceptional** →
  Idempotency is a spec requirement with its own scenario, enforced at the effect boundary. This
  is the one place the rejected alternative was genuinely cheaper (see D6), so it is the one
  place most likely to be under-built if treated as an implementation detail.
- **Schema ownership automation is blocked on an identity that does not exist** → The gap is
  recorded as blocked rather than pending, so it is visible at scheduling time instead of
  surfacing when UAT is stood up. Standing up UAT is not currently possible anyway, which buys
  time but does not remove the need.
- **The workflow engine is built before any state machine registers against it**, risking an
  abstraction that fits no real caller → Mitigated by designing the registration surface against
  two real machines already specified elsewhere — the posting lifecycle and the
  candidate-posting lifecycle in the reference spec §11.1 and §11.2 — without registering them.
- **Unapplied UAT and Prod Terraform drifts from reality in ways validation does not catch** →
  Inherited risk, unchanged. Mitigated by keeping environments structurally identical and
  parameterized by variables only, which is now a spec requirement rather than a convention.
- **Correlation identifiers are easy to lose at an asynchronous boundary**, which would make the
  observability requirement true only within a request → Made a spec requirement with its own
  scenario rather than left as an implementation detail, because it is invisible when broken.

## Migration Plan

There is no data migration; this is substrate. Sequence:

1. **Close the two named residual gaps first** (`TS-BL-001`, `TS-BL-002`). Both touch live
   infrastructure, both are small, and leaving them open makes every later item's deployment
   story slightly untrue.
2. **`TS-BL-004` before `TS-BL-006`.** The async pattern's dispatcher needs a transition
   vocabulary to attribute background transitions against; building it first would mean
   inventing one and then replacing it.
3. **`TS-BL-005` last, not concurrently.** An earlier draft of this plan had it running parallel
   to `TS-BL-004` on the basis that it depended only on `TS-BL-002`; resolving the dispatch
   question in D9 added `TS-BL-006` to its dependencies and removed that parallelism. This
   feature is now a single chain: `001 → 002 → 004 → 006 → 005`. **Pull task 5.4 forward**
   regardless — the payload contract has no dispatch dependency, and `design-system`'s
   `TS-BL-012` is blocked on it rather than on the finished engine.
4. **Feature-flag the async substrate.** It ships disabled and is enabled in Dev by a flag
   change, which also exercises the flag mechanism a second time before Phase 2 depends on it.
5. **Rollback** is redeploy-previous-artifact plus migration-down. This feature creates tables
   and infrastructure rather than transforming data, so rollback stays low-risk throughout.

## Open Questions

Each is genuinely deferrable — none changes the specs, the approach, or the task breakdown.

- **Which administrative identity automates schema ownership** (D10's second gap). Needs an
  owner decision about what the pipeline may assume. Until then the manual step is documented
  in `KNOWN_ISSUES.md` and repeats on re-provision.
- **Frontend edge shape** — load balancer with CDN, or a simpler static-hosting arrangement. The
  spec requires TLS and runtime API-address resolution; both options satisfy it.
- **Backlog threshold values** for the queue-depth alert. The alert is required; the number
  belongs to whoever operates Dev once real work flows through it.
- **Whether UAT is ever stood up under the current billing cap.** Affects how much the manual
  schema-ownership step actually costs. Not blocking — nothing in this feature assumes it.

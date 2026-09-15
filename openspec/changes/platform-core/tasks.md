# Platform Core — Backlog

This feature's complete backlog: six items, `TS-BL-001` through `TS-BL-006`, decomposed in
`exploration-notes.md` D.9. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.
Assigning sprints here would be inventing a schedule this feature has no authority to set. Each
item still carries a stated goal.

**Three items are partly or wholly built.** `TS-BL-001`–`TS-BL-003` were shipped by Sprint 0 of
`talentsphere-wave-1-foundation` and are live in GCP Dev. Their completed work is checked off
rather than restated as future work; only genuinely remaining scope is open. See that change's
`sprint-0-outcome.md` for the historical record and `design.md` D10/D11 here for what was
inherited.

---

## 1. TS-BL-001 — Base infrastructure: workload Terraform skeleton and CI/CD pipeline

**Goal:** every commit is built, tested, and deployable by the same pipeline that will deploy
it, into infrastructure defined entirely as code. Covers `platform/delivery-foundation`.

Scope note: this item is **wider than its D.9 one-line title**. It also carries the
application-layer platform baseline Sprint 0 shipped alongside the pipeline — error handling,
observability, feature flags, runtime configuration. No other item in the 79-item backlog covers
these, and D.11 permits refining item boundaries within a feature.

```yaml
backlog_items:
  - id: TS-BL-001
    feature: platform-core
    depends_on: []
    status: in-progress
```

- [x] 1.1 Create the repository skeleton — FastAPI backend, React + TypeScript frontend,
      Terraform, pipeline definitions — with linting and formatting configured per language
- [x] 1.2 Adopt the landing zone: move project vending, CI identity, WIF binding and Artifact
      Registry to `platform-infra`, keeping only the workload in this repository (design D1)
- [x] 1.3 Separate platform and workload Terraform state into two prefixes, so the deploy
      identity never loads the platform stack into a plan (design D2)
- [x] 1.4 Write workload Terraform for all four environments at one root per environment,
      structurally identical and parameterized by variables only
- [x] 1.5 Verify UAT and Prod definitions pass `validate` and `plan` in CI without provisioning
      any billable resource
- [x] 1.6 Consume the shared pipeline template by pinned tag, adding the gates it lacks (ruff,
      mypy, bandit, gitleaks, release notes) and explicitly overriding its `test:backend` and
      `security:dependency-scan` jobs, which are gated on a file this project does not use and
      would otherwise skip silently with the whole backend suite behind them
- [x] 1.7 Wire secret injection from the secret manager, with a CI gate failing on any
      credential-shaped value committed to source or configuration
- [x] 1.8 Set up versioned, reversible migrations applied by a dedicated job run with `--wait`,
      so a failed migration fails the deploy rather than leaving services on an unmigrated schema
- [x] 1.9 Add the provenance gate: compare the digest reported by the running service against
      the digest the pipeline built, and fail on mismatch (design D5)
- [x] 1.10 Add OpenTelemetry structured logging, metrics and traces, with a correlation ID that
      survives into background tasks and a redaction processor applied centrally rather than at
      call sites
- [x] 1.11 Implement the global error handler producing one body shape — code, message, trace
      ID, correlation ID, field errors — asserting no stack trace, secret, or prompt content
      reaches a response, and that rejected input values are not echoed back
- [x] 1.12 Implement the feature-flag mechanism against a closed declared registry, where an
      unknown flag raises rather than resolving false and a disabled capability answers 404
- [x] 1.13 Implement the runtime configuration store with a closed registry and an audited
      change path carrying a mandatory reason, with no unaudited setter
- [x] 1.14 Add the release-notes step publishing changes, included migrations, and known issues,
      placed so it stays reachable regardless of any blocking manual approval gate
- [ ] 1.15 Provision the frontend edge — load balancer and CDN in front of the existing bucket —
      serving over TLS, so the built frontend is actually reachable
- [ ] 1.16 Verify one built frontend artifact resolves its API address at runtime and can be
      promoted between environments unchanged, without a rebuild
- [ ] 1.17 Establish the interactive performance measurement harness and record a first baseline,
      so later features have something to regress against rather than a threshold nobody measures

---

## 2. TS-BL-002 — Cloud SQL and IAM database authentication wiring

**Goal:** services reach the database privately and without any stored credential, under
identities separated so that append-only grants are enforceable rather than decorative. Covers
`platform/database-access`.

```yaml
backlog_items:
  - id: TS-BL-002
    feature: platform-core
    depends_on: [TS-BL-001]
    status: in-progress
```

- [x] 2.1 Provision Cloud SQL PostgreSQL with a private IP, no public address, and encrypted
      connections only
- [x] 2.2 Create two database identities — a runtime identity holding DML only, and a migration
      identity owning the schema and holding DDL (design D3)
- [x] 2.3 Enable IAM database authentication and remove the generated password entirely, so no
      database credential exists in source, in Terraform state, or in the secret manager
- [x] 2.4 Grant the runtime and migration identities the roles needed to reach the instance and
      log in as IAM database users, with no secret-manager entry for the database
- [x] 2.5 Derive the connection URL in exactly one module, with tests asserting the application
      and the migration runner resolve from it and agree (design D4)
- [x] 2.6 Support password authentication for Local and IAM authentication for deployed
      environments, selected by configuration rather than by branching at call sites
- [x] 2.7 Write the least-privilege grant script, run as instance admin outside Alembic since a
      role cannot grant itself privileges
- [ ] 2.8 Automate schema ownership assignment, currently applied by hand through the console on
      every re-provision. **Blocked** — needs an owner decision on which administrative identity
      the pipeline may assume (design D10, Open Questions)
- [ ] 2.9 Build the integration suite that verifies grants against a real Cloud SQL instance
      rather than a local database where the developer is a superuser. `access-control-and-admin`'s
      `TS-BL-020` consumes this harness to prove the `audit_logs` narrowing, which has never run
      against a real table

---

## 3. TS-BL-003 — API Gateway ingress skeleton

**Goal:** one controlled way in, with no path around it. Covers `platform/api-ingress`.

```yaml
backlog_items:
  - id: TS-BL-003
    feature: platform-core
    depends_on: [TS-BL-001]
    status: done
```

- [x] 3.1 Provision the API Gateway with its own service identity, generating the ingress
      configuration from the Terraform definitions rather than maintaining it by hand
- [x] 3.2 Grant service invocation rights to the gateway identity and to no other caller, so a
      direct call to the service address is refused
- [x] 3.3 Accept only TLS-encrypted connections at the edge
- [x] 3.4 Assign a correlation identifier at the edge when the client supplies none, carry it
      through the request, and echo it on the response
- [x] 3.5 Route the pipeline's post-deploy health verification through the gateway rather than
      around it, so the check exercises the real request path

**Cross-feature obligation, tracked not absorbed:** Sprint 0's four endpoints (`/health`,
`/build`, and two diagnostics routes) carry no declarative permission requirement, because the
mechanism arrives with `access-control-and-admin`'s `TS-BL-018`. That feature's endpoint
enumeration must classify these four rather than skip them. Diagnostics are restricted to
non-production environments meanwhile.

---

## 4. TS-BL-004 — Workflow / state-machine engine substrate

**Goal:** every domain state change in the product executes through one framework, which itself
contains no domain logic. Covers `platform/workflow-engine`.

```yaml
backlog_items:
  - id: TS-BL-004
    feature: platform-core
    depends_on: [TS-BL-002]
    status: not-started
```

- [ ] 4.1 Design the declarative registration surface — states, permitted transitions, reason
      requirements, and the permission each transition demands — against the posting and
      candidate-posting lifecycles in `reference/spec.md` §11.1 and §11.2 **without registering
      either**, so the surface is shaped by real machines while the framework stays empty
- [ ] 4.2 Migrate the transition record: actor, prior state, new state, reason, timestamp,
      correlation identifier, and source module
- [ ] 4.3 Implement the transition service so it is the only path that writes a workflow-governed
      state field, and add a test asserting no other code path does
- [ ] 4.4 Implement rejection of transitions the current state does not define, naming the
      current state and the transitions available from it
- [ ] 4.5 Implement rejection of unknown transition names as errors rather than no-ops
- [ ] 4.6 Enforce reason on any transition a machine marks reason-required, as a field-level
      validation error with no state change
- [ ] 4.7 Write the transition and its audit record in one transaction through the existing
      `app/audit/port.py`, so a failed audit write fails the transition and `TS-BL-020` can
      substitute the durable writer with no call-site change (design D8)
- [ ] 4.8 Attribute system-generated transitions to a service account, correlated to the
      triggering event, never to an arbitrary human user
- [ ] 4.9 Enforce the transition's declared permission through the permission evaluator before
      any state change. Depends on `access-control-and-admin`'s `TS-BL-018`; build against its
      interface if it has not landed
- [ ] 4.10 Reject transitions from an unauthenticated actor, and assert no caller can trigger a
      transition by producing advisory output — the workflow half of the advisory-only guarantee
      that `ai-platform-governance`'s `TS-BL-029` completes
- [ ] 4.11 Implement optimistic concurrency so exactly one of two simultaneous transitions
      succeeds and the other is rejected with the current state reported back
- [ ] 4.12 Retain transition history for the life of the resource, including past terminal states
- [ ] 4.13 Prove the framework ships empty: assert at deploy time that the service is operational
      with no posting, Application, offer, or closure machine registered

---

## 5. TS-BL-005 — Notification engine, internal delivery

**Goal:** workflow events reach internal people reliably, and cannot reach anyone else. Covers
`platform/notifications`.

```yaml
backlog_items:
  - id: TS-BL-005
    feature: platform-core
    depends_on: [TS-BL-002, TS-BL-006]
    status: not-started
```

**`TS-BL-006` is a dependency because delivery dispatches through the shared async substrate
rather than defining its own retry mechanism** (design D9). That makes this the last item in the
feature. **Task 5.4 does not wait for it** — the payload contract is a schema definition with no
dispatch dependency, and `design-system`'s `TS-BL-012` is blocked on that contract rather than
on the finished engine, so 5.4 should be pulled forward ahead of the rest of this item.

- [ ] 5.1 Migrate the task model: type, subject reference, assignee, owner, created, due,
      status, and completion attribution
- [ ] 5.2 Implement task assignment, completion with attribution, and audited reassignment
- [ ] 5.3 Implement the sender-side internal-recipient guard, refusing and recording any
      recipient that does not resolve to an internal user record — regardless of what a template
      is configured with (design D9)
- [ ] 5.4 Define the delivery payload contract — severity, title, body, originating event
      reference, optional action reference — as the shape `design-system`'s `TS-BL-012` renders
      against. Severity is an enumeration here; its appearance belongs to the design system's
      status-surface triplets
- [ ] 5.5 Store notification payloads as record references rather than copies of personal data,
      resolved against the reader's permissions at read time
- [ ] 5.6 Implement the in-app channel, with read state tracked per user
- [ ] 5.7 Implement the internal email channel, addressed to corporate addresses only
- [ ] 5.8 Register email delivery as a job type against `TS-BL-006`'s dispatch pattern, declaring
      its retry policy per channel rather than implementing a retry loop here. The in-app channel
      is a synchronous write and registers nothing
- [ ] 5.9 Surface delivery status to authorized users from the substrate's job-status record
      rather than a second status store
- [ ] 5.10 Enforce task visibility through the permission evaluator, presenting a task without
      subject detail the recipient cannot read and denying the subject itself
- [ ] 5.11 Make aging thresholds and notification templates runtime-configurable, through the
      audited configuration path rather than a deployment
- [ ] 5.12 Prove an email outage does not block work: the workflow action succeeds and the
      notification queues for retry

---

## 6. TS-BL-006 — Async orchestration pattern

**Goal:** one dispatch pattern for background work, established once and inherited by every
feature that needs it. Covers `platform/async-orchestration`.

```yaml
backlog_items:
  - id: TS-BL-006
    feature: platform-core
    depends_on: [TS-BL-001, TS-BL-004]
    status: not-started
```

- [ ] 6.1 Confirm the landing zone's async modules — their input/output interface, their required
      IAM, and the version to pin — and record the pinned version the way the shared pipeline
      template's tag already is (design D6). **Do this first**: it determines how much of the rest
      of this item is configuration rather than code
- [ ] 6.2 Consume the module providing Pub/Sub topics and subscriptions plus the Eventarc triggers
      that deliver message events and resource events through one surface, parameterized for
      TalentSphere's job types
- [ ] 6.3 Consume the module providing the Cloud Workflows definition and Cloud Run Job execution
      target, reusing the one-image/overridden-command pattern the migration job already proves
- [ ] 6.4 Build the dispatcher's routing table and job-type registry — TalentSphere's own, not the
      module's — so routing lives in one testable place and a new job type is a registration
      rather than an infrastructure change
- [ ] 6.5 Implement job submission returning a job identifier promptly, with the work executing
      outside the interactive request
- [ ] 6.6 Implement status retrieval by job identifier — pending, running, succeeded, failed —
      preserving failure detail for investigation
- [ ] 6.7 Carry the triggering request's correlation identifier through the event, the
      dispatcher, and the job, and assert the whole chain is traceable from one identifier
- [ ] 6.8 Declare retry policy per job type, including job types declared unsafe to retry, which
      record a failure for human action instead
- [ ] 6.9 Implement idempotency at the effect boundary so a duplicate event delivery produces the
      effect once. Pub/Sub is at-least-once, and unlike the rejected Cloud Tasks alternative there
      is no native task-name deduplication to inherit — this is the one place D6's choice is
      genuinely more expensive, so it is built deliberately rather than assumed
- [ ] 6.10 Retain work that exhausts its retry policy with its failure reason, surfaced to
      authorized users rather than discarded
- [ ] 6.11 Configure the queue-depth backlog alert, alongside the job-failure-spike alert the
      observability requirement already calls for
- [ ] 6.12 Execute background jobs as a named service account, subject to the same permission
      evaluation as any other actor, denying actions its account does not hold
- [ ] 6.13 Prove graceful degradation: with an external dependency unavailable, interactive reads
      and non-dependent workflow actions continue, and affected work queues or records a failure
- [ ] 6.14 Ship the substrate behind a feature flag, enabling it in Dev by flag change — which
      exercises the flag mechanism a second time before Phase 2 depends on it
- [ ] 6.15 Exercise the whole chain end-to-end with one real job, so the pattern is run rather
      than merely compiled before its first Phase 2 consumer arrives

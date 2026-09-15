# Access Control and Admin — Backlog

This feature's complete backlog: ten items — `TS-BL-018` through `TS-BL-026` from
`exploration-notes.md` D.9, plus `TS-BL-080` from D.10.1. The range is non-contiguous by design:
`AGENTS.md` fixes backlog IDs globally and permanently, so an item discovered later takes the next
free number rather than triggering a renumber. Each item is independently deployable to dev, uat
and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built.** Sprint 0 of `talentsphere-wave-1-foundation` shipped platform-layer code
only; the permission model and the audit table were Sprint 1 and Sprints 6–10 in the superseded
plan, and none of those sprints ran. Three Sprint 0 artifacts are consumed rather than rebuilt:
`backend/app/audit/port.py`, the already-written `audit_logs` narrowing in
`backend/sql/01_least_privilege.sql`, and the edge-assigned correlation identifier.

**Dependency edges pointing outside this feature.** `TS-BL-018` needs `identity-and-access`'s
`TS-BL-014`; `TS-BL-020` needs `platform-core`'s `TS-BL-002` and `identity-and-access`'s
`TS-BL-017`; `TS-BL-023` needs `design-system`'s `TS-BL-008`; `TS-BL-080` needs
`identity-and-access`'s `TS-BL-016`. In the other direction, `platform-core`'s `TS-BL-004`,
`TS-BL-005` and `TS-BL-006` each carry a task that completes only once `TS-BL-018` lands, and
eleven features name `TS-BL-018` or `TS-BL-020` as an enabler.

**Four boundary refinements this feature makes**, recorded in `design.md` D13 under D.11's
permission to refine internal item boundaries: enforcement and the endpoint enumeration ship with
`TS-BL-018` rather than as a separate item; audit retention ships with `TS-BL-020`; `TS-BL-021`
depends on `TS-BL-018` as well as `TS-BL-020`, an edge D.9's table does not carry; and `TS-BL-022`
seeds the page catalog as well as the matrix values.

**Revised 2026-08-27 — reporting page-catalog granularity.** `insight-and-reporting`'s `design.md`
D4 found that §14.2's single `Reports` screen cannot hold the Auditor posture this feature's own
seed requires. `TS-BL-022` now splits it into operational and governance reporting entries
(tasks 5.1a, 5.7, 5.7a, 5.14; `design.md` D5). This cleared the third row of
`exploration-notes.md`'s **Pending cross-feature obligations** table, which gated
`/opsx:apply` on `TS-BL-071` and `TS-BL-074`.

---

## 1. TS-BL-018 — Permission evaluator engine

**Goal:** one evaluator that every page, endpoint and background actor consults, denying by default,
losing to no explicit denial, and answering fresh on every request — installed behind the
resolution port `identity-and-access` already ships so that no consumer changes. Covers
`access-control/authorization`.

```yaml
backlog_items:
  - id: TS-BL-018
    feature: access-control-and-admin
    depends_on: [TS-BL-014]
    status: not-started
```

- [ ] 1.1 Read `identity-and-access`'s `identity/user-model` before writing any interface, and
      record which of its `PermissionResolver` scenarios this item must satisfy — in particular
      that installing the evaluator modifies no consumer (design D3)
- [ ] 1.2 Create `page_permissions`, `role_permissions` and `user_permission_overrides` by
      reversible migration, with grant/deny/unset states, mandatory reason on overrides, and
      foreign keys into `roles` — adding no column to `roles`, `user_roles` or `users` (design D3)
- [ ] 1.3 Implement the evaluation pass resolving `(actor, page, action, scope) → verdict`,
      producing the explanation data in the same pass so `TS-BL-019` reconstructs nothing
- [ ] 1.4 Implement the nine action flags per page, asserting `Run AI` and `Approve` are
      independently controllable from `View` and `Edit`
- [ ] 1.5 Implement unset as "no opinion": it neither grants nor denies, and loses to a grant on
      another of the user's roles
- [ ] 1.6 Implement deny-wins precedence at both role and user level, and assert a direct denial
      cannot be defeated by adding a role or a direct grant
- [ ] 1.7 Implement the scope-predicate vocabulary (`all`, `assigned-postings`, `own-assignments`)
      carried on grants, with writes assignment-scoped and reads matrix-governed (design D4)
- [ ] 1.8 Return a scope filter for list operations and route every list endpoint through a helper
      taking that filter as a required argument, so omitting it is visible rather than silent
- [ ] 1.9 Assert by test that no resolved verdict is persisted into any derived record, index or
      cache, and that stored authorization metadata carries decision *inputs* only — the property
      `matching-and-ranking`'s `TS-BL-050` depends on, and Interaction A's whole point (design D4)
- [ ] 1.10 Implement the declarative per-endpoint permission requirement, with an endpoint
      declaring nothing denying every request
- [ ] 1.11 Add the endpoint enumeration check to CI and startup, failing the build on any
      unclassified route — the "assert a negative" gate shape, not a convention (design D9)
- [ ] 1.12 **Classify Sprint 0's four routes rather than skipping them**: `/health` and `/build`
      public; each diagnostics route permission-gated on a System Administrator `Administer` grant
      *in addition to* its existing non-production restriction. `platform-core`'s `tasks.md` and
      `KNOWN_ISSUES.md` both hand this obligation here (design D9)
- [ ] 1.13 Enforce the six routes `identity-and-access`'s D7 already declared, asserting no edit to
      those declarations was needed
- [ ] 1.14 Implement service-account authorization as a first-class actor, with `Approve`
      structurally unassignable rather than merely unassigned — the authorization half of the
      advisory-only guarantee
- [ ] 1.15 Assert by test that permission, override and role changes take effect on the next
      request with no cached verdict and no re-authentication, including the administrative surface
- [ ] 1.16 Assert by test that there is exactly one evaluator: enumerate every access decision in
      the codebase and confirm none derives from role assignments directly, and that no
      administrative bypass path exists
- [ ] 1.17 Install the evaluator as what the existing `PermissionResolver` port returns, asserting
      the `/api/auth/me` response shape is unchanged and only its values differ (design D3)

---

## 2. TS-BL-019 — Permission explanation / introspection endpoint

**Goal:** an administrator facing an unexpected denial gets the reason, from the same pass that
produced the verdict, and a user can see their own effective permissions and nobody else's. Covers
`access-control/permission-explanation`.

```yaml
backlog_items:
  - id: TS-BL-019
    feature: access-control-and-admin
    depends_on: [TS-BL-018]
    status: not-started
```

- [ ] 2.1 Implement the explanation endpoint for a given subject, page and action, naming the
      deciding factor — which denial overrode which grant, and which roles were consulted
- [ ] 2.2 Distinguish an explicit denial from an absence of any grant in the response, since they
      send an administrator to different cells
- [ ] 2.3 Name the scope predicate where it decided or narrowed the verdict, including on a
      positive verdict whose scope excludes the resource — the case most often misread, because the
      matrix cell shows a grant
- [ ] 2.4 Assert by test that the explanation always agrees with the verdict the same request would
      receive, and that no explanation is produced by a code path that did not also produce a
      verdict (design D1)
- [ ] 2.5 Gate the explanation surface on an administrative permission, classified declaratively
      like any other endpoint
- [ ] 2.6 Assert the explanation payload names roles, grants, denials and scopes only, and carries
      no candidate personal data even when explaining a candidate-data page
- [ ] 2.7 Implement self-introspection: a user retrieves their own effective permissions without an
      administrative grant, and is denied another user's

---

## 3. TS-BL-020 — Audit log write substrate

**Goal:** the audit trail stops being a log line — a durable, append-only, transaction-coupled
writer installed behind the port that already exists, with the append-only grant finally proven
against a real database. Covers `platform/audit-trail`.

```yaml
backlog_items:
  - id: TS-BL-020
    feature: access-control-and-admin
    depends_on: [TS-BL-002, TS-BL-017]
    status: not-started
```

- [ ] 3.1 **Confirm `platform-core`'s task 2.9 Cloud SQL integration harness exists before starting
      this item.** It was built specifically so this item can prove the `audit_logs` narrowing, and
      task 3.11 below is unachievable without it. If it does not exist, that is a scheduling
      conversation, not something to work around with `docker-compose` (design D7)
- [ ] 3.2 Create `audit_logs` by reversible migration with the `reference/spec.md` §12.2 field set —
      actor user, actor service account, action, target type and id, previous and new value, reason,
      correlation id, source address, immutable timestamp
- [ ] 3.3 Implement the durable `AuditSink` and register it as what `get_audit_sink()` returns,
      **without editing `port.py`, redefining `AuditEvent`, or moving a call site** (design D8)
- [ ] 3.4 Write the audit record in the same transaction as its change, and assert that a failed
      audit write fails the operation rather than letting the change proceed
- [ ] 3.5 Prove the substitution against the events already in flight: `identity-and-access`'s
      authentication vocabulary — including a failed login attributed to the authentication
      subsystem with the attempted identity as *target* — and the runtime-configuration change path,
      each landing in `audit_logs` unchanged
- [ ] 3.6 Assert `durable: false` no longer appears for those events, by showing events that
      previously reached the structured log now reach the table under the same correlation
      identifier
- [ ] 3.7 Implement the write-time redaction policy as a declared sensitivity registry, storing
      references or redaction markers instead of personal data. **Do not reuse the Sprint 0
      structlog redaction key list** — it protects a different sink and reusing it unreviewed is the
      two-independently-correct-implementations defect `platform-core`'s D4 warns about (design D6)
- [ ] 3.8 Default an unclassified field to sensitive: referenced rather than stored verbatim, so
      the failure direction is over-redaction, which is recoverable
- [ ] 3.9 Assert by test that reading `audit_logs` **directly, bypassing every application
      surface**, yields no candidate personal data — the property that makes this a schema
      constraint rather than a display filter
- [ ] 3.10 Enforce mandatory reasons where the acting workflow requires one, rejecting the action
      with a field-level error and writing no audit record
- [ ] 3.11 **Prove the append-only narrowing on a real Cloud SQL instance** using task 3.1's
      harness: the runtime identity's `UPDATE`, `DELETE` and `TRUNCATE` on `audit_logs` all refused.
      This closes the `KNOWN_ISSUES.md` entry that has stood since Sprint 0 (design D7)
- [ ] 3.12 Assert no application surface offers update or delete of an audit record, including to
      an administrator
- [ ] 3.13 Implement the configured retention policy with disposal executed as the **schema-owning
      identity, never the runtime identity**, and record each disposal — so the runtime grant stays
      absolute rather than acquiring an exception (design D7)
- [ ] 3.14 Make the retention period changeable through the audited runtime-configuration path
      rather than by deployment

---

## 4. TS-BL-021 — Audit log query surface

**Goal:** an Auditor can investigate what happened to a record, follow one correlation identifier
across audit and AI activity, and export under permission — without the trail ever showing them a
candidate's name. Covers `access-control/audit-review`.

```yaml
backlog_items:
  - id: TS-BL-021
    feature: access-control-and-admin
    depends_on: [TS-BL-020, TS-BL-018]
    status: not-started
```

**`TS-BL-018` is a dependency this feature adds.** D.9's table lists only `TS-BL-020`, but an
Auditor-role-gated surface with permission-gated export cannot be built or tested without an
evaluator — Sprint 0's own handover flagged exactly this shape. A within-feature edge, refined here
per D.11 (design D13).

- [ ] 4.1 Implement audit search by actor, target type, target identifier, action, module and time
      range, returning results in chronological order with their correlation identifiers
- [ ] 4.2 Gate the search surface and its endpoints on an audit View permission through the
      evaluator, denying a direct endpoint call as firmly as a screen load
- [ ] 4.3 Assert the Auditor role resolves to denied for Create, Edit, Delete, Approve and Assign on
      every recruitment resource
- [ ] 4.4 Present records as stored: identifiers, not resolved names — and assert the review surface
      never resolves an identifier into candidate personal data on the reader's behalf (design D6)
- [ ] 4.5 Make following a reference out of an audit record require permission on the referenced
      record, evaluated on its own terms
- [ ] 4.6 Implement retrieval by correlation identifier spanning audit and AI run records, including
      across an asynchronous boundary where a background job performed the write
- [ ] 4.7 Implement export gated on Export permission, carrying a data classification label, and
      generating its own audit event recording who exported what, when, and under which filter
- [ ] 4.8 Assert exported content carries no candidate personal data, exactly as the stored records
      do
- [ ] 4.9 Index every filter combination the surface exposes and assert an unindexed combination
      fails the test suite rather than degrading silently — audit volume grows with time rather than
      with load, so this screen's worst case arrives on its own
- [ ] 4.10 Build the screen on `design-system`'s dense-data-table pattern rather than a new table

---

## 5. TS-BL-022 — Seed data: page catalog and the 9-role permission matrix

**Goal:** a freshly deployed environment has a complete, reviewable permission catalog and a seeded
posture that is a recorded product decision — including the deliberate read widening without which
cross-pool resurfacing silently returns nothing. Covers `access-control/permission-seed`.

```yaml
backlog_items:
  - id: TS-BL-022
    feature: access-control-and-admin
    depends_on: [TS-BL-018]
    status: not-started
```

**Scope note: this item seeds the catalog as well as the matrix.** Its D.9 title says "9-role
permission matrix"; a matrix has no cells without a catalog of pages and actions to be cells *of*,
and the inherited decision seeds that catalog product-wide. One migration, one auditable seed state
(design D5, D13).

- [ ] 5.1 Seed a catalog entry for every screen in `reference/spec.md` §14.2's inventory across
      every applicable action flag, **including screens whose routes arrive in Phase 3** — unrouted
      pages grant nothing, and per-phase seeding would make the matrix a moving target (design D5)
- [ ] 5.1a Split any §14.2 screen whose capabilities are required by two roles with differing
      postures into one entry per posture. **Today that is exactly one screen: `Reports`**, which
      seeds as *operational reporting* (`insight-and-reporting`'s `TS-BL-071`/`072`/`073`) and
      *governance reporting* (`TS-BL-074`). One key cannot hold both postures and no override can
      rescue it, since a per-user deny lands on the whole key (design D5;
      `insight-and-reporting` design D4)
- [ ] 5.2 Attach permission assignments to the nine existing role records, creating, renaming and
      altering none of them — the split at the join table `identity-and-access`'s D4 drew (design D3)
- [ ] 5.3 Protect the nine seeded roles from deletion while leaving their permission assignments
      editable
- [ ] 5.4 Seed reads **open within role** for Recruiter, Practice Manager and Recruitment Manager
      across the candidate and posting pools, and verify a freshly seeded Recruiter can see
      candidates outside their assigned postings — the property Interaction B shows resurfacing
      depends on
- [ ] 5.5 Seed writes assignment-scoped for Recruiters on posting-bound resources, and verify an
      unassigned-posting write is denied despite the open read scope
- [ ] 5.6 Seed Interviewer and Hiring Panel Member as `own-assignments` only, with no Export and no
      Approve
- [ ] 5.7 Seed the Auditor read-only on audit and AI run records, with no Create, Edit, Delete,
      Approve or Assign anywhere — **plus View on the governance-reporting entry**, which is what
      makes `insight-and-reporting`'s `TS-BL-074` reachable by the only role it is built for. Its
      content is AI run metadata over `D16`'s input *references*, so the grant carries no candidate
      personal data (design D5)
- [ ] 5.7a Leave the Auditor's operational-reporting cell **unset, not an explicit deny**.
      Deny-by-default already denies it for an Auditor-only user, while an explicit deny would cross
      roles under `AUTHZ-005` and silently strip operational reporting from a user who also holds
      Practice Manager. Assert the posture rather than the cell state (design D5)
- [ ] 5.8 Seed both Administrator roles configuration-only, with **no View on candidate personal
      data** (`D16`)
- [ ] 5.9 Seed the AI Service Account with only what its background work requires, and assert
      `Approve` is rejected by both the matrix path and the override path
- [ ] 5.10 Record the seeded posture as an auditable configuration state in the audit trail rather
      than a silent script effect, and route it to the business owner for sign-off — the seeded
      values are a product decision, not an implementation default (design D5)
- [ ] 5.11 Make the seed idempotent: re-running duplicates no catalog entry, assignment or user
- [ ] 5.12 Assert re-running the seed **preserves an administrator's later change** to a seeded
      value rather than resetting it
- [ ] 5.13 Seed at least one usable administrative user so a fresh environment is administrable at
      all
- [ ] 5.14 Assert the reporting posture by test, both directions: an Auditor-only user reaches
      governance reporting and is denied operational reporting; a user holding Auditor **and**
      Practice Manager still reaches operational reporting through the Practice Manager grant; and a
      grant on either reporting key confers nothing on the other

---

## 6. TS-BL-023 — Admin Cockpit UI shell

**Goal:** an administrator can see and change who has access to what, in one place, inside the
shared application shell — and nobody else can see that place exists. Covers
`access-control/admin-cockpit`.

```yaml
backlog_items:
  - id: TS-BL-023
    feature: access-control-and-admin
    depends_on: [TS-BL-022, TS-BL-008]
    status: not-started
```

**The dependency is `design-system`'s `TS-BL-008` (Authenticated Shell), not `TS-BL-009` (the dense
data table).** D.9 made that correction deliberately; the table is consumed inside individual
screens rather than being a top-level enabler (design D12).

- [ ] 6.1 Read `design-system`'s `design-system/app-shell` spec before building, and record which of
      its contracts the Cockpit must satisfy — page header, page templates, navigation
      non-evaluation, affordances default-absent
- [ ] 6.2 Build the Cockpit area inside the authenticated shell, defining no parallel layout, no
      second sidebar, and no raw color or spacing value
- [ ] 6.3 Supply the shell with an **already-filtered** navigation list built from evaluator
      verdicts, since the sidebar evaluates nothing itself and renders empty when given nothing
      (design D12)
- [ ] 6.4 Supply explicit affirmative decisions for every permission-governed affordance, and assert
      a screen omitting a decision renders no control
- [ ] 6.5 Gate every Cockpit route and endpoint on an administrative permission through the same
      evaluator as any other surface, asserting no administrator bypass path exists
- [ ] 6.6 Assert a direct Cockpit endpoint call by a non-administrator is refused by the server even
      though no navigation entry was ever rendered — the hidden entry is presentation, not the
      control
- [ ] 6.7 Build user administration: create, activate, deactivate, update and assign roles, each
      audited, driving `identity-and-access`'s activation states rather than reimplementing them
- [ ] 6.8 Wire deactivation from the Cockpit to session revocation, so neither outcome is observable
      without the other
- [ ] 6.9 Build role administration: create, update, deactivate and assign permissions, rejecting a
      duplicate role name with a field-level error, and creating a new role with nothing granted
- [ ] 6.10 Build the matrix screen on the dense-data-table pattern: users or roles as rows, modules,
      pages or actions as columns, cells settable to grant, deny or unset
- [ ] 6.11 Implement matrix filtering by user, role, module, practice and active status
- [ ] 6.12 Implement atomic bulk matrix update where every changed cell produces its own audit
      record, and assert one invalid cell applies none of the batch
- [ ] 6.13 Render an explicit denial as visually distinct from merely ungranted, since they mean
      different things and are fixed differently
- [ ] 6.14 Build the explanation panel, reachable from a selected matrix cell without leaving the
      matrix, consuming `TS-BL-019`'s endpoint rather than recomputing anything
- [ ] 6.15 Implement search, filtering, sorting and pagination on administrative listings, with
      export rendered only under Export permission, labelled with its classification, and audited

---

## 7. TS-BL-024 — UserPermissionOverride grant/deny flow

**Goal:** the exception mechanism for what a role cannot express — always reasoned, always audited,
immediate, and incapable of granting what the model structurally prohibits. Covers
`access-control/permission-overrides`.

```yaml
backlog_items:
  - id: TS-BL-024
    feature: access-control-and-admin
    depends_on: [TS-BL-023]
    status: not-started
```

- [ ] 7.1 Implement creating, changing and removing a direct per-user grant or denial from the
      Cockpit, against the `user_permission_overrides` table `TS-BL-018` created
- [ ] 7.2 Require a reason, rejecting a missing one with a field-level validation error and creating
      no override
- [ ] 7.3 Surface an existing override's reason, creator and creation time wherever it is inspected
- [ ] 7.4 Assert an override survives a role reassignment and continues to apply until explicitly
      removed
- [ ] 7.5 Assert removing an override reverts the user to what their roles alone produce, on the
      next request
- [ ] 7.6 Assert a direct denial cannot be defeated by any ordering of edits — not by adding a
      direct grant, not by assigning a granting role
- [ ] 7.7 Audit every override change with actor, subject, previous value, new value, timestamp,
      reason and affected page and action, and assert a failed audit write fails the change
- [ ] 7.8 Assert an override change bites on the subject's next request with no re-authentication
      and no stale-verdict interval — the incident case, where a delay is least acceptable
- [ ] 7.9 Reject an override attempting to grant a structurally prohibited permission, and enumerate
      those prohibitions so both the matrix path and the override path reject the same set

---

## 8. TS-BL-025 — Break-glass request / approval / expiry flow

**Goal:** an Administrator can reach candidate personal data for a genuine support case, briefly,
with a reason, with someone else told — and the access ends on its own. Covers
`access-control/break-glass`.

```yaml
backlog_items:
  - id: TS-BL-025
    feature: access-control-and-admin
    depends_on: [TS-BL-024]
    status: not-started
```

**Why this depends on `TS-BL-024` rather than standing alone:** break-glass *is* the override
mechanism, with an expiry and a stricter reason. Building it as anything else would create the
second authorization path this feature exists to prevent (design D10).

- [ ] 8.1 Assert the seeded Administrator posture holds: candidate personal data denied without an
      active elevation, configuration surfaces granted without one
- [ ] 8.2 Implement elevation as a user-level override carrying an expiry, evaluated by the same
      evaluator and explained by the same explanation surface as any other grant
- [ ] 8.3 Assert by enumeration that no parallel elevated mode or alternative authorization path
      exists — the property that makes break-glass cheap rather than a second system (design D10)
- [ ] 8.4 Require a typed reason, rejecting a request without one and creating no elevation
- [ ] 8.5 Require a bounded duration, rejecting an unbounded or absent one, and make the default
      duration changeable through the audited runtime-configuration path
- [ ] 8.6 Implement automatic expiry: elevated access ceases on the next request once the duration
      elapses, with no administrator action
- [ ] 8.7 Notify the configured recipients on grant — provisionally all other Application
      Administrators plus the Auditor role — through `platform-core`'s notification engine,
      retaining and surfacing a delivery failure rather than discarding it
- [ ] 8.8 Make the recipient set configurable without deployment
- [ ] 8.9 Scope the elevation to the permissions its request names, denying anything broader, and
      assert `Approve` remains denied to an elevated administrator
- [ ] 8.10 Implement early revocation of an active elevation, effective on the subject's next
      request, and idempotent when the elevation has already expired

---

## 9. TS-BL-026 — Break-glass → audit trail integration

**Goal:** the elevation trail answers what was actually accessed, not only who was permitted —
which is the whole reason `D16` rejected after-the-fact detection as the only control. Covers
`access-control/elevation-audit`.

```yaml
backlog_items:
  - id: TS-BL-026
    feature: access-control-and-admin
    depends_on: [TS-BL-025, TS-BL-020]
    status: not-started
```

- [ ] 9.1 Audit each stage of the elevation lifecycle as its own event: request, grant, early
      revocation, and **expiry recorded as an event rather than inferred from a timestamp passing**
      (design D10)
- [ ] 9.2 Record a refused elevation request, with no elevation created
- [ ] 9.3 Record every access to candidate personal data performed while an elevation is active,
      **including reads**, attributed to that elevation — one of the two audited read exceptions
- [ ] 9.4 Assert ordinary permitted reads produce no audit record, so the exception stays an
      exception and audit volume stays investigable
- [ ] 9.5 Assert an access attempted after expiry is denied and attributed to no elevation
- [ ] 9.6 Share one correlation identifier across an elevation and every access under it, and assert
      an investigator holding it retrieves the grant, the accesses and the ending together
- [ ] 9.7 Make the authorizing elevation retrievable from any single elevated-access record, the
      reverse direction
- [ ] 9.8 Assert elevation records name accessed records by identifier and contain none of the
      personal data that was viewed — recording elevated access in a form that copies that data into
      the table the Auditor reads would defeat the purpose exactly
- [ ] 9.9 Build the elevation review surface: retrieve elevations by administrator, subject, reason
      and time range with the accesses under each, gated on audit View permission — so a periodic
      review can ask "has this administrator been elevating unusually often"

---

## 10. TS-BL-080 — Ownership reassignment on deactivation

**Goal:** a departing user's twelve open postings go to a named successor in the same sitting as
their deactivation, nothing is deleted or orphaned, and no historical record changes who did it.
Covers `access-control/ownership-reassignment`.

```yaml
backlog_items:
  - id: TS-BL-080
    feature: access-control-and-admin
    depends_on: [TS-BL-016, TS-BL-023]
    status: not-started
```

**This item exists because `exploration-notes.md` D.10.1 found it missing.** `D08` and `G-11` each
asserted ownership reassignment was needed work; neither was ever decomposed, while D.10 and D.11
declared the decomposition complete. It lives here rather than in `identity-and-access` because
deactivation is performed from the Admin Cockpit (`ADM-001`) and reassignment is the
administrator's action in the same sitting.

- [ ] 10.1 Read `identity-and-access`'s `identity/user-activation` spec first and record the
      invariant this workflow must satisfy — deactivation deletes nothing, orphans nothing,
      re-attributes nothing, and leaves owned work reassignable (design D11)
- [ ] 10.2 Build the declared registry of ownership relations, starting with assigned postings, open
      tasks and pending approvals, so a Phase 2 or 3 feature adds its relation by registration
      rather than by editing this workflow
- [ ] 10.3 Raise on an unregistered ownership relation rather than skipping it — a silently skipped
      relation is orphaned work nobody notices (design D11)
- [ ] 10.4 Implement transfer of owned work to a named successor, recording the departing user, the
      successor and each item transferred
- [ ] 10.5 Support partial reassignment, leaving untransferred items still naming the original owner
- [ ] 10.6 Assert by test that **no historical actor attribution changes** — not on audit records,
      shortlist reasons, interview notes or approvals — and that no existing audit record is
      modified
- [ ] 10.7 Keep deactivation and reassignment separately committed: deactivation succeeds with no
      successor chosen, and the sessions are revoked regardless. Coupling them would leave the
      leaver's session live while someone picks a successor, which is the `G-11` audit finding the
      pairing exists to close (design D11)
- [ ] 10.8 Support reassigning the work of a user deactivated earlier, without reactivating them
- [ ] 10.9 Offer the reassignment step inside the Cockpit's deactivation flow with the owned work
      enumerated, so the default path is the one that reassigns
- [ ] 10.10 Validate the successor: refuse a deactivated user, refuse no successor, and refuse a
      successor lacking the permissions the transferred work requires — naming what they would need
- [ ] 10.11 Require a reason and audit the reassignment with actor, departing user, successor,
      reason and each item transferred, asserting a failed audit write transfers nothing
- [ ] 10.12 Assert deletion of a user remains refused **even after all their work is reassigned**,
      because historical attribution is the actual reason deletion is refused

---

## Cross-feature seams and obligations

Not tasks — commitments this feature makes to others, and obligations others have handed here.
Listed so none is treated as internal detail and quietly changed.

| Seam or obligation | Direction | Counterpart |
|---|---|---|
| The evaluator installed behind the existing `PermissionResolver` port, no consumer modified | this feature owes | `identity-and-access` `TS-BL-013` |
| The durable `AuditSink` behind the existing `audit/port.py`, no call site moved | this feature owes | `platform-core` `TS-BL-004`, `identity-and-access` `TS-BL-017` |
| Query-time authorization, verdicts never baked into stored data | this feature owes | `matching-and-ranking` `TS-BL-050` |
| Transition permissions enforced through the evaluator | this feature owes | `platform-core` task 4.9 |
| Background actors evaluated as actors | this feature owes | `platform-core` task 6.12 |
| Task visibility enforced through the evaluator | this feature owes | `platform-core` task 5.10 |
| `audit_logs` searchable and correlated with AI runs | this feature owes | `insight-and-reporting` `TS-BL-074` |
| Separate operational- and governance-reporting page keys, with the Auditor granted the second | this feature owes → discharged in 5.1a, 5.7, 5.7a, 5.14 | `insight-and-reporting` `TS-BL-071`, `TS-BL-074` |
| Classify Sprint 0's four unclassified endpoints | owed to this feature → discharged in 1.12 | `platform-core` `TS-BL-003` |
| Cloud SQL integration harness for the append-only proof | owed to this feature → consumed in 3.1, 3.11 | `platform-core` task 2.9 |
| Two database identities making the append-only grant enforceable | owed to this feature | `platform-core` `TS-BL-002` |
| Authentication event vocabulary as the writer's first input | owed to this feature | `identity-and-access` `TS-BL-017` |
| The nine `roles` rows the matrix attaches to | owed to this feature | `identity-and-access` `TS-BL-013` |
| The deactivation ownership invariant | owed to this feature | `identity-and-access` `TS-BL-016` |
| Authenticated Shell, and a sidebar that evaluates nothing itself | owed to this feature | `design-system` `TS-BL-008` |

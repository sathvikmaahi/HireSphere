# Access Control and Admin

## Why

`identity-and-access` decides *who someone is*. This feature decides *what they may do* — and
records that every material thing anyone did actually happened. It is the load-bearing half of
TalentSphere's governance posture: one permission evaluator that every page, endpoint and
background actor consults, and one append-only audit trail that a compliance reviewer can read
without ever seeing candidate personal data.

**Why now, and why this feature is on the critical path for almost everything else.** Three
features already declare seams that resolve to items in here and to nothing else:

- `identity-and-access` ships a `PermissionResolver` port backed by a `NoGrantsResolver` that
  returns an empty set. Until `TS-BL-018` installs the real evaluator behind it, **every activated
  user in the product is powerless by construction** — correct interim behaviour that two other
  features specify, but not a state anything can ship on.
- `platform-core` ships `app/audit/port.py` with a `LoggingAuditSink` recording `durable: false`.
  Runtime-configuration changes and, from `TS-BL-017`, every authentication event are being
  emitted through it today with nowhere durable to land. `TS-BL-020` is the writer.
- `platform-core`'s `TS-BL-004` writes a workflow transition and its audit record in one
  transaction *against that same port*, so the audit trail's most important invariant — a failed
  audit write fails the operation — is currently enforced against a log line.

Two things Sprint 0 built for this feature specifically are also still unproven. The
`INSERT`/`SELECT`-only narrowing on `audit_logs` sits in `backend/sql/01_least_privilege.sql` and
**has never run against a real table**, because the table does not exist. And four Sprint 0
endpoints (`/health`, `/build`, two diagnostics routes) carry no declarative permission
requirement, because the mechanism arrives here.

## What Changes

Ten backlog items: `TS-BL-018` through `TS-BL-026` from
[D.9](../talentsphere/exploration-notes.md), plus `TS-BL-080` from
[D.10.1](../talentsphere/exploration-notes.md). **None is built.** Sprint 0 shipped platform-layer
code only; the permission model was Sprint 6-10 scope in the superseded plan and none of those
sprints ran.

- **`TS-BL-018` Permission evaluator engine.** One evaluator resolving
  `(actor, page, action, scope) → verdict`, deny-by-default (`AUTHZ-001`), nine action flags
  including `Run AI` (§9.3), grant/deny/unset cells (`ADM-005`), explicit denial beating every
  grant (`AUTHZ-005`), server-side on every page and endpoint (`AUTHZ-003`, `AUTHZ-004`),
  uncached so a revocation bites on the next request. Creates `page_permissions`,
  `role_permissions` and `user_permission_overrides` — the three tables `identity-and-access`'s
  D4 deliberately left unclaimed. **Carries the endpoint-classification obligation**
  `platform-core` recorded: the enumeration must classify Sprint 0's four routes, not skip them.
  Depends on `TS-BL-014`.
- **`TS-BL-019` Explanation / introspection endpoint.** "Why can/can't I" (`AUTHZ-006`,
  `ADM-006`), produced by the same evaluation pass as the verdict rather than reconstructed by a
  second code path.
- **`TS-BL-020` Audit log write substrate.** The durable `audit_logs` writer, installed **behind
  `platform-core`'s existing port with no call-site changes**; write-time redaction so records
  carry references and diffs but never candidate personal data (`D16`, `D6`); append-only
  enforced by the database grant, **proven against a real Cloud SQL instance** using the harness
  `platform-core`'s task 2.9 exists to provide. Depends on `TS-BL-002` and `TS-BL-017`.
- **`TS-BL-021` Audit log query UI.** Search by actor, target, action, module and time range for
  the Auditor role, with permission-gated, classification-labelled, self-auditing export
  (`SEC-012`, `PRV-007`).
- **`TS-BL-022` Seed data: the 9-role permission matrix.** The page/action catalog for the whole
  product and the seeded matrix values — which, per
  [Interaction B](../talentsphere/exploration-notes.md), are **a product decision rather than an
  implementation default**, because a literally least-privileged read scope silently breaks
  cross-pool resurfacing.
- **`TS-BL-023` Admin Cockpit UI shell.** User administration, role administration, the matrix
  surface and the explanation panel (`ADM-001`–`ADM-008`), built on `design-system`'s
  **Authenticated Shell (`TS-BL-008`)**.
- **`TS-BL-024` `UserPermissionOverride` grant/deny flow.** Direct per-user grants and denials
  with a mandatory reason, independent of role membership and surviving role reassignment.
- **`TS-BL-025` Break-glass request/approval/expiry.** An Administrator's time-boxed, reasoned
  elevation to candidate personal data — an ordinary override with an expiry, not a second access
  mode (`D16`, inherited `D5`).
- **`TS-BL-026` Break-glass → audit trail integration.** Elevation grant, use and expiry all land
  in the trail, notified to configured recipients, correlated so an investigator can answer *what
  was actually accessed under this elevation*.
- **`TS-BL-080` Ownership reassignment on deactivation.** A departing user's owned work is handed
  to a named successor in the same administrative sitting as the deactivation, with historical
  actor attribution preserved unchanged. Closes the gap `D.10.1` recorded after `D08` and `G-11`
  each asserted this work was needed and no item ever built it.

**Explicitly not in this change:**

- **No identity.** `users`, `roles` and `user_roles` are `identity-and-access`'s
  (`TS-BL-013`), per that feature's `design.md` D4. This feature attaches to `roles.id` and
  touches none of those three tables.
- **No second evaluator, and no evaluation anywhere but the evaluator.** Including none inlined
  in this feature's own Admin Cockpit.
- **No AI run logging.** `AIRun` records are `ai-platform-governance`'s `TS-BL-028`. They share
  this feature's correlation identifier and its no-raw-personal-data constraint; they are not
  written here.
- **No dashboards or reporting surfaces.** `TS-BL-021` is an investigative query surface for the
  Auditor. The AI Audit Dashboard and the audit/AI-run *reporting* surface are
  `insight-and-reporting`'s `TS-BL-074`, which depends on `TS-BL-021`.
- **No vector-index authorization.** [Interaction A](../talentsphere/exploration-notes.md) forces
  authorization to be evaluated at query time rather than baked into the index; this feature
  supplies the query-time evaluation, and `matching-and-ranking`'s `TS-BL-050` consumes it.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is
new. Two paths are **preserved exactly** from `talentsphere-wave-1-foundation` rather than
renamed, matching what `platform-core` and `identity-and-access` did with theirs.

- `access-control/authorization`: the evaluation model itself — deny-by-default, the nine action
  flags, grant/deny/unset, deny-wins precedence, data-scope predicates, service-account
  authorization, server-side enforcement on every surface, and immediate effect with no cached
  verdict. *(`TS-BL-018`)* — **path preserved**
- `access-control/permission-explanation`: the administrator-facing account of any verdict,
  produced by the same evaluation pass that produced it. *(`TS-BL-019`)*
- `platform/audit-trail`: what an audit record contains, what it may never contain, that it is
  append-only at the database rather than by application convention, and that a failed audit
  write fails its operation. *(`TS-BL-020`)* — **path preserved**
- `access-control/audit-review`: reading the trail — search, chronology, correlation with AI
  activity, and permission-gated export carrying a classification label and its own audit event.
  *(`TS-BL-021`)*
- `access-control/permission-seed`: the product-wide page and action catalog, the nine roles'
  seeded matrix values as a recorded and auditable product decision, and idempotent re-seeding.
  *(`TS-BL-022`)*
- `access-control/admin-cockpit`: the administrative surface — user administration, role
  administration, the matrix view and its filters, the explanation panel, and the rule that
  non-administrators see neither its routes nor its APIs. *(`TS-BL-023`)* — **path preserved**
- `access-control/permission-overrides`: direct per-user grants and denials, their mandatory
  reason, and their independence from role membership. *(`TS-BL-024`)*
- `access-control/break-glass`: time-boxed elevation to candidate personal data — request,
  approval, bounded duration, automatic expiry, and the absence of any parallel elevated mode.
  *(`TS-BL-025`)*
- `access-control/elevation-audit`: the evidence trail an elevation leaves — grant, notification,
  the accesses performed under it, and expiry — correlated so one investigation spans all of it.
  *(`TS-BL-026`)*
- `access-control/ownership-reassignment`: handing a deactivated user's owned work to a named
  successor without deleting, orphaning, or silently re-attributing anything. *(`TS-BL-080`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**Overlap to resolve outside this change.** `talentsphere-wave-1-foundation` remains active and
unarchived, and its `access-control/authorization`, `access-control/admin-cockpit` and
`platform/audit-trail` delta specs currently carry, in three files, requirements that D.9 splits
across ten independently deployable items. `design.md` D2 carries the requirement-by-requirement
redistribution so an auditor comparing the two changes can see where each inherited requirement
landed. Retiring that change spans five features and is not this change's to perform — but **this
change completes the set**: with `access-control-and-admin` proposed, every one of
`talentsphere-wave-1-foundation`'s eighteen decisions and fourteen delta specs has a named
successor across the five Phase-1 features.

## Impact

**New application code** — `backend/app/authz/` (the evaluator, the scope-predicate vocabulary,
the declarative endpoint requirement and its enumeration check), `backend/app/audit/writer.py`
plus the redaction policy (installed behind the existing `port.py`, which is not modified),
`backend/app/api/routes/admin/` (roles, matrix, overrides, elevations, audit search),
`backend/app/api/routes/authz.py` (the explanation endpoint), seed scripts for the page catalog
and matrix, and `frontend/src/` Admin Cockpit screens composed from `design-system`'s
Authenticated Shell and dense data table.

**New tables** — `page_permissions`, `role_permissions`, `user_permission_overrides`,
`audit_logs`, and the elevation and reassignment records. All by reversible migration, applied by
the migration identity, never by hand.

**Existing code consumed, not modified** — `backend/app/audit/port.py` (the writer is substituted
behind it), `identity-and-access`'s `PermissionResolver` port (the evaluator is what it resolves
to), `backend/app/core/correlation.py`, `backend/app/core/errors.py`, and
`backend/sql/01_least_privilege.sql`, whose existing `audit_logs` grant becomes exercisable for
the first time.

**Infrastructure** — no new infrastructure. This feature consumes `platform-core`'s Cloud SQL
integration harness (task 2.9) to prove the append-only grant against a real instance, and adds
its routes to the gateway's generated ingress contract.

**Downstream features that block on this one** — all of them, in practice. `platform-core`'s
`TS-BL-004` (transition permissions) and `TS-BL-006` (background actors evaluated as actors),
`hiring-postings`' first item, `matching-and-ranking`'s Ranking Board and its query-time
authorization, `interview-pipeline`'s note-edit matrix control, and `insight-and-reporting`'s
audit surface all name `TS-BL-018`, `TS-BL-020` or `TS-BL-021` as their enabler.

**External constraints carried, not solved here** — the **seeded matrix values need business
sign-off** (Interaction B, inherited D4). They are seeded as a recorded, auditable configuration
state rather than a silent script effect precisely so the decision is reviewable; changing them
is configuration, not code. The **break-glass notification target** (all other Application
Administrators plus the Auditor role, provisionally) is likewise an owner decision, not a
technical one.

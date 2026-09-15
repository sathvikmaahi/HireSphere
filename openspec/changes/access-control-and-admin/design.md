## Context

See `proposal.md` — Why. The design-relevant state, in one paragraph each:

**The code that exists, and what it means for this feature.** `backend/app/` is platform-layer
only. Three things in it are this feature's foundation rather than its neighbours':
`audit/port.py` defines `AuditEvent` (actor user or service account, action, target type and id,
previous and new value, reason, correlation id, source address, timestamp) and the `AuditSink`
protocol, backed today by `LoggingAuditSink` writing `durable: false`;
`backend/sql/01_least_privilege.sql` **already contains the `INSERT`/`SELECT`-only narrowing on
`audit_logs`**, written in Sprint 0 against a table that does not exist and therefore never once
executed against a real one; and `core/correlation.py` assigns an identifier at the edge that
every audit record and every AI run record must carry. There is no `authz` package, no
`page_permissions`, no `audit_logs`, and no Admin Cockpit route.

**The plan that exists.** `talentsphere-wave-1-foundation` carries three delta specs this feature
inherits — `access-control/authorization`, `access-control/admin-cockpit`, `platform/audit-trail`
— and decisions **D1–D7**, assigned here by `platform-core`'s D11. Its Sprint 6–10 would have
built all of this; none of those sprints ran, so **every decision below is inherited plan, not
inherited code.** The one exception is D7's grant, which was written and shipped early because
`platform-core`'s D3 needed the two-identity split to have a reason.

**The constraint that shapes almost everything.** This feature is consumed through two ports that
already exist and must not change shape: `identity-and-access`'s `PermissionResolver` (declared,
returning no grants) and `platform-core`'s `AuditSink` (declared, writing to a log). Both were
built specifically so this feature substitutes an implementation rather than retrofitting callers.
Designing a third interface here would waste both.

**Reconciliation, stated plainly.** Sprint 0 built no permission logic, no audit table, and no
administrative screen. What it genuinely contributed to this feature is: the audit *port*, the
*unexercised* audit grant, the two database identities that make that grant enforceable, the
correlation identifier, and a structured-log redaction processor that is **not** the audit
redaction policy (it redacts log keys; D6's policy governs stored audit *values* — related
discipline, different mechanism). Everything else here is unbuilt.

## Goals / Non-Goals

**Goals:**

- One evaluator, consulted by every page, endpoint and background actor, whose verdict and
  explanation come from the same pass — so the explanation can never disagree with the decision.
- An audit trail whose append-only property is a database fact rather than an application
  convention, and which is proven so against a real instance rather than a superuser's laptop.
- An administrative surface that configures access without ever becoming a way to read candidate
  data — including through the audit trail it administers.
- Honest reconciliation: every requirement inherited from `talentsphere-wave-1-foundation`'s
  three delta specs lands somewhere traceable, and nothing is quietly dropped or quietly renamed.

**Non-Goals (design-level, beyond the proposal's scope boundary):**

- **No verdict cache, in any form, in this feature.** Not in the session, not in a request-scoped
  memo that outlives one evaluation, not "just for the matrix screen". D1 explains why this is a
  correctness property rather than a performance preference.
- **No second evaluation path for administrators.** The Admin Cockpit is a permission-gated
  surface evaluated by the same evaluator as any other page. An admin-only bypass is the exact
  shape of defect `AUTHZ-004` exists to prevent.
- **No opinion on evaluator latency until something measures it.** `platform-core`'s `TS-BL-001`
  performance harness is the trigger; the matrix screen is the known worst case by design.
- **No audit *reporting*.** Aggregation, dashboards and AI-run correlation surfaces are
  `insight-and-reporting`'s `TS-BL-074`. `TS-BL-021` answers "what happened to this record", not
  "how are we doing".

## Decisions

### D1 — Permission evaluation is centralized, synchronous, and uncached

Inherited from `talentsphere-wave-1-foundation`'s **D1**, unchanged. A single evaluator resolves
`(actor, page, action, resource scope) → verdict + explanation`. Every endpoint declares its
required permission declaratively; an endpoint declaring nothing denies everything. The
explanation is a by-product of the same evaluation pass, never reconstructed afterwards.

*Why one pass:* `AUTHZ-006` requires an explanation an administrator can act on. An explanation
computed by a second code path will eventually disagree with the verdict, and a confidently wrong
explanation is worse than none — it sends an administrator to change the wrong cell.

*Why uncached, specifically, and why it is not negotiable here:* three separate requirements
converge on it. `AUTHZ-007`/`ADM-007` require permission changes to be effective immediately;
`AUTH-007` requires a revoked session to be invalid on the next request; and
`identity-and-access`'s D6 already made sessions server-side records **for this reason**, paying a
per-request database read to get it. Caching verdicts in the session would hand that property
straight back, on the same request path, for the same user population (`D07`: a few dozen staff
users, <5k candidates). *Alternative considered:* verdict caching with an invalidation channel.
Rejected — the invalidation channel is precisely where this bug class lives, and it buys
performance nothing has yet asked for. A cache can later go **behind** the evaluator interface
without touching a caller, which is the whole point of having the interface.

*Why not framework decorators alone:* decorators cover endpoints and miss background actors. The
AI Service Account must be evaluated by the same rules as a human (`WF-004`,
`platform-core`'s task 6.12), and a decorator on a route cannot reach a Cloud Run Job.

### D2 — Where every inherited requirement lands, stated explicitly

D.9 splits what `talentsphere-wave-1-foundation` treated as three delta specs into ten
independently deployable items. The redistribution, so that "missing" can be distinguished from
"moved" by anyone auditing the two changes side by side:

| Inherited requirement | Inherited path | Lands in | Item |
|---|---|---|---|
| Deny-by-default authorization | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Nine action flags | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Grant, deny, and unset cell states | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Explicit denial overrides grants | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Server-side enforcement on every surface | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Data scope predicates | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Service account authorization | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Immediate effect of authorization changes | `access-control/authorization` | `access-control/authorization` | `TS-BL-018` |
| Permission explanation | `access-control/authorization` | `access-control/permission-explanation` | `TS-BL-019` |
| Per-user permission overrides | `access-control/authorization` | `access-control/permission-overrides` | `TS-BL-024` |
| Seeded role set | `access-control/authorization` | **split** — role rows are `identity-and-access`'s `TS-BL-013`; their protection-from-deletion and their seeded grants are `access-control/permission-seed` | `TS-BL-022` |
| Least-privilege defaults | `access-control/authorization` | `access-control/permission-seed` | `TS-BL-022` |
| Audited permission changes | `access-control/authorization` | `access-control/permission-overrides` + `platform/audit-trail` | `TS-BL-024`, `TS-BL-020` |
| Audit on every material write | `platform/audit-trail` | `platform/audit-trail` | `TS-BL-020` |
| Audit record content | `platform/audit-trail` | `platform/audit-trail` | `TS-BL-020` |
| Audit records contain no candidate personal data | `platform/audit-trail` | `platform/audit-trail` | `TS-BL-020` |
| Mandatory reasons on sensitive changes | `platform/audit-trail` | `platform/audit-trail` | `TS-BL-020` |
| Immutability | `platform/audit-trail` | `platform/audit-trail` | `TS-BL-020` |
| Audit search | `platform/audit-trail` | `access-control/audit-review` | `TS-BL-021` |
| Audited, permission-controlled export | `platform/audit-trail` | `access-control/audit-review` | `TS-BL-021` |
| Correlation with AI activity | `platform/audit-trail` | `access-control/audit-review` | `TS-BL-021` |
| Admin Cockpit restricted to administrators | `access-control/admin-cockpit` | `access-control/admin-cockpit` | `TS-BL-023` |
| User administration | `access-control/admin-cockpit` | `access-control/admin-cockpit` | `TS-BL-023` |
| Role administration | `access-control/admin-cockpit` | `access-control/admin-cockpit` | `TS-BL-023` |
| Permission matrix surface | `access-control/admin-cockpit` | `access-control/admin-cockpit` | `TS-BL-023` |
| Explanation panel | `access-control/admin-cockpit` | `access-control/admin-cockpit` | `TS-BL-023` |
| Administrative table behavior | `access-control/admin-cockpit` | `access-control/admin-cockpit` | `TS-BL-023` |
| Administrators are configuration-only on candidate data | `access-control/admin-cockpit` | `access-control/break-glass` | `TS-BL-025` |
| Seed data | `access-control/admin-cockpit` | `access-control/permission-seed` | `TS-BL-022` |

Three paths are preserved exactly (`access-control/authorization`,
`access-control/admin-cockpit`, `platform/audit-trail`), matching what `platform-core` did with
its three and `identity-and-access` with its two. `platform/audit-trail` keeps its `platform/`
prefix even though this feature owns it: the capability is genuinely platform-level — every
feature writes to it — and `platform-core`'s D11 deliberately did **not** claim that path,
leaving it for whoever inherited D6 and D7. Renaming it would create a parallel path describing
the same behaviour, which is the thing preservation exists to avoid.

**Nothing inherited is dropped.** Unlike `identity-and-access`, which deliberately did not carry a
user-level practice attribute, every requirement in the three inherited specs has a destination
above. One is *split* rather than moved — "Seeded role set" — because `identity-and-access`'s D4
already drew the line through it at the join table.

### D3 — The evaluator is what `PermissionResolver` resolves to, and the split is at the join table

`identity-and-access`'s D4 fixed the boundary precisely: `roles`, `user_roles` and `users` are
its; `page_permissions`, `role_permissions`, `user_permission_overrides`, the matrix values,
deny-wins precedence and the explanation endpoint are entirely this feature's. This design holds
that line exactly — `role_permissions.role_id` is a foreign key into a table this feature never
alters, and `TS-BL-018` adds no column to `users`.

The consumption side is equally fixed. `identity/user-model`'s "Permission resolution is consumed,
never computed here" requirement declares a `PermissionResolver` port with a `NoGrantsResolver`
default and a scenario stating that when the evaluator arrives, **no consumer of the interface
requires modification**. `TS-BL-018` therefore implements the declared interface. It does not
design a better one, and it does not ask `identity-and-access` to adapt.

*Why this matters more than it looks:* the port's default returns an *empty set*, which
`identity-and-access`'s D5 and `design-system`'s app-shell spec both treat as a **correct state**,
not a stub — an activated user with no grants reaches the landing route and sees an empty state.
That means installing the real evaluator is not "turning authorization on"; it is replacing a
resolver whose answer was always "nothing" with one whose answer is computed. Nothing about the
denial path changes, which is why this can ship without a flag day.

*Alternative considered:* have the evaluator return a richer object than the port declares —
verdict plus explanation plus scope filter — and widen the port. Rejected for the port's own
sake: `/api/auth/me` is consumed by the frontend shell from `TS-BL-014` onward, and a response
shape that gains a field later is a breaking change to every consumer written against it. The
explanation and the scope filter are reached through `TS-BL-019`'s endpoint and D4's filter
contract respectively, not by widening a shape three features already bind to.

### D4 — Scope predicates are declared on grants, and authorization is evaluated at query time

Inherited from `talentsphere-wave-1-foundation`'s **D2**. A grant carries an optional scope
predicate (`all`, `assigned-postings`, `own-assignments`). For list operations the evaluator
returns a verdict *and* a scope filter that the query layer applies; it does not fetch rows and
drop them afterwards, which leaks counts and breaks pagination.

*Why the vocabulary is fixed here while the resources arrive later:* `C-03` resolved write scope
to posting assignment (`BR-005`, `job_postings.recruiter_ids[]`) and read scope to configuration
(`CAN-006`, `AUTHZ-008`). Hard-coding predicates into each query would make that configuration a
lie. Most of the resources being filtered arrive in Phase 2, so this feature defines and enforces
the vocabulary against the resources that exist and leaves the rest to their owning features.

**One thing this decision must get right for a feature that has not been proposed yet.**
[Interaction A](../talentsphere/exploration-notes.md) shows that `C-01`'s adoption of `VEC-003`
(vector retrieval applies the same authorization rules as relational access) plus `C-03`'s
configurable reads means a matrix change would invalidate any authorization verdict baked into a
vector index — forcing a `VEC-005` re-index on every permission edit. The design choice it forces
is **evaluate at query time, never bake a resolved verdict into stored data**, and correspondingly
that `authorization_scope_json` on a vector record carries the *inputs* to a decision (candidate
id, posting id, practice) rather than its outcome. The evaluator's filter contract is therefore
specified as a set of predicate inputs the caller applies, not as a precomputed allow-list of ids.
`matching-and-ranking`'s `TS-BL-050` is the consumer; getting this wrong there is expensive and
getting it wrong here is cheap to avoid.

*Trade-off:* the query layer must consistently apply the returned filter, which no type system
enforces. Mitigated exactly as the inherited decision proposed — every list endpoint routes
through a helper taking the filter as a required argument, so omitting it is a visible mistake
rather than a silent one.

### D5 — The page catalog is seeded product-wide, and the seeded values are a product decision

Inherited from `talentsphere-wave-1-foundation`'s **D3** and **D4**, both unchanged.

Every screen in `reference/spec.md` §14.2's inventory is seeded as page/action permission rows in
`TS-BL-022`, including screens whose routes arrive in Phase 3. Unrouted pages are harmless —
deny-by-default means an unreachable page grants nothing — and the alternative is four more
re-seed migrations, each needing its own audit story, against a matrix that is a moving target
until Phase 5.

**One exception to one-entry-per-screen, and the test for when it applies.** *(Added 2026-08-27,
correcting a gap found by `insight-and-reporting`'s `design.md` D4.)* A §14.2 screen splits into
more than one catalog entry when its listed capabilities are required by two roles whose seeded
postures differ. The reason is mechanical rather than aesthetic: a page key is the unit both the
matrix and `user_permission_overrides` act on, so a single key cannot hold two postures, and no
override can rescue it — a per-user deny lands on the whole key.

**`Reports` is the one screen currently failing that test.** Its §14.2 capabilities span
*"practice dashboard, recruiter workload, candidate stage counts, aging"* — `insight-and-reporting`'s
`TS-BL-071`/`072`/`073` — and *"AI audit"*, which is `TS-BL-074`. Granting the Auditor View to reach
the AI Audit Dashboard would also grant candidate-derived aggregates, contradicting `C-02` and this
feature's own Auditor posture below; withholding it leaves `TS-BL-074` unreachable for the only role
it is built for. It therefore seeds as **two** entries: operational reporting and governance
reporting.

*Why the Auditor may legitimately hold the governance entry:* `TS-BL-074`'s content is AI run
metadata — runs, prompt and model versions, human overrides, failure rates, flagged outputs — and
`D16` already requires `AIRun` to store input *references* rather than raw payloads, precisely so
that AI run logs do not become the PII back door for the roles that read them. The governance entry
therefore carries no candidate personal data, which is what makes this grant consistent with the
config-only-on-candidate-data posture rather than an exception to it.

*Why the Auditor's operational-reporting cell is left **unset** rather than seeded as an explicit
deny* — the obligation as originally worded said "denied", and taking that literally would be a
defect. `AUTHZ-005`'s deny-wins precedence crosses roles: `access-control/authorization` carries the
scenario "a user holds two roles, one granting and one explicitly denying the same action → access
is denied". An explicit deny on the Auditor role would therefore strip operational reporting from a
person who is *also* a Practice Manager — a multi-role case `identity-and-access`'s
`identity/user-model` explicitly contemplates ("a manager who also interviews") — and it would fail
silently, since nothing distinguishes it from an ordinary denial. Deny-by-default already produces
the required outcome for an Auditor-only user at no such cost. **The property to assert is the
posture, not the cell state**: an Auditor-only user reaches governance reporting and not operational
reporting. Recorded here so this is not later "corrected" into an explicit deny. Contrast the AI
Service Account's `Approve`, which *is* absolute and is enforced as structurally unassignable rather
than as a matrix cell at all.

*Why two entries and not three:* `TS-BL-073`'s closure-readiness view is read by the same roles under
the same posture as `TS-BL-071` and `072`, so splitting it further would add matrix surface with no
posture to justify it — against this decision's own moving-target argument. `Audit Center` needs no
split either: §14.2 already lists it as its own screen, and it is `TS-BL-021`'s surface.

The **values** are the part that is not an implementation detail.
[Interaction B](../talentsphere/exploration-notes.md) is explicit: with `AUTHZ-008` applied
literally to reads, a Recruiter cannot see candidates outside their assigned postings, so the
resurfacing suggestion queue — a headline feature — silently returns nothing. The inherited seed
therefore deliberately widens reads:

- **Writes** assignment-scoped for Recruiters on posting-bound resources; least-privilege
  elsewhere.
- **Reads** open within role for Recruiter, Practice Manager and Recruitment Manager across the
  candidate and posting pools — a deliberate widening of the default, recorded as such.
- **Interviewer / Hiring Panel Member** `own-assignments` only, no Export, no Approve.
- **Auditor** read-only on audit and AI run logs; View on the governance-reporting entry and
  nothing on the operational-reporting one; structurally no Create, Edit, Delete, Approve or Assign
  anywhere.
- **Application / System Administrator** configuration only, with **no View on candidate personal
  data** (`D16`).
- **AI Service Account** only what its background jobs require; **Approve is unassignable**, not
  merely unassigned.

*Why open-within-role rather than closed:* both are wrong in one direction. A closed default
breaks a headline feature *quietly* — an empty queue looks like "no matches". An open default is
wrong *loudly* — someone sees a candidate they did not expect to, and says so. Choosing the
failure mode that surfaces itself is the whole argument. It is nonetheless **flagged for owner
sign-off** and recorded in Open Questions, because who sees whose candidates is a business policy,
not an engineering choice, and `TS-BL-022` seeds it as an auditable configuration state rather
than a silent script effect so that reviewing it is possible at all.

### D6 — Audit records carry references and diffs, never personal data, redacted at write time

Inherited from `talentsphere-wave-1-foundation`'s **D6**, unchanged. Audit rows carry actor,
action, target type and id, previous and new values, reason, correlation id and timestamp. Where a
changed field holds personal data, the stored value is a redaction marker or a reference, applied
by policy **at write time**.

*Why this is a schema constraint and not a display filter:* `D16` made Administrators config-only
and `C-02` gave the Auditor read-only access to the trail. If audit rows contained candidate names
or resume text, **both boundaries would be decorative** — the audit table would be the PII back
door, reachable by the two roles specifically designed not to read candidate data. A display
filter fails open the first time someone adds an export, a debug endpoint, or a support query.

*Not to be confused with what Sprint 0 built.* `platform-core` shipped a structlog processor that
redacts sensitive keys from log lines. That is a different mechanism protecting a different sink;
it does not govern stored audit values, and reusing its key list here without review would be the
kind of "two independently-correct implementations of the same rule" defect `platform-core`'s D4
warns about. The redaction policy is defined once, in `TS-BL-020`, and the log processor is left
alone.

*Trade-off, accepted:* an investigator sometimes wants the previous value verbatim. They resolve
the reference, which requires permission on the referenced record — which is the point, not a
gap.

### D7 — Append-only is a database grant, and `TS-BL-020` is where it is finally proven

Inherited from `talentsphere-wave-1-foundation`'s **D7**. Audit and AI run tables are append-only
from the application's perspective: no update or delete surface exists, and the runtime database
identity holds `INSERT` and `SELECT` only on them.

**This is the half of a split that `platform-core` has already built the other half of.** Its D3
records the two-identity separation — runtime holds DML only, a separate migration identity owns
the schema and holds DDL — and states that the split exists *because* this feature's append-only
grant "is decorative if the same identity can `ALTER TABLE`". `platform-core`'s
`platform/database-access` spec carries the scenario ("the runtime identity attempts to update or
delete a row in a table granted `INSERT` and `SELECT` only → the database refuses"); this
feature's `platform/audit-trail` spec carries the requirement that `audit_logs` **is** such a
table. Neither is complete without the other, and the shape is fixed by the built half: the grant
this feature requires must be exactly `INSERT`, `SELECT` for the runtime identity, with the
migration identity as owner. No `UPDATE`, no `DELETE`, and no `TRUNCATE` for the runtime role,
including for retention disposal — see below.

**And it has never run.** `backend/sql/01_least_privilege.sql` already contains this narrowing.
It was verified against a local PostgreSQL where the developer is a superuser, which is precisely
the environment in which a least-privilege narrowing cannot be proven, and Sprint 0 recorded that
honestly in `KNOWN_ISSUES.md`. `platform-core`'s task 2.9 builds a Cloud SQL integration harness
*specifically so this item can prove it*. `TS-BL-020` uses that harness. Proving it against
`docker-compose`'s Postgres would reproduce the exact Sprint 0 failure mode — the grant script was
verified as a real superuser locally, while Cloud SQL gives `cloudsqlsuperuser`, which behaves
differently.

*The one thing this decision has to answer that the inherited version did not:* `RET-004` requires
retention rules on audit logs, and disposal is a `DELETE`. Retention is therefore executed by a
process running as the **migration identity**, not the runtime identity, and disposal is itself
recorded — so the append-only grant on the runtime role stays absolute rather than acquiring an
exception that a bug could reach.

*Alternative considered:* database triggers rejecting `UPDATE`/`DELETE`. Equivalent in effect,
more machinery, and it would let the runtime role keep rights it then declines to use. The grant
is simpler, self-documenting, and removes the capability rather than guarding it — the same
argument `platform-core`'s D2 and D10 make in two other domains.

### D8 — The durable writer is substituted behind the existing port, and the port does not change

`TS-BL-020` implements `AuditSink` with a durable, transactional writer and registers it as what
`get_audit_sink()` returns. `AuditEvent` is not redefined, `port.py` is not edited, and no call
site moves.

*Why this is stated as a decision rather than assumed:* `platform-core`'s D8 built its
workflow-transition audit write against the port **on the strength of this promise** — it is what
lets `TS-BL-004` depend on `TS-BL-002` rather than on `TS-BL-020`, which is what lets the workflow
engine and the audit substrate be built concurrently by different people. Introducing a new
audit-writing interface here would silently invalidate that sequencing and require reworking
`platform-core`'s task 4.7, `identity-and-access`'s `TS-BL-017`, and the runtime-config change
path, all of which already emit through the port.

Two consequences worth naming:

1. **The writer must accept the events that already exist**, not the events it would have
   preferred. `identity-and-access`'s `TS-BL-017` defines the authentication vocabulary — including
   a failed login attributed to the authentication subsystem as a service-account actor with the
   attempted identity as *target* — and the runtime-config path already emits configuration
   changes. `TS-BL-020` depends on `TS-BL-017` in exactly this direction, and the first
   integration test is that vocabulary landing in `audit_logs` unchanged.
2. **`durable: false` disappears as an observable, not as a code change at call sites.** The
   honest interim state described in `KNOWN_ISSUES.md` ends when the sink is swapped; the way to
   verify it is to assert that events previously reaching the log now reach the table, with the
   same correlation identifier.

### D9 — Endpoint classification is declarative and enumerated, and it starts by classifying the four routes that predate it

`AUTHZ-003` and `AUTHZ-004` require server-side enforcement on every page *and* endpoint, with a
direct API call failing even when the UI hides the control. `platform-core`'s
`platform/api-ingress` spec already states the ingress-side obligation — every exposed route
classified as public, authenticated or permission-gated, an endpoint declaring nothing denying
everything — and names this feature as the owner of the mechanism.

`TS-BL-018` therefore ships three things together: a declarative per-endpoint permission
requirement, a default that denies when none is declared, and a **CI/startup enumeration check
that fails on any unclassified route**. The check is the part that lasts; the declaration alone
decays the first time someone adds a route in a hurry, and this is the "assert a negative" gate
shape `platform-core`'s D5 identified as the only kind that caught real Sprint 0 defects.

**The four routes that predate the mechanism are classified, not skipped.** `platform-core`'s
`tasks.md` and `KNOWN_ISSUES.md` both record `/health`, `/build` and two diagnostics routes as
carrying no declarative requirement, and both explicitly hand the obligation here. The
classification: `/health` and `/build` are **public** — they are the pipeline's provenance and
readiness path and must work before any session exists (`platform-core`'s task 3.5 routes the
post-deploy check through the gateway); the two diagnostics routes are **permission-gated on a
System Administrator `Administer` grant**, *in addition to* the existing non-production
environment restriction rather than instead of it. Diagnostics expose configuration state, which
is exactly what `ADM-008` says a non-administrator may not see.

`identity-and-access`'s D7 has already classified its six routes ahead of the mechanism existing,
declaring requirements nothing yet enforces. Those declarations become enforced by this item with
no edit to them — which is the outcome that discipline was for.

### D10 — Break-glass is a time-boxed override, and the elevation must evidence what it was used for

Inherited from `talentsphere-wave-1-foundation`'s **D5**, extended by one requirement the
inherited version left implicit.

Administrator access to candidate personal data is granted as an ordinary user-level permission
override with an expiry, a mandatory typed reason, an audit record and a notification. It expires
on its own, with no administrator action.

*Why reuse the override mechanism:* break-glass then inherits the audit trail, the explanation
endpoint and the evaluator for free. A parallel "elevated mode" would be a second authorization
path — the precise thing D1 exists to prevent — and would need its own explanation story, at which
point an administrator asking "why can this person see this" gets two answers. This is also why
`TS-BL-025` depends on `TS-BL-024` rather than being independent of it: the override flow is the
mechanism, and break-glass is that mechanism with an expiry and a stricter reason.

*Why time-boxed rather than per-request approval:* per-request needs a second human available on
demand, which real support work does not have. Expiry bounds exposure without adding a synchronous
dependency on another person.

**What `TS-BL-026` adds, and why it is a separate item rather than a paragraph in `TS-BL-025`.**
An elevation that is logged only at grant time answers "who was allowed to look" and not "what
they looked at" — and `D16`'s whole argument is that after-the-fact detection is the weak control
that option 2 was rejected for. So the elevation carries a correlation identifier, every access
performed while it is active is attributed to it, and expiry is recorded as its own event rather
than inferred from a timestamp passing. That is a distinct, independently deployable slice: the
elevation is usable without it, and materially less accountable.

*One thing this deliberately does not do:* it does not audit ordinary reads. `platform/audit-trail`
inherits the rule that reads require no audit record **except** the separately audited categories
of export and elevated access. Elevated access is one of the two exceptions; widening it to all
reads would produce a table whose volume makes investigation harder, not easier.

### D11 — `TS-BL-080` builds the workflow; `identity-and-access` keeps the invariant; the boundary between them is the successor

`identity-and-access`'s `identity/user-activation` spec already states the invariant:
deactivation deletes nothing, orphans nothing, re-attributes no historical record, and leaves
owned work reassignable. Its `TS-BL-016` enforces that today, against the tables that exist.
`TS-BL-080` builds the workflow that hands owned work to a **named successor**, and per
[D.10.1](../talentsphere/exploration-notes.md) it belongs here because deactivation is performed
from the Admin Cockpit (`ADM-001`) and reassignment is the administrator's action in the same
sitting. Splitting one screen's behaviour across two features would be the worse outcome.

**The boundary, precisely, refined per D.11 as that section permits:**

- *Attribution is never rewritten.* Reassignment changes who **owns** work going forward. It never
  alters who **did** something historically — the actor on an audit record, a shortlist reason, an
  interview note. The invariant and the workflow agree on this and neither may relax it.
- *Reassignment is a separate, reasoned, audited act — not a side effect of deactivation.*
  Deactivation succeeds whether or not reassignment follows; the two are presented together and
  committed separately. Coupling them would mean a deactivation blocked because no successor was
  chosen yet, and the leaver's session stays live in the meantime — reintroducing the exact
  `G-11` audit finding the pairing exists to close.
- *The successor is named and validated.* An active user who could legitimately hold that work.
  Reassigning to another deactivated user, or to nobody, is refused.
- *What "owned work" means is enumerable, and it grows.* Today: assigned postings
  (`job_postings.recruiter_ids[]`), open tasks, and pending approvals. In Phase 2 and 3 it extends
  to Applications, interview assignments and offer records. The item ships a **declared registry
  of ownership relations**, so a later feature adds its own relation by registration rather than by
  editing this workflow — the same closed-registry discipline `runtime_config` and the feature
  flags already use, and the same reason: an unregistered relation must raise rather than be
  silently skipped, because a silently skipped relation is orphaned work that nobody notices.

*Alternative considered:* reassign in bulk from a separate "unassigned work" queue, decoupled from
deactivation entirely. Rejected — it is the same design that produces the orphaned-work problem in
the first place, since the queue is only visited when someone remembers it exists. Offering the
successor at the moment of deactivation is what makes it the default path.

*Still genuinely open, and not blocking:* the direction of leaver detection — Hubble push,
TalentSphere poll, or discovery on failed login (`D08`, `OD-001`). `TS-BL-080` triggers on a
TalentSphere-side deactivation however that deactivation arrives.

### D12 — The Admin Cockpit needs the shell, and consumes the table without owning it

`TS-BL-023` depends on `design-system`'s **`TS-BL-008` (Authenticated Shell)**, not on
`TS-BL-009` (the dense data table). D.9 made this correction deliberately when the old
decomposition's dependency did not survive translation to the finer grain, and the shell spec
bears it out: the Cockpit is a multi-page administrative area needing sidebar navigation, the page
header pattern, page templates and the permission-aware-affordance contract — all of which are
`design-system/app-shell`. The dense table is consumed *inside* individual Cockpit screens (the
matrix, the user list, the audit search), which is a within-screen composition rather than a
top-level enabler.

Two obligations the shell spec places on this feature, worth naming because they invert the
usual direction:

1. **The sidebar does not evaluate permissions.** `design-system/app-shell` requires that the
   navigation renders the list it is given and evaluates nothing itself; supplied nothing, it
   renders empty rather than defaulting to a full menu. So *this* feature supplies the
   already-filtered navigation list, from evaluator verdicts. `ADM-008`'s "non-admin users shall
   not view Admin Cockpit routes" is satisfied by the server refusing the route, with the hidden
   navigation entry as presentation only.
2. **Affordances default to absent.** The shell renders a permission-governed affordance only on
   an affirmative decision. A Cockpit screen that omits a decision therefore hides the control —
   the fail-closed direction — and the endpoint refuses it regardless (`AUTHZ-004`).

### D13 — Four item-boundary refinements, made under D.11 and recorded rather than assumed

D.11 permits a feature's propose conversation to refine its own items' internal boundaries. Four
refinements, each with the reason it is not simply scope creep:

1. **Enforcement lives in `TS-BL-018`, not in an item of its own.** The superseded plan had a
   separate "Enforcement & Explanation Surface" sprint. D.9's finer grain has no enforcement item,
   and an evaluator whose verdict nothing enforces is not deployable — so the declarative
   requirement, the deny-by-default fallback and the enumeration check ship with the evaluator.
   This is also what `platform-core` and `identity-and-access` already assume when they hand the
   four unclassified routes and their six declared-but-unenforced routes here.
2. **Audit retention lives in `TS-BL-020`.** `RET-004` requires it, no item names it, and it is
   inseparable from D7's grant because disposal is the one `DELETE` the design must account for.
3. **`TS-BL-021` depends on `TS-BL-018` as well as `TS-BL-020`.** D.9's table lists only
   `TS-BL-020`. But `TS-BL-021` is an *Auditor-role-gated* surface with permission-gated export —
   it cannot be built or tested without an evaluator, and Sprint 0's own handover flagged exactly
   this shape ("audited, permission-gated export needs the evaluator, which arrives later"). This
   is a within-feature edge, so it is refined here rather than in `exploration-notes.md`; D.10's
   `TS-BL-053` shows the decomposition adding evaluator edges where it noticed them, and simply
   did not notice this one.
4. **`TS-BL-022` seeds the page catalog, not only the matrix.** Its D.9 title says "9-role
   permission matrix"; a matrix has no cells without a catalog of pages and actions to be cells
   *of*, and D5's inherited decision seeds the catalog product-wide. Both belong to the same item
   because they are one migration and one auditable seed state.

*A fifth refinement was added later, by `/opsx:update` on 2026-08-27 rather than by this propose
conversation: `TS-BL-022` splits §14.2's `Reports` screen into two catalog entries. It is recorded
in D5 rather than here because it corrects a granularity claim D5 makes, and because it originated
in a sibling feature's artifacts (`insight-and-reporting`'s D4) rather than in this feature's own
reasoning.*

**No correction to `exploration-notes.md` was required by this conversation.** The convention
exists (`AGENTS.md`, and D.10.1 and D09 as worked examples) and was checked against: `D16`,
`C-02`, `C-03`, Interaction A, Interaction B, `G-11` and `D08` are mutually consistent on
permissions, audit and administration, and D.10.1 already closed the one real gap in this
feature's territory. The four items above are internal boundaries, which D.11 assigns to this
conversation rather than to the exploration record.

**Two stale references found in `KNOWN_ISSUES.md`, recorded rather than silently corrected.** Its
audit-narrowing entry says "task 2.3 is where it must be proven on Cloud SQL" and its endpoint
entry says "the mechanism arrives in Sprint 7 (tasks 8.1 and 8.2)". Both cite the superseded
per-sprint plan; the real owners are `TS-BL-020` and `TS-BL-018`. That file feeds
`scripts/release_notes.py`, so anyone reading a release note today is pointed at task numbers that
no longer exist. Amending it is a repository edit outside this planning change — the same handling
`identity-and-access` gave the mock-that-does-not-exist line.

## Risks / Trade-offs

- **Uncached evaluation becomes a latency problem as pages get denser**, with the matrix screen
  the worst case by construction → Evaluation sits behind an interface producing verdict and
  explanation together, so a cache can be introduced later without changing a caller (D1).
  `platform-core`'s `TS-BL-001` performance harness is where a real regression would surface;
  until it exists there is no measurement, only opinion.
- **The seeded matrix (D5) is wrong for the business**, and it decides whether cross-pool
  resurfacing works on day one → Seeded as a recorded, auditable configuration state with a named
  rationale rather than buried in a script, so it is reviewable and changeable as configuration.
  The failure mode chosen is the visible one. Still needs owner sign-off (Open Questions).
- **The append-only grant is proven for the first time in this feature, against infrastructure
  that also does not yet exist** — `platform-core`'s task 2.9 harness is itself unbuilt and its
  sibling task 2.8 is *blocked* on an owner decision about schema-ownership automation → Named as
  a real sequencing dependency in the Migration Plan rather than discovered during `TS-BL-020`.
  If the harness slips, the honest fallback is to ship `TS-BL-020` with the narrowing still
  unproven and the `KNOWN_ISSUES.md` entry still standing — which is worse than it sounds, because
  it is the third consecutive release in which an entry saying "never run against a real table"
  remains true.
- **Ten items, and the last four all queue behind `TS-BL-023`**, which itself queues behind
  `TS-BL-022`, `TS-BL-018` and another feature's `TS-BL-008` → This feature is substantially
  sequential and is on the critical path for every Phase 2 feature. Mitigated only by ordering:
  the Migration Plan puts the two items other features are blocked on (`TS-BL-018`, `TS-BL-020`)
  first and refuses to reorder them behind UI work.
- **`TS-BL-020` depends on `TS-BL-017`, which depends on a chain reaching back to an unconfirmed
  external contract** (`OD-001`, Hubble) → The dependency is on the *event vocabulary*, not on
  Hubble itself, and `identity-and-access`'s mock makes that vocabulary real without the contract.
  Residual risk accepted: if `TS-BL-013` slips badly, the audit substrate slips with it, and the
  workflow engine keeps writing to a log meanwhile.
- **The redaction policy is only as good as its classification of which fields hold personal
  data**, and it is written before most of those fields exist → Mitigated by making it a declared
  registry a later feature registers into, and by the deny-direction default: an unclassified
  field in an audited value is treated as sensitive and referenced rather than stored. An
  over-redacted audit record is recoverable; an over-shared one is not.
- **Break-glass is the one designed path from "configuration only" to candidate personal data**,
  and it is used exactly when someone is under time pressure → Bounded by expiry, mandatory typed
  reason, notification to a second party, and `TS-BL-026`'s attribution of every access performed
  under it. The residual exposure is deliberate: `D16` rejected the hard-block option because it
  makes real support work impossible.
- **Ownership reassignment's registry of ownership relations is defined before most relations
  exist** → An unregistered relation raises rather than being skipped (D11), so the failure mode
  is a loud error at reassignment time rather than work quietly left orphaned. The cost is that a
  Phase 2 feature adding an ownership relation must register it, which is a task on that feature
  and is named as a seam.
- **Three active changes now describe `access-control/*` and `platform/audit-trail`** →
  The same accepted interim state `platform-core` and `identity-and-access` recorded. This change
  is authoritative for these ten items from now on; D2's redistribution table exists so the
  overlap is legible rather than confusing. With this feature proposed, retiring
  `talentsphere-wave-1-foundation` becomes possible for the first time.

## Migration Plan

No data migration; these are new tables. The sequence matters more here than in most features,
because two of the ten items unblock eleven other features:

1. **`TS-BL-018` first, and alone.** It creates the three permission tables, implements the
   evaluator behind the existing `PermissionResolver` port, and ships enforcement with the
   enumeration check. Every other item in this feature and most items in every later feature read
   what it defines. Its first task is classifying the four Sprint 0 routes, because the
   enumeration check fails the build until they are.
2. **`TS-BL-020` next, in parallel with `TS-BL-019` and `TS-BL-022`.** These three touch disjoint
   code — the audit writer, the explanation endpoint, the seed — and `TS-BL-020` is the other item
   other features are blocked on. **Confirm `platform-core`'s task 2.9 harness exists before
   starting `TS-BL-020`**, not during it; if it does not, that is a scheduling conversation, not
   something to work around by testing against `docker-compose`.
3. **`TS-BL-021` after `TS-BL-020` and `TS-BL-018`** (D13's refined edge). It is the first
   consumer proving the evaluator gates a real read surface, and the first proving the trail is
   readable without exposing personal data.
4. **`TS-BL-023` once `TS-BL-022` and `design-system`'s `TS-BL-008` have landed**, then
   `TS-BL-024` → `TS-BL-025` → `TS-BL-026` in strict order. Each genuinely needs its predecessor:
   an override flow needs a surface to perform it on, break-glass is an override with an expiry,
   and elevation evidence needs elevations to evidence.
5. **`TS-BL-080` after `TS-BL-023`**, and after `identity-and-access`'s `TS-BL-016`. It extends
   the Cockpit's deactivation screen rather than adding a new area.
6. **Seeds are idempotent and run as a deployment step**, so a fresh environment reaches a usable
   state with the nine roles, the page catalog, the seeded matrix and one usable administrator.
   Re-running duplicates nothing.

**Rollback.** Each item is independently revertible and every migration is reversible per the
standing convention. Two asymmetries worth knowing: reverting `TS-BL-018` returns the product to
the empty-grant state, which is *correct behaviour* rather than an outage (D3) — every user is
activated and powerless, exactly as `identity-and-access` ships today. And `audit_logs` rows
cannot be removed by a rollback, by construction (D7); a reverted migration that drops the table
is the only way, which is a decision to make deliberately rather than in an incident.

## Open Questions

Each is genuinely deferrable — none changes the specs, the approach, or the task breakdown.

- **Seeded matrix values (D5) need business sign-off** on read scope across the candidate pool.
  Inherited as open from `talentsphere-wave-1-foundation`, unanswered since, and the one question
  here with a headline feature attached to it. Defaults are seeded and auditable; changing them is
  configuration, not code — which is precisely why it can ship before the answer arrives.
- **The break-glass notification target.** Provisionally all other Application Administrators plus
  the Auditor role. Security may want a different recipient; it is a configuration value either
  way.
- **Break-glass elevation duration.** Bounded and expiring is the requirement; the number belongs
  to whoever owns the support process. Seeded provisionally through the audited runtime-config
  path, so changing it is not a deployment.
- **Audit retention period** (`RET-004` defers to Company policy). The mechanism and its disposal
  attribution are specified; the interval is a policy input.
- **Whether the two Sprint 0 diagnostics routes should remain non-production-only once they are
  permission-gated** (D9). Gating them makes the environment restriction belt-and-braces rather
  than the only control. Keeping both is the conservative default and costs nothing; relaxing it
  is an owner decision about production support.
- **Leaver detection direction** (`D08`, `OD-001`) — Hubble push, poll, or discovery on failed
  login. Adds a trigger to `TS-BL-080`; does not change its flow.

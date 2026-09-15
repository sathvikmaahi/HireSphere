## Context

See `proposal.md` — Why. The design-relevant state, in one paragraph each:

**The code that exists.** `backend/app/` is platform-layer only. Relevant to this feature:
`runtime_config/registry.py` already declares `session.expiration_minutes` (480) and
`auth.rate_limit_attempts_per_minute` (10) under an `# --- Identity ---` heading, both closed-
registry keys with an audited change path; `audit/port.py` defines `AuditEvent` (actor, action,
target type and id, previous/new value, reason, correlation id, source address, timestamp) and
the `AuditSink` protocol, backed today by `LoggingAuditSink` writing `durable: false`;
`core/correlation.py`, `core/errors.py` and the OTel logging setup are in place. There is no
`identity` package, no auth route, and no Hubble adapter — including no mock, despite
`KNOWN_ISSUES.md` describing one in the present tense.

**The plan that exists.** `talentsphere-wave-1-foundation` carries two `identity/*` delta specs
and decision **D8** (Hubble behind an adapter with a contract test, identity keyed on the Hubble
identifier because email is the claim most likely to change or be absent). `platform-core`'s D11
assigns D8 to this feature. `platform-core`'s `platform/api-ingress` spec is written and fixes
the contract this feature's routes land on: one gateway, TLS-only, correlation id assigned at the
edge, and **every exposed endpoint classified — an endpoint declaring nothing denies everything.**
`design-system`'s `design-system/app-shell` spec fixes the two screens this feature drives: the
bare shell is used *only* for sign-in, deep links survive sign-in, and an activated user holding
no grants at all reaches the landing route and sees an empty state rather than a denial.

**The constraint that shapes almost everything below.** `OD-001` — the Hubble login contract — is
unconfirmed and owned by another team. `AUTH-009` requires timeout, refresh behaviour, token
shape and identity payload be confirmed before implementation is finalized. This feature cannot
wait for it and must not pretend to know it.

## Goals / Non-Goals

**Goals:**

- One seam for Hubble, so `OD-001` landing later is an adapter change and not a search.
- A session that a revocation can actually reach, on the next request, with no caching window.
- A `users` model `access-control-and-admin` can consume without this feature guessing at
  evaluator internals — and without that feature having to alter the table when it arrives.
- Honest reconciliation: every requirement inherited from `talentsphere-wave-1-foundation` lands
  somewhere traceable, and nothing Sprint 0 actually built is re-proposed as new.

**Non-Goals (design-level, beyond the proposal's scope boundary):**

- No permission *decision* logic of any kind, including no "temporary" role check inlined at a
  route while `TS-BL-018` is unbuilt. A temporary evaluator is the thing most likely to become
  the second evaluator, and `config.yaml`'s standing rule is that there is exactly one.
- No opinion on session storage *performance*. A per-request database read is the correct default
  at `D07`'s volume (<5k candidates, <30 postings, a few dozen staff users); optimizing it is a
  later problem with a measurable trigger, and `TS-BL-001`'s performance harness is where that
  trigger would show up.
- No frontend component design. The sign-in and access-denied views are compositions of
  `design-system` primitives; this feature specifies their behaviour, not their appearance.

## Decisions

### D1 — The Hubble adapter and its mock are the same first task, and the mock is a shipped artifact

All Hubble interaction is confined to one adapter exposing `authenticate(credentials) → identity
claims`, per inherited **D8**. Everything downstream consumes a normalized internal identity.
Unknown or missing claims are mapped explicitly rather than silently defaulted, and a recorded
contract test states what we currently believe the response shape to be.

The mock is built in the same task as the adapter, not after it, and lives in application source
rather than in the test tree — Local and Dev run against it until `OD-001` lands, and every
feature from `access-control-and-admin` onward develops against it.

*Why this is restated rather than simply inherited:* `KNOWN_ISSUES.md` asserts the mock already
exists. It does not. Inheriting D8 without checking would have produced a `tasks.md` that skipped
building the one artifact four downstream features depend on. **Recorded as a correction of
record; the `KNOWN_ISSUES.md` line should be amended to describe the mock as planned rather than
running, which is a repository edit outside this planning change.**

*Alternative considered:* wait for `OD-001` before building authentication at all. Rejected for
the reason the inherited decision gives — the whole of Phase 1 sits behind a session, and
blocking on another team's contract stalls eleven features to avoid one adapter rewrite.

### D2 — Where every inherited requirement lands, stated explicitly

D.9 splits what `talentsphere-wave-1-foundation` treated as one slice into three items
(`TS-BL-013` login, `TS-BL-014` session, `TS-BL-015` revocation). Its `identity/authentication`
delta spec therefore redistributes. The mapping, so that "missing" can be distinguished from
"moved" by anyone auditing the two changes side by side:

| Inherited requirement | Inherited path | Lands in | Item |
|---|---|---|---|
| Hubble-backed authentication | `identity/authentication` | `identity/authentication` | `TS-BL-013` |
| No credential retention | `identity/authentication` | `identity/authentication` | `TS-BL-013` |
| Identity mapping | `identity/authentication` | `identity/authentication` | `TS-BL-013` |
| Authentication rate limiting | `identity/authentication` | `identity/authentication` | `TS-BL-013` |
| Authentication logging | `identity/authentication` | `identity/authentication-audit` | `TS-BL-017` |
| Session content | `identity/authentication` | `identity/session-management` | `TS-BL-014` |
| Session lifecycle | `identity/authentication` | `identity/session-management` | `TS-BL-014` |
| Forced session revocation | `identity/authentication` | `identity/session-revocation` | `TS-BL-015` |
| Activation required for access | `identity/user-activation` | `identity/user-activation` | `TS-BL-016` |
| Local user records | `identity/user-activation` | `identity/user-model` | `TS-BL-013` |
| Activation state transitions | `identity/user-activation` | `identity/user-activation` | `TS-BL-016` |
| Leaver handling and ownership continuity | `identity/user-activation` | `identity/user-activation` (invariant) + `TS-BL-080` (workflow) | `TS-BL-016` |
| Multiple roles per user | `identity/user-activation` | `identity/user-model` | `TS-BL-013` |
| Practice as a data attribute | `identity/user-activation` | **not carried — dropped, see D4** | — |
| Access-denied surface discloses nothing | `identity/user-activation` | `identity/user-activation` | `TS-BL-016` |

Two paths are preserved exactly (`identity/authentication`, `identity/user-activation`), matching
what `platform-core` did for its three. The three new paths exist because D.9 created three items
that did not exist when the old spec was written — not because the old organization was wrong.

**One inherited requirement is deliberately not carried**, and the row above says so rather than
omitting it — an inherited requirement that simply vanished from the table would be
indistinguishable from one nobody noticed. D4 records why.

*Alternative considered:* keep all fifteen requirements under the two inherited paths. Rejected:
three of the five backlog items would then have no capability of their own, and "which spec
governs `TS-BL-015`" would have no answer — which is the question an implementer actually asks.

### D3 — `users.hubble_user_id` and the onboarding Hubble ID are different fields in different tables, and must stay that way

Two things in this product are called a Hubble identifier. They are not the same integration
point, and `glossary.md` says so directly:

|  | This feature | `decision-and-offers` (`TS-BL-068`) |
|---|---|---|
| Field | `users.hubble_user_id` | `offer_onboarding_records.hubble_id` |
| Attaches to | a TalentSphere **user** | an **Application**'s onboarding record |
| Arrives from | the login response, automatically | a recruiter typing it, manually (`OFF-004`) |
| Means | "this staff member authenticated" | "this hire exists in the HR system" (`G-13`, `BR-019`) |
| Uniqueness | one local user per Hubble identifier | unique **among recruited candidates** (`OFF-006`/`OFF-007`) |

**No foreign key, no shared uniqueness constraint, and no shared column name between them.** Two
concrete reasons, not just tidiness:

1. Candidates have **zero system access** — a deliberate non-goal in `project.md`. A hired
   candidate acquiring a Hubble ID must not, by that fact, acquire a `users` row.
2. `D08` raises internal candidates and rehires: an employee applying internally already holds a
   Hubble ID *at submission*. That person legitimately appears as a recruiter in `users` and as a
   candidate on an onboarding record, with the same underlying enterprise identity. A cross-entity
   uniqueness constraint would reject a real and expected case.

*Alternative considered:* one `hubble_identities` table both reference. Rejected — it models the
two as one relationship when they are two, and it would make the constraint in (2) the natural
thing to add rather than the thing to avoid.

### D4 — The nine roles are seeded here as identity; what they may do is seeded by `TS-BL-022`

`C-02` fixes nine roles. `domain-model.md` fixes `User ──N:M──▶ Role ──N:M──▶ PagePermission`.
This feature owns the left edge and none of the right:

- **Here (`TS-BL-013`):** `roles` with identity-only columns (`id`, `key`, `name`, `description`,
  `is_system_role`, `status`) and the nine rows seeded; `user_roles` as the N:M assignment;
  `users` per `reference/spec.md` §12.2 (`hubble_user_id`, `display_name`, `email`, `status`,
  `last_login_at`) — and nothing beyond §12.2's columns.
- **`access-control-and-admin` (`TS-BL-018`, `TS-BL-022`):** `page_permissions`,
  `role_permissions`, `user_permission_overrides`, the matrix values, deny-wins precedence, and
  the explanation endpoint.

The split is at the join table, and it is what lets both features own a coherent thing:
`role_permissions` references `roles.id` and needs those rows to exist, while nothing about
seeding a role's *name* requires knowing what it may do. `TS-BL-022`'s D.9 title is "Seed data:
9-role permission **matrix**" — the matrix, which is exactly the part left here undone.

**Why there is no `practice` column on `users`.** An earlier draft of this design carried one,
inherited from `talentsphere-wave-1-foundation`'s `identity/user-activation` spec and cited to
`D02`/`C-03`. Verification found the citation does not hold: `D02` recommends modelling Practice
as a data attribute **on JobPosting** — and marks even that "not yet confirmed" — while
`glossary.md` likewise defines Practice as "a data attribute on postings." Neither decision
discusses a practice attribute on a staff member, and `reference/spec.md` §12.2's `users` table
has no such column.

A user-level practice attribute may well be right eventually, but nothing in this feature reads
it: the only behaviour ever specified for it was an administrator filtering the user list or the
permission matrix, which is `access-control-and-admin`'s `TS-BL-023` and `ADM-003`. Carrying an
uncited column with no consumer would also prejudge an open question this design already
records — whether Hubble owns the practice hierarchy, in which case practice belongs on the user
as a refreshed claim rather than as locally maintained data.

So it is dropped here, and whichever feature first has a real consumer adds it with a real
citation and a decided answer to that question. The cost of deferring is one nullable column in a
reversible migration; the cost of keeping it is a field whose provenance reads as settled when it
is not.

*Alternative considered:* keep the column and re-describe it in D4 as a deliberate extension of
`D02`/`C-03`'s never-authorizes principle to a case those decisions did not cover. Honest, and
preferable to the miscitation — but the only reason available to write down was "a later feature
will filter on it," which is the definition of a field added for later.

*Alternative considered:* defer the `roles` table entirely to `access-control-and-admin`.
Rejected: `TS-BL-013` ships first and `AUTH-004` requires the session to carry roles, so it would
have to invent a placeholder role representation and then migrate off it — creating precisely the
guess-at-internals this feature is meant to avoid.

### D5 — Resolved permissions arrive through a port, exactly as audit already does

`AUTH-004` requires the session to carry resolved page permissions. The component that resolves
them is `TS-BL-018`, which **depends on `TS-BL-014`** — so the requirement and its implementation
cannot ship together, by construction of the dependency graph.

Sprint 0 already solved this shape once. `audit/port.py` defines the real event and protocol, with
a placeholder sink, so that call sites are written against the real port from the first commit and
Sprint 1 replaces an implementation rather than retrofitting callers. This feature does the same:
it declares a `PermissionResolver` port returning a user's resolved permissions, and ships a
`NoGrantsResolver` that returns an empty set. `TS-BL-018` installs the real one; no call site
changes.

**The empty set is a correct state, not a stub state.** `talentsphere-wave-1-foundation`'s
inherited requirement covers it (an activated user with no grants is admitted to the shell and
denied every page), and `design-system`'s app-shell spec builds the landing route that renders it
— explicitly because *activated but not yet granted anything* is the most common user state
during `access-control-and-admin`'s rollout. So the interim behaviour is one that two other
features already specify, not a behaviour invented to fill a gap.

*What this feature must not do:* infer permissions from `user_roles` at a route. That is
permission evaluation, it would be the second evaluator, and it would be wrong — `AUTHZ-005`'s
direct-denial-overrides-grants rule lives in data this feature does not own.

*Alternative considered:* omit permissions from the session until `TS-BL-018` exists and add the
field later. Rejected: `/api/auth/me` is consumed by the frontend shell from `TS-BL-014` onward,
and a response shape that gains a field later is a breaking change to every consumer written
against it.

### D6 — Sessions are server-side records, not self-validating tokens

The session is a row. Each request presents an opaque session identifier; the request path loads
the row and checks status and expiry.

*Why, and why this is forced rather than chosen:* `AUTH-007` requires revoked sessions to be
invalid **immediately** for subsequent requests, and `config.yaml` states the same rule for the
evaluator — "uncached, so revocation bites on the next request." A self-validating JWT is valid
until it expires by definition; making one revocable requires a per-request denylist check, which
is a session lookup with extra cryptography and a second source of truth about whether a session
is alive. `G-11`'s entire point is the leaver holding a live session.

Transport: the identifier travels in an `HttpOnly`, `Secure`, `SameSite` cookie rather than in
JavaScript-reachable storage. `SEC-007` (OWASP) and the fact that no public routes exist make the
cross-site-request surface small and the token-theft surface the one worth closing.

*Alternative considered:* short-lived JWT plus refresh token. Rejected: it buys statelessness this
system has no need for — a single Cloud Run service against one Cloud SQL instance at `D07`'s
volume — and pays for it with a revocation story that is either delayed by the access-token
lifetime or reintroduces the lookup it was meant to remove.

### D7 — The login route is the first authenticated route through the gateway, so it classifies from day one

`platform/api-ingress` requires every exposed route to be classified as public, authenticated, or
permission-gated, and treats an endpoint declaring nothing as denying everything. Sprint 0 left
four routes unclassified because the mechanism did not exist; that is recorded as residual scope
on `TS-BL-003` and `TS-BL-018`.

This feature adds the first routes that are not diagnostics, and classifies them at the point of
adding rather than waiting for the enumeration:

| Route | Classification |
|---|---|
| `POST /api/auth/login` | public — the only unauthenticated non-diagnostic route in the product |
| `POST /api/auth/logout` | authenticated |
| `POST /api/auth/refresh` | authenticated |
| `GET /api/auth/me` | authenticated |
| `POST /api/auth/revoke-session` | permission-gated (declared; enforced when `TS-BL-018` lands) |
| `GET|POST|PATCH /api/admin/users*` | permission-gated (declared; enforced when `TS-BL-018` lands) |

Declaring a requirement that nothing yet enforces is deliberate: a route carrying a declaration
fails closed under the ingress spec's default, while a route carrying none is the thing that gets
missed in `TS-BL-018`'s enumeration. This is the same discipline `platform-core` applied to the
four Sprint 0 routes — name them so they are classified, rather than skipped.

### D8 — Deactivation's ownership invariant is specified here; the reassignment workflow is `TS-BL-080`, and it was missing from the backlog

The inherited leaver requirement asserts that a deactivated user's owned work "remains intact and
remains reassignable to another user," and that historical actor attribution is preserved
unchanged. Checking that against D.9/D.10 turned up a real gap: **no backlog item builds it.**
`D08` and `G-11` each assert ownership reassignment is needed; neither was ever decomposed, while
D.10 and D.11 declare the decomposition complete.

Fixed at source rather than worked around here, per `AGENTS.md`'s convention — see
`exploration-notes.md` **D.10.1** (2026-08-25), which records the contradiction and closes it as
`TS-BL-080`, owned by `access-control-and-admin`, taking the backlog to 80 items.

**The split this feature holds to:** `TS-BL-016` specifies and enforces the *invariant* —
deactivation revokes sessions immediately, deletes nothing, orphans nothing, and rewrites no
historical attribution. `TS-BL-080` builds the *workflow* that hands work to a named successor.
The invariant is testable today against the tables that exist; deferring it until there is work
to reassign would leave the destructive case unguarded in the interim.

### D9 — Authentication audit events are emitted through the existing port and name no credential

`TS-BL-017` adds an authentication event vocabulary — login success, login failure, logout,
forced revocation, activation-state change — constructed as `AuditEvent`s and passed to
`get_audit_sink()`.

Three consequences worth stating, because each is easy to get wrong:

1. **`TS-BL-020` depends on `TS-BL-017`, not the reverse.** The durable writer is built to accept
   the login event; until it exists these events go to the structured log with `durable: false`,
   which is the same honest interim state the runtime-config path is already in.
2. **A failed login has no user actor.** `AuditEvent.__post_init__` requires a user or a service
   account, and an unknown or unactivated identity supplies neither. Failed authentication is
   therefore recorded with a service-account actor representing the authentication subsystem, and
   the attempted identity as the *target*, not the actor — which is also the honest reading:
   the system, not the stranger, is the thing that acted.
3. **`AUTH-008` and `SEC-015` bind the payload.** No password, no token, no bearer value, no
   Hubble response body reaches an audit record or a log line. What is recorded is timestamp,
   outcome, reason category, source address and correlation id.

## Risks / Trade-offs

- **`OD-001` lands and contradicts the recorded contract test** → The adapter is the only
  affected surface by construction (D1), and the contract test is the thing that fails loudly
  when the belief turns out wrong. Residual risk accepted: if Hubble returns an identity payload
  with no stable non-email identifier, D8's keying decision itself needs revisiting, which is a
  data-model change and not an adapter change. That is the one outcome this design cannot absorb
  cheaply, and it is worth asking the platform team about before `TS-BL-013` starts.
- **`KNOWN_ISSUES.md` describes a mock that does not exist**, and it feeds release notes → Named
  in D1 and in the proposal rather than quietly corrected. Anyone reading the release notes today
  is told authentication runs against a mock; it runs against nothing.
- **The permission port ships with a resolver that grants nothing**, so for the whole interval
  between `TS-BL-014` and `TS-BL-018` every user is activated and powerless → Correct per two
  other features' specs (D5), and visibly so: the landing route says as much and offers a route
  to request access. The real risk is someone "temporarily" inlining a role check to unblock a
  demo; called out as an explicit non-goal above for that reason.
- **Session rows are read on every request, adding a database round trip to the hot path** →
  Accepted deliberately (D6). At this product's volume the round trip is cheap and the
  alternative trades a correctness property for it. `TS-BL-001`'s performance baseline is where
  a real regression would surface.
- **Two active changes describe `identity/authentication` and `identity/user-activation`** →
  Same accepted interim state `platform-core` recorded. This change is authoritative for these
  five items from now on; the redistribution table in D2 exists so the overlap is legible rather
  than confusing.
- **The nine roles are seeded before anything can use them**, so a mis-seeded role key is
  invisible until `TS-BL-022` → Mitigated by seeding role keys as a closed, tested vocabulary in
  `TS-BL-013` — the same closed-registry discipline `runtime_config` and `features/flags.py`
  already use, where an undeclared value raises rather than resolving to nothing.
- **A first-login pending record is a write performed by an unauthenticated, unactivated
  stranger** — anyone Hubble authenticates can create a row → Bounded by
  `auth.rate_limit_attempts_per_minute` (already declared), by the record carrying no permissions
  and no application data, and by Hubble authentication being a precondition, so the actor is a
  known employee rather than an anonymous caller.

## Migration Plan

No data migration — these are the product's first domain tables. Sequence:

1. **`TS-BL-013` first and alone.** It creates `users`, `roles`, `user_roles` and the adapter;
   the other four items all read what it defines.
2. **`TS-BL-014` before `TS-BL-016`.** D.9 has both depending only on `TS-BL-013`, so they are
   formally parallel, but activation's access-denied path is only observable once there is a
   session to be denied *within*. Building them in this order avoids testing activation through a
   surface that does not yet exist. This is a sequencing preference, not a dependency edge — the
   `depends_on` in `tasks.md` records the real graph.
3. **`TS-BL-015` and `TS-BL-017` after `TS-BL-014`**, in either order; they touch disjoint code.
4. **Hand off to `access-control-and-admin`** with three named seams: the `PermissionResolver`
   port (D5), the `roles` rows `TS-BL-022`'s matrix attaches to (D4), and the authentication
   events `TS-BL-020`'s writer must accept (D9).

**Rollback.** Each item is independently revertible; the migrations are reversible per the
standing convention. The one irreversible act is the first real login writing a `users` row, and
it is irreversible only in the sense that any first record is.

## Open Questions

Deferrable without changing the specs, the approach, or the task breakdown. All five come from
`D08`'s integration-surface list and none has been answered by assumption:

- **Leaver/deactivation sync direction** — does Hubble push deactivation, does TalentSphere poll,
  or is a leaver discovered on failed login? `TS-BL-016` handles a TalentSphere-side deactivation
  however it is triggered, so the answer adds a trigger rather than changing the flow.
- **Does Hubble own the practice/organization hierarchy?** Unanswered, and now genuinely
  deferrable rather than quietly prejudged: no user-level practice attribute is built here (D4),
  so whichever feature first needs one decides then whether it is a refreshed Hubble claim or
  locally maintained data. Either way it never authorizes (`glossary.md`), so the answer does not
  touch the permission boundary.
- **Is MFA Hubble's responsibility?** Assumed yes, consistent with `AUTH-001`/`AUTH-002` and with
  never holding a reusable credential. Worth confirming rather than discovering.
- **Token lifetime and refresh on the Hubble side** — distinct from TalentSphere's own session
  expiry, which is already configurable. Part of `AUTH-009`.
- **The provisional 480-minute session lifetime.** Inherited from Sprint 0's registry, whose own
  description marks it provisional pending the confirmed enterprise standard. Changing it is a
  runtime-config write, not a deployment.

**Not deferred, and already asked in the proposal:** whether a Hubble validation endpoint exists
(`OD-002`) belongs to `decision-and-offers`, not here — this feature never validates an
onboarding Hubble ID (D3).

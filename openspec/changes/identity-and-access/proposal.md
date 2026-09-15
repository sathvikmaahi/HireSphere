# Identity and Access

## Why

Nothing else in TalentSphere can be attributed to anybody until this feature exists. The audit
substrate needs an actor, the permission evaluator needs a session to evaluate *for*, and the
authenticated shell needs someone to be authenticated as. `identity-and-access` is the point at
which an enterprise employee becomes a named, activated, revocable TalentSphere user — and the
point at which the project's foundational split is made real: **authentication is external and
belongs to Hubble; authorization is local, separate, and belongs to
`access-control-and-admin`.**

**Why now, and what this is not.** Unlike `platform-core`, this feature is genuinely unbuilt.
Sprint 0 of `talentsphere-wave-1-foundation` shipped ~2,000 lines of platform-layer code and
explicitly no domain code; `users`, `roles`, `user_roles` and the session store were Sprint 1's
scope, and Sprint 1 never ran. There is no authentication code in `backend/app/` — no Hubble
adapter, no session store, no `/api/auth/*` route.

**A correction of record, made honestly rather than inherited quietly.** `KNOWN_ISSUES.md` states
that *"Authentication runs against a mock in Local and Dev until the real contract is recorded."*
That is not true of the repository as it stands: neither the adapter nor the mock exists. The
sentence describes `design.md` D8's *intent* as though it were shipped state. This change treats
the mock as work to be done in `TS-BL-013`, not as a dependency already satisfied. See `design.md`
D1.

**What Sprint 0 did leave behind, and this feature must consume rather than re-invent:**

- `session.expiration_minutes` (default 480) and `auth.rate_limit_attempts_per_minute`
  (default 10) are **already declared** in `backend/app/runtime_config/registry.py`, under an
  `# --- Identity ---` heading, with a closed registry and an audited change path. Session expiry
  and login throttling are configurable today; this feature reads those keys and adds none of its
  own settings mechanism.
- `backend/app/audit/port.py` defines `AuditEvent` and the `AuditSink` protocol, currently backed
  by `LoggingAuditSink` (`durable: false`). `TS-BL-017` emits through that port, exactly as the
  runtime-config path already does.
- Correlation IDs, the single global error-body shape, and OpenTelemetry logging exist and are
  load-bearing for `AUTH-008` (log failures without exposing credentials) and for the
  access-denied surface's correlation identifier.

Per [D.5](../talentsphere/exploration-notes.md) as adjusted to feature level by
[D.8.4](../talentsphere/exploration-notes.md), and per `platform-core`'s `design.md` D11 which
mapped the old change's decisions across the five Phase-1 features, this feature inherits
`talentsphere-wave-1-foundation`'s **D8** (Hubble behind an authentication adapter with a recorded
contract test) and its two `identity/*` delta specs. That change **splits; it is not rewritten.**

## What Changes

Five backlog items, `TS-BL-013` through `TS-BL-017`, exactly as decomposed in
[D.9](../talentsphere/exploration-notes.md). None is built.

- **`TS-BL-013` User model and Hubble SSO login.** The `users` and `user_roles` tables, the nine
  role rows as an identity vocabulary (`C-02`), and the Hubble authentication adapter behind a
  single `authenticate(credentials) → identity claims` seam with a recorded contract test and a
  first-class mock for Local and Dev. Identity maps on the Hubble user identifier, never on
  email (`AUTH-001`, `AUTH-002`, inherited D8). Depends on `platform-core`'s `TS-BL-001` and
  `TS-BL-003` — the login route has to land on a real ingress, whose contract is now written
  (`platform/api-ingress`), not assumed.
- **`TS-BL-014` Session issuance and token handling.** Server-side session records, the session
  content `AUTH-004` fixes, `/api/auth/login|logout|refresh|me`, and expiry read from the existing
  `session.expiration_minutes` key. Sessions are **stored and looked up per request, not
  self-validating tokens** — `AUTH-007` and `config.yaml`'s "uncached, so revocation bites on the
  next request" leave no room for a stateless bearer token that stays valid until it expires.
- **`TS-BL-015` Forced session revocation (`G-11`).** Revocation by an Application Administrator,
  invalid immediately for every subsequent request, with no permissible caching window
  (`AUTH-006`, `AUTH-007`). This is the answer to `D08`'s leaver problem — a terminated recruiter
  holding a live session is an audit finding.
- **`TS-BL-016` Account activation and first-login flow.** Activation states
  (active/inactive/pending/deactivated), the pending record created on first login of an unknown
  identity, administrator pre-provisioning, deactivation with a mandatory reason, and the
  access-denied surface that discloses nothing (`AUTH-005`, `BR-001`, `ADM-001`).
- **`TS-BL-017` Login → audit event hook.** Login success, login failure, logout, and revocation
  emitted as `AuditEvent`s through the existing port. `TS-BL-020` (the durable writer, in
  `access-control-and-admin`) **depends on this item**, not the other way round — the login event
  is the first event the audit substrate is built to accept.

**The scope boundary that matters most.** The `users` model carries role *assignments*; it does
not evaluate them. Deciding what a role may do — the matrix, deny-wins precedence, the explanation
endpoint — is `access-control-and-admin`'s `TS-BL-018`, unproposed as of this writing. This
feature therefore declares a permission-resolution seam and ships a resolver that grants nothing,
which lands users in the *activated-but-no-grants* state `design-system`'s app-shell spec already
specifies a landing route for. See `design.md` D4 and D5.

**Explicitly not in this change:**

- **No permission evaluation, no permission matrix, no Admin Cockpit UI** (`TS-BL-018`–`TS-BL-026`).
- **No Hubble ID capture at onboarding.** Per `glossary.md`, that is a *separate integration
  point* — a manually captured proof-of-hire field on the onboarding record (`D08`, `OFF-004`),
  owned by `decision-and-offers`' `TS-BL-068`. It shares a vendor name with this feature and
  nothing else: it attaches to a candidate, not to a user, and candidates have zero system access
  by design. `design.md` D3 records why the two identifiers must not share a namespace.
- **No user provisioning into Hubble.** Ruled out in `D08`; TalentSphere never creates employee
  records.
- **No ownership reassignment workflow.** The invariant that deactivation must not orphan or
  re-attribute owned work is specified here; the workflow that hands work to a new owner is a
  genuine gap in the 79-item backlog, now recorded as `TS-BL-080` — see `design.md` D8.
- **No sign-in screen components.** The bare shell and its sign-in route are `design-system`'s
  `TS-BL-008`; this feature supplies the flow those components call.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is
new. Paths preserve the `identity/` organization established by `talentsphere-wave-1-foundation`,
and two of them reuse that change's exact existing paths rather than inventing parallel ones.

- `identity/authentication`: Hubble-backed login as the only interactive credential path, no
  credential retention, identity mapping keyed on the Hubble identifier, rate limiting, and
  behavior when the identity provider is unavailable. *(`TS-BL-013`)* — **path preserved**
- `identity/user-model`: the local user and role-assignment data model — user record fields and
  their uniqueness constraints, multiple roles per user, the nine-role vocabulary as identity
  rather than capability, and the seam across which resolved permissions arrive from a component
  this feature does not own. *(`TS-BL-013`)*
- `identity/session-management`: session issuance, content, transport, renewal, expiry and
  logout — server-side records that a revocation can reach. *(`TS-BL-014`)*
- `identity/session-revocation`: administrative forced revocation and revocation-on-deactivation,
  invalid immediately with no caching window. *(`TS-BL-015`)*
- `identity/user-activation`: activation states and transitions, first-login pending records,
  administrator pre-provisioning, leaver handling, and an access-denied surface that discloses
  nothing. *(`TS-BL-016`)* — **path preserved**
- `identity/authentication-audit`: the authentication event vocabulary emitted through the
  existing audit port, and what those records may and may not contain. *(`TS-BL-017`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**Overlap to resolve outside this change, and a redistribution to declare.**
`talentsphere-wave-1-foundation` remains active and unarchived, and its `identity/authentication`
delta spec currently carries session content, session lifecycle and forced revocation alongside
login. D.9 splits those into three independently deployable items, so those requirements move to
`identity/session-management` and `identity/session-revocation` here. `design.md` D2 carries the
requirement-by-requirement mapping so an auditor comparing the two changes can see where every
inherited requirement landed — including the one that is deliberately **not** carried forward, a
user-level practice attribute whose cited sources turn out to be about postings rather than
users (`design.md` D4). Retiring that change spans five features and is not this change's to
perform.

## Impact

**New application code** — `backend/app/identity/` (Hubble adapter and its mock, session store,
activation service), `backend/app/api/routes/auth.py`, `backend/app/api/routes/users.py`, new
Alembic migrations for `users`, `roles`, `user_roles` and the session table, and
`frontend/src/` sign-in and access-denied views rendered inside `design-system`'s bare shell.

**Existing code consumed, not modified** — `backend/app/runtime_config/registry.py` (the two
identity keys already declared), `backend/app/audit/port.py`, `backend/app/api/deps.py`,
`backend/app/core/errors.py`, `backend/app/core/correlation.py`.

**Infrastructure** — Hubble base URL and credentials as injected settings from Secret Manager
(never committed, per the standing convention); the four `/api/auth/*` routes added to the API
Gateway's generated ingress contract. `TS-BL-013` is the **first feature to add authenticated
routes to that gateway**, and so the first to exercise `platform/api-ingress`'s "every exposed
endpoint is classified" requirement against something other than the four Sprint 0 diagnostics.

**Downstream features that block on this one** — `access-control-and-admin` (`TS-BL-018` needs a
session to evaluate for; `TS-BL-020` needs `TS-BL-017`'s login event as its first accepted
event), `design-system`'s sign-in route, and every feature that attributes a write to an actor,
which is all of them.

**External constraints carried, not solved here** — the Hubble login contract `OD-001` is
unconfirmed and owned by another team (`AUTH-009` requires timeout, refresh, token shape and
identity payload be confirmed before implementation is finalized). This feature does not block on
it: the adapter is the seam, the mock is a first-class artifact, and `TS-BL-013`'s contract test
records what we currently believe the response shape to be. Five further questions `D08` raises —
leaver/deactivation sync direction, whether Hubble owns the practice hierarchy, MFA ownership,
token lifetime, and internal-candidate identity collision — are recorded as open questions in
`design.md` rather than answered by assumption.

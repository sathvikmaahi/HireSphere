# Identity and Access — Backlog

This feature's complete backlog: five items, `TS-BL-013` through `TS-BL-017`, decomposed in
`exploration-notes.md` D.9. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built.** Sprint 0 of `talentsphere-wave-1-foundation` shipped platform-layer
code only; `users`, `roles`, `user_roles` and the session store were Sprint 1's scope and Sprint 1
never ran. There is no Hubble adapter and no mock, notwithstanding `KNOWN_ISSUES.md`'s present-
tense description of one — see `design.md` D1. What Sprint 0 *did* leave is consumed rather than
rebuilt: the two identity keys in `backend/app/runtime_config/registry.py`, the audit port, the
correlation-ID and error-shape substrate.

**Two dependency edges point outside this feature.** `TS-BL-013` needs `platform-core`'s
`TS-BL-001` and `TS-BL-003`; both are proposed and largely built. In the other direction,
`access-control-and-admin`'s `TS-BL-018` and `TS-BL-020` depend on items here, so the seams named
in `design.md` D4, D5 and D9 are deliverables, not internal detail.

---

## 1. TS-BL-013 — User model and Hubble SSO login

**Goal:** an enterprise employee can authenticate through Hubble and be recognized as exactly one
local user holding named roles, with every Hubble-specific detail confined to one adapter that
`OD-001` can later change without touching anything else. Covers `identity/authentication` and
`identity/user-model`.

```yaml
backlog_items:
  - id: TS-BL-013
    feature: identity-and-access
    depends_on: [TS-BL-001, TS-BL-003]
    status: not-started
```

- [ ] 1.1 Confirm `platform-core`'s ingress contract before writing a route: read
      `platform/api-ingress`, and record which of its requirements the new auth routes must
      satisfy (TLS-only edge, edge-assigned correlation ID, mandatory classification)
- [ ] 1.2 Create the `users` table by reversible migration — Hubble user identifier (unique),
      display name, email (unique), status, last login, created/updated — per
      `reference/spec.md` §12.2 and carrying no column beyond it (design D4)
- [ ] 1.3 Create the `roles` table with identity-only columns and seed exactly the nine `C-02`
      roles as a closed vocabulary, rejecting any key outside it (design D4)
- [ ] 1.4 Create the `user_roles` assignment table supporting multiple roles per user, with no
      permission-bearing columns of its own
- [ ] 1.5 Assert by test that no column, constraint, or index ties `users.hubble_user_id` to the
      onboarding Hubble ID owned by `decision-and-offers`, and that one person may legitimately
      appear as both a user and a recruited candidate (design D3)
- [ ] 1.6 Build the Hubble authentication adapter exposing `authenticate(credentials) → identity
      claims`, mapping unknown or missing claims explicitly and failing rather than producing a
      partial identity
- [ ] 1.7 Build the substitute provider for Local and Dev in application source, not the test
      tree, and record the contract test stating the response shape currently believed correct
      (design D1)
- [ ] 1.8 Implement `POST /api/auth/login` against the adapter, classified public — the only
      unauthenticated non-diagnostic route in the product (design D7)
- [ ] 1.9 Implement identity mapping on the Hubble identifier: refresh display name and email on
      the existing record, and never create a second user for a changed email
- [ ] 1.10 Handle the email collision the uniqueness constraint exposes — a refreshed email
      already held by another user establishes the session, leaves the stored email unchanged,
      and records the collision for administrative attention
- [ ] 1.11 Enforce authentication rate limiting from the existing
      `auth.rate_limit_attempts_per_minute` key, asserting a runtime change takes effect without
      redeployment
- [ ] 1.12 Assert by test that no submitted password, provider token, or raw provider response is
      present in the database, logs, session record, or any error response
- [ ] 1.13 Declare the `PermissionResolver` port and ship the resolver that returns no grants,
      following the audit port's precedent so `TS-BL-018` replaces an implementation rather than
      retrofitting call sites (design D5)
- [ ] 1.14 Assert by test that no access decision anywhere in this item is derived from role
      assignments directly — there is one evaluator and this feature is not it
- [ ] 1.15 Verify the identity-provider-unavailable path: authentication service failure reported,
      no cached or local fallback, and existing valid sessions unaffected

---

## 2. TS-BL-014 — Session issuance and token handling

**Goal:** an authenticated user holds a session the system can inspect, renew, end, and — the
property everything else depends on — reach and invalidate. Covers `identity/session-management`.

```yaml
backlog_items:
  - id: TS-BL-014
    feature: identity-and-access
    depends_on: [TS-BL-013]
    status: not-started
```

- [ ] 2.1 Create the session table by reversible migration, holding user reference, status,
      login and expiry timestamps, and the outcome that distinguishes expiry from logout from
      revocation
- [ ] 2.2 Issue sessions on successful login and resolve session state from storage on every
      authenticated request (design D6)
- [ ] 2.3 Assert by test that no credential the system issues can be validated without consulting
      stored session state — the requirement `AUTH-007` rests on
- [ ] 2.4 Carry the full `AUTH-004` session content, with resolved permissions obtained through
      the port and present-but-empty until `TS-BL-018` installs the evaluator
- [ ] 2.5 Transport the session credential out of reach of page scripts, over encrypted
      connections only, restricted from cross-site submission
- [ ] 2.6 Implement `GET /api/auth/me`, returning the caller's own identity, roles, permissions
      and session state and never another user's
- [ ] 2.7 Implement `POST /api/auth/refresh`: extend expiry retaining session identity, refuse
      renewal of an expired session
- [ ] 2.8 Implement `POST /api/auth/logout`, invalidating the session for all subsequent requests
- [ ] 2.9 Read expiry from the existing `session.expiration_minutes` key, asserting a runtime
      change applies to sessions established afterwards without redeployment
- [ ] 2.10 Classify every route this item exposes, and assert that a route carrying no
      declaration denies rather than allows (design D7)
- [ ] 2.11 Resolve session context once per request and assert every component handling that
      request sees the same identity, roles and permissions
- [ ] 2.12 Hand the frontend a session-aware API client and wire `design-system`'s bare shell
      sign-in route to it, preserving a deep link requested before sign-in

---

## 3. TS-BL-015 — Forced session revocation (G-11)

**Goal:** a leaver's live session ends the moment someone says so, with no cache, token lifetime,
or replication interval able to delay it. Covers `identity/session-revocation`.

```yaml
backlog_items:
  - id: TS-BL-015
    feature: identity-and-access
    depends_on: [TS-BL-014]
    status: not-started
```

- [ ] 3.1 Implement revocation of all of a user's active sessions, invalidating every one rather
      than only the most recent
- [ ] 3.2 Implement `POST /api/auth/revoke-session`, classified permission-gated and denying in
      the absence of an evaluator verdict rather than falling back to a role check (design D5, D7)
- [ ] 3.3 Assert by test that a request presenting a revoked session immediately after revocation
      is rejected — the no-caching-window property, tested as a negative
- [ ] 3.4 Make revocation idempotent: revoking an expired or already-ended session succeeds
      without error
- [ ] 3.5 Couple deactivation to revocation as one operation, so neither outcome is observable
      without the other, and assert re-activation restores no prior session
- [ ] 3.6 Record the ended-session outcome distinguishing revocation from expiry and logout, while
      asserting the response to the holder discloses which occurred to no one
- [ ] 3.7 Emit the revocation audit event naming both the acting administrator and the affected
      user (consumed from `TS-BL-017`'s vocabulary if that item lands first; defined here if not)

---

## 4. TS-BL-016 — Account activation and first-login flow

**Goal:** being authenticated by Hubble gets you nothing until an administrator says otherwise —
and a user who leaves takes their access with them but not their work. Covers
`identity/user-activation`.

```yaml
backlog_items:
  - id: TS-BL-016
    feature: identity-and-access
    depends_on: [TS-BL-013]
    status: not-started
```

- [ ] 4.1 Implement the four-state user status (active, inactive, pending, deactivated) with
      exactly one held at a time
- [ ] 4.2 Create a pending, permission-less record on first login of an unknown identity, refreshed
      rather than duplicated on repeat attempts, and visible to administrators for activation
- [ ] 4.3 Implement administrator pre-provisioning: create and activate a local user for a known
      Hubble identity so that user is admitted on next login with roles assigned
- [ ] 4.4 Implement `GET|POST|PATCH /api/admin/users`, classified permission-gated, for listing,
      creating/activating, and updating user status and profile metadata
- [ ] 4.5 Require a reason on deactivation, rejecting a missing one with a field-level validation
      error
- [ ] 4.6 Audit every status change with actor, previous value, new value, timestamp, reason and
      correlation ID, and assert a failed audit write fails the transition
- [ ] 4.7 Assert the ownership invariant by test: deactivation deletes nothing, orphans nothing,
      re-attributes no historical record, and leaves owned work reassignable — the invariant
      `TS-BL-080` will satisfy with a real workflow (design D8)
- [ ] 4.8 Refuse deletion of a user holding owned or authored work, in favour of deactivation
- [ ] 4.9 Build the access-denied surface disclosing only the denial, a correlation identifier and
      contact guidance — and assert it does not distinguish unknown from pending from deactivated
- [ ] 4.10 Verify the activated-but-ungranted path end to end: the user reaches
      `design-system`'s landing route and sees an empty state with a route to request access,
      never a denial (design D5)

---

## 5. TS-BL-017 — Login → audit event hook

**Goal:** the audit trail's first events exist before the trail does, emitted through the port
Sprint 0 already defined, so `TS-BL-020` builds a writer against real events rather than
hypothetical ones. Covers `identity/authentication-audit`.

```yaml
backlog_items:
  - id: TS-BL-017
    feature: identity-and-access
    depends_on: [TS-BL-014]
    status: not-started
```

- [ ] 5.1 Define the authentication event vocabulary — login success, login failure, logout,
      forced revocation, status change — as `AuditEvent`s constructed against the existing port
- [ ] 5.2 Attribute failed authentication to the authentication subsystem as actor with the
      attempted identity as target, satisfying the port's every-event-has-an-actor invariant
      without inventing a user (design D9)
- [ ] 5.3 Record a reason category on failures distinguishing rejected credentials, throttling, an
      unavailable identity provider, and denial for an unactivated identity
- [ ] 5.4 Assert by test that no password, provider token, session credential, or raw provider
      response body reaches an audit event or a log line
- [ ] 5.5 Assert every authentication event carries the edge-assigned correlation identifier, in
      both the pre-durable and durable states
- [ ] 5.6 Assert by test that every audit write this feature makes passes through the single audit
      interface, with no second path
- [ ] 5.7 Confirm the handover to `TS-BL-020`: the event vocabulary is the first input that
      substrate must accept, and installing the durable writer requires no change to any emitting
      surface

---

## Cross-feature seams this feature owes others

Not tasks — deliverables that another feature's items consume, listed so they are not treated as
internal detail and quietly changed.

| Seam | Defined by | Consumed by |
|---|---|---|
| `PermissionResolver` port, granting nothing until filled | `TS-BL-013` | `access-control-and-admin` `TS-BL-018` |
| The nine `roles` rows the permission matrix attaches to | `TS-BL-013` | `access-control-and-admin` `TS-BL-022` |
| Authentication event vocabulary through the audit port | `TS-BL-017` | `access-control-and-admin` `TS-BL-020` |
| The deactivation ownership invariant | `TS-BL-016` | `access-control-and-admin` `TS-BL-080` |
| Session context on every authenticated request | `TS-BL-014` | every feature that attributes a write |

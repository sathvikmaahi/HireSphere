## Purpose

Handing a departing user's owned work to a named successor, in the same administrative sitting as
their deactivation, without deleting anything, orphaning anything, or rewriting who did what. This
is the workflow that satisfies the invariant `identity-and-access` already enforces. Owned by
`TS-BL-080`.

## ADDED Requirements

### Requirement: Owned work is reassignable to a named successor

An authorized administrator SHALL be able to transfer the work a user owns to a named successor
user. The transfer SHALL identify the departing user, the successor, and the work transferred.

*Source: `exploration-notes.md` D.10.1, which closed this as `TS-BL-080` after `D08` ("a departing
recruiter who owned 12 open postings needs ownership reassignment") and `G-11` ("pairs with
ownership reassignment, still open") each asserted the work was needed and no backlog item ever
built it. `identity-and-access`'s `identity/user-activation` carries the invariant that owned work
"remains reassignable to another user"; this is the workflow that makes it true.*

#### Scenario: Departing user's work transferred

- **WHEN** an administrator reassigns a deactivated user's owned work to a named successor
- **THEN** the successor becomes the owner of that work
- **AND** the transfer is recorded with both users named

#### Scenario: Partial reassignment

- **WHEN** an administrator transfers some of a user's owned work and not the rest
- **THEN** the transferred items change owner and the remainder continues to name the original
  owner

### Requirement: Historical attribution is never rewritten

Reassignment SHALL change ownership going forward only. It SHALL NOT alter the actor recorded on
any historical activity, including audit records, shortlist reasons, interview notes, approvals, or
any other record of something a person did.

*Source: `identity-and-access`'s `identity/user-activation` — "SHALL NOT alter the actor recorded
on historical activity" — and `platform/audit-trail`'s immutability. `design.md` D11 records this
as the fixed boundary between the invariant and the workflow: the two must agree on it and neither
may relax it.*

#### Scenario: Historical actor preserved

- **WHEN** work is reassigned from one user to another
- **THEN** every historical record of an action the original user performed still names that user
  as the actor

#### Scenario: Audit records unaffected

- **WHEN** reassignment completes
- **THEN** no existing audit record is modified

### Requirement: Reassignment is a separate act from deactivation

Deactivation SHALL succeed whether or not reassignment follows. Reassignment SHALL be a distinct,
separately committed action, and SHALL NOT be a precondition of deactivating a user.

*Source: `design.md` D11. Coupling them would mean a deactivation blocked because no successor had
been chosen — and the leaver's session stays live in the meantime, which reintroduces the exact
`G-11` audit finding the pairing exists to close. `identity-and-access`'s `TS-BL-015` couples
deactivation to session revocation; ownership is the part that may legitimately wait.*

#### Scenario: Deactivation without reassignment

- **WHEN** an administrator deactivates a user and does not reassign their work
- **THEN** the deactivation succeeds and the sessions are revoked
- **AND** the owned work remains intact, still naming the deactivated user as its owner

#### Scenario: Reassignment after the fact

- **WHEN** an administrator reassigns the work of a user deactivated earlier
- **THEN** the reassignment succeeds without reactivating that user

#### Scenario: Presented together

- **WHEN** an administrator deactivates a user who owns work
- **THEN** the reassignment step is offered in the same flow, with the owned work enumerated

### Requirement: The successor must be a valid recipient

The named successor SHALL be an active user permitted to hold the work being transferred. A
transfer to a deactivated user, to no user, or to a user who could not hold the work SHALL be
refused.

*Source: `AUTHZ-008` least privilege — reassignment must not become a way to grant someone access
they were never given. `design.md` D11.*

#### Scenario: Successor is deactivated

- **WHEN** an administrator names a deactivated user as the successor
- **THEN** the transfer is refused

#### Scenario: No successor named

- **WHEN** a reassignment is submitted with no successor
- **THEN** the transfer is refused rather than leaving the work unowned

#### Scenario: Successor cannot hold the work

- **WHEN** the named successor lacks the permissions the transferred work requires
- **THEN** the transfer is refused, naming what the successor would need

### Requirement: Ownership relations are declared, and an unregistered relation fails loudly

The kinds of work that constitute ownership SHALL be held in a declared registry. Reassignment
SHALL cover every registered relation, and an ownership relation that is not registered SHALL raise
rather than be silently skipped.

*Source: `design.md` D11. Today's relations are assigned postings, open tasks and pending
approvals; Phase 2 and 3 add Applications, interview assignments and offer records. The
closed-registry discipline matches `runtime_config` and the feature flags, where an undeclared
value raises rather than resolving to nothing — and here the reason is sharper: a silently skipped
relation is orphaned work nobody notices.*

#### Scenario: All registered relations transferred

- **WHEN** a user's work is reassigned
- **THEN** every registered ownership relation for that user is transferred

#### Scenario: Unregistered relation encountered

- **WHEN** work is owned through a relation not present in the registry
- **THEN** the reassignment raises rather than completing with that work untransferred

#### Scenario: New relation registered by a later feature

- **WHEN** a feature introduces a new kind of owned work and registers its relation
- **THEN** reassignment covers it without modification to the reassignment workflow itself

### Requirement: Reassignment is audited and reasoned

Every reassignment SHALL require a reason and SHALL produce an audit record identifying the acting
administrator, the departing user, the successor, the reason, and each item transferred.

*Source: `AUTHZ-007`, `SEC-010`, and `platform/audit-trail`'s mandatory-reason rule. Ownership
transfer changes who is accountable for open work, which is exactly the category of change that
must be answerable later.*

#### Scenario: Reassignment without a reason

- **WHEN** an administrator submits a reassignment with no reason
- **THEN** the request is rejected with a field-level validation error and nothing is transferred

#### Scenario: Transfer recorded item by item

- **WHEN** a reassignment transfers several items
- **THEN** the audit trail identifies each item transferred, its previous owner, and its new owner

#### Scenario: Failed audit fails the transfer

- **WHEN** the audit record for a reassignment cannot be written
- **THEN** no ownership changes

### Requirement: Deletion remains refused in favour of deactivation and reassignment

Deleting a user who holds owned or authored work SHALL remain refused, including after their work
has been reassigned.

*Source: `identity-and-access`'s `identity/user-activation` — "Deletion refused". Reassignment
moves ownership; it does not remove the historical attribution that is the actual reason deletion
is refused (`RET-004`, `SEC-011`).*

#### Scenario: Deletion after reassignment

- **WHEN** deletion of a user is attempted after all their owned work has been reassigned
- **THEN** it is still refused, because historical attribution remains

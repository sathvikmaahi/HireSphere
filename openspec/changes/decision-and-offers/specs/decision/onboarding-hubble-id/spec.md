## Purpose

Onboarding completion and Hubble ID capture — the manually entered identifier that proves a hire
exists in the HR system, unique among recruited candidates, kept strictly separate from the Hubble
identifier a TalentSphere user logs in with. Owned by `TS-BL-068`.

## ADDED Requirements

### Requirement: The Hubble ID is captured manually and never provisioned

The Hubble ID SHALL be entered by a user against an onboarding record after onboarding completion.
This capability SHALL NOT create, update or request creation of a record in the HR system.

*Source: `D08`'s resolution — "**Manual capture, with optional validation. Provisioning is ruled out**"
— resting on §4.2 out-of-scope #10, "HRIS employee master creation", and `OFF-004`, "Recruiters shall
enter Hubble ID after onboarding completion". The recommended non-goal "Hubble write/provisioning" is
confirmed by the reference spec.*

#### Scenario: Entry after onboarding completion

- **WHEN** a Hubble ID is entered for a record whose onboarding status is complete
- **THEN** it is accepted

#### Scenario: Entry before onboarding completion

- **WHEN** a Hubble ID is entered for a record whose onboarding status is not complete
- **THEN** it is rejected, naming the outstanding onboarding completion

#### Scenario: Provisioning path sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no write to the HR system

### Requirement: The onboarding Hubble ID and the user login identifier are unrelated

The onboarding record's Hubble ID SHALL be a distinct field on a distinct record from the identifier
carried on a user. There SHALL be no foreign key between them, no shared uniqueness constraint, and no
code path that resolves one from the other.

*Source: `identity-and-access` `design.md` D3, which built this split and gives two reasons that are
constraints rather than preferences: candidates have zero system access, a deliberate non-goal in
`project.md`, so "a hired candidate acquiring a Hubble ID must not, by that fact, acquire a `users`
row"; and `D08` raises internal candidates and rehires, where "an employee applying internally already
holds a Hubble ID *at submission*" and legitimately appears as both a user and a candidate — "a
cross-entity uniqueness constraint would reject a real and expected case." `glossary.md` distinguishes
the two. `design.md` D6 consumes the split rather than re-deriving it.*

#### Scenario: Same value on both

- **WHEN** an onboarding record carries the same Hubble ID value as an existing user's login identifier
- **THEN** it is accepted

#### Scenario: No user record created

- **WHEN** a Hubble ID is captured for a recruited candidate
- **THEN** no user record is created or modified

#### Scenario: Cross-resolution sought

- **WHEN** the codebase is inspected for a path resolving a user from an onboarding Hubble ID, or the
  reverse
- **THEN** none exists

### Requirement: The Hubble ID is unique among recruited candidates

The Hubble ID SHALL be unique across records whose Application has reached recruited. A duplicate
within that set SHALL be blocked. Uniqueness SHALL NOT be enforced across records outside that set.

*Source: `OFF-006` — "Hubble ID uniqueness shall be validated among recruited candidates" — and
`OFF-007` — "Duplicate Hubble IDs shall be blocked". The scope is load-bearing: two non-recruited
records may legitimately hold the same value, typically a mistyped entry, and `design.md` D6 records
that the constraint is enforced at the same scope the requirement states rather than application code
enforcing a narrower rule than the schema.*

#### Scenario: Duplicate among recruited candidates

- **WHEN** a Hubble ID already held by a recruited candidate is entered on another record
- **THEN** it is blocked, naming the conflict

#### Scenario: Duplicate outside the recruited set

- **WHEN** two records whose Applications have not reached recruited hold the same Hubble ID
- **THEN** neither is blocked

#### Scenario: Global uniqueness sought

- **WHEN** the constraint is inspected
- **THEN** its scope is the recruited set and not the whole table

### Requirement: A duplicate is correctable only through a permission-gated administrative path

Correcting a blocked duplicate Hubble ID SHALL require a distinct permission, a mandatory reason and an
audit record. The correction SHALL NOT be available on the ordinary update path.

*Source: `OFF-007` — "Duplicate Hubble IDs shall be blocked **unless corrected through controlled
administrative procedure**". `ADM-005`'s grant/deny/unset cells and `AUTHZ-003`'s server-side
authorization make this a matrix decision on the administer action rather than a role comparison;
`D16` keeps administrators configuration-only by default, so the permission is the control.
`design.md` D6.*

#### Scenario: Correction with the permission

- **WHEN** a user holding the correction permission corrects a duplicate with a reason
- **THEN** it succeeds and an audit record names the actor, both records and the reason

#### Scenario: Correction without the permission

- **WHEN** a user without the correction permission attempts the same correction
- **THEN** it is rejected

#### Scenario: Correction on the ordinary path

- **WHEN** a duplicate is submitted through the ordinary Hubble ID entry path
- **THEN** it is blocked rather than corrected

### Requirement: Validation is optional, runs behind a port, and never blocks

Hubble ID validation SHALL run through a port with a stub implementation. Where no validator is
configured, capture SHALL succeed and the validated marker SHALL remain false. Where a validator is
configured and reports failure, the outcome SHALL be surfaced and SHALL NOT block capture, recruitment
or closure.

*Source: `D08`'s finding table — validation is "conditional, not guaranteed", confidence **Low**, with
§24's "Exact endpoint must be confirmed" and `OD-002` still open. §12.2 defines `hubble_id_validated` as
"True when validation available and successful", describing an optional outcome. `identity-and-access`
`design.md` D1's shape — a port whose stub is a shipped artifact rather than test scaffolding — is
followed; the port is separate from the login adapter because `design.md` D3 establishes these as two
integration points with two different open questions. `G-12`'s graceful degradation.*

#### Scenario: No validator configured

- **WHEN** a Hubble ID is captured with no validator configured
- **THEN** capture succeeds and the validated marker is false

#### Scenario: Validator reports failure

- **WHEN** a configured validator reports an identifier unknown
- **THEN** the outcome is surfaced and capture is not reversed

#### Scenario: Validator unreachable

- **WHEN** a configured validator is unreachable
- **THEN** capture succeeds and the validated marker is false

### Requirement: Recruited requires onboarding complete and a present, unique Hubble ID

An Application SHALL reach recruited only when its onboarding status is complete and its Hubble ID is
present and unique among recruited candidates. Reaching recruited SHALL fill the candidate's reserved
vacancy slot. Recruited SHALL NOT require the validated marker to be true.

*Source: `OFF-003` — "Candidate cannot be marked Recruited until onboarding status is Complete" — as a
necessary and not sufficient condition; `OFF-005` and `BR-019` — a candidate does not count toward
vacancy fulfilment without a valid Hubble ID; `G-13`'s slot machine, `reserved ──onboard + Hubble
ID──▶ filled`. `design.md` D6 records why "valid" resolves to present-and-unique rather than validated:
requiring the marker would make every finite posting in the product permanently uncloseable while
`OD-002` stays open. §11.2's separate `OnboardingComplete` state is the window `OFF-004`'s
after-completion entry happens in.*

#### Scenario: Both conditions met

- **WHEN** onboarding is complete and a unique Hubble ID is present
- **THEN** the Application reaches recruited and its reserved slot becomes filled

#### Scenario: Onboarding complete, no Hubble ID

- **WHEN** recruited is attempted with onboarding complete and no Hubble ID
- **THEN** it is rejected, naming the missing identifier

#### Scenario: Unvalidated Hubble ID

- **WHEN** recruited is attempted with a present, unique, unvalidated Hubble ID
- **THEN** it succeeds

#### Scenario: Validated marker as a gate sought

- **WHEN** the codebase is inspected for a gate on the validated marker
- **THEN** none exists on recruitment or closure

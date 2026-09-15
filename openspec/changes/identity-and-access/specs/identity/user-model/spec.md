## Purpose

The local user record and its role assignments — who exists in TalentSphere, which of the nine
roles they hold, and which of their attributes may never influence an access decision. This
capability owns role *identity* and stops precisely where role *capability* begins: what a role
may do is decided by the permission evaluator, which this capability declares a seam for and does
not implement. Owned by `TS-BL-013`.

## ADDED Requirements

### Requirement: Local user records are the authority for access

The system SHALL maintain a local user record for each known Hubble identity, created either by
administrative action or on first authentication, and SHALL treat local records as the authority
for access rather than the Hubble directory.

*Source: `AUTH-003`, `BR-001`, `ADM-001`. Hubble says who someone is; TalentSphere says whether
they are in.*

#### Scenario: Administrator pre-provisions a user

- **WHEN** an Application Administrator creates a local user for a known Hubble identity and
  activates it
- **THEN** that user is admitted on next login with the roles assigned

#### Scenario: Present in Hubble, absent locally

- **WHEN** an identity Hubble authenticates has no activated local user record
- **THEN** access is refused regardless of that identity's standing in the Hubble directory

### Requirement: User record content

A user record SHALL carry a system-assigned identifier, the Hubble user identifier, display name,
email address, status, and last successful login timestamp, together with creation and update
timestamps. The Hubble user identifier SHALL be unique across user records, and the email address
SHALL be unique across user records.

*Source: `reference/spec.md` §12.2 `users`, which specifies both a "Unique Hubble identity
reference" and a "Unique email address". The two uniqueness constraints answer different
questions and both hold: the Hubble identifier is what identity **maps on** (inherited D8 — email
is the claim most likely to change or be absent), while email uniqueness keeps a person
addressable by one record. Keying on the identifier is not a reason to stop enforcing the other.*

#### Scenario: Record created from a login

- **WHEN** a local user record is created following successful Hubble authentication
- **THEN** it carries the Hubble user identifier, display name and email from the returned claims
- **AND** its status reflects that it has not yet been activated

#### Scenario: Duplicate Hubble identifier rejected

- **WHEN** a second user record is created for a Hubble identifier that already has one
- **THEN** the write is rejected rather than producing two records for one identity

#### Scenario: Duplicate email rejected

- **WHEN** a user record is created carrying an email address another user record already holds
- **THEN** the write is rejected rather than producing two records sharing an email

### Requirement: The user identifier and the onboarding Hubble ID are separate

The Hubble user identifier held on a user record SHALL be independent of the Hubble ID captured on
an onboarding record. The two SHALL NOT share a uniqueness constraint, and possession of one
SHALL NOT create, imply, or require the other.

*Source: `glossary.md` — Hubble is a login-only integration here, and the onboarding Hubble ID is
"not the same integration point." `D08`, `OFF-004`, `OFF-006`/`OFF-007` own the onboarding side.
Candidates have zero system access by design (`project.md`), and `D08` raises internal candidates
and rehires who legitimately hold both.*

#### Scenario: A hire does not become a user

- **WHEN** a Hubble ID is captured against an onboarding record for a recruited candidate
- **THEN** no user record is created and no access is granted by that capture alone

#### Scenario: An employee applies internally

- **WHEN** a person holding a user record also appears as a candidate whose onboarding record
  carries a Hubble ID
- **THEN** both records are permitted to exist
- **AND** neither is rejected on the grounds that the identifier appears elsewhere

### Requirement: Nine roles exist as a closed vocabulary

The system SHALL define exactly nine roles — Recruiter, Practice Manager, Recruitment Manager,
Interviewer, Hiring Panel Member, Application Administrator, System Administrator, Auditor, and AI
Service Account — each with a stable key, a name, a description, a system-role marker, and a
status. A role key that is not one of the nine SHALL be rejected rather than accepted as unknown.

*Source: `C-02`, which amends `D01` from four roles to nine; `domain-model.md` "Access and
governance". The closed-vocabulary discipline matches the runtime-configuration and feature-flag
registries, where an undeclared value raises rather than resolving to nothing.*

#### Scenario: Role vocabulary enumerated

- **WHEN** the defined roles are enumerated
- **THEN** exactly the nine named roles are present, each with a stable key

#### Scenario: Undeclared role key

- **WHEN** a role assignment references a key outside the nine
- **THEN** the assignment is rejected

#### Scenario: AI Service Account is a role, not a person

- **WHEN** the AI Service Account role is examined
- **THEN** it is present as a first-class actor role
- **AND** it is not treated as an interactive identity that can authenticate through Hubble

### Requirement: Multiple roles per user

A user SHALL be assignable to more than one role. Effective permissions SHALL be the union of all
grants from all assigned roles, subject to explicit denials and to the most restrictive applicable
data scope.

*Source: `domain-model.md` `User ──N:M──▶ Role`; `AUTHZ-005` for denial precedence. The union and
precedence rules are stated here as the contract role assignment must support; the component that
computes them is `TS-BL-018`.*

#### Scenario: A manager who also interviews

- **WHEN** one user holds both the Practice Manager and Interviewer roles
- **THEN** the user receives the union of both roles' action grants
- **AND** where the two roles imply different data scopes for the same resource, the more
  restrictive scope applies

#### Scenario: Role assignment removed

- **WHEN** a role is unassigned from a user
- **THEN** grants that role contributed are no longer part of that user's effective permissions

### Requirement: Permission resolution is consumed, never computed here

This capability SHALL obtain a user's resolved permissions from the permission evaluator through a
declared interface, and SHALL NOT derive, infer, or cache any access decision from role
assignments. Until the evaluator is present, resolution SHALL yield no grants rather than an
assumed grant.

*Source: `config.yaml`'s standing rule — one permission evaluator, consulted by every page,
endpoint and background actor, with verdict and explanation from the same pass; `AUTHZ-001`
deny-by-default. `TS-BL-018` in `access-control-and-admin` owns the evaluator, and depends on
`TS-BL-014`, so it cannot ship alongside this capability. `design.md` D5 records the port.*

#### Scenario: Evaluator not yet present

- **WHEN** a user's permissions are resolved before the evaluator exists
- **THEN** the result is an empty set of grants
- **AND** the user is admitted to the application shell while every page and action is denied

#### Scenario: Evaluator installed later

- **WHEN** the permission evaluator becomes available
- **THEN** resolved permissions come from it
- **AND** no consumer of the resolution interface requires modification

#### Scenario: No second decision path

- **WHEN** the access decisions made anywhere in this capability are enumerated
- **THEN** none is derived from role assignments directly

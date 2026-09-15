## Purpose

The candidate record and the rules that decide whether an incoming resume belongs to someone already
known: five independent match signals, two auto-resolving cases, a review queue for the rest, a
reversible audited merge, and a remembered not-a-duplicate verdict. Also where the candidate-posting
Application link is created. Owned by `TS-BL-046`.

## ADDED Requirements

### Requirement: Candidate identity is derived only from deterministic fields

The signals used to establish or match a candidate's identity SHALL be the deterministically
extracted or manually entered contact fields and the resume file hash. Model-produced enrichment
SHALL NOT be an input to identity resolution.

*Source: `D18` — "dedup keys come from deterministic logic rather than model output. Candidate
identity should not depend on a model's mood"; `D23` — "match inputs come from the **deterministic**
half of parsing"; `CAN-004`.*

#### Scenario: Identity resolved

- **WHEN** identity resolution runs for a resume
- **THEN** it reads only the deterministic contact fields and the file hash

#### Scenario: Enrichment used as a signal

- **WHEN** a code path uses enriched skills, seniority or summaries as a match signal
- **THEN** the test suite fails

#### Scenario: Resolution before enrichment

- **WHEN** a resume has been parsed deterministically and never enriched
- **THEN** identity resolution runs and completes normally

### Requirement: Each match signal is evaluated independently

Identity resolution SHALL evaluate each of the following independently, so that any one can raise its
own outcome without another signal being present:

| Signal | Outcome |
|---|---|
| File hash matches, same candidate | Link to the existing resume; create no new version |
| File hash matches, different candidate | Review queue |
| Exact email match | Auto-merge |
| Name and phone match, emails differ | Review queue |
| Name, employer and role match, email and phone both differ | Review queue |
| No signal matches | New candidate |

*Source: `D23`, whose revision was made specifically because the fuzzy checks had been "tied to email
and would have missed a common real case — the same candidate applying with a personal email once and
a work email another time, with no other shared identifier surfacing."*

#### Scenario: Personal and work email, no other shared identifier

- **WHEN** a resume matches an existing candidate on name and phone while the emails differ
- **THEN** a review item is raised, without an email match being required to reach it

#### Scenario: Employer and role match only

- **WHEN** a resume matches an existing candidate on name, employer and role while both email and
  phone differ
- **THEN** a review item is raised

#### Scenario: No signal matches

- **WHEN** no signal matches any existing candidate
- **THEN** a new candidate record is created

#### Scenario: Signal evaluated as a secondary check

- **WHEN** a signal is implemented such that it can only fire when another signal has already matched
- **THEN** the test suite fails

### Requirement: Only an exact email match auto-merges

Auto-merge SHALL occur only on an exact email match. A file hash matching under a different candidate
SHALL raise a review item and SHALL NOT merge candidates.

*Source: `D23`'s table; `G-04`'s note that the hash is a **document-level** signal — "an identical
file proves two *documents* are the same, not that two *candidate records* are the same person — the
failure case being a recruiter uploading candidate A's PDF under candidate B's name."*

#### Scenario: Exact email match

- **WHEN** an incoming resume's email exactly matches an existing candidate's
- **THEN** the resume is attached to that candidate without human intervention, and the merge is
  audited

#### Scenario: Same file under a different name

- **WHEN** an uploaded file's hash matches a resume belonging to a different candidate
- **THEN** a review item is raised naming both candidates, and no merge occurs

#### Scenario: Same file under the same candidate

- **WHEN** an uploaded file's hash matches a resume already belonging to the same candidate
- **THEN** it links to the existing resume and no review item is raised

### Requirement: Every merge is reversible and audited

A candidate merge SHALL record both source records, the signal that caused it, the actor where a human
performed it, and a timestamp, and SHALL be reversible by an explicit un-merge action that restores
the separated records. Automatic merges SHALL be recorded with the same completeness as human ones.

*Source: `D23` — "merge is **reversible and audited** in all cases"; `config.yaml`'s audit rules;
`API-003`.*

#### Scenario: Merge audited

- **WHEN** any merge occurs, automatic or human
- **THEN** an audit record is written in the same transaction carrying both references, the signal
  and the actor, and containing no candidate personal data

#### Scenario: Merge reversed

- **WHEN** an authorized user reverses a merge
- **THEN** the separated candidate records are restored and the reversal is itself audited

#### Scenario: Audit write fails

- **WHEN** the audit record for a merge cannot be written
- **THEN** the merge fails and no records are combined

### Requirement: A not-a-duplicate verdict is remembered

Where a human decides a reviewed pair is not a duplicate, that verdict SHALL be recorded so the same
pair does not re-enter the review queue on a later submission.

*Source: `D23` — "a 'not a duplicate' verdict must be **remembered** (suppression list) so the same
pair does not re-queue on every submission."*

#### Scenario: Pair dismissed then resubmitted

- **WHEN** a pair judged not a duplicate matches again on a later submission
- **THEN** no review item is raised for that pair

#### Scenario: Suppression is per pair, not per candidate

- **WHEN** a candidate whose pair with one other candidate was suppressed matches a third candidate
- **THEN** a review item is raised for the new pair

#### Scenario: Suppression recorded

- **WHEN** a not-a-duplicate verdict is given
- **THEN** it is recorded with actor, timestamp and reason

### Requirement: A merge never silently alters an application

A candidate merge SHALL NOT merge, withdraw, re-stage or re-rank any application. Where a merge leaves
one candidate holding more than one application against a single posting, that condition SHALL be
recorded and surfaced for human resolution, and the merge SHALL take no further action on it.

*Source: `config.yaml` — "every state transition goes through the workflow service. No surface writes
a state field directly"; `project.md`'s central rule that material state changes rest with an
accountable human. `D23a`'s follow-on rule for which application survives is explicitly not decided —
see `design.md` D1(b), which records why it is excluded rather than invented here.*

#### Scenario: Merge produces two applications on one posting

- **WHEN** a merge results in one candidate holding two applications against the same posting
- **THEN** the condition is recorded and surfaced, and neither application is withdrawn, merged,
  re-staged or re-ranked

#### Scenario: Merge across different postings

- **WHEN** a merge results in one candidate holding applications against different postings
- **THEN** all of them survive unchanged, since a candidate holding several postings is ordinary

#### Scenario: Merge attempts a stage write

- **WHEN** a code path in the merge writes an application's stage field
- **THEN** the test suite fails

### Requirement: Identity resolution creates the candidate-posting application link

On resolving a resume to a candidate, this capability SHALL create the application record linking that
candidate to the resume's source posting, carrying the submitting recruiter and the submission source
type, unless a link between that candidate and that posting already exists.

*Source: `CAN-003` — "the upload pipeline shall create or update candidate profiles through
deduplication"; `CAN-005`, `CAN-007`, `G-10`; `domain-model.md`'s `Application` as candidate ×
posting. `design.md` D5 and D13 — the link cannot exist before the candidate does, and must exist the
moment it does.*

#### Scenario: New candidate resolved

- **WHEN** a resume resolves to a newly created candidate
- **THEN** an application record is created linking that candidate to the resume's source posting

#### Scenario: Existing candidate, new posting

- **WHEN** a resume for an existing candidate resolves against a posting they have no application for
- **THEN** a new application is created, and the candidate's existing applications are untouched

#### Scenario: Existing candidate, same posting

- **WHEN** a resume for an existing candidate resolves against a posting they already have an
  application for
- **THEN** no second application is created

#### Scenario: Source type recorded

- **WHEN** an application is created
- **THEN** it records a submission source type from the defined set, defaulting to the recruiter
  submission where none is supplied

### Requirement: A candidate cannot exist without the fields identity depends on

A candidate record SHALL NOT be created from a resume whose deterministic contact fields are absent.
Such a resume SHALL be held in a manual-entry state, and the candidate SHALL be creatable once a
permitted human supplies the fields.

*Source: `D18` — "manual entry required before the candidate record can be created, because dedup
depends on those fields"; `design.md` D5, D9, which records that the manual-entry path is how this
capability is verifiable rather than an optional extra.*

#### Scenario: Resume with no contact fields

- **WHEN** identity resolution is attempted for a resume with no email, phone or name
- **THEN** it is refused and the resume is held in the manual-entry state

#### Scenario: Fields supplied manually

- **WHEN** a permitted human supplies the contact fields
- **THEN** identity resolution runs against them exactly as it would against extracted values, and
  the entry is audited

### Requirement: A resume with no resolved candidate is not usable downstream

A resume that has not resolved to a candidate SHALL NOT be addressable as a resume version reference,
SHALL NOT be attachable to an application, and SHALL NOT be sent for enrichment.

*Source: `design.md` D5, which makes `candidate_id` nullable until resolution as a cited divergence
from §12.2 and states the two constraints that make the nullable column safe.*

#### Scenario: Unresolved resume referenced

- **WHEN** a caller attempts to use an unresolved resume as a resume version reference
- **THEN** the reference is refused

#### Scenario: Unresolved resume enriched

- **WHEN** enrichment is requested for an unresolved resume
- **THEN** it is refused

#### Scenario: Unresolved resume visible

- **WHEN** an authorized user views the intake surface
- **THEN** unresolved resumes are visible with their state and the reason they are unresolved

### Requirement: Contact fields are protected at rest and returned only with permission

Candidate contact fields SHALL be stored under approved sensitive-data controls, and SHALL be returned
by an API only to a caller whose verdict permits it. Absence of permission SHALL omit the fields
rather than failing the whole response.

*Source: `SEC-002`, `SEC-003`, `API-006`; `D16`, under which Administrators are configuration-only and
reach candidate personal data through audited break-glass rather than by default.*

#### Scenario: Permitted caller

- **WHEN** a caller permitted to see contact fields reads a candidate
- **THEN** the fields are returned

#### Scenario: Unpermitted caller

- **WHEN** a caller not permitted to see contact fields reads a candidate
- **THEN** the candidate is returned with the contact fields omitted

#### Scenario: Administrator by default

- **WHEN** an Application or System Administrator with no active break-glass grant reads a candidate
- **THEN** the contact fields are omitted

### Requirement: Candidates remain searchable and their history is preserved

A submitted candidate SHALL remain retrievable and searchable to authorized users subject to retention
and access rules, and a candidate's history SHALL be preserved whether or not they were selected for
any posting.

*Source: `CAN-006`, `BR-020`, `RET-001`, `RET-003`. The retention interval itself is configuration
(§29 item 12) and is open pending `OD-004`.*

#### Scenario: Candidate not selected

- **WHEN** a candidate is not selected for the posting they were submitted against
- **THEN** the candidate, their resume versions and their application history remain retrievable

#### Scenario: Search scoped by permission

- **WHEN** a user searches the candidate database
- **THEN** results are filtered by the evaluator's verdict as a query predicate rather than by
  discarding fetched rows

## Purpose

The gate that turns an AI draft into an evaluation of record — and the mechanism that keeps it honest
after `C-08` removed the note lock: a post-approval note edit supersedes the scorecard and requires
re-approval. Owned by `TS-BL-061`.

## ADDED Requirements

### Requirement: A scorecard remains Draft until an authorized human approves it

An AI-generated scorecard SHALL remain in Draft status until approved by an authorized human user.
Approval SHALL record the approver and the time. No approval SHALL be reachable from AI output or from
a service account.

*Source: `SCR-006`, `AI-001`, `AI-010`, `BR-007`, §12.2's `scorecards.approved_by` and `approved_at`,
§13.2's `POST /api/scorecards/{scorecardId}/approve`, and `ai-platform-governance`'s advisory-only
guarantee, under which `Approve` is ungrantable to the AI Service Account.*

#### Scenario: Unapproved scorecard

- **WHEN** a generated scorecard has not been approved
- **THEN** its status is Draft and it carries the AI-generated marking

#### Scenario: Approval recorded

- **WHEN** an authorized human approves a scorecard
- **THEN** the approver and timestamp are recorded and the AI-generated marking is cleared for the
  approved content

#### Scenario: Service account approval

- **WHEN** the AI Service Account requests approval
- **THEN** it is denied, and the `Approve` action is not grantable to it

#### Scenario: Unauthorized approval

- **WHEN** an actor without the `Approve` action on this surface requests approval
- **THEN** it is refused server-side and the control is absent in the interface

### Requirement: Human adjustment requires a mandatory reason and never deletes the AI output

A human reviewer MAY adjust a scorecard's comments or recommendations only with a mandatory reason.
The original AI output SHALL remain retrievable. The adjustment SHALL be recorded through the
platform's existing human-override record, not through a second mechanism.

*Source: `SCR-007`, `BR-016`, `RANK-006`'s companion rule that an override never deletes original AI
output, `G-06` (human override as a first-class record), and `ai-platform-governance`'s override
record — which `matching-and-ranking`'s board already consumes for ranking overrides.*

#### Scenario: Adjustment with a reason

- **WHEN** a reviewer adjusts a recommendation with a reason
- **THEN** the adjustment is recorded with actor, reason, timestamp, the original output reference and
  the human value

#### Scenario: Adjustment without a reason

- **WHEN** a reviewer attempts an adjustment with no reason
- **THEN** it is rejected

#### Scenario: Original output retrievable

- **WHEN** an adjusted scorecard is inspected
- **THEN** the original AI output is still retrievable

#### Scenario: Second override mechanism sought

- **WHEN** this capability is inspected for an override store of its own
- **THEN** none exists, and adjustments are written through the shared record

### Requirement: Divergence between approved scorecards and AI drafts is measurable from stored records

The rate at which approved scorecards diverge from the AI drafts they came from SHALL be computable
from stored override records alone, without re-deriving it from content comparison.

*Source: `exploration-notes.md`'s signature dashboard — **AI agreement rate**: "how often humans
shortlist the AI's top picks; how often approved scorecards diverge from AI drafts" — whose second half
is this capability's to make possible. `insight-and-reporting` computes it; what this capability owes
is that the premise is evidenced rather than asserted. `C-10`'s parallel argument for the ranking half.*

#### Scenario: Divergence enumerated

- **WHEN** override records for scorecard approvals are enumerated
- **THEN** the divergence rate is computable from them alone

#### Scenario: Approval without adjustment

- **WHEN** a scorecard is approved with no adjustment
- **THEN** that case is distinguishable from an approval with an adjustment

### Requirement: A note version created after approval supersedes the scorecards drawn from that note

When a new interview note version is created after a scorecard was approved, every approved scorecard
whose recorded note-version set names an earlier version of that note SHALL be set to `superseded` and
SHALL require re-approval. A scorecard whose recorded set does not name that note SHALL be unaffected.

*Source: `C-08`'s recorded open consequence — "With no lock, a note can change *after* the consolidated
scorecard drawn from it was approved, so an approved evaluation can silently stop matching its
evidence. The spec provides the tool but does not wire it up: `scorecards.status` already includes
**`superseded`**. Recommended follow-up: a note version created after scorecard approval marks that
scorecard `superseded` and requires re-approval." `exploration-notes.md` tracked that as **"Not yet
decided"**; `design.md` D3 decides it and records the two rejected alternatives (reintroduce the lock;
warn and leave the approval standing). §12.2's `scorecards.status` enum; `SCR-008`; `D24`'s surviving
purpose that an approved evaluation and its evidence agree.*

#### Scenario: Note edited after approval

- **WHEN** a new version is created of a note that an approved scorecard's recorded set names
- **THEN** that scorecard's status becomes `superseded` and it requires re-approval

#### Scenario: Unrelated note edited

- **WHEN** a new version is created of a note that the recorded set does not name
- **THEN** the scorecard remains approved

#### Scenario: Draft scorecard at the time of the edit

- **WHEN** the scorecard drawn from the note is still in Draft
- **THEN** it is not superseded, because there is no approval to invalidate

#### Scenario: Superseded scorecard is not silently current

- **WHEN** a superseded scorecard is read
- **THEN** its status is visible as superseded rather than presented as approved

### Requirement: Approval is refused where the draft's evidence has already moved

Approval SHALL be refused where the recorded note-version set of the draft names a version that is no
longer current, and the refusal SHALL direct the approver to regenerate.

*Source: `D24`'s interaction note, which survives its own reversal — "the AI drafts the scorecard from
notes that are still editable, so a panelist could edit notes *after* the AI draft is generated but
*before* approving it". `design.md` D9 records why this check exists separately from supersession:
superseding a scorecard one second after approving it would be technically consistent and would read
as a malfunction.*

#### Scenario: Evidence moved before approval

- **WHEN** approval is requested for a draft whose recorded note versions are no longer current
- **THEN** it is refused and the response identifies the notes that changed

#### Scenario: Evidence current

- **WHEN** approval is requested for a draft whose recorded note versions are all current
- **THEN** approval proceeds

### Requirement: Supersession preserves the prior approved version and its audit trail

Supersession SHALL NOT overwrite or delete the previously approved content. The prior approved version,
its approver, its timestamp and the note-version set it was approved against SHALL remain readable, and
re-approval SHALL create a new version.

*Source: `SCR-008` ("all scorecard changes shall be versioned and auditable"), §12.2's
`scorecards.version_number`, `BR-020`'s history preservation, and the principle `RANK-006` states for
overrides — the original is never deleted. `design.md` D9 records the required trail: approved at T1
against {v2, v1}, superseded at T2 by note v3, re-approved at T3 against {v3, v1}.*

#### Scenario: Prior approval readable

- **WHEN** a superseded scorecard is inspected
- **THEN** its prior approved content, approver, timestamp and note-version set remain readable

#### Scenario: Re-approval

- **WHEN** a superseded scorecard is re-approved
- **THEN** a new version is created and the superseded version remains readable

#### Scenario: Supersession is not a new version

- **WHEN** a scorecard is superseded without being re-approved
- **THEN** its version number is unchanged and only its status has moved

### Requirement: Re-approval presents what changed

Re-approval after supersession SHALL present the difference between the note versions the prior
approval was made against and the current ones, rather than requiring the approver to re-read
everything unaided.

*Source: `design.md` D3's stated proportionality measure, and `UI-003`'s requirement that the interface
make the state of the record legible. This mitigates the friction the supersede rule introduces without
weakening it: no edit is classified as immaterial, and the approver still approves explicitly.*

#### Scenario: Re-approval view

- **WHEN** an approver opens a superseded scorecard
- **THEN** the changed note versions and their differences are presented

#### Scenario: Materiality judgement sought

- **WHEN** this capability is inspected for a rule that classifies a note edit as too small to
  supersede
- **THEN** none exists

### Requirement: Approval and supersession are workflow transitions and are audited

Approval, supersession, rejection and re-approval SHALL each be executed through the workflow service
and SHALL write an audit record in the same transaction, carrying actor, action, target reference,
previous and new value, reason where required and the correlation identifier. Supersession performed by
the system SHALL be attributed to a service account and correlated to the triggering note-version
event. A failed audit write SHALL fail the operation.

*Source: `WF-001`, `WF-003`, `WF-004` (system-generated transitions attributed to a service account and
correlated to the triggering event), `WF-005`, `SCR-008`, §28 item 13 (scorecard generation, edit,
approval and rejection are named auditable events), `config.yaml`'s standing audit rule.*

#### Scenario: Approval audited

- **WHEN** a scorecard is approved
- **THEN** an audit record carries actor, action, target reference, previous and new value and the
  correlation identifier

#### Scenario: System supersession attributed

- **WHEN** the system supersedes a scorecard in response to a note-version event
- **THEN** the transition names the service account as actor and carries the triggering event's
  identifier

#### Scenario: Audit write fails

- **WHEN** the audit record for an approval cannot be written
- **THEN** the approval does not take effect

#### Scenario: Personal data in the audit record

- **WHEN** a scorecard audit record is inspected
- **THEN** it carries references and values, and no candidate personal data

### Requirement: The first approved scorecard advances the posting into scorecard review

Approving the first scorecard on a posting SHALL advance the posting's state from interviewing to
scorecard review. The advance SHALL occur only where the posting is in interviewing, SHALL be
attributed to the approving actor, and SHALL be a no-op for every subsequent approval, including a
re-approval after supersession.

*Source: §11.1's `Interviewing → ScorecardReview` transition, which `hiring-postings`' `TS-BL-038`
declares and no feature invoked — `decision-and-offers` `design.md` D9a records the investigation and
assigns this advance to the item that owns its triggering event; `design.md` D10a adopts it.
`WF-003`'s actor attribution and `WF-004`'s correlation. The advance is attributed to a human actor
inside that human's action, which is what `hiring-postings`' requirement that no transition fires
without an authenticated actor demands. The posting's state is `job_postings.status`, distinct from
the Application states in §11.2 and from `scorecards.status`. This advance is what makes
`decision-and-offers`' `scorecard_review → selection` advance reachable.*

#### Scenario: First approval on a posting

- **WHEN** the first scorecard is approved on a posting in interviewing
- **THEN** the posting advances to scorecard review, attributed to the approving actor

#### Scenario: Second approval

- **WHEN** a second candidate's scorecard is approved on the same posting
- **THEN** the posting's state is unchanged and no error is raised

#### Scenario: Re-approval after supersession

- **WHEN** a superseded scorecard is re-approved on a posting already past interviewing
- **THEN** no advance is attempted and the posting is not moved backwards

#### Scenario: Supersession does not move the posting

- **WHEN** a scorecard is superseded by a post-approval note edit
- **THEN** the posting's state is unchanged

#### Scenario: Advance without an actor

- **WHEN** the codebase is inspected for a posting advance that fires with no authenticated actor
- **THEN** none exists

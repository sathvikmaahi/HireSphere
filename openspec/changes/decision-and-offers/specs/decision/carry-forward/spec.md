## Purpose

Scorecard carry-forward — citing a consolidated scorecard from an earlier Application as prior
evidence for a new posting, under an attestation, with one confirmatory round still required. Owned
by `TS-BL-066`.

## ADDED Requirements

### Requirement: A scorecard is linked to a second Application, not copied or re-parented

Carry-forward SHALL create a link record between an existing consolidated scorecard and a second
Application, carrying the source Application, the attesting user, a timestamp, a typed reason and the
route taken. The scorecard SHALL NOT be duplicated, moved or re-parented, and the second Application
SHALL NOT acquire a scorecard of its own by this act.

*Source: `D17`'s consequence — "**`Scorecard` cannot be owned by a single application.** It needs to be
linkable to multiple applications — a link record carrying `carried_forward`, the source application,
the attesting user, timestamp, and reason." `domain-model.md`: "A `Scorecard` can link to **multiple**
Applications when priority-lane carry-forward applies (`D17`)." `interview-pipeline`'s
`interview/scorecard-consolidation` spec builds the readable, citable document and states explicitly
that no "carry-forward eligibility, attestation, or a link from a scorecard to a second Application"
exists there.*

#### Scenario: Carry-forward performed

- **WHEN** a scorecard is carried forward to a second Application
- **THEN** a link record exists carrying the source Application, attesting user, timestamp, reason and
  route

#### Scenario: Origin still identifiable

- **WHEN** a carried scorecard is read from the second Application
- **THEN** its origin Application, posting and approval remain identifiable

#### Scenario: Duplication sought

- **WHEN** the second Application is inspected for a scorecard of its own created by carry-forward
- **THEN** none exists

### Requirement: The carried scorecard is prior evidence, and one confirmatory round is still required

A carried scorecard SHALL be cited as prior evidence only. The Application receiving it SHALL still
complete one interview round for its own posting before it becomes eligible for its own scorecard or
for selection. Carry-forward SHALL NOT provide an entry point that bypasses interviewing for the new
posting.

*Source: `C-07`'s resolution of 2026-08-14, which amends `D17` and `D21`: "The carried consolidated
scorecard is cited as **prior evidence**; the candidate still completes **one interview round for the
new posting**. ... One round for the posting satisfies `SCR-001`, `BR-010`, `BR-011`, and `AI-009` — so
no spec rule is broken." Its explicit **Dropped** clause: "the non-round-1 entry point — candidates now
enter at a round, not at selection." `design.md` records that satisfying one of `C-07`'s reversals and
missing the other is the failure mode here.*

#### Scenario: Carry-forward with no round for the new posting

- **WHEN** selection is attempted for an Application whose only evidence is a carried scorecard
- **THEN** it is rejected, naming the outstanding confirmatory round

#### Scenario: Confirmatory round completed

- **WHEN** the Application completes one interview round for its own posting
- **THEN** it becomes eligible for its own scorecard and for selection

#### Scenario: Selection-stage entry point sought

- **WHEN** this capability is inspected for a path that admits an Application directly at selection
- **THEN** none exists

### Requirement: Same-job-description carry-forward is the fast path; a different description requires fuller attestation

Where the source and target postings share a job description, in any version, carry-forward SHALL
require a lightweight attestation carrying a short typed reason. Where they do not, carry-forward SHALL
require a fuller justification including an explicit acknowledgement of the competency difference.
Neither path SHALL be satisfiable without a typed reason.

*Source: `D21`'s decision — "same-JD carry-forward is the fast path; similar-but-different JDs go
through PM attestation as the exception" — and its recorded reconciliation of the open point it raised
against `D17`: "same-JD still records a lightweight attestation (one click plus a short reason), while
different-JD requires the fuller justification", because `D17` established that "the attestation *is*
the control" and must be a typed reason, not a checkbox.*

#### Scenario: Shared job description

- **WHEN** carry-forward is attempted between two postings sharing a job description
- **THEN** it succeeds on a short typed reason

#### Scenario: Different job description

- **WHEN** carry-forward is attempted between postings with different job descriptions
- **THEN** a fuller justification with a competency-difference acknowledgement is required

#### Scenario: Checkbox attestation sought

- **WHEN** either path is inspected for an attestation satisfiable without typed text
- **THEN** none exists

### Requirement: Eligibility expires after a configurable window, and expiry demotes rather than deletes

Carry-forward SHALL be unavailable where the source scorecard's approval is older than the configured
priority-lane window. Beyond that window the source scorecard SHALL remain readable as context. The
window SHALL be configurable with changes audited, and SHALL NOT be settable beyond the configured
data-retention period.

*Source: `D22`'s 90-day priority-lane TTL and its consequence — "After 90 days, a candidate still
resurfaces but as a **standard match** — prior scorecards become readable context rather than
carry-forward-eligible evidence. The lane does not delete people; it demotes them." `C-11`'s
resolution amends `D22` to make the windows configuration seeded at those values with changes audited,
and records the hard constraint: "the resurfacing window can never exceed the retention period set by
`OD-004`." `platform-core`'s audited runtime-configuration registry holds the value.*

#### Scenario: Within the window

- **WHEN** carry-forward is attempted from a scorecard approved 30 days ago
- **THEN** it is available

#### Scenario: Beyond the window

- **WHEN** carry-forward is attempted from a scorecard approved 200 days ago
- **THEN** it is refused and the scorecard remains readable as context

#### Scenario: Window changed

- **WHEN** the window is changed
- **THEN** the change is audited with actor and reason

#### Scenario: Window set beyond retention

- **WHEN** the window is set longer than the configured retention period
- **THEN** it is rejected

### Requirement: Mandatory criteria are re-evaluated for the new posting regardless of carried evidence

Mandatory criteria status SHALL be evaluated against the target posting's own requirements. Carried
evidence SHALL NOT satisfy, substitute for, or pre-populate that evaluation.

*Source: `C-07`'s closing clause — "**Reinforced by `G-01`:** mandatory criteria are re-evaluated for
the new posting regardless of carried evidence." `G-01`'s AI-proposes-human-confirms model applies to
the new posting's evaluation as to any other.*

#### Scenario: Carried candidate on a new posting

- **WHEN** a carried candidate is evaluated for the target posting
- **THEN** mandatory criteria status is computed against that posting's requirements

#### Scenario: Carried status reuse sought

- **WHEN** this capability is inspected for a path that copies mandatory criteria status forward
- **THEN** none exists

### Requirement: The route taken is recorded for every carry-forward

Every carry-forward SHALL record whether it took the shared-description path or the attested path, in
a form that supports comparing volumes between the two.

*Source: `OA-01`'s instrumentation clause — "the carry-forward path must record which route was taken
(same-JD vs. attestation) so this comparison is possible at all. That is a slice 7 reporting
requirement, not something to add later." `D21`'s consequence: "the route taken is instrumented for
`OA-01`."*

#### Scenario: Shared-description carry-forward

- **WHEN** a carry-forward takes the shared-description path
- **THEN** the route is recorded as such

#### Scenario: Volumes compared

- **WHEN** carry-forward records are aggregated
- **THEN** the two routes' volumes are separable

### Requirement: Carry-forward operates on a selected-not-offered Application however it was surfaced

Carry-forward SHALL be available for an Application whose earlier selection was tagged selected not
offered, independent of how that Application was surfaced for the new posting. This capability SHALL
NOT implement resurfacing, precedence ordering, or a suggestion pool.

*Source: `SEL-005` retains the selection reason and marks the candidate for future resurfacing
precedence; `D.10` assigns precedence ordering to `resurfacing-and-communications`' `TS-BL-076`, which
depends on `TS-BL-064` rather than on this item — so the lane reads what this feature writes.
`design.md` D8 records the split: the tag and the state are this feature's, precedence is
`TS-BL-076`'s.*

#### Scenario: Manually surfaced candidate

- **WHEN** a selected-not-offered candidate is brought to a new posting by any route
- **THEN** carry-forward is available on the same terms

#### Scenario: Precedence logic sought

- **WHEN** this capability is inspected for a ranking of resurfaced candidates against each other
- **THEN** none exists

#### Scenario: Suggestion pool sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no suggestion pool or resurfacing engine

### Requirement: Carry-forward is audited as a decision, not as a data operation

Every carry-forward, and every refusal, SHALL be recorded through the platform's audit writer with the
actor, both Applications, the route, the reason and the outcome.

*Source: `D17`'s governance note — "this is the one place in the system where a hiring decision rests
on evidence gathered for a different role. The attestation *is* the control, so it must require a typed
reason, not a checkbox." `API-003` requires audit on every material write; `G-06` establishes the
human-decision record as first-class.*

#### Scenario: Carry-forward audited

- **WHEN** a carry-forward is performed
- **THEN** an audit record exists naming the actor, both Applications, the route and the reason

#### Scenario: Refusal audited

- **WHEN** a carry-forward is refused for staleness or a missing confirmatory round
- **THEN** the refusal and its cause are recorded

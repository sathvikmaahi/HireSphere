## Purpose

One scorecard per Application, consolidated across every interview round, with each panelist's own
recommendation preserved and disagreement surfaced rather than averaged away. Owned by `TS-BL-062`.

## ADDED Requirements

### Requirement: One scorecard per Application, consolidated across all rounds

An Application SHALL have at most one active scorecard, keyed to the Application rather than to an
interview round, drawing on the notes of every completed round.

*Source: `C-05`'s resolution of 2026-08-14, which **amends `D11`**: "One AI-drafted consolidated
scorecard per application, `Draft` until human-approved (`SCR-006`)". §12.2's
`scorecards.candidate_application_id`. `domain-model.md` records the same shape: "Scorecard (one per
Application, consolidated across all rounds — see `C-05`)". `D11`'s original per-round choice is
superseded.*

#### Scenario: Second round completed

- **WHEN** a second round completes for an Application that already has a scorecard
- **THEN** the existing scorecard is the one that reflects both rounds, and no second scorecard is
  created

#### Scenario: Per-round scorecard sought

- **WHEN** this capability is inspected for a scorecard keyed to an interview round
- **THEN** none exists

#### Scenario: Two Applications for one candidate

- **WHEN** one candidate has Applications against two postings
- **THEN** each Application has its own scorecard and neither draws on the other

### Requirement: Per-panelist evaluation is preserved and individually attributable

A consolidated scorecard SHALL present each contributing panelist's own recommendation and ratings,
attributed to that panelist and to the round they conducted. Consolidation SHALL NOT replace per-
panelist evaluation with a single combined value.

*Source: `C-05`'s reconciliation finding — "their `interview_notes` already carries `recommendation`
and `ratings_json` **per round, per panelist** — that *is* a per-round panelist evaluation, just named
a note rather than a scorecard. Adopting their model does not lose per-panelist attribution."
`D11`'s stated concern was attribution, and this is how it survives the reversal.*

#### Scenario: Three panelists

- **WHEN** a consolidated scorecard drawn from three panelists' notes is read
- **THEN** each panelist's own recommendation and ratings are shown, attributed to them and to their
  round

#### Scenario: Attribution after consolidation

- **WHEN** a panelist's contribution is traced from the consolidated scorecard
- **THEN** it resolves to their own note version, author identity and round

### Requirement: Disagreement is surfaced, never resolved arithmetically

Where contributing panelists' recommendations differ, the consolidated scorecard SHALL render the
disagreement explicitly. The overall recommendation SHALL NOT be derived by any arithmetic over
panelist recommendations — no mean, median, majority rule or weighting.

*Source: `D11`'s remaining open item — "the rule for consolidating conflicting panelist scorecards into
one selection decision — currently 'the PM decides'" — and `C-05`'s explicit ground for reversing
`D11`: consolidation as specified by `SCR-004`/`SCR-005` "explicitly [is] not" averaging. `D11`
rejected consolidation because it was framed as averaging; if consolidation resolved conflicts
arithmetically, `D11`'s objection would be correct and `C-05` reversed on a false premise.
`design.md` D4.*

#### Scenario: Conflicting recommendations

- **WHEN** panelists recommend `strong_yes`, `no` and `maybe`
- **THEN** the scorecard renders the three recommendations as a disagreement rather than a single
  computed value

#### Scenario: Arithmetic derivation sought

- **WHEN** the codebase is inspected for a path deriving the overall recommendation by arithmetic over
  panelist recommendations
- **THEN** none exists

#### Scenario: Unanimous recommendations

- **WHEN** all panelists recommend the same value
- **THEN** the individual recommendations are still individually shown

### Requirement: The overall recommendation is AI-proposed and human-decided

The consolidated scorecard's overall recommendation SHALL be proposed by the AI draft, SHALL carry the
AI-generated marking until approved, and SHALL be changeable only through the mandatory-reason
adjustment path. The approving human is the accountable decider.

*Source: `SCR-003` (overall recommendation is a fixed dimension), §12.2's
`scorecards.overall_recommendation`, `AI-001`, `UI-004`, `SCR-007`, `BR-016`, and `D11`'s "the PM
decides", retained and made concrete. `design.md` D4 records why the field is proposed rather than left
null: it exists in the schema and something must fill it, and an AI-proposed, marked, reason-guarded
value is the shape every other AI output in this product takes.*

#### Scenario: Proposed value

- **WHEN** a consolidated draft is generated
- **THEN** an overall recommendation is proposed carrying the AI-generated marking

#### Scenario: Approver diverges

- **WHEN** the approver sets a different overall recommendation
- **THEN** a reason is mandatory and the divergence is recorded through the shared override record with
  the AI-proposed value still visible

#### Scenario: Recommendation outside the scale

- **WHEN** an overall recommendation outside `strong_yes`, `yes`, `maybe`, `no`, `strong_no` is
  supplied
- **THEN** it is rejected

### Requirement: Adding a round makes the consolidated scorecard regenerable, not automatically approved

When a further round completes after a scorecard exists, the scorecard SHALL be regenerable to include
the new round's evidence, and the regenerated content SHALL require approval on its own terms. No
completion of a round SHALL by itself change an approval state to approved.

*Source: `SCR-006`, `SCR-008`, `AI-010`, `BR-007`. Consolidating across rounds means the evidence set
grows over time, and an approval granted against a two-round evidence set is not an approval of a
three-round one.*

#### Scenario: Third round completed after approval

- **WHEN** a third round completes after the scorecard was approved for two
- **THEN** the scorecard is regenerable and the regenerated content requires its own approval

#### Scenario: Approval by round completion

- **WHEN** a round completes
- **THEN** no scorecard becomes approved as a result

### Requirement: The scorecard summary is registered as a vector source type

This capability SHALL register `scorecard_summary` against the platform's vector source-type
registration seam. It SHALL NOT implement a second embedding pipeline or vector store, and SHALL NOT
build a search surface over interview history.

*Source: `VEC-001`'s four source types; `matching-and-ranking` `design.md` D2, which built the pipeline
source-type-driven with all four enum values present so that "Phase 3 adds a registration and no schema
change" and named this capability as the registrant of this type. `C-01`'s scope note records semantic
search over interview history as "a genuine capability that is **not** among the original 19 features"
and it remains out of scope and open. `design.md` D6.*

#### Scenario: Type registered

- **WHEN** a scorecard's summary content is available
- **THEN** vector records are produced under the `scorecard_summary` source type through the existing
  pipeline

#### Scenario: Second pipeline sought

- **WHEN** this capability is inspected for its own embedding pipeline or vector store
- **THEN** none exists

#### Scenario: Search surface sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no search over interview history

### Requirement: The consolidated scorecard is a citable artifact for a later Application

The consolidated scorecard SHALL be readable and citable as prior evidence from outside its own
Application, with its origin Application, posting and approval identifiable. This capability SHALL NOT
implement carry-forward eligibility, attestation, or a link from a scorecard to a second Application.

*Source: `C-05`'s second stated ground for reversing `D11` — "`D17` carry-forward is far simpler
carrying one consolidated document than N per-round ones"; `domain-model.md`'s note that a Scorecard can
link to multiple Applications when priority-lane carry-forward applies; `D17`, `D21` and `C-07`, all of
which are `decision-and-offers`' `TS-BL-066`. What this capability owes the carrier is a readable,
attributable document — not the carrying mechanism.*

#### Scenario: Cited from elsewhere

- **WHEN** a scorecard is read from outside its own Application
- **THEN** its origin Application, posting, approval and evidence remain identifiable

#### Scenario: Carry-forward mechanism sought

- **WHEN** this capability is inspected for carry-forward eligibility, attestation or a multi-Application
  link
- **THEN** none exists

### Requirement: A merge involving a decided Application is refused rather than resolved

Where a candidate merge leaves one candidate holding two Applications against one posting, and both
Applications' stages fall within the shortlist-to-scorecard range, the Application further along the
lifecycle SHALL survive, with ties broken by earliest creation; the other SHALL be transitioned to a
terminal state with a reason naming the merge, retaining its rounds, notes and scorecard. Where either
Application has progressed beyond this range, the merge SHALL be refused and surfaced for human
resolution. Notes SHALL NOT be re-parented under any circumstances.

*Source: `D23a`'s follow-on rule — "which application survives, which ranking score is retained (or is
a re-rank forced), and what happens if the two sat at different stages, all need explicit rules" —
deferred to this feature by name twice: `candidate-intake` `design.md` D1(b) ("an application's stage
belongs to `interview-pipeline`") and `matching-and-ranking` `design.md` D5 ("the surviving-application
half ... depends on the application stage, which is `interview-pipeline`'s"). `BR-020`, `WF-005`,
`WF-006`. `INT-007` binds a note to its round and author, so re-parenting would forge attribution.
`design.md` D7 records what is answered here, what is refused, and why the later stages belong to
`decision-and-offers`.*

#### Scenario: Both Applications within this feature's range

- **WHEN** a merge leaves two Applications on one posting, one interviewed and one shortlisted
- **THEN** the interviewed Application survives and the other is transitioned to a terminal state with
  a reason naming the merge

#### Scenario: Same stage

- **WHEN** both Applications sit at the same stage
- **THEN** the earlier-created Application survives

#### Scenario: Application beyond this feature's range

- **WHEN** either Application has progressed beyond the scorecard stage
- **THEN** the merge is refused and the condition is surfaced for human resolution

#### Scenario: History of the non-surviving Application

- **WHEN** the non-surviving Application is inspected
- **THEN** its rounds, notes and scorecard remain readable and attached to it

#### Scenario: Note re-parenting sought

- **WHEN** the codebase is inspected for a path that moves a note to a different round or Application
- **THEN** none exists

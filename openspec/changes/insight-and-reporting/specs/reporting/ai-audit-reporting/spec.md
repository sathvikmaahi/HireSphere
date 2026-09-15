## Purpose

`§20.1`'s AI Audit Dashboard — the Auditor's governance view over AI activity: what ran, on which prompt
and model versions, how often it failed, how often a human diverged from it, and what was flagged.
Aggregates over substrates that are already PII-free at source, and adds no second store. Owned by
`TS-BL-074`.

## ADDED Requirements

### Requirement: The dashboard presents the seven elements `§20.1` specifies

The AI Audit Dashboard SHALL present AI runs, prompt versions, model versions, outputs, human
overrides, failure rates, and flagged outputs.

*Source: `reference/spec.md` `§20.1`'s AI Audit Dashboard row, quoted in full — the row carries no
requirement ID, so it is cited directly. `§14.2`'s Reports screen, whose capabilities include "AI
audit". `§4` item 18 — "Dashboards, reports, audit logs, AI run logs, and export controls" as in-scope.
`§33`'s Phase 4 scope, which names the AI audit center and model governance.*

#### Scenario: Dashboard rendered

- **WHEN** an authorized Auditor loads the dashboard
- **THEN** all seven elements are present

#### Scenario: Grouped by version

- **WHEN** activity is grouped by prompt template version or model version
- **THEN** the grouping uses the versions recorded on the run records themselves

#### Scenario: Failure rate over a period

- **WHEN** a failure rate is requested for a period
- **THEN** it is computed from run records with failed status retained over that period

### Requirement: The dashboard aggregates over the existing run log and audit trail, adding no second store

This capability SHALL compute every element from the AI run records, disclosure records, override
records and audit records the platform already stores. It SHALL NOT create an aggregation store, a
second override record, a second run log, or a copy of any of them.

*Source: `ai-platform/ai-run-logging`'s `Run log access and search` requirement, whose own source note
states that "the reporting surface built on this search is `insight-and-reporting`'s `TS-BL-074`; this
requirement is the queryable substrate, not a dashboard", and which already makes runs filterable by
family, template version, model version, status, safety flag and time range.
`access-control/audit-review`'s audit search and correlation retrieval. The run log's append-only
requirement, whose scenario requires a later-known fact to be a separate linked record rather than an
amendment — a reporting aggregate is a read, so it writes nothing at all. `design.md` D11.*

#### Scenario: Aggregation source

- **WHEN** an element is computed
- **THEN** it is derived from the stored run, disclosure, override and audit records

#### Scenario: Second store sought

- **WHEN** this capability's persistence surface is enumerated
- **THEN** it contains no aggregation table, no override record and no run log of its own

#### Scenario: Reporting writes nothing

- **WHEN** any element of this dashboard is computed
- **THEN** no run, disclosure, override or audit record is created, modified or removed

### Requirement: Human override rate is computed from the existing override records

The dashboard SHALL present the rate at which human decisions diverged from AI output over a requested
period, computed from the stored override records alone.

*Source: `G-06` — human override as a first-class record, whose text states that the divergence "becomes
measurable — which is what makes the AI agreement rate dashboard possible".
`ai-platform/ai-run-logging`'s `Human override records` requirement and its `Divergence is measurable`
scenario: "WHEN override records are queried over a period THEN the rate at which humans diverged from
AI output can be computed from the stored records alone". `RANK-006`, `BR-016`. `§20.2` item 10 — human
override of AI output is an audited event. Two producers verify this path end-to-end for this consumer:
`interview-pipeline`'s task 6.14 and `matching-and-ranking`'s task 5.11.*

#### Scenario: Divergence rate over a period

- **WHEN** the override rate is requested for a period
- **THEN** it is computed from override records within that period with no content comparison performed

#### Scenario: Scorecard divergence

- **WHEN** the rate at which approved scorecards diverged from their AI drafts is requested
- **THEN** it is computed from override records, and an approval with no adjustment is distinguishable
  from one with an adjustment

#### Scenario: Original output retained

- **WHEN** an override contributing to the rate is inspected
- **THEN** the original AI output remains retrievable, the override having deleted nothing

### Requirement: Shortlist concordance is computed against the ranking version active at disposition time

The dashboard SHALL present how often shortlist dispositions selected the AI's highest-ranked candidates.
For each disposition, the AI rank SHALL be taken from the ranking version that was active for that
posting at the disposition's recorded time, resolved from the audited ranking-activation event. It SHALL
NOT be taken from the currently active ranking version.

*Source: `design.md` D7. The chain is: `platform/workflow-engine`'s `Transition records` requirement
timestamps every disposition, and `interview/shortlisting` makes every disposition a workflow transition;
`platform/audit-trail`'s `Audit on every material write` and `Audit record content` requirements record
each ranking activation with its previous and new values and an immutable timestamp — the activation of a
ranking version being the single writer of the application's rank per `matching-and-ranking`'s
`design.md` D4, and its task 3.13 stating the audit obligation outright;
`access-control/audit-review`'s `Audit search` requirement makes that log queryable by target, action and
time range in chronological order; and `matching/ranking-score`'s `Prior board retrievable` scenario
returns a past version's "full entry set... as it stood", with `No ranking entry SHALL be updated in
place` guaranteeing it reads unchanged. `matching-and-ranking`'s `design.md` D4 reaches the same
conclusion. The measure originates in the **unratified** "Signature dashboards" recommendation.*

#### Scenario: Concordance over a period

- **WHEN** shortlist concordance is requested for a period
- **THEN** each contributing disposition is scored against the ranking version active at that
  disposition's recorded time

#### Scenario: Board re-ranked after the disposition

- **WHEN** a posting is re-ranked after a disposition was made
- **THEN** that disposition's contribution is unchanged, having been scored against the version that was
  active when it was made

#### Scenario: Recomputation after a re-rank

- **WHEN** a past period's concordance is recomputed after any number of subsequent re-ranks
- **THEN** it returns the same value

#### Scenario: Active-version approximation sought

- **WHEN** this capability is inspected for a concordance computed against the currently active ranking
  version
- **THEN** none exists

### Requirement: Dispositions with no active ranking version are excluded from concordance

A disposition made when the posting had no active ranking version SHALL be excluded from the concordance
denominator. It SHALL NOT be counted as discordant.

*Source: `matching/ranking-score`'s requirement that each posting have **at most** one active ranking
version, which permits zero — a posting never ranked, or a candidate shortlisted before the first ranking
completed. Its `Posting never ranked` scenario makes that state distinguishable from a score of zero.
`design.md` D7 records why exclusion rather than discordance: counting a decision the AI never informed
would measure the wrong thing, in the direction that flatters the AI's absence.*

#### Scenario: Posting never ranked

- **WHEN** a disposition was made on a posting with no ranking version active at that time
- **THEN** it is excluded from the denominator and not counted as discordant

#### Scenario: Excluded population reported

- **WHEN** concordance is presented
- **THEN** the count of excluded dispositions is reported alongside it rather than left implicit

### Requirement: Dispositions scored against a stale ranking entry are reported distinguishably

Where a disposition is scored against a ranking entry marked stale, its contribution SHALL be
distinguishable from one scored against a current entry.

*Source: `matching/ranking-score`'s staleness requirement, which marks an entry whose recorded resume
version or ranking-relevant posting input has moved on, and whose `Stale entry readable` scenario makes
staleness visible and names the version the entry was computed against. `G-03`'s insufficiency posture —
returning the limitation rather than a confident number. `design.md` D7.*

#### Scenario: Stale entry contributed

- **WHEN** a disposition's contributing ranking entry is marked stale
- **THEN** its contribution is reported distinguishably from contributions scored against current entries

#### Scenario: Stale contributions not pooled silently

- **WHEN** concordance is presented over a period containing stale contributions
- **THEN** the share scored against stale entries is visible rather than absorbed into the headline rate

### Requirement: Concordance reports the horizon its substrate supports

Concordance SHALL be computed only over periods for which the underlying ranking versions are retained.
Where a requested period extends beyond that, the surface SHALL report the horizon rather than return a
rate computed over partly disposed boards.

*Source: `matching/ranking-score`'s retention requirement, which disposes ranking versions on configured
policy and records the disposal, and whose `Disposal does not orphan a run` scenario confirms disposal is
real. `platform/audit-trail`'s retention is governed separately, so activation events can outlive the
boards they name. `OD-004` leaves both intervals to policy. `design.md` D7 — a rate over a period whose
boards are partly gone is a silently wrong number, the failure class Sprint 0's handover names as the
dangerous one.*

#### Scenario: Period within the horizon

- **WHEN** concordance is requested for a period whose ranking versions are all retained
- **THEN** it is computed and returned

#### Scenario: Period beyond the horizon

- **WHEN** a requested period extends earlier than the retained ranking versions
- **THEN** the surface reports the horizon and does not return a rate over the unretained portion

#### Scenario: Activation event outlives its board

- **WHEN** a ranking-activation audit record is retained but the ranking version it names has been
  disposed of
- **THEN** the affected period is treated as beyond the horizon rather than scored as no-active-version

### Requirement: The surface is read-only and available to the Auditor without candidate personal data

This surface SHALL be read-only. It SHALL be available to the Auditor role, and SHALL disclose no
candidate personal data. Following a reference out of an aggregate SHALL require permission on the
referenced record.

*Source: `C-02`'s Auditor — "read-only on audit and AI logs, structurally unable to modify recruitment
decisions"; `TS-BL-022` task 5.7's seeded posture. `ai-platform/ai-run-logging`'s `Reference-only
inputs` requirement, which keeps resume text and contact data out of run records precisely so this
access is not a back door, and its guarantee that the log is readable "by the Auditor role without
granting access to candidate personal data". `access-control/audit-review`'s reading-discloses-no-PII
requirement. `design.md` D11 — the substrate is PII-free at source, so this surface inherits the
boundary rather than re-implementing it. Expressing Auditor access to this surface while denying the
candidate-derived dashboards needs page-catalog granularity that does not yet exist; `design.md` D4.*

#### Scenario: Auditor reads the dashboard

- **WHEN** an Auditor loads the dashboard
- **THEN** it renders with governance metadata and no candidate personal data

#### Scenario: Auditor attempts a write

- **WHEN** the Auditor role is evaluated for Create, Edit, Delete, Approve or Assign on this surface or
  any record it references
- **THEN** access is denied

#### Scenario: Drill-through to a run

- **WHEN** a reader follows an aggregate to an individual run or audit record
- **THEN** they reach the existing run-log or audit search surface, with permission evaluated there

#### Scenario: Unauthorized access

- **WHEN** a user without permission on this surface calls its endpoint directly
- **THEN** the request is denied

### Requirement: Small-cohort suppression applies to per-candidate AI activity

An aggregate whose contributing population falls below the configured floor SHALL be suppressed on this
surface as on any other, including where the population is the runs relating to a single candidate or a
single actor.

*Source: `reporting/read-model`'s suppression requirement, restated here because the substrate being
PII-free at source makes this the surface where suppression looks unnecessary — an aggregate over one
candidate's runs is still an aggregate over one person, and combined with a posting filter it can
identify them without any personal-data column being read. `PRV-001`. `design.md` D11.*

#### Scenario: Runs for a single candidate

- **WHEN** an aggregate's contributing runs relate to a single candidate
- **THEN** it is suppressed under the read model's floor

#### Scenario: Single actor

- **WHEN** an override aggregate's contributing population is a single actor
- **THEN** it is suppressed, so the surface does not become a per-person performance measure

### Requirement: Export uses the existing audit export, not the report export path

Exporting from this surface SHALL use the audited, classification-labelled audit export owned by
`access-control/audit-review`. This capability SHALL NOT route through the report export job type.

*Source: `§20.2` item 16 audits "report **and** audit exports" as two distinct things; `SEC-012`.
`access-control/audit-review`'s `Audited, permission-controlled export` requirement already provides
Export gating, a classification label, an export audit event and the guarantee that exported content
carries no candidate personal data. `design.md` D11 — two export paths, correctly, because this surface
exports audit and AI-run content while the operational dashboards export candidate-derived aggregates.*

#### Scenario: Export from this surface

- **WHEN** an authorized user exports from this surface
- **THEN** the audit export path produces it, carrying its classification label and its own audit event

#### Scenario: Report export path sought

- **WHEN** this capability is inspected for a registration against the report export job type
- **THEN** none exists

#### Scenario: Export without permission

- **WHEN** a user without Export permission requests an export from this surface
- **THEN** the request is denied

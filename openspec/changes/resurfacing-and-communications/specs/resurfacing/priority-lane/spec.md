## Purpose

Ranks selected-but-not-offered candidates ahead of standard matches within the `MatchSuggestion`
pool `resurfacing/resurfacing-engine` builds, reading the selection tag `decision-and-offers`
already writes rather than creating a second tagging mechanism. Owned by `TS-BL-076`.

## ADDED Requirements

### Requirement: A selected-not-offered candidate is flagged and ranked within the pool

Where a candidate carrying `selection_tag = selected_not_offered` on a prior Application appears
among `MatchSuggestion`s for a different, open posting, the system SHALL set `priority_lane = true`
on that suggestion and rank it ahead of standard matches for the same posting.

*Source: `glossary.md` — "Candidates who were selected but never offered get precedence in that
resurfacing (the priority lane)." `BR-015` — "A previously selected but not offered candidate shall
receive precedence in future relevant resurfacing lists." `SEL-005` — "Selected Not Offered
candidates shall retain selection reason and shall be prioritized for future resurfacing."
`design.md` D6.*

#### Scenario: Selected-not-offered candidate resurfaces

- **WHEN** a candidate tagged `selected_not_offered` on a prior posting appears in a
  `MatchSuggestion` for a different open posting
- **THEN** that suggestion is flagged `priority_lane = true` and ranked ahead of standard matches

#### Scenario: This capability creates no tag

- **WHEN** the codebase is inspected for a path in this capability that writes `selection_tag` or
  `selected_not_offered`
- **THEN** none exists — the tag is written only by `decision-and-offers`

### Requirement: Priority-lane precedence is subject to the target posting's mandatory criteria

A candidate's `selected_not_offered` history SHALL NOT rank them ahead of a candidate who meets the
target posting's mandatory criteria while they do not.

*Source: `RANK-011` — "Prior selected-not-offered status, subject to mandatory criteria checks" is
listed among permitted ranking signals with that qualifier attached. `BR-015`'s identical
qualifier. `design.md` D6.*

#### Scenario: Lane candidate fails mandatory criteria

- **WHEN** a `selected_not_offered` candidate does not meet the target posting's mandatory criteria
  and a standard match does
- **THEN** the standard match is not ranked behind the lane candidate on that basis alone

#### Scenario: Lane candidate meets mandatory criteria

- **WHEN** a `selected_not_offered` candidate meets the target posting's mandatory criteria
- **THEN** the priority-lane ranking in the first requirement applies

### Requirement: Priority-lane status expires on an audited TTL and demotes rather than removes

A candidate's `priority_lane` flag SHALL clear once the configured priority-lane TTL elapses since
their `selected_not_offered` tagging, after which they remain in the resurfacing pool as a standard
match.

*Source: `D22` as amended by `C-11` — seeded 90-day priority-lane TTL, configurable through the
audited runtime-configuration registry. "After 90 days, a candidate still resurfaces but as a
standard match — prior scorecards become readable context rather than carry-forward-eligible
evidence. The lane does not delete people; it demotes them." `design.md` D6.*

#### Scenario: Within the TTL

- **WHEN** a candidate's `selected_not_offered` tagging is within the configured TTL
- **THEN** the priority-lane ranking applies

#### Scenario: TTL elapsed

- **WHEN** a candidate's `selected_not_offered` tagging exceeds the configured TTL
- **THEN** `priority_lane` clears, and the candidate remains eligible as a standard match through
  `resurfacing/resurfacing-engine`

#### Scenario: TTL change is audited

- **WHEN** the priority-lane TTL's configured value is changed
- **THEN** the change requires a mandatory reason and is recorded in the audit trail

### Requirement: This capability surfaces lane candidates; it does not perform carry-forward

Ranking a candidate into the priority lane SHALL NOT itself invoke scorecard carry-forward or
mandate a confirmatory interview round. Carry-forward eligibility SHALL be evaluated by the
capability that owns it when the candidate is later shortlisted.

*Source: `decision-and-offers` `design.md` D8 — "`TS-BL-066` builds carry-forward as a mechanism
that operates on a selected-not-offered Application however it was surfaced... `TS-BL-076` builds
the lane that surfaces candidates into it. Neither depends on the other's internals." `glossary.md`'s
carry-forward description — carrying a prior scorecard forward "subject to completing one
confirmatory interview round." `design.md` D6.*

#### Scenario: Lane candidate promoted and later shortlisted

- **WHEN** a priority-lane `MatchSuggestion` is promoted to an Application and later shortlisted
- **THEN** carry-forward eligibility is evaluated by the existing carry-forward capability, not by
  this one

#### Scenario: Carry-forward logic sought here

- **WHEN** this capability's surface area is inspected for scorecard carry-forward or confirmatory-
  round logic
- **THEN** none exists

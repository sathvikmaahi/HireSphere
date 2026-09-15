## Purpose

Builds the `MatchSuggestion` pool that keeps candidates who don't fit their submitted posting from
being lost — evaluating both a newly opened posting against the existing candidate database and a
newly closed posting's bench against other open postings — and gates every suggestion behind
Practice Manager review before it can become a real Application. Owned by `TS-BL-075`.

## ADDED Requirements

### Requirement: A newly opened posting is evaluated against the existing candidate database

When a posting opens, the system SHALL evaluate the existing candidate database against it through
the `resurfacing` AI prompt family and SHALL create a `MatchSuggestion` for each candidate the
evaluation surfaces.

*Source: §16.1's AI Capability Matrix — "Candidate Resurfacing \| New posting opened \| Existing
candidate database, prior outcomes \| Existing candidate list and priority lane \| Practice Manager
review required." §25's "Existing Candidate Resurfacing \| Posting open" background job row.
`hiring-postings` `design.md` D7, which publishes `posting.opened` with `TS-BL-075` named as one of
two independent subscribers alongside `matching-and-ranking`'s `TS-BL-052`. `design.md` D1.*

#### Scenario: Posting opens with matching history

- **WHEN** a posting opens and an existing candidate's profile matches its requirements
- **THEN** a `MatchSuggestion` is created referencing that candidate and the newly opened posting

#### Scenario: Publication failure does not block opening

- **WHEN** the `posting.opened` event fails to deliver to this capability
- **THEN** the posting remains open, and the evaluation is retried as a background job

#### Scenario: Independent of the ranking engine's own registration

- **WHEN** `matching-and-ranking`'s ranking engine and this capability both register against
  `posting.opened`
- **THEN** each runs its own evaluation, and neither depends on the other's completion or result

### Requirement: A newly closed posting's bench is evaluated against other open postings

When a posting closes, the system SHALL evaluate that posting's terminated bench Applications
against other currently open postings through the `resurfacing` AI prompt family and SHALL create a
`MatchSuggestion` for each candidate the evaluation surfaces.

*Source: `decision-and-offers` `design.md` D8 — the closure cascade's fourth consequence, "move
bench to the resurfacing pool," published through `posting.closed` with `TS-BL-075` named as the
intended consumer, carrying "the posting reference, the closure kind, and references to the
applications this feature just terminated." `design.md` D1.*

#### Scenario: Posting closes with an unresolved bench

- **WHEN** a posting closes as filled with candidates still at earlier stages
- **THEN** each terminated bench Application's candidate is evaluated against other open postings,
  and a `MatchSuggestion` is created for each match found

#### Scenario: No other open postings

- **WHEN** a posting closes and no other posting is open
- **THEN** no `MatchSuggestion` is created, and the evaluation is not treated as a failure

#### Scenario: Publication failure does not block closure

- **WHEN** the `posting.closed` event fails to deliver to this capability
- **THEN** the posting remains closed, and the evaluation is retried as a background job

### Requirement: A suggestion carries prior outcome, mandatory criteria status, and AI provenance

Every `MatchSuggestion` SHALL carry a prior-outcome summary, a mandatory-criteria status, and a
reference to the `AIRun` that produced it, with each claim labeled by evidence source.

*Source: §16.3's Prompt Template Registry — the `resurfacing` template family's required output
format is "Candidate list with prior outcome and mandatory criteria status." `glossary.md`'s
evidence-source labeling requirement. `domain-model.md`'s `RankingScore` provenance tuple reasoning,
applied identically. `design.md` D2.*

#### Scenario: Suggestion created with provenance

- **WHEN** a `MatchSuggestion` is created
- **THEN** it references the `AIRun` that produced it, and that run's resume, posting, template and
  model versions are recoverable from the reference

#### Scenario: Every claim is source-labeled

- **WHEN** a `MatchSuggestion`'s prior outcome or mandatory-criteria status is read
- **THEN** each carries a label identifying it as resume-sourced, interview-sourced,
  scorecard-sourced, or a human decision

### Requirement: A suggestion becomes a real Application only on explicit human promotion

A `MatchSuggestion` SHALL be promotable to a real Application only by an authorized human actor. No
automated process SHALL create an Application from a `MatchSuggestion`.

*Source: §16.1's Human Gate for this capability — "Practice Manager review required." `AI-010` —
the system shall not automatically select, offer, hire, onboard, or close candidates based on AI
output alone; `project.md`'s governing rule extends the same posture to creating the Application
that begins that path. `domain-model.md`'s description of `MatchSuggestion` as "promoted into a real
`Application` only when a human acts on it." `design.md` D2.*

#### Scenario: Practice Manager promotes a suggestion

- **WHEN** an authorized Practice Manager promotes a `MatchSuggestion`
- **THEN** a real Application is created on the target posting, and the suggestion's review state
  becomes `promoted`

#### Scenario: Automatic promotion sought

- **WHEN** the codebase is inspected for a path that creates an Application from a `MatchSuggestion`
  with no human actor
- **THEN** none exists

#### Scenario: Dismissal requires a reason

- **WHEN** a `MatchSuggestion` is dismissed
- **THEN** it requires a mandatory reason, and its review state becomes `dismissed`

### Requirement: A promoted Application writes the existing resurfacing source type

Promoting a `MatchSuggestion` SHALL write `source_type = existing_database` on the resulting
Application, using the field and vocabulary `candidate-intake` already defines.

*Source: `G-10` — "`candidate_posting_applications.source_type`:
`recruiter \| referral \| job_board \| campus \| agency \| direct_application \| existing_database`.
The `existing_database` value is resurfacing, which makes the resurfacing-yield metric computable
rather than inferred." `candidate-intake`'s `TS-BL-046` task 6.12 creates the field; this capability
writes into it rather than defining a parallel classification. `insight-and-reporting`'s `design.md`
D6, which reads this exact column for its resurfacing-yield metric. `design.md` D4.*

#### Scenario: Promotion writes the existing field

- **WHEN** a `MatchSuggestion` is promoted
- **THEN** the resulting Application's `source_type` is `existing_database`

#### Scenario: No parallel classification

- **WHEN** the Application schema is inspected for a resurfacing-specific source field other than
  `source_type`
- **THEN** none exists

### Requirement: A promoted Application is an ordinary Application from that point on

A promoted Application SHALL be visible to every existing posting-scoped surface — ranking,
shortlisting, interview, scorecard, selection — through the same code path a directly submitted
Application uses, with no resurfacing-specific branch downstream of promotion.

*Source: `RANK-007` — "Existing candidate matches shall appear before or alongside new submissions
when a posting opens" — satisfied by this capability's independent `posting.opened` subscription
(this spec's first requirement) rather than a merged list; §14.2 keeps *Existing Candidate
Resurfacing* and *AI Ranking Board* as distinct screens. `design.md` D2.*

#### Scenario: Promoted Application appears on the ranking board

- **WHEN** a promoted Application's target posting is open
- **THEN** the Application is eligible for that posting's ranking board exactly as any other
  Application on the posting is

#### Scenario: No resurfacing-specific downstream branch

- **WHEN** the shortlisting, interview, or scorecard capabilities are inspected for logic
  conditioned on `source_type = existing_database`
- **THEN** none exists

### Requirement: Suggestions expire on an audited, configurable recency window

A `MatchSuggestion` SHALL drop out of the active pool once its candidate's resume exceeds the
configured resurfacing recency window, unless a newer resume version resets the window.

*Source: `D22` as amended by `C-11` — seeded 12-month resurfacing recency window, configurable
through the audited runtime-configuration registry, capped by the retention period `OD-004` sets. A
resume older than the window "drops out of the resurfacing pool unless a new resume version
arrives, which resets the clock." `design.md` D2.*

#### Scenario: Resume within the window

- **WHEN** a candidate's most recent resume is within the configured recency window
- **THEN** that candidate remains eligible for evaluation under both triggers

#### Scenario: Resume outside the window

- **WHEN** a candidate's most recent resume exceeds the configured recency window
- **THEN** that candidate is excluded from new `MatchSuggestion` creation until a newer resume
  version arrives

#### Scenario: Window change is audited

- **WHEN** the resurfacing recency window's configured value is changed
- **THEN** the change requires a mandatory reason and is recorded in the audit trail

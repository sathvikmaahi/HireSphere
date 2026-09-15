## Purpose

The one question this feature asks twice — where is work stuck? `§20.1`'s SLA Aging Dashboard and the
cross-posting closure-readiness view, sharing a single aging-and-blocker projection computed from
`decision-and-offers`' closure predicate and the platform's configured aging thresholds. Owned by
`TS-BL-073`.

## ADDED Requirements

### Requirement: One aging-and-blocker projection serves every reader

This capability SHALL own a single projection carrying, per posting and per Application, the stage ages
`§20.1`'s SLA Aging row names, the configured threshold each is measured against, and the closure
blockers. Every surface presenting an age, a threshold breach, or a closure blocker SHALL read this
projection.

*Source: `§20.1`'s SLA Aging Dashboard row — "posting age, shortlist age, interview scheduling age,
feedback delay, offer delay, onboarding delay, and closure blockers" — and its Practice Manager and
Recruitment Manager rows, which name closure blockers and aging respectively.
`reporting/read-model`'s one-aggregation-path requirement. `NFR-009`. `design.md` D1(b) and D5 record
why the SLA Aging Dashboard is absorbed here rather than split across two items.*

#### Scenario: Projection read by several surfaces

- **WHEN** the workload dashboard's closure blockers and the pipeline oversight dashboard's aging are
  rendered
- **THEN** both read this projection and report values consistent with this surface

#### Scenario: Second projection sought

- **WHEN** the reporting surfaces are inspected for a second stage-age, threshold or closure-blocker
  computation
- **THEN** none exists

#### Scenario: Terminal postings included

- **WHEN** the projection is computed
- **THEN** postings in terminal states are included, so historical ages remain readable

### Requirement: Aging thresholds are read from audited configuration and never defined here

Threshold values SHALL be read from the platform's audited runtime configuration. This capability SHALL
NOT define, default, or hard-code a threshold value.

*Source: `§29` item 9 — "Aging thresholds for postings, interviews, feedback, offer, and onboarding" as
configurable without code changes. `platform-core`'s `platform/notifications` and its task 5.11, which
makes aging thresholds runtime-configurable through the audited path — `design.md` D1(b) records that
this is why the apparent `TS-BL-005` overlap is not one: that item owns the threshold, this one owns the
view. `hiring-postings` seeds the posting thresholds and `interview-pipeline`'s task 2.14 seeds the
interview and feedback thresholds, both explicitly deferring the views to this feature. `D22`, which
caps configurable windows by the retention period.*

#### Scenario: Threshold applied

- **WHEN** an age is compared against its threshold
- **THEN** the threshold value comes from audited configuration

#### Scenario: Threshold changed

- **WHEN** an authorized administrator changes a threshold
- **THEN** the change is audited with a mandatory reason and the next projection reflects it without a
  deployment

#### Scenario: Hard-coded threshold sought

- **WHEN** this capability is inspected for a threshold value in source
- **THEN** none exists

#### Scenario: Threshold unset

- **WHEN** a threshold has no configured value
- **THEN** the age is reported without a breach determination rather than against an invented default

### Requirement: Aging drives reporting only, and no workflow transition

No state transition SHALL be triggered by an age or a threshold breach computed here. This capability
SHALL present conditions and SHALL NOT act on them.

*Source: `interview-pipeline`'s task 2.14, which asserts no transition in that feature is driven by an
aging threshold because "aging drives `insight-and-reporting`'s SLA views, not the state machine".
`decision/posting-closure`'s requirement that closing is actor-initiated and does not fire "on a timer,
a threshold, a data condition or a configuration change", with a scenario asserting no such path
exists. `project.md`'s rule that every material hiring decision rests with an accountable human.
`WF-001`.*

#### Scenario: Threshold breached

- **WHEN** a posting or Application breaches an aging threshold
- **THEN** the condition is reported and no state changes

#### Scenario: Transition path sought

- **WHEN** this capability is inspected for a path that performs a state transition
- **THEN** none exists

### Requirement: Cross-posting closure readiness calls the closure predicate rather than re-deriving it

Closure readiness SHALL be computed by invoking the fulfilment predicate owned by
`decision/posting-closure`, evaluated across postings. This capability SHALL NOT implement a second
fulfilment computation.

*Source: `decision/posting-closure`'s requirement that "the checklist and the guard on the closure
action SHALL be computed from the same predicate", whose scenario inspects the codebase for a fulfilment
computation used by only one of the two and fails if it finds one — this surface is a third reader.
`CLS-004`'s closure checklist showing missing Hubble IDs and blocking workflow states; `CLS-001`'s
fulfilment condition; `G-13`. `§13.2` exposes only the per-posting
`GET /api/postings/{postingId}/closure-checklist`, so `design.md` D8 records the cross-posting route as
an addition with reasoning.*

#### Scenario: Readiness across postings

- **WHEN** closure readiness is requested across postings
- **THEN** each posting's readiness equals what its own closure checklist reports

#### Scenario: Report agrees with the action

- **WHEN** this surface reports a posting ready to close
- **THEN** the closure action for that posting succeeds

#### Scenario: Blockers named by kind

- **WHEN** a posting is not ready to close
- **THEN** the blocking condition is named by kind — missing Hubble ID, or an unresolved salary, offer
  or onboarding state

#### Scenario: Second fulfilment computation sought

- **WHEN** this capability is inspected for its own fulfilment computation
- **THEN** none exists

### Requirement: Blocked candidates are counted, not identified

Closure blockers SHALL be reported as counts and conditions with references. Candidate identity SHALL
NOT be resolved on this surface.

*Source: `reporting/read-model`'s projection and drill-through requirements; `D16`. Recorded explicitly
here because a closure blocker is inherently about a specific person — a named candidate missing a
Hubble ID — which makes this the element most likely to acquire a helpful name resolution. `CLS-004`'s
"shall show missing Hubble IDs" is satisfied at the posting's own checklist, where permission on the
Application is evaluated.*

#### Scenario: Missing Hubble ID reported

- **WHEN** a posting has candidates who completed onboarding with no Hubble ID
- **THEN** the count and the condition are reported without candidate names

#### Scenario: Identifying the candidate

- **WHEN** a reader needs to know which candidate is blocked
- **THEN** they navigate to that posting's closure checklist, where access is evaluated on the
  Application

### Requirement: The surface is rendered from existing shared components with no new visual primitive

The view SHALL be built from the design system's existing tokens, primitives, page templates and
dense-data-table pattern, and SHALL introduce no chart, sparkline or visualization primitive.

*Source: as the other reporting surfaces — `AGENTS.md`'s reuse bar, `design-spec.md` `§1.6`, `S.5`, and
`design.md` D2, which also records the in-cell magnitude bar as considered and declined for aging
columns specifically, on the ground that a shared cell renderer belongs to the design system's table
pattern rather than to a consuming feature.*

#### Scenario: Ages rendered

- **WHEN** stage ages and breaches are presented
- **THEN** they are rendered as values and table rows, with breaches carried by the shared badge and
  status-surface components

#### Scenario: New primitive sought

- **WHEN** this capability's component surface is enumerated
- **THEN** it contains no chart, sparkline, magnitude bar or other visualization primitive, and no
  locally defined color or spacing value

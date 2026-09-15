## Purpose

`§20.1`'s Recruitment Manager Dashboard — the oversight lens `C-02` gives that role: who is carrying
what, how the funnel is converting, and where it is slowing down. Carries resurfacing yield, whose
substrate exists two phases before the feature that fills it. Owned by `TS-BL-072`.

## ADDED Requirements

### Requirement: The dashboard presents the seven elements `§20.1` specifies

The Recruitment Manager dashboard SHALL present recruiter workload, postings, candidates submitted,
shortlist conversion, interview scheduling, offer progress, and aging.

*Source: `reference/spec.md` `§20.1`'s Recruitment Manager Dashboard row, quoted in full — the row
carries no requirement ID, so it is cited directly. `§13.2`'s
`GET /api/reports/recruiter-workload`, described as "Recruitment manager pipeline view". `C-02`'s
Recruitment Manager, "oversight above Recruiter: workload, pipeline, closure readiness". `§9.1`'s role
description — "Oversees recruiters, postings, pipelines, offer progress, closure readiness, and
recruitment dashboards".*

#### Scenario: Dashboard rendered

- **WHEN** an authorized Recruitment Manager loads the dashboard
- **THEN** all seven elements are present

#### Scenario: Workload attributed by assignment

- **WHEN** recruiter workload is presented
- **THEN** each recruiter's load is derived from posting assignment rather than from any per-person
  judgement recorded elsewhere

#### Scenario: Conversion over a period

- **WHEN** shortlist conversion is requested for a period
- **THEN** it is computed over Applications dispositioned within that period, including those on
  postings since closed

### Requirement: The aging element reads the shared projection and computes no second one

The dashboard's aging element SHALL read the projection owned by `reporting/closure-and-aging`. This
capability SHALL NOT compute stage ages or threshold breaches itself.

*Source: `reporting/read-model`'s one-aggregation-path requirement. `§20.1` lists "aging" on this row
and gives the SLA Aging Dashboard its own row, which `design.md` D1(b) resolves by making the aging
projection a single owned thing with several readers rather than two implementations. This is the
dependency edge `D.10` does not carry, added under `D.11` and recorded in `design.md` D5.*

#### Scenario: Aging shown

- **WHEN** the aging element is rendered
- **THEN** its values equal those the closure-and-aging surface reports for the same filters

#### Scenario: Second aging computation sought

- **WHEN** this capability is inspected for its own stage-age or threshold computation
- **THEN** none exists

### Requirement: Resurfacing yield is computed from application source type

The dashboard SHALL present resurfacing yield as the share of hires whose Application source type
records them as sourced from the existing candidate database. It SHALL be computed from the Application
record's source type and SHALL NOT depend on the resurfacing engine or the suggestion pool.

*Source: `G-10`'s adopted application source type, whose `existing_database` value `G-10` states "is
resurfacing, which makes the resurfacing-yield metric computable rather than inferred".
`candidate-intake`'s `candidate/candidate-identity` and its task 6.12, which create the Application
record carrying `CAN-007`/`G-10`'s source type in Phase 2; its `design.md` D13(2) records that no other
item creates that table. `domain-model.md`'s Application source values. The metric originates in the
**unratified** "Signature dashboards" recommendation — `design.md` D6 states that status and why the
metric is nonetheless buildable now with no Phase 5 dependency.*

#### Scenario: Yield with resurfaced hires present

- **WHEN** hires exist whose Application source type records the existing candidate database
- **THEN** the yield is the share of hires those represent

#### Scenario: No resurfaced applications yet

- **WHEN** no Application carries the existing-database source type
- **THEN** the surface reports that no resurfaced applications exist, distinctly from reporting a zero
  percent yield

#### Scenario: Resurfacing engine absent

- **WHEN** this capability is inspected for a dependency on the resurfacing engine, the suggestion pool
  or the priority lane
- **THEN** none exists

#### Scenario: Metric provenance shown

- **WHEN** resurfacing yield is presented
- **THEN** it is labelled as deriving from a recommendation that has not been through an explicit
  decision

### Requirement: Oversight is aggregate and carries no candidate identity

Every element SHALL be presented as an aggregate over the reporting projection. Recruiter identity MAY
be shown, as recruiters are internal users; candidate identity SHALL NOT.

*Source: `reporting/read-model`'s projection and drill-through requirements. `D16`. `D02`/`C-03` —
Practice is a data attribute for filtering and reporting, not a permission boundary, and internal user
attribution is what workload oversight is for. The asymmetry is deliberate: `project.md` records
candidates as having zero system access and their data as the thing being protected, while a recruiter's
workload is the subject of the oversight `C-02` assigns this role.*

#### Scenario: Recruiter named

- **WHEN** recruiter workload is presented
- **THEN** recruiters are named as the internal users they are

#### Scenario: Candidate not named

- **WHEN** any element counts or ranks candidates
- **THEN** no candidate name or contact detail appears

#### Scenario: Small cohort in a breakdown

- **WHEN** a per-recruiter or per-practice breakdown produces a cohort below the configured floor
- **THEN** that cell is suppressed under the read model's suppression rule

### Requirement: The dashboard is rendered from existing shared components with no new visual primitive

The dashboard SHALL be built from the design system's existing tokens, primitives, page templates and
dense-data-table pattern, and SHALL introduce no chart, sparkline or visualization primitive.

*Source: as `reporting/workload-dashboard`'s corresponding requirement — `AGENTS.md`'s reuse bar,
`design-spec.md` `§1.6`, `S.5`, and `design.md` D2. Restated here because conversion and aging are the
two elements in the product most likely to attract a chart, so the constraint needs to be checkable on
this surface specifically rather than inherited implicitly.*

#### Scenario: Conversion rendered

- **WHEN** shortlist conversion is presented
- **THEN** it is rendered as values and table rows rather than as a chart

#### Scenario: New primitive sought

- **WHEN** this capability's component surface is enumerated
- **THEN** it contains no chart, sparkline or visualization primitive, and no locally defined color or
  spacing value

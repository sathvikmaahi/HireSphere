## Purpose

What a reporting query may return, and what it may never return. This is the boundary that keeps an
aggregate surface from becoming the product's PII back door — by row-level drill-through, by
aggregates over cohorts small enough to identify a person, or by an export carrying fields the screen
omits. Owned by `TS-BL-071`, inherited by every reporting surface after it.

## ADDED Requirements

### Requirement: Reporting queries execute against a projection with no candidate-identifying columns

Every reporting query SHALL execute against a declared reporting projection. The projection's column
set SHALL NOT include candidate names, contact fields, address data, resume text or extracted resume
content, or the free-text bodies of reasons and notes. Where a candidate, Application or resume must be
referenced, it SHALL be referenced by opaque identifier only.

*Source: `D16`, whose consequence list states the constraint as a schema property — "a schema
constraint on slice 1, not a UI filter". `glossary.md`'s evidence-source labelling entry, which states
the purpose plainly: this exists "so an Admin's audit-log access (config-only) doesn't become a PII
back door". `access-control/audit-review`'s corresponding read-side requirement, which this mirrors for
aggregates rather than for records. `design.md` D3.*

#### Scenario: Projection columns enumerated

- **WHEN** the reporting projection's column set is enumerated
- **THEN** no column resolves to a candidate personal-data field, and the test suite fails if one is
  added

#### Scenario: Candidate referenced in a report

- **WHEN** a reporting result must refer to a specific candidate
- **THEN** the candidate appears as an opaque identifier with no name or contact detail alongside it

#### Scenario: Personal data requested explicitly

- **WHEN** a caller requests a reporting endpoint with any parameter, filter or expansion that would
  return candidate personal data
- **THEN** the request does not return it, whether by omitting the field or by rejecting the parameter

### Requirement: Drill-through is a navigation evaluated on the target, not a report-side join

A reporting result MAY carry a reference that lets a reader navigate to the record behind a number. That
navigation SHALL be resolved by the feature owning the target record, with the permission evaluator
evaluating access to that record on its own terms. The reporting surface SHALL NOT join to, embed, or
return the target's personal data on the reader's behalf.

*Source: `D16`. `access-control/audit-review`'s `Resolving a reference requires its own permission`
scenario, which this reproduces for reporting. `AUTHZ-003`/`AUTHZ-004` — server-side evaluation on
every page and endpoint, with a direct API call failing even where the UI hides the control. `design.md`
D3, vector 1.*

#### Scenario: Reader follows a drill-through

- **WHEN** a reader follows a reference out of a reporting result
- **THEN** access to the referenced record is evaluated on that record's own terms and denied without
  the required permission

#### Scenario: Aggregate names a blocked candidate

- **WHEN** a report shows a count of candidates blocked on a workflow condition
- **THEN** the count is returned without candidate identity, and identifying who requires navigating to
  the owning screen

#### Scenario: Report-side join sought

- **WHEN** the reporting surface is inspected for a query joining the projection to candidate personal
  data
- **THEN** none exists

### Requirement: Aggregates over cohorts below the configured floor are suppressed

An aggregate whose contributing population is smaller than a configured minimum SHALL be reported as
below the reporting threshold rather than as a value. The minimum SHALL be held in the platform's
audited runtime configuration.

*Source: `D16`'s reasoning that a config-only boundary must not be "decorative"; `PRV-001`'s
data-minimisation posture. `§29`'s pattern of runtime-configurable operational values, and
`platform-core`'s runtime configuration store with its audited change path and mandatory reason.
`§20.1` specifies no cohort rule at all, so `design.md` D3 records this as a decision this design makes
rather than one it inherits.*

#### Scenario: Cohort of one

- **WHEN** an aggregate's contributing population is a single individual
- **THEN** the value is reported as below the reporting threshold and the underlying number is not
  returned

#### Scenario: Cohort above the floor

- **WHEN** an aggregate's contributing population is at or above the configured minimum
- **THEN** the value is returned

#### Scenario: Floor changed

- **WHEN** an authorized administrator changes the minimum
- **THEN** the change is audited with a mandatory reason and subsequent aggregates apply the new value

#### Scenario: Suppression applies to the export

- **WHEN** a suppressed aggregate is included in an export
- **THEN** it is suppressed in the exported content exactly as on screen

### Requirement: Suppression is applied to the complement that would recover it by subtraction

Where suppressing an aggregate leaves its value recoverable by subtracting the published sibling values
from a published total, the recovering values SHALL be suppressed as well.

*Source: `design.md` D3, vector 2 — suppression applied only to the small cell is defeated by one
subtraction, which would make the previous requirement decorative in exactly the way `D16` warns
against. No external source specifies this; it is recorded as this design's own decision.*

#### Scenario: Differencing attack

- **WHEN** a total and all but one of its contributing parts would be published, and the omitted part
  is suppressed
- **THEN** the value cannot be recovered by subtraction, and a test constructs this case and asserts it

#### Scenario: Suppression not required

- **WHEN** no published combination allows a suppressed value to be recovered
- **THEN** the remaining values are published unchanged

### Requirement: Reporting surfaces are permission-gated per surface, server-side

Each reporting surface and each of its endpoints SHALL require a View permission resolved by the
permission evaluator for that surface. A caller without it SHALL be denied whether loading the screen or
calling the endpoint directly. The candidate-derived reporting surfaces SHALL be denied to both
Administrator roles by default, and SHALL be denied to the Auditor role.

*Source: `AUTHZ-001` deny-by-default, `AUTHZ-003`, `AUTHZ-004`. `D16` — Administrators are config-only
with no candidate PII by default. `C-02` — the Auditor is read-only on audit and AI run logs, which are
not candidate records. `access-control-and-admin`'s `TS-BL-022` task 5.8 seeds the Administrator
posture. Expressing per-surface gating requires page-catalog granularity that does not yet exist; see
`design.md` D4, recorded as a cross-feature obligation.*

#### Scenario: Unauthorized reporting call

- **WHEN** a user without the View permission for a reporting surface calls its endpoint directly
- **THEN** the request is denied

#### Scenario: Administrator reads a candidate-derived dashboard

- **WHEN** an Application or System Administrator with no active break-glass grant loads a
  candidate-derived reporting surface
- **THEN** access is denied

#### Scenario: Auditor reads an operational dashboard

- **WHEN** an Auditor loads a candidate-derived reporting surface
- **THEN** access is denied, while the audit and AI-run reporting surface remains available to them

### Requirement: Every reporting metric has exactly one aggregation path

Each metric SHALL be computed by one aggregation over the reporting projection, consumed by every
surface that presents it. No two reporting surfaces SHALL compute the same metric independently.

*Source: `NFR-009` — business rules centralized rather than duplicated. `decision/posting-closure`'s
requirement that the closure checklist and the closure guard share one predicate, with a scenario
failing the suite where a second computation exists; `design.md` D5 applies the same rule to reporting
so two dashboards cannot disagree about one number.*

#### Scenario: Metric shown on two surfaces

- **WHEN** the same metric appears on two reporting surfaces
- **THEN** both read one aggregation and report the same value for the same filters

#### Scenario: Second implementation sought

- **WHEN** the reporting surfaces are inspected for a metric computed in two places
- **THEN** none exists

### Requirement: Exposed filter combinations resolve through indexed access paths

Every filter combination a reporting surface exposes SHALL be supported by an indexed access path.
Where a combination cannot be served within the interactive bound, it SHALL be offered as a background
export rather than served slowly.

*Source: `NFR-001`'s three-second bound on common authenticated screens; `§32`'s rule that "heavy
reports shall run as background exports when query execution exceeds interactive thresholds";
`NFR-005`'s growth requirement. `access-control/audit-review` sets the identical requirement for audit
search. `design.md` D10.*

#### Scenario: Filtered report under grown volume

- **WHEN** a filtered reporting query runs against a substantially populated database
- **THEN** it resolves through an indexed access path rather than a full scan

#### Scenario: Unindexed combination

- **WHEN** a filter combination the surface exposes has no supporting index
- **THEN** the test suite fails rather than the query silently degrading

#### Scenario: Combination beyond the interactive bound

- **WHEN** a requested combination cannot be computed within the interactive bound
- **THEN** it is offered as a background export rather than returned slowly

### Requirement: Reporting includes terminal records and reads only stored data

The reporting projection SHALL include postings and Applications in terminal states. Reporting SHALL be
computed entirely from stored records and SHALL NOT invoke an AI provider, the vector store, or any
external integration.

*Source: `RET-003` — closed posting history retained and visible to authorized users; `WF-007` —
closed postings continue to inform candidate history and analytics, so excluding them makes every
historical rate wrong. `G-12`/`NFR-004` graceful degradation: an AI or vector outage must not prevent
viewing records, which for this feature means no reporting surface may depend on one. `design.md`
Migration Plan.*

#### Scenario: Closed posting in a historical rate

- **WHEN** a rate is computed over a period containing closed and filled postings
- **THEN** those postings contribute to it

#### Scenario: AI provider unavailable

- **WHEN** the AI provider, vector store or notification service is unavailable
- **THEN** every reporting surface continues to render from stored records

#### Scenario: External call sought

- **WHEN** the reporting surfaces are inspected for a call to a model provider or vector store
- **THEN** none exists

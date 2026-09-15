## Purpose

`§20.1`'s Practice Manager Dashboard — the operational worklist a Practice Manager and Recruiter open
to decide what to act on next: what is open, what is waiting on them, and what is blocked. Owned by
`TS-BL-071`.

## ADDED Requirements

### Requirement: The dashboard presents the seven elements `§20.1` specifies

The Practice Manager dashboard SHALL present open postings, candidates by stage, pending interviews,
scorecards due, selected candidates, offer status, and closure blockers. Each element SHALL be derived
from the reporting projection and SHALL carry the reference needed to navigate to the work it names.

*Source: `reference/spec.md` `§20.1`'s Practice Manager Dashboard row, quoted in full — the row carries
no requirement ID, so it is cited directly. `§14.2`'s Reports screen, whose capabilities include the
practice dashboard and candidate stage counts. `UI-003` — every page shows workflow state, owner, next
action, blockers and last updated. `§34` item 18 — dashboards available to authorized users is MVP
Definition of Done.*

#### Scenario: Dashboard rendered

- **WHEN** an authorized Practice Manager loads the dashboard
- **THEN** all seven elements are present, each computed over the postings within the reader's scope

#### Scenario: Element carries its next action

- **WHEN** an element names outstanding work
- **THEN** it carries the reference needed to navigate to that work

#### Scenario: Nothing outstanding

- **WHEN** an element has no outstanding items
- **THEN** it renders an explicit empty state rather than being omitted or showing a bare zero without
  context

### Requirement: Closure blockers are read from the shared aging-and-blocker projection

The dashboard's closure-blockers element SHALL read the projection owned by `reporting/closure-and-aging`
and SHALL NOT compute closure readiness itself.

*Source: `decision/posting-closure`'s requirement that the closure checklist and the closure guard be
computed from one predicate, with a scenario failing the suite where a second computation exists — this
surface is a third reader of that predicate, not a second implementation.
`reporting/read-model`'s one-aggregation-path requirement. `CLS-004`. `design.md` D5.*

#### Scenario: Blockers shown

- **WHEN** a posting within scope has candidates missing a Hubble ID or in blocking workflow states
- **THEN** the element reports the blocked count for that posting

#### Scenario: Agreement with the closure action

- **WHEN** the element reports a posting as having no closure blockers
- **THEN** the closure action for that posting is available, both having been computed from one
  predicate

#### Scenario: Second predicate sought

- **WHEN** this capability is inspected for its own fulfilment or blocker computation
- **THEN** none exists

### Requirement: The dashboard is rendered from existing shared components with no new visual primitive

The dashboard SHALL be built from the design system's existing token set, primitives, page templates and
dense-data-table pattern. It SHALL introduce no chart, sparkline, or other visualization primitive, and
SHALL define no color or spacing value of its own.

*Source: `AGENTS.md`'s standing bar — visual work follows the token system with no raw hex or one-off
spacing, and existing shared patterns are reused before new ones are invented, naming the
dense-data-table specifically. `design-spec.md` `§1.6`, which forbids introducing a one-off color for a
single use case. `S.5`'s conciseness and craft standard. `design.md` D2 records why number-and-table is
the right form here rather than a limitation, and the route charting would take if it were ever wanted.*

#### Scenario: Components used

- **WHEN** the dashboard is rendered
- **THEN** its tabular content uses the shared dense-data-table pattern and its status treatments use
  the shared badge and status-surface components

#### Scenario: New primitive sought

- **WHEN** this capability's component surface is enumerated
- **THEN** it contains no chart, sparkline or visualization primitive, and no locally defined color or
  spacing value

#### Scenario: Progressive detail

- **WHEN** the dashboard is rendered
- **THEN** headline values are presented prominently with supporting detail expandable rather than all
  detail displayed at once

### Requirement: Reads are scoped by the permission evaluator, not by the dashboard

The dashboard SHALL present only postings and Applications the reader is permitted to read, with the
scope applied as a query predicate rather than by discarding fetched rows. Scope SHALL be resolved by
the permission evaluator.

*Source: `C-03`'s resolution — reads governed by the permission matrix, writes scoped by posting
assignment; `AUTHZ-008`'s least-privilege default, widened deliberately for reads by
`access-control-and-admin`'s D5. `candidate-intake`'s task 6.17 sets the query-predicate precedent for
the same reason: discarding fetched rows means the rows were fetched. `NFR-009`.*

#### Scenario: Scoped read

- **WHEN** a reader loads the dashboard
- **THEN** only postings and Applications within their permitted scope contribute to every element

#### Scenario: Scope as a predicate

- **WHEN** the dashboard's queries are inspected
- **THEN** scope is applied as a query predicate and no result set is filtered after retrieval

#### Scenario: Scope changes

- **WHEN** a reader's permissions change
- **THEN** the next dashboard load reflects the new scope without a cache serving the previous one

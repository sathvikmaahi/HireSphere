## Purpose

The Priority Selection Slate — the surface where a Practice Manager accumulates a selection across a
posting's ranked candidates and commits it, built on the shared Floating Action Panel rather than a
new component. Owned by `TS-BL-065`.

## ADDED Requirements

### Requirement: Selection is accumulated in the shared Floating Action Panel

The in-progress selection SHALL be presented in the shared Floating Action Panel, which SHALL be
consumed as built. This capability SHALL supply the panel's items, controls and running output, and
SHALL NOT introduce a second panel, a modal alternative, or a variant component.

*Source: `S.1` — the panel is "a strong, close match for **Priority Selection**: a Practice Manager
browsing the ranked list across a posting, picking candidates into the 5×n slots, reviewing the
running selection, confirming", carried forward as "the intended UI for that workflow".
`design-system`'s `design-system/action-panel` spec states the panel "SHALL render the items,
controls and output it is given, and SHALL NOT own, fetch or mutate the underlying task", and names
this as its first consumer. `reference/layout-spec.md` §6. `AGENTS.md`'s standing bar: reuse existing
shared patterns before inventing new ones.*

#### Scenario: First candidate added

- **WHEN** the first candidate is added to an in-progress selection
- **THEN** the panel appears carrying that candidate as a chip

#### Scenario: Navigating during selection

- **WHEN** the user navigates to a different page with a selection in progress
- **THEN** the panel and its accumulated selection persist

#### Scenario: A second panel sought

- **WHEN** this capability is inspected for its own panel, modal or drawer implementation
- **THEN** none exists

#### Scenario: Panel mutating selection state

- **WHEN** a chip's removal control is used
- **THEN** the panel invokes this capability's handler rather than changing the selection itself

### Requirement: Two presentations — ranked slots and an unranked list

The slate SHALL be presented as ranked positions for a finite posting and as a flat unranked list for
an evergreen posting. Neither presentation SHALL render a fixed number of empty boxes for a posting
whose vacancy count is unbounded.

*Source: `D14`'s "UI implication: the priority-slots surface needs a variant that is not five fixed
boxes", and `D20`'s "Evergreen postings have no slots at all: selection there is a flat unranked list.
The UI needs both presentations."*

#### Scenario: Finite posting

- **WHEN** the slate is opened for a three-vacancy finite posting
- **THEN** ranked positions are presented against the cap of fifteen

#### Scenario: Evergreen posting

- **WHEN** the slate is opened for an evergreen posting
- **THEN** an unranked list is presented with no cap and no fixed slot boxes

### Requirement: A selection carries a tag and a mandatory reason

Committing a selection SHALL require a tag of primary, backup, hold or selected not offered, and a
mandatory reason. A selection SHALL NOT be committable without both.

*Source: `SEL-004`'s four tags; §12.2's `priority_selections.reason` marked "Mandatory";
`WF-003`'s actor, prior state, new state and reason on every transition. `SEL-005` requires the
reason to be retained for selected-not-offered candidates specifically.*

#### Scenario: Commit without a reason

- **WHEN** a selection is committed with a tag and no reason
- **THEN** it is rejected

#### Scenario: Commit without a tag

- **WHEN** a selection is committed with a reason and no tag
- **THEN** it is rejected

#### Scenario: Tag outside the four values

- **WHEN** a tag other than primary, backup, hold or selected not offered is supplied
- **THEN** it is rejected

### Requirement: The slate is grouped by tag and ordered by rank within a tag

The finite slate SHALL be presented grouped by tag, ordered by rank within each group. Rank SHALL
remain unique across the whole slate.

*Source: `D20`'s explicit design-time invitation — "ranking 15 people in strict preference order may
be more precision than a PM can genuinely supply. Tiered ranking within the slot set may be worth
considering during design" — answered by `SEL-004`'s tags acting as the tier, with `SEL-002` retained.
`design.md` D4 records why no separate tier field is introduced.*

#### Scenario: Mixed tags

- **WHEN** a slate holds primary, backup and hold selections
- **THEN** they are presented in tag groups, ordered by rank within each

#### Scenario: A separate tier field sought

- **WHEN** this capability is inspected for a tier attribute distinct from the tag
- **THEN** none exists

### Requirement: Selection is permission-gated and scoped

Committing, retagging, reordering and removing a selection SHALL each be authorized through the
platform's permission evaluator against the posting in question. This capability SHALL NOT decide
authorization by comparing role names.

*Source: `SEL-001` names the Practice Manager and §12.2's `selected_by` reads "Practice Manager **or
authorized user**"; `AUTHZ-003` requires server-side authorization on every endpoint; `C-02`'s nine
roles and `C-03`'s scoped writes make this a matrix question. `access-control-and-admin`'s evaluator
and scope predicates are consumed.*

#### Scenario: Unauthorized commit

- **WHEN** a user without the required permission on that posting commits a selection
- **THEN** it is rejected and the attempt is audited

#### Scenario: Role comparison sought

- **WHEN** this capability is inspected for a hard-coded role comparison in an authorization path
- **THEN** none exists

### Requirement: Evidence is displayed, never generated

The slate SHALL present each candidate's ranking output, approved consolidated scorecard and mandatory
criteria status as already produced elsewhere, carrying their existing evidence-source and
AI-generated marking. This capability SHALL make no AI call, generate no summary and produce no run
record.

*Source: `AI-001`/`UI-004`'s marking of AI-generated content, `G-02`'s evidence-source labelling, and
`G-01`'s mandatory criteria status. `interview-pipeline` closed the ten prompt families with none
assigned to this feature; `design.md` records that this feature authors and invokes none, so the
product's most consequential decision has no AI touchpoint.*

#### Scenario: Candidate evidence shown

- **WHEN** a candidate is inspected on the slate
- **THEN** their ranking output and approved scorecard are shown with their existing markings intact

#### Scenario: AI call sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no AI invocation, prompt payload or run record

### Requirement: The surface adds no new design-system component

Every visual element SHALL resolve from the existing design tokens, page templates, dense data table,
forms and the Floating Action Panel. No raw color, spacing or type value SHALL appear, and no new
shared component SHALL be introduced.

*Source: `AGENTS.md`'s standing product-quality bar — the token system in `reference/design-spec.md`
and the layout system in `reference/layout-spec.md`, no raw hex values, no one-off spacing, reuse
before invention. `design-system`'s `TS-BL-008`–`TS-BL-012`.*

#### Scenario: Raw value sought

- **WHEN** this capability's styles are inspected for a literal color, spacing or type value
- **THEN** none exists

#### Scenario: New shared component sought

- **WHEN** this capability's components are enumerated
- **THEN** each is either composed from design-system components or specific to this surface

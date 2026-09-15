## Purpose

The Floating Action Panel — the persistent bottom-right panel for an in-progress task that spans
pages, carrying selected-item chips and a running output. Owned by `TS-BL-010`; its first real
consumer is Priority Selection in a later feature.

## ADDED Requirements

### Requirement: Panel states

The panel SHALL have exactly two states. Collapsed, it SHALL present a compact control showing an
icon, a count badge and a control to re-expand, filled with the accent gradient token. Expanded, it
SHALL present a fixed-width card capped to the viewport width on narrow viewports, with an
accent-colored top border, the raised elevation and the large corner radius.

*Source: `reference/layout-spec.md` §6. The gradient resolves from a named token whose stops are
existing accent tokens rather than from the product-switcher tiles the guide points at, because
`exploration-notes.md` S.2 rules the switcher out — see `design.md` D15.*

#### Scenario: Collapsed presentation

- **WHEN** the panel is collapsed
- **THEN** it shows an icon, the count of items in progress, and a control to re-expand

#### Scenario: Expanded presentation

- **WHEN** the panel is expanded
- **THEN** it renders at the defined width with the accent top border and raised elevation

#### Scenario: Narrow viewport

- **WHEN** the viewport is narrower than the panel's fixed width plus its margins
- **THEN** the panel is capped to the viewport width rather than overflowing it

### Requirement: The panel renders only when a task is in progress

The panel SHALL render only once the underlying task has at least one item. It SHALL NOT occupy
space or present an empty state otherwise.

*Source: `reference/layout-spec.md` §6.*

#### Scenario: No in-progress task

- **WHEN** no cross-page task is in progress
- **THEN** the panel renders nothing and shows no empty state

#### Scenario: First item added

- **WHEN** the first item is added to the underlying task
- **THEN** the panel appears

### Requirement: Panel content structure

The expanded panel SHALL present a scrollable row of selected-item chips with their contextual
controls, and a bottom row showing the resulting output alongside a minimize control.

*Source: `reference/layout-spec.md` §6.*

#### Scenario: More chips than fit

- **WHEN** the selected items exceed the panel's width
- **THEN** the chip row scrolls within the panel rather than expanding it

#### Scenario: Running output

- **WHEN** the selection changes
- **THEN** the output row reflects the current selection

### Requirement: Panel persists across pages

The panel SHALL remain present and unchanged as the user navigates between pages while the
underlying task is in progress.

*Source: `reference/layout-spec.md` §6's "in-progress, cross-page task", and
`exploration-notes.md` S.1 — Priority Selection is a Practice Manager browsing a ranked list across
a posting while accumulating a selection, which is the workflow this behavior exists for.*

#### Scenario: Navigation during a task

- **WHEN** the user navigates to a different page while the task is in progress
- **THEN** the panel and its contents persist

### Requirement: Collapse state is independent of the task

The collapsed or expanded state SHALL be user-controlled and SHALL be independent of clearing the
underlying task. Transitions between states SHALL be animated rather than instant.

*Source: `reference/layout-spec.md` §6.*

#### Scenario: User minimizes an active task

- **WHEN** the user minimizes the panel while items remain in progress
- **THEN** the panel collapses and the underlying task is unaffected

#### Scenario: State transition

- **WHEN** the panel expands or collapses
- **THEN** the transition is animated

### Requirement: The panel owns no task state

The panel SHALL render the items, controls and output it is given, and SHALL NOT own, fetch or
mutate the underlying task.

*Source: `design.md` D1. The panel is built before its consumer exists; owning task semantics here
would embed one feature's workflow — Priority Selection's 1-to-5 slots per vacancy (`D20`) — into a
component that other features are also meant to use.*

#### Scenario: Item removed

- **WHEN** the user removes a chip
- **THEN** the panel invokes the removal handler it was given rather than mutating state itself

#### Scenario: No consumer in this scope

- **WHEN** this feature is deployed
- **THEN** the panel exists and is mountable into the shell's panel region
- **AND** no screen in this scope mounts it

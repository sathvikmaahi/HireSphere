## Purpose

The dense-data-table pattern — the fourth page template the supplied layout guide does not provide —
for TalentSphere's genuinely tabular screens: the permission matrix, the ranking board, and the
audit and AI-run logs. Owned by `TS-BL-009`.

## ADDED Requirements

### Requirement: Dense data table as a first-class template

The dense data table SHALL be a first-class page template built entirely from existing color,
spacing, radius and type tokens, without introducing new tokens. Row and cell density SHALL use the
compact type steps. It SHALL conform to the page-template contract defined by
`design-system/app-shell` and SHALL also be usable outside a page template.

*Source: `exploration-notes.md` S.3, which records that the guide's three templates are
card-oriented and offer nothing for the product's tabular screens, and directs that a fourth
template be built from existing tokens rather than by repurposing the card grid;
`talentsphere-wave-1-foundation` D14, inherited per `design.md` D14. Usable outside a template per
`design.md` D12 — this is why the item depends on the token layer alone.*

#### Scenario: Built from existing tokens

- **WHEN** the dense table renders
- **THEN** every color, spacing, radius and type value resolves from an existing token
- **AND** no new token was introduced for it

#### Scenario: Table inside a detail view

- **WHEN** a dense table is placed inside another template's content column
- **THEN** it renders correctly without the page-header wrapper

#### Scenario: Not a restyled card grid

- **WHEN** a screen's data is inherently tabular
- **THEN** it uses the dense table rather than the card grid template

### Requirement: Table capabilities

The dense table SHALL support search, filtering, sorting and pagination.

*Source: `UI-008`.*

#### Scenario: Sorting a column

- **WHEN** a user sorts by a sortable column
- **THEN** the rows are ordered by that column and the sorted column indicates its direction

#### Scenario: Filtered result set

- **WHEN** a filter excludes every row
- **THEN** the empty state renders inside the table container rather than replacing the layout

#### Scenario: Paginated result set

- **WHEN** a result set exceeds one page
- **THEN** pagination controls are presented and the current position is indicated

### Requirement: Horizontal overflow is contained by the table

When a table's columns exceed the available width, the table SHALL scroll horizontally within its
own container. The page body SHALL NOT scroll horizontally.

*Source: `exploration-notes.md` S.3's dense-data requirement, and `reference/layout-spec.md` §4's
fluid content area — a table wide enough to push the page sideways would break the shell's scrolling
contract, which is the failure this pattern exists to prevent.*

#### Scenario: Wide table

- **WHEN** a dense table's columns exceed the available width
- **THEN** the table scrolls horizontally within its own container
- **AND** the page body does not scroll horizontally

### Requirement: Export is gated on a supplied permission decision

An export control SHALL be presented only when the caller supplies an affirmative export decision,
and SHALL NOT be presented when no decision is supplied. The table SHALL NOT evaluate export
permission itself.

*Source: `UI-008`'s "export where permission allows" and `UI-002`'s server-authoritative rule, as
applied by `design.md` D1 and D6. The table takes the decision as input so this item depends on the
token layer alone rather than on `access-control-and-admin`'s evaluator, which is scheduled after
this entire feature.*

#### Scenario: Export gating

- **WHEN** a user without Export permission views a dense table
- **THEN** no export control is offered
- **AND** a direct export request is denied by the server

#### Scenario: No decision supplied

- **WHEN** the table is rendered without an export decision
- **THEN** no export control is offered

### Requirement: Row density does not compromise accessibility

Compact density SHALL NOT reduce contrast below the accessibility baseline, remove focus indication,
or prevent keyboard traversal of interactive cells.

*Source: `UI-009`. Stated explicitly because density and accessibility pull against each other, and
the compact type steps are the smallest in the system.*

#### Scenario: Keyboard traversal of a dense table

- **WHEN** a user traverses a dense table's interactive cells by keyboard
- **THEN** each focused element is visibly indicated

#### Scenario: Header and cell semantics

- **WHEN** a screen reader encounters the table
- **THEN** column headers are associated with their cells

### Requirement: Loading and empty states occupy the table container

Loading and empty states SHALL render inside the table's own container rather than swapping to a
different layout.

*Source: `reference/layout-spec.md` §4.2's rule for the browsable grid, applied to the fourth
template for consistency.*

#### Scenario: Loading rows

- **WHEN** table data is loading
- **THEN** a loading state renders within the table container
- **AND** the surrounding page layout is unchanged

## Purpose

The application shell and page structure every TalentSphere screen renders inside: shell states,
sidebar behavior, the page header pattern, page templates, responsive rules, the floating action
panel region, and the single landing route a signed-in user reaches. Owned by `TS-BL-008`.

## ADDED Requirements

### Requirement: Three shell states

The application SHALL provide exactly three shell states: an authenticated shell with a collapsible
sidebar, a scrollable main content area and a floating action panel region; a bare shell with no
sidebar and no floating panel; and a presentation shell with chrome fully removed. Which state
applies SHALL be resolved by asking first whether the route is a bare route and then whether the
visitor is authenticated.

*Source: `reference/layout-spec.md` §1, including its stated evaluation order. `design.md` D11
records that D.9's "sidebar 3-states" phrasing means these three shell states plus the sidebar's two
widths; no third sidebar width exists in either guide.*

#### Scenario: Authenticated route

- **WHEN** an activated, authenticated user loads a standard route
- **THEN** the authenticated shell renders with sidebar and main content area

#### Scenario: Shell does not scroll as a whole

- **WHEN** page content exceeds the viewport height
- **THEN** only the main content column scrolls, and the sidebar and floating panel region stay
  fixed

#### Scenario: Presentation shell has no assigned screen yet

- **WHEN** this feature is deployed
- **THEN** the presentation shell state exists and is selectable by a route
- **AND** no screen in this scope uses it

### Requirement: Bare shell limited to sign-in

The bare shell SHALL be used only for the sign-in route. The system SHALL NOT expose any public,
embeddable or shared read-only content view.

*Source: `exploration-notes.md` S.2, which rules the guide's embeddable-document case inapplicable
because candidates have zero system access and no public routes exist. Expressed as a property of
the route table rather than a convention.*

#### Scenario: Unauthenticated visitor requests any route

- **WHEN** an unauthenticated visitor requests any route other than sign-in
- **THEN** the visitor is redirected to sign-in and no content is rendered

#### Scenario: No public content route exists

- **WHEN** the route table is inspected
- **THEN** sign-in is the only route that renders without authentication

### Requirement: Authenticated landing route

The system SHALL define exactly one default landing route that an activated user reaches after
signing in. The landing route SHALL NOT require any permission beyond activation, and SHALL surface
the signed-in user's own outstanding work when that data is supplied to it.

*Source: `talentsphere-wave-1-foundation` D17, inherited per `design.md` D14 and `platform-core`'s
D11. The no-grant rule is structural: the most common state during permission rollout is activated
but not yet granted anything, and routing that user to a permissioned page produces a dead end at
exactly the moment an administrator is trying to fix their access.*

#### Scenario: Successful sign-in

- **WHEN** an activated user completes sign-in without having requested a specific route
- **THEN** the user arrives at the landing route inside the authenticated shell

#### Scenario: Deep link preserved through sign-in

- **WHEN** an unauthenticated visitor requests a specific authorized route and then signs in
- **THEN** the user arrives at the originally requested route rather than the landing route

#### Scenario: Landing route requires no page grant

- **WHEN** an activated user holds no page grants at all
- **THEN** the landing route still renders, showing an empty state and a route to request access
  rather than an access denial

#### Scenario: Landing content is scoped to the viewer

- **WHEN** the landing route is supplied with outstanding work to display
- **THEN** it shows only what its caller supplied for the signed-in user, and resolves nothing
  itself

### Requirement: No product switcher

The sidebar header SHALL NOT present a product or workspace switcher.

*Source: `exploration-notes.md` S.2 — the guide's §2.3 switcher assumes a shell hosting multiple
products; TalentSphere is one product.*

#### Scenario: Sidebar header in expanded state

- **WHEN** the sidebar renders expanded
- **THEN** its header shows the product identity and a menu toggle, with no switcher dropdown

### Requirement: Sidebar behavior

The sidebar SHALL have two widths, expanded and collapsed, SHALL transition between them smoothly,
SHALL default to expanded, and SHALL persist the user's choice across sessions. The sidebar SHALL
render on its own fixed dark surface drawn from a declared theme-independent token family and SHALL
NOT participate in the light and dark theme toggle. Collapsing SHALL hide labels and count badges,
leaving centered icons each carrying an accessible name.

*Source: `reference/layout-spec.md` §2 and §2.2. The token family is required by
`exploration-notes.md` S.6, which records that the guide mandates this surface and its hover and
active tints without defining any of them; values are fixed in `design.md` D4.*

#### Scenario: Collapsed state

- **WHEN** the user collapses the sidebar
- **THEN** labels and count badges are hidden, icons remain centered, and each icon exposes its
  label as an accessible name or tooltip

#### Scenario: Choice persists

- **WHEN** a user who collapsed the sidebar returns in a later session
- **THEN** the sidebar renders collapsed

#### Scenario: Theme toggle does not affect the sidebar

- **WHEN** the user switches the content theme
- **THEN** the sidebar surface, hover and active colors are unchanged while the main content area
  re-themes

### Requirement: Navigation items and counts

Navigation items SHALL pair an icon with a label, SHALL indicate the active item distinctly from
hover, and MAY carry a numeric count badge whose displayed value is capped. The sidebar SHALL render
the navigation items it is given and SHALL NOT evaluate permissions itself.

*Source: `reference/layout-spec.md` §2.2. The non-evaluation clause is `design.md` D1 and D6: a
consuming feature supplies an already-filtered list, so the shell depends on the token layer alone
and the server remains authoritative per `UI-002`.*

#### Scenario: Active item

- **WHEN** a navigation item corresponds to the current route
- **THEN** it renders in the active treatment, distinct from the hover treatment

#### Scenario: Large count

- **WHEN** a count badge value exceeds the display cap
- **THEN** the capped form is shown rather than the full number

#### Scenario: Navigation reflects permissions

- **WHEN** the caller supplies a navigation list excluding pages the user cannot view
- **THEN** those pages have no navigation item
- **AND** requesting such a page directly is still refused by the server

#### Scenario: No navigation supplied

- **WHEN** no navigation items are supplied
- **THEN** the navigation list renders empty rather than defaulting to a full menu

### Requirement: Sidebar utility footer

The sidebar SHALL present, in its footer, a theme toggle, a user menu showing the signed-in identity
and offering sign-out, and a brand slot shown only in the expanded state.

*Source: `reference/layout-spec.md` §2.1 and §2.4.*

#### Scenario: Sign-out

- **WHEN** a user selects sign-out from the user menu
- **THEN** the sign-out action supplied to the shell is invoked and the user is returned to sign-in

#### Scenario: Collapsed footer

- **WHEN** the sidebar is collapsed
- **THEN** the brand slot is omitted entirely

### Requirement: Main content area rules

The main content area SHALL be fluid with no default maximum width, SHALL apply the defined content
padding with additional bottom clearance so content never sits under the floating panel region, and
SHALL allow a route to opt out of padding entirely.

*Source: `reference/layout-spec.md` §4.*

#### Scenario: Full-bleed route

- **WHEN** a route declares itself full-bleed
- **THEN** the content area renders without the standard padding wrapper

#### Scenario: Content beneath the panel region

- **WHEN** a page's content extends to the bottom of the scroll area
- **THEN** bottom clearance keeps it clear of the floating panel region

### Requirement: Page header pattern

Every standard content page SHALL open with a page header presenting a title, a one-line description
and at most one primary action on the same row. On narrow viewports the action SHALL reduce to
icon-only or be hidden rather than allowing the header to wrap.

*Source: `reference/layout-spec.md` §4.1.*

#### Scenario: Narrow viewport

- **WHEN** a standard page renders below the small breakpoint
- **THEN** the header stays on one row, with the primary action reduced or hidden

#### Scenario: Consistent section rhythm

- **WHEN** a page renders multiple top-level sections
- **THEN** they are separated by the single defined section gap rather than per-section margins

### Requirement: Page templates

The system SHALL provide a page-template contract and three templates conforming to it: a browsable
card grid with a filter row and infinite-scroll trigger; a detail view splitting into a flexible
main column and a fixed-width metadata rail that collapses to a single stacked column below the
large breakpoint; and a standalone full-bleed view. Additional templates MAY be contributed by other
capabilities against the same contract. Loading and empty states SHALL render inside the same
container the content would occupy.

*Source: `reference/layout-spec.md` §4.2. The contract-plus-contributions shape is `design.md` D12:
the dense data table is the fourth template and is specified in `design-system/data-table`, so
neither item waits on the other.*

#### Scenario: Empty result set

- **WHEN** a browsable grid has no results
- **THEN** the empty state renders within the grid container rather than replacing the layout

#### Scenario: Detail view on a narrow viewport

- **WHEN** a detail view renders below the large breakpoint
- **THEN** the columns stack with main content first

#### Scenario: Metadata rail sizing

- **WHEN** a detail view's main column is taller than its rail
- **THEN** the rail sizes to its own content rather than stretching

### Requirement: Card grid progression

Card listings SHALL follow the fixed responsive column progression and a constant gap at every
breakpoint, without ad hoc column counts.

*Source: `reference/layout-spec.md` §5.*

#### Scenario: Widening viewport

- **WHEN** the viewport crosses each defined breakpoint
- **THEN** the column count follows the defined progression exactly

### Requirement: Floating action panel region

The shell SHALL provide a floating action panel region anchored to the bottom-right, layered above
page content and the sidebar, which renders only when content is mounted into it and occupies no
space otherwise.

*Source: `reference/layout-spec.md` §6. The region is the shell's; the panel that mounts into it is
`design-system/action-panel`.*

#### Scenario: Nothing mounted

- **WHEN** no content is mounted into the region
- **THEN** the region renders nothing and shows no empty state

#### Scenario: Layering

- **WHEN** content is mounted and the page is scrolled
- **THEN** the mounted content stays anchored above both page content and the sidebar

### Requirement: Responsive breakpoints

The system SHALL use the defined breakpoint scale, with the small breakpoint as the primary
mobile-to-tablet pivot and the large breakpoint as the tablet-to-desktop pivot. The sidebar SHALL
NOT change behavior at any breakpoint; it is collapse-toggled by the user only.

*Source: `reference/layout-spec.md` §7, which explicitly frames sidebar auto-collapse as an
extension to design deliberately rather than assume, and `UI-001`'s desktop-first framing.*

#### Scenario: Sidebar at a narrow viewport

- **WHEN** the viewport narrows below the large breakpoint
- **THEN** the sidebar retains the user's expanded or collapsed choice rather than auto-collapsing

### Requirement: Workflow state presentation

The shell SHALL provide a reusable presentation for a workflow-governed record's current state,
owner, next action, blockers and last-updated timestamp, for pages that display such a record.

*Source: `UI-003`. Values are supplied by the consuming feature; `platform-core`'s workflow engine
owns the transitions themselves.*

#### Scenario: Record with a blocker

- **WHEN** a page supplies a workflow-governed record with a blocking condition
- **THEN** the blocker is shown alongside the state and the owner

### Requirement: Permission-aware affordances default to absent

Where the shell exposes an affordance a permission governs, it SHALL render that affordance only
when the caller supplies an affirmative decision, and SHALL render nothing when no decision is
supplied.

*Source: `UI-002` and `design.md` D6. Default-absent rather than default-enabled or
default-disabled, because deny-by-default is the architectural rule for the evaluator and a renderer
that defaulted permissive would be the one place in the stack inverting it.*

#### Scenario: No decision supplied

- **WHEN** a permission-governed affordance receives no decision
- **THEN** it is not rendered

#### Scenario: Server remains authoritative

- **WHEN** an affordance is hidden and the underlying endpoint is invoked directly
- **THEN** the request is still refused by the server

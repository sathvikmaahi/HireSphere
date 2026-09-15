## Purpose

The application shell and page structure every TalentSphere screen renders inside: shell states, sidebar behavior, page header and page templates including a dense-data-table pattern, responsive rules, and the shared status components that later AI-bearing screens depend on.

## ADDED Requirements

### Requirement: Three shell states

The application SHALL provide exactly three shell states: an authenticated shell with a collapsible sidebar, a scrollable main content area, and an optional floating action panel region; a bare shell with no sidebar and no floating panel; and a presentation shell with chrome fully removed.

#### Scenario: Authenticated route

- **WHEN** an activated, authenticated user loads a standard route
- **THEN** the authenticated shell renders with sidebar and main content area

#### Scenario: Shell does not scroll as a whole

- **WHEN** page content exceeds the viewport height
- **THEN** only the main content column scrolls, and the sidebar and floating panel region stay fixed

#### Scenario: Presentation shell has no assigned screen yet

- **WHEN** this wave is deployed
- **THEN** the presentation shell state exists and is selectable by a route
- **AND** no screen in this scope uses it

### Requirement: Bare shell limited to sign-in

The bare shell SHALL be used only for the sign-in route. The system SHALL NOT expose any public, embeddable, or shared read-only content view.

#### Scenario: Unauthenticated visitor requests any route

- **WHEN** an unauthenticated visitor requests any route other than sign-in
- **THEN** the visitor is redirected to sign-in and no content is rendered

#### Scenario: No public content route exists

- **WHEN** the route table is inspected
- **THEN** sign-in is the only route that renders without authentication

### Requirement: Authenticated landing route

The system SHALL define exactly one default landing route that an activated user reaches after signing in. The landing route SHALL surface the signed-in user's own outstanding work and SHALL NOT require any permission beyond activation.

#### Scenario: Successful sign-in

- **WHEN** an activated user completes sign-in without having requested a specific route
- **THEN** the user arrives at the landing route inside the authenticated shell

#### Scenario: Landing content is scoped to the viewer

- **WHEN** the landing route renders
- **THEN** it shows only the signed-in user's own tasks, notifications, and pending items
- **AND** it shows nothing the user lacks permission to see

#### Scenario: Deep link preserved through sign-in

- **WHEN** an unauthenticated visitor requests a specific authorized route and then signs in
- **THEN** the user arrives at the originally requested route rather than the landing route

#### Scenario: Landing route requires no page grant

- **WHEN** an activated user holds no page grants at all
- **THEN** the landing route still renders, showing an empty state rather than an access denial

### Requirement: No product switcher

The sidebar header SHALL NOT present a product or workspace switcher.

#### Scenario: Sidebar header in expanded state

- **WHEN** the sidebar renders expanded
- **THEN** its header shows the product identity and a menu toggle, with no switcher dropdown

### Requirement: Sidebar behavior

The sidebar SHALL have two widths, expanded and collapsed, SHALL transition between them smoothly, SHALL default to expanded, and SHALL persist the user's choice across sessions. The sidebar SHALL render on its own fixed dark surface and SHALL NOT participate in the light and dark theme toggle. Collapsing SHALL hide labels and count badges, leaving centered icons each carrying an accessible name.

#### Scenario: Collapsed state

- **WHEN** the user collapses the sidebar
- **THEN** labels and count badges are hidden, icons remain centered, and each icon exposes its label as an accessible name or tooltip

#### Scenario: Choice persists

- **WHEN** a user who collapsed the sidebar returns in a later session
- **THEN** the sidebar renders collapsed

#### Scenario: Theme toggle does not affect the sidebar

- **WHEN** the user switches the content theme
- **THEN** the sidebar surface color is unchanged while the main content area re-themes

### Requirement: Navigation items and counts

Navigation items SHALL pair an icon with a label, SHALL indicate the active item distinctly from hover, and MAY carry a numeric count badge whose displayed value is capped.

#### Scenario: Active item

- **WHEN** a navigation item corresponds to the current route
- **THEN** it renders in the active treatment, distinct from the hover treatment

#### Scenario: Large count

- **WHEN** a count badge value exceeds the display cap
- **THEN** the capped form is shown rather than the full number

#### Scenario: Navigation reflects permissions

- **WHEN** a user lacks View permission on a page
- **THEN** that page's navigation item is not rendered

### Requirement: Sidebar utility footer

The sidebar SHALL present, in its footer, a theme toggle, a user menu showing the signed-in identity and offering sign-out, and a brand logo shown only in the expanded state.

#### Scenario: Sign-out

- **WHEN** a user selects sign-out from the user menu
- **THEN** the session is invalidated and the user is returned to sign-in

### Requirement: Main content area rules

The main content area SHALL be fluid with no default maximum width, SHALL apply the defined content padding with additional bottom clearance so content never sits under the floating panel region, and SHALL allow a route to opt out of padding entirely.

#### Scenario: Full-bleed route

- **WHEN** a route declares itself full-bleed
- **THEN** the content area renders without the standard padding wrapper

### Requirement: Page header pattern

Every standard content page SHALL open with a page header presenting a title, a one-line description, and at most one primary action on the same row. On narrow viewports the action SHALL reduce to icon-only or be hidden rather than allowing the header to wrap.

#### Scenario: Narrow viewport

- **WHEN** a standard page renders below the small breakpoint
- **THEN** the header stays on one row, with the primary action reduced or hidden

#### Scenario: Consistent section rhythm

- **WHEN** a page renders multiple top-level sections
- **THEN** they are separated by the single defined section gap rather than per-section margins

### Requirement: Page templates

The system SHALL provide four page templates: a browsable card grid with a filter row and infinite-scroll trigger; a detail view splitting into a flexible main column and a fixed-width metadata rail that collapses to a single stacked column below the large breakpoint; a standalone full-bleed view; and a dense data table. Loading and empty states SHALL render inside the same container the content would occupy.

#### Scenario: Empty result set

- **WHEN** a browsable grid has no results
- **THEN** the empty state renders within the grid container rather than replacing the layout

#### Scenario: Detail view on a narrow viewport

- **WHEN** a detail view renders below the large breakpoint
- **THEN** the columns stack with main content first

#### Scenario: Metadata rail sizing

- **WHEN** a detail view's main column is taller than its rail
- **THEN** the rail sizes to its own content rather than stretching

### Requirement: Dense data table pattern

The dense data table SHALL be a first-class template built from existing color, spacing, radius, and type tokens, without introducing new tokens. It SHALL support search, filtering, sorting, pagination, horizontal overflow within its own container, and export gated on Export permission. Row and cell density SHALL use the compact type steps.

#### Scenario: Wide table

- **WHEN** a dense table's columns exceed the available width
- **THEN** the table scrolls horizontally within its own container and the page body does not scroll horizontally

#### Scenario: Export gating

- **WHEN** a user without Export permission views a dense table
- **THEN** no export control is offered, and a direct export request is denied

#### Scenario: First consumer

- **WHEN** the permission matrix screen renders
- **THEN** it uses the dense data table pattern rather than a bespoke table

### Requirement: Card grid progression

Card listings SHALL follow the fixed responsive column progression and a constant gap at every breakpoint, without ad hoc column counts.

#### Scenario: Widening viewport

- **WHEN** the viewport crosses each defined breakpoint
- **THEN** the column count follows the defined progression exactly

### Requirement: Floating action panel region

The shell SHALL provide a floating action panel region anchored to the bottom-right, layered above page content and the sidebar, which renders only when an in-progress task has at least one item and occupies no space otherwise.

#### Scenario: No in-progress task

- **WHEN** no cross-page task is in progress
- **THEN** the region renders nothing and shows no empty state

#### Scenario: No consumer in this scope

- **WHEN** this wave is deployed
- **THEN** the region is available for mounting
- **AND** no feature in this scope mounts content into it

### Requirement: Responsive breakpoints

The system SHALL use the defined breakpoint scale, with the small breakpoint as the primary mobile-to-tablet pivot and the large breakpoint as the tablet-to-desktop pivot. The sidebar SHALL NOT change behavior at any breakpoint; it is collapse-toggled by the user only.

#### Scenario: Sidebar at a narrow viewport

- **WHEN** the viewport narrows below the large breakpoint
- **THEN** the sidebar retains the user's expanded or collapsed choice rather than auto-collapsing

### Requirement: Workflow state and validation presentation

Every page displaying a workflow-governed record SHALL show its current state, owner, next action, blockers, and last-updated timestamp where applicable. Validation errors SHALL be presented at both field level and summary level.

#### Scenario: Record with a blocker

- **WHEN** a workflow-governed record has a blocking condition
- **THEN** the page shows the blocker alongside the state and the owner

#### Scenario: Form with multiple errors

- **WHEN** a submitted form fails validation on several fields
- **THEN** each field shows its own error and a summary lists them together

### Requirement: Permission-aware controls with server authority

The interface SHALL hide actions the user cannot perform, while the server remains authoritative.

#### Scenario: Hidden control

- **WHEN** a user lacks permission for an action
- **THEN** the control is not rendered
- **AND** invoking the underlying endpoint directly is still denied

### Requirement: AI disclosure components

The system SHALL provide a reusable label marking content as AI-generated or AI-assisted until approved, and a reusable human-review disclaimer surface for AI-derived recommendations that avoids claiming bias-free output. Both SHALL be built from the status-surface and badge patterns.

#### Scenario: Unapproved AI content displayed

- **WHEN** a screen renders AI-produced content that has not been approved by a human
- **THEN** the AI-generated label is displayed with it

#### Scenario: Disclaimer wording

- **WHEN** the human-review disclaimer renders
- **THEN** it states that output is an AI-assisted recommendation subject to human review
- **AND** it makes no claim that the output is free of bias

#### Scenario: Available before its consumers ship

- **WHEN** this wave is deployed
- **THEN** both components exist and are usable
- **AND** the screens that consume them arrive in later waves

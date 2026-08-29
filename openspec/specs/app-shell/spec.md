# App Shell Specification

## Purpose

Define the persistent application chrome — sidebar, main content area, bare/public shell, and floating action panel — per `layout-spec.md` and visual tokens per `design-spec.md`. Every authenticated feature route MUST render inside this shell unless explicitly using the presentation/full-bleed template.

## Requirements

### Requirement: Three shell states

The application SHALL support three shell states: authenticated (sidebar + main), bare/public (no sidebar), and presentation (chrome hidden).

#### Scenario: Unauthenticated visitor

- **GIVEN** a user is not authenticated
- **WHEN** they visit any route except narrowly-scoped public previews
- **THEN** they MUST be redirected to `/login`
- **AND** the bare shell MUST render (64px sticky header, no sidebar)

#### Scenario: Authenticated user

- **GIVEN** a user is authenticated and active
- **WHEN** they visit an application route
- **THEN** the authenticated shell MUST render with collapsible sidebar and scrollable main content
- **AND** the sidebar MUST remain fixed while main content scrolls

### Requirement: Sidebar dimensions and collapse

The sidebar MUST support expanded width 224px and collapsed width 56px. Collapse state MUST persist across sessions, defaulting to expanded.

#### Scenario: Collapse toggle

- **GIVEN** the sidebar is expanded
- **WHEN** the user clicks the collapse toggle
- **THEN** the sidebar MUST animate to 56px width within ~200ms
- **AND** text labels and count badges MUST be hidden, with icons remaining and accessible names available

#### Scenario: Sidebar theme independence

- **GIVEN** the user toggles light/dark theme in the main content area
- **WHEN** the theme changes
- **THEN** the sidebar MUST retain its fixed dark navy surface
- **AND** MUST NOT follow the main content theme toggle

### Requirement: Design tokens

All colors, typography, corner radii, shadows, and button variants MUST be implemented from `design-spec.md` as CSS variables with explicit light/dark pairs.

#### Scenario: Theme switch

- **GIVEN** the user toggles theme
- **WHEN** the theme changes
- **THEN** color transitions MUST animate over ~250ms
- **AND** accent color `#00AAE7` MUST remain constant across themes

### Requirement: Page templates

Standard pages MUST use one of three templates: browsable grid, detail split (280px metadata rail), or full-bleed standalone.

#### Scenario: Browsable listing page

- **GIVEN** a list route (e.g. candidates, postings)
- **WHEN** the page renders
- **THEN** it MUST include the page header pattern (title, description, primary action)
- **AND** a responsive card grid following column progression 1 → 2 → 3 → 4 columns at defined breakpoints
- **AND** loading and empty states MUST occupy the same grid container as cards

#### Scenario: Detail page on narrow viewport

- **GIVEN** a detail route with metadata rail
- **WHEN** viewport width is below the Large breakpoint (1024px)
- **THEN** main content and rail MUST stack in a single column with main content first

### Requirement: Floating action panel

A bottom-right floating panel SHALL appear only when a cross-page task has at least one item in progress (e.g. shortlist or priority selection).

#### Scenario: Panel hidden when empty

- **GIVEN** no in-progress multi-select task
- **WHEN** the user navigates the app
- **THEN** the floating panel MUST NOT render

#### Scenario: Panel visible with selections

- **GIVEN** the user has selected one or more items for a supported task
- **WHEN** they navigate away from the source page
- **THEN** the floating panel MUST remain visible in collapsed or expanded state
- **AND** MUST be layered above sidebar and page content

### Requirement: HireSphere branding only

The shell MUST use HireSphere logo assets from `design-spec.md` / `assets/`. Miracle branding MUST NOT appear anywhere in the shell.

#### Scenario: Logo display

- **GIVEN** the sidebar footer is visible (expanded state)
- **WHEN** the brand logo renders
- **THEN** it MUST use the HireSphere logo variant appropriate to the sidebar's dark background
- **AND** MUST NOT display Miracle or third-party product branding except functional UI icons

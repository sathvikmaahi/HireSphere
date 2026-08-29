# Layout Guide

This document is a standalone specification of page structure, navigation composition, and responsive/grid behavior for applications that should feel consistent with this application shell.

Like the Style Guide, it describes design decisions only — dimensions, breakpoints, and structural behavior — not a particular programming language, styling framework, or grid system. Color, type, icon, corner-radius, and shadow specifications live in the Style Guide and are only referenced here by name, never repeated.

---

## 1. App Shell

The application uses a persistent **sidebar + main content** shell, with a floating utility panel layered on top. Three shell states exist:

1. **Authenticated shell** — collapsible sidebar (left) + scrollable main content area (right) + an optional floating action panel (bottom-right).
2. **Bare/public shell** — no sidebar, no floating panel; used for a sign-in screen and any standalone/embeddable public view (e.g. a shared read-only document). A bare shell may still show a lightweight top header for brand identity, but nothing else.
3. **Presentation/fullscreen shell** — content only, chrome fully removed (sidebar and top-level nav hidden), for a distraction-free full-viewport view of one piece of content.

```
┌───────────────────────────────────────────────────┐
│ ┌────────┐  ┌───────────────────────────────────┐ │
│ │        │  │                                   │ │
│ │Sidebar │  │        Main content area          │ │
│ │        │  │        (scrolls independently)    │ │
│ │        │  │                                   │ │
│ │        │  │                        ┌────────┐ │ │
│ │        │  │                        │Floating│ │ │
│ │        │  │                        │ panel  │ │ │
│ └────────┘  └───────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

The shell itself occupies the full viewport height and never scrolls as a whole; only the main content column scrolls independently, so the sidebar and floating panel stay fixed in place as the user scrolls page content.

Which shell state applies is driven by two independent conditions, evaluated in order: (1) is this a bare/public route (e.g. sign-in, an embeddable preview), and (2) is the visitor authenticated. An unauthenticated visitor is only ever allowed through to a single, narrowly-scoped public content view (never the full shell) — everything else redirects to the sign-in route.

---

## 2. Sidebar

A fixed-height, collapsible left rail, always rendered against its own dark navy surface — **the sidebar does not participate in the light/dark theme toggle**; only the main content area does. This keeps a stable navigational anchor regardless of the user's content-area theme choice.

### 2.1 Structure (top to bottom)

1. **Header row** — either a product switcher (expanded state) or a plain menu toggle (collapsed state), with a dividing line separating it from the nav list.
2. **Navigation list** — vertical stack of nav items (icon + label + optional count badge), scrolling independently if it overflows.
3. **Utility footer** — theme toggle, user menu (avatar + name/email, opening a small dropdown with sign-out), and a bottom-anchored brand logo, each section separated by a dividing line.

### 2.2 Collapse behavior

- Two widths: **224px expanded**, **56px collapsed**. Width changes with a short, smooth transition (roughly a fifth of a second); the background-color transition is slightly slower to match the theme-switch timing elsewhere in the app.
- The collapsed state hides all text labels and count badges — only icons remain, centered, each with a text label available as a tooltip/accessible-name substitute.
- Expand/collapse is user-controlled via a toggle button and persists across sessions, defaulting to **expanded**.
- Nav items pair a 17px icon with a Small-size, Medium-weight label (see Style Guide type scale). The active item gets accent-colored text, a subtly lightened background, and steps up to Semibold weight. Inactive items lighten their background and text color on hover. Both the hover and active tints are two fixed constants scoped to the sidebar's own navy surface — they are not the shared light/dark theme colors.
- An optional numeric count badge (a small rounded pill in Extra-Small, Semibold text) may sit at the end of a nav item to show a live count for that section; cap displayed values at "99+".

### 2.3 Product switcher (if the shell hosts more than one product/workspace)

When a single shell hosts multiple distinct products or workspaces under one navigation rail, the sidebar header becomes a **switcher** rather than a static logo: a dropdown trigger showing the current product's icon and wordmark, opening a small menu that lists every available product (icon, name, and a checkmark on the active one). Selecting a different product swaps both the active area and the nav item list below it — nav items are product-scoped, not global.

Each product gets its own icon glyph inside a rounded icon tile (roughly 32px) filled with a two-stop diagonal gradient, so products stay visually distinguishable from each other in the switcher menu.

### 2.4 Sidebar brand logo

The bottom-most element (expanded state only) is a centered logo lockup, capped at a fixed height (not width) with a maximum-width guard, sitting above a dividing line for separation. It is omitted entirely in the collapsed state.

---

## 3. Top Header (bare/public shell only)

Used only on routes that render without the sidebar (e.g. sign-in). A sticky bar pinned to the top of the viewport:

- Fixed height of 64px, staying pinned as the page scrolls beneath it, with a translucent background (roughly 90% opaque) over a soft background blur, and a hairline border along its bottom edge.
- Inner row: horizontally centered, capped at a maximum width of 1400px, with side padding of 16px that steps up to 32px at the Small breakpoint (see §7); content spreads to the far left and right edges of the row, vertically centered.
- Left: logo (fixed height, proportional width) followed by a thin vertical divider and a bold wordmark.
- Right: minimal actions only (e.g. a theme toggle) — this header is intentionally sparse since primary navigation lives in the sidebar once authenticated.

---

## 4. Main Content Area

- No maximum-width constraint by default — content is fluid within the space left by the sidebar (this differs from the top header in §3, which does center at a fixed maximum width). Only apply a maximum width when a specific page template calls for it (e.g. a narrow form).
- Standard content padding: 24px on the sides, stepping up to 40px at the Small breakpoint; 24px at the top; extra clearance at the bottom (roughly 96px) so content never sits directly under the floating panel described in §6.
- A route may opt out of this padding entirely (e.g. a full-bleed content view) — treat the padded wrapper as the default, not a hard requirement.

### 4.1 Page header pattern

Every standard content page opens with the same header shape:

```
┌──────────────────────────────────────────────┐
│  Page Title (large, bold)        [Primary    │
│  One-line description (small,     Action]    │
│  secondary text color)                        │
└──────────────────────────────────────────────┘
```

- The title block sits on the left, a single primary action sits on the right, both vertically centered on the same row. On narrow viewports, prefer hiding the action's label (icon-only) or hiding the action entirely over letting the header wrap to a second line.
- Title uses the 2X Large size at Bold weight (see Style Guide type scale); the description sits directly beneath it in Small size, Secondary Text color, with a small gap between the two lines.
- Below the header, page sections stack with a consistent gap of roughly 20px between them — a single consistent rhythm across the page's top-level sections, rather than ad hoc margins on each one.

### 4.2 Page templates

Three recurring page shapes cover nearly all content routes:

1. **Browsable grid** — page header (§4.1), then a filter row (search input plus toggleable filter chips), then a responsive card grid (§5), then an infinite-scroll trigger at the bottom of the list. Loading and empty states render in the same grid slot the cards would occupy, never as a separate layout.
2. **Detail view** — a two-column split once space allows: a flexible-width main column alongside a fixed 280px-wide metadata/actions rail, separated by a 24px gap. Below the Large breakpoint (§7) these collapse into a single stacked column, main content first. The rail should size to its own content rather than stretching to match a taller main column.
3. **Standalone/full-bleed view** — no page-header wrapper, no side padding — used when a route needs the entire content area for itself (e.g. a full-screen presentation mode).

---

## 5. Grid System (card listings)

Card/tile listings use a fixed responsive column progression — do not introduce ad hoc column counts:

| Breakpoint | Columns |
|---|---|
| default (narrowest) | 1 |
| Small (≥640px) | 2 |
| Large (≥1024px) | 3 |
| Extra Large (≥1280px) | 4 |

The gap between cards is a flat 16px at every breakpoint. Loading skeletons and empty/no-results states occupy the same grid container (e.g. a skeleton card repeated, or a single message spanning the grid) rather than swapping to a different layout container.

---

## 6. Floating Action Panel

A persistent, bottom-right-anchored panel for an in-progress, cross-page task (e.g. a running selection/build flow), independent of whatever page is currently active.

- Position: anchored 24px from the bottom and right edges of the viewport, stepping to 40px from the right at the Small breakpoint; always layered above page content and above the sidebar.
- Two states:
  - **Collapsed**: a small pill/circle button showing an icon, a count badge, and a chevron to re-expand. Filled with the same two-stop accent gradient as the product-switcher icon tiles (§2.3), for visual continuity.
  - **Expanded**: a fixed-width card (460px, capped at the viewport width minus 48px on narrow viewports) with an accent-colored top border, raised elevation (see Style Guide §4.2), and generously rounded corners; internally split into a scrollable row of "selected item" chips plus contextual controls, and a bottom row showing the resulting output with a minimize control.
- The panel only renders once the underlying task has at least one item in progress; it does not occupy space or show an empty state otherwise.
- Expand/collapse transitions are animated (a quarter- to a third-of-a-second ease), not instant, and the collapsed/expanded state is user-dismissible independent of clearing the underlying task.

---

## 7. Responsive Breakpoints

A standard breakpoint scale, expressed here by name rather than tied to any particular framework's naming convention:

| Name | Min width |
|---|---|
| Small | 640px |
| Medium | 768px |
| Large | 1024px |
| Extra Large | 1280px |
| 2X Large | 1536px |

Usage guidance:
- **Small** is the primary mobile→tablet pivot: content padding steps up, card grids go from 1 to 2 columns, secondary header actions reappear.
- **Large** is the primary tablet→desktop pivot: card grids go to 3 columns, and any two-column detail/split layout (§4.2) collapses below this point and reflows above it.
- **Extra Large** is reserved for widening the card grid further (3 to 4 columns) on large monitors — avoid introducing new structural behavior at this breakpoint beyond column count.
- **Medium** and **2X Large** exist in the scale but are not used as behavioral pivots by any pattern in this guide — reserve them for cases a specific page genuinely needs.
- The sidebar itself does not change behavior at any breakpoint in this pattern — it is collapse-toggled by the user, not by viewport size. If adapting this shell for small-viewport/mobile-first use, treat sidebar auto-collapse-below-Large as an extension to design deliberately, not an assumed default.

---

## 8. Spacing Rhythm

A small, consistent set of spacing steps is reused throughout the layout, rather than arbitrary one-off values:

| Step | Value |
|---|---|
| Extra tight | 2px |
| Tight | 4px |
| Compact | 6px |
| Small | 8px |
| Cozy | 12px |
| Default | 16px |
| Comfortable | 20px |
| Loose | 24px |
| Wide | 32px |
| Extra wide | 40px |

Usage guidance:
- **Icon-to-label gaps**: Compact-to-Cozy (6–12px), depending on density — tight chips use the smaller end, nav items use the larger end.
- **Section stacking**: Comfortable (20px) between top-level page sections; tighter internal stacks (e.g. within a card) use Tight-to-Small (4–8px).
- **Control padding**: compact controls (badges, chips) use roughly 10px horizontal / 2px vertical; standard buttons and inputs use Cozy-to-Default (12–16px) horizontal padding, sized to the height steps defined for buttons in the Style Guide.
- Prefer the step values above over introducing new arbitrary spacing values unless a specific alignment problem genuinely requires it.

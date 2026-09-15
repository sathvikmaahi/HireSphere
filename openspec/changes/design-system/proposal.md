# Design System

## Why

Every screen in TalentSphere — nine roles, five phases, seventy-nine backlog items — renders
through the same visual language, and two supplied guides
(`reference/design-spec.md`, `reference/layout-spec.md`) define that language precisely enough
that nothing about it needs inventing. What does not exist is any expression of it in code: the
frontend is the Sprint 0 scaffold, whose `App.tsx` says in a comment that it deliberately carries
no layout, color, or typography because *"anything styled now would be built from raw values the
token set is meant to replace, and would have to be torn out."* That deferral is now due.

**Why before almost everything else.** `AGENTS.md`'s standing quality bar forbids a raw hex value
or a one-off spacing value anywhere in the product, and `design-system` is the feature that makes
that enforceable instead of aspirational. Every later feature that draws a table, a form, a shell,
or a toast either consumes what this feature builds or reinvents it — and `TS-BL-023`, the Admin
Cockpit, is already recorded in `exploration-notes.md` D.9 as blocked on this feature's shell.

## What Changes

Six backlog items, `TS-BL-007` through `TS-BL-012`, exactly as decomposed in
[D.9](../talentsphere/exploration-notes.md). Nothing in this feature exists yet; all six are new
work against an existing but bare React + TypeScript scaffold (`frontend/`, per `TS-BL-001`).

- **`TS-BL-007` Design tokens and the primitives that are nothing but a token expression.** The
  full token set — color, typography, spacing, shape, elevation, iconography — every color as an
  explicit light/dark pair, plus the theme runtime that resolves them and persists the user's
  choice, plus a build gate that fails on a raw hex outside the token definitions or a color token
  missing half its pair (D13). **Widened beyond its D.9 one-line title** to also carry Button,
  Badge, Icon and the §1.5 status-surface renderer, along with the AI-generated label, the
  human-review disclaimer, and the evidence-source tag. This is not a new boundary: the inherited
  `design-system/foundations` delta spec in `talentsphere-wave-1-foundation` already places
  buttons, badges, focus indication and brand assets in *foundations*, and D.9's five sibling items
  each depend on `TS-BL-007` and on nothing else — a graph that only holds if the primitives they
  all share live here. See `design.md` D2.
- **`TS-BL-008` Authenticated Shell.** Three shell states (authenticated, bare, presentation), the
  collapsible sidebar with its expanded and collapsed widths and persisted choice, the page header
  pattern, three page templates, the card-grid progression, the breakpoint scale, the floating
  action panel *region*, and the single landing route that requires no page grant (D17).
- **`TS-BL-009` Dense-data-table pattern.** The fourth page template the supplied guide does not
  have (D14, closing S.3's first gap): search, filter, sort, pagination, its own horizontal
  overflow container, compact type steps, and an export control gated on a caller-supplied
  permission decision.
- **`TS-BL-010` Floating Action Panel.** The bottom-right cross-page task panel from layout §6 —
  collapsed pill and expanded 460px card, selected-item chips, running output, animated transitions
  — mounted into `TS-BL-008`'s region. Its first real consumer is Priority Selection in Phase 3
  (S.1, D20); this feature builds the component, not that screen.
- **`TS-BL-011` Core form and input components.** Text input, textarea, select, checkbox, radio,
  switch, the form-field wrapper carrying label/description/error, and the validation summary —
  satisfying `UI-007`'s requirement that errors appear at both field and summary level.
- **`TS-BL-012` Notification and toast component.** Renders `platform-core`'s notification payload
  contract — severity, title, body, originating event reference, optional action reference — mapping
  severity onto the §1.5 status-surface triplets. `platform-core` owns the semantics; this feature
  owns the appearance, and that boundary is why the two can be built independently.

**Two corrections to the supplied guides are carried, not worked around.** Both were found in this
conversation and recorded in `exploration-notes.md`
[S.6](../talentsphere/exploration-notes.md) rather than patched silently here: §1.5's Success
status-surface triplet is character-for-character the Info triplet and orphans §1.4's Success
green, and the sidebar's mandated "dark navy surface" plus its two hover/active constants have no
token anywhere in the style guide. `TS-BL-007` implements the resolutions.

**Explicitly not in this change:** no screen belongs to this feature. Not the Admin Cockpit
(`TS-BL-023`), not the ranking board (`TS-BL-054`), not the permission matrix (`TS-BL-022`), not
Priority Selection. No component here calls an API, queries the permission evaluator, or subscribes
to the notification engine — permission and data decisions arrive as props from a consuming feature
(`design.md` D1). No candidate-facing surface of any kind. And no presentation-shell *screen*: the
state exists and stays unassigned, exactly as S.3 left it.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability is new. Two
of the six paths (`design-system/foundations`, `design-system/app-shell`) are the exact paths
`talentsphere-wave-1-foundation` already established and are reused rather than renamed, the same
way `platform-core` preserved `platform/delivery-foundation`. Four are new, because D.9's finer
decomposition splits work that change had bundled into `app-shell`.

- `design-system/foundations`: the token set as light/dark pairs, theme behavior and persistence,
  type and weight scales, iconography rules, shape and elevation, the button/badge/status-surface
  shape patterns, focus indication, the WCAG 2.1 AA baseline, spacing rhythm, brand asset usage,
  and the AI-disclosure and evidence-source labels built from those primitives. *(`TS-BL-007`)*
- `design-system/app-shell`: shell states, sidebar structure and collapse behavior, navigation and
  count badges, the utility footer, main content area rules, the page header pattern, the three
  card/detail/standalone page templates, the card grid progression, responsive breakpoints, the
  floating action panel region, and the landing route. *(`TS-BL-008`)*
- `design-system/data-table`: the dense data table as a first-class fourth page template — density,
  overflow containment, search/filter/sort/pagination, and permission-gated export. *(`TS-BL-009`)*
- `design-system/action-panel`: the floating action panel component itself — its two states,
  selected-item chips, running output, and conditional rendering. *(`TS-BL-010`)*
- `design-system/forms`: form controls, the field wrapper, and field-plus-summary validation
  presentation. *(`TS-BL-011`)*
- `design-system/notifications-ui`: the toast and in-app notification presentation, and the
  severity-to-status-surface mapping. *(`TS-BL-012`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**Overlap to resolve outside this change:** `talentsphere-wave-1-foundation` remains an active,
unarchived change whose delta specs cover `design-system/foundations` and `design-system/app-shell`.
Both changes therefore describe those two paths until that change is retired or split per D.5 — the
same accepted interim state `platform-core` recorded. That migration spans five features and is not
this change's to perform.

## Impact

**Code** — `frontend/` throughout, which today holds `src/App.tsx`, `src/main.tsx` and `src/api/`
and nothing else. New: a token layer, a theme provider, a component library, and the shell.
`App.tsx`'s deliberate placeholder is replaced by the shell; its `/build` call and the
`ApiError`/correlation-ID handling it demonstrates are preserved, because the error shape it renders
is `platform-core`'s global handler contract.

**Dependencies** — a router is required by `TS-BL-008` (shell state is route-driven, and deep links
must survive sign-in) and the scaffold has none. Self-hosted font files for Plus Jakarta Sans and
JetBrains Mono are required by §2.1 and must be vendored. Both interact with a live constraint:
`registry.npmjs.org` is blocked on the corporate network, so a developer there cannot `npm ci` at
all today (`sprint-0-outcome.md`) — CI is unaffected because the lockfile is canonical. See
`design.md` D8 for how this is handled rather than assumed away.

**Missing input, not a decision to make here** — `reference/design-spec.md` §5 names three brand
assets under `reference/assets/`; that directory does not exist in this repository. The logo slots
in the sidebar footer and the bare-shell header are specified and built, and render a text wordmark
until the files are supplied. Tracked as an open question, not silently substituted.

**Downstream features that block on this one** — `access-control-and-admin` (`TS-BL-023` needs the
shell; `TS-BL-021` and `TS-BL-022` consume the dense table), `identity-and-access` (the sign-in
screen is the bare shell's only route), `matching-and-ranking` (`TS-BL-054`'s ranking board is the
dense table plus the AI-disclosure components), `decision-and-offers` (Priority Selection mounts the
floating action panel), and every feature that renders a form or a toast.

**Upstream contract consumed** — `platform-core`'s `platform/notifications` "Delivery payload
contract" requirement. `TS-BL-012` depends on that contract, which is finished, and not on
`TS-BL-005`'s delivery mechanism, which is not built. `platform-core`'s own migration plan already
pulls task 5.4 forward for exactly this reason.

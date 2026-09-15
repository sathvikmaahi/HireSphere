# Design System — Backlog

This feature's complete backlog: six items, `TS-BL-007` through `TS-BL-012`, decomposed in
`exploration-notes.md` D.9. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`. Assigning
sprints here would be inventing a schedule this feature has no authority to set. Each item carries a
stated goal instead.

**Nothing here is built.** The `frontend/` scaffold exists from `TS-BL-001` — React 18, TypeScript,
Vite, strict lint and type configuration, Vitest, an `@/` alias — and `App.tsx` deliberately carries
no styling because this feature is what it was waiting for. This is a scaffold to build on, not an
empty directory.

**Shape of the dependency graph.** `TS-BL-007` is the only item any sibling depends on; the other
five are mutually independent and can proceed concurrently once it lands. That is a property of the
design (`design.md` D1: every component here is a pure renderer, so nothing needs the permission
evaluator, the notification engine or an API to exist), not a coincidence of ordering.

---

## 1. TS-BL-007 — Design tokens and the primitives that express them

**Goal:** one enforceable token set that every later feature consumes and no component bypasses,
plus the primitives that are nothing but a token expression. Covers `design-system/foundations`.

Scope note: this item is **wider than its D.9 one-line title**, which reads "Design tokens
(color/type/spacing/shape, light + dark)". It also carries Button, Badge, Icon, the status-surface
renderer, the focus and disabled treatments, and the three label components. `design.md` D2 gives
the reasoning: D.9's own graph has five siblings depending on this item and nothing else, which only
holds if the primitives they all share live here, and the inherited `design-system/foundations`
delta spec already drew the line in exactly this place.

```yaml
backlog_items:
  - id: TS-BL-007
    feature: design-system
    depends_on: []
    status: not-started
```

- [ ] 1.1 Declare the token source as typed data — color as explicit `{ light, dark }` pairs,
      plus type scale, weights, spacing steps, radii, elevations — from `design-spec.md` §1–§4 and
      `layout-spec.md` §8, with no value invented (design D3)
- [ ] 1.2 Add the `sidebar` token family with the six values fixed in design D4, declared
      theme-independent in the source so the pair-completeness check treats it as intentional rather
      than as a missing half
- [ ] 1.3 Add the corrected Success status-surface triplet from `exploration-notes.md` S.6, keeping
      Info on the accent-blue family, so no two statuses render identically
- [ ] 1.4 Add the `gradient.accent` token as a two-stop diagonal between the existing Accent and
      Accent Hover tokens (design D15), since the guide's only definition of it lives in the product
      switcher S.2 rules out
- [ ] 1.5 Generate CSS custom properties and typed accessors from the token source; theme resolves
      from one attribute on the document root
- [ ] 1.6 Build the theme runtime — switch, ~250ms transition, persistence across sessions — and
      assert a returning user's stored choice is honored without re-selection
- [ ] 1.7 Vendor and subset the Plus Jakarta Sans and JetBrains Mono web fonts to the weights §2.3
      uses; wire `@font-face` against the local files only
- [ ] 1.8 Add the build gate: fail on a color literal in component source outside the token
      definitions, on a color token missing a half without a theme-independent declaration, on an
      off-scale spacing value, and on any remote font reference (design D13)
- [ ] 1.9 Add contrast tests computing every text-on-surface pairing in both themes against
      `UI-009`'s 4.5:1 threshold, failing the build below it
- [ ] 1.10 Wire the token gate and contrast tests into the existing `lint:frontend` CI job alongside
      `eslint`, `format:check` and `typecheck`
- [ ] 1.11 Build the icon module: one internal re-export of a curated outline subset, enforcing the
      two size steps and color inheritance centrally so no component imports an icon directly
      (design D7)
- [ ] 1.12 Build Button — six variants, four size steps — from tokens only, with the uniform
      disabled treatment and the focus ring
- [ ] 1.13 Build Badge/pill and the §1.5 status-surface renderer, both variant-driven from tokens
- [ ] 1.14 Build the AI-generated label and the human-review disclaimer from Badge and the status
      surface; assert the disclaimer states human review and makes no bias-free claim
      (`UI-004`/`AI-001`, `RANK-004`/`UI-006`)
- [ ] 1.15 Build the evidence-source label distinguishing resume, interview, scorecard and
      human-decision sources by icon and text, with an unrecognized value rendering neutrally rather
      than failing (`UI-005`/`G-02`, S.3)
- [ ] 1.16 Build the brand slot: theme-swapped logo, fixed height, clear space — rendering a text
      wordmark while `reference/assets/` is absent, and never a substitute image (design D9)
- [ ] 1.17 Add the long-form prose treatment from §2.4 — relaxed line height, stepped headings,
      secondary-color body, styled inline code and code blocks, links underlined at rest

---

## 2. TS-BL-008 — Authenticated Shell

**Goal:** the shell every authenticated screen renders inside, complete enough that
`access-control-and-admin`'s Admin Cockpit can be built against it without inventing layout. Covers
`design-system/app-shell`.

Reading note: D.9's title says "sidebar 3-states". Per `design.md` D11 that means the layout guide's
three *shell* states plus the sidebar's two widths — there is no third sidebar width in either
guide, and a literal reading would send someone hunting for one.

```yaml
backlog_items:
  - id: TS-BL-008
    feature: design-system
    depends_on: [TS-BL-007]
    status: not-started
```

- [ ] 2.1 Add the router and define the route table with its three shell states, resolved in the
      guide's stated order — bare route first, then authentication (design D8, D10)
- [ ] 2.2 Assert in a test that sign-in is the **only** route rendering without authentication, so
      the "no public routes" non-goal is a route-table property rather than a convention (S.2)
- [ ] 2.3 Build the authenticated shell: full-viewport height, sidebar fixed, only the main content
      column scrolling
- [ ] 2.4 Build the bare shell and its sticky top header (layout §3) for sign-in, with the brand
      slot and a theme toggle and nothing else
- [ ] 2.5 Build the presentation shell — chrome removed, content full-viewport — and leave it
      unassigned to any screen, as S.3 leaves it
- [ ] 2.6 Build the sidebar at both widths on the `sidebar` token family, with the transition
      timings §2.2 gives and the user's choice persisted across sessions, defaulting to expanded
- [ ] 2.7 Build the sidebar header with product identity and a menu toggle — explicitly no product
      switcher (S.2)
- [ ] 2.8 Build the navigation list: icon plus label, distinct active and hover treatments, capped
      count badges, collapsed state showing centered icons with accessible names
- [ ] 2.9 Render navigation strictly from the supplied list, defaulting to empty when none is given,
      and assert no permission evaluation happens in the shell (design D1, D6)
- [ ] 2.10 Build the utility footer — theme toggle, user menu with the signed-in identity and
      sign-out, brand slot in the expanded state only
- [ ] 2.11 Assert that switching the content theme leaves the sidebar's surface, hover and active
      colors unchanged (layout §2)
- [ ] 2.12 Build the main content area: fluid width, the defined padding steps, bottom clearance
      clearing the panel region, and a full-bleed opt-out
- [ ] 2.13 Build the page header pattern — title, one-line description, at most one primary action,
      never wrapping to a second row on narrow viewports
- [ ] 2.14 Define the page-template contract and build the browsable-grid, detail-view and
      standalone templates against it, leaving the fourth to `TS-BL-009` (design D12)
- [ ] 2.15 Build the card grid's fixed column progression and constant gap, with loading and empty
      states inside the grid container
- [ ] 2.16 Build the floating action panel region — bottom-right anchored, layered above content and
      sidebar, rendering nothing when nothing is mounted
- [ ] 2.17 Build the landing route requiring no page grant, showing an empty state and a route to
      request access for a user with no grants at all (D17)
- [ ] 2.18 Preserve deep links through sign-in so an authorized requested route wins over the
      landing route
- [ ] 2.19 Build the workflow-state presentation — state, owner, next action, blockers, last updated
      — from values supplied by the caller (`UI-003`)
- [ ] 2.20 Replace `App.tsx`'s Sprint 0 placeholder with the shell, keeping its `/build` call and
      its `ApiError` and correlation-ID rendering as the shell's error presentation rather than
      discarding `platform-core`'s error contract
- [ ] 2.21 Verify keyboard traversal of the whole shell — nav, footer, user menu, theme toggle —
      with visible focus at every step (`UI-009`)

---

## 3. TS-BL-009 — Dense-data-table pattern

**Goal:** the fourth page template the supplied guide does not have, so the permission matrix, the
ranking board and the audit logs are not each a bespoke table. Covers `design-system/data-table`.

```yaml
backlog_items:
  - id: TS-BL-009
    feature: design-system
    depends_on: [TS-BL-007]
    status: not-started
```

- [ ] 3.1 Build the table from existing tokens only, at the compact type steps, and assert no new
      token was introduced for it (S.3, D14)
- [ ] 3.2 Contain horizontal overflow inside the table's own container, and assert the page body
      never scrolls horizontally with a wide table present
- [ ] 3.3 Implement search, filtering, sorting and pagination, with the sorted column indicating
      direction (`UI-008`)
- [ ] 3.4 Render loading and empty states inside the table container rather than swapping layout
- [ ] 3.5 Gate the export control on a caller-supplied decision, absent by default, and assert the
      table performs no permission evaluation itself (design D1, D6)
- [ ] 3.6 Verify keyboard traversal of interactive cells with visible focus, and header-to-cell
      association for screen readers, at the compact density (`UI-009`)
- [ ] 3.7 Verify the table conforms to `TS-BL-008`'s page-template contract **and** renders
      correctly outside one — inside a detail view's main column, which is how the audit log will
      use it (design D12)

---

## 4. TS-BL-010 — Floating Action Panel

**Goal:** the cross-page in-progress-task panel, built once so Priority Selection consumes it rather
than inventing it. Covers `design-system/action-panel`.

Scope note: this item builds the component. Priority Selection — the workflow S.1 identified as its
close match, and `D20`'s 1-to-5 slots per vacancy — belongs to `decision-and-offers`, and none of
its semantics are embedded here.

```yaml
backlog_items:
  - id: TS-BL-010
    feature: design-system
    depends_on: [TS-BL-007]
    status: not-started
```

- [ ] 4.1 Build the collapsed state — icon, count badge, re-expand control — filled with the
      `gradient.accent` token from task 1.4
- [ ] 4.2 Build the expanded state at the fixed width, capped to the viewport on narrow screens,
      with the accent top border, raised elevation and large radius
- [ ] 4.3 Build the scrollable chip row with contextual controls, scrolling within the panel rather
      than widening it
- [ ] 4.4 Build the output row with its minimize control, reflecting the current selection
- [ ] 4.5 Render nothing at all when the underlying task has no items — no empty state, no
      reserved space
- [ ] 4.6 Animate expand and collapse at the timing layout §6 gives, and keep the collapse state
      independent of clearing the task
- [ ] 4.7 Persist the panel and its contents across page navigation while the task is in progress
- [ ] 4.8 Take items, controls and output as inputs and invoke supplied handlers — assert the panel
      owns and mutates no task state (design D1)
- [ ] 4.9 Verify it mounts into `TS-BL-008`'s panel region and layers above both page content and
      the sidebar

---

## 5. TS-BL-011 — Core form and input components

**Goal:** every data-entry screen in the product assembled from one control set, with validation
presented the way `UI-007` requires. Covers `design-system/forms`.

```yaml
backlog_items:
  - id: TS-BL-011
    feature: design-system
    depends_on: [TS-BL-007]
    status: not-started
```

- [ ] 5.1 Build text input, multi-line text, select, checkbox, radio group and toggle from tokens
      only, at the defined control heights and padding steps
- [ ] 5.2 Build the field wrapper carrying label, optional description and optional error, with the
      label programmatically associated and errors announced to assistive technology
- [ ] 5.3 Build the validation summary listing every field error together, with activation moving
      focus to the corresponding field (`UI-007`)
- [ ] 5.4 Clear a field's error and its summary entry together when the field becomes valid
- [ ] 5.5 Render server-returned field errors identically to client-detected ones, so a submission
      refused by `platform-core`'s global error handler presents through the same path
- [ ] 5.6 Apply the uniform disabled treatment across every control — reduced opacity, no pointer
      interactivity, no bespoke disabled palette
- [ ] 5.7 Verify keyboard-only completion of a representative form and visible focus on every
      control (`UI-009`)
- [ ] 5.8 Assert the controls hold no submission logic, call no API, and derive no validation rules
      of their own (design D1)

---

## 6. TS-BL-012 — Notification and toast component

**Goal:** one rendering surface for every notification the product will ever send, mapping
`platform-core`'s severity semantics onto this design system's status surfaces. Covers
`design-system/notifications-ui`.

**On the cross-feature dependency.** D.9 records `TS-BL-012` as depending on `TS-BL-005`, and
`platform-core`'s design confirms what that dependency actually is: the **payload contract**, which
is a finished spec requirement, not `TS-BL-005`'s delivery mechanism, which is unbuilt. That
change's own migration plan pulls its task 5.4 forward for exactly this reason. If no shared type
exists in code when this item starts, mirror the contract from the spec and replace it when one
does.

```yaml
backlog_items:
  - id: TS-BL-012
    feature: design-system
    depends_on: [TS-BL-007, TS-BL-005]
    status: not-started
```

- [ ] 6.1 Express `platform/notifications`' delivery payload contract as a type — severity, title,
      body, originating event reference, optional action reference — consumed as given, not
      re-derived
- [ ] 6.2 Build the toast on the status-surface renderer, rendering title, body and any action
      without feature-specific handling
- [ ] 6.3 Map severity onto the status-surface triplets, using the corrected Success triplet from
      task 1.3, and assert a success and an informational notification differ visually
- [ ] 6.4 Fall back to the neutral surface for an unrecognized severity and assert it — the payload
      contract promises new notification types render without modifying existing surfaces, and this
      failure mode is invisible until production (design D5)
- [ ] 6.5 Render without an action control when no action reference is present, rather than a
      disabled or empty one
- [ ] 6.6 Make toasts dismissible and announce them to assistive technology at an urgency matching
      severity (`UI-009`)
- [ ] 6.7 Verify a toast and the floating action panel do not obscure each other, since both claim
      the bottom-right corner
- [ ] 6.8 Bound the concurrent toast stack, keeping the remainder retrievable on the in-app
      notification surface rather than discarding them silently (S.5)
- [ ] 6.9 Build the in-app notification surface presenting supplied notifications, rendering
      references as given and resolving none itself
- [ ] 6.10 Assert the components own no subscription, mark nothing read, and retrieve no history —
      that is the consuming feature's, and is what lets this item ship ahead of delivery (design D1)

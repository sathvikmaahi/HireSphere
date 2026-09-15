# Design System — Design

## Context

See `proposal.md` — Why. What shapes the approach beyond that:

- **The visual language is supplied, not designed here.** `reference/design-spec.md` and
  `reference/layout-spec.md` are complete enough that almost no visual decision is open. Both are
  explicitly product-agnostic (Part S), so the decisions this document makes are about *expression*
  — how a supplied value becomes an enforceable token — and about the handful of places the guides
  are silent, contradictory, or inapplicable to a single-product recruiting tool.
- **The frontend exists but is bare.** `TS-BL-001` shipped React 18 + TypeScript on Vite with
  strict `tsconfig`, ESLint `strictTypeChecked`, Prettier, Vitest and an `@/` path alias. There is
  no router, no styling layer, no component beyond `App.tsx`, whose own comment defers styling to
  this feature precisely so nothing would need tearing out. This is a scaffold to build on, not an
  empty directory and not a codebase to refactor.
- **Two guide defects were found in this conversation** and fixed in `exploration-notes.md`
  [S.6](../talentsphere/exploration-notes.md) rather than here: §1.5's Success status-surface
  triplet duplicates Info's, and the sidebar's mandated navy surface has no token. D4 and D5 below
  implement those resolutions.
- **`platform-core` is proposed and its notification payload contract is a finished requirement**,
  while its delivery mechanism (task 5.4 onward) is not built. `TS-BL-012` is designed against the
  contract, which is all propose-stage design needs.
- **`registry.npmjs.org` is blocked on the corporate network** (`sprint-0-outcome.md`). The
  committed lockfile is canonical and CI is unaffected; a developer on that network already cannot
  `npm ci` at all. This constrains what a new dependency costs — see D8.
- **`reference/assets/` does not exist**, although `design-spec.md` §5 specifies three brand assets
  in it. Handled as a missing input (D9), not as licence to substitute artwork.

## Goals / Non-Goals

**Goals:**

- One token set, generated from one typed source, that every later feature consumes and no
  component bypasses — enforced by a build gate rather than by review (D13).
- Six backlog items that are genuinely independent of each other beyond their shared dependency on
  the token layer, matching D.9's graph exactly rather than quietly adding edges.
- Components that are safe to build now and correct later: nothing here needs the permission
  evaluator, the notification engine, or any API endpoint to exist, and nothing here becomes wrong
  when they do.
- Every requirement traceable to a `reference/spec.md` UI requirement, a guide section, or a
  `D`/`C`/`G`/`S` decision.

**Non-Goals:**

- No product screen. The first consumer of each component is named, and belongs to another feature.
- No design work. Where the guides decide, they decide; this document does not re-open colors,
  sizes, or breakpoints it merely expresses.
- No visual regression / screenshot infrastructure, and no published component playground. Both are
  defensible later; neither is needed for six items whose correctness is mostly assertable in unit
  tests and a token validator.
- No mobile-first adaptation. `UI-001` says desktop-first, and layout §7 explicitly makes
  sidebar-auto-collapse an extension to design deliberately rather than assume.

## Decisions

### D1 — Every component in this feature is a pure renderer: no API, no permission query, no subscription

A `design-system` component receives what it needs as props and returns markup. It does not fetch,
does not consult the permission evaluator, and does not subscribe to a notification stream. A
consuming feature owns the data and the authorization decision and passes the result in — a table
receives `canExport: boolean`, the shell receives an already-filtered list of navigation items, the
toast receives a payload object.

*Why:* three separate things fall out of one rule.

1. **D.9's dependency graph becomes true rather than aspirational.** Five of this feature's six
   items are recorded as depending on `TS-BL-007` and nothing else. A table that queried Export
   permission itself would depend on `access-control-and-admin`'s `TS-BL-018`, which is scheduled
   after this entire feature; a shell that filtered its own navigation would too. The graph would
   have to be corrected — or, worse, silently violated.
2. **`UI-002` stays honest.** The rule is *hide what the user cannot do, and keep the server
   authoritative*. A component that decides visibility from its own client-side evaluation
   invites the reading that the client is the enforcement point. Receiving a decision as a prop
   makes the client unambiguously a renderer of someone else's verdict.
3. **It is what makes these components testable without a backend**, which matters when only Local
   and Dev exist and the shell must be verifiable in neither.

*Alternative considered — a data-aware component library* (components that fetch their own rows,
resolve their own permissions, subscribe to their own notifications). Rejected: it inverts the
dependency direction of the whole Phase-1 plan, and every consuming feature would inherit
`design-system`'s choice of data layer whether or not it fit.

*Consequence, stated plainly:* `TS-BL-012` is a component that renders a notification, not a
notification centre. Somebody must own the subscription that feeds it. That is the consuming
feature's, and the first one is `access-control-and-admin`'s Admin Cockpit.

### D2 — Where the line falls between `TS-BL-007` and `TS-BL-011`, and why `TS-BL-007` is wider than its title

`TS-BL-007` carries the token set **plus** the primitives that are a direct expression of it and
carry no independent state: Button (six variants × four size steps, design §4.3), Badge/pill, Icon,
the §1.5 status-surface renderer, the focus/selection ring, and the uniform disabled treatment. It
also carries the three label components those primitives compose into — the AI-generated label
(`UI-004`/`AI-001`), the human-review disclaimer (`RANK-004`/`UI-006`), and the evidence-source tag
(`UI-005`/`G-02`).

`TS-BL-011` carries the controls that hold state and participate in validation: text input,
textarea, select, checkbox, radio, switch, the field wrapper, and the validation summary.

*Why not keep `TS-BL-007` literally "design tokens" as D.9's title reads:* because the same title
column also records that `TS-BL-008`, `009`, `010`, `011` and `012` each depend on `TS-BL-007`
alone. The sidebar needs a Badge for its count pills and a Button for its toggle; the table needs a
Badge in cells and a Button for export; the panel needs both for its chips and controls. If those
primitives lived in `TS-BL-011`, four items would gain a dependency edge D.9 does not have, and
`TS-BL-011` would become a bottleneck in front of the entire feature. Widening the first item is the
smaller correction, and D.11 explicitly permits a feature's propose conversation to refine its own
item boundaries.

*This is not a new boundary being invented.* `talentsphere-wave-1-foundation`'s inherited
`design-system/foundations` delta spec already places component shape patterns (buttons, badges,
the disabled state), focus indication, the accessibility baseline and brand assets in *foundations*
— exactly this line. The only thing moved is the three label components, which that change had put
in `app-shell`. They move because they are badge and status-surface compositions with no layout and
no shell involvement; leaving them in `app-shell` would make the application shell the owner of
content labeling, which is not a shell's job.

*Alternative considered — split `TS-BL-007` into a token item and a primitives item.* Rejected:
D.9's item numbering is referenced across the delivery plan, and this feature's brief is six items,
`TS-BL-007`–`TS-BL-012`. The tasks within `TS-BL-007` are sequenced so tokens land and pass their
build gate before the first primitive consumes them, which is what a split would have bought.

### D3 — Tokens are CSS custom properties generated from one typed TypeScript source; no CSS framework

A single TypeScript module declares every token as data — each color as an explicit
`{ light, dark }` pair — and a generator emits (a) the CSS custom property declarations under
`:root` and `[data-theme="dark"]`, and (b) the typed accessors components import. Components
reference tokens; they never write a value. Theme switching sets one attribute on the document
root, so a change is one paint, and the ~250ms color transition design §1.6 asks for is a single
CSS rule rather than per-component animation.

*Why one typed source rather than hand-written CSS:* D13 requires a build gate that fails on a raw
hex outside the token definitions *and* on any color token missing its light or dark half. Both
checks need the tokens to be enumerable data at build time. Hand-written CSS makes the first check
a fragile regex and the second impossible.

*Why not a CSS framework.* Tailwind or similar would mean a second source of truth (its config)
that a build gate would then have to reconcile against the first, and the guides specify a small,
closed set of values whose whole point is that it is *not* extended ad hoc — the generative
utility model optimizes for the opposite. It is also a substantial dependency added under D8's
constraint for no capability we lack.

*Why not CSS-in-JS.* Runtime style injection buys dynamic theming we do not need — the theme is one
attribute and two value sets — at the cost of a runtime dependency and a serialization step on
every render.

*What components use instead:* CSS Modules, already supported by the existing Vite setup with no
new dependency, scoping class names per component with values pulled from the custom properties.

### D4 — The sidebar gets its own theme-independent token family, with values fixed here

Layout §2 requires the sidebar to sit on a fixed dark navy surface outside the light/dark toggle,
and §2.2 requires two further fixed constants for its hover and active tints. `exploration-notes.md`
S.6 records that none of these exist in the style guide, which simultaneously forbids inventing a
one-off color and permits extending the set. This design extends it — a named `sidebar` family,
defined once, theme-independent by construction rather than by a component opting out of theming:

| Token | Value | Source |
|---|---|---|
| `sidebar.surface` | `#101A2C` | New — the navy §2 mandates and §1 omits |
| `sidebar.border` | `#1E2B42` | New — the dividing lines §2.1 requires between header, nav and footer |
| `sidebar.text` | `#C3CBD9` | New — inactive nav label; 10.7:1 on `sidebar.surface` |
| `sidebar.text-active` | Accent `#00AAE7` | Existing token, §2.2's "accent-colored text"; 6.6:1 |
| `sidebar.hover-tint` | `rgba(255,255,255,0.06)` | New — §2.2's "lighten their background" constant |
| `sidebar.active-tint` | `rgba(0,170,231,0.14)` | Derived from Accent — §2.2's "subtly lightened background" |

Only four genuinely new values, two of them alpha overlays, all named and none writable at a call
site. Contrast ratios above are computed against `sidebar.surface` and clear `UI-009`'s 4.5:1
threshold.

*Why fix concrete values here rather than defer:* every one of them is required by
`TS-BL-008`, which is the second item built. Deferring would mean building the sidebar against
placeholders and revisiting it, which is exactly what `App.tsx`'s Sprint 0 comment was avoiding.

*Consequence for the token validator:* it must treat the `sidebar` family as intentionally
single-valued, or D13's "every color token has a light and a dark half" check will fail the build on
the one family the layout guide requires to have neither. The exemption is declared in the token
source as a property of the family, not special-cased in the checker.

### D5 — Severity is `platform-core`'s enumeration; appearance is this feature's mapping; unknown values render Neutral

`platform-core`'s `platform/notifications` "Delivery payload contract" requirement fixes the shape
`TS-BL-012` renders: severity, title, body, originating event reference, optional action reference.
That shape is consumed as given and not re-derived. This feature owns only the mapping from severity
onto the §1.5 status-surface triplets, and it does so with the resolution from
`exploration-notes.md` S.6 applied — otherwise a success toast and an info toast would be
indistinguishable, making the enumeration decorative at the one place a user meets it:

| Severity | Status surface | Note |
|---|---|---|
| success | Success triplet **as corrected in S.6** | `#E9F9EF`/`#8FE0AE`/`#0F6B36` light, `#04170C`/`#17512F`/`#22C55E` dark |
| info | Info triplet | Unchanged — keeps the accent-blue family |
| warning | Warning triplet | Unchanged |
| error | Error triplet | Unchanged |
| *anything else* | Neutral triplet | See below |

*Why an unknown severity must render rather than throw.* `platform-core`'s spec carries the
scenario *"a feature introduces a new notification type → existing rendering surfaces present it
without modification."* A mapping that only handles four names breaks that guarantee the first time
a Phase-3 feature adds a fifth. The toast therefore falls back to the Neutral triplet — which §1.5
supplies precisely for status communication with no semantic color — and the fallback is asserted
in a test, since the failure mode is invisible until it happens in production.

*Why the boundary sits here and not in `platform-core`:* recorded in that feature's `design.md` D9 —
defining severity as an enumeration rather than a color is what stops the boundary inverting. This
design accepts that split rather than restating it; the practical payoff is that `TS-BL-012` needs
only the contract, so it is unblocked now even though `TS-BL-005`'s delivery mechanism is unbuilt.

*Note that severity's member list is not enumerated anywhere in `platform-core`'s spec.* The four
names above are this feature's reading of §1.5's status set, and the Neutral fallback is what makes
that reading safe to be wrong about. If `platform-core`'s implementation lands a different member
list, the fix is a mapping table entry, not a design change — and per `AGENTS.md` it would be an
`/opsx:update` against this feature.

### D6 — Permission-sensitive affordances are props with an explicit default of *absent*

Where a component has an affordance that permissions govern — the table's export control, the
shell's navigation list, any action in the page header — the component takes the decision as input
and renders nothing when it is not supplied. Not "renders it enabled"; not "renders it disabled".

*Why default-absent:* the alternative default is a control that appears for a user nobody has
authorized yet, which is the most common state during `access-control-and-admin`'s own rollout.
Deny-by-default is already the architectural rule for the evaluator (`config.yaml`); a renderer
whose default was permissive would be the one place in the stack that inverted it. The server
remains authoritative regardless — `UI-002`'s second clause — so a hidden control that is invoked
directly is still refused.

### D7 — Icons come from one internal module wrapping an outline set, never imported directly

Components import icons from a single internal module that re-exports a curated subset and enforces
the two size steps design §3 defines (14–16px in chrome, 20–24px for a lone emphasis control). No
component imports an icon package directly.

*Why the indirection is worth a file:* §3's rules — never mix sizes within a region, never mix
outline with filled, always inherit the surrounding text color — are unenforceable if every
component reaches for its own import. Centralizing makes them a property of the module, and makes
the underlying set swappable without touching call sites, which matters under D8.

*Which set:* an outline/stroke icon library with consistent stroke weight and rounded joins and
endpoints, which is §3's description almost verbatim. The alternative — hand-vendoring SVGs as
components — avoids a dependency but means someone draws or sources every icon the product ever
needs, one screen at a time, with no consistency guarantee. Rejected as a false economy.

### D8 — Three dependencies are added deliberately, under a blocked-registry constraint that changes nothing about how

`TS-BL-008` needs a **router** — shell state is route-driven (layout §1: bare route first, then
authentication), deep links must survive sign-in, and the landing route is a route. The scaffold has
none. `TS-BL-007` needs an **icon set** (D7) and **font files** for Plus Jakarta Sans and JetBrains
Mono (§2.1).

*The constraint, stated accurately:* `registry.npmjs.org` is filtered by hostname on the corporate
network. CI is unaffected — the committed lockfile is canonical and the pipeline resolves against
it — and a developer on that network cannot `npm ci` *today*, with zero project dependencies
installed. Adding three does not change that story; it is the same blocker either way, and it
belongs to network access, not to this feature.

*What this design does about it:* nothing clever, and deliberately so. Each addition is pinned in
the lockfile, and the lockfile is what CI installs from. Fonts are **vendored into the repository as
files** rather than fetched from a font CDN at runtime — the CDN would be an external egress on
every page load, from browsers on the same filtered network, for an asset the guide names
specifically. A build check asserts the font files are present and that no stylesheet references a
remote font host, because a silently-missing web font degrades to a fallback that looks almost
right, which is the hardest kind of visual regression to notice.

*Alternative considered — no router, hash-based navigation in the shell.* Rejected: it re-implements
route matching, nested layouts and redirect-after-sign-in inside `TS-BL-008`, which is more code
with more edge cases than the dependency avoids, and `identity-and-access` would then build sign-in
against a bespoke navigation model.

### D9 — The brand assets are missing; the slots are built and render a wordmark until they arrive

`design-spec.md` §5 names `assets/logo-dark.png`, `assets/logo-light.png` and
`assets/brand-mark.svg`; `reference/assets/` does not exist in this repository. The sidebar footer
lockup (layout §2.4) and the bare-shell header lockup (§3) are specified and built with their
sizing, clear-space and theme-swap rules intact, and render a text wordmark in the specified weight
until the files are supplied. Dropping the files in later is a file copy, not a change to either
component.

*Why not substitute artwork:* a logo is an identity, not a design token. Inventing one and shipping
it into a shell every screen renders is the kind of thing that survives to production because it
looks finished. A visible text wordmark does not.

### D10 — Shell state resolves in the guide's stated order, and the landing route can never deny

Layout §1 fixes the evaluation order: is this a bare/public route, then is the visitor
authenticated. That order is implemented literally. Per S.2, the bare shell's "embeddable public
document" case is dropped outright — **sign-in is the only route that renders without
authentication**, which is the standing non-goal that no public routes exist, expressed as a route
table property rather than a convention.

The landing route (D17) requires no page grant. A user with no grants at all reaches it and sees an
empty state with a route to request access — never an access denial. Its populated content (the
signed-in user's own tasks and notifications) belongs to whichever later feature supplies that data;
this feature builds the route and the empty state.

*Why the no-grant rule is structural rather than a convenience:* the most common user state during
`access-control-and-admin`'s rollout is *activated but not yet granted anything*. Routing that user
to a permissioned page produces a dead end at exactly the moment an administrator is trying to fix
their access.

### D11 — D.9's "sidebar 3-states" is read as three *shell* states plus two sidebar widths

`TS-BL-008`'s title in D.9 is *"Authenticated Shell (sidebar 3-states)"*. Layout §1 defines three
**shell** states (authenticated, bare, presentation) and §2.2 defines two **sidebar** widths
(224px expanded, 56px collapsed). There is no third sidebar width anywhere in either guide; the
sidebar's third condition is simply not being rendered, in the two non-authenticated shells.

Read as shorthand for "the shell's three states, including the sidebar's own two," which is how the
inherited `design-system/app-shell` delta spec already states it. Recorded because a literal reading
would have someone hunting for a third sidebar width that does not exist. This is a loose title, not
a contradiction — no correction to `exploration-notes.md` is warranted.

### D12 — The dense table is a component that a page template composes, which is why it depends only on tokens

D14 calls the dense data table "a fourth page template," and S.3 asks for a fourth template rather
than a repurposed card grid. Both are about *composition*: the table is a component, and it becomes
the fourth template when the shell's page-header-plus-content-area contract wraps it.

So `design-system/app-shell` specifies the page-template contract and the three card/detail/
standalone templates, and `design-system/data-table` specifies the table plus its conformance to
that contract. This keeps D.9's `TS-BL-009 → TS-BL-007` edge correct — the table renders anywhere,
including inside a detail view's main column, which is how `TS-BL-021`'s audit log will use it —
without either item waiting on the other.

*Density is not a style choice here:* row and cell text use the Extra Small step, which design §2.2
names as intended for "dense table cells." The overflow container is the table's own, so a wide
table scrolls within itself and the page body never scrolls horizontally — the failure this pattern
exists to prevent.

### D13 — The token gate is a build failure, not a lint warning

A build step fails on: a raw hex, `rgb()`, or `hsl()` literal in any component source outside the
token definitions; a color token missing its light or dark half (excepting families declared
single-valued, per D4); and a spacing value outside §8's ten steps in any component source. It runs
in CI alongside `lint:frontend`'s existing `eslint` / `format:check` / `typecheck`.

*Why a gate and not a convention:* this is D13's own reasoning and `AGENTS.md`'s standing quality
bar — "no raw hex values, no one-off spacing values, no component that bypasses the design tokens"
is unenforceable by review across twelve features, five phases and a team of 3–4 junior developers.
A one-off hex is a two-character diff that looks correct in every screenshot.

*Contrast is asserted, not assumed.* `UI-009` requires WCAG 2.1 AA, so the token tests compute
contrast ratios for every text-on-surface pairing in both themes and fail below 4.5:1. This is the
check that would have caught S.6's Success/Info collision as a *duplicate*, and it is why the
corrected values in D4 and D5 are quoted with their computed ratios rather than eyeballed.

### D14 — What this feature inherits from `talentsphere-wave-1-foundation`, and what it leaves behind

Per D.5 as adjusted to feature level by D.8.4, that change splits rather than being rewritten. This
feature inherits its **D13** (tokens as single source of truth, generated once, build-gated),
**D14** (dense table as a fourth template) and **D17** (one landing route, thin at first), which
`platform-core`'s D11 explicitly assigned here, and the delta specs `design-system/foundations` and
`design-system/app-shell` — whose capability paths are preserved exactly rather than renamed.

Content that change had bundled into `app-shell` is redistributed to match D.9's finer items: the
dense table to `design-system/data-table`, the floating panel component to
`design-system/action-panel`, validation presentation to `design-system/forms`, and the AI
disclosure components to `design-system/foundations` (D2). Nothing is dropped.

*What this change does not do:* retire or archive `talentsphere-wave-1-foundation`. Two active
changes therefore describe `design-system/foundations` and `design-system/app-shell` until that
migration completes across all five Phase-1 features — the same accepted interim state
`platform-core` recorded, for the same reason.

### D15 — The floating panel's accent gradient is defined from existing tokens, not inherited from a component we are not building

Layout §6 specifies the collapsed panel filled with "the same two-stop accent gradient as the
product-switcher icon tiles (§2.3)" — and S.2 rules the product switcher out, since TalentSphere is
a single product. The gradient's only definition in the guide therefore lives in a component this
product does not build.

Defined here as a `gradient.accent` token: a two-stop diagonal from Accent `#00AAE7` to Accent Hover
`#2368A0`. Both stops are existing §1.1 tokens, so no new color enters the system and §1.6 is
satisfied — an extension of the existing set, which is exactly what it asks for.

## Risks / Trade-offs

- **`TS-BL-007` is the widest item in the feature and everything else waits on it** → Its tasks are
  ordered so the token layer and its build gate land first and the primitives follow, so a slip in
  the primitives does not block `TS-BL-008`'s start against finished tokens. Residual risk accepted:
  if `TS-BL-007` slips entirely, all five siblings slip. That is inherent to D.9's graph, not
  introduced here.
- **Five components are built before any real screen consumes them**, risking abstractions that fit
  no caller — the same shape of risk `platform-core` accepted for its async substrate → Mitigated by
  naming a specific first consumer for each (Admin Cockpit for the shell, permission matrix for the
  table, Priority Selection for the panel, the toast for the Cockpit's own notifications) and
  designing against that screen's stated needs. Not mitigated by building the screen — that belongs
  to another feature.
- **The severity member list is this feature's reading, not `platform-core`'s enumeration** →
  Neutral fallback plus a test (D5), so an incorrect reading degrades to a plain notification rather
  than a crash or an unstyled surface.
- **Corrected Success triplet values are derived, not supplied by the guide's author** → Recorded as
  derived in S.6 with the construction rule and computed contrast ratios stated, so a designer can
  adjust them without having to reverse-engineer the intent. The alternative was shipping two
  severities that render identically.
- **Three new npm dependencies under a blocked registry** → Real, but orthogonal: the blocker
  already prevents `npm ci` with zero dependencies. Mitigated by pinning, by CI installing from the
  canonical lockfile, and by D7's indirection making the icon set replaceable at one file.
- **Vendored fonts are a licensing and repository-size commitment** → Both faces are open-source and
  freely redistributable as web fonts, which §2.1 states; subset to the weights §2.3 actually uses
  rather than shipping the full families.
- **The sidebar sits outside the theme system by design**, so a theme bug there is invisible to the
  theme-switching tests → Its tokens are declared single-valued explicitly (D4) rather than by
  omission, so the validator distinguishes "intentionally one value" from "someone forgot the dark
  half" — which is the failure this risk actually describes.
- **The presentation shell ships with no consumer** and may be built to fit a screen nobody has
  designed → Kept minimal: chrome removed, content full-viewport, nothing more. S.3 leaves its
  assignment open and this feature does not close it.
- **A design system built before its first real screen tends to grow speculative components** →
  Scope is fixed to the six items and their named requirements; anything a later feature needs that
  is not here is that feature's to propose, or an `/opsx:update` against this one.

## Migration Plan

No data migration; this is a frontend layer with no persistence beyond two user preferences (theme
and sidebar state) stored client-side. Sequence:

1. **`TS-BL-007` first, tokens before primitives.** The token source, the generator, the theme
   runtime, and the build gate land and pass before the first primitive consumes a token — otherwise
   the primitives are written against values the gate has not yet validated.
2. **`TS-BL-008` next.** It unblocks `access-control-and-admin`'s `TS-BL-023`, the only item in
   Phase 1 recorded as blocked on this feature.
3. **`TS-BL-009`, `010`, `011`, `012` are mutually independent** and can proceed in any order or
   concurrently once `TS-BL-007` is done. This is the feature's parallelism, and it is why five of
   its six items can share a wave — but that scheduling decision belongs to the delivery
   conversation (D.11), not here.
4. **`TS-BL-012` needs only `platform-core`'s payload contract**, which is a finished spec
   requirement. It does not wait on `TS-BL-005`'s delivery mechanism. If the contract is not yet
   expressed in code when `TS-BL-012` starts, it is mirrored as a TypeScript type against the spec
   and replaced with the shared type when one exists.
5. **`App.tsx`'s Sprint 0 placeholder is replaced, not deleted around.** Its `/build` call and its
   `ApiError` / correlation-ID rendering demonstrate `platform-core`'s global error-handler contract
   and become the shell's error presentation rather than being discarded.
6. **Rollback** is redeploy-previous-artifact. The frontend is a static bundle; there is no schema
   and no server state to reverse.

## Open Questions

- **Which screen, if any, uses the presentation/fullscreen shell.** S.3 lists the Interview Console
  during an active interview and a full-screen document viewer as candidates and decides neither.
  The state is built and selectable either way, so this is answerable by `interview-pipeline` without
  changing anything here.
- **When the brand assets arrive** (D9). The slots and their rules are built; the files are a copy.
- **Whether the sidebar should auto-collapse below the Large breakpoint.** Layout §7 explicitly
  frames this as an extension to design deliberately rather than an assumed default, and `UI-001`
  says desktop-first. Not built. Revisit only if a real need appears.
- **Exact subsetting of the vendored font families** — which weights and character ranges ship.
  §2.3's weight scale bounds it; the precise subset is a build-configuration detail that changes no
  requirement.

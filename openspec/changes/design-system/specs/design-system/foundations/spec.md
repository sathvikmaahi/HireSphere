## Purpose

The visual design language every TalentSphere screen is built from — color, typography,
iconography, shape, elevation and spacing — expressed as an enforceable token set, together with
the primitives that are a direct expression of those tokens and carry no independent state. Owned
by `TS-BL-007`.

## ADDED Requirements

### Requirement: Colors defined as light and dark pairs

Every color token SHALL be defined as an explicit light-mode and dark-mode pair. A token SHALL NOT
be considered complete with only one value, and the second value SHALL NOT be derived
automatically. A token family that is theme-independent by specification SHALL declare itself as
such, and SHALL be the only exception.

*Source: `reference/design-spec.md` §1 and §1.6. The declared-exception clause exists because
`layout-spec.md` §2 requires the sidebar to sit outside the theme system — see `design.md` D4;
without it, the completeness check would fail the build on the one family required to have a single
value, and the distinction between "intentionally one value" and "someone forgot the dark half"
would be lost.*

#### Scenario: Token added with one value

- **WHEN** a color token is defined with only a light or only a dark value, and its family is not
  declared theme-independent
- **THEN** the design-token validation fails the build

#### Scenario: Declared theme-independent family

- **WHEN** a token belongs to a family declared theme-independent
- **THEN** a single value is accepted
- **AND** the declaration is recorded in the token source rather than as an exception in the checker

#### Scenario: Theme switch

- **WHEN** the user switches theme
- **THEN** every surface, text and border color in the main content area resolves from the
  corresponding pair, with no unthemed element remaining

### Requirement: Tokens are the only source of visual values

No component SHALL define a raw color literal or a spacing value outside the defined scales. A build
step SHALL fail on a color literal appearing in component source outside the token definitions, and
on a spacing value outside the defined spacing steps.

*Source: `talentsphere-wave-1-foundation` D13, inherited per `design.md` D14, and `AGENTS.md`'s
standing product-quality bar. A gate rather than a convention because a one-off hex is a
two-character change that looks correct in every screenshot.*

#### Scenario: Raw color literal in a component

- **WHEN** component source contains a hex, `rgb()` or `hsl()` color literal outside the token
  definitions
- **THEN** the build fails

#### Scenario: Off-scale spacing value

- **WHEN** component source uses a spacing value that is not one of the defined steps
- **THEN** the build fails

### Requirement: Fixed brand accent

The brand accent SHALL be the one color constant across both themes and SHALL be used for links,
primary actions, active and selected states, focus indication and highlighted text.

*Source: `reference/design-spec.md` §1.1.*

#### Scenario: Accent across themes

- **WHEN** the theme changes between light and dark
- **THEN** the accent value is unchanged while surrounding surface and text colors invert in tone

### Requirement: No one-off colors

New colors SHALL NOT be introduced for a single use case. A new need SHALL be met by extending the
semantic set, the status-surface pattern, or a named token family — never by a value written at a
call site.

*Source: `reference/design-spec.md` §1.6.*

#### Scenario: New status need

- **WHEN** a screen requires a color treatment not covered by existing tokens
- **THEN** a defined token is added rather than a local value being hardcoded

#### Scenario: Gradient fill

- **WHEN** a surface requires the accent gradient
- **THEN** it resolves from a named gradient token whose stops are existing accent tokens, rather
  than from new color values

### Requirement: Status surfaces as triplets

Status communication SHALL use a tint background, saturated border and readable text triplet per
status, in both themes, rather than a flat solid fill of the semantic color. Each defined status
SHALL be visually distinguishable from every other defined status.

*Source: `reference/design-spec.md` §1.5, as corrected by `exploration-notes.md` S.6 — the guide's
Success triplet is character-for-character its Info triplet, which would make two statuses render
identically and orphan §1.4's Success color. See `design.md` D5.*

#### Scenario: Inline alert

- **WHEN** a warning is displayed inline
- **THEN** it renders with the warning triplet's background, border and text values for the active
  theme

#### Scenario: Success distinguishable from information

- **WHEN** a success surface and an informational surface are rendered in the same theme
- **THEN** their background, border and text values differ

### Requirement: Type scale and weight usage

The system SHALL define a restrained type scale biased toward compact, information-dense interface
text, with the page-level heading sizes used at most once per screen, and a weight scale in which
the heaviest weight is reserved for the brand lockup only.

*Source: `reference/design-spec.md` §2.2 and §2.3, reinforced by `exploration-notes.md` S.5 —
compact type only reads well when the content is also short.*

#### Scenario: Interface chrome sizing

- **WHEN** interface chrome such as buttons, labels and table cells is rendered
- **THEN** it uses the small or extra-small steps of the scale

#### Scenario: Page title

- **WHEN** a standard content page renders its header
- **THEN** exactly one page-level heading appears at the page-title size and weight

#### Scenario: Brand weight misuse

- **WHEN** the heaviest weight is applied to anything other than the brand lockup
- **THEN** the usage is rejected as non-conforming

### Requirement: Typefaces are self-hosted

The defined interface and monospace typefaces SHALL be served from the application's own assets. The
system SHALL NOT depend on an external font host at page load.

*Source: `reference/design-spec.md` §2.1 names both faces. Self-hosting per `design.md` D8: the
corporate network filters hostnames, and a silently-missing web font degrades to a fallback that
looks almost right — the hardest kind of visual regression to notice.*

#### Scenario: Remote font reference

- **WHEN** a stylesheet references a remote font host
- **THEN** the build fails

#### Scenario: Font assets missing

- **WHEN** the vendored font files are absent from the build output
- **THEN** the build fails rather than falling back silently

### Requirement: Long-form content typography

Rendered long-form content SHALL use a distinct, more relaxed typographic treatment than interface
chrome, with a larger line height, stepped headings, body copy in the secondary text color, styled
inline code, bordered code blocks, and links underlined at rest.

*Source: `reference/design-spec.md` §2.4.*

#### Scenario: Long-form block rendered

- **WHEN** long-form content is displayed
- **THEN** it uses the prose treatment rather than the interface type settings

#### Scenario: Inline code

- **WHEN** inline code appears in long-form content
- **THEN** it renders in the monospace face on a tinted background with a thin border and small
  radius, never unstyled

### Requirement: Iconography rules

Icons SHALL be outline or stroke-based with consistent stroke weight and rounded joins, SHALL
inherit the surrounding text color rather than carrying a fixed color, and SHALL follow the defined
size steps. Icon sizes SHALL NOT be mixed within a UI region, and outline icons SHALL NOT be mixed
with filled icons in the same context.

*Source: `reference/design-spec.md` §3.*

#### Scenario: Icon in a hover state

- **WHEN** a control containing an icon is hovered, activated or disabled
- **THEN** the icon color follows the text color of that state without a special case

#### Scenario: Third-party brand mark

- **WHEN** an external brand mark is displayed as an identifier
- **THEN** it may render as a fixed full-color glyph
- **AND** it is not used for an ordinary functional action

### Requirement: Shape and elevation

The system SHALL define corner-radius steps and elevation levels, including a resting card
elevation, a raised elevation for floating content, and a focus or selection ring. Dark-mode shadows
SHALL use higher opacity than their light-mode counterparts.

*Source: `reference/design-spec.md` §4.1 and §4.2.*

#### Scenario: Card on background

- **WHEN** a card renders on the page background
- **THEN** it uses the standard radius and the resting elevation for the active theme

#### Scenario: Dark-mode shadow

- **WHEN** an elevated surface renders in dark mode
- **THEN** its shadow opacity is meaningfully higher than the light-mode value so the elevation
  remains perceptible

### Requirement: Component shape patterns

Buttons SHALL be provided as the defined variant set built from color tokens rather than bespoke
colors, with the defined size steps. Badges SHALL be pill-shaped with extra-small semibold text and
a thin variant-colored border. The disabled state SHALL be a uniform opacity reduction with pointer
interactivity removed, never a bespoke disabled palette.

*Source: `reference/design-spec.md` §4.3.*

#### Scenario: New button need

- **WHEN** a screen needs a button treatment
- **THEN** it uses one of the defined variants rather than introducing a new fill color

#### Scenario: Disabled control

- **WHEN** any interactive element is disabled
- **THEN** it renders at the uniform reduced opacity and does not respond to pointer input

### Requirement: Always-visible focus indication

Every interactive element SHALL present a clearly visible focus state. A focus indicator SHALL NOT
be removed without an equally visible replacement.

*Source: `reference/design-spec.md` §4.2 and §4.3; satisfies `UI-009` per `exploration-notes.md`
S.4.*

#### Scenario: Keyboard traversal

- **WHEN** a user moves through a screen using the keyboard
- **THEN** the focused element is visibly indicated at every step

### Requirement: Accessibility baseline

Core UI workflows SHALL align with WCAG 2.1 AA for keyboard navigation, contrast, labels, focus
states and screen-reader semantics. Contrast SHALL be verified against the token set rather than
assumed.

*Source: `UI-009`.*

#### Scenario: Contrast check

- **WHEN** token pairs are validated
- **THEN** every defined text-on-surface pairing meets the AA threshold in both themes, and a
  pairing below it fails the build

#### Scenario: Keyboard-only operation

- **WHEN** a core workflow is attempted without a pointing device
- **THEN** it can be completed using the keyboard alone

### Requirement: Theme behavior

Theme changes SHALL animate rather than snap, and the user's theme choice SHALL persist across
sessions.

*Source: `reference/design-spec.md` §1.6.*

#### Scenario: Returning user

- **WHEN** a user who previously selected dark mode returns in a later session
- **THEN** the interface renders in dark mode without re-selection

#### Scenario: Theme transition

- **WHEN** the theme is switched
- **THEN** color changes animate rather than snapping instantly

### Requirement: Brand asset usage

The logo variant SHALL be swapped to contrast with the active background, constrained by a fixed
height rather than width, and given clear space at least equal to the mark's height. Where the
asset files are not present, a brand slot SHALL render a text wordmark rather than substitute
artwork or render empty.

*Source: `reference/design-spec.md` §5. The fallback clause exists because `reference/assets/` is
not present in this repository — see `design.md` D9. A substituted logo would survive to production
because it looks finished; a text wordmark does not.*

#### Scenario: Logo on a dark surface

- **WHEN** the logo renders against a dark background
- **THEN** the light variant is used

#### Scenario: Asset files absent

- **WHEN** a brand slot renders and the asset files are not available
- **THEN** a text wordmark is displayed in the specified weight
- **AND** no placeholder or substitute image is shown

### Requirement: Spacing rhythm

Layout SHALL use the defined spacing steps, with new arbitrary values avoided unless a specific
alignment problem requires one.

*Source: `reference/layout-spec.md` §8.*

#### Scenario: Section stacking

- **WHEN** top-level page sections are stacked
- **THEN** the gap between them uses the defined section-stacking step consistently rather than
  per-section margins

### Requirement: AI disclosure components

The system SHALL provide a reusable label marking content as AI-generated or AI-assisted until
approved, and a reusable human-review disclaimer surface for AI-derived recommendations that avoids
claiming bias-free output. Both SHALL be built from the status-surface and badge patterns.

*Source: `UI-004`/`AI-001` and `RANK-004`/`UI-006`; `exploration-notes.md` S.4 identifies the badge
and status-surface patterns as their implementation vehicle.*

#### Scenario: Unapproved AI content displayed

- **WHEN** a screen renders AI-produced content that has not been approved by a human
- **THEN** the AI-generated label is displayed with it

#### Scenario: Disclaimer wording

- **WHEN** the human-review disclaimer renders
- **THEN** it states that output is an AI-assisted recommendation subject to human review
- **AND** it makes no claim that the output is free of bias

#### Scenario: Available before its consumers ship

- **WHEN** this feature is deployed
- **THEN** both components exist and are usable
- **AND** the screens that consume them arrive in later features

### Requirement: Evidence-source labeling

The system SHALL provide a reusable label that distinguishes resume-sourced, interview-sourced,
scorecard-sourced and human-decision evidence. The distinction SHALL be carried by icon and text
rather than by a new color family. The label SHALL render the source value it is given without
interpreting or deriving it.

*Source: `UI-005`/`G-02`; `exploration-notes.md` S.3 records that the guide's only categorical color
system is semantic status, which means state rather than source, and directs that this be solved
with icon plus label consistent with §1.6. The non-derivation clause mirrors `design.md` D1 — the
source classification is `ai-platform-governance`'s `TS-BL-031` substrate; only its appearance is
owned here.*

#### Scenario: Mixed-evidence insight

- **WHEN** a screen presents claims from more than one evidence source
- **THEN** each claim carries a label distinguishing its source

#### Scenario: No new color family

- **WHEN** the evidence-source labels render
- **THEN** they are distinguished by icon and text within the existing token set, without a new
  categorical color family

#### Scenario: Unrecognized source value

- **WHEN** the label receives a source value it does not recognize
- **THEN** it renders the value in a neutral treatment rather than failing or omitting the label

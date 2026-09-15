## Purpose

The visual design language every TalentSphere screen is built from — color, typography, iconography, shape, elevation, and spacing — expressed as tokens so later slices consume them rather than reinventing them.

## ADDED Requirements

### Requirement: Colors defined as light and dark pairs

Every color token SHALL be defined as an explicit light-mode and dark-mode pair. A token SHALL NOT be considered complete with only one value, and the second value SHALL NOT be derived automatically.

#### Scenario: Token added with one value

- **WHEN** a color token is defined with only a light or only a dark value
- **THEN** the design-token validation fails

#### Scenario: Theme switch

- **WHEN** the user switches theme
- **THEN** every surface, text, and border color resolves from the corresponding pair with no unthemed element remaining

### Requirement: Fixed brand accent

The brand accent SHALL be the one color constant across both themes and SHALL be used for links, primary actions, active and selected states, focus indication, and highlighted text.

#### Scenario: Accent across themes

- **WHEN** the theme changes between light and dark
- **THEN** the accent value is unchanged while surrounding surface and text colors invert in tone

### Requirement: No one-off colors

New colors SHALL NOT be introduced for a single use case. A new need SHALL be met by extending the semantic set or the status-surface pattern.

#### Scenario: New status need

- **WHEN** a screen requires a color treatment not covered by existing tokens
- **THEN** the semantic or status-surface set is extended as a defined token rather than a local value being hardcoded

### Requirement: Status surfaces as triplets

Status communication SHALL use a tint background, saturated border, and readable text triplet per status, in both themes, rather than a flat solid fill of the semantic color.

#### Scenario: Inline alert

- **WHEN** a warning is displayed inline
- **THEN** it renders with the warning triplet's background, border, and text values for the active theme

### Requirement: Type scale and weight usage

The system SHALL define a restrained type scale biased toward compact, information-dense interface text, with the page-level heading sizes used at most once per screen, and a weight scale in which the heaviest weight is reserved for the brand lockup only.

#### Scenario: Interface chrome sizing

- **WHEN** interface chrome such as buttons, labels, and table cells is rendered
- **THEN** it uses the small or extra-small steps of the scale

#### Scenario: Page title

- **WHEN** a standard content page renders its header
- **THEN** exactly one page-level heading appears at the page-title size and weight

#### Scenario: Brand weight misuse

- **WHEN** the heaviest weight is applied to anything other than the brand lockup
- **THEN** the usage is rejected in review as non-conforming

### Requirement: Long-form content typography

Rendered long-form content SHALL use a distinct, more relaxed typographic treatment than interface chrome, with a larger line height, stepped headings, body copy in the secondary text color, styled inline code, bordered code blocks, and links underlined at rest.

#### Scenario: Long-form block rendered

- **WHEN** long-form content is displayed
- **THEN** it uses the prose treatment rather than the interface type settings

#### Scenario: Inline code

- **WHEN** inline code appears in long-form content
- **THEN** it renders in the monospace face on a tinted background with a thin border and small radius, never unstyled

### Requirement: Iconography rules

Icons SHALL be outline or stroke-based with consistent stroke weight and rounded joins, SHALL inherit the surrounding text color rather than carrying a fixed color, and SHALL follow the defined size steps. Icon sizes SHALL NOT be mixed within a UI region, and outline icons SHALL NOT be mixed with filled icons in the same context.

#### Scenario: Icon in a hover state

- **WHEN** a control containing an icon is hovered, activated, or disabled
- **THEN** the icon color follows the text color of that state without a special case

#### Scenario: Third-party brand mark

- **WHEN** an external brand mark is displayed as an identifier
- **THEN** it may render as a fixed full-color glyph
- **AND** it is not used for an ordinary functional action

### Requirement: Shape and elevation

The system SHALL define corner-radius steps and elevation levels, including a resting card elevation, a raised elevation for floating content, and a focus or selection ring. Dark-mode shadows SHALL use higher opacity than their light-mode counterparts.

#### Scenario: Card on background

- **WHEN** a card renders on the page background
- **THEN** it uses the standard radius and the resting elevation for the active theme

#### Scenario: Dark-mode shadow

- **WHEN** an elevated surface renders in dark mode
- **THEN** its shadow opacity is meaningfully higher than the light-mode value so the elevation remains perceptible

### Requirement: Component shape patterns

Buttons SHALL be provided as the defined variant set built from color tokens rather than bespoke colors, with the defined size steps. Badges SHALL be pill-shaped with extra-small semibold text and a thin variant-colored border. The disabled state SHALL be a uniform opacity reduction with pointer interactivity removed, never a bespoke disabled palette.

#### Scenario: New button need

- **WHEN** a screen needs a button treatment
- **THEN** it uses one of the defined variants rather than introducing a new fill color

#### Scenario: Disabled control

- **WHEN** any interactive element is disabled
- **THEN** it renders at the uniform reduced opacity and does not respond to pointer input

### Requirement: Always-visible focus indication

Every interactive element SHALL present a clearly visible focus state. A focus indicator SHALL NOT be removed without an equally visible replacement.

#### Scenario: Keyboard traversal

- **WHEN** a user moves through a screen using the keyboard
- **THEN** the focused element is visibly indicated at every step

### Requirement: Accessibility baseline

Core UI workflows SHALL align with WCAG 2.1 AA for keyboard navigation, contrast, labels, focus states, and screen-reader semantics.

#### Scenario: Contrast check

- **WHEN** token pairs are validated
- **THEN** text and interface contrast ratios meet the AA threshold in both themes

#### Scenario: Keyboard-only operation

- **WHEN** a core workflow is attempted without a pointing device
- **THEN** it can be completed using the keyboard alone

### Requirement: Theme behavior

Theme changes SHALL animate rather than snap, and the user's theme choice SHALL persist across sessions.

#### Scenario: Returning user

- **WHEN** a user who previously selected dark mode signs in again
- **THEN** the interface renders in dark mode without re-selection

### Requirement: Brand asset usage

The logo variant SHALL be swapped to contrast with the active background, constrained by a fixed height rather than width, and given clear space at least equal to the mark's height.

#### Scenario: Logo on a dark surface

- **WHEN** the logo renders against a dark background
- **THEN** the light variant is used

### Requirement: Spacing rhythm

Layout SHALL use the defined spacing steps, with new arbitrary values avoided unless a specific alignment problem requires one.

#### Scenario: Section stacking

- **WHEN** top-level page sections are stacked
- **THEN** the gap between them uses the defined section-stacking step consistently rather than per-section margins

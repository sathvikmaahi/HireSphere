# Visual Style Guide

This document is a standalone specification of the visual design language — color, typography, iconography, shape, and elevation — for applications that should share a consistent look and feel with this design system.

It describes design decisions only: names, values, and intended usage. It does not prescribe any programming language, styling framework, or file format — a team implementing this guide is free to express it as CSS, a native design-token system, a mobile styling layer, or anything else that fits their stack.

Reference assets for this guide live in [`./assets`](./assets).

---

## 1. Color Palette

Every color in this system is defined as a **pair** — one value for light mode, one for dark mode — except where noted. Both values must always be supplied together; a color is not considered complete with only one.

### 1.1 Brand accent

| Name | Light value | Dark value | Usage |
|---|---|---|---|
| Accent | `#00AAE7` | `#00AAE7` | Primary brand color. Links, primary buttons, active/selected states, focus indication, highlighted text. |
| Accent Hover | `#2368A0` | `#2368A0` | Hover/pressed state for accent-colored elements. |
| Accent Tint | rgba(0, 170, 231, 0.08) | rgba(0, 170, 231, 0.12) | Soft tinted background behind accent-adjacent content — e.g. a highlighted chip, an initials avatar, a selected row. |

The accent color is the one constant across both themes — it never shifts between light and dark.

### 1.2 Surface & Background

| Name | Light value | Dark value | Usage |
|---|---|---|---|
| Background | `#F0F2F5` | `#09090B` | The page/app background, behind all other content. |
| Surface | `#FFFFFF` | `#18181B` | Cards, panels, and primary content containers. |
| Elevated Surface | `#E8EAEE` | `#27272A` | Content that sits visually above `Surface` — popovers, dropdown menus, code blocks. |
| Border | `#DDE0E6` | `#3F3F46` | All hairline borders and dividers. |

### 1.3 Text

| Name | Light value | Dark value | Usage |
|---|---|---|---|
| Primary Text | `#232527` | `#FAFAFA` | Headings, body copy, primary content. |
| Secondary Text | `#8C8C8C` | `#A1A1AA` | Supporting copy, descriptions, secondary labels. |
| Muted Text | `#B7B2B3` | `#71717A` | Placeholder text, disabled labels, the lowest-emphasis text in the system. |

### 1.4 Semantic / Status Colors

| Name | Value (both themes) | Usage |
|---|---|---|
| Success | `#22C55E` | Success states, confirmations. |
| Warning | `#F59E0B` | Warnings, caution states. |
| Info | `#3B82F6` | Informational states. |
| Destructive / Error | `#EF4048` | Errors, destructive actions, delete confirmations. |

### 1.5 Status Surface Colors

Status communication (banners, inline alerts, toast-style notifications) should use a **tint background + saturated border + readable text** triplet rather than a flat, solid fill of the semantic color. Each status has its own light/dark triplet:

| Status | Light — background / border / text | Dark — background / border / text |
|---|---|---|
| Neutral | `#FFFFFF` / `#DDE0E6` / `#232527` | `#18181B` / `#3F3F46` / `#FAFAFA` |
| Success | `#E6F7FD` / `#80D3F0` / `#006A90` | `#091E2E` / `#1A4D6E` / `#00AAE7` |
| Info | `#E6F7FD` / `#80D3F0` / `#006A90` | `#091E2E` / `#1A4D6E` / `#00AAE7` |
| Warning | `#FEF9E8` / `#F5D87A` / `#92560A` | `#1E1500` / `#5C3C00` / `#F59E0B` |
| Error | `#FFF0F0` / `#F5B8B8` / `#C81E26` | `#200A0A` / `#5C1A1A` / `#EF4048` |

### 1.6 Usage Principles

- Every color must be supplied as an explicit light/dark pair — never assume one theme and derive the other.
- The accent color is the single fixed point across themes; all other colors invert in tone between light and dark.
- When a user switches themes, color changes should animate smoothly (roughly a quarter of a second) rather than snap instantly.
- Never introduce a new one-off color for a single use case — extend the semantic set (§1.4) or the status-surface pattern (§1.5) instead.

---

## 2. Typography

### 2.1 Typefaces

| Role | Typeface | Fallback | Weights available |
|---|---|---|---|
| Primary (UI + body) | **Plus Jakarta Sans** | the platform's default sans-serif | Regular (400), Medium (500), Semibold (600), Bold (700), Extrabold (800) |
| Monospace (code) | **JetBrains Mono** | the platform's default monospace | Regular (400) |

Both are open-source typefaces freely available as web fonts. Text should render with smoothed/antialiased edges wherever the platform supports it, for crisper small-size legibility.

### 2.2 Type scale

A small, restrained set of sizes, biased heavily toward compact, information-dense text rather than spacious marketing-style type:

| Name | Size | Typical usage |
|---|---|---|
| Extra Small | 12px | Badges, meta text, avatar initials, dense table cells — the smallest and single most frequent size in interface chrome. |
| Small | 14px | Default body/UI text — buttons, form labels, secondary navigation labels — the most common size overall. |
| Base | 16px | Standard paragraph text, wordmark text. |
| Large | 18px | Minor emphasis / sub-headings. |
| Extra Large | 20px | Section headings. |
| 2X Large | 24px | Page-level headings. |
| 3X Large | 30px | Rare, top-of-page hero-style headings. |

Guidance: default to the Small/Extra Small sizes for interface chrome. Reserve the 2X Large/3X Large sizes for page titles only — they should appear at most once per screen.

### 2.3 Font weight scale

| Weight | Usage |
|---|---|
| Regular (400) | Rare; reserved for long-form prose body copy, where a lighter weight improves extended readability. |
| Medium (500) | The default weight for interactive UI text — buttons, nav items, form controls. The most common weight overall. |
| Semibold (600) | Headings, emphasized labels, active/selected states. |
| Bold (700) | Strong emphasis, section titles. |
| Extrabold (800) | Reserved for the wordmark/brand lockup only — do not use it elsewhere. |

### 2.4 Long-form content (prose) typography

Rendered long-form content — documentation, articles, editor output — should use a distinct, more relaxed typographic treatment than interface chrome:

- Base size 15px, line height 1.7 — noticeably more open than the UI's default density, because sustained reading needs more breathing room than interface chrome.
- Headings step down through 24px / 20px / 17.5px for the first three heading levels, all at Semibold weight; the second heading level gets a thin dividing rule beneath it to mark section breaks.
- Body paragraphs, lists, and table cells use Secondary Text, not Primary Text — this keeps headings prominent while body copy recedes slightly.
- Inline code uses the monospace typeface at roughly 87% of the surrounding text size, sitting on a subtly tinted background with a thin border and a small rounded corner — inline code should never appear bare/unstyled.
- Code blocks get a fully bordered container with generously rounded corners and generous internal padding; syntax highlighting uses its own dedicated light/dark color pairing, distinct from the semantic status colors in §1.4.
- Blockquotes get a colored left border (using the accent color), italicized text, and Secondary Text coloring.
- Links within long-form content are always accent-colored and underlined — unlike interface chrome, where a hover-only underline is acceptable, prose links should stay underlined at rest for clear scannability.

---

## 3. Iconography

- **Style**: outline/stroke-based line icons, not solid/filled shapes. Consistent stroke weight throughout the set, with friendly rounded joins and endpoints rather than sharp corners.
- **Color**: icons should inherit the surrounding text color rather than carry a fixed, independent color — this way they automatically follow theme changes and state changes (hover, active, disabled) without special-casing.
- **Sizing**: default to 14–16px in compact interface chrome (inline with small labels, inside standard buttons); scale up to 20–24px only for emphasis (e.g. a lone icon acting as a primary control).
- **Consistency**: never mix icon sizes within the same UI region, and never mix an outline icon with a solid/filled icon in the same context.
- **Exception**: a small, separate category of fixed, full-color glyphs is permitted specifically to represent external or third-party brand marks (e.g. a partner or platform logo used as an identifier) — these are allowed to break the outline/monochrome rule because they represent a fixed external identity rather than a themable interface icon. They should never be used for ordinary functional UI actions.

---

## 4. Shape & Elevation

### 4.1 Corner radius

| Name | Value | Usage |
|---|---|---|
| Small | 6px | Small controls — tight chips, inner elements of a badge. |
| Default | 8px | The standard radius — buttons, inputs, cards. |
| Large | 12px | Larger surfaces — modals, code blocks, elevated panels. |
| Extra Large | 16px | Rare, largest containers only. |

### 4.2 Elevation (shadow)

| Name | Light mode | Dark mode | Usage |
|---|---|---|---|
| Resting (Card) | a soft, two-layer shadow: a 3px-blur layer at 8% black plus a 2px-blur layer at 5% black, both with almost no vertical offset | the same two-layer shape at 40% and 30% black | The default resting elevation for cards sitting on the background. |
| Raised (Popover/Modal) | a single soft shadow, ~16px blur, 4px vertical offset, at 12% black | the same shadow at 50% black | Content floating above the page — popovers, dropdowns, modals. |
| Focus / Selection Ring | a 2px ring in the accent color at 35% opacity, with no gap from the element's edge | the same ring at 40% opacity | An alternative or supplement to a plain focus outline, used to indicate keyboard focus or an active selection. |

Dark-mode shadows should always use meaningfully higher opacity than their light-mode counterparts — a shadow that reads clearly against a light background all but disappears against a dark one unless it's intensified.

### 4.3 Component shape patterns

**Buttons** — six visual variants, all built from the color tokens in §1 rather than bespoke colors:
- *Primary*: filled with the Accent color, white text.
- *Destructive*: filled with the Destructive color, white text.
- *Outline*: transparent fill, a visible border, primary-colored text; fills with a subtle tint on hover.
- *Secondary*: filled with the Elevated Surface color, primary-colored text.
- *Ghost*: no fill and no border at rest; fills with a subtle tint on hover.
- *Link*: no fill or border, accent-colored text, underlines on hover.

Size steps: Small (~36px tall), Default (~40px tall), Large (~44px tall), and an Icon-only square (~40px × 40px). Horizontal padding scales with size, from roughly 12px at Small up to 32px at Large.

**Badges/pills** — fully rounded (pill-shaped), Extra Small text at Semibold weight, tight padding (roughly 10px horizontal, 2px vertical), a thin border matching the variant's color.

**Disabled state** — applied uniformly to any interactive element: reduce opacity to roughly 50% and remove pointer interactivity. Never invent a bespoke "disabled" color palette.

**Focus indication** — every interactive element must show a clearly visible focus state (see the Focus/Selection Ring above). Never remove a focus indicator without providing an equally visible replacement.

---

## 5. Brand Assets

See [`./assets`](./assets) for the source files referenced below.

| Asset | File | Usage |
|---|---|---|
| Logo — dark variant | [`assets/logo-dark.png`](./assets/logo-dark.png) | Use on **light** backgrounds. |
| Logo — light variant | [`assets/logo-light.png`](./assets/logo-light.png) | Use on **dark** backgrounds. |
| Brand mark (icon-only) | [`assets/brand-mark.svg`](./assets/brand-mark.svg) | Compact, icon-only usage where a full wordmark won't fit — e.g. a favicon or app icon. |

Guidance:
- Always swap the logo variant to match the active theme (dark logo on light background, light logo on dark background) — never place a same-contrast logo against a matching-tone background.
- In a horizontal lockup, constrain the logo to a fixed height (not width) so it scales proportionally, optionally paired with a thin vertical divider before any adjacent wordmark or product name text.
- Maintain clear space around the logo at least equal to the height of the mark itself; do not crowd it with adjacent elements.

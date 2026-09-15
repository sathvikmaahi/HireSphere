## Purpose

The presentation of a notification to a user — the transient toast and the in-app notification
surface — rendering `platform-core`'s delivery payload contract and mapping its severity onto the
design system's status surfaces. Owned by `TS-BL-012`.

## ADDED Requirements

### Requirement: Renders the platform notification payload contract

The notification components SHALL render a payload carrying a severity, a title, a body, an
originating event reference and an optional action reference, without feature-specific knowledge of
what produced it. The payload shape SHALL be consumed as defined by `platform/notifications`, not
redefined here.

*Source: `platform-core`'s `platform/notifications` "Delivery payload contract" requirement, which
exists specifically so a rendering surface can present a notification without knowing which feature
produced it. `exploration-notes.md` D.9 records that `TS-BL-012` depends on `TS-BL-005` precisely
because a toast needs a real payload shape rather than a visual mock — that dependency is on the
contract, which is finished, not on the delivery mechanism, which is not built.*

#### Scenario: Rendering a notification

- **WHEN** a notification payload is presented
- **THEN** its severity, title, body and any action are rendered without feature-specific handling

#### Scenario: New notification type introduced

- **WHEN** a feature introduces a new notification type
- **THEN** the existing rendering surfaces present it without modification

#### Scenario: Notification without an action

- **WHEN** a payload carries no action reference
- **THEN** it renders without an action control rather than with a disabled or empty one

### Requirement: Severity maps onto status surfaces

Each severity value SHALL render using the corresponding status-surface triplet for the active
theme. Distinct severity values SHALL be visually distinguishable from one another.

*Source: `reference/design-spec.md` §1.5, as corrected by `exploration-notes.md` S.6 — the guide's
Success triplet duplicates its Info triplet, which would make two severities render identically and
reduce the enumeration to decoration at the one place a user meets it. `platform-core`'s `design.md`
D9 fixes the boundary: severity is semantics there, appearance is this capability's.*

#### Scenario: Warning notification

- **WHEN** a notification of warning severity is presented
- **THEN** it renders with the warning triplet's background, border and text for the active theme

#### Scenario: Success and information notifications

- **WHEN** a success notification and an informational notification are presented in the same theme
- **THEN** their background, border and text values differ

### Requirement: Unrecognized severity renders neutrally

A severity value the mapping does not recognize SHALL render using the neutral status surface. It
SHALL NOT fail, render unstyled, or suppress the notification.

*Source: `platform/notifications`' requirement that a new notification type presents without
modification to existing surfaces. `design.md` D5 records that the severity member list is not
enumerated in `platform-core`'s spec, so the neutral fallback is what makes this capability's
reading of it safe to be wrong about — and the failure mode is invisible until it reaches
production, so it is asserted rather than assumed.*

#### Scenario: Unknown severity value

- **WHEN** a payload carries a severity value the mapping does not recognize
- **THEN** the notification renders on the neutral status surface with its title and body intact

### Requirement: Toasts are dismissible, accessible and do not obstruct work

A toast SHALL be dismissible by the user, SHALL be announced to assistive technology, and SHALL NOT
obscure the floating action panel region or block interaction with page content.

*Source: `UI-009`; `reference/layout-spec.md` §6, which anchors the panel region to the same corner
a toast would naturally occupy. `DEP-008`/`G-12`'s graceful-degradation posture applied to the
surface: a notification must never be the reason work cannot continue.*

#### Scenario: User dismisses a toast

- **WHEN** the user dismisses a toast
- **THEN** it is removed and no further action is required

#### Scenario: Toast and panel both present

- **WHEN** a toast is shown while the floating action panel is rendered
- **THEN** neither obscures the other

#### Scenario: Assistive announcement

- **WHEN** a toast appears
- **THEN** it is announced to assistive technology at an urgency matching its severity

### Requirement: Multiple notifications are presented in a bounded stack

Concurrent toasts SHALL be presented as a bounded stack rather than accumulating without limit, and
notifications beyond the bound SHALL remain available on the in-app surface rather than being
discarded silently.

*Source: `exploration-notes.md` S.5's standing rule that the interface never feels overcompacted,
and its instruction to surface the single most important thing prominently with supporting detail
progressive rather than all displayed at once.*

#### Scenario: Burst of notifications

- **WHEN** more notifications arrive at once than the stack bound allows
- **THEN** the bound is respected on screen
- **AND** the remainder are still retrievable from the in-app notification surface

### Requirement: Notification presentation carries references, not resolved personal data

The notification components SHALL render the values supplied to them and SHALL NOT resolve a
record reference into personal data themselves.

*Source: `platform/notifications`' "Notifications carry references, not personal data" requirement,
which resolves references against the recipient's permissions at read time; and `design.md` D1. A
component that resolved references itself would place that permission-sensitive read on the client.*

#### Scenario: Notification about a candidate record

- **WHEN** a notification references a candidate record
- **THEN** the component renders what its caller supplied
- **AND** it performs no lookup of its own

### Requirement: The components own no subscription

The notification components SHALL render the notifications they are given. Subscribing to a
notification stream, marking a notification read, and retrieving history SHALL be the consuming
feature's responsibility.

*Source: `design.md` D1. This is what allows `TS-BL-012` to be built and tested against the payload
contract while `platform-core`'s delivery mechanism is still unbuilt.*

#### Scenario: Built ahead of delivery

- **WHEN** this feature is deployed and no delivery mechanism is producing notifications
- **THEN** the components exist, render a supplied payload correctly, and mount nowhere by
  themselves

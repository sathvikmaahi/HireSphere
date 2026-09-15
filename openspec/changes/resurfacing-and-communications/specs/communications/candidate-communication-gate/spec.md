## Purpose

Holds `platform-core`'s structural candidate-facing exception closed until Legal's disclosure
sign-off (`OD-005`) resolves, and gives that future resolution a named, tracked landing point rather
than leaving it with nowhere to attach. Owned by `TS-BL-078`.

## ADDED Requirements

### Requirement: The candidate-facing communication flag is declared, seeded disabled

A `candidate_communication` capability flag SHALL be declared in the audited runtime-configuration
registry, seeded `disabled`. Enabling it SHALL require a mandatory reason and SHALL be audited.

*Source: `platform-core` `design.md` D9 — "`resurfacing-and-communications`' `TS-BL-078` is where
the gate is deliberately opened, and it remains blocked on `OD-005`." This project's standing rule
that a disabled capability answers 404 and an unknown flag raises rather than resolving false.
`C-04`'s resolution — candidate-facing email "Deferred, gated on `OD-005` (Legal)." `design.md` D8.*

#### Scenario: Flag declared and disabled

- **WHEN** the runtime-configuration registry is inspected for `candidate_communication`
- **THEN** it is declared and its seeded value is `disabled`

#### Scenario: Enabling requires a reason

- **WHEN** an authorized actor attempts to enable `candidate_communication`
- **THEN** the change requires a mandatory reason and is recorded in the audit trail

### Requirement: No candidate-reachable channel exists while the gate is disabled

While `candidate_communication` is `disabled`, the system SHALL provide no code path that delivers a
notification, email, or any other message to a candidate.

*Source: `D04`'s scope line — "internal email, no candidate email" — kept exact, not loosened.
`platform-core` `design.md` D9 — "'We will not send to candidates' is not a control on its own; a
misconfigured template violates it silently. A sender-side check means the deferral cannot be
violated by configuration." This capability's non-goal: build no candidate email, even a stub.
`design.md` D8.*

#### Scenario: Candidate-reachable channel sought

- **WHEN** this feature's surface area is inspected for a channel, template, or endpoint that can
  deliver a message to a candidate
- **THEN** none exists

#### Scenario: Internal notifications unaffected

- **WHEN** `communications/internal-notifications` sends a notification while
  `candidate_communication` is `disabled`
- **THEN** that notification is delivered normally, because it is addressed to an internal
  recipient and does not depend on this flag

### Requirement: This capability builds no candidate template or disclosure text

This capability SHALL NOT define a candidate-facing message template, disclosure text, or send
mechanism. That work begins only after `OD-005` resolves, in a future change.

*Source: `D04`'s recorded consequence — "the candidate email template is a governance artifact, not
just copy" — meaning it is Legal's artifact to define, not this feature's to draft ahead of
sign-off. `design.md` D8's explicit non-goal.*

#### Scenario: Candidate template sought

- **WHEN** this feature's artifacts are inspected for a candidate-facing message template or
  disclosure text
- **THEN** none exists

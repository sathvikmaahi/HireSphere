## Purpose

Registers resurfacing- and priority-lane-specific notification triggers and templates against
`platform-core`'s existing internal notification engine, delivering `D04`'s early, internal-only
scope with no new delivery mechanism of its own. Owned by `TS-BL-077`.

## ADDED Requirements

### Requirement: A new resurfacing suggestion notifies the owning Practice Manager internally

When one or more `MatchSuggestion`s are created for a posting, the system SHALL notify the Practice
Manager who owns that posting through the in-app and internal-email channels.

*Source: `D04`'s resolution — "Internal notification email (Notification Service module) \|
Early — with slice 5's scheduling tasks." §16.1's Human Gate for resurfacing — "Practice Manager
review required" — which this notification exists to make actionable. `platform-core`'s `TS-BL-005`
task 5.4 payload contract and task 5.6/5.7 channels, consumed as-is. `design.md` D7.*

#### Scenario: Suggestions created for an owned posting

- **WHEN** `resurfacing/resurfacing-engine` creates one or more `MatchSuggestion`s for a posting
- **THEN** the posting's owning Practice Manager receives an in-app and internal-email notification

#### Scenario: Delivered through the existing engine

- **WHEN** this notification is sent
- **THEN** it is dispatched through `platform-core`'s existing notification engine, using its
  existing payload contract, channels and retry policy — this capability defines no channel,
  retry loop, or payload shape of its own

### Requirement: Priority-lane entry notifies the owning Practice Manager internally

When a candidate's `MatchSuggestion` becomes `priority_lane = true`, the system SHALL notify the
target posting's owning Practice Manager through the in-app and internal-email channels.

*Source: `D04`'s early scope, same as above. `glossary.md`'s priority-lane precedence — surfacing
this to the Practice Manager is what makes the precedence actionable rather than latent.
`design.md` D7.*

#### Scenario: Candidate enters the priority lane

- **WHEN** `resurfacing/priority-lane` sets `priority_lane = true` on a suggestion
- **THEN** the target posting's owning Practice Manager receives an in-app and internal-email
  notification naming the candidate and the prior selection

### Requirement: Every notification this capability sends resolves to an internal recipient only

No notification this capability triggers SHALL be deliverable to a candidate or any recipient not
resolving to an internal user record.

*Source: `platform-core` `design.md` D9's sender-side internal-recipient guard — "Any recipient not
resolving to an internal user record is refused by the sender... a sender-side check means the
deferral cannot be violated by configuration." `D04`'s scope line — internal email only, in early
scope. `design.md` D7.*

#### Scenario: Recipient resolves internally

- **WHEN** a notification from this capability is addressed to the owning Practice Manager
- **THEN** it is delivered because the recipient resolves to an internal user record

#### Scenario: Candidate-addressed notification sought

- **WHEN** this capability's triggers are inspected for one addressed to a candidate record
- **THEN** none exists — this capability relies entirely on `platform-core`'s existing guard rather
  than implementing its own recipient check

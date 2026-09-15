## Purpose

Leaves a disabled, well-shaped seam for the calendar integration `D06` deferred, so a later slice
can implement it without a call-site change, without building any live calendar behavior now. Owned
by `TS-BL-079`.

## ADDED Requirements

### Requirement: The calendar integration flag is declared, seeded disabled

A `calendar_integration` capability flag SHALL be declared in the audited runtime-configuration
registry, seeded `disabled`. No route behind it SHALL be reachable while it is disabled.

*Source: `D06`'s decision (full calendar API) superseded by `C-04`'s resolution — "Google Calendar
API + free/busy \| Deferred to a late slice." This project's standing rule that a disabled
capability answers 404. `design.md` D9.*

#### Scenario: Flag declared and disabled

- **WHEN** the runtime-configuration registry is inspected for `calendar_integration`
- **THEN** it is declared and its seeded value is `disabled`

#### Scenario: No reachable route while disabled

- **WHEN** any calendar-related route is requested while `calendar_integration` is `disabled`
- **THEN** it answers 404

### Requirement: A one-way-projection adapter interface exists for a later slice to implement

An adapter interface SHALL exist, shaped so that a future calendar-write implementation is a
one-way projection from TalentSphere to the calendar, with TalentSphere remaining the source of
truth. The interface SHALL NOT be invoked while `calendar_integration` is disabled.

*Source: `D06`'s recorded recommendation — "TalentSphere is the source of truth; the calendar is a
one-way projection... reschedules must happen in TalentSphere." `TS-BL-079`'s dependency on
`interview-pipeline`'s `TS-BL-057` (round scheduling) supplies the interface's input shape — a
scheduled round's date, time and panelist set. `design.md` D9.*

#### Scenario: Interface shaped for one-way projection

- **WHEN** the adapter interface is inspected
- **THEN** it accepts a scheduled interview round's date, time, and panelist set as input, and
  defines no inbound path for calendar changes to reach TalentSphere

#### Scenario: Interface not invoked while disabled

- **WHEN** an interview round is scheduled while `calendar_integration` is `disabled`
- **THEN** the adapter interface is not called

### Requirement: No live calendar behavior is built in this change

This capability SHALL NOT perform a Calendar API call, a free/busy lookup, or generate an `.ics`
attachment.

*Source: `C-04`'s resolution table — Calendar API deferred; ".ics attachment \| Not adopted"
explicitly, not merely deferred. The accepted interim gap: "panelists receive an assignment email
but no calendar entry; they add the meeting themselves" — met by
`communications/internal-notifications`'s panelist notification, not by this capability.
`design.md` D9.*

#### Scenario: Live calendar call sought

- **WHEN** this feature's surface area is inspected for a Calendar API call, a free/busy lookup, or
  `.ics` generation
- **THEN** none exists

#### Scenario: Panelist still notified without a calendar entry

- **WHEN** an interview round is scheduled while `calendar_integration` is `disabled`
- **THEN** the assigned panelist still receives an internal notification of the assignment, with no
  calendar entry created

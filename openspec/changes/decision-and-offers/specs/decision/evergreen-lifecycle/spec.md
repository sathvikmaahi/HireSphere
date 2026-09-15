## Purpose

Evergreen posting lifecycle rules — pause, reopen and manual close with a reason, the assertions that
keep fulfilment closure unreachable from a posting with no vacancy count, and the two hygiene and
reporting obligations an unbounded posting creates. Owned by `TS-BL-070`.

## ADDED Requirements

### Requirement: An evergreen posting is paused, reopened and closed manually, each with a reason

An evergreen posting SHALL support pause, reopen and manual close, each performed by an authorized
actor with a mandatory reason and an audit record.

*Source: `CLS-005` — "Evergreen postings shall support Pause, Reopen, and Manual Close with reason and
shall not use finite fulfillment closure logic" — and `BR-014`. `D14`'s consequence: "No auto-close for
evergreen; **manual pause/close only**. Feature 17 resolves to: finite = automatic per rules plus
manual; evergreen = manual only." §11.1's `Open → Paused`, `Paused → Open`, `Open → Closed` and
`Paused → Closed` transitions are already registered by `hiring-postings`; this capability supplies no
new transition, per `hiring-postings` `design.md` D5, which predicted that this item "needs to add
nothing at all, exactly as `D14` predicted".*

#### Scenario: Pause with a reason

- **WHEN** an authorized actor pauses an open evergreen posting with a reason
- **THEN** it is paused and the action is audited

#### Scenario: Close without a reason

- **WHEN** a manual close is attempted on an evergreen posting with no reason
- **THEN** it is rejected

#### Scenario: Reopen after pause

- **WHEN** a paused evergreen posting is reopened with a reason
- **THEN** it returns to open

#### Scenario: A new transition sought

- **WHEN** this capability is inspected for a state or transition it registers
- **THEN** none exists

### Requirement: Fulfilment closure is unreachable from an evergreen posting

Fulfilment closure, the closure checklist, the fulfilment condition and vacancy-slot fulfilment SHALL
be unavailable for a posting whose vacancy type is evergreen. Requesting any of them SHALL be rejected
rather than returning an empty or vacuously satisfied result.

*Source: `BR-014` — "Evergreen postings do not close by vacancy fulfillment count" — and `CLS-005`'s
"shall not use finite fulfillment closure logic". `D14`'s "no auto-close for evergreen". `JOB-004`
means an evergreen posting has no vacancy slots, so a fulfilment count over zero slots would be
vacuously true — the failure mode this requirement exists to prevent.*

#### Scenario: Closing an evergreen posting as filled

- **WHEN** fulfilment closure is requested for an evergreen posting
- **THEN** it is rejected, naming the vacancy type

#### Scenario: Checklist for an evergreen posting

- **WHEN** the closure checklist is requested for an evergreen posting
- **THEN** it is rejected rather than returned empty

#### Scenario: Vacuous satisfaction sought

- **WHEN** the fulfilment condition is evaluated for a posting with no vacancy slots
- **THEN** it does not evaluate as satisfied

### Requirement: An evergreen posting with no hires is nudged for review after a configurable interval

Where an evergreen posting has been open for longer than a configured interval with no candidate
recruited, a review prompt SHALL be raised to the posting's owner. The interval SHALL be configurable
with changes audited, and SHALL be settable to disabled. The prompt SHALL NOT change the posting's
state.

*Source: `D14`'s recommendation — "a **staleness nudge** (open 240 days, 0 hires → prompt a review) so
evergreen postings do not become zombies" — seeded at 240 days in `platform-core`'s audited
runtime-configuration registry, per §29's configurable-without-code-change pattern and the same
treatment `C-11` gave the eligibility windows. `design.md` D11 records that a nudge which paused or
closed a posting would be exactly the automatic transition `D14` and `hiring-postings` `design.md` D5
both rule out, and that this runs on the existing dispatch chain without adding a §25 job type.*

#### Scenario: Interval exceeded with no hires

- **WHEN** an evergreen posting has been open beyond the configured interval with no recruited candidate
- **THEN** a review prompt is raised to its owner

#### Scenario: Interval exceeded with a hire

- **WHEN** an evergreen posting beyond the interval has recruited a candidate
- **THEN** no prompt is raised

#### Scenario: Nudge changes state

- **WHEN** the prompt is raised
- **THEN** the posting's state is unchanged

#### Scenario: Interval disabled

- **WHEN** the interval is set to disabled
- **THEN** no prompt is raised for any posting

#### Scenario: Interval changed

- **WHEN** the interval is changed
- **THEN** the change is audited with actor and reason

### Requirement: Each hire records its own fill interval

When a candidate is recruited against a posting, a per-hire fill record SHALL be written carrying the
Application, the posting and the interval from the posting's opening to that hire. The record SHALL be
written for evergreen and finite postings alike.

*Source: `D14`'s reporting consequence — "time-to-fill is a per-posting metric that never resolves for
a posting that never closes. For evergreen, time-to-fill must be computed **per hire**, not per
posting." Written for both types because the measure is well-defined for both and a per-hire record is
not reconstructible after the fact. `design.md` D11 records that this capability makes the computation
possible and computes nothing — `insight-and-reporting`'s `TS-BL-071`–`TS-BL-073` are the consumers.*

#### Scenario: Hire on an evergreen posting

- **WHEN** a candidate is recruited against an evergreen posting
- **THEN** a fill record exists carrying that Application and its interval from posting open

#### Scenario: Third hire on one evergreen posting

- **WHEN** a third candidate is recruited against the same evergreen posting
- **THEN** three separate fill records exist

#### Scenario: Aggregation performed here

- **WHEN** this capability is inspected for a computed time-to-fill metric, average or dashboard
- **THEN** none exists

### Requirement: Vacancy type is not changeable, and this capability does not add an exception

This capability SHALL NOT provide any path that changes a posting's vacancy type or vacancy count after
the posting has opened.

*Source: `D14`'s open edge case — "can a posting change type mid-life (evergreen → finite, or the
reverse)? Probably should be blocked; needs an explicit rule" — closed by `hiring-postings`'
`hiring/job-posting` spec, which makes type and count immutable from the point a posting opens, and its
`design.md` D12, which records why: finite-to-evergreen orphans slots `JOB-004` says evergreen cannot
have, and evergreen-to-finite requires choosing a count retroactively against an unbounded selection
set. `design.md` D3 records the consequence this feature carries as a result.*

#### Scenario: Type change through this capability

- **WHEN** a vacancy type change is attempted on an open posting through any path this capability
  exposes
- **THEN** it is rejected

#### Scenario: Override path sought

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains no vacancy type or count override

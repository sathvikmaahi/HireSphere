## Purpose

What an interviewing actor may see of a candidate's AI ranking context, and the record every
disclosure leaves: source-labelled fitment and gaps, per candidate, with no score, no rank position
and no sight of anyone else. Owned by `TS-BL-054`.

## ADDED Requirements

### Requirement: Interviewing actors see fitment and gaps, and not the score

An actor without board access SHALL be able to view source-labelled fitment and gap summaries for a
candidate they are assigned to, together with the job description and resume. That actor SHALL NOT
receive the match score, the rank position, the cohort size, or any other candidate's data.

*Source: `C-10`'s resolution of 2026-08-14, which **amends `D05`, reversing it** — "Interviewers see
AI-suggested questions and source-labelled fitment and gaps (`INT-004`, `INT-005`), plus the JD and
resume. They do **not** see `latest_match_score`, rank position, or other candidates"; `INT-005`;
§8's surface allocation and §9.1's description of the role as reviewing "candidate fitment and gaps"
with no mention of score or rank. `design.md` D7 records that this is built against `C-10`, not
against `D05`'s original "full context upfront" decision.*

#### Scenario: Assigned interviewer views context

- **WHEN** an assigned interviewing actor requests a candidate's AI context
- **THEN** source-labelled fitment and gap summaries are returned, with the job description and
  resume

#### Scenario: Score withheld

- **WHEN** that response is inspected
- **THEN** it carries no match score, no rank position and no cohort size

#### Scenario: Direct request for the score

- **WHEN** that actor requests the score or the board directly
- **THEN** the request is denied server-side rather than merely hidden in the interface

#### Scenario: Other candidates

- **WHEN** that actor's response is inspected
- **THEN** it contains no reference to any other candidate on the posting

### Requirement: The projection is enforced server-side and is not a display filter

The withheld values SHALL be absent from the response the server produces, not removed at the
presentation layer. No client-side configuration or interface state SHALL be capable of revealing
them.

*Source: `AUTHZ-003`'s server-side enforcement on every page **and** endpoint, `AUTHZ-004` — a
direct API call fails even when the interface hides the control — `UI-002`, `SEC-005`;
`config.yaml`'s rule that redaction is applied at write or produce time, not as a display filter.
`design.md` D7 records why this capability is a projection rather than a screen: the screen it feeds
is `interview-pipeline`'s Interview Console, which is not yet proposed.*

#### Scenario: Response payload inspected

- **WHEN** the projected response is inspected on the wire
- **THEN** the withheld values are absent from the payload rather than present and hidden

#### Scenario: Client requests unfiltered data

- **WHEN** a client requests the unprojected payload
- **THEN** the server produces the projection regardless of what the client asked for

#### Scenario: Presentation-layer filtering sought

- **WHEN** the codebase is inspected for a display-time filter that removes score or rank
- **THEN** none is relied upon for this guarantee

### Requirement: The projection is driven by the permission verdict, not by a role name

Whether an actor receives the projected or the full response SHALL be determined by the central
permission evaluator's verdict on the ranking surface, not by a role identifier compared in code.
Granting or denying that action through the matrix SHALL change the response with no code change.

*Source: `AUTHZ-001` deny-by-default, `AUTHZ-005` direct denial overrides role grants, `ADM-005`'s
grant / deny / unset cells, and `INT-003`'s explicit escape hatch — interviewers see only assigned
interviews "unless broader permission is granted." Hard-coding the role would make `C-10`'s
surface allocation a code constant rather than the seeded matrix decision `Interaction B` says it
is. `design.md` D7.*

#### Scenario: Seeded matrix

- **WHEN** an actor holding the seeded Interviewer permissions requests AI context
- **THEN** they receive the projected response, because the board action is not granted to them

#### Scenario: Broader permission granted

- **WHEN** an administrator grants the board action to a specific user
- **THEN** that user receives the full response with no code change, and the grant is audited

#### Scenario: Direct denial

- **WHEN** a user holds the board action by role but a direct denial is recorded against them
- **THEN** they receive the projected response, because denial overrides the grant

#### Scenario: Role comparison in code

- **WHEN** this capability is inspected for a comparison against a role name to decide the
  projection
- **THEN** none exists

### Requirement: Assignment scope applies to the projection

An interviewing actor SHALL receive AI context only for candidates they are assigned to interview.
The scope SHALL be applied as a query predicate, and a request for an unassigned candidate SHALL be
denied rather than returned empty.

*Source: `INT-003`, `D01`'s assignment-scoped predicate as retained by `C-02`; `D02` as resolved by
`C-03`; `access-control-and-admin`'s requirement that scope is applied as a query filter rather than
by discarding fetched rows.*

#### Scenario: Assigned candidate

- **WHEN** an interviewing actor requests context for a candidate they are assigned to
- **THEN** the projected response is returned

#### Scenario: Unassigned candidate

- **WHEN** the same actor requests context for a candidate they are not assigned to
- **THEN** the request is denied

#### Scenario: Denial distinguishable from emptiness

- **WHEN** a denied request is compared with a permitted request that matched nothing
- **THEN** the two responses are distinguishable

### Requirement: Projected claims are per-candidate and carry no comparative statement

Fitment and gap claims in the projected response SHALL be statements about the candidate alone. A
claim comparing the candidate to other candidates, to a cohort, or to a ranked position SHALL be
rejected from the projection.

*Source: `D05`'s "consequence / design detail," which `C-10` did not reverse — "AI context shown to
an Interviewer should be **per-candidate**, not the comparative leaderboard. Rank position ('#1 of
12') leaks the existence and standing of other candidates and should stay hidden from the Interviewer
role"; §16.1's Fitment Summary and Gap Summary input columns, which name resume and interview
evidence and no cohort. `design.md` D8 records why suppressing the score is insufficient on its own:
a comparative sentence leaks the same standing the withheld number would have.*

#### Scenario: Comparative claim in output

- **WHEN** a fitment or gap claim states or implies a comparison with other candidates
- **THEN** it is rejected from the projection rather than rendered

#### Scenario: Absolute claim

- **WHEN** a claim describes the candidate's evidence against the posting's requirements alone
- **THEN** it is included

#### Scenario: Ranked-position phrasing

- **WHEN** projected content is inspected for ranked-position phrasing
- **THEN** none is present

### Requirement: Every disclosure of AI output is recorded through the existing disclosure record

Each occasion on which AI ranking output is disclosed to a human actor SHALL be recorded through the
platform's append-only disclosure record, naming the actor, the output disclosed, the time and the
context. This capability SHALL NOT create a second disclosure mechanism, and SHALL NOT amend the AI
run.

*Source: `ai-platform-governance`'s `ai-platform/ai-run-logging` disclosure-record requirement and
its `design.md` D11, which built the mechanism for exactly this class of question — "whether a human
judgement was formed before or after seeing AI output" — and states that "`matching-and-ranking`'s
`TS-BL-054` owns the visibility *rules*; this requirement is the substrate that makes them
reviewable." `D05`'s dated correction block records why the record cannot be a field on the run.
`design.md` D9.*

#### Scenario: Projected context disclosed

- **WHEN** projected AI context is returned to an interviewing actor
- **THEN** a disclosure record is written naming the actor, the output, the time and the pre-interview
  context
- **AND** the AI run record is unchanged

#### Scenario: Board disclosure

- **WHEN** ranking output is disclosed to a board user
- **THEN** a disclosure record is written for that actor and occasion

#### Scenario: Second mechanism sought

- **WHEN** the codebase is inspected for a disclosure store owned by this capability
- **THEN** none exists, and all disclosures are written through the shared record

#### Scenario: Reviewing anchoring after the fact

- **WHEN** a reviewer asks whether an interviewer's judgement was formed before or after seeing AI
  output
- **THEN** the disclosure records answer it from stored data alone

### Requirement: No disclosure record shows a score disclosed to an interviewing actor before their notes

The disclosure history SHALL contain no record of a match score or rank position having been
disclosed to an interviewing actor in a pre-interview context.

*Source: `C-10`'s stated consequence — "**the AI agreement rate metric becomes meaningful.** With the
score hidden, comparing a panelist's verdict against the AI ranking measures genuine agreement rather
than a self-fulfilling one." The metric is `insight-and-reporting`'s to compute; what this capability
owes is that the premise is true and evidenced rather than asserted.*

#### Scenario: Disclosure history audited

- **WHEN** disclosure records for interviewing actors in pre-interview contexts are enumerated
- **THEN** none names a match score or rank position

#### Scenario: A path that would produce one

- **WHEN** a code path is added that would disclose a score to an actor holding only the projected
  view
- **THEN** the test suite fails

### Requirement: Evidence source labels are visible in the projection

Each projected claim SHALL carry its source label and evidence reference, so a reader can see which
claims rest on the resume and which rest on an earlier interview or scorecard. Resolving a reference
SHALL remain permission-scoped on the referenced record's own terms.

*Source: `G-02`, which records that source labelling "makes the `D05` mitigation real — an
interviewer can see which claims rest on the resume versus an earlier interview"; `INT-005`'s
source-labelled summaries; `BR-008`; `ai-platform-governance`'s permission-scoped reference
resolution, under which the label and reference are always visible and the referenced content is not.*

#### Scenario: Labelled projection

- **WHEN** projected claims are returned
- **THEN** each carries its own source label and evidence reference

#### Scenario: Carried evidence

- **WHEN** a claim rests on evidence originating on a different Application
- **THEN** the reference identifies the record it came from, so its carried origin is visible

#### Scenario: Reference to a record the reader cannot see

- **WHEN** the actor resolves a reference to a record they lack permission on
- **THEN** the label and reference identifier are returned and the content is denied

### Requirement: This capability builds no interview surface and no note template

This capability SHALL provide the projection and its enforcement only. It SHALL NOT build the
Interview Console, the structured note template, interview question generation, or any interview
state.

*Source: `C-10`'s retained mitigation — "the evidence-required note template remains valuable and is
retained" — which belongs to `interview-pipeline`'s `TS-BL-058`/`TS-BL-059` along with `INT-004`'s
question generation and `INT-006`'s structured fields. `D.10` places all of them in that feature.
`design.md` D7 records the seam: the contract is a payload the console renders, mirroring how
`candidate-intake` kept its duplicate warning to a payload this feature's board renders.*

#### Scenario: Scope inspected

- **WHEN** this capability's surface area is enumerated
- **THEN** it contains the projection and its enforcement, and no interview screen, note template,
  question generation or interview state

#### Scenario: Console consumes the projection

- **WHEN** an interview surface needs a candidate's AI context
- **THEN** it consumes this projection rather than assembling its own view of ranking output

## Purpose

The surface an interviewer works from: their assigned interviews, the AI context they are permitted
to see, AI-suggested questions they may use, edit or ignore, and the fixed structured note fields they
submit. Owned by `TS-BL-058`.

## ADDED Requirements

### Requirement: The console shows only assigned interviews

An interviewing actor SHALL see only the interviews they are assigned to, unless broader permission is
granted through the permission matrix. The scope SHALL be applied as a query predicate.

*Source: `INT-003`, `D01` as retained by `C-02`, `C-03`, and `interview/round-scheduling`'s assignment
scope, which this capability consumes rather than re-deriving.*

#### Scenario: Assigned interviews listed

- **WHEN** an interviewing actor opens the console
- **THEN** only interviews assigned to them are listed

#### Scenario: Broader permission granted

- **WHEN** an administrator grants a broader interview-read permission to a specific user
- **THEN** that user's listing widens with no code change, and the grant is audited

#### Scenario: Role comparison in code

- **WHEN** this capability is inspected for a role-name comparison deciding what is listed
- **THEN** none exists, and the evaluator's verdict decides it

### Requirement: The console consumes the ranking-context projection and builds no second view

The AI ranking context shown in the console SHALL be the projection the matching capability produces.
This capability SHALL NOT assemble its own view of ranking output, SHALL NOT display a match score or
rank position, and SHALL NOT reference any other candidate on the posting.

*Source: `C-10`, which **amends `D05`, reversing it** — "gaps yes, score no"; `INT-005`'s
source-labelled fitment and gap summaries; `matching-and-ranking`'s `matching/ai-context-visibility`
requirement "This capability builds no interview surface and no note template", whose companion
scenario states that an interview surface "consumes this projection rather than assembling its own
view of ranking output". `D05`'s per-candidate / no-rank-position rule stands and is the tighter
constraint. `design.md` D5.*

#### Scenario: Context rendered

- **WHEN** an assigned interviewer opens a candidate in the console
- **THEN** source-labelled fitment and gap summaries are shown with the job description and resume

#### Scenario: Score sought

- **WHEN** the console's payload is inspected on the wire
- **THEN** it carries no match score, no rank position and no cohort size

#### Scenario: Other candidates

- **WHEN** the console's payload is inspected
- **THEN** it references no other candidate on the posting

#### Scenario: Second view sought

- **WHEN** this capability is inspected for its own assembly of ranking output
- **THEN** none exists, and the projection is consumed

### Requirement: The active-interview state removes navigation chrome

The console SHALL offer an active-interview state rendered in the design system's presentation shell,
with navigation chrome removed and content full-viewport. This state SHALL be defence in depth only:
every withholding guarantee SHALL remain enforced server-side and by the permission evaluator,
independent of which shell state is rendered.

*Source: the layout specification's third shell state, built by `design-system`'s `TS-BL-008` and
recorded there as shipping with no consumer; `S.3`, which names the Interview Console during an active
interview as a candidate and decides neither. `design.md` D5 assigns it and records why the guarantee
must not rest on it — `config.yaml`'s rule that withholding is applied at produce time, not as a
display filter.*

#### Scenario: Active interview

- **WHEN** an interviewer enters the active-interview state
- **THEN** the presentation shell renders with navigation chrome absent

#### Scenario: Guarantee independent of shell state

- **WHEN** the same actor's payload is compared between the authenticated and presentation shells
- **THEN** the withheld values are absent from both

#### Scenario: New component added

- **WHEN** this capability's components are enumerated
- **THEN** all come from the design system and none is newly defined here

### Requirement: AI-suggested questions are generated at interview setup and are advisory

The system SHALL generate a categorized question set for an interview round from the job requirements,
fitment and gaps, and attach it to that round. The interviewer SHALL be able to use, edit or ignore any
question. The question set SHALL NOT constrain what may be asked or what may be recorded.

*Source: `INT-004`; §16.1's Interview Question Generation — trigger "shortlist or interview setup",
human gate "Interviewer may use, edit, or ignore"; §16.3's `interview_questions` family, required
output "categorized JSON question list"; §25's Question Generation job type; §12.2's
`interview_rounds.question_set_id`. `design.md` D1 records that this family had no owning item in
`D.10` and why it lands here rather than on the scheduling item.*

#### Scenario: Question set generated

- **WHEN** a round is created
- **THEN** a categorized question set is generated and attached to that round

#### Scenario: Questions ignored

- **WHEN** an interviewer submits notes without using any suggested question
- **THEN** submission succeeds and no completeness requirement references the question set

#### Scenario: Generation unavailable

- **WHEN** question generation fails or the provider is unavailable
- **THEN** the round remains conductable and notes remain submittable, with the failure visible rather
  than silent

#### Scenario: Marked as AI-generated

- **WHEN** a question set is displayed
- **THEN** it carries the AI-generated marking, which no approval in this capability clears

### Requirement: Question generation runs through the gateway on references and registers one job type

Question generation SHALL be invoked through the AI gateway in its fixed request envelope, carrying
references rather than content, as a new version of the already-registered `interview_questions`
family. It SHALL register the Question Generation background job type subscribing to the round-created
event, and SHALL register no job type for the gateway call itself.

*Source: `ai-platform-governance`'s `ai-platform/ai-gateway` (sole egress, uniform envelope,
references never content), its `design.md` D3 and D10 — "AI runs register as a job type; they do not
get a queue" — `AI-003`'s versioned registry, `TS-BL-030`'s ten registered families, and §25's
Question Generation row. `matching-and-ranking` D1 records this same both-halves-true pattern and
names getting it wrong in either direction as the most likely error in a feature of this shape.*

#### Scenario: Envelope contents

- **WHEN** a question-generation request is inspected
- **THEN** it carries posting and application references and no job description text, no resume text
  and no candidate personal data

#### Scenario: One job type

- **WHEN** this capability's registered job types are enumerated
- **THEN** exactly one is present — the round-created subscriber — and none corresponds to a gateway
  call

#### Scenario: Provider call sought

- **WHEN** this capability is inspected for an outbound call to a model provider
- **THEN** none exists

#### Scenario: Duplicate delivery

- **WHEN** the same round-created event is delivered twice
- **THEN** one question set exists for that round

### Requirement: Structured note fields are the fixed set

A submitted interview note SHALL carry strengths, gaps, technical validation, project depth,
communication, concerns, recommendation and next-step suggestion. The recommendation SHALL be one of
`strong_yes`, `yes`, `maybe`, `no`, `strong_no`. The field set SHALL be the same for every interview
type.

*Source: `INT-006` as adopted by `C-06`'s resolution — "Interview note fields (`INT-006`, fixed)" —
and the 5-point scale carried unchanged from `D15` and confirmed by `C-06`. `C-06` also closed the
"note templates per interview type" open item: one fixed field set across all types.*

#### Scenario: Note submitted

- **WHEN** a note is submitted
- **THEN** it carries all eight structured fields and a recommendation from the five-value scale

#### Scenario: Recommendation outside the scale

- **WHEN** a note is submitted with a recommendation outside the five values
- **THEN** it is rejected

#### Scenario: Type-specific field set sought

- **WHEN** the note field sets for two interview types are compared
- **THEN** they are identical

### Requirement: Notes are stored with the forms downstream evidence depends on

A note SHALL be stored as normalized structured fields, semantic tags, raw notes, embeddings,
interviewer identity, source references and timestamps.

*Source: `INT-007`. The embedding form is what makes `interview_note_section` registrable against the
vector pipeline (`design.md` D6); the interviewer identity and timestamps are what keep attribution
intact when `C-08` permits a non-author edit (`design.md` D8).*

#### Scenario: Stored forms

- **WHEN** a submitted note is inspected
- **THEN** structured fields, semantic tags, raw notes, an embedding reference, interviewer identity,
  source references and timestamps are all present

#### Scenario: Attribution recorded at submission

- **WHEN** a note is submitted
- **THEN** the submitting identity is recorded on it

### Requirement: The interview note section is registered as a vector source type

This capability SHALL register `interview_note_section` against the platform's vector source-type
registration seam. It SHALL NOT implement a second embedding pipeline, a second document segmenter, or
a second vector store.

*Source: `VEC-001`'s four source types; `matching-and-ranking` `design.md` D2, which embedded two of
the four, built the pipeline source-type-driven with all four enum values present "so Phase 3 adds a
registration and no schema change", and named this capability as the registrant of this type.
`design.md` D6.*

#### Scenario: Type registered

- **WHEN** a note is submitted
- **THEN** vector records are produced under the `interview_note_section` source type through the
  existing pipeline

#### Scenario: Second pipeline sought

- **WHEN** this capability is inspected for its own embedding pipeline, chunker or vector store
- **THEN** none exists

#### Scenario: Schema change required

- **WHEN** the registration is performed
- **THEN** it requires no change to the vector record schema

### Requirement: The console is a recording surface and decides nothing

This capability SHALL NOT set an Application's disposition, shortlist status or final outcome, and
SHALL NOT trigger a workflow transition other than a round's own completion.

*Source: `WF-001`, `config.yaml`'s standing rule, `project.md`'s governing rule. Stated because a
recommendation field on a note is the most natural place for someone to wire a disposition; the
disposition surface is `interview/shortlisting` and the evaluation gate is
`interview/scorecard-approval`. A panelist's `strong_no` is evidence, not an outcome.*

#### Scenario: Disposition write sought

- **WHEN** this capability is inspected for a write to an Application's disposition fields
- **THEN** none exists

#### Scenario: Strong negative recommendation

- **WHEN** a note is submitted recommending `strong_no`
- **THEN** no disposition occurs and the Application's state is unchanged

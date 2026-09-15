## Purpose

`G-03` applied to ranking output: where the evidence does not support a conclusion, the system says
so instead of guessing — and, the consequence that only bites here, absence of evidence is never
scored as weakness. Owned by `TS-BL-055`.

## ADDED Requirements

### Requirement: Missing evidence produces an insufficiency marker, not a value

Where the available evidence does not support a ranking criterion, a fitment claim or a gap
classification, the output SHALL carry an insufficiency marker for that element rather than an
inferred value. The marker SHALL identify what was missing.

*Source: `G-03`, `AI-008` — "AI outputs shall return insufficiency reasons instead of guessing when
evidence is missing or unclear"; `DOC-009`. The wording, shape and vocabulary are
`candidate-intake`'s `candidate/resume-enrichment` requirement of the same name, reused deliberately
rather than reinvented — `design.md` D10 records why one vocabulary across both AI stages matters
more than a locally tuned one.*

#### Scenario: Criterion with no supporting evidence

- **WHEN** a permitted ranking criterion has no supporting evidence in the retrieved set
- **THEN** the output marks that criterion insufficient rather than assigning it a value

#### Scenario: Marker identifies what was missing

- **WHEN** an insufficiency marker is read
- **THEN** it names the criterion or claim it applies to and what evidence was absent

#### Scenario: Inferred value in output

- **WHEN** output assigns a criterion a value with no evidence reference
- **THEN** the output is rejected as a contract violation and recorded as a failed run

### Requirement: Insufficiency is distinguishable from a field never requested

A ranking element marked insufficient SHALL be distinguishable from one that was never evaluated,
and both SHALL be distinguishable from one evaluated as weak.

*Source: `candidate-intake`'s `candidate/resume-enrichment` scenario "Insufficiency distinguishable
from absence," reused verbatim in shape; `G-03`, `DOC-009`. The third distinction is this
capability's addition, and it is the one that matters for ranking — see the next requirement.*

#### Scenario: Three states read

- **WHEN** a ranking entry's elements are read
- **THEN** insufficient, not evaluated and evaluated-as-weak are three distinguishable states

#### Scenario: Not-evaluated element

- **WHEN** a criterion was not part of this posting's evaluation at all
- **THEN** it reads as not evaluated rather than as insufficient

### Requirement: Absence of evidence is not scored as weakness

An element marked insufficient SHALL NOT contribute to the score as a negative signal. A candidate
whose evidence is thin SHALL receive a score that is withheld or explicitly qualified, never a low
score derived from what was absent.

*Source: `G-03`'s stated reason for adoption — insufficiency "attacks the failure mode most likely to
erode trust in ranking"; `AI-007`'s prohibition on unsupported claims; `project.md`'s advisory-only
rule, under which a score is a recommendation to a human and a silently deflated score is a
recommendation the human cannot interrogate. `design.md` D10, and `design.md` D6 for the unenriched
candidate this most often describes.*

#### Scenario: Thin resume scored

- **WHEN** a candidate's evidence supports few criteria
- **THEN** the score is withheld or qualified, and the insufficient criteria are listed rather than
  counted against the candidate

#### Scenario: Weak against insufficient

- **WHEN** a candidate with weak evidence and a candidate with absent evidence are compared
- **THEN** their entries are distinguishable, and the absent-evidence candidate's score is not lower
  by virtue of the absence

#### Scenario: Negative contribution sought

- **WHEN** the scoring path is inspected for a negative contribution from an insufficiency marker
- **THEN** none exists

### Requirement: Insufficient mandatory-criteria evidence resolves to unclear or manager review, never to a block

Where the evidence does not establish whether a mandatory criterion is met, the AI-proposed
mandatory-criteria status SHALL be unclear or manager review required. It SHALL NOT be does not meet.

*Source: `G-01`'s adopted asymmetry, and its stated purpose — the priority lane must be safe, so "a
pre-vetted candidate cannot surface for a role they are categorically ineligible for," while the AI
must not gate a decision about a person. `matching/ranking-engine` carries the general prohibition;
this requirement is the specific case where the temptation is strongest, because absent evidence and
disqualifying evidence look alike to a model.*

#### Scenario: Mandatory criterion unevidenced

- **WHEN** the retrieved evidence does not establish a mandatory criterion either way
- **THEN** the proposed status is unclear or manager review required

#### Scenario: Blocking value from absence

- **WHEN** output proposes does not meet on the basis of absent evidence
- **THEN** the output is rejected as a contract violation

#### Scenario: Human resolution available

- **WHEN** a status of unclear or manager review required is presented
- **THEN** the human resolution path on the ranking board is available and its use is audited

### Requirement: Insufficiency is a first-class insight, retrievable and displayed

Insufficiency output SHALL be persisted as its own insight type, retrievable with the ranking entry,
and SHALL be displayed on every surface that displays the affected claim or score.

*Source: §12.2's `ai_insights.insight_type` enum, which already carries an `insufficiency` value
alongside fitment, gap, question, summary, risk and recommendation; `RANK-002`'s displayed columns;
`UI-003`'s requirement that a page show state, owner, next action and blockers. An insufficiency
stored and never shown is a guess from the reader's point of view.*

#### Scenario: Insufficiency persisted

- **WHEN** a ranking run produces insufficiency output
- **THEN** it is stored as an insufficiency insight linked to its run and its ranking entry

#### Scenario: Board display

- **WHEN** a ranking entry with insufficiency renders on the board
- **THEN** the insufficiency is visible alongside the affected claim or score, not only in a detail
  view

#### Scenario: Projected display

- **WHEN** projected AI context is returned to an interviewing actor
- **THEN** insufficiency affecting the fitment or gap claims they receive is present in the projection

### Requirement: Insufficiency output is bounded and carries no unsupported detail

Each free-text field in insufficiency output SHALL declare a machine-checkable length bound, and an
insufficiency reason SHALL describe what was missing without asserting anything about the candidate.

*Source: `AGENTS.md`'s standing bar and `S.5`, whose concern applies with particular force to a field
that exists to say "we do not know"; `ai-platform-governance` `design.md` D7's per-field bound rule;
`AI-007`'s prohibition on unsupported claims — an insufficiency reason that speculates about why the
evidence is missing is itself an unsupported claim.*

#### Scenario: Over-long reason

- **WHEN** an insufficiency reason exceeds its declared bound
- **THEN** the run is recorded as failed with the violation preserved and no content is persisted

#### Scenario: Speculative reason

- **WHEN** an insufficiency reason asserts a fact about the candidate rather than describing the
  absent evidence
- **THEN** it is rejected as a contract violation

### Requirement: Insufficiency is covered by the evaluation corpus for all three families

Each of the three ranking families SHALL have corpus cases exercising insufficiency, including a
low-information input, and production activation SHALL be refused until those cases pass.

*Source: `AI-015` and `C-09`, which name "low-information resumes" among the required ranking-template
test inputs; `ai-platform-governance`'s promotion gate and its `design.md` D8, which argues the gate
biting before the corpus exists is the correct order. Without a low-information case, the
insufficiency path is code that never ran.*

#### Scenario: Corpus cases present

- **WHEN** the corpus for the three ranking families is enumerated
- **THEN** each has at least one low-information case asserting insufficiency rather than an inferred
  value

#### Scenario: Promotion without cases

- **WHEN** a ranking family's template version is promoted to production with no passing
  insufficiency case
- **THEN** activation is refused

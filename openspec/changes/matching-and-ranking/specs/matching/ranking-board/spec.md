## Purpose

The AI Ranking Board: the Practice Manager and Recruitment Manager surface where a posting's
candidates are presented with score, mandatory-criteria status, fitment, gaps, evidence and run
version — every claim traceable, every recommendation labelled as advisory, and every human
divergence recorded rather than substituted. Owned by `TS-BL-053`.

## ADDED Requirements

### Requirement: The board presents the full ranking row

The board SHALL display, for each candidate on a posting, rank, match score, mandatory-criteria
status, fitment summary, gap summary, evidence references and the AI run version behind the entry.

*Source: `RANK-002`; §14.2's AI Ranking Board row — "ranked candidates, match score, mandatory
criteria, fitment, gaps, evidence, override." The run version is on the row rather than behind a
detail view because `RANK-002` lists it among the displayed columns, and because a score whose
provenance takes a second navigation is a score read without its provenance.*

#### Scenario: Board rendered

- **WHEN** an authorized user opens the ranking board for a posting
- **THEN** each candidate row carries rank, match score, mandatory-criteria status, fitment, gaps,
  evidence references and the run version

#### Scenario: Posting never ranked

- **WHEN** a posting has never been ranked
- **THEN** the board renders an explicit not-yet-ranked state inside the table container rather than
  an empty grid or a zero score

#### Scenario: Entry marked stale

- **WHEN** an entry's inputs have moved on since it was computed
- **THEN** the row shows the staleness and names the version it was computed against

### Requirement: The board is built from existing design-system components and adds none

The board SHALL be assembled from the dense data table, the AI-disclosure marking and the
evidence-source label components, and SHALL introduce no new component, no raw color and no one-off
spacing value.

*Source: `AGENTS.md`'s standing product-quality bar — reuse shared patterns before inventing new
ones; `design-system`'s `design-system/data-table` capability, whose Purpose names the ranking board
as one of the three screens it exists for; `config.yaml`'s rule that design tokens are the single
source of truth. `design-system`'s evidence-source label component "renders the source value it is
given without interpreting or deriving it," so this surface derives none.*

#### Scenario: Components inspected

- **WHEN** the board's components are enumerated
- **THEN** each is an existing design-system component and no new one was introduced for it

#### Scenario: Source values rendered

- **WHEN** an evidence label renders
- **THEN** its source value came from the closed vocabulary rather than being derived on this surface

#### Scenario: Wide board

- **WHEN** the board's columns exceed the available width
- **THEN** the table scrolls horizontally within its own container and the page body does not

### Requirement: The board is a Practice Manager, Recruitment Manager and permitted-Recruiter surface

Access to the board SHALL be evaluated by the central permission evaluator. The Interviewer and
Hiring Panel Member roles SHALL NOT have access to it. Recruiter access SHALL be resolved through
the configured read scope and the assigned-postings predicate rather than by logic local to this
surface.

*Source: §8, which lists AI Ranking Board users as "Practice Managers, Recruitment Managers,
permitted Recruiters" — Interviewer is not among them; `C-10`'s resolution, which states that "the AI
Ranking Board remains a Practice Manager / Recruitment Manager surface (§8)"; `C-03` as resolved and
`access-control-and-admin`'s `assigned-postings` scope predicate; `AUTHZ-003`, `AUTHZ-004`.*

#### Scenario: Interviewer requests the board

- **WHEN** an Interviewer requests the ranking board directly
- **THEN** the request is denied server-side, not merely hidden in the interface

#### Scenario: Recruiter scope applied

- **WHEN** a Recruiter opens the board
- **THEN** the scope filter is applied as a query predicate and the reported count reflects only
  in-scope records

#### Scenario: Permission-aware affordances

- **WHEN** a user lacks an action the board offers
- **THEN** that affordance is absent rather than present and disabled, and a direct request for it is
  refused server-side

### Requirement: The board states that rankings are advisory and claims no bias-freedom

The board SHALL state that rankings are AI-assisted recommendations subject to human review and
bias monitoring, and SHALL NOT present any claim that its recommendations are free of bias. AI
content SHALL carry its AI-generated marking until a recorded human approval.

*Source: `RANK-004`, `UI-006` — ranking pages "shall display human-review disclaimers and avoid
claiming bias-free recommendations"; `AI-001`, `UI-004`; `project.md`'s central rule.
`PRV-006` keeps bias auditing itself out of scope, which is precisely why the disclaimer is
load-bearing rather than decorative here.*

#### Scenario: Disclaimer present

- **WHEN** the board renders
- **THEN** the human-review disclaimer is visible without interaction

#### Scenario: Bias claim

- **WHEN** the board's text is inspected
- **THEN** no statement asserts that its recommendations are bias-free

#### Scenario: Unapproved content marked

- **WHEN** AI-produced fitment, gap or score content renders before human approval
- **THEN** it carries its AI-generated marking

### Requirement: A human override requires a reason and never replaces the AI output

Overriding an AI ranking output SHALL require a mandatory reason and SHALL record actor, timestamp,
the original AI output reference and the human value. The original AI output SHALL remain
retrievable. An override submitted without a reason SHALL be rejected.

*Source: `RANK-006`, `BR-016`, `G-06`, which calls this "the concrete implementation of
advisory-only: AI output stays on the record, the human decision sits beside it, and the divergence
becomes measurable"; §13.2's override endpoint;
`ai-platform-governance`'s human-override-record requirement, which this capability consumes rather
than rebuilds.*

#### Scenario: Override recorded

- **WHEN** an authorized user overrides an AI ranking output with a reason
- **THEN** the override record captures actor, timestamp, reason, the original output reference and
  the human value, and the original output remains readable

#### Scenario: Override without a reason

- **WHEN** an override is submitted with no reason
- **THEN** it is rejected

#### Scenario: Original output after override

- **WHEN** an overridden entry is read
- **THEN** both the AI output and the human value are present and distinguishable

#### Scenario: Divergence measurable

- **WHEN** override records for a period are queried
- **THEN** the rate at which humans diverged from AI ranking output is computable from stored records
  alone

### Requirement: Only a human sets a blocking mandatory-criteria status, from this surface

Setting the mandatory-criteria status to does not meet SHALL be a human action taken on this
surface, requiring a mandatory reason and producing an audit record. The AI-proposed value SHALL
remain visible alongside the human value.

*Source: `G-01`'s adopted decision — "only a human may set a blocking value, with a reason, audited";
§12.2's `mandatory_criteria_status`; `AI-010`, `BR-007`. `design.md` D11 records the split between
this capability and `matching/ranking-engine`, which may propose three of the four values and never
the fourth.*

#### Scenario: Human sets the blocking value

- **WHEN** an authorized user sets does not meet with a reason
- **THEN** the value is stored, the change is audited with previous and new value, and the
  AI-proposed value remains visible

#### Scenario: Blocking value without a reason

- **WHEN** the blocking value is submitted with no reason
- **THEN** it is rejected

#### Scenario: Unpermitted actor

- **WHEN** a user without the required action attempts to set the blocking value
- **THEN** the request is denied server-side

### Requirement: Prior ranking versions are viewable from the board

The board SHALL present the active ranking version by default and SHALL allow an authorized user to
view a prior version as it stood, with its own tuple values.

*Source: `G-07`, `RANK-005`'s preservation requirement — a preserved prior version that no surface
can open is preserved in name only. `matching/ranking-score` holds the versions; this requirement is
the board reading them.*

#### Scenario: Prior version opened

- **WHEN** an authorized user opens a prior ranking version
- **THEN** its entries render as they stood, with the versions that produced them named

#### Scenario: Active version indicated

- **WHEN** the board renders
- **THEN** which version is active is unambiguous

### Requirement: The board renders the advisory duplicate warning it is given

Where a posting-level duplicate warning exists for a candidate on this posting, the board SHALL
render it as a non-blocking advisory marker identifying the counterpart. The board SHALL NOT
evaluate duplicate similarity itself, and the warning SHALL NOT suppress, reorder or block any row.

*Source: `candidate-intake`'s `candidate/duplicate-backstop` capability and that feature's
`design.md` D1, which deliberately kept the contract to "a warning payload the list renders, so
`TS-BL-053` consumes rather than accommodates it," and which asserts that no ranking write follows
from the condition.*

#### Scenario: Warning rendered

- **WHEN** a duplicate warning exists for a candidate on the posting
- **THEN** the row carries a non-blocking advisory marker naming the counterpart

#### Scenario: Warning absent or capability disabled

- **WHEN** no warning payload is supplied
- **THEN** the board renders normally with no placeholder and no error

#### Scenario: Warning does not affect ranking

- **WHEN** a warning is present
- **THEN** the row's rank, score and available actions are unchanged

### Requirement: The board decides nothing about a candidate's progress

This capability SHALL NOT set an application's stage, shortlist status, shortlist reason or final
outcome, and SHALL NOT trigger any workflow transition.

*Source: `config.yaml`'s rule that every state transition goes through the workflow service and no
surface writes a state field directly; `BR-007`; the shortlisting decision surface is
`interview-pipeline`'s `TS-BL-056`, which depends on this item. Stated as a requirement because a
board that shows a score next to a candidate is the most natural place for someone to add a
shortlist button.*

#### Scenario: State write sought

- **WHEN** this capability is inspected for writes to an application's stage, shortlist or outcome
  fields
- **THEN** none exists

#### Scenario: Transition triggered

- **WHEN** any board action is exercised
- **THEN** no workflow transition results

### Requirement: The board's comparative data is not available through the per-candidate insight path

The per-application insight response SHALL NOT carry rank position, match score, cohort size, or any
value from which the standing of other candidates can be derived. Comparative data SHALL be
available only through the board's own posting-scoped response.

*Source: `D05`'s still-standing consequence — "AI context shown to an Interviewer should be
**per-candidate**, not the comparative leaderboard. Rank position ('#1 of 12') leaks the existence
and standing of other candidates" — which `C-10` did not reverse; §13.2's two distinct endpoints, one
posting-scoped and one application-scoped. `design.md` D8 records why this constrains the board even
though the board's own audience may see rank: the per-candidate path is what a role-restricted
surface reuses, and a leak there is a leak everywhere downstream.*

#### Scenario: Per-application insight response inspected

- **WHEN** the per-application insight response is inspected
- **THEN** it carries fitment, gaps, evidence and mandatory-criteria status, and no rank, score or
  cohort size

#### Scenario: Cohort size derivable

- **WHEN** the per-application insight response is examined for values from which the number or
  standing of other candidates could be computed
- **THEN** none is present

#### Scenario: Board response carries rank

- **WHEN** an authorized board user loads the posting-scoped response
- **THEN** rank and score are present, because that audience's task is comparison

### Requirement: Export is offered only against a supplied affirmative decision

An export control SHALL be presented only when an affirmative export decision is supplied by the
caller, and a direct export request SHALL be denied server-side without the Export action. Every
export SHALL produce an audit record.

*Source: `UI-008`, `UI-002`; `design-system`'s data-table requirement that "the table takes the
decision as input" and evaluates no permission itself; `API-003`, and `access-control-and-admin`'s
audited-export treatment. An exported board is candidate assessment data leaving the system, which
is why the audit record is not optional.*

#### Scenario: No export decision supplied

- **WHEN** the board renders without an affirmative export decision
- **THEN** no export control is offered

#### Scenario: Direct export without permission

- **WHEN** an export is requested directly by a user without the Export action
- **THEN** it is denied

#### Scenario: Export audited

- **WHEN** an export completes
- **THEN** an audit record is written carrying references and no candidate personal data

### Requirement: The board ships behind a feature flag

The board and its endpoints SHALL be gated behind a declared feature flag, disabled by default, and
a disabled capability SHALL answer as not found.

*Source: `config.yaml`'s rule that feature flags are declared and never invented, that an unknown
flag raises rather than resolving false, and that a disabled capability answers 404; `ENG-009`. The
same shape every Phase-2 surface ships behind.*

#### Scenario: Flag disabled

- **WHEN** the board's flag is disabled
- **THEN** its routes and endpoints answer as not found

#### Scenario: Undeclared flag

- **WHEN** an undeclared flag name is resolved
- **THEN** it raises rather than resolving false

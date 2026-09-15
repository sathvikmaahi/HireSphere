## Purpose

The three prompt families that turn a posting and a resume into a score, a fitment summary and a
gap summary — invoked through the one governed egress on two references rather than on content,
constrained to role-relevant signals, and structurally incapable of asserting anything that blocks
a person. Owned by `TS-BL-052`.

## ADDED Requirements

### Requirement: Ranking is invoked through the gateway in the fixed envelope, carrying a posting reference and a resume version reference

Each ranking invocation SHALL be requested through the AI gateway using the uniform request
envelope, with the input payload carrying a posting reference and a resume version reference as
identifier-and-version. The request SHALL NOT carry job description content, posting text, resume
text, or any candidate personal data.

*Source: `ai-platform-governance`'s `ai-platform/ai-gateway` uniform-envelope requirement and its
`design.md` D3, which fixed the envelope against a different family and generalized it to this one
**by name**: "`candidate_ranking` sends posting and resume references." This is the first consumer
sending two references; `hiring-postings`' and `candidate-intake`'s families each sent one.
`design.md` D1.*

#### Scenario: Ranking requested

- **WHEN** ranking is requested for an application
- **THEN** the request carries the family, the calling context, a posting reference and a resume
  version reference, in the same envelope every other family uses

#### Scenario: Request payload inspected

- **WHEN** a ranking request payload is inspected
- **THEN** it contains no job description text, no posting text, no resume text and no candidate
  personal data

#### Scenario: References resolve to specific versions

- **WHEN** a later resume version or a later job description version exists
- **THEN** the references recorded on a completed ranking still resolve to the versions it was run
  against

#### Scenario: One reference supplied

- **WHEN** a ranking request is submitted with only one of the two references
- **THEN** it is rejected as a contract violation rather than run against a partial input

### Requirement: Three registered families, three runs, one envelope

Candidate ranking, fitment summary and gap summary SHALL each be invoked as its own registered
prompt family with its own template version, model configuration, output contract and length
bounds. They SHALL NOT be collapsed into a single invocation.

*Source: §16.1, which lists Candidate Ranking, Fitment Summary and Gap Summary as three distinct
capabilities with distinct triggers and outputs; §16.3, which registers `candidate_ranking`,
`fitment_summary` and `gap_summary` as three of the ten families with three different required
output formats; `ai-platform-governance`'s per-family model configuration and per-family conciseness
bounds, both of which attach to a family. `design.md` D1.*

#### Scenario: Three families invoked

- **WHEN** ranking, fitment and gap output are produced for an application
- **THEN** three runs exist, one per family, each naming its own template version and model version

#### Scenario: Per-family configuration honoured

- **WHEN** two of the three families are configured with different models
- **THEN** each family's runs use its own configured model and each run records which was used

#### Scenario: Families collapsed

- **WHEN** the invocation paths are enumerated
- **THEN** no path produces output for more than one registered family from a single run

### Requirement: No provider is called from this capability, and only the batch orchestrator is a registered job type

Every model invocation SHALL go through the gateway. This capability SHALL NOT contain an outbound
call to a model provider and SHALL NOT register a background job type for an individual gateway
call. It SHALL register exactly one job type: the batch ranking orchestrator that subscribes to the
posting-opened event.

*Source: `config.yaml`'s rule that the gateway is the only egress; `ai-platform-governance`
`design.md` D10 — "AI runs register as a job type; they do not get a queue" — under which every
gateway call already dispatches asynchronously inside the gateway; and §25, which lists Candidate
Ranking as a background job type triggered by posting open, whose subscriber `hiring-postings`
deliberately did not register. `design.md` D1 records why both halves are true at once, which is the
error `candidate-intake` named as the most likely in a feature of this shape.*

#### Scenario: Outbound provider call

- **WHEN** this capability's code is inspected for outbound calls to a model provider
- **THEN** none exists

#### Scenario: Job types enumerated

- **WHEN** the job types this capability registers are enumerated
- **THEN** exactly one appears — the batch ranking orchestrator — and no per-family invocation job
  appears

#### Scenario: Orchestrator uses the shared dispatch pattern

- **WHEN** the batch ranking orchestrator is dispatched
- **THEN** it runs as a registered job type through the platform's single dispatch pattern, carrying
  the originating correlation identifier

#### Scenario: Duplicate delivery of a ranking request

- **WHEN** the same ranking request is delivered more than once
- **THEN** exactly one run is issued per family and exactly one ranking entry results

### Requirement: Ranking is triggered by posting open, by an authorized user, and by a ranking-relevant requirement change

Ranking SHALL be triggerable by an authorized user for all candidates linked to a posting, SHALL be
triggered by the posting-opened event, and SHALL be triggered by a change to a posting's
ranking-relevant requirement fields. No other change SHALL trigger a ranking automatically.

*Source: `RANK-001`; `JOB-010` and `hiring-postings`' posting-lifecycle requirement, which publishes
the event and registers no subscriber; §25's Candidate Ranking row, whose triggers are "posting open,
manual trigger, requirement change." A live posting's job description version is immutable
(`hiring-postings`' pinning requirement), so "requirement change" resolves to the posting's own
authoritative requirement fields. `design.md` D5 records the full table, including what deliberately
does not trigger.*

#### Scenario: Posting opens

- **WHEN** a posting opens
- **THEN** the batch ranking orchestrator runs for its linked candidates

#### Scenario: Manual trigger

- **WHEN** an authorized user triggers ranking for a posting
- **THEN** a new ranking version is produced for all candidates linked to it

#### Scenario: Ranking-relevant posting field changed

- **WHEN** a ranking-relevant requirement field on a posting changes
- **THEN** a new ranking version is produced and the prior version is preserved

#### Scenario: Non-triggering change

- **WHEN** an application is re-pointed to a different resume version, or a candidate merge occurs
- **THEN** no ranking is triggered, and the affected entry is marked stale instead

#### Scenario: Event publication failed upstream

- **WHEN** the posting-opened event was never delivered
- **THEN** the posting is still open and ranking remains available by manual trigger

### Requirement: Only role-relevant signals are used, and prohibited attributes are structurally excluded

Ranking SHALL consider only the permitted role-relevant signals. Age, gender, race, ethnicity,
religion, disability, marital status, nationality, candidate photograph and any other protected or
sensitive attribute SHALL NOT be used as a ranking signal, and SHALL be redacted from scoring input
before the prompt is composed.

*Source: `AI-005`, §16.4's eleven permitted and ten prohibited signals;
`ai-platform-governance`'s personal-data-minimization requirement, which places
protected-attribute redaction at the gateway so a new consumer inherits it. This capability is the
first family in the product whose input is destined for scoring, so that clause first applies here.*

#### Scenario: Permitted signals

- **WHEN** a ranking run's criteria are enumerated
- **THEN** each is one of §16.4's permitted signals

#### Scenario: Protected attribute in input

- **WHEN** input destined for a ranking family contains a protected or sensitive attribute
- **THEN** it is redacted before the prompt is composed, and the run records that redaction applied

#### Scenario: Prohibited signal in output

- **WHEN** output cites a prohibited attribute as a criterion
- **THEN** the output is rejected as a contract violation and recorded as a failed run

#### Scenario: Contact information excluded

- **WHEN** the content the resolved references supply to a ranking prompt is inspected
- **THEN** the structured email, phone and name fields are absent, and the controlled exception
  permitting candidate contact information is not invoked by any of the three families

### Requirement: Resume-derived content is marked untrusted at the boundary

Content the resolved resume reference supplies SHALL be marked as untrusted document-derived
content, so that the gateway's separation of trusted instructions from untrusted content applies to
it.

*Source: `AI-012`, `AI-013`, `AI-014`, `SEC-008`, `SEC-014`; `reference/spec.md` §36's
prompt-injection risk, and `D18`, which names the ranking stage as the larger injection surface
rather than intake. `candidate-intake` marks its own boundary for the same reason; the isolation
mechanism is the gateway's, and what this capability owes is the marking.*

#### Scenario: Input marked

- **WHEN** ranking input is handed to the gateway
- **THEN** resume-derived content is marked as untrusted document-derived content

#### Scenario: Instruction-shaped resume content

- **WHEN** a resume contains text shaped as an instruction to raise its own score
- **THEN** the output contract is still enforced, the run is recorded, and no instruction in the
  document alters the family, the caller's permissions, the criteria used, or the output contract

### Requirement: Every claim carries a source label, an evidence reference, and a criterion

Each fitment claim, gap claim and contributing criterion SHALL carry a source label drawn from the
closed evidence-source vocabulary and at least one evidence reference identifying the record and
version it rests on. Output containing a claim without both SHALL be rejected.

*Source: `AI-007`, `G-02`, `BR-008`, `RANK-002`'s evidence-reference column and `RANK-003`'s
requirement that explanations list the criteria contributing to a recommendation;
`ai-platform-governance`'s `ai-platform/evidence-labeling` capability, whose vocabulary this
consumes and whose mixed-source rule requires per-claim rather than per-insight labelling.*

#### Scenario: Claim inspected

- **WHEN** a fitment or gap claim is inspected
- **THEN** it carries a source label and a resolvable versioned reference to the record it came from

#### Scenario: Claim with no evidence reference

- **WHEN** output contains a claim with no evidence reference
- **THEN** the output is rejected as a contract violation and recorded as a failed run

#### Scenario: Mixed-source insight

- **WHEN** an insight draws on a resume and on a prior interview note
- **THEN** each claim carries its own label and references rather than the insight carrying one

#### Scenario: Criteria explanation

- **WHEN** a score is presented
- **THEN** the criteria that contributed to it are listed, each with its own evidence

### Requirement: Gap output uses the four gap categories and states evidence status

Gap output SHALL classify each gap as missing, weak, unclear or conflicting, and SHALL state the
evidence status behind that classification.

*Source: §16.1's Gap Summary row — "Missing, weak, unclear, conflicting gaps" — and §16.3's
`gap_summary` contract requiring "gap category and evidence status." The four categories are what
keep "we found no evidence" from rendering identically to "we found evidence of absence," which is
the distinction `matching/ranking-insufficiency` exists to preserve.*

#### Scenario: Gap classified

- **WHEN** a gap is produced
- **THEN** it carries one of the four categories and an evidence status

#### Scenario: Category outside the set

- **WHEN** output carries a gap category outside the four
- **THEN** the output is rejected as a contract violation

### Requirement: The AI proposes a mandatory-criteria status and can never assert a blocking one

A ranking run MAY propose a mandatory-criteria status of meets, unclear or manager review required.
It SHALL NOT assert does not meet. The blocking value SHALL be settable only by a human, with a
mandatory reason, audited.

*Source: `G-01`'s adopted decision, stated in those terms — "the AI may output only
`meets | unclear | manager_review_required`. It may **never** assert `does_not_meet`; only a human
may set a blocking value, with a reason, audited. This keeps the advisory-only principle intact —
otherwise the AI would be gating a decision about a person"; §12.2's
`candidate_posting_applications.mandatory_criteria_status`; `AI-010`, `BR-007`. `design.md` D11
records the item split: this capability proposes, `matching/ranking-board` confirms.*

#### Scenario: AI proposes a status

- **WHEN** a ranking run produces a mandatory-criteria status
- **THEN** it is one of meets, unclear or manager review required, and it is marked as AI-proposed

#### Scenario: AI asserts a blocking value

- **WHEN** a ranking run's output carries does not meet
- **THEN** the output is rejected as a contract violation and the value is not persisted

#### Scenario: Persistence path to the blocking value

- **WHEN** a path from AI output to a stored does-not-meet value is sought
- **THEN** none exists

### Requirement: A candidate with no enrichment is ranked, not excluded and not penalised

Ranking SHALL proceed for a candidate whose resume was never enriched or whose enrichment failed,
using the evidence that exists. The absence of enrichment SHALL be represented as insufficiency
rather than as a low score, and SHALL NOT remove the candidate from the ranking version.

*Source: `D18`'s deliberate decoupling — "LLM enrichment fails → the candidate still exists, just
unenriched → **retryable**, no blocking" — and `candidate-intake`'s Risks section, which records that
"`matching-and-ranking` ranking an unenriched candidate needs a defined answer, and that is its
feature's to give." `G-03`, `AI-008`. `design.md` D6 is the answer.*

#### Scenario: Unenriched candidate ranked

- **WHEN** a posting is ranked and one candidate has no enrichment
- **THEN** that candidate appears in the ranking version with insufficiency recorded for what
  enrichment would have supplied

#### Scenario: Absence is not weakness

- **WHEN** an unenriched candidate's score is compared with a candidate whose evidence was found
  weak
- **THEN** the two are distinguishable, and the unenriched candidate is not scored low for the
  absence

#### Scenario: Silent exclusion

- **WHEN** the ranking version's entries are compared with the posting's linked applications
- **THEN** every application appears, with insufficiency or failure recorded where applicable, and
  none is silently omitted

### Requirement: Every free-text field in each family's contract carries a declared bound

Each free-text field in each of the three families' output contracts SHALL declare a
machine-checkable length bound. Output exceeding a bound SHALL be recorded as a failed run and SHALL
NOT be persisted as content. Bounds SHALL be held in audited configuration with machine-readable
provenance marking them provisional.

*Source: `AGENTS.md`'s standing product-quality bar; `S.5`, which names this feature's surface as
"ranking/fitment/gap — the highest-volume AI surface in the product";
`ai-platform-governance` `design.md` D7's three enforcement points, consumed rather than
reimplemented, and its D3 point 3 rule that a bound is declared **per free-text field**.*

#### Scenario: Over-long fitment summary

- **WHEN** the model returns a fitment summary exceeding its declared bound
- **THEN** the run is recorded as failed with the violation preserved and no content is persisted

#### Scenario: Field with no bound

- **WHEN** one of these families' contracts declares a free-text field with no bound
- **THEN** registration is refused

#### Scenario: Gap list bounded as a list

- **WHEN** a gap summary is produced
- **THEN** it is a bounded short list rather than prose, per its declared bound

#### Scenario: Bound changed

- **WHEN** an authorized administrator changes a bound
- **THEN** the change is an audited configuration change with no deployment

### Requirement: Batch ranking reports per-candidate success and failure

Batch ranking SHALL execute asynchronously with retrievable progress and SHALL report success and
failure per candidate. A partial failure SHALL NOT be reported as a whole-batch failure, and SHALL
NOT prevent the successful entries from forming a ranking version.

*Source: `ERR-007` — "partial failures in batch ranking shall report candidate-level success and
failure details"; §32's Ranking row and `NFR-002`'s asynchronous-with-progress rule;
`ai-platform-governance`'s partial-batch-failure scenario, which this is the first real consumer of.*

#### Scenario: Partial batch failure

- **WHEN** ranking succeeds for some candidates and fails for others
- **THEN** per-candidate outcomes are reported and the successful entries form a ranking version
  that records which candidates failed

#### Scenario: Progress retrievable

- **WHEN** a batch ranking is in flight
- **THEN** its progress and completion are retrievable by identifier

#### Scenario: Run reference on failure

- **WHEN** ranking fails for a candidate for any reason
- **THEN** a run reference is available for that candidate's failed run

### Requirement: Ranking criteria weights are configuration, and no approved values exist yet

Where ranking criteria weights are applied, they SHALL be held in audited configuration rather than
in code, and SHALL carry machine-readable provenance marking them provisional until product
approval is recorded.

*Source: §29 item 8 — "ranking criteria weights **where product approved**" — and the fact that no
such approval exists anywhere in this project's decision record. The same treatment
`ai-platform-governance` D7 gave conciseness bounds and `hiring-postings` gave its field bounds: a
wrong-but-labelled-and-enforced value is fixable configuration, an absent one is an unenforceable
contract.*

#### Scenario: Weights read from configuration

- **WHEN** ranking applies criteria weights
- **THEN** they are read from the audited configuration registry, not from constants in code

#### Scenario: Provenance visible

- **WHEN** the configured weights are read
- **THEN** they carry provenance marking them provisional and unapproved

#### Scenario: Weight changed

- **WHEN** an authorized administrator changes a weight
- **THEN** the change is audited with previous and new value and a mandatory reason, with no
  deployment

### Requirement: Ranking requires the Run AI action and degrades gracefully

Requesting ranking SHALL require the Run AI action on the ranking surface, distinct from view and
edit, evaluated by the central evaluator. With the model provider or the vector backend
unavailable, every non-AI action SHALL remain fully usable and existing ranking versions SHALL
remain readable.

*Source: §9.3's nine action flags and `C-02`'s note that a distinct `Run AI` action is what makes
triggering AI separately controllable from viewing its output; `AUTHZ-003`, `AUTHZ-004`; `G-12`,
`NFR-004`, `DEP-008`.*

#### Scenario: Actor without Run AI

- **WHEN** a user lacking the Run AI action views the ranking surface
- **THEN** the trigger control is absent, and a request submitted directly is refused server-side

#### Scenario: Background run under the service account

- **WHEN** the batch orchestrator runs under the AI Service Account
- **THEN** its permission is evaluated by the same evaluator on the same terms as a human caller

#### Scenario: Provider unavailable

- **WHEN** the model provider is unavailable
- **THEN** existing ranking versions and boards render, non-AI actions work, and the trigger control
  reports temporary unavailability

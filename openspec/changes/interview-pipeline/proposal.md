# Interview Pipeline — Shortlisting, the Interview Console and the Scorecard Center

## Why

`matching-and-ranking` produces a judgement about a candidate and then, deliberately, stops: its
board *"decides nothing about a candidate's progress"* and writes no `status`, no `shortlist_reason`
and no `final_outcome`. **This feature is where a human first acts on that judgement.** `TS-BL-056`
is the product's first disposition — the first place a person says yes or no about another person
inside TalentSphere — and everything after it, in this feature and in `decision-and-offers`, hangs
off that act.

**It is where the product's central promise stops being architectural and becomes procedural.**
Advisory-only has so far been enforced by absence: the gateway holds no transition capability, the
board writes no state field. Here the human decision exists as a real workflow transition with a
mandatory reason, and the AI draft it may agree or disagree with sits beside it. `project.md`'s rule
— *AI recommends, a human decides* — is satisfied structurally upstream; here it is satisfied by an
accountable person leaving a reasoned record.

**It also closes the AI capability matrix.** §16.1 lists ten capabilities and §16.3 registers ten
prompt families. Seven are claimed: `job_description_generation` and `job_posting_generation`
(`hiring-postings`), `resume_extraction` (`candidate-intake`), `candidate_ranking`,
`fitment_summary` and `gap_summary` (`matching-and-ranking`), and `resurfacing`
(`resurfacing-and-communications`' `TS-BL-075`, unproposed). **This feature claims the remaining
three** — `scorecard_generation`, which `D.10` already placed here, plus `interview_questions` and
`interview_note_summary`, which `D.10`'s dependency table left wired to no feature at all. See
`design.md` D1 for the gap and how it is closed.

**Two decisions this feature builds on were reversed, and one of the reversals had a loose end.**
`C-08` amends `D24`: interview notes **never lock** — versioning, forever, by permission-matrix rule
rather than author-only. `C-05` amends `D11`: one consolidated scorecard per Application, not one
per round. The loose end is that `C-08` removed the lock without removing the goal the lock existed
for — that an approved evaluation and its evidence agree — and `exploration-notes.md` tracked the
resulting **scorecard drift** as an open question rather than deciding it. This feature decides it
(`design.md` D3) rather than inheriting a gap by silence.

**Nothing here is built, and nothing was inherited.** `talentsphere-wave-1-foundation` has no
`interview/` delta spec — its fourteen cover `access-control/`, `ai-platform/`, `design-system/`,
`identity/` and `platform/`. Its `design.md` D1–D18 are fully allocated across the five Phase-1
features (`platform-core` D11 records the split) and **none landed here.** `sprint-0-outcome.md`
records 13 platform-layer tasks with *"no hiring feature exist[ing] yet, by design,"* and
`KNOWN_ISSUES.md` carries no entry about shortlisting, interviews, notes or scorecards. Checked
rather than assumed — see `design.md` D13.

## What Changes

Eight backlog items, `TS-BL-056` through `TS-BL-063`, from
[D.10](../talentsphere/exploration-notes.md#d10--phases-2-5-backlog-decomposition-and-the-feature-list-is-now-finalized-2026-08-25).
None is built.

- **`TS-BL-056` Shortlisting decision UI, reason required on every disposition.** `SHL-001`–`SHL-005`
  — shortlist with a reason, remove with a reason preserving history, and the scheduling task that
  `SHL-003` says shortlisting creates. This item **registers the Application state machine** with
  `platform-core`'s workflow service (which ships with none) for §11.2's shortlist-and-disposition
  transitions, and it is the first writer of `candidate_posting_applications.status` and
  `shortlist_reason` — columns `candidate-intake`'s `TS-BL-046` created and deliberately left
  unwritten. **"Reason required on every disposition" is broader than `SHL-002`'s shortlist-only
  requirement and traces to the "Recommendations made, not yet formally decided" table; it is
  treated as adopted, and `design.md` D2 says so explicitly rather than citing it as settled.**
  Depends on `matching-and-ranking`'s `TS-BL-053` and `platform-core`'s `TS-BL-004`.
- **`TS-BL-057` Interview round scheduling.** `INT-001`'s multiple rounds per Application and
  `INT-002`'s round record (§12.2 `interview_rounds`) — type, interviewer, scheduled window, status,
  question-set link. `SHL-005`'s block on scheduling a non-shortlisted candidate, with an authorized
  recorded override. Assignment is what makes `INT-003`'s assignment scope evaluable, so this item is
  what `matching-and-ranking`'s `TS-BL-054` projection has been scoped against all along. No calendar
  integration — `D06`/`C-04` defer it to `resurfacing-and-communications`' `TS-BL-079`, and panelists
  receive an assignment notification with no calendar entry, which `C-04` records as an accepted gap.
  Depends on `TS-BL-056`.
- **`TS-BL-058` Interview Console, structured notes, multiple rounds.** §14.2's Interview Console —
  assigned interviews only (`INT-003`), `INT-006`'s eight fixed note fields, `INT-007`'s stored
  forms, and the 5-point `strong_yes`…`strong_no` recommendation. It **consumes
  `matching-and-ranking`'s `TS-BL-054` projection** rather than assembling its own view of ranking
  output — gaps yes, score no, per `C-10`. It **authors `interview_questions`**, the first of this
  feature's three prompt families, triggered at interview setup and rendered here per `INT-004`
  (`design.md` D1). It **registers `interview_note_section`** against `matching-and-ranking`'s
  source-type seam, which `TS-BL-049` built expecting exactly this. It also **answers
  `design-system`'s open question** about which screen uses the presentation shell (`design.md` D5).
  Depends on `TS-BL-057` and — **an edge `D.10` does not carry** —
  `ai-platform-governance`'s `TS-BL-027`.
- **`TS-BL-059` Interview note versioning without lock (`C-08`/`D24`).** Built against **`C-08`, not
  `D24`**: `INT-008`/`BR-017` versioning with **no freeze on scorecard approval** and **no
  author-only restriction** — who may edit is the `Edit` action on the Interview Console page,
  seeded to Interviewer only. It **authors `interview_note_summary`**, the second family, gated by
  `INT-009` on at least one submitted note (`design.md` D1). It emits the note-version event that
  `TS-BL-061`'s supersede mechanism consumes. `D24`'s own text still claims it *"satisfies the D11
  requirement that notes lock on scorecard approval"*; that line and D11's matching consequence are
  both stale, and the un-annotated half was corrected in `exploration-notes.md` during this
  conversation (`design.md` D14). Depends on `TS-BL-058` and — **a second edge `D.10` does not
  carry** — `ai-platform-governance`'s `TS-BL-027`.
- **`TS-BL-060` Scorecard Center, AI-drafted generation.** §14.2's Scorecard Center listing
  **interviewed candidates only** (`SCR-002`, `BR-011`), with generation refused for a candidate
  with no completed round (`SCR-001`, `AI-009`, `BR-010`). Authors `scorecard_generation`, the third
  family, through the gateway on references — never note or resume content. `SCR-004`'s separation
  of resume-derived from interview-derived evidence and `SCR-005`'s per-dimension source and
  confidence are output-contract clauses, not display choices. **It records which note versions the
  draft was generated from**, which is `D24`'s one surviving consequence and the precondition for
  drift detection. Depends on `TS-BL-058` and `ai-platform-governance`'s `TS-BL-027`.
- **`TS-BL-061` Scorecard human-approval workflow.** `SCR-006`'s Draft-until-approved gate,
  `SCR-007`/`BR-016`'s mandatory reason on any human adjustment, `SCR-008`'s versioning and
  auditability. **This item builds the supersede-and-re-approve mechanism** that `C-08`'s reversal
  made necessary: a note version created after approval sets `scorecards.status = superseded` and
  requires re-approval. `exploration-notes.md` recorded that fix as recommended and **"not yet
  decided"**; it is decided here — see `design.md` D3 for the decision and the alternatives that
  were rejected, and that entry now points back to it. Depends on `TS-BL-060`.
- **`TS-BL-062` Consolidated scorecard across rounds (`C-05`/`D11`).** One scorecard per Application
  (`scorecards.candidate_application_id`), consolidated across every round, with per-panelist
  attribution preserved in the notes it draws from rather than averaged away — `C-05`'s explicit
  correction of the mischaracterisation `D11` rejected consolidation on. Carries `D11`'s surviving
  open item: the rule for consolidating **conflicting** panelist recommendations, which is answered
  as *surfaced, never resolved arithmetically* (`design.md` D4). Registers `scorecard_summary`, the
  second of `matching-and-ranking`'s two deferred vector source types. Depends on `TS-BL-061`.
- **`TS-BL-063` Fixed scorecard dimensions/competencies (`C-06`/`D15`).** `SCR-003`'s eight fixed
  dimensions and the 5-point scale, **global rather than JD-scoped** — `C-06` explicitly superseded
  `D15`'s "competencies become part of the JD version," so editing a JD never touches scorecard
  structure. `OD-007` means the set is unapproved **even in the organisation that wrote it**, so it
  ships as a **seeded default** in `platform-core`'s audited runtime-configuration registry, not as
  a hard-coded truth. `C-06` said "configurable per §29 #14"; §29 #14 is *role and permission matrix
  values* and §29 has no scorecard-dimension entry — corrected in `exploration-notes.md` during this
  conversation (`design.md` D14). Depends on `TS-BL-062`.

**Explicitly not in this change:**

- **No priority selection, no offers, no closure.** `SEL-001`–`SEL-005`, `OFF-001`–`OFF-005`,
  `BR-012`–`BR-014` and `D20`'s five-slots-per-vacancy mechanics are `decision-and-offers`'
  `TS-BL-064`–`TS-BL-070`, which depend on `TS-BL-062`. This feature carries an Application to
  `ScorecardReady` (§11.2) and stops.
- **No carry-forward.** `D17`/`D21`/`C-07`'s priority-lane scorecard reuse is `TS-BL-066`
  (`decision-and-offers`). `domain-model.md` records that a `Scorecard` can link to multiple
  Applications when carry-forward applies; this feature builds the one-per-Application shape `C-05`
  specifies and does not build the multi-link path. `TS-BL-062` records the constraint the carrier
  will need — a consolidated document is far simpler to carry than N per-round ones, which was one
  of `C-05`'s two stated reasons for reversing `D11`.
- **No calendar integration and no candidate-facing communication.** `D06`/`C-04` defer both;
  `TS-BL-079` and `TS-BL-078` own them, and `OD-005` blocks the second on legal sign-off. Panelist
  assignment notifications go through `platform-core`'s internal delivery (`TS-BL-005`), which
  `D04` kept in early scope.
- **No ranking, no re-rank, and no ranking board.** This feature reads ranking output through
  `TS-BL-054`'s projection and `TS-BL-053`'s board; it computes nothing and triggers no re-rank.
  `matching-and-ranking` D5's trigger table is authoritative and a disposition is not among its
  triggers.
- **No new AI gateway, prompt registry, run log, disclosure record, evidence-source vocabulary,
  evaluation harness, permission evaluator, audit writer, dispatch substrate, workflow framework,
  or design-system component.** Eleven substrates are consumed. The three new prompt families are
  **new template versions of already-registered families**, not registry additions — `TS-BL-030`
  registered all ten.
- **No semantic search over interview history.** `C-01`'s scope note records it as a genuine
  capability *not among the original 19 features*. This feature registers the two source types that
  make it reachable and builds no search surface (`design.md` D6).
- **No bias auditing and no scorecard weighting.** `PRV-006` keeps the first out of scope. `OD-007`
  covers dimensions **and weightings**; no weighting is approved, so `TS-BL-063` ships dimensions
  with no weights rather than inventing them — the same treatment `matching-and-ranking` gave §29
  item 8.
- **No application-merge resolution.** `D23a`'s follow-on rule — which of two applications on one
  posting survives a candidate merge — was deferred to this feature twice, by `candidate-intake` and
  by `matching-and-ranking`. It is **partly** answerable here and is answered only that far
  (`design.md` D7).

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is new.
None is preserved from `talentsphere-wave-1-foundation`: that change has no `interview/` delta spec
at all.

- `interview/shortlisting`: the disposition surface — reason required on every disposition from a
  coded list, the Application state machine registration, and the scheduling task shortlisting
  creates. *(`TS-BL-056`)*
- `interview/round-scheduling`: rounds, panelist assignment, the shortlist precondition and its
  audited override, and the assignment scope every downstream interview surface is evaluated
  against. *(`TS-BL-057`)*
- `interview/interview-console`: the assigned-interviews surface — `INT-006`'s fixed note fields,
  the consumed ranking-context projection, and AI-suggested questions. *(`TS-BL-058`)*
- `interview/note-versioning`: `C-08`'s model — every edit a new version, forever, no lock, editable
  by matrix rule — plus the AI interview summary generated from submitted notes. *(`TS-BL-059`)*
- `interview/scorecard-generation`: interviewed-candidates-only eligibility, the drafted scorecard
  with separated evidence, and the note versions the draft was drawn from. *(`TS-BL-060`)*
- `interview/scorecard-approval`: the Draft-until-approved gate, mandatory-reason adjustment, and
  the supersede-on-post-approval-note-edit mechanism that keeps an approved evaluation honest
  without a lock. *(`TS-BL-061`)*
- `interview/scorecard-consolidation`: one scorecard per Application across all rounds, panelist
  attribution preserved, disagreement surfaced rather than averaged. *(`TS-BL-062`)*
- `interview/scorecard-dimensions`: `SCR-003`'s eight fixed, global dimensions and the 5-point
  scale, seeded as audited configuration because `OD-007` leaves them unapproved. *(`TS-BL-063`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**No overlap to resolve.** All five Phase-1 features have drawn their inheritance across from
`talentsphere-wave-1-foundation`, which `ai-platform-governance` recorded as complete with "no
unaccounted content left." Nothing under `interview/` was ever written there. See `design.md` D13.

**Two corrections were made to `exploration-notes.md` during this conversation**, per `AGENTS.md`'s
convention for genuine gaps in the documents every feature reads, each at the point of the error and
each quoting the prior wording: `D11`'s consequence *"Interview notes must **lock on scorecard
approval**"* carried no supersession marker eleven days after `C-08` reversed the mechanism, because
`C-08` was written as an amendment to `D24` and D11's index row points only at `C-05`, which resolves
D11's structure half and is silent on locking. And `C-06`'s *"configurable per §29 #14"* cites *role
and permission matrix values*; §29 has no scorecard-dimension item at all. See `design.md` D14.

## Impact

**New application code** — `backend/app/interview/` containing the Application state machine
registration and its disposition rules, the round aggregate and its assignment scope, the note
aggregate with its version chain, and the scorecard aggregate with its draft/approve/supersede
lifecycle and its three prompt payload builders.
`backend/app/api/routes/interview/` for §13.2's shortlisting, interview and scorecard endpoints:
`POST /api/applications/{applicationId}/shortlist`, `.../remove-shortlist`, `.../interviews`,
`GET /api/interviews/assigned`, `GET /api/interviews/{interviewId}`,
`POST /api/interviews/{interviewId}/notes`, `PATCH /api/interviews/{interviewId}/notes/{noteId}`,
`POST /api/interviews/{interviewId}/complete`, `POST /api/applications/{applicationId}/ai/questions`,
`GET /api/postings/{postingId}/scorecards/eligible`,
`POST /api/applications/{applicationId}/scorecards/generate`,
`POST /api/scorecards/{scorecardId}/approve`, and `PATCH /api/scorecards/{scorecardId}`.
Frontend: the **Shortlist View**, the **Interview Console** and the **Scorecard Center** (§14.2),
assembled from `design-system`'s dense data table, page templates, forms, AI-disclosure marking and
evidence-source labels — **adding no new component**, and consuming the previously unused
**presentation shell** (`design.md` D5).

**New tables** — `interview_rounds`, `interview_notes` and `scorecards` (§12.2), all by reversible
migration. Three existing `candidate_posting_applications` columns are written here for the first
time: `status`, `shortlist_reason` and — on the negative dispositions this feature owns —
`final_outcome`, all three created by `candidate-intake`'s `TS-BL-046` and deliberately left
unwritten for this feature and `decision-and-offers`.

**The first domain state machine registered with the workflow service.** `platform-core`'s
`platform/workflow-engine` spec ships the framework *"with no posting, Application, offer, or
closure state machine defined"* and names this feature as one of two that would register one.
§11.2's Application lifecycle from `Shortlisted` through `ScorecardReady`, plus its `NotSelected`
and `Withdrawn` exits, is declared here — states, permitted transitions, per-transition permission,
and `WF-005`'s reason-required marking on every negative and terminal transition.

**It also invokes three transitions on a second, distinct machine it does not register.**
§11.1's posting lifecycle is `hiring-postings`' `TS-BL-038`, which declares all thirteen states while
**nothing invoked the six advances between `open` and `onboarding`** — a gap `decision-and-offers`
found during its verification and disclosed in its `design.md` D9a. Three of those advances are
triggered by this feature's events and are built here: `open → screening` on the first shortlist
(`TS-BL-056`), `screening → interviewing` on the first round scheduled (`TS-BL-057`), and
`interviewing → scorecard_review` on the first scorecard approved (`TS-BL-061`). Each is monotone,
idempotent and attributed to the human whose action triggered it, inside that action's transaction.
`job_postings.status` and §11.2's Application states are separate and must not be conflated
(`design.md` D10a).

**Three prompt families move from stub to authored** — `interview_questions`,
`interview_note_summary` and `scorecard_generation`, three of the ten `TS-BL-030` registered with
stub text on purpose. Each becomes a **new template version** through the registry's versioning and
promotion gate, which refuses production activation with no passing corpus run recorded — so all
three are Dev-deployable and production-blocked until `TS-BL-032`'s harness carries their cases
(`ai-platform-governance` D8). `AI-015` names *conflicting interview notes* as an adversarial class,
which this feature is the first to be able to construct for real.

**Three new background job types** registered against `platform-core`'s dispatch pattern, matching
§25's rows exactly: **Question Generation** (trigger: interview setup), **Interview Summary**
(trigger: interview note submitted), and **Scorecard Generation** (trigger: eligible candidate
selected). No job type is registered per gateway call — `ai-platform-governance` D10 already
dispatches those.

**Two vector source types registered, not built** — `interview_note_section` and `scorecard_summary`,
the two of `VEC-001`'s four that `matching-and-ranking`'s `TS-BL-049` deliberately did not embed
because their records did not exist. Its D2 built the pipeline source-type-driven with all four enum
values present from the first migration specifically so this feature adds **a registration and no
schema change**. Whether that seam holds is an open question `matching-and-ranking` recorded and
this feature answers by using it.

**Existing capability consumed, not modified** — `platform-core`'s workflow framework, dispatch
pattern, notification delivery and audited runtime configuration; `access-control-and-admin`'s
permission evaluator, its `assigned-postings` and assignment-scope predicates, its query-time filter
contract and its durable audit writer; `ai-platform-governance`'s gateway, prompt registry, run log,
disclosure record, override record, evidence-source vocabulary, advisory-only guarantee and
evaluation harness; `matching-and-ranking`'s `TS-BL-054` projection, its board, its insufficiency
vocabulary and its vector source-type seam; `candidate-intake`'s Application record and resume
versions; `hiring-postings`' pinned `jd_version`; `design-system`'s table, forms, shell states,
disclosure marking and label components.

**Three questions handed to this feature by name are answered rather than deferred again.**
`matching-and-ranking` D5 and `candidate-intake` D1(b) both recorded that `D23a`'s
which-application-survives rule *"depends on the application stage, which is `interview-pipeline`'s"*
— answered as far as this feature's stages reach, and no further (`design.md` D7).
`design-system`'s Open Questions asks which screen, if any, uses the presentation shell — answered
(`design.md` D5). `matching-and-ranking`'s Open Questions asks whether the Phase-3 source types need
any change to its registration seam — answered by registering them (`design.md` D6).

**Downstream features that block on this one** — `decision-and-offers`' `TS-BL-064` (on
`TS-BL-062`) and therefore its entire chain through offers, onboarding and closure;
`resurfacing-and-communications`' `TS-BL-079` (on `TS-BL-057`) and, indirectly, `TS-BL-076`'s
priority lane, whose carried evidence is this feature's consolidated scorecard;
`insight-and-reporting`'s AI-agreement-rate dashboard, whose second half — *"how often approved
scorecards diverge from AI drafts"* — is computable only from the override records `TS-BL-061`
writes.

**External constraints carried, not solved here** — **`OD-007`** leaves the scorecard dimensions and
weightings unapproved by the business, so `SCR-003` ships seeded and configurable and no weighting
ships at all. **`innovation practice relevance` is Miracle-Labs-specific** and `C-06` asks that it be
confirmed or renamed; it is seeded as specified with the question recorded, because renaming a
dimension in seeded configuration is cheaper than guessing. **`OD-003`** (the AI model provider) is
open, so all three families run on the deterministic stub in Dev. **`OD-004`**'s retention period
governs how long superseded scorecard versions and note versions are kept, which `SCR-008` and
`INT-008` require be preserved. **`OD-005`** keeps candidate-facing communication out of scope, so a
scheduled interview reaches the candidate by a manual channel outside the system — `C-04`'s recorded
accepted gap, inherited rather than introduced here.

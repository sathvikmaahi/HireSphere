# Decision & Offers — Priority Selection, the Offer Tracker and Posting Closure

## Why

`interview-pipeline` carries an Application to `ScorecardReady` and stops. **This feature is where
the product's central promise is finally cashed: a human picks a person.** Everything before it
produces evidence — a ranking, a shortlist, notes, an approved consolidated scorecard. `TS-BL-065`
is where a Practice Manager, looking at that evidence, says *this one, first*. And unlike every
decision surface before it, **this one has no AI touchpoint at all.**

That is not an omission. `interview-pipeline` closed the ten-family AI capability matrix — all ten
prompt families are owned, with `resurfacing` (`TS-BL-075`) the only one still unproposed. **No
family is authored here, and none is invoked.** So the most consequential decision in the product —
which of five candidates gets an offer — runs on human judgement over AI-marked evidence, with the
AI contributing nothing at the moment of choice. `project.md`'s rule that *AI recommends, a human
decides* has been satisfied structurally upstream by absence of capability; here it is satisfied by
there being nothing to advise with.

**It is also where the pipeline finally terminates.** `hiring-postings` registered §11.1's posting
machine with **no automatic transition at all** — `filled` and `closed` are reachable states nothing
drives into. Its `design.md` D5 is explicit that this was deliberate, splitting the remaining work
along exactly this feature's item boundary: finite fulfilment closure to `TS-BL-069`, evergreen's
manual-only rule to `TS-BL-070`. This feature is the other half of a seam an earlier feature built
on purpose and left open.

**One decision this feature is named after was superseded, and its supersession is easy to miss.**
The *"Closure semantics"* recommendation — a `Recommendations made, not yet formally decided` entry,
the same status class `candidate-intake` handled for `D23a` and `interview-pipeline` handled for
"reason on every disposition" — proposes auto-close on `offers_accepted == vacancy_count`, an
intermediate `FILLING` state, an audited reopen path, and a four-way closure cascade. `G-13` then
**superseded its trigger in so many words**: *"This supersedes the earlier working assumption of
closing on `offers_accepted == vacancy_count`."* The recommendation's other three parts were not
superseded and are built here. `design.md` D1 names the status class explicitly rather than citing
the recommendation as settled, and separates what survives from what `G-13` replaced.

**Two of its four cascade consequences reach past this feature's edge**, and one reaches into a
feature that does not exist yet. `design.md` D8 publishes the event a future
`resurfacing-and-communications` (`TS-BL-075`) subscribes to and registers no handler — the fourth
time this shape has come up, after `hiring-postings`' unregistered `posting.opened` and
`ai-platform-governance`'s pre-designed envelope.

**Nothing here is built, and nothing was inherited.** `talentsphere-wave-1-foundation` has no
`decision/` delta spec — its fourteen cover `access-control/`, `ai-platform/`, `design-system/`,
`identity/` and `platform/` — and its `design.md` D1–D18 are fully allocated across the five Phase-1
features, none landing here (`platform-core` D11 records the split). `sprint-0-outcome.md` records
13 platform-layer tasks with *"no hiring feature exist[ing] yet, by design."* `KNOWN_ISSUES.md`
carries no entry about selection, offers, onboarding or closure. Checked rather than assumed — see
`design.md` D12.

## What Changes

Seven backlog items, `TS-BL-064` through `TS-BL-070`, from
[D.10](../talentsphere/exploration-notes.md#d10--phases-2-5-backlog-decomposition-and-the-feature-list-is-now-finalized-2026-08-25).
None is built.

- **`TS-BL-064` Priority Slot mechanics (`D20`: up to 5 × vacancy).** `SEL-001`–`SEL-003`,
  `BR-012` — `5 × n` ranked selections per finite posting, rank unique within the slate,
  displacement and insertion re-sequencing ranks as an **audited reorder, not a silent renumber**.
  This item creates the `vacancy_slots` rows `hiring-postings` deliberately did not
  (*"Do not create `vacancy_slots` — slots are reserved and filled by `TS-BL-064` and the offer
  path"*) and **registers the remainder of §11.2's Application machine** — `PrioritySelected`
  onward, which `interview-pipeline` declared reachable and registered no transitions for.
  **`D20`'s two names for "slot" are two different tables** and `design.md` D2 separates them.
  **`D20`'s vacancy-reduction consequence is unreachable as written** — `hiring-postings` made
  `vacancy_count` immutable once a posting opens, closing `D14`'s own open edge case; the eviction
  rule attaches instead to vacancy-slot cancellation, which is the mechanism that survives
  (`design.md` D3). Evergreen carries **no slots, no cap and no rank** (`D14`). It also drives the
  first of three **posting-state advances** this feature owns — `scorecard_review → selection` on the
  first committed selection (`design.md` D9a). Depends on `interview-pipeline`'s `TS-BL-062`.
- **`TS-BL-065` Priority Selection UI (Floating Action Panel, `S.1`).** §14.2's Priority Selection
  Slate, built on `design-system`'s **already-shipped Floating Action Panel** — whose own spec names
  this as its first consumer and states the panel *"owns no task state"*, so the 5×n semantics live
  here and the component is consumed unchanged. `SEL-004`'s four tags, `SEL-005`'s retained reason.
  **`D20`'s tiered-ranking UX concern is answered without a new mechanism** — `SEL-004`'s tags *are*
  the tiering, and `SEL-002`'s unique rank stays (`design.md` D4). Two presentations, ranked slots
  and evergreen's flat list, per `D14`. Depends on `TS-BL-064` and `design-system`'s `TS-BL-010`.
- **`TS-BL-066` Carry-forward eligibility & scorecard reuse (`D17`/`D21`/`C-07`).** Built against
  **`C-07`'s resolution, not `D17` as originally written**: the carried consolidated scorecard is
  cited as **prior evidence** and the candidate still completes **one confirmatory round** for the
  new posting, which is what dissolves the conflict with `SCR-001`/`BR-010`/`BR-011`/`AI-009` rather
  than overriding it. `D17`'s non-round-1 entry point is **dropped** — `C-07` says so explicitly.
  Builds the multi-Application scorecard link `interview-pipeline` deliberately did not
  (*"none exists"*), consuming its consolidated scorecard's citable-from-outside contract as
  written. `D21`'s two paths, `C-11`'s configurable 90-day TTL, `G-01`'s re-evaluation of mandatory
  criteria regardless of carried evidence, and `OA-01`'s route instrumentation, which is *"a slice 7
  reporting requirement, not something to add later."* Depends on `TS-BL-065`.
- **`TS-BL-067` Offer creation & tracking.** `OFF-001`, `OFF-002`, `OFF-008`, `G-14` — the
  `offer_onboarding_records` tracker with salary, offer, joining and onboarding **statuses and no
  amounts** (`D19`), `G-14`'s `joining_status` and required `blocker_reason`, and a reason on every
  terminal or negative update. **`OFF-001`'s "salary finalization" and "candidate communication
  status" are status fields, not capabilities** — `D19` rules out compensation data and `C-04`/`OD-005`
  rule out candidate-facing communication (`design.md` D5). It drives the other two **posting-state
  advances** this feature owns — `selection → offer` on the first offer extended and
  `offer → onboarding` on the first accepted (`design.md` D9a). Depends on `TS-BL-065`.
- **`TS-BL-068` Onboarding record & Hubble ID capture (`D08`).** `OFF-003`–`OFF-007`, `BR-019` —
  Hubble ID entered manually after onboarding completion, unique **among recruited candidates**,
  duplicates blocked except through a controlled administrative correction. **The two-Hubble-identifier
  split is consumed from `identity-and-access` D3, not re-derived**: separate table, separate column
  name, **no foreign key and no shared uniqueness constraint** with `users.hubble_user_id`.
  `OD-002` is still open, so validation runs behind a port with a stub — and **closure does not gate
  on `hubble_id_validated`**, which would make closure unreachable while `OD-002` stays open
  (`design.md` D6). Depends on `TS-BL-067`.
- **`TS-BL-069` Closure rules (onboarded + valid Hubble ID, `G-13`).** `CLS-001`–`CLS-004`,
  `BR-013`, `BR-019` — the closure checklist, the fulfilment test, and the cascade. **Closure is a
  human action that becomes *available* when the condition holds, not a rule that fires on its
  own** — which is what `CLS-002`/`CLS-003`'s *"shall be disabled when"* and the `close-filled`
  endpoint actually describe, and it means this feature **adds a transition with an actor rather
  than contradicting `hiring-postings`' registered no-automatic-transition assertion**
  (`design.md` D7). The recommendation's `FILLING` state is **not built** — §11.1's `Onboarding`
  state already is it, and `G-13` says so. **One transition §11.1 does not define is registered
  here**: `filled → open`, the audited reopen path `G-13` requires for a renege discovered after a
  posting reaches `Filled`. The cascade's fourth consequence publishes an event and registers no
  handler (`design.md` D8). Depends on `TS-BL-068` and — **an edge `D.10` does not carry** —
  `hiring-postings`' `TS-BL-038`.
- **`TS-BL-070` Evergreen posting lifecycle rules (`D14`).** `BR-014`, `CLS-005` — pause, reopen and
  manual close with reason, and **no fulfilment closure logic reachable from an evergreen posting**.
  Per `hiring-postings` D5 this item *"needs to add nothing at all, exactly as `D14` predicted"* to
  the machine; what it adds is the **assertions that make that true** and the two consequences `D14`
  leaves as reporting and hygiene concerns: `D14`'s recommended **240-day zero-hire staleness
  nudge**, seeded as audited configuration, and the **per-hire fill record** that makes time-to-fill
  computable for a posting that never closes. Depends on `TS-BL-069` and `hiring-postings`'
  `TS-BL-038`.

**Explicitly not in this change:**

- **No resurfacing pool and no priority lane.** `TS-BL-069` writes the `selected_not_offered` tag
  and the `SelectedNotOffered` state (`SEL-004`, `SEL-005`, §11.2) and publishes the closure event;
  `resurfacing-and-communications`' `TS-BL-075` builds the `MatchSuggestion` pool and `TS-BL-076`
  the lane precedence — which depends on `TS-BL-064` for exactly the tag written here.
  `TS-BL-066` builds carry-forward as a mechanism that operates on a selected-not-offered
  Application **however it was surfaced**; how it gets surfaced is `TS-BL-076`'s (`design.md` D8).
- **No AI, of any kind.** No prompt family is authored or invoked, no `AIRun` is produced, no
  gateway call is made, no evidence label is generated. The ten families are closed elsewhere.
- **No offer letter, e-signature, compensation amount or approval chain.** `D19` and the reference
  schema agree — §12.2's `offer_onboarding_records` contains no salary amount, only a
  `salary_status` enum.
- **No candidate-facing communication.** `OD-005` blocks it and `C-04` defers it to `TS-BL-078`. An
  offer reaches a candidate by a manual channel outside the system; the tracker records **that** it
  did, not the message. `C-04`'s recorded accepted gap, inherited rather than introduced here.
- **No new posting state machine, workflow engine, dispatch substrate, notification delivery,
  permission evaluator, audit writer, audited configuration registry, or design-system component.**
  Eight substrates are consumed. Transitions are **added** to machines other features registered,
  and `design.md` D7, D9 and D9a record why each is an addition rather than a second machine.
- **No driver for §11.1's first three posting advances.** `open → screening`,
  `screening → interviewing` and `interviewing → scorecard_review` are triggered by
  `interview-pipeline`'s events, which this feature cannot observe. They are flagged for an
  `/opsx:update` run against `interview-pipeline` rather than built or polled for
  (`design.md` D9a, Open Questions).
- **No dashboard, report or export.** `insight-and-reporting`'s `TS-BL-071`/`073` depend on
  `TS-BL-069`; the closure-readiness *view* is theirs. `CLS-004`'s closure checklist is a
  precondition surface on the posting, not a reporting surface, and `design.md` D7 draws that line.
- **No re-ranking on selection.** `matching-and-ranking` D5's trigger table is authoritative and
  neither a selection nor an offer is among its triggers.
- **No change to `interview-pipeline`'s refusal on merges past `ScorecardReady`.** Its D7 refused
  those merges and said *"`decision-and-offers` can replace the refusal with a rule; it cannot
  un-merge a merge this feature performed wrongly."* This feature **keeps the refusal** and records
  why replacing it is not this feature's call to make on the evidence available (`design.md` D10).

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is new.
None is preserved from `talentsphere-wave-1-foundation`: that change has no `decision/` delta spec.

- `decision/priority-slots`: the `5 × n` cap arithmetic, ranked selections with audited
  re-sequencing, vacancy slots and their `open → reserved → filled` machine, the evergreen variant
  with no cap, and the remainder of §11.2's Application machine. *(`TS-BL-064`)*
- `decision/priority-selection`: the Priority Selection Slate on the consumed Floating Action Panel
  — tags, reasons, both presentations, and the selection commit. *(`TS-BL-065`)*
- `decision/carry-forward`: the multi-Application scorecard link, the same-JD and attested paths,
  the mandatory confirmatory round, the configurable TTL, and route instrumentation. *(`TS-BL-066`)*
- `decision/offer-tracking`: the offer and onboarding record — four status tracks, no amounts,
  joining status with a required blocker reason, and a reason on every negative update.
  *(`TS-BL-067`)*
- `decision/onboarding-hubble-id`: manual Hubble ID capture, uniqueness among recruited candidates,
  the administrative correction path, and validation behind a port `OD-002` has not specified.
  *(`TS-BL-068`)*
- `decision/posting-closure`: the fulfilment test, `CLS-004`'s checklist, closure as an available
  action rather than an automatic transition, the cascade, and the audited reopen path.
  *(`TS-BL-069`)*
- `decision/evergreen-lifecycle`: pause/reopen/manual-close with reason, the assertions that keep
  fulfilment logic unreachable from an evergreen posting, the staleness nudge, and the per-hire fill
  record. *(`TS-BL-070`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**No overlap to resolve.** All five Phase-1 features have drawn their inheritance across from
`talentsphere-wave-1-foundation`, which `ai-platform-governance` recorded as complete with *"no
unaccounted content left."* Nothing under `decision/` was ever written there. See `design.md` D12.

**Additions are made to state machines registered by other features**, and none is a modification to
those features' specs: the remainder of §11.2's Application transitions (`TS-BL-064`), which
`interview-pipeline` declared reachable and left unregistered on purpose; the posting machine's
`filled → open` reopen edge plus its actor-initiated fulfilment close (`TS-BL-069`), which
`hiring-postings` D5 left to this feature by name; and **drivers for three of §11.1's six
undriven posting advances** — `scorecard_review → selection` (`TS-BL-064`), `selection → offer` and
`offer → onboarding` (`TS-BL-067`). `design.md` D7, D9 and D9a record why each is an addition to a
declaration rather than a competing machine.

**One defect was found during verification of this change, in the same class as `D20`'s.**
`hiring-postings` `TS-BL-038` declares §11.1's transition set whole — all thirteen states, and
cancellation reachable *from* the six pipeline states — but its implementing tasks provide a caller
only for the entry, pause/reopen and terminal transitions, and no other feature writes posting state
at all. So `onboarding` was declared and undriven, leaving `TS-BL-069`'s central transition guarding
a state nothing reached. **The three advances triggered by this feature's own writes are now built
here; the three triggered by `interview-pipeline`'s events are flagged for an `/opsx:update` run
against it**, with the item and trigger named for each. `design.md` D9a records the investigation,
including the finding that this is *not* a citation mixup between §11.1 and §11.2 — the two machines
use deliberately different state names, and `job_postings.status` is its own stored enum.

**No correction was needed to any shared document.** `exploration-notes.md`, `domain-model.md`,
`glossary.md` and `project.md` were each checked against this feature's decisions under `AGENTS.md`'s
shared-document convention. Two genuine tensions were found and **neither lives in a shared
document**: `D20`'s vacancy-reduction consequence is superseded by a rule `hiring-postings` decided
in its own `design.md` (`design.md` D3), and the *"Closure semantics"* recommendation's trigger was
already superseded by `G-13` **with the supersession stated in `G-13`'s own text** — so there is
nothing unannotated to annotate. Recorded here so the absence of a correction reads as checked
rather than skipped.

## Impact

**New application code** — `backend/app/decision/` containing the selection aggregate with its cap
arithmetic and rank re-sequencing, the vacancy-slot aggregate and its three-state machine, the
carry-forward link and its attestation rules, the offer/onboarding aggregate with four independent
status tracks, the Hubble ID capture path with its uniqueness rule and validation port, and the
closure evaluator with its checklist and cascade.
`backend/app/api/routes/decision/` for §13.2's endpoints:
`POST /api/postings/{postingId}/priority-selections`,
`PATCH /api/priority-selections/{selectionId}`,
`GET /api/postings/{postingId}/offer-tracker`,
`PATCH /api/offer-records/{recordId}`,
`POST /api/offer-records/{recordId}/complete-onboarding`,
`POST /api/offer-records/{recordId}/hubble-id`,
`GET /api/postings/{postingId}/closure-checklist`, and
`POST /api/postings/{postingId}/close-filled`. **`POST /api/job-postings/{id}/close` already belongs
to `hiring-postings`' lifecycle**; the two closure routes are not duplicates and `design.md` D9
records which is which. Frontend: the **Priority Selection Slate** and the **Offer and Onboarding
Tracker** (§14.2), assembled from `design-system`'s Floating Action Panel, dense data table, page
templates and forms — **adding no new component**.

**Three new tables** — `priority_selections`, `vacancy_slots` and `offer_onboarding_records`
(§12.2), all by reversible migration, plus a scorecard-to-Application link table §12.2 does not
carry because the reference spec has no carry-forward mechanism at all (`C-07`: *"This is us
extending beyond the spec"*). Two existing `candidate_posting_applications` columns are written here
for the first time: `status` on §11.2's `PrioritySelected`-onward transitions, and `final_outcome`
on the positive terminal outcome — the column `candidate-intake` created and left unwritten for
`interview-pipeline`'s negative dispositions and this feature's.

**No new prompt family, no new AI run type, no new vector source type.** The first feature since
Phase 1 to add none of the three. `VEC-001`'s four source types are fully registered —
`matching-and-ranking` took two and `interview-pipeline` took two — and this feature adds no
embeddable record type.

**One new background job type is *not* registered.** §25's eleven rows contain no selection, offer
or closure job. The staleness nudge (`TS-BL-070`) is the one candidate and it is a scheduled
evaluation rather than an event-triggered job; `design.md` D11 records how it runs on
`platform-core`'s dispatch chain without inventing a twelfth §25 row.

**One event published with no subscriber** — the closure cascade's *"move bench to the resurfacing
pool"* consequence. `resurfacing-and-communications`' `TS-BL-075` is named as the intended consumer
and depends on `TS-BL-069` for exactly this. Same shape as `hiring-postings`' unregistered
`posting.opened` and, before it, `ai-platform-governance`'s pre-designed envelope (`design.md` D8).

**Existing capability consumed, not modified** — `platform-core`'s workflow service, dispatch
pattern, internal notification delivery and audited runtime-configuration registry;
`access-control-and-admin`'s permission evaluator, assignment-scope predicates, query-time filter
contract, seeded matrix and durable audit writer; `identity-and-access`'s `users.hubble_user_id`
boundary (consumed as a boundary — the rule is that nothing crosses it);
`interview-pipeline`'s consolidated scorecard, its Application-machine registration and its
disposition vocabulary; `hiring-postings`' posting machine, `vacancy_type`/`vacancy_count` and its
immutability rule; `candidate-intake`'s Application record; `design-system`'s Floating Action Panel,
dense data table, forms and shell states.

**Downstream features that block on this one** — `insight-and-reporting`'s `TS-BL-071`, `TS-BL-072`
and `TS-BL-073` (the last named *closure-readiness reporting*, on `TS-BL-069`), and
`resurfacing-and-communications`' `TS-BL-075` (on `TS-BL-069`) and therefore `TS-BL-076`'s priority
lane, whose precedence rule reads the tag `TS-BL-069` writes and whose carried evidence flows
through `TS-BL-066`'s mechanism.

**External constraints carried, not solved here** — **`OD-002`** leaves it unknown whether a Hubble
ID validation endpoint exists at all, so validation runs behind a port with a stub and
`hubble_id_validated` stays false rather than blocking closure. **`OD-004`**'s retention period caps
`C-11`'s configurable eligibility windows — *"you cannot resurface a candidate whose data has been
deleted"* — so `TS-BL-066`'s TTL configuration carries that ceiling as a validation rule it cannot
yet evaluate. **`OD-005`** keeps candidate-facing communication out of scope. **`OA-01`** records
that this feature's carry-forward volume assumption may be wrong; `TS-BL-066` ships the
instrumentation that would reveal it and changes nothing on the assumption itself. **`OD-001`**
(the Hubble login contract, per `KNOWN_ISSUES.md`) is unrelated to the identifier captured here and
`identity-and-access` D3 is why — recorded so the two are not conflated during apply.

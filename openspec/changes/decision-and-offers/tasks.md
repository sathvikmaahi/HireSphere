# Decision & Offers — Backlog

This feature's complete backlog: seven items, `TS-BL-064` through `TS-BL-070`, decomposed in
`exploration-notes.md` D.10. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built, and nothing was inherited.** `talentsphere-wave-1-foundation` has no
`decision/` delta spec, none of its `design.md` D1–D18 was allocated here (`platform-core` D11
accounts for all eighteen elsewhere), Sprint 0 shipped platform-layer code only with *"no hiring
feature exist[ing] yet, by design"*, and `KNOWN_ISSUES.md` carries nothing about selection, offers,
onboarding or closure. Checked rather than assumed — see `design.md` D12, which also corrects a
citation three prior features have made: **`KNOWN_ISSUES.md` does not record the open AI-provider
question (`OD-003`)** — its only Hubble entry is `OD-001`, the *login* contract. Recording the
provider is `ai-platform-governance`'s still-unbuilt task 1.18. This feature invokes no AI at all,
so `OD-003` does not constrain it either way.

**`TS-BL-069`'s core citation is a recommendation, and a quarter of it is superseded.** *"Closure
semantics"* sits in `exploration-notes.md`'s **"Recommendations made, not yet formally decided"**
table — the same status class `candidate-intake` D1 handled for `D23a` and `interview-pipeline` D2
handled for "reason on every disposition", and the same discipline applies: name the class, do not
cite it as settled. Unlike those two, part of this one is superseded **in writing**: `G-13` states
*"This supersedes the earlier working assumption of closing on `offers_accepted == vacancy_count`."*
`design.md` D1 splits it four ways. **Build the `G-13` test, not the recommendation's trigger. Do not
build a `FILLING` state** — §11.1's `onboarding` already is it, and `G-13` says so.

**Build `TS-BL-066` against `C-07`, not `D17` as written.** `C-07` reverses `D17` twice and it is
possible to satisfy one reversal and miss the other: the **confirmatory round for the new posting is
required**, and `D17`'s **non-round-1 entry point is dropped** — *"candidates now enter at a round,
not at selection."* A carry-forward that admitted a candidate straight to selection would satisfy
`D17`'s original text and break `SCR-001`, `BR-010`, `BR-011` and `AI-009`, which is precisely the
conflict `C-07` dissolved.

**Two names for "slot" are two different tables, and merging them passes `BR-012` while silently
failing `CLS-001`.** `D20`'s "15 ranked slots" are `priority_selections` rows; `G-13`'s
`open → reserved → filled` machine runs on `vacancy_slots` rows, of which a 3-vacancy posting has
three. `design.md` D2 tabulates both. Read `D20` and `G-13` together without that separation and a
3-vacancy posting with two joiners looks like fifteen rows, thirteen of them in an undefined state.

**`D20`'s vacancy-reduction consequence cannot be built as written.** `hiring-postings` made vacancy
count immutable once a posting opens, closing `D14`'s own open edge case, so a count reduction on an
open posting is rejected and a 15-slate never shrinks by that route. The eviction rule attaches to
**vacancy-slot cancellation** instead, and both `BR-012`'s cap and `CLS-001`'s closure test are
implemented against the count of **non-cancelled vacancy slots** — `design.md` D3 records why the
literal reading leaves a posting with a cancelled vacancy permanently uncloseable with no remedy.
This is a deliberate reinterpretation of two requirements' literal wording; do not quietly revert it
during apply, and do not extend it further.

**Consume two things earlier features built; do not re-derive either.** `identity-and-access`
`design.md` D3 already built the two-Hubble-identifier table — **no foreign key, no shared uniqueness
constraint, no shared column name** — and re-deriving it is how a cross-entity constraint gets added
by accident, rejecting the internal-candidate case D3 names. `interview-pipeline`'s
`interview/scorecard-consolidation` spec already fixes what a completed consolidated scorecard is and
states that it carries **no** carry-forward link; `TS-BL-066` builds the link against that document
as written, not against an assumed shape.

**`onboarding` was declared and undriven, and three of the six advances that reach it are built
here.** Found during verification. `hiring-postings` `TS-BL-038` declares §11.1's transition set
whole — all thirteen states, cancellation reachable *from* the six pipeline states — but implements a
caller only for the entry, pause/reopen and terminal transitions, and no other feature writes posting
state at all. So `TS-BL-069`'s `onboarding → filled` guarded a state nothing reached. `TS-BL-064`
drives `scorecard_review → selection`, `TS-BL-067` drives `selection → offer` and
`offer → onboarding`; **`open → screening`, `screening → interviewing` and
`interviewing → scorecard_review` are `interview-pipeline`'s and are flagged, not built** —
`design.md` D9a assigns each to its owning item and trigger. **Until that `/opsx:update` run against
`interview-pipeline` lands, the closure path is exercisable only from `scorecard_review` onward.** Do
not close that gap by relaxing the closure guard; §11.1 fixes the path and a permissive guard would
hide another feature's gap behind this one.

**These are posting states, not Application states.** §11.1 puts `screening`/`interviewing`/
`scorecard_review`/`selection`/`offer`/`onboarding` on `job_postings.status`; §11.2's Application
stages carry deliberately different names (`Shortlisted`, `InterviewScheduled`, `Interviewed`,
`ScorecardReady`, `PrioritySelected`, `OfferInProgress`, `OfferAccepted`, `OnboardingComplete`,
`Recruited`). The adjacent names make it easy to read `onboarding → filled` as an Application
transition and put fulfilment closure on the wrong entity — `design.md` D9 asserts the split.

**Closure is an action that becomes available, not a rule that fires.** `CLS-002`/`CLS-003` say
*"shall be disabled when"*, and §13.2 exposes `POST /api/postings/{postingId}/close-filled`. That
reading is also the only one compatible with `hiring-postings`' **shipped** assertion that no
transition it registers fires without an authenticated actor. `design.md` D7 records the reasoning
and the one-predicate-two-readers rule that keeps `CLS-004`'s checklist and the closure guard from
disagreeing.

**One event is published with no subscriber, on purpose.** The cascade's *"move bench to the
resurfacing pool"* consequence belongs to `resurfacing-and-communications`' `TS-BL-075`, unproposed.
`TS-BL-069` publishes `posting.closed` and registers no handler, with `TS-BL-075` named as the
intended consumer — the same shape as `hiring-postings`' unregistered `posting.opened`
(`design.md` D8). **Do not build the resurfacing side**, and do not let closure depend on delivery
succeeding.

**Reuse, do not rebuild, eight mechanisms that already exist.** `design-system`'s **Floating Action
Panel** is what `TS-BL-065` renders into — its own spec names Priority Selection as its first
consumer and states it *"owns no task state"*, so the 5×n semantics live here and the component is
consumed unchanged. `platform-core`'s **workflow service** executes every transition, its **dispatch
pattern** carries the closure event and the staleness sweep, its **internal notification delivery**
notifies panelists, and its **audited runtime-configuration registry** holds the TTL and the nudge
interval. `access-control-and-admin`'s **evaluator** and **scope predicates** decide every
authorization, and its **durable audit writer** records every material write.

**Dependency edges pointing outside this feature.** `TS-BL-064` needs `interview-pipeline`'s
`TS-BL-062`; `TS-BL-065` needs `design-system`'s `TS-BL-010`; `TS-BL-069` needs `hiring-postings`'
`TS-BL-038` (**an edge `D.10` does not carry**, justified on `design.md` D9's own reasoning rather
than by any standing grant — `TS-BL-069` registers transitions on the posting machine `TS-BL-038`
declares, and `TS-BL-038` sits on no transitive path to this item, so without the edge the
registration has nothing to register against); `TS-BL-070` needs `TS-BL-038` as `D.10` records. Nine further
interfaces are built against without being `depends_on` edges: `platform-core`'s `TS-BL-002`
(audited runtime configuration), `TS-BL-004` (workflow service), `TS-BL-005` (notification delivery)
and `TS-BL-006` (dispatch); `access-control-and-admin`'s `TS-BL-018` (evaluator), `TS-BL-020`
(durable audit writer) and `TS-BL-022` (seeded matrix); `hiring-postings`' `TS-BL-037`
(`vacancy_type`/`vacancy_count` and its immutability rule); and `candidate-intake`'s `TS-BL-046`
(the Application record). Where an interface has not landed, build against its declared shape and
say so.

**One question handed forward is deliberately answered "no change".** `interview-pipeline` D7
refused merges involving an Application at or beyond `PrioritySelected` and offered this feature the
option to replace the refusal with a rule. **The refusal is kept** — `design.md` D10 records why: two
of the three questions it named are now answerable, but the third is about an external side effect
(what the organisation believes it hired), not about data. `TS-BL-064` adds earlier detection instead,
which is where `SEL-003` cannot reach.

**Four things are deliberately unresolved and must not be decided during apply.** **`OD-002`** leaves
it unknown whether a Hubble validation endpoint exists, so validation runs behind a port with a
shipped stub and **closure never gates on `hubble_id_validated`** — gating on it would make every
finite posting permanently uncloseable while `OD-002` stays open. **`OD-004`**'s retention period
caps `C-11`'s configurable windows and cannot yet be evaluated against. **`OA-01`**'s carry-forward
volume assumption may be wrong; ship the instrumentation, change nothing on the assumption.
**`OD-005`** keeps candidate-facing communication out of scope, so an offer reaches a candidate by a
manual channel outside the system — `C-04`'s recorded accepted gap, inherited rather than introduced
here.

---

## 1. TS-BL-064 — Priority Slot mechanics

**Goal:** the selection arithmetic cannot be corrupted — the cap holds, ranks stay unique, no
candidate occupies two positions, every reorder and eviction leaves an audited reason, and the
remainder of the Application lifecycle exists so a selected candidate can travel it. Covers
`decision/priority-slots`.

```yaml
backlog_items:
  - id: TS-BL-064
    feature: decision-and-offers
    depends_on: [TS-BL-062]
    status: not-started
```

**Read `design.md` D2, D3 and D9 before starting.** D2 separates the two tables both called "slots".
D3 records that `D20`'s vacancy-reduction trigger is unreachable and that the cap and the closure
test both key on **non-cancelled vacancy slots** rather than the literal `vacancy_count`. D9 fixes
that this item declares the **whole** remaining §11.2 range in one registration and that
`TS-BL-066`–`TS-BL-069` invoke transitions rather than declaring them.

- [ ] 1.1 Migrate `priority_selections` by reversible migration with §12.2's columns —
      `candidate_application_id`, `priority_rank`, `selection_tag`, mandatory `reason`, `selected_by`,
      `selected_at`, `status` — and **no `vacancy_slot_id`**: §12.2 gives it none, and adding one
      would imply a selection is made against a particular vacancy, which `SEL-002`'s
      posting-scoped rank uniqueness contradicts (`design.md` D2)
- [ ] 1.2 Migrate `vacancy_slots` by reversible migration with §12.2's columns — `job_posting_id`,
      `slot_number`, `status`, `recruited_candidate_application_id` — this being the item
      `hiring-postings` task 5.1 explicitly deferred it to (*"Do not create `vacancy_slots`"*)
- [ ] 1.3 Create one vacancy slot row per vacancy for a finite posting, and **none** for an
      evergreen posting (`JOB-004`, `D14`)
- [ ] 1.4 Implement the cap as **five times the count of non-cancelled vacancy slots**, rejecting an
      over-cap selection by naming the cap and the current count (`SEL-001`, `BR-012`,
      `design.md` D3). Assert equality with `5 × vacancy_count` for any posting with no cancelled
      vacancy — the divergence must be reachable only through cancellation
- [ ] 1.5 Implement evergreen selection as **uncapped and unranked**, with rank uniqueness still
      enforced wherever a rank is present (`D14`, `D20`, `SEL-002`)
- [ ] 1.6 Enforce unique `priority_rank` among a posting's active selections, rejecting a collision
      rather than resolving it, and freeing a rank on removal (`SEL-002`, `design.md` D4)
- [ ] 1.7 Enforce one active selection per Application **and** per candidate per posting — the
      second evaluated across the candidate's Applications to that posting, since `D23a`'s stated
      corruption is one person holding two of five slots (`SEL-003`)
- [ ] 1.8 Implement displacement and insertion as a **single audited reorder** carrying actor, prior
      and resulting order, and a mandatory reason; assert no code path changes a rank without writing
      one (`D20`, `BR-018`)
- [ ] 1.9 Implement vacancy-slot cancellation: reduce the cap, evict selections beyond it with a
      mandatory reason each, tag them `selected_not_offered`, re-sequence remaining ranks as one
      audited reorder, and **refuse cancellation of a filled slot** (`D20`, `SEL-004`, `SEL-005`,
      `design.md` D3)
- [ ] 1.10 Implement the vacancy-slot machine — `open → reserved` on offer acceptance,
      `reserved → filled` on recruitment, release back to `open`, and **no other transition** —
      per `G-13`'s explicit diagram
- [ ] 1.11 Register the **remainder of the Application state machine** declaratively against
      `platform-core`'s workflow service: §11.2's `PrioritySelected`, `OfferInProgress`,
      `OfferAccepted`, `OnboardingComplete`, `Recruited`, `SelectedNotOffered`, `OfferDeclined`,
      `OfferWithdrawn`, their permitted transitions, the permission each declares, and `WF-005`'s
      reason-required marking on every negative and terminal one. `interview-pipeline` D10 declared
      these states reachable and left the transitions to this item by name
- [ ] 1.12 Assert the forward path is complete — `priority selected` appears among the transitions
      available from `scorecard ready` — and that no other capability in this feature declares a
      state or transition (`design.md` D9)
- [ ] 1.13 Assert no code path writes an Application's `status` outside the workflow service
      (`WF-001`, `NFR-009`)
- [ ] 1.14 Advance the posting from `scorecard_review` to `selection` on the **first** committed
      selection, attributed to the committing actor inside that action, and as a **no-op** where the
      posting is not in `scorecard_review` — never moving a posting backwards. One of three advances
      this feature owns; `hiring-postings` `TS-BL-038` declares the transition and nothing invoked it
      (`design.md` D9a)
- [ ] 1.15 Surface `D23a`'s posting-scoped possible-duplicate flag at the point of selection as a
      **non-blocking warning**, blocking nothing and merging nothing (`design.md` D10) — this is the
      earlier detection that makes `interview-pipeline`'s merge refusal cheaper, since `SEL-003`
      evaluates on candidate identity and two unmerged duplicates disagree on exactly that
- [ ] 1.16 Audit every selection, retag, reorder and eviction through the durable audit writer with
      actor, target, prior and new value, reason and correlation identifier (`API-003`, `D6`)

## 2. TS-BL-065 — Priority Selection UI

**Goal:** the Practice Manager's selection is accumulated across a posting's ranked candidates in the
shared Floating Action Panel and committed with a tag and a typed reason — the product's most
consequential decision, made on human judgement over AI-marked evidence with no AI in the loop.
Covers `decision/priority-selection`.

```yaml
backlog_items:
  - id: TS-BL-065
    feature: decision-and-offers
    depends_on: [TS-BL-064, TS-BL-010]
    status: not-started
```

**Read `design.md` D4 before starting.** It answers `D20`'s explicit design-time invitation about
tiered ranking — `SEL-004`'s tags **are** the tier, `SEL-002`'s unique rank is retained, and no
separate tier field is introduced. `S.1` records that the panel fit *"emerged from reading the guide;
it wasn't designed in from our side"*, and `design-system`'s `action-panel` spec was written against
this consumer.

- [ ] 2.1 Build the Priority Selection Slate (§14.2) into `design-system`'s **Floating Action
      Panel**, supplying its chips, controls and running output; assert **no second panel, modal or
      drawer** exists in this capability (`S.1`, `design-system/action-panel`)
- [ ] 2.2 Keep the panel stateless with respect to the selection — removal and reorder invoke this
      capability's handlers, per the panel spec's *"SHALL NOT own, fetch or mutate the underlying
      task"*
- [ ] 2.3 Verify the panel persists across navigation with a selection in progress, which is the
      cross-page property `reference/layout-spec.md` §6 exists for and `S.1` matched to this workflow
- [ ] 2.4 Build the **two presentations** — ranked positions against the cap for finite, a flat
      unranked list for evergreen — and assert neither renders a fixed set of five empty boxes for an
      unbounded posting (`D14`, `D20`)
- [ ] 2.5 Require a tag from `SEL-004`'s four values **and** a mandatory reason to commit, rejecting a
      commit missing either and a tag outside the four (§12.2's `reason` "Mandatory", `SEL-005`)
- [ ] 2.6 Present the finite slate **grouped by tag, ordered by rank within tag** (`design.md` D4),
      and assert no tier attribute distinct from the tag exists
- [ ] 2.7 Authorize commit, retag, reorder and removal through
      `access-control-and-admin`'s evaluator against the posting, and assert **no hard-coded role
      comparison** in any authorization path — §12.2's `selected_by` reads "Practice Manager **or
      authorized user**", so the matrix decides (`AUTHZ-003`, `C-02`, `C-03`)
- [ ] 2.8 Render each candidate's ranking output, approved consolidated scorecard and mandatory
      criteria status with their **existing** evidence-source and AI-generated markings intact
      (`G-01`, `G-02`, `AI-001`, `UI-004`) — consumed as produced, not re-labelled here
- [ ] 2.9 Assert this capability makes **no AI call**: no prompt payload, no gateway invocation, no
      run record, no generated summary. The ten families are closed elsewhere and none is this
      feature's
- [ ] 2.10 Assert no raw color, spacing or type value and no new shared component — every element
      resolves from design tokens, page templates, the dense data table, forms and the panel
      (`AGENTS.md`'s standing bar, `TS-BL-008`–`TS-BL-012`)

## 3. TS-BL-066 — Carry-forward eligibility & scorecard reuse

**Goal:** a consolidated scorecard gathered for one posting can be cited as prior evidence for
another under a named human's typed attestation — with the confirmatory round still required, so no
spec rule is broken and the attestation remains a real control rather than a checkbox. Covers
`decision/carry-forward`.

```yaml
backlog_items:
  - id: TS-BL-066
    feature: decision-and-offers
    depends_on: [TS-BL-065]
    status: not-started
```

**Read `interview-pipeline`'s `interview/scorecard-consolidation` spec before starting**, not a
mental model of it. It fixes what a completed consolidated scorecard is — one per Application, each
panelist's own recommendation preserved and individually attributable, disagreement rendered and
never resolved arithmetically, the overall recommendation AI-proposed and human-decided — and it
carries a requirement stating that the document is *"readable and citable as prior evidence from
outside its own Application"* while *"no carry-forward eligibility, attestation, or a link from a
scorecard to a second Application"* exists there. **That is the contract this item builds against.**
`C-05`'s second stated ground for reversing `D11` was that *"`D17` carry-forward is far simpler
carrying one consolidated document than N per-round ones"* — this item is where that pays off.

- [ ] 3.1 Migrate a scorecard-to-Application link table by reversible migration carrying the
      scorecard, the target Application, the source Application, `carried_forward`, the attesting
      user, the timestamp, the typed reason and the route taken (`D17`'s consequence,
      `domain-model.md`). §12.2 has no such table — `C-07` records that *"This is us extending
      beyond the spec"*, so this is an addition, not an omission being corrected
- [ ] 3.2 Assert the scorecard is **linked, never duplicated, moved or re-parented**, and that the
      target Application acquires no scorecard of its own by the act
- [ ] 3.3 Assert the carried scorecard's origin Application, posting and approval remain identifiable
      when read from the target Application — the property `interview-pipeline` built for this
      consumer
- [ ] 3.4 Require **one completed interview round for the target posting** before the Application is
      eligible for its own scorecard or for selection, rejecting selection on carried evidence alone
      and naming the outstanding round (`C-07`, `SCR-001`, `BR-010`, `BR-011`, `AI-009`)
- [ ] 3.5 Assert **no selection-stage entry point exists** — `C-07`'s explicit *Dropped* clause
      retires `D17`'s non-round-1 entry: *"candidates now enter at a round, not at selection"*
- [ ] 3.6 Implement `D21`'s two paths — shared job description in any version takes a short typed
      reason; a different description requires a fuller justification including an explicit
      competency-difference acknowledgement — and assert **neither is satisfiable without typed
      text**, per `D17`'s *"the attestation is the control"* and `D21`'s recorded reconciliation of
      the gap that created
- [ ] 3.7 Gate eligibility on a **configurable priority-lane window** seeded at 90 days, held in
      `platform-core`'s audited runtime-configuration registry, with changes audited (`D22`, `C-11`)
- [ ] 3.8 Implement expiry as **demotion, not deletion** — beyond the window the source scorecard
      remains readable as context and carry-forward is refused (`D22`: *"The lane does not delete
      people; it demotes them"*)
- [ ] 3.9 Reject a window set longer than the configured retention period, carrying `C-11`'s hard
      constraint (*"you cannot resurface a candidate whose data has been deleted"*). `OD-004` is
      open, so the rule is written and the ceiling is a configuration value — build the validation,
      do not invent the period
- [ ] 3.10 Re-evaluate mandatory criteria against the **target** posting's requirements regardless of
      carried evidence, and assert no path copies a prior mandatory-criteria status forward
      (`C-07`'s closing clause, `G-01`)
- [ ] 3.11 Record the **route taken** on every carry-forward in a form that supports comparing the
      two routes' volumes — `OA-01` calls this *"a slice 7 reporting requirement, not something to
      add later"*, and it is the only instrumentation that can reveal whether `OA-01`'s assumption is
      wrong
- [ ] 3.12 Make carry-forward available for a selected-not-offered Application **however it was
      surfaced**, and assert this capability contains **no precedence ordering and no suggestion
      pool** — `TS-BL-076` owns the lane and depends on `TS-BL-064`, so it reads what this feature
      writes (`design.md` D8)
- [ ] 3.13 Audit every carry-forward **and every refusal** with actor, both Applications, route,
      reason and outcome — `D17`'s governance note makes this the one place a hiring decision rests
      on evidence gathered for a different role (`API-003`, `G-06`)

## 4. TS-BL-067 — Offer creation & tracking

**Goal:** the acceptance-to-joining gap where candidates are most often lost becomes visible and
accountable — four independent status tracks, a reason on every negative move, no compensation amount
anywhere, and no message sent to a candidate. Covers `decision/offer-tracking`.

```yaml
backlog_items:
  - id: TS-BL-067
    feature: decision-and-offers
    depends_on: [TS-BL-065]
    status: not-started
```

**Read `design.md` D5 before starting.** `OFF-001` names five things a recruiter manages and two of
them read like capabilities this feature does not have: *"salary finalization"* is a status enum with
no amount (`D19`), and *"candidate communication status"* is a field recording that a manual,
out-of-system communication happened (`OD-005`, `C-04`). Building either as a capability would
contradict `D19`'s decision and `C-04`'s deferral.

- [ ] 4.1 Migrate `offer_onboarding_records` by reversible migration with §12.2's columns —
      `candidate_application_id`, `salary_status`, `offer_status`, `joining_status`,
      `onboarding_status`, `hubble_id`, `hubble_id_validated`, `owner_id`, `blocker_reason`,
      `updated_at` — and **no salary amount column** (`D19`, and §12.2 independently has none)
- [ ] 4.2 Permit at most one record per Application, creatable only for an Application holding an
      active priority selection (`OFF-001`, §11.2's `PrioritySelected → OfferInProgress`)
- [ ] 4.3 Keep the four status tracks **independent** — no update to one cascading to another — and
      assert no such path exists (§12.2's four columns, `G-14`)
- [ ] 4.4 Reject an update carrying a salary figure rather than storing it in a free-text field, and
      assert no field holds a compensation amount, rate, band or currency value (`D19`'s *"No
      compensation data, which means no extra access tier on top of `D16`"*)
- [ ] 4.5 Authorize salary and offer field edits through the evaluator (`OFF-002`, `AUTHZ-003`), and
      assert no hard-coded role comparison
- [ ] 4.6 Require a reason on any track entering a blocked, declined, withdrawn or no-show value,
      and require `blocker_reason` on the record wherever a track is blocked — held on the record as
      well as in the audit trail, because `CLS-003`'s checklist must render it (`OFF-008`, `G-14`,
      `WF-005`)
- [ ] 4.7 Drive the Application transitions from `offer_status` through the workflow service —
      extended, accepted, declined, withdrawn — invoking transitions `TS-BL-064` declared and
      declaring none here (§11.2, `WF-001`, `design.md` D9)
- [ ] 4.8 Reserve an open vacancy slot on acceptance and release it on a decline, withdrawal or
      post-acceptance renege, per `G-13`'s *"slot returns to `open`; posting was never closed"*
- [ ] 4.9 Advance the posting `selection → offer` on the **first** offer extended and
      `offer → onboarding` on the **first** offer accepted — each attributed to the updating actor
      inside that action, each a **no-op** unless the posting is in the immediately preceding state,
      and neither ever moving a posting backwards. **This is what makes `TS-BL-069`'s
      `onboarding → filled` transition reachable at all** (`design.md` D9a). Assert no posting
      advance fires with no authenticated actor, which is what keeps `hiring-postings`' own assertion
      true
- [ ] 4.10 Record candidate communication **status** and assert this capability contains **no
      candidate-addressed message, template or delivery path** (`OD-005`, `C-04`'s accepted gap)
- [ ] 4.11 Build the Offer and Onboarding Tracker surface (§14.2) from the dense data table, page
      templates and forms, adding no new shared component, and expose §13.2's
      `GET /api/postings/{postingId}/offer-tracker` and `PATCH /api/offer-records/{recordId}`
- [ ] 4.12 Audit every status change with actor, field, prior and new value, reason where required,
      and a correlation identifier, carrying **references rather than candidate personal data**
      (`API-003`, `BR-018`, `D6`)

## 5. TS-BL-068 — Onboarding record & Hubble ID capture

**Goal:** a hire is provable — an identifier a recruiter entered by hand, unique among the people this
product says it recruited, kept rigorously separate from the identifier a staff member logs in with,
and never a gate that a still-unspecified validation endpoint can freeze. Covers
`decision/onboarding-hubble-id`.

```yaml
backlog_items:
  - id: TS-BL-068
    feature: decision-and-offers
    depends_on: [TS-BL-067]
    status: not-started
```

**Read `identity-and-access` `design.md` D3 before starting, and consume it.** It already built the
two-Hubble-identifier table and gives two reasons that are constraints rather than preferences:
candidates have zero system access, so a hired candidate must not acquire a `users` row by holding an
identifier; and an internal candidate or rehire legitimately holds both, so a cross-entity uniqueness
constraint *"would reject a real and expected case."* **Do not re-derive the split** — re-deriving it
is precisely how the constraint D3 warns against gets added.

**Read `design.md` D6 for the one gate that must not exist.** `OD-002` is open. If closure required
`hubble_id_validated`, **no finite posting in this product could ever close** — the largest silent
failure available in this feature.

- [ ] 5.1 Implement manual Hubble ID entry against an onboarding record whose onboarding status is
      complete, rejecting entry before completion and naming the outstanding completion (`OFF-004`,
      `D08`). Expose §13.2's `POST /api/offer-records/{recordId}/complete-onboarding` and
      `POST /api/offer-records/{recordId}/hubble-id`
- [ ] 5.2 Assert **no write to the HR system** exists — `D08` rules provisioning out on §4.2
      out-of-scope #10 (*"HRIS employee master creation"*), and the non-goal is confirmed by the
      reference spec
- [ ] 5.3 Assert the onboarding Hubble ID and `users.hubble_user_id` are **unrelated**: no foreign
      key, no shared uniqueness constraint, no shared column name, and no code path resolving one
      from the other in either direction (`identity-and-access` D3)
- [ ] 5.4 Assert the same value on both is **accepted**, and that capturing a Hubble ID creates or
      modifies no user record — D3's internal-candidate and rehire case, and `project.md`'s
      zero-candidate-access non-goal
- [ ] 5.5 Enforce Hubble ID uniqueness **scoped to recruited candidates** as a database constraint
      at that scope, blocking a duplicate within the set and **not** enforcing uniqueness outside it
      (`OFF-006`, `OFF-007`, `design.md` D6) — application code must not enforce a narrower rule
      than the schema
- [ ] 5.6 Implement `OFF-007`'s *"controlled administrative procedure"* as a distinct
      permission-gated correction with a mandatory reason and an audit record, unavailable on the
      ordinary update path, gated on the administer action through the evaluator rather than a role
      name (`ADM-005`, `D16`)
- [ ] 5.7 Implement validation behind a **port with a shipped stub**, following
      `identity-and-access` D1's shape (the mock is a shipped artifact, not test scaffolding) and
      **separate from the login adapter** — D3 establishes two integration points, and `OD-001` and
      `OD-002` are two different open questions with two different owners
- [ ] 5.8 Make validation non-blocking in all three cases: no validator configured, validator
      reports unknown, validator unreachable. Capture succeeds, the outcome is surfaced, and
      `hubble_id_validated` stays false (`G-12`, §12.2's *"True when validation available and
      successful"*)
- [ ] 5.9 Gate `Recruited` on onboarding complete **and** a present, unique Hubble ID, filling the
      candidate's reserved vacancy slot, and assert **no gate on `hubble_id_validated`** exists on
      recruitment or closure (`OFF-003`, `OFF-005`, `BR-019`, `G-13`, `design.md` D6). §11.2's
      separate `OnboardingComplete` state is the window `OFF-004`'s after-completion entry happens in
- [ ] 5.10 Audit Hubble ID entry, correction and validation outcome, carrying the identifier's
      presence and scope rather than treating it as free-form personal data (`API-003`, `D6`)

## 6. TS-BL-069 — Closure rules

**Goal:** *Filled* means someone actually started, provable against an HR identifier — with a
checklist that names what is missing rather than a button that silently does nothing, a cascade that
leaves nobody stranded on a closed posting, and a reopen path for the renege that arrives too late.
Covers `decision/posting-closure`.

```yaml
backlog_items:
  - id: TS-BL-069
    feature: decision-and-offers
    depends_on: [TS-BL-068, TS-BL-038]
    status: not-started
```

**The `TS-BL-038` edge is added here, and `D.10` does not carry it.** This item registers transitions
on the posting machine `TS-BL-038` declares, and `TS-BL-038` sits on no transitive path to this item,
so without the edge the registration has nothing to register against. **Justified on `design.md`
D9's own reasoning, not by a standing grant** — `D.11` permits a feature to refine its *internal*
item boundaries, which is a different thing from adding a cross-feature edge. Consistent with
`hiring-postings` D5, which named this item as the owner of the finite half of the split it left
open.

**Read `design.md` D1, D7, D8, D9 and D9a before starting.** D1 splits the *"Closure semantics"*
recommendation four ways — build `G-13`'s test, not `offers_accepted == vacancy_count`; build no
`FILLING` state. D7 fixes closure as an actor-initiated action whose availability is computed, and
the one-predicate-two-readers rule. D8 fixes which cascade effects are built here and which is
published. D9 records the two additions to machines other features registered, and why the two
closure routes are not duplicates.

- [ ] 6.1 Implement fulfilment closure as an **actor-initiated, permission-gated action** exposed at
      §13.2's `POST /api/postings/{postingId}/close-filled`, and assert **no path closes a posting
      with no authenticated actor** — preserving the absence `hiring-postings`'
      `hiring/posting-lifecycle` spec already asserts (`CLS-002`, `CLS-003`, `WF-001`,
      `design.md` D7)
- [ ] 6.2 Assert **no state exists between onboarding and filled** — the recommendation's `FILLING`
      state is not built, because §11.1's `onboarding` already is it and `G-13` says the objection it
      was proposed to answer *"was **wrong**"* (`design.md` D1)
- [ ] 6.3 Implement the fulfilment condition as recruited Applications — onboarding complete, Hubble
      ID present and unique — **equal to the count of non-cancelled vacancy slots** (`CLS-001`,
      `BR-013`, `G-13`, `design.md` D3). Assert that accepted-but-not-joined does **not** satisfy it,
      which is the phantom-fill cost `G-13`'s comparison table names
- [ ] 6.4 Block closure where the condition does not hold **and** any selected candidate remains in
      an unresolved salary, offer or onboarding state, naming that candidate (`CLS-003`) — a
      condition about candidates *not* counted toward fulfilment, and one reason a
      fulfilment-threshold rule would be wrong
- [ ] 6.5 Compute `CLS-004`'s checklist and the closure guard from **one predicate with two
      readers**, exposed at §13.2's `GET /api/postings/{postingId}/closure-checklist`, showing
      missing Hubble IDs and blocking workflow states; assert no fulfilment computation exists that
      only one of the two uses (`CLS-002`, `CLS-004`, `UI-003`, `design.md` D7)
- [ ] 6.6 Register the actor-initiated `onboarding → filled` transition with its computed
      precondition against the posting machine (`design.md` D9)
- [ ] 6.7 Implement the cascade's first effect: **cancel the posting's pending interview tasks** with
      a reason naming the closure, through `platform-core`'s task model
- [ ] 6.8 Implement the second: **notify affected panelists** through `platform-core`'s internal
      notification delivery (`TS-BL-005`), which `D04` kept in early scope — no new transport
- [ ] 6.9 Implement the third: **tag remaining active selections `selected_not_offered`** with their
      original selection reasons retained, and transition those Applications to `SelectedNotOffered`
      (`SEL-004`, `SEL-005`, §11.2). The tag and the state are this feature's; **precedence over
      other candidates is `TS-BL-076`'s** and is not built here (`design.md` D8)
- [ ] 6.10 Implement the fourth's local half: transition the posting's remaining **non-terminal
      Applications** to a terminal state with a reason naming the closure, attributed to a service
      account per `WF-004` and reason-carrying per `WF-005`. This is the only place in the feature
      where a candidate's outcome changes without an individual human act — `design.md`'s Risks
      section records why the alternative (applications left live on a closed posting) is worse under
      `WF-006`/`WF-007`
- [ ] 6.11 Assert a closed posting's candidate and workflow history remain readable (`WF-006`,
      `WF-007`)
- [ ] 6.12 **Publish `posting.closed` through `platform-core`'s dispatch pattern and register no
      subscriber**, carrying the posting reference, the closure kind and references to the terminated
      Applications. **`TS-BL-075` is the intended consumer** and `D.10` makes it depend on this item
      for exactly this. Assert no handler exists in this feature and no suggestion pool, resurfacing
      engine or precedence ordering is built (`design.md` D8)
- [ ] 6.13 Assert **closure does not depend on publication succeeding** — if publication fails the
      posting is still closed and the failure is recorded (`G-12`). Publication is at-least-once with
      idempotency at the consumer, per `platform-core`'s async contract
- [ ] 6.14 Register the **`filled → open` reopen transition** — an edge §11.1 does not define at all
      — requiring a mandatory reason, returning the affected vacancy slot to `open`, clearing
      `closed_at`, and audited. `G-13` requires it in words: *"The audited reopen path remains
      necessary for reneges discovered after a posting does reach `Filled`."* One unguarded edge to
      `open`, not a conditional return to `onboarding` — `design.md` D9 records why, following
      `hiring-postings` D5's refusal to ask the registration surface to grow guards
- [ ] 6.15 On reopen, **recompute the posting's pipeline state in one recorded step** to the furthest
      state its live Applications justify, attributed to the reopening actor, and assert a repeated
      recomputation with no Application changed moves nothing. Without this, a reopen whose remaining
      vacancy is filled by an already-accepted candidate strands the posting in `open` with no
      first-occurrence event left to fire — the concrete failure `design.md` D9's discarded
      "re-traversable" assumption was hiding (`design.md` D9a)
- [ ] 6.16 Verify `onboarding` is actually reachable before asserting the closure path works
      end-to-end. Three advances land here (1.14, 4.9); **the three upstream ones —
      `open → screening`, `screening → interviewing`, `interviewing → scorecard_review` — belong to
      `interview-pipeline` `TS-BL-056`/`057`/`061` and are not built by this feature.** Write the
      open-to-filled end-to-end test and **expect it to fail** until the `/opsx:update` run against
      `interview-pipeline` lands; do not make it pass by relaxing the closure guard
      (`design.md` D9a, Open Questions, Risks)
- [ ] 6.17 Assert this capability ends fulfilment closure in **`filled`**, contains **no manual close
      path**, and does not duplicate `hiring-postings`' `POST /api/job-postings/{id}/close` —
      §11.1 carries both states so that *why* a posting ended stays answerable, which `WF-007`
      depends on (`design.md` D9)

## 7. TS-BL-070 — Evergreen posting lifecycle rules

**Goal:** an unbounded posting is governed by the two things it actually needs — manual-only lifecycle
control and protection from becoming a zombie — with fulfilment logic provably unreachable from it,
and with per-hire timing recorded so a posting that never closes is still measurable. Covers
`decision/evergreen-lifecycle`.

```yaml
backlog_items:
  - id: TS-BL-070
    feature: decision-and-offers
    depends_on: [TS-BL-069, TS-BL-038]
    status: not-started
```

**`hiring-postings` D5 predicted this item adds nothing to the machine, and that prediction is
correct.** §11.1's pause, reopen and manual-close transitions are already registered and already
actor-initiated with reason requirements. **What this item adds is the assertions that make the
prediction true** — that fulfilment closure is unreachable rather than vacuously satisfied — plus the
two `D14` consequences that are not lifecycle at all. Read `design.md` D11 before starting: the
staleness nudge runs on the existing dispatch chain and adds **no twelfth §25 job type**, and the
per-hire fill record is a data obligation, not a report.

- [ ] 7.1 Implement pause, reopen and manual close for an evergreen posting, each permission-gated
      with a mandatory reason and an audit record, **using the transitions `TS-BL-038` already
      registered**; assert this capability registers no state and no transition (`CLS-005`,
      `BR-014`, `D14`, `hiring-postings` D5)
- [ ] 7.2 Reject fulfilment closure, the closure checklist, the fulfilment condition and vacancy-slot
      fulfilment for an evergreen posting, **naming the vacancy type** rather than returning an empty
      result (`BR-014`, `CLS-005`)
- [ ] 7.3 Assert the fulfilment condition does **not** evaluate as satisfied for a posting with zero
      vacancy slots — `JOB-004` means evergreen has none, so a count-over-zero-slots test would be
      vacuously true, which is the specific failure this assertion prevents
- [ ] 7.4 Implement the **staleness nudge** as a scheduled sweep on `platform-core`'s dispatch chain
      raising a review prompt to the posting owner where an evergreen posting has been open beyond
      the configured interval with no recruited candidate (`D14`'s recommendation). Register **no new
      §25 job type** — §25's eleven rows are all event-triggered and its Notifications row already
      covers workflow-event notifications (`design.md` D11)
- [ ] 7.5 Seed the interval at **240 days** in `platform-core`'s audited runtime-configuration
      registry, settable to disabled, with changes audited — the same treatment `C-11` gave the
      eligibility windows and `interview-pipeline` gave `SCR-003`'s unapproved dimensions. Rejecting
      the nudge must cost a configuration change, not a revert
- [ ] 7.6 Assert the nudge **changes no state** — a prompt, never a transition. `BR-014` and
      `CLS-005` make evergreen closure manual-only, and a nudge that paused or closed a posting would
      be exactly the automatic transition `D14` and `hiring-postings` D5 both rule out
- [ ] 7.7 Write a **per-hire fill record** at the moment a vacancy slot reaches `filled`, carrying
      the Application, the posting and the interval from posting open to that hire — for evergreen
      **and finite** postings alike (`D14`: *"For evergreen, time-to-fill must be computed **per
      hire**, not per posting"*). Not reconstructible after the fact, which is why it is written now
- [ ] 7.8 Assert this capability computes **no** metric, average or dashboard — the record exists so
      `insight-and-reporting`'s `TS-BL-071`–`TS-BL-073` can compute (`design.md` D11)
- [ ] 7.9 Assert this capability exposes **no vacancy type or count override** — `hiring-postings`
      closed `D14`'s open mid-life-change question with immutability from the point a posting opens,
      and this feature carries the consequence in `design.md` D3 rather than adding an exception here

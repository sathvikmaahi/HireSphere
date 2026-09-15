# Decision & Offers — Design

## Context

See `proposal.md` — Why, for motivation, and the seven delta specs under `specs/` for the behaviour
contracts. This document covers only the technical decisions this feature must settle: the
recommendation it is named after whose trigger was already superseded, two terminology collisions
that produce wrong code if read literally, the seam `hiring-postings` built and left open on
purpose, the two consequences of closure that reach past this feature's edge, and the reconciliation
checks behind the proposal's claim that nothing was inherited.

Constraints that shape the approach:

- **`TS-BL-069`'s core citation is a recommendation, not a decision.** *"Closure semantics"* sits in
  `exploration-notes.md`'s **"Recommendations made, not yet formally decided"** table. `G-13` then
  superseded one of its four parts explicitly. D1 separates the two.
- **`hiring-postings` registered a posting machine with no automatic transition at all**, and its
  `design.md` D5 states that this was deliberate and that the remaining work splits along exactly
  this feature's item boundary. Its `hiring/posting-lifecycle` spec asserts the absence with a
  test-suite scenario. D7 and D9 build against that, not around it.
- **`interview-pipeline` declared §11.2's `PrioritySelected`-onward states reachable and registered
  no transitions for them**, so a correct application does not look terminal at `ScorecardReady`.
  The transitions are this feature's to register.
- **`hiring-postings` made `vacancy_count` immutable from the moment a posting opens**, closing
  `D14`'s own open edge case. That decision post-dates `D20` and makes one of `D20`'s stated
  consequences unreachable as written. D3.
- **Two of `C-07`'s reversals of `D17` matter to implementation** and it is possible to satisfy one
  and miss the other: the confirmatory round is **required**, and `D17`'s non-round-1 entry point is
  **dropped**.
- **`identity-and-access` D3 already built the two-Hubble-identifier table.** It is consumed
  verbatim; re-deriving it is how a shared uniqueness constraint gets added by accident. D6.
- **`OD-002` leaves it unknown whether a Hubble validation endpoint exists at all.** Any design that
  makes closure depend on successful validation makes closure unreachable. D6.
- **No AI.** The ten prompt families are closed elsewhere; this feature authors and invokes none.
  That removes the gateway, the run log, the disclosure record and the evidence vocabulary from
  every decision below — a simplification worth stating once rather than re-checking per item.
- **Settled stack** (`D09`): FastAPI on Python, PostgreSQL on Cloud SQL, Terraform on GCP with
  GitLab CI/CD, OpenTelemetry, background work on the landing zone's Pub/Sub → Eventarc → Workflows
  → Cloud Run Job chain.
- **Only Local and Dev are provisionable.**

## Goals / Non-Goals

**Goals:**

- A selection whose arithmetic cannot be corrupted: the cap holds, ranks stay unique, every reorder
  and every eviction leaves an audited reason, and no candidate occupies two slots.
- A closure that cannot lie: `Filled` means someone actually started, provable against an HR
  identifier, and the checklist names what is missing rather than a button silently doing nothing.
- Carry-forward whose attestation is a real control — a typed reason from a named human, not a
  checkbox — and which never lets carried evidence substitute for the confirmatory round `C-07`
  requires or for `G-01`'s re-evaluation of mandatory criteria.
- Guarantees checkable by absence: no fulfilment path reachable from an evergreen posting, no
  transition that fires with no actor, no gate on `hubble_id_validated`, no compensation amount, no
  foreign key between the two Hubble identifiers, no AI call anywhere in the feature.
- An honest boundary at the resurfacing edge: publish the seam, name the consumer, build neither
  half of the other feature.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No decision-quality measurement.** Whether the right candidate was picked is a question this
  feature makes answerable — ranked slate, reasons, outcomes — and does not answer.
- **No negotiation model.** `D19` is a status tracker. There is no counter-offer entity, no offer
  version chain, and no approval routing.
- **No second workflow engine, dispatch substrate, notification transport, evaluator, audit writer,
  configuration registry or design-system component.** All are consumed.
- **No reporting surface.** `CLS-004`'s checklist is a precondition display on one posting;
  cross-posting closure readiness is `insight-and-reporting`'s `TS-BL-073`.

## Decisions

### D1 — `TS-BL-069`'s core citation is a recommendation, and `G-13` already superseded a quarter of it

**Stated first, because building the whole recommendation would build a state the product does not
need and a trigger the notes already rejected.** The *"Closure semantics"* row reads:

> Auto-close finite postings on `offers_accepted == vacancy_count`, with an intermediate `FILLING`
> state and an audited **reopen** path for reneges. Closure must **cascade**: cancel pending
> interview tasks, notify panelists, move slot holders to the priority lane, move bench to the
> resurfacing pool.

It sits in **"Recommendations made, not yet formally decided"** — the same status class `D23a`
occupies, which `candidate-intake` D1 handled by name rather than citing as settled, and the same
class *"mandatory reason on rejection"* occupies, which `interview-pipeline` D2 handled the same way.
The discipline both used applies here: name the class, separate what is independently settled from
what rests only on the recommendation, and say which parts are being treated as adopted and why.

Unlike those two cases, **part of this one is not merely unratified — it is superseded, in writing.**
`G-13` states it directly: *"This supersedes the earlier working assumption of closing on
`offers_accepted == vacancy_count`."* So the recommendation splits four ways, not two:

| Part | Status | What this feature builds |
|---|---|---|
| Trigger: `offers_accepted == vacancy_count` | **Superseded by `G-13`**, which is `DECIDED — adopted` | The `G-13` test instead: recruited candidates with onboarding complete and a valid Hubble ID, equal to the effective vacancy count |
| Intermediate `FILLING` state | **Redundant**, and `G-13` says so | Nothing. §11.1's `Onboarding` state already is it |
| Audited reopen path for reneges | **Independently required by `G-13`** — *"the audited reopen path remains necessary"* | The `filled → open` transition §11.1 does not define (D9) |
| Four-way cascade | **Rests only on the recommendation** — treated as adopted | Three consequences directly, one as a published event (D8) |

**Why `FILLING` is not built.** The recommendation proposed it to stop postings sitting *open* for
weeks awaiting joiners. `G-13` records that objection being raised and answered: *"an initial
objection — that postings would sit open for weeks or months — was **wrong**. The spec's posting
lifecycle (§11.1) already routes through a distinct `Onboarding` state."* `hiring-postings` already
registers thirteen posting states, `onboarding` among them, so `FILLING` would be a fourteenth
duplicating one of the thirteen — and every dashboard would then have two ways to render the same
condition.

**Why the cascade is treated as adopted despite resting only on the recommendation.** Three grounds,
the same shape `interview-pipeline` D2 used:

1. **Two of the four consequences are independently required.** `WF-006`/`WF-007` require a closed
   posting to retain candidate and workflow history and remain informative — which is unachievable
   if bench applications are left indefinitely in a live stage on a closed posting. And `SEL-005`
   requires selected-not-offered candidates to *"retain selection reason and be prioritized for
   future resurfacing"*, which is the slot-holder consequence stated as a requirement.
2. **The cost of being wrong is bounded.** Panelist notification is `platform-core`'s existing
   delivery with no new transport; task cancellation is a status write on an existing task model.
   Rejecting either removes a call site, not a schema or a state.
3. **The one consequence with real forward cost is not built here at all.** The resurfacing-pool
   consequence is published as an event with no handler (D8), so if the recommendation is rejected
   the event is simply never subscribed to.

*Alternative considered:* build the trigger as written, since the recommendation is what `D.10`
titles the item after. Rejected — `G-13` is a `DECIDED` entry that names the recommendation's trigger
and replaces it, and `AGENTS.md` is explicit that a decision's supersession travels with it.
Building `offers_accepted == vacancy_count` would produce exactly the phantom fills `G-13`'s
comparison table lists as the cost of that option.

### D2 — Two different tables are both called "slots", and reading `D20` literally gets the arithmetic wrong

`D20` says *"A 3-vacancy posting has **15 ranked slots**"* and `G-13` says
*`vacancy_slot: open ──accept──▶ reserved ──onboard + Hubble ID──▶ filled`*. Read together, those
describe fifteen rows moving through a three-state machine, which is wrong in both directions.

§12.2 carries two tables:

|  | `priority_selections` | `vacancy_slots` |
|---|---|---|
| Rows per finite posting | up to **5 × n** | exactly **n** |
| Keyed to | an Application | the posting |
| Carries | `priority_rank`, `selection_tag`, mandatory `reason`, `selected_by` | `slot_number`, `status`, `recruited_candidate_application_id` |
| States | `active`, `removed`, `superseded` | `open`, `reserved`, `filled`, `cancelled` |
| The rule it enforces | `BR-012`'s cap, `SEL-002`'s unique rank | `G-13`'s fulfilment machine |
| What `D20` calls it | "15 ranked slots" | — |
| What `G-13` calls it | — | "`vacancy_slot`" |

**They are separated in code and in the specs, and neither name is reused for the other.** The
`5 × n` cap counts `priority_selections` rows; the three-state machine runs on `vacancy_slots` rows;
a selection is *promoted* into a vacancy slot when its offer is accepted, and that promotion is the
only relationship between them. `priority_selections` carries no `vacancy_slot_id` — §12.2 gives it
none, and adding one would imply a selection is made *against a particular vacancy*, which
`SEL-001`'s "five candidates per vacancy by priority" arithmetic does not mean: the slate is ranked
across the posting, not per-vacancy, which is why `SEL-002` scopes rank uniqueness to *the posting*.

*Why this is worth a decision rather than a footnote:* the two readings differ in what a 3-vacancy
posting looks like when two candidates have joined — three rows with two `filled`, versus fifteen
rows with two `filled` and thirteen in an undefined state. `CLS-001` counts the first. An
implementation that merged the tables would satisfy `BR-012` and silently fail `CLS-001`.

### D3 — `D20`'s vacancy-reduction consequence is unreachable as written; the eviction rule attaches to slot cancellation instead

`D20`'s third consequence: *"**Vacancy count reduction** (3 → 1) shrinks 15 slots to 5; the 10
evicted candidates require a reason and flow into the **priority lane**."*

**That path no longer exists.** `hiring-postings`' `hiring/job-posting` spec requires that vacancy
type and count are *"editable while a posting is in draft or pending approval, and immutable from
the point the posting opens"*, and its `design.md` D12 explains why: `D14` left the mid-life type
change open in its own words, and immutability closes it. A count reduction on an open posting is
rejected, so a 15-slate never shrinks by that route.

**The situation `D20` described is real; only its trigger is gone.** §12.2's `vacancy_slots.status`
includes `cancelled` — the schema anticipates a vacancy that will not be filled without the posting
changing its declared count. So `D20`'s consequence attaches there:

- **Cancelling a vacancy slot reduces the effective vacancy count**, and therefore the cap.
- Selections beyond the reduced cap are **evicted with a mandatory reason**, tagged
  `selected_not_offered`, and their ranks re-sequenced as an audited reorder.
- The candidates land where `D20` said they should — `SEL-005`'s precedence set, which is exactly
  what *"selected but not offered"* means.

**One consequence of this reaches `CLS-001` and must be stated, or closure becomes unreachable.**
`BR-012` reads *"vacancy count multiplied by five"* and `CLS-001` reads *"equal the finite vacancy
count"*, both literally naming `job_postings.vacancy_count`. **Both are implemented against the
count of non-cancelled vacancy slots instead.** Under the literal reading, cancelling one vacancy on
a 3-vacancy posting leaves a posting that can never reach three recruited candidates and therefore
can never close — permanently, with no path out, since the count is immutable. The divergence is
narrow and total: `vacancy_count` and the non-cancelled slot count are equal for every posting where
no vacancy has been cancelled, which is every posting until someone cancels one.

*Alternative considered:* ask `hiring-postings` to permit a downward count revision on an open
posting, via `/opsx:update`. Rejected on its own reasoning — a count change would orphan or
retroactively create `vacancy_slots` rows, and `D14`'s open edge case was closed deliberately.
*Second alternative:* keep the cap on the literal `vacancy_count` and let a cancelled vacancy simply
never fill. Rejected — it makes an uncloseable posting a reachable state with no operator remedy,
which `CLS-004`'s checklist would then have to render as a permanent blocker.

### D4 — `D20`'s tiered-ranking concern is answered by `SEL-004`'s tags, and needs no new mechanism

`D20` closes with an explicit invitation: *"**UX concern (not a blocker):** ranking 15 people in
strict preference order may be more precision than a PM can genuinely supply. Tiered ranking within
the slot set may be worth considering during design."* This is design, so it is answered.

**The tier already exists.** `SEL-004` makes every selection taggable `primary`, `backup`, `hold` or
`selected_not_offered`. That is a four-band tier over the slate, supplied by the same action that
creates the selection. `SEL-002`'s unique `priority_rank` is retained unchanged, and the surface
presents the slate **grouped by tag, ordered by rank within tag** — so a PM expresses coarse
judgement by tag and only orders within a band, which is the precision `D20` doubted could be
supplied globally.

*Why not weaken `SEL-002` to permit ties:* a tie in `priority_rank` makes "who gets the next offer"
undefined at the moment the offer is extended, which is precisely the question the ranking exists to
answer. The tag absorbs the imprecision without introducing an undefined ordering.

*Alternative considered:* a separate `tier` column independent of the tags. Rejected — it would
place two competing groupings on one slate, and `SEL-004`'s four values already read as a tier
(`primary` before `backup` before `hold`).

### D5 — The offer tracker is four independent status tracks, and two `OFF-001` phrases are fields rather than capabilities

`OFF-001` reads: *"Recruiters shall manage **salary finalization**, offer progress, joining
confirmation, onboarding progress, and **candidate communication status**."* Two of those five read
like capabilities this feature does not have.

- **"Salary finalization" is `salary_status`, and no amount is stored.** `D19` chose status-tracker
  scope and recorded the consequence: *"**No compensation data**, which means no extra access tier
  on top of `D16`."* §12.2 agrees — `salary_status` is an enum
  (`not_started | in_progress | finalized | blocked`) and the table contains no salary column. So
  the tracker records *that* compensation was settled, by whom and when, and never *what* it was.
  `OFF-002` still applies: editing the field is permission-gated, evaluated through
  `access-control-and-admin`'s evaluator rather than a role comparison.
- **"Candidate communication status" is a status field, not a communication capability.** `OD-005`
  blocks candidate-facing messaging and `C-04` defers it to `TS-BL-078`. The offer reaches the
  candidate through a manual channel outside the system — `C-04`'s explicitly accepted gap — and the
  tracker records that a recruiter says it happened. It sends nothing.

**The four tracks are independent, not a single pipeline.** `salary_status`, `offer_status`,
`joining_status` and `onboarding_status` each move on their own; §12.2 models them as four columns,
not one state field, and `G-14` added `joining_status` specifically to cover *"the acceptance-to-
joining gap where candidates are most often lost"* — a gap that only exists because acceptance and
joining are separately tracked. What ties them to the Application's §11.2 state is a small set of
gates (D6, D7), not a mirror.

`OFF-008` (*"terminal or negative status updates shall require a reason"*) and `G-14`'s
`blocker_reason` (*"required when blocked"*) are the same rule at two grains and both are enforced:
any track entering `blocked`, `declined`, `withdrawn` or `no_show` requires a reason, and the reason
is stored on the record as well as on the audit entry, because `CLS-003`'s checklist has to render
it.

### D6 — The Hubble identifier split is consumed, not re-derived, and closure does not gate on validation

**Consumed verbatim from `identity-and-access` D3**, which built the table this feature would
otherwise guess at:

| | `identity-and-access` | this feature (`TS-BL-068`) |
|---|---|---|
| Field | `users.hubble_user_id` | `offer_onboarding_records.hubble_id` |
| Attaches to | a TalentSphere **user** | an Application's onboarding record |
| Arrives from | the login response, automatically | a recruiter typing it, manually (`OFF-004`) |
| Means | "this staff member authenticated" | "this hire exists in the HR system" |
| Uniqueness | one local user per Hubble identifier | unique **among recruited candidates** |

**No foreign key, no shared uniqueness constraint, no shared column name.** D3's two reasons are
this feature's constraints, not its choices: candidates have zero system access, so acquiring a
Hubble ID must not produce a `users` row; and an internal candidate or rehire legitimately holds
both, so a cross-entity uniqueness constraint would reject a real and expected case. **The rule is
enforced by absence**, which the spec asserts: no code path resolves one identifier from the other.

**Uniqueness is scoped, and the scope is load-bearing.** `OFF-006` says *"among recruited
candidates"*, not globally. Two candidates can hold the same value on non-recruited records —
typically because a recruiter mistyped one — and only the recruited set must be unique. Implemented
as a partial uniqueness constraint over recruited records, so the database enforces the same scope
the requirement states rather than application code enforcing a narrower rule than the schema.
`OFF-007`'s *"controlled administrative procedure"* is the exception path: a permission-gated
correction with a mandatory reason and an audit entry, gated on the `Administer` action through the
evaluator — not on a role name, per the matrix discipline every prior feature applied.

**Closure gates on present-and-unique, never on `hubble_id_validated`.** `OD-002` — *"Confirm Hubble
ID validation source and API availability"* — is open, and §24 records *"Exact endpoint must be
confirmed."* §12.2 defines `hubble_id_validated` as *"True when validation available and
successful"*, which is a field describing an optional outcome, not a precondition. If closure
required it, then for as long as `OD-002` stays open **no finite posting in the product could ever
close** — the single most consequential silent-failure mode available in this feature.

So `G-13`'s *"valid Hubble ID"* resolves to: **present, non-empty, and unique among recruited
candidates.** Validation, when a validator exists, sets `hubble_id_validated` and is surfaced on the
checklist as information. Validation *failure* is surfaced and does **not** block, because a
validator that is wrong about a real hire must not be able to freeze a posting.

**The validation adapter follows `identity-and-access` D1's shape without reusing its adapter.** A
port with a stub implementation, the stub a shipped artifact rather than test scaffolding. It is a
*separate* port from the login adapter: D3's whole point is that these are two integration points,
and `OD-001` (the login contract, carried in `KNOWN_ISSUES.md`) and `OD-002` are two different open
questions with two different owners.

*Alternative considered:* gate closure on validation where a validator is configured, and skip the
gate where none is. Rejected — it makes closure behaviour depend on deployment configuration, so the
same posting closes in Dev and blocks in Prod, and `CLS-002`'s checklist would have to explain a
blocker whose cause is an environment variable.

### D7 — Closure is an action that becomes available, not a transition that fires

**This is the decision that reconciles this feature with an assertion `hiring-postings` already
shipped.** Its `hiring/posting-lifecycle` spec carries *"No transition is automatic"*, with the
scenarios *"filled is a registered state with no automatic transition into it"* and *"WHEN a
transition is registered that fires without an authenticated actor THEN the test suite fails."*
Meanwhile `D.10` titles `TS-BL-069` *closure rules* and its D5 hands finite auto-closure here.

**Read carefully, there is no contradiction, and the reference spec settles which reading is right.**
`CLS-002` and `CLS-003` do not say closure fires when conditions are met — they say
*"Close as Filled **shall be disabled when**..."*. That is the vocabulary of a control that is
enabled or greyed out. §13.2 confirms it with a route:
`POST /api/postings/{postingId}/close-filled` — *"Close finite posting as filled after validation."*
An endpoint someone calls, validating before it acts.

**So closure is a human action, permission-gated, whose availability is computed.** `TS-BL-069` adds
an actor-initiated `onboarding → filled` transition with a computed precondition. Three things follow:

- **`hiring-postings`' assertion holds unchanged.** Its scenario's subject is a transition *"that
  fires without an authenticated actor"*; this one has one. The absence it asserts — no timer, no
  threshold, no data condition driving a transition on its own — is preserved by this feature rather
  than replaced by it.
- **`project.md`'s rule is satisfied at the point that matters.** *"No code path exists where AI
  output alone ... closes a candidate"* is trivially true here (there is no AI in this feature), but
  the stronger property — a material decision resting with an accountable human — is what
  human-initiated closure delivers. A threshold-fired close would attribute the most consequential
  status change on a posting to a service account.
- **`CLS-004`'s checklist becomes the primary surface, not a decoration.** Because the action is
  human-initiated, something must tell the human whether it is available and why not: *"shows
  missing Hubble IDs and blocking workflow states."* The checklist is computed from the same
  predicate that guards the transition — **one predicate, two readers** — so a checklist showing
  "ready" and a transition rejecting the close is structurally impossible rather than merely tested.

*What "the fulfilment rule" then means, precisely:* recruited candidates with onboarding complete and
a valid Hubble ID (D6), counted against the non-cancelled vacancy slot count (D3). The chain each
candidate travels, with each requirement landing on exactly one step:

```
offer_status = accepted        → Application OfferAccepted,      slot open → reserved
onboarding_status = complete   → Application OnboardingComplete  (OFF-003's precondition met)
hubble_id entered & unique     → Application Recruited,          slot reserved → filled  (OFF-005, BR-019)
all non-cancelled slots filled → close-as-filled becomes available (CLS-001)
```

`OFF-003` (*"cannot be marked Recruited until onboarding status is Complete"*) is necessary but not
sufficient, and `OFF-004` (*"enter Hubble ID after onboarding completion"*) needs a window to happen
in — which is what §11.2's separate `OnboardingComplete` state provides. `Recruited` therefore
requires both, and that is why §11.2 has two states here rather than one.

*Alternative considered:* fire the close automatically and let the checklist be read-only reporting.
Rejected on all three grounds above, and on a fourth: `CLS-003` blocks closure while *"selected
candidates remain in unresolved salary, offer, or onboarding states and vacancy count has not been
satisfied"* — a condition about candidates who are **not** counted toward fulfilment. An automatic
rule keyed on fulfilment alone would close a posting that `CLS-003` says must not close.

### D8 — The cascade has four consequences; three are built and the fourth is published with no subscriber

`D1`'s cascade — *"cancel pending interview tasks, notify panelists, move slot holders to the
priority lane, move bench to the resurfacing pool"* — crosses this feature's edge twice, and one of
those crossings lands in a feature that does not exist yet.

| Consequence | Owner | How |
|---|---|---|
| Cancel pending interview tasks | **Here** | Status write on `platform-core`'s task model, reason naming the closure |
| Notify panelists | **Here** | `platform-core`'s internal delivery (`TS-BL-005`), which `D04` kept in early scope |
| Move slot holders to the priority lane | **Here, for the part that is a state change** | `selection_tag = selected_not_offered`, Application → `SelectedNotOffered`, selection reason retained (`SEL-004`, `SEL-005`, §11.2) |
| Move bench to the resurfacing pool | **`TS-BL-075`, unproposed** | Bench applications transitioned to `NotSelected` here with a reason naming the closure; the pool membership is published as an event with **no handler registered** |

**Why the third row is here and not in `resurfacing-and-communications`.** `SEL-004` and `SEL-005`
put the tag and its retained reason on the selection record, which is `TS-BL-064`'s table, and
`D.10` makes `TS-BL-076` (*priority lane precedence logic*) depend on `TS-BL-064` — the dependency
direction says the lane **reads** what this feature writes. So the split is: **the tag and the state
are this feature's; precedence over other candidates is `TS-BL-076`'s.** Nothing here ranks a
selected-not-offered candidate against anything.

**Why the fourth row is published rather than built.** *"The resurfacing pool"* is a
`MatchSuggestion` set, and `domain-model.md` is explicit that `MatchSuggestion` is *"lighter than
`Application`, promoted into a real `Application` only when a human acts on it"* — a different entity
with its own promotion semantics, owned by `TS-BL-075`, which `D.10` makes depend on `TS-BL-069` for
exactly this reason.

**`TS-BL-069` publishes `posting.closed` through `platform-core`'s single dispatch pattern and
registers no subscriber.** The event carries the posting reference, the closure kind, and references
to the applications this feature just terminated. **`TS-BL-075` is the intended consumer, named
here.** This is the fourth occurrence of this shape and it is deliberately identical to the third:
`hiring-postings` D7 published `posting.opened` with no subscriber and named `TS-BL-052` and
`TS-BL-075`; `ai-platform-governance` designed its envelope before either consumer existed;
`platform-core` shipped its registration surface with nothing registered.

*The property that must hold and is easy to lose:* **closure must not become dependent on
delivery.** If publication fails, the posting is still closed. `G-12`'s degradation reasoning applies
directly, and the alternative would make a future resurfacing feature load-bearing for closing a
posting today. Publication is at-least-once with idempotency at the consumer, which is what
`platform-core`'s async contract already requires.

**One further thing is published and not built: carry-forward's surfacing.** `TS-BL-066` builds
carry-forward as a mechanism that operates on a selected-not-offered Application **however it was
surfaced** — an attestation, a link, a confirmatory-round requirement, a TTL check. `TS-BL-076`
builds the lane that surfaces candidates into it. Neither depends on the other's internals, which is
why `D.10` gives `TS-BL-066` no dependency outside this feature.

### D9 — Two transitions are added to machines other features registered, and the two closure routes are not duplicates

Neither addition creates a second machine. `platform-core`'s workflow service takes declarative
registrations; both of these are registrations against an existing declaration.

**On the Application machine (§11.2), `TS-BL-064` registers the remainder.**
`interview-pipeline` D10 declared `Ranked` through `ScorecardReady` and explicitly declared
`PrioritySelected` onward *"as reachable but not owned"* so that *"a machine that ended at
`ScorecardReady` with no forward edge would [not] make a correct application look terminal"*, adding
that *"their transitions are registered by `decision-and-offers`."* `TS-BL-064` registers them:
`PrioritySelected`, `OfferInProgress`, `OfferAccepted`, `OnboardingComplete`, `Recruited`,
`SelectedNotOffered`, `OfferDeclined`, `OfferWithdrawn` — permitted transitions, the permission each
demands, and `WF-005`'s reason-required marking on every negative and terminal one.

**One declaration, in the first item** — the same item-boundary reasoning `interview-pipeline` D10
gave: four items each appending states would mean four migrations of one registry and no single
place where §11.2 is checkable against the code. `TS-BL-066`, `TS-BL-067`, `TS-BL-068` and
`TS-BL-069` **invoke** transitions; they declare none. Recorded as an item-boundary refinement under
`D.11`'s permission.

**On the posting machine (§11.1), `TS-BL-069` registers two things `hiring-postings` left out.**
The actor-initiated `onboarding → filled` transition with its computed precondition (D7), and
`filled → open` — **an edge §11.1 does not define at all.** §11.1 makes `Filled` terminal: no
outgoing edge, and no path from `Closed` back either. `G-13` requires one anyway: *"The audited
reopen path remains necessary for reneges discovered after a posting does reach `Filled`."*

**These are posting states, not Application states, and the distinction is worth asserting because
the two machines' state names invite exactly the opposite conclusion.** §11.1 puts `Screening`,
`Interviewing`, `ScorecardReview`, `Selection`, `Offer` and `Onboarding` on the **posting**;
§11.2's Application lifecycle uses deliberately different names for its parallel stages —
`Shortlisted`, `InterviewScheduled`, `Interviewed`, `ScorecardReady`, `PrioritySelected`,
`OfferInProgress`, `OfferAccepted`, `OnboardingComplete`, `Recruited`. §12.2 carries
`job_postings.status` as its own stored enum. So `onboarding → filled` is `job_postings.status`
moving, driven by a rule over Applications rather than by any single Application's own state. The
adjacent temptation — reading `onboarding → filled` as an Application transition, since
`OnboardingComplete` and `Recruited` sit right there — would put fulfilment closure on the wrong
entity entirely.

**Reopen goes to `open`, one edge, no guard**, requiring a mandatory reason, clearing `closed_at`,
and returning the affected vacancy slot to `open`. The alternative — returning to `onboarding` when
other slots remain reserved and `open` otherwise — is a conditional, and `hiring-postings` D5
rejected vacancy-type guards on exactly the ground that `platform-core`'s registration surface
*"offers no guards, and this design does not ask it to grow any."* The same restraint applies to a
slot-state guard. What makes `open` a safe landing point is D9a's advance rule, not an assumption
that the path back up is walked again by itself.

### D9a — `onboarding` is declared but undriven, so the closure transition is registered against a state nothing reaches

**Found during verification of this change, and it is the same class of defect this feature already
disclosed for vacancy-count immutability (D3): a rule that is correct against the requirement and
unreachable against what is actually built.** An earlier draft of D9 asserted that after a reopen
§11.1's `Open → Screening → … → Onboarding` path *"is re-traversable from there"* — which assumed it
was traversable at all, and never checked.

**What is declared.** `hiring-postings` `TS-BL-038` registers the machine whole: its
`hiring/posting-lifecycle` spec names all thirteen states and its task 6.1 registers them *"with
§11.1's transition set"*. Its `Pause, reopen, cancel and close` requirement makes cancellation
available *from* `screening`, `interviewing`, `scorecard review`, `selection`, `offer` and
`onboarding`, which only means something if a posting can be in them. **The declaration is complete
and this is not `hiring-postings` having registered a partial machine.**

**What is missing is a caller.** `TS-BL-038`'s implementing tasks cover the entry transitions (6.5,
6.6), pause/reopen/cancel/close (6.8), and §13.2's endpoint set (6.13: *"open, pause, reopen,
cancel, close"*). **Nothing invokes the six advances between `open` and `onboarding`** — and nothing
in any other feature does either: `interview-pipeline` and `matching-and-ranking` write no posting
state at all. So `job_postings.status` reaches `open` and stops, `onboarding` is unreachable, and
`TS-BL-069`'s central transition guards a state no posting arrives at.

**Who drives each advance, by the rule D9 already applies** — the item that owns the triggering event
registers the transition that event causes:

| Advance | Triggering event | Owner |
|---|---|---|
| `open → screening` | first candidate shortlisted on the posting | `interview-pipeline` `TS-BL-056` |
| `screening → interviewing` | first interview round scheduled | `interview-pipeline` `TS-BL-057` |
| `interviewing → scorecard_review` | first scorecard approved | `interview-pipeline` `TS-BL-061` |
| `scorecard_review → selection` | first priority selection committed | **`TS-BL-064`** |
| `selection → offer` | first offer extended | **`TS-BL-067`** |
| `offer → onboarding` | first offer accepted | **`TS-BL-067`** |

**The bottom three are built here.** They are triggered by this feature's own writes, and no other
feature can observe them. **The top three are flagged, not built** — their triggers are events this
feature cannot see, and inventing a poll over shortlists or scorecards to detect them would
duplicate `interview-pipeline`'s own state in a second place. They need an `/opsx:update` run
against `interview-pipeline`, named in Open Questions with the item and trigger for each.

**Three properties the advances must hold**, each of which is easy to lose:

- **Monotone and idempotent.** An advance fires only when the posting is in the immediately
  preceding state; otherwise it is a no-op. A second selection must not pull a posting in `offer`
  back to `selection`, and a fifth offer must not re-fire `selection → offer`.
- **Attributed to the human whose action triggered it, inside that action's transaction.** The
  recruiter who extended the offer is the actor on `selection → offer`. This is what keeps
  `hiring-postings`' *"no transition fires without an authenticated actor"* assertion true — it is
  the same reconciliation D7 makes for closure itself, applied one level down, and it is why these
  are advances-within-an-action rather than a scheduled sweep over postings.
- **Recomputed once on reopen.** After `filled → open`, the posting is advanced in one step to the
  furthest state its live Applications justify, attributed to the reopening actor. Without this, a
  reopen whose remaining vacancy is filled by a candidate who has *already* accepted strands the
  posting in `open` with no event left to fire — the concrete failure the discarded
  "re-traversable" assumption hid.

*Alternative considered:* treat all six advances as `TS-BL-038`'s unfinished work and flag the whole
set to `hiring-postings`. Rejected — `hiring-postings` can observe neither selections nor offers, so
it would need dependency edges onto `TS-BL-064` and `TS-BL-067`, inverting the direction `D.10`
records and making a Phase-2 item depend on two Phase-3 ones. *Second alternative:* build all six
here. Rejected — three of the triggers are `interview-pipeline`'s events, and observing them from
this feature means either a poll or a second copy of its state.

*Why this is disclosed rather than worked around:* the tempting fix is to make closure available
from any state where the fulfilment condition holds, which would sidestep the whole chain. That
contradicts §11.1, and it would hide a real gap in another feature behind a permissive guard here.

**The two closure routes are different closures and must not be merged.**
`POST /api/job-postings/{id}/close` is `hiring-postings`' — the manual close reachable from `open` or
`paused`, ending in `Closed`, which is `TS-BL-070`'s route for evergreen (`CLS-005`).
`POST /api/postings/{postingId}/close-filled` is this feature's — fulfilment closure ending in
`Filled` (`CLS-001`). §11.1 carries both states for a reason: `Filled` means the vacancies were
filled, `Closed` means the posting stopped. A merged route would make "why did this posting end"
unanswerable, and `WF-007`'s requirement that closed postings *"continue to inform candidate history
and analytics"* depends on the distinction.

### D10 — `interview-pipeline`'s refusal on merges past `ScorecardReady` is kept, not replaced

`interview-pipeline` D7 answered `D23a`'s which-application-survives rule as far as its own stages
reached, refused merges involving an application at or beyond `PrioritySelected`, and handed the rest
forward: *"`decision-and-offers` can replace the refusal with a rule; it cannot un-merge a merge this
feature performed wrongly."* It named three questions a rule would have to answer: whether an offer
can be re-pointed, what `BR-012`'s arithmetic does when a slot holder merges, and whether `SEL-003`
already prevents the case.

**Two of the three are now answerable, and the third is the one that matters.**

- **`SEL-003` does not prevent it.** It blocks *"duplicate active priority selection for the same
  candidate and posting"* — evaluated on `candidate_id`, which is precisely the field two
  unmerged duplicate records disagree on. The block fires only *after* a merge makes them one
  candidate, which is too late to prevent the state and is exactly the corruption `D23a` was raised
  about: *"the same person can occupy **two of the five priority slots** ... it corrupts the cap
  arithmetic."*
- **`BR-012`'s arithmetic is recoverable.** Merging two selections into one frees a slate position;
  the surviving selection keeps the better rank and the ranks re-sequence as an audited reorder
  (D3's mechanism, already built here).
- **Whether an offer can be re-pointed is not a data question.** An extended offer, an accepted
  offer, an entered Hubble ID and a `filled` vacancy slot are each a communication with a person or
  a record in an HR system outside TalentSphere. Re-pointing an accepted offer from one application
  row to another silently changes who the organisation believes it hired. `OFF-007`'s existence —
  duplicate Hubble IDs corrected only *"through controlled administrative procedure"* — is the
  reference spec's own acknowledgement that identifier corrections at this stage need a human
  procedure rather than an automatic resolution.

**So the refusal is kept, and `TS-BL-064` adds the one thing that makes it cheaper: earlier
detection.** `D23a`'s posting-scoped duplicate check is a *"non-blocking warning on the ranked
list"*, which the Priority Selection surface renders — so a PM sees "possible duplicate of #7"
before committing two slots to one person, rather than a merge being refused weeks later. That is
prevention where `SEL-003` cannot reach.

*Why keeping the refusal is not deferral:* the honest statement is that the rule requires a decision
about an external side effect the notes have not made, and `interview-pipeline`'s framing —
*"inventing an answer would repeat the mistake `candidate-intake` and `matching-and-ranking` each
declined to make"* — applies to this feature as squarely as to that one. What changes is that the
refusal is now backed by a stated reason rather than an absence of stages. Recorded in Open Questions
with the owner named.

### D11 — The staleness nudge runs on the existing dispatch chain without inventing a twelfth §25 job row

`D14`'s fourth consequence: *"Recommended: a **staleness nudge** (open 240 days, 0 hires → prompt a
review) so evergreen postings do not become zombies."* §25's eleven job rows contain nothing like
it, and every row there is event-triggered.

**It is a scheduled evaluation, not a job type.** `platform-core`'s background chain
(Pub/Sub → Eventarc → Workflows → Cloud Run Job, per `D09`) already carries scheduled invocation;
the nudge is a periodic sweep producing `platform-core` notifications, which §25's **Notifications**
row already covers as *"trigger: workflow events"*. So nothing is registered as a new job type and
§25 stays at eleven.

**The threshold is audited configuration, not a constant.** 240 days and "0 hires" are a
*recommendation* inside a decided entry, so they are seeded into `platform-core`'s audited
runtime-configuration registry — the same treatment `interview-pipeline` gave `SCR-003`'s
unapproved dimensions and `C-11` gave the eligibility windows. Rejecting the nudge costs a
configuration change: setting the threshold to disabled. It is a **prompt**, never a transition —
`BR-014` and `CLS-005` make evergreen closure manual-only, and a nudge that paused or closed a
posting would be exactly the automatic transition `D14` and `hiring-postings` D5 both rule out.

**The second `D14` consequence is a data obligation, not a report.** *"For evergreen, time-to-fill
must be computed **per hire**, not per posting"* — a posting that never closes has no
posting-level time-to-fill. So `TS-BL-070` records a **per-hire fill record** at the moment a
vacancy slot reaches `filled`: the application, the posting, and the interval from posting open to
fill. `insight-and-reporting` computes; this feature makes the computation possible, which it would
not be retroactively.

### D12 — Reconciliation: what was checked, and the one thing three features have now cited wrongly

**Checked rather than assumed**, per the discipline every feature so far has followed.

- **`talentsphere-wave-1-foundation` has no `decision/` delta spec.** Its fourteen cover
  `access-control/`, `ai-platform/`, `design-system/`, `identity/` and `platform/`. Nothing under
  selection, offers, onboarding or closure was written there.
- **None of its `design.md` D1–D18 was allocated here.** `platform-core` D11 accounts for all
  eighteen across the five Phase-1 features. **Expected clean, and clean** — no old decision named
  this feature, which is unsurprising: D1–D18 are evaluator, audit, seed-matrix, design-token, AI
  gateway and infrastructure decisions, none of which reaches Phase 3.
- **`sprint-0-outcome.md` is platform-layer only** — 13 tasks, *"no hiring feature exist[ing] yet,
  by design"*, with 2,000 lines of error handling, observability, feature flags, runtime config,
  audit port and IAM database auth. Its "What Sprint 1 needs to decide" section names task 2.7 and
  task 2.3; neither touches this feature. Its carry-forward table — `audit_logs` narrowing
  unexercised, schema ownership by hand, Sprint-0 endpoints declaring no permission, audit as a port
  — is entirely Wave 1's.
- **`KNOWN_ISSUES.md` carries no entry about selection, offers, onboarding or closure.** Its nine
  entries are environments not provisioned, the mocked Hubble **login** contract (`OD-001`),
  non-durable audit, the unexercised `audit_logs` grant, hand-applied schema ownership, `/build`
  reporting unknowns, the blocked npm registry, and Sprint-0 endpoints without declared permissions.

**One citation correction, made where the correction belongs — in this document, because the error
is not in a shared one.** Three prior propose conversations have attributed the open AI-provider
question (`OD-003`) to `KNOWN_ISSUES.md`. **It is not recorded there.** The file's only Hubble-
related entry is `OD-001`, the *login* contract, and there is no provider entry at all. Recording
the provider decision is `ai-platform-governance`'s **task 1.18**, still unbuilt — so the absence is
correct and expected, not a gap in `KNOWN_ISSUES.md`. This feature invokes no AI at all, so `OD-003`
does not constrain it either way; the note exists so the fourth feature to look does not repeat the
attribution. **No shared document needed editing for this**, and none was edited.

**No `exploration-notes.md`, `domain-model.md`, `glossary.md` or `project.md` correction was needed.**
Two genuine tensions were found and both live outside the shared documents: `D20`'s
vacancy-reduction consequence is superseded by a rule `hiring-postings` decided in **its own**
`design.md` (D3 above), and the *"Closure semantics"* trigger was superseded by `G-13`
**with the supersession written into `G-13`'s own text** (D1 above) — so unlike `D11`'s note-locking
consequence, which `interview-pipeline` found carrying no marker at all, there is nothing
unannotated here to annotate. `domain-model.md`'s spine renders `PrioritySlot` as *"(1-5 ×
vacancy)"* and `Offer & Onboarding Record` as *"(Hubble ID)"*, and its note that a `Scorecard` *"can
link to **multiple** Applications when priority-lane carry-forward applies (D17)"* is exactly what
`TS-BL-066` builds — all three correct as written.

## Risks / Trade-offs

- **[D3's divergence from `BR-012`/`CLS-001`'s literal wording is a real reinterpretation, made
  without the decision-owner in the room.]** → It is narrow (identical behaviour until a vacancy is
  cancelled), it is asserted in the spec so it cannot be silently reverted, and the alternative is a
  permanently uncloseable posting with no operator remedy. If it is judged wrong, the fix is one
  predicate and an `/opsx:update` against this change — not a schema change, since `vacancy_slots`
  carries `cancelled` either way.
- **[D7 reads `CLS-002`/`CLS-003`'s "shall be disabled when" as describing a human-initiated action,
  and the item is titled *closure rules*.]** → The `close-filled` endpoint is independent
  corroboration, and the reading is the only one compatible with `hiring-postings`' shipped
  no-automatic-transition assertion. The residual risk is narrow: if the intent was a fired rule,
  the change is to invoke the same predicate from a scheduled evaluation — the predicate, the
  checklist and the cascade are unaffected.
- **[The `filled → open` reopen edge is not in §11.1 at all.]** → `G-13` requires it in words, and
  without it a renege after `Filled` has no representable outcome. Recorded as an addition rather
  than smuggled in as an implementation detail, so a reviewer sees it.
- **[Until `interview-pipeline` drives `open → screening`, `screening → interviewing` and
  `interviewing → scorecard_review`, this feature's closure chain is reachable only from
  `scorecard_review` onward — so fulfilment closure is unreachable end-to-end in the built system
  even after every item here is complete.]** → D9a establishes the split and this feature builds its
  own three advances, so the gap is one `/opsx:update` run against `interview-pipeline` rather than a
  redesign, and Open Questions names the item and trigger for each. Two independent reasons it is
  disclosed rather than patched here: the permissive alternative (closure available from any state
  where the fulfilment condition holds) contradicts §11.1, and observing shortlist or scorecard
  events from this feature means polling or duplicating `interview-pipeline`'s state. **Sequencing
  consequence worth naming:** an end-to-end test carrying a posting from open to filled cannot pass
  until that run lands, so it should be written and expected to fail rather than omitted.
- **[The three advances this feature does build fire inside another action's transaction, which is a
  weaker form of "actor-initiated" than closure's explicit button.]** → The actor is authenticated,
  holds the triggering action's permission and is recorded on the advance, which is what
  `hiring-postings`' assertion actually requires; and the advance is monotone, so a failure to fire
  is recoverable by the reopen recomputation rather than leaving a posting permanently mis-stated.
  Flagged because it is a second, quieter instance of D7's reconciliation and a reviewer checking
  only the closure route would miss it.
- **[Closure terminates bench applications on a system-generated disposition — a negative outcome
  for a candidate that no human individually clicked.]** → It is attributed to a service account per
  `WF-004`, carries a reason naming the closure per `WF-005`, and follows a decision a human *did*
  make (closing the posting). The alternative is worse: applications left in a live stage on a closed
  posting, which `WF-006`/`WF-007` and every dashboard would then have to special-case. Flagged
  because it is the only place in this feature where a candidate's outcome changes without an
  individual human act.
- **[`OD-002` may resolve to "no validation endpoint exists".]** → D6 makes that the *default*
  rather than a failure mode: capture works, closure works, `hubble_id_validated` stays false. If a
  validator arrives, one adapter implementation lands behind the existing port and nothing else
  changes.
- **[`OA-01`'s assumption may be wrong — attestation could become the routine path rather than the
  exception, making `D21`'s PM sign-off a bottleneck in the flow `D17` existed to speed up.]** →
  `TS-BL-066` ships the route instrumentation `OA-01` calls *"a slice 7 reporting requirement, not
  something to add later"*, so the assumption becomes measurable on first real use. No mitigation is
  built into the flow itself, deliberately — `OA-01` names two candidate remedies and says to pick
  one against data.
- **[Six items depend on `TS-BL-065`'s surface, so a slow UI item stalls the chain.]** → The
  dependency is on the **selection record** it commits, not on its pixels. `TS-BL-066` and
  `TS-BL-067` both need a committed selection and neither needs the panel, so they can be built
  against `TS-BL-064`'s records with `TS-BL-065` in flight. Named so the sequencing conversation can
  see the option rather than reading the chain as strictly serial.
- **[`TS-BL-069` publishes an event whose only consumer is unproposed, so the cascade's fourth
  consequence is unobservable end-to-end until Wave 5.]** → Publication is asserted and tested at the
  boundary; the same accepted gap `hiring-postings` took on `posting.opened` and for the same reason.
  Building the consumer here would build `resurfacing-and-communications`.

## Migration Plan

Seven independently deployable items, in dependency order. All schema changes are reversible
migrations; `TS-BL-064`'s three tables are additive and nothing reads them until `TS-BL-065` writes
one. The two state-machine registrations (D9) are declarative and additive to existing declarations
— rolling back `TS-BL-064` removes the `PrioritySelected`-onward transitions and returns the
Application machine to `interview-pipeline`'s range, with the states still declared reachable, which
is the state the product is in today. `TS-BL-069`'s posting-machine additions roll back the same way,
returning `filled` to a state with no automatic *or* actor-initiated transition into it — which is
`hiring-postings`' shipped position, so the rollback target is a configuration the product has
already run in. No data migration is required at any step: every table is new, and the two
`candidate_posting_applications` columns written here (`status`, `final_outcome`) already exist and
are nullable.

## Open Questions

- **Who drives §11.1's first three posting advances, and when.** `open → screening`,
  `screening → interviewing` and `interviewing → scorecard_review` are declared by
  `hiring-postings` `TS-BL-038` and invoked by nobody (D9a). They belong to `interview-pipeline`
  `TS-BL-056` (first candidate shortlisted), `TS-BL-057` (first round scheduled) and `TS-BL-061`
  (first scorecard approved) respectively, and need an **`/opsx:update` run against
  `interview-pipeline`** — not against `hiring-postings`, whose declaration is already complete, and
  not against this change. **This one is not safely deferrable in the usual sense:** it does not
  change this feature's specs, approach or task breakdown — the three advances this feature owns are
  unaffected either way — but until it lands, `TS-BL-069`'s closure path cannot be exercised
  end-to-end. Recorded here, in Risks, and in `tasks.md` so it is not rediscovered during apply.
- **Whether `D23a`'s which-application-survives rule extends past `PrioritySelected` at all**, and
  if so what happens to an accepted offer or an entered Hubble ID on the non-surviving application.
  D10 records why this feature keeps `interview-pipeline`'s refusal rather than inventing a rule: the
  question is about an external side effect — what the organisation believes it hired — not about
  data. Owner: whoever owns `D23`/`D23a`, whose own status is *"recommendation made, not yet
  confirmed."* Safe to defer: the refusal is a defined, surfaced behaviour, and replacing it later
  changes one branch without unpicking anything.
- **Whether the 240-day zero-hire nudge threshold survives contact with real evergreen postings**
  (D11). Seeded configuration, audited on change; answering it costs a configuration edit.
- **Whether `OD-004`'s retention period will force `C-11`'s seeded 12-month resurfacing window
  down.** `C-11` records the constraint as hard — *"the resurfacing window can never exceed the
  retention period"* — and `TS-BL-066` carries it as a validation rule with nothing yet to validate
  against. Owner: Product, HR, Legal. Safe to defer: the rule is written and the ceiling is a
  configuration value.

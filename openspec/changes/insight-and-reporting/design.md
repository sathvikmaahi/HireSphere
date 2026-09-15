## Context

See `proposal.md` — Why. Three constraints shape everything below, and none of them is negotiable at
this feature's level.

**This feature reads; it does not own.** Every number on every screen here is derived from a record a
sibling feature already writes under its own governance. That makes the central design risk not
correctness but **disclosure**: an aggregate surface is the one place where records that are
individually permission-gated get combined, counted, ranked and written to a file. `D16` makes
Administrators config-only with no candidate PII; `C-02` scopes the Auditor to audit and AI run logs
and nothing else; `G-02`'s evidence-source labelling exists specifically so audit-log access does not
become a PII back door through AI run logs. A reporting surface that helpfully resolves an identifier,
or an export that carries a column the screen omits, reopens all three at once.

**Specification density here is the lowest in the product.** `§20.1` is a five-row table and `§20.2` a
sixteen-item list; neither carries a single requirement ID. There is no `RPT-001` to cite —
`§19`'s module-level requirements stop at `§19.10` Closure. So this design leans on `D`/`C`/`G`/`S`
decisions and on the sibling artifacts that already delegated work here, and cites `§20.1` rows and
`§20.2` items directly where it must. Where a decision has no external anchor at all, it says so.

**The substrates are finished and specific.** `ai-platform-governance`'s `ai-platform/ai-run-logging`
already carries a `Run log access and search` requirement stating in its own source note that *"the
reporting surface built on this search is `insight-and-reporting`'s `TS-BL-074`; this requirement is
the queryable substrate, not a dashboard"*, and a `Human override records` requirement whose
`Divergence is measurable` scenario is the AI-agreement-rate metric in all but name.
`access-control-and-admin`'s `access-control/audit-review` already carries audit search, correlation
retrieval and an audited, classification-labelled export. `decision-and-offers`'
`decision/posting-closure` already computes closure readiness behind a single predicate with a scenario
that fails the suite if a second one appears. Consuming these is not a convenience — reinventing any of
them would contradict a shipped requirement.

## Goals / Non-Goals

**Goals**

- Make the PII boundary a property of the read model that a test can check, not a rule reviewers must
  remember.
- Add exactly one aggregation path per metric, so no two screens can disagree about the same number.
- Consume `TS-BL-069`'s closure predicate, `TS-BL-028`'s run log and override records, and
  `TS-BL-021`'s audit search and export at their existing contracts.
- Close this feature's share of `§25`'s job-type list and `§13.2`'s endpoint list against a named owner.

**Non-Goals** (design-level, beyond the proposal's scope statement)

- **No BI or warehouse layer.** `§24` lists *"Reporting or BI Layer | **Optional** | curated analytics
  views or exports"*. Optional, and a second data store holding candidate-derived rows is a second PII
  surface with its own access model. Ruled out; D10 records what replaces it.
- **No metric this feature defines on its own authority.** Every number traces to a `§20.1` row, a
  `§29` item, or a named `D`/`C`/`G` decision. Where a recommendation is unratified, the metric ships
  labelled as such (D6, D7) rather than being promoted by being built.
- **No candidate-facing or per-candidate surface.** The per-candidate view is `candidate-intake`'s
  (D1); candidates have zero system access, which is a standing non-goal.
- **No new visual primitive**, chart or otherwise (D2).
- **No bias or fairness analytics.** `PRV-006` and `OD-010` — fairness measurement needs demographic
  data that cannot be collected without legal sign-off, and this is the surface where someone would
  reasonably expect to find it. Named so its absence reads as deliberate.

## Decisions

### D1 — Two item-boundary refinements, and `TS-BL-071` is wider than its `D.10` title

All under `D.11`'s permission for a feature's own propose conversation to refine internal item
boundaries. **No item moves between features, one specified dashboard leaves this feature entirely
because it is already built, and no new backlog ID is created.** Same treatment `candidate-intake`'s
`design.md` D13 gave the same class of refinement.

**(a) Candidate Timeline is not this feature's, and needs no new item.** `§20.1` lists it among five
dashboards, which is what makes it look like an orphan against `D.10`'s four items. It is not one:

- `§14.2` puts the timeline inside the **Candidate Profile** screen — *"Profile, resume versions,
  applications, interview history, scorecards, offers, **timeline**"* — not the **Reports** screen.
- `§13.2` exposes it as `GET /api/candidates/{candidateId}/timeline`, a candidate-scoped route, not a
  `/api/reports/*` route.
- `candidate-intake`'s `TS-BL-046` **already builds both**: its task 6.17 implements *"the candidate
  endpoints §13.2 enumerates — search, profile, **timeline** and resume listing"*, and its task 6.20
  builds *"the candidate profile and **timeline** screens"* on the design system's page templates.

`D13`'s old slice table listed *"candidate timeline"* under **both** `S6` (Candidate Intake) and `S13`
(Insight, Reporting & Export) — a genuine double-listing in a superseded table, which is the whole
reason it reads as unassigned now. `S6`'s successor claimed it and built it. That is also the right
home on the merits: a timeline is per-candidate and inherently PII-dense, so it belongs behind
`TS-BL-046`'s contact-field gating (its task 6.16, which verifies an Administrator with no active
break-glass grant receives no contact fields) rather than behind an aggregate surface whose entire
design premise (D3) is that it holds no candidate-identifying column at all. Putting it here would
require this feature to carry a PII-resolving path — the exact thing D3 forbids.

**(b) The SLA Aging Dashboard is absorbed into `TS-BL-073`**, which becomes *closure-readiness and
SLA aging*. Reasons, in order of weight:

1. **Two sibling features already assigned the aging views here by name** — `hiring-postings`'
   non-goal (*"the nudges and dashboards that act on them are `insight-and-reporting`"*) and
   `interview-pipeline`'s task 2.14 (*"aging drives `insight-and-reporting`'s SLA views"*). This is
   not a judgement call this conversation is making unaided; it is an obligation two verified changes
   recorded against this one.
2. **`§20.1` already couples aging to closure blockers.** The SLA Aging row's seven metrics are
   *"posting age, shortlist age, interview scheduling age, feedback delay, offer delay, onboarding
   delay, **and closure blockers**"* — its last element is `TS-BL-073`'s entire subject, and the same
   element also closes the Practice Manager row. Aging and closure-blocking are one question ("where
   is work stuck?") asked at two points in the pipeline, and they read the same projection (D5).
3. **The thresholds are already elsewhere, and correctly so.** `§29` item 9's aging thresholds are
   runtime configuration owned by `platform-core`'s `TS-BL-005` (its task 5.11 makes aging thresholds
   and notification templates runtime-configurable through the audited path), seeded per workflow by
   the features that own the aged records — `hiring-postings` for postings, `interview-pipeline`'s task
   2.14 for interviews and feedback. **This feature reads thresholds and never defines one.** So the
   `TS-BL-005` overlap the item boundary raises is not an overlap at all: `TS-BL-005` owns the
   *threshold*, `TS-BL-073` owns the *view*, and neither needs the other's half.

*Alternative considered and rejected:* absorbing aging into `TS-BL-072`, whose `§20.1` row does list
*"aging"* as its seventh metric. Rejected because it splits one specified dashboard across two items
and leaves `TS-BL-073` reading a projection it does not own. Instead `TS-BL-072` **consumes**
`TS-BL-073`'s projection, which adds one within-feature dependency edge `D.10` does not carry —
recorded in D5.

*Alternative considered and rejected:* new backlog items `TS-BL-081`/`082` for the two `§20.1` rows.
Rejected for Candidate Timeline because the work exists and a second item would duplicate it; rejected
for SLA Aging because a separate aging item would need its own copy of the projection `TS-BL-073`
already builds, and `D.11` prefers refining a boundary over adding an item where the work is genuinely
one cohesive slice.

**(c) `TS-BL-071` carries the feature's substrate as well as its dashboard.** Its `D.10` title reads
*"Recruiter/PM workload dashboard"*; it also covers `reporting/read-model` and
`reporting/report-export`, because it is the first reporting surface and there is no reporting
substrate for it to inherit. The alternative — a dashboard built directly on operational tables, with
the boundary and the export retrofitted under `TS-BL-072` — would mean the first surface ships without
the guarantee that is this feature's whole point. Exactly the reasoning `platform-core`'s `TS-BL-001`
and `design-system`'s `TS-BL-007` used for the same shape.

### D2 — These dashboards are numbers, ranked lists and tables. No charts, and no new visual primitive

**There is no charting capability anywhere in this product, and this is the feature where that becomes
real.** Verified rather than assumed, in both places it could live:

- `design-system`'s six items are the token set and primitives (`TS-BL-007`), the Authenticated Shell
  (`TS-BL-008`), the dense-data-table (`TS-BL-009`), the Floating Action Panel (`TS-BL-010`), form
  controls (`TS-BL-011`) and the notification toast (`TS-BL-012`). No chart, no sparkline, no
  visualization primitive.
- `reference/design-spec.md` and `reference/layout-spec.md` contain no chart, axis, series, or
  categorical-palette definition. `design-spec.md`'s only categorical color system is `§1.4`'s four
  semantic statuses, which mean **state**, not series — the same mismatch `S.3` already identified for
  evidence-source tagging.

**Decision: number-and-table-only.** Every metric renders as a value, a ranked list, or a row in
`TS-BL-009`'s dense-data-table, with `TS-BL-007`'s Badge and status-surface components carrying
threshold breaches. This is not a concession — it is the right form for what `§20.1` actually
specifies:

- **Every one of the five rows' metrics is a count, an age, a rate, or a status.** Not one is a time
  series or a distribution. *"Open postings"*, *"scorecards due"*, *"posting age"*, *"shortlist
  conversion"*, *"failure rates"* — a chart would render a single number as a picture of a single
  number.
- **These are operational worklists, not analytical exploration.** The question each answers is *"what
  do I act on next"*, and the action is to open the posting, the Application or the run. A table row
  is a link to the thing; a chart bar is not. `UI-003`'s requirement that every page show state,
  owner, next action and blockers is a table's native shape.
- **`S.5` sets the bar and a table can meet it.** *"Visually excellent and never overcompacted"* plus
  *"surface the single most important thing prominently, with supporting detail progressive/expandable"*
  is satisfied by a small set of headline values above a dense table, which is precisely
  `TS-BL-008`'s page-header-plus-content pattern with `TS-BL-009` in the content column.
- **`AGENTS.md`'s standing bar:** *"Reuse existing shared patterns before inventing new ones"*, naming
  the dense-data-table specifically.

**The condition under which this must be revisited, stated so it is not lost.** If a genuine
multi-series or trend visualization is ever wanted, it is **not** buildable here. A categorical series
palette is a new color family, and `design-spec.md` `§1.6` is explicit: *"Never introduce a new one-off
color for a single use case — extend the semantic set (§1.4) or the status-surface pattern (§1.5)
instead."* The precedent is already set: `S.6` resolved the missing sidebar colors by adding a named
token family to `design-system`'s `TS-BL-007` rather than letting a call site improvise. Charting would
take the same route — a `chart.series.*` token family and a chart primitive in `design-system`, as a
new backlog item in **that** feature. It is therefore recorded here as a condition, not as a
cross-feature obligation, because **nothing in this feature's specified scope requires it** and
`AGENTS.md`'s obligation table is for work that is actually owed.

*Alternative considered and rejected:* an in-cell magnitude bar — a single-series proportional fill
using the existing accent token, introducing no new color and so permitted by `§1.6`. Genuinely
tempting for aging and conversion columns, and rejected anyway: a shared cell renderer belongs to
`TS-BL-009`'s table pattern, and this feature has no authority to add a component to the design system.
If it is wanted it is a `design-system` obligation like any other, and the dashboards read correctly
without it.

### D3 — The read model holds no candidate-identifying column, and four named vectors are closed structurally

**This is the feature's primary design risk and its most important decision.** The boundary is not a
display filter, a code-review rule, or a note: it is a property of the projection every reporting query
runs against, with a test per vector.

**The projection.** All reporting queries execute against a declared read model whose column set is
enumerated in code. Candidate-identifying columns — name, email, phone, address, resume text, extracted
contact fields, and free-text reason bodies — **are not in it**. Where a reference is unavoidable, a
candidate appears as an opaque identifier only. A test enumerates the projection's columns and fails if
any resolves to a candidate personal-data field. This is the read-side mirror of what `platform/audit-trail`
does at write time and what `access-control/audit-review` does on the reader — *"a review surface that
helpfully resolved identifiers would reopen the back door the storage constraint closes"*. Three
features applying the same two-sided pattern is the pattern working, not duplication.

**Vector 1 — row-level drill-through.** Closed by making drill-through a **navigation, not a join**. A
report row's link carries the target's identifier and nothing else; the target screen belongs to the
owning feature and evaluates its own permission on that record. No reporting endpoint returns candidate
personal data under any parameter, filter or expansion. Directly modelled on `audit-review`'s
`Resolving a reference requires its own permission` scenario. The consequence is accepted deliberately:
a Recruitment Manager who can see *"3 candidates blocked on Hubble ID"* cannot see who they are from
this surface, and must open the posting — where `TS-BL-022`'s seeded read scope decides.

**Vector 2 — small-cohort aggregates.** Closed by suppression **with its differencing complement**. Any
aggregate whose contributing population falls below a configured floor is reported as *below reporting
threshold* rather than as a number. Naive suppression is trivially defeated — publish a total and all
but one part and the suppressed part is a subtraction — so the rule is applied to the complement too:
if suppressing one cell leaves it recoverable from the published total and siblings, the complement is
suppressed as well. The floor is audited runtime configuration (`§29`'s pattern), not a constant. A test
constructs the differencing attack against a published total and asserts the value cannot be recovered.
This has **no external anchor** — `§20.1` says nothing about cohort size — and is recorded here as a
decision this design makes, on `D16`'s reasoning that a boundary which is decorative is worse than an
absent one.

**Vector 3 — export carrying more than the screen.** Closed by **one projection, two renderers**. The
export is generated from the identical read model under the identical filters as the view that offered
it, so a column the screen omits cannot appear in a file. A test asserts the export's column set equals
the view's for the same filter set. Same shape as `audit-review`'s
`Export honors the same content constraint`.

**Vector 4 — the role posture itself.** Both Administrator roles hold no View on the candidate-derived
dashboards (`D16`; `TS-BL-022` task 5.8 seeds no View on candidate personal data), and the Auditor holds
`TS-BL-074` only, because `C-02` scopes that role to audit and AI run logs and not to candidate records.
Expressing this needs a page-catalog change — see D4.

*Alternative considered and rejected:* enforce the boundary in the presentation layer, letting the query
return whatever the operational tables hold. Rejected because `design-system`'s D1 makes every component
a pure renderer performing no permission evaluation, so the presentation layer is structurally the wrong
place — and because `AUTHZ-004` requires a direct API call to fail even where the UI hides the control,
which a display filter cannot deliver.

### D4 — One `Reports` page key cannot express the required posture — a cross-feature obligation

**Found in an already-verified sibling, recorded rather than worked around.**
`access-control-and-admin`'s `TS-BL-022` task 5.1 seeds *"a catalog entry for every screen in
`reference/spec.md` §14.2's inventory across every applicable action flag"*, and its `design.md` D5
confirms the granularity is one entry per `§14.2` screen. `§14.2` carries a **single** `Reports` screen,
whose capabilities are *"Practice dashboard, recruiter workload, candidate stage counts, aging, **AI
audit**"* — all four of this feature's items behind one key.

That key cannot hold the posture two decisions require:

- Granting the **Auditor** View on `Reports` to reach `TS-BL-074`'s AI Audit Dashboard also grants
  `TS-BL-071`'s and `TS-BL-072`'s candidate-derived aggregates, contradicting `C-02` and `TS-BL-022`'s
  own task 5.7 (*"Seed the Auditor read-only on audit and AI run records"*).
- Denying it leaves `TS-BL-074` unreachable for the only role it is built for.

There is no third option inside one key, and no override can fix it: `user_permission_overrides` are
per-user grants and denials on a page, so a deny lands on the whole `Reports` key too.

**Owed:** at minimum two catalog entries — operational reporting (`TS-BL-071`/`072`/`073`) and
governance reporting (`TS-BL-074`) — with `TS-BL-022`'s seed granting the Auditor View on the second and
denying it on the first. Recorded in `exploration-notes.md`'s **Pending cross-feature obligations**
table against `access-control-and-admin`, per `AGENTS.md`'s revised section. **This one must clear
before `/opsx:apply` reaches `TS-BL-071` or `TS-BL-074`**, because it decides what those items gate
against; it blocks no further propose work.

*Why this feature does not simply define its own page keys:* the catalog is seeded product-wide by one
item precisely so the matrix is not a moving target, and `TS-BL-022`'s task 5.10 routes the seeded
posture to the business owner for sign-off. A page key invented here would be outside that review.

### D5 — One aging-and-blocker projection, three readers, and the closure predicate is not re-derived

`TS-BL-073` owns a single projection carrying, per posting and per Application, the ages `§20.1`'s SLA
row names, the configured threshold each is measured against, and the closure blockers `CLS-004` names.
Three surfaces read it: `TS-BL-073`'s own view, `TS-BL-072`'s *"aging"* metric, and `TS-BL-071`'s
*"closure blockers"* element.

**The closure half is computed by calling `decision-and-offers`' predicate, never by re-deriving it.**
`decision/posting-closure` requires that *"the checklist and the guard on the closure action SHALL be
computed from the same predicate"*, with a scenario that inspects the codebase for a fulfilment
computation used by only one of the two and fails if it finds one. This feature adds a **third reader of
that predicate**, not a second predicate — a cross-posting aggregate over the same computation. A
reporting view that said *"ready to close"* while the closure action refused would be the exact defect
that requirement exists to make structurally impossible.

**Consequence — one dependency edge `D.10` does not carry:** `TS-BL-072` depends on `TS-BL-073` as well
as `TS-BL-071`. Without it, `TS-BL-072`'s aging metric is a second implementation of `TS-BL-073`'s
projection. A within-feature edge added under `D.11`, the same refinement `access-control-and-admin`
made adding `TS-BL-021` → `TS-BL-018`.

**Closed postings stay in scope.** `RET-003` retains closed-posting history visibly to authorized users
and `WF-007` requires closed postings to keep informing analytics, so the projection includes terminal
postings; excluding them would make every historical rate wrong.

### D6 — Resurfacing yield is built now, over a Phase 2 column, with no edge to Phase 5

**Status first: unratified.** *"Signature dashboards"* sits in `exploration-notes.md`'s
**Recommendations made, not yet formally decided** table — proposed, not objected to, never through an
explicit decision. Same status class as `D23a`, closure semantics and reason-on-every-disposition.
Nothing below cites it as settled, and the metric ships labelled as deriving from an unratified
recommendation.

**The apparent problem:** resurfacing yield is *"% of hires from the resurfacing pool"*, the resurfacing
pool is `resurfacing-and-communications`' `TS-BL-075`/`076` — the one feature not yet proposed — and
`D.10` gives `TS-BL-071` no edge to it. A Phase 4 surface appearing to need Phase 5 data, which is the
reverse of the usual direction.

**It dissolves on inspection.** The metric's substrate is not the resurfacing *engine*; it is
`G-10`'s `candidate_posting_applications.source_type`, whose `existing_database` value `G-10` itself
says *"is resurfacing, which makes the **resurfacing-yield metric computable** rather than inferred"*.
That column is created in **Phase 2** by `candidate-intake`'s `TS-BL-046` — its task 6.12 creates the
Application record *"carrying candidate, posting, submitting recruiter and `CAN-007`/`G-10`'s source
type"*, and its `design.md` D13(2) records that no other item among the eighty creates that table.

**So `D.10`'s missing edge is correct, not an oversight**, and this design adds none. The aggregation is
`hires grouped by source_type ÷ total hires`, defined entirely over columns that exist two phases
earlier.

**What Phase 5 changes is the data, not the code** — which is the third of the three options, chosen
deliberately: the surface is designed so the metric slots in without rework. Until `TS-BL-075` creates
resurfaced Applications, no row carries `existing_database`, and the surface renders a **distinct
empty state** — *"no resurfaced applications yet"* — rather than `0%`. That distinction is the design
content: `0%` asserts that resurfacing was tried and yielded nothing, which would be a false statement
about an unbuilt feature, and it is exactly the *"empty queue looks like no matches"* failure mode
`access-control-and-admin`'s D5 argues against choosing.

*Alternatives rejected:* scoping the metric out until Phase 5 (splits one dashboard across two phases
and needs a new item for the remainder); adding a `TS-BL-071` → `TS-BL-075` edge (inverts the phase
order, blocking a Phase 4 item behind a Phase 5 one, for data the metric does not structurally need).

### D7 — AI agreement rate ships whole; the activation history is an audit trail, not a field

Same unratified status as D6; same treatment. The recommendation names two measurements. Both are
supported, but by different substrates, and the second one's substrate is not where an earlier draft of
this decision looked for it.

> **Corrected 2026-08-27.** An earlier version of this decision deferred the second measurement,
> concluding that *"joining a disposition timestamp to a ranking version needs activation windows the
> spec does not record either."* **That was wrong.** It checked for an activation-window *field* and for
> version-creation timestamps, found neither, and stopped — without checking whether activation is
> recorded as an *event*. It is. The corrected reasoning is below, and it agrees with
> `matching-and-ranking`'s own `design.md` D4, which the earlier draft neither cited nor engaged with
> while reaching the opposite conclusion about the same mechanism.

**In scope — scorecard divergence.** *"How often approved scorecards diverge from AI drafts"* is
computable from the override records `ai-platform-governance`'s `Human override records` requirement
defines, whose `Divergence is measurable` scenario states that *"the rate at which humans diverged from
AI output can be computed from the stored records alone"*. Two producers already verify it end-to-end
for this consumer: `interview-pipeline`'s task 6.14 (*"the divergence rate between approved scorecards
and AI drafts is computable from stored override records alone... this is what makes the second half of
`insight-and-reporting`'s AI-agreement-rate dashboard possible"*) and `matching-and-ranking`'s task 5.11
for ranking overrides. **This feature consumes that path and adds no aggregation store and no second
override record** — `TS-BL-074` queries the existing records over a period, which is the operation the
scenario was written for.

**In scope — shortlist concordance.** *"How often humans shortlist the AI's top picks"* is **not an
override at all**, and that subtlety survives the correction and is worth keeping: a Practice Manager who
shortlists the candidate ranked seventh has overridden nothing — they made a decision the AI never gated,
so no override record is written and none should be. Concordance is therefore a **join**, not an
override query, and it needs the AI rank **as it stood when the disposition was made**.

Two facts about the substrate are true simultaneously, and holding only the first is what produced the
earlier error:

- **No ranking-version record carries an activation timestamp.** `matching/ranking-score`'s
  *"Exactly one ranking version is active per posting"* is a **current-state invariant**, not a history;
  its `New version activated` scenario describes the effect and records no time. There is no
  `activated_at`, and no version-creation timestamp either.
- **But every activation is an audited event, and the audit record carries the timestamp.**
  `platform/audit-trail`'s *"Audit on every material write"* covers it — activation writes
  `latest_rank`/`latest_match_score`, which `matching-and-ranking` D4 itself calls a denormalization
  whose *"single writer [is] activation of a ranking version"*. Its task 3.13 states the obligation
  outright: *"Write **every ranking activation** and every human approval through the audit path in the
  same transaction."*

**The query, and the requirement each step rests on:**

1. **When was each disposition made?** `platform/workflow-engine`'s *"Transition records"* — actor, prior
   state, new state, **timestamp**, correlation identifier, module — and `interview/shortlisting` makes
   every shortlist, removal and negative disposition a workflow transition rather than a direct state
   write.
2. **Which ranking version was active then?** The most recent ranking-activation audit record for that
   posting with timestamp ≤ the disposition's. `platform/audit-trail`'s *"Audit record content"* gives
   actor, action, target type and identifier, **previous and new values**, and an **immutable
   timestamp** — so the record names the version that became active *and* the one it replaced.
   `access-control/audit-review`'s *"Audit search"* makes exactly this query available: by target type,
   target identifier, action and **time range**, *"returned in chronological order"*.
3. **What was that Application's rank in it?** `matching/ranking-score`'s *"Prior board retrievable"*
   scenario — *"its full entry set is returned as it stood, with its own tuple values"* — combined with
   its rule that the application's rank is derived from the version rather than written independently,
   so rank within a past version is recoverable from that version's preserved entries. Its
   *"Regenerating a ranking creates a new version and preserves prior versions"* requirement is what
   guarantees the board is still there to read, and *"No ranking entry SHALL be updated in place"* is
   what guarantees it reads as it did.

**The `AIRun` route corroborates but does not carry this.** `ranking-score`'s tuple requirement puts an
**AI run reference** on every entry and refuses an entry with any tuple term absent, and `AI-002`
requires start and completion timestamps on every run — so a version's *creation* time is independently
recoverable. It is the weaker path: *"at most one active"* permits zero active versions and does not
guarantee that activation coincides with creation, so creation time is not activation time. The audit
trail is the load-bearing route; the run timestamps are a cross-check.

**This agrees with `matching-and-ranking`'s D4, which reached the right conclusion first.** D4 states
that *"`insight-and-reporting`'s AI-agreement-rate metric... needs the version that was current when the
human decided, not the current one. That query is available from the version table and unavailable from
two columns."* Correct, and the reason its argument holds is the version table plus the audited
activation event: the table alone establishes which boards exist, while the audit record establishes
which one was live at a given moment. D4's sentence names only the first half. That is imprecise rather
than false — its own task 3.13 mandates the audit — so it is recorded as a prose fix in the
cross-feature obligations table rather than treated as a defect.

**The earlier draft's objection inverts.** It argued that a metric which *"silently changes value when
someone re-ranks a posting is worse than an absent one"*. The opposite holds:
`platform/audit-trail`'s *"Append-only enforced by the database, not by the application"* means an
activation record cannot be amended, and `ranking-score` preserves the board it points at. So once a
period's concordance is computed it is **immutable** — a later re-rank writes a new activation record and
changes nothing about the earlier one. Stability was the objection, and the audit trail is what supplies
it.

**Three qualifications, built into the requirement rather than left to implementation.** Each is a case
where a naive join would manufacture a number:

- **No active version at disposition time.** *"At most one"* permits **zero** — a posting never ranked,
  or a candidate shortlisted before the first ranking completed. Those dispositions are **excluded from
  the denominator**, not counted as discordant. Counting them would penalise decisions the AI never
  informed, which measures the wrong thing in the direction that flatters the AI's absence.
- **Dispositions against a stale entry.** `ranking-score`'s staleness requirement marks an entry whose
  inputs have moved on, and its `Stale entry readable` scenario makes staleness visible. A disposition
  made against a stale board is reported **distinguishably** rather than pooled silently — the rank was
  computed against a superseded input, which is exactly the insufficiency posture `G-03` takes
  elsewhere.
- **Retention horizon.** `ranking-score`'s retention requirement disposes ranking versions on a
  configured policy, and `platform/audit-trail`'s retention is **separate**. So the activation events can
  outlive the boards they point at, and concordance is computable only back to the ranking-version
  boundary. The surface **reports its horizon** rather than returning a rate over a period whose boards
  are partly disposed.

**Consequence for dependencies:** `TS-BL-074` gains `TS-BL-051` (the ranking version record) and
`TS-BL-056` (the shortlist disposition) as edges. Both are Phase 2 and Phase 3 respectively, so no phase
order inverts, and `TS-BL-074` remains independent of this feature's other three items.

### D8 — Two endpoints beyond `§13.2`, and this feature's share of two enumerations closed

`§13.2`'s *Reports and Audits* block gives four routes this feature claims —
`GET /api/reports/practice-dashboard`, `GET /api/reports/recruiter-workload`, `GET /api/reports/aging`,
`POST /api/reports/export` — and two it does not: `GET /api/audit-logs` stays
`access-control-and-admin`'s and `GET /api/ai-run-logs` stays `ai-platform-governance`'s.

Two routes are **added**, with reasoning rather than by assumption:

- **A cross-posting closure-readiness route.** `§13.2` has only the per-posting
  `GET /api/postings/{postingId}/closure-checklist`, which is `decision-and-offers`'. `TS-BL-073` is a
  reporting view across postings, and iterating the per-posting route client-side would be N calls and,
  worse, would put the aggregation in the browser where D3's boundary cannot be enforced.
- **An AI-audit aggregate route.** `GET /api/ai-run-logs` is a record search, and `TS-BL-074` needs
  counts, rates and groupings over it. The aggregate is a distinct operation with a distinct
  permission, and `ai-platform-governance`'s requirement anticipates a reporting surface built *on* its
  search rather than served *by* it.

**Enumeration closure.** `interview-pipeline`'s `design.md` records as a residual risk that *"§25's
eleven job types and §13.2's endpoint list are the two most likely candidates [for an enumeration
nobody has counted against its owners], and neither has been audited feature-by-feature."* This feature
owns `§25`'s Report Export row and `§13.2`'s four `/api/reports/*` routes, and claims them explicitly
here so that share of the risk is closed rather than inherited.

### D9 — Report Export is `§25`'s eleventh row, registered on the existing chain

`§25` has **eleven** rows and one of them is already this: *"Report Export | User action | Retry export
build only when safe. | Export file and audit event."* Nothing new is invented, and `§25` stays at
eleven — the same discipline `decision-and-offers`' D11 applied when it declined to add a twelfth row
for its staleness nudge.

*A note on the count, since it was raised:* no sibling miscounts this. Every reference across the ten
proposed features says eleven — `decision-and-offers`' three uses of *"twelfth"* are all the correct
construction (*"without inventing a twelfth §25 row"*, i.e. there are eleven and adding one would make
twelve), and `interview-pipeline` says *"§25's eleven job types"* directly. There is no miscount to
correct and so no obligation recorded for one.

**Registration, not implementation.** Export becomes a job type in `platform-core`'s `TS-BL-006`
registry — its task 6.4 exists so *"a new job type is a registration rather than an infrastructure
change"*. This feature declares the type and its retry policy and implements the build; it implements
no queue, no retry loop and no status store, reading status from the substrate's job-status record the
way `TS-BL-005`'s task 5.9 does.

**"Only when safe" mapped onto the substrate's actual primitives.** `TS-BL-006`'s task 6.8 provides for
job types *"declared unsafe to retry, which record a failure for human action instead"*, and task 6.9
provides idempotency at the effect boundary. So: an export build **is** retry-safe while its artifact
has not been published, and **is not** once it has — a retry after publication would risk a second file
under the same export identity, and a partially written artifact must never be delivered. That is the
`§25` wording turned into a rule the dispatcher can act on rather than an adjective.

**What rides along:** `SEC-012`'s data classification label, `§20.2` item 16's export audit event
(*"Report and audit exports"*), `PRV-007`'s permission gate, `§29` item 13's configured export limits,
and `§32`'s rule that *"heavy reports shall run as background exports when query execution exceeds
interactive thresholds"* — which is what connects this decision to D10.

### D10 — Query-time projection against operational tables, with the export as the overflow path

**Decision:** the read model is a **query-time projection** — declared views and indexed access paths
over the operational tables — not a materialized reporting store, and not a warehouse.

*Why:*

- **A materialized copy of candidate-derived rows is a second PII surface** with its own access model,
  its own retention question against `RET-006`'s deletion workflows, and its own drift. D3's boundary is
  cheap to guarantee on a projection whose columns are declared once; it would have to be re-guaranteed
  on every refresh path of a copy.
- **`§24` makes the BI layer *optional***, and nothing in `§20.1` needs it.
- **The volume does not justify it.** `D07` fixes target volume at fewer than 5,000 candidates and
  fewer than 30 postings.
- **`§32` already supplies the escape hatch** for the case where it is not enough: heavy reports become
  background exports past the interactive threshold. So the answer to a slow report is D9's export
  path, not a second data store.

**The obligation this creates:** `NFR-001`'s three-second bound applies to these screens, so every
filter combination a reporting surface exposes must be supported by an indexed access path, with the
test suite failing on an unindexed combination rather than the query degrading silently — the identical
requirement `access-control/audit-review` sets for audit search, and for the identical reason. Where a
combination cannot be served interactively, it is offered as an export instead of being offered slowly.

### D11 — `TS-BL-074` inherits a PII-free substrate rather than re-implementing the boundary

The Auditor's surface is the one place in this feature where D3's projection work is largely already
done, and saying so prevents a redundant second boundary:

- `ai-platform-governance`'s `Reference-only inputs` requirement already guarantees run records store
  references, never resume text or contact data — *"required so an Admin's audit-log access
  (config-only) doesn't become a PII back door through AI run logs"*.
- Its `Run log access and search` requirement already guarantees readability *"by the Auditor role
  without granting access to candidate personal data"*.
- `access-control/audit-review` already guarantees the audit trail discloses no candidate personal data
  and that following a reference out of a record requires permission on that record.

So `TS-BL-074` aggregates over data that is PII-free **at source**, and its own obligations reduce to:
apply D3's small-cohort rule (an AI-run aggregate over one candidate's runs is still an aggregate over
one person), keep drill-through as navigation into `TS-BL-021`'s and `TS-BL-028`'s existing search
surfaces, and use `TS-BL-021`'s already-specified audited, classification-labelled **audit** export
rather than D9's **report** export. Two export paths, correctly: `§20.2` item 16 audits *"report and
audit exports"* as two things.

`§20.2` item 10 (*"Human override of AI output"*) is the audited event this dashboard reports on, and is
written by the producers named in D7 — not by this feature.

### D12 — Practice stays a string, but its vocabulary should be closed — answering a question `hiring-postings` deferred here

`hiring-postings`' Open Questions asks *"whether `practice` becomes a reference table or stays a
string"*, notes `§12.2` stores it as a string and `D02` makes Practice *"a data attribute on postings
for filtering and reporting. Not a permission boundary"*, and defers explicitly: *"`insight-and-reporting`
is the feature with a real stake, and it is unproposed."* It is proposed now, so this answers it.

**The stake is real.** Practice's stated purpose is filtering and reporting; `ADM-004` filters by
practice and `§32` makes it a candidate-search filter. Free text fragments every group-by it appears in
— *"Data & AI"*, *"Data and AI"* and *"data-ai"* become three practices — and a practice renamed later
silently splits its own history across two labels, which makes every historical rate on `TS-BL-072`
wrong in a way no one notices.

**But a reference table is not what fixes it.** The cheaper mechanism already exists in this product:
`interview-pipeline`'s `interview/shortlisting` holds the disposition reason vocabulary in *"the
platform's audited runtime configuration"*, where retiring a code leaves historical records readable
under the code they were recorded with. Practice needs exactly that shape — a closed, audited vocabulary
selected from rather than typed, with no schema change to postings and no new join.

**Recorded as a cross-feature obligation against `hiring-postings`, and it blocks nothing here.** These
dashboards group on whatever values exist; the fix improves grouping quality rather than enabling the
feature. Marked as such so it is not mistaken for a prerequisite.

## Risks / Trade-offs

- **[D3's boundary makes the dashboards deliberately less useful than an unconstrained one.]** A
  Recruitment Manager sees *"3 blocked on Hubble ID"* and must navigate to find out who. → Accepted,
  and it is the point. Mitigated by making drill-through one click to a screen that *does* resolve
  identity under its own permission, so the information is reachable by the audited path rather than
  absent.
- **[Small-cohort suppression will suppress legitimately useful cells in a small organization.]** At
  `D07`'s volumes — under 30 postings — a per-practice, per-recruiter breakdown can plausibly have
  cohorts of one or two, so suppression may fire often enough to frustrate. → Mitigated by making the
  floor audited runtime configuration rather than a constant, so the business can set it against its
  real distribution; and by suppressing the *cell*, not the row, so the surrounding aggregate still
  reads.
- **[No charts may read as unfinished to a stakeholder expecting a dashboard to look like one.]** →
  Mitigated by D2's reasoning being on the record with a named revisit route (a `design-system` token
  family plus primitive), so the conversation starts from "here is what it would cost and where it
  belongs" rather than from someone adding a chart library locally. Residual: a real risk that this is
  re-litigated at demo time.
- **[Two of this feature's metrics rest on an unratified recommendation.]** If *"Signature dashboards"*
  is ultimately rejected, resurfacing yield and AI agreement rate are built work with no mandate. →
  Mitigated by both being additive elements on dashboards whose `§20.1` rows stand on their own, so
  rejection removes an element rather than an item. Deliberately **not** mitigated by treating the
  recommendation as ratified.
- **[D10's query-time projection could miss `NFR-001` under real data despite `D07`'s volumes.]** →
  Mitigated by the indexed-access-path requirement with a failing test, and by `§32`'s export overflow
  as the specified answer. Residual: `platform-core`'s task 1.17 performance baseline is the thing that
  would detect it, and it is unbuilt.
- **[Four of the five dependency substrates are unbuilt when this feature is planned.]** `TS-BL-069`,
  `TS-BL-028`, `TS-BL-021` and `TS-BL-006` are all specified and none is implemented. → This is `D.4`'s
  accepted risk, not a new one. Mitigated by consuming each at a *written requirement* rather than at a
  guessed interface, and by citing the specific requirement in each case so an apply conversation can
  check it rather than re-derive it.
- **[D4's page-catalog obligation, if missed, ships a real disclosure defect rather than a nuisance.]**
  Granting the Auditor View on a single `Reports` key would hand a read-only governance role the
  candidate-derived aggregates. → Mitigated by recording it in the shared obligations table with an
  explicit *must clear before apply* marker, which is the one place every stage reads.

## Migration Plan

No data migration: this feature adds read paths and one job type, and alters no existing table.

**Order within the feature.** `TS-BL-071` first — it carries `reporting/read-model` and
`reporting/report-export`, which every later item inherits. Then `TS-BL-073` (the aging-and-blocker
projection), then `TS-BL-072` (which reads both). `TS-BL-074` is independent of all three **within this
feature**: it reads a different data domain through different substrates and shares no projection with
them, so it can proceed in parallel from the start — a genuine property of the design, not an ordering
convenience. Its cross-feature edges are a separate matter: D7 adds `TS-BL-051` and `TS-BL-056` to them,
both landing in earlier phases.

**Before apply.** D4's page-catalog obligation must clear, because `TS-BL-071` and `TS-BL-074` gate
against its outcome. D7's prose fix against `matching-and-ranking` and D12's vocabulary obligation do
not block.

**Rollback.** Every surface here ships behind a declared feature flag (`platform-core`'s task 1.12
mechanism, where a disabled capability answers 404), so a dashboard can be withdrawn without a
deployment. The export job type is a registration; withdrawing it leaves the dispatcher's other types
untouched.

**Degradation.** `G-12`/`NFR-004` are satisfied by construction: every number here is read from a stored
record, so an AI provider, vector store or notification outage leaves all four surfaces fully
functional. The one real dependency is the database. Worth stating because a "dashboard" is where
someone would reasonably expect a live AI call, and there is none.

## Open Questions

Genuinely deferrable — none changes a spec, the approach, or the task breakdown.

- **The small-cohort floor's value.** D3 fixes the mechanism, the complement rule and the audited
  configuration path; the number needs the organization's real practice and recruiter distribution,
  which does not exist yet. A configuration value, not a requirement.
- **Which aggregates the business wants on the Practice Manager dashboard first.** `§20.1`'s row gives
  seven elements with no priority, and `S.5` requires surfacing *"the single most important thing
  prominently"* — which one that is, is a product call. Affects layout ordering, not the projection.
- **Whether `TS-BL-072` needs per-practice grouping before D12's vocabulary lands.** Grouping works on
  the values present either way; whether the fragmented view is worth shipping in the interim is a
  judgement the first real data makes easy and speculation makes hard.
- **Whether the SLA views should also emit the nudges `hiring-postings` mentions.** Its non-goal says
  *"the nudges **and** dashboards that act on them are `insight-and-reporting`"*, and its own Open
  Questions leaves the RM stalled-review nudge undecided. A nudge is a `platform-core` notification off
  the same projection this feature builds, so it is additive whenever it is decided — and
  `decision-and-offers`' D11 already established the pattern (a scheduled sweep on the existing chain,
  registering no new `§25` job type). Not specified here because nothing has decided it.
- **`OD-003`, `OD-004` and `OD-005`** are open and none of them shapes this feature: no surface here
  calls a model provider, retention affects how much history exists rather than how it is aggregated,
  and nothing here is candidate-facing.

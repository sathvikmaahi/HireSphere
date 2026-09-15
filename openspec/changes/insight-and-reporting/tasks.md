# Insight & Reporting — Backlog

This feature's complete backlog: four items, `TS-BL-071` through `TS-BL-074`, decomposed in
`exploration-notes.md` D.10's Phase 4 table. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed *before*
the sprint/wave schedule is redone, in its own conversation against `delivery/`. Assigning sprints here
would be inventing a schedule this feature has no authority to set. Each item carries a stated goal
instead.

**Nothing here is built,** and nothing this feature reads is built either — `TS-BL-069`, `TS-BL-028`,
`TS-BL-021` and `TS-BL-006` are all specified and none is implemented. That is `D.4`'s accepted risk.
Every consumption point below cites the **written requirement** it consumes rather than a guessed
interface, so an apply conversation can check it instead of re-deriving it.

**Four dependency edges `D.10`'s table does not carry**, all added under `D.11`. Two are within this
feature: `TS-BL-072` → `TS-BL-073` (the aging projection has one owner and several readers,
`design.md` D5), and `TS-BL-072`/`TS-BL-073` → `TS-BL-071` (the read model and the export path are
`TS-BL-071`'s, `design.md` D1(c)). Two are cross-feature, on `TS-BL-074`: `TS-BL-051` and `TS-BL-056`,
which shortlist concordance joins (`design.md` D7). `TS-BL-074` still shares no projection with this
feature's other three items and can proceed in parallel with them from the start.

**One dependency edge deliberately *not* added.** `TS-BL-072` carries resurfacing yield and takes **no**
edge to `resurfacing-and-communications`' `TS-BL-075`. The metric's substrate is `G-10`'s application
source type, created in Phase 2 by `candidate-intake`'s `TS-BL-046`, not the Phase 5 resurfacing engine.
`D.10`'s omission is correct; `design.md` D6 records why.

**Blocking obligation before apply.** `design.md` D4's page-catalog granularity fix in
`access-control-and-admin` must clear before `/opsx:apply` reaches `TS-BL-071` or `TS-BL-074` — those
items gate against its outcome. Recorded in `exploration-notes.md`'s **Pending cross-feature
obligations** table. D7's prose fix against `matching-and-ranking` and D12's vocabulary obligation block
nothing.

---

## 1. TS-BL-071 — Reporting read model, report export, and the Practice Manager workload dashboard

**Goal:** the first reporting surface in the product, and with it the boundary every later one inherits —
an aggregate read model that structurally cannot disclose candidate PII, an export that carries exactly
what the screen carries, and the Practice Manager's operational worklist on top of both. Covers
`reporting/read-model`, `reporting/report-export` and `reporting/workload-dashboard`.

**Scope note: this item is wider than its `D.10` one-line title,** which reads "Recruiter/PM workload
dashboard". It also carries the read model and the export path, because it is the first reporting item
and there is no reporting substrate to inherit — a dashboard built directly on operational tables with
the boundary retrofitted later would ship the first surface without the guarantee that is this feature's
whole point. `design.md` D1(c); the same reasoning `platform-core`'s `TS-BL-001` and `design-system`'s
`TS-BL-007` recorded for the same shape.

```yaml
backlog_items:
  - id: TS-BL-071
    feature: insight-and-reporting
    depends_on: [TS-BL-069, TS-BL-009, TS-BL-018, TS-BL-006]
    status: not-started
```

**On the dependency list.** `D.10` gives `TS-BL-069` and `TS-BL-009`. Two are added under `D.11`:
`TS-BL-018` (no reporting surface can be gated or tested without the permission evaluator — the same
edge `access-control-and-admin` added to its `TS-BL-021`, and the shape Sprint 0's handover flagged for
its own task 2.7) and `TS-BL-006` (the export dispatches on the shared async substrate rather than
defining its own retry mechanism — the same edge `platform-core`'s `TS-BL-005` carries for the same
reason).

- [ ] 1.1 Declare the reporting projection's column set as typed data, and add the test that fails if any
      column resolves to a candidate personal-data field — name, contact field, address, resume text,
      extracted content, or a free-text reason or note body (`D16`, `design.md` D3)
- [ ] 1.2 Build the projection as query-time views and indexed access paths over the operational tables,
      with **no materialized reporting store and no second copy** of candidate-derived rows
      (`design.md` D10; `§24` makes the BI layer optional and `D07` fixes volume under 5k candidates)
- [ ] 1.3 Include postings and Applications in terminal states in the projection, and assert a rate
      computed over a period containing filled and closed postings includes them (`RET-003`, `WF-007`)
- [ ] 1.4 Implement candidate references as opaque identifiers, and assert no reporting endpoint returns
      candidate personal data under **any** parameter, filter or expansion — including one that asks for
      it explicitly (`AUTHZ-004`)
- [ ] 1.5 Implement drill-through as a reference the reader navigates, resolved by the owning feature
      with the evaluator evaluating that record on its own terms; add the test that fails if the
      reporting surface joins the projection to candidate personal data
      (`access-control/audit-review`'s equivalent scenario, `design.md` D3 vector 1)
- [ ] 1.6 Implement small-cohort suppression against a floor held in `platform-core`'s audited runtime
      configuration, reporting *below reporting threshold* rather than a value, and assert a cohort of
      one is suppressed
- [ ] 1.7 Implement the differencing complement: where suppressing a cell leaves it recoverable from a
      published total and its siblings, suppress the recovering values too. **Write the attack as the
      test** — construct a total plus all-but-one part and assert the omitted value cannot be recovered
      (`design.md` D3 vector 2; no external anchor, this design's own decision)
- [ ] 1.8 Establish the single-aggregation-path rule in code: each metric computed once and consumed by
      every surface, with the test that fails if a metric is computed in two places (`NFR-009`)
- [ ] 1.9 Index every filter combination the surface exposes and assert an unindexed combination **fails
      the suite** rather than degrading silently — the identical requirement
      `access-control/audit-review` sets for audit search, for the identical reason (`NFR-001`,
      `NFR-005`)
- [ ] 1.10 Route a combination that cannot meet the interactive bound to the export path rather than
      serving it slowly (`§32`'s "heavy reports shall run as background exports when query execution
      exceeds interactive thresholds")
- [ ] 1.11 Register **report export** as a job type against `TS-BL-006`'s registry — a registration, not
      an infrastructure change (its task 6.4) — implementing no queue, no retry loop and no job-status
      store, and reading status from the substrate's record (the precedent `TS-BL-005`'s task 5.9 sets)
- [ ] 1.12 Declare the retry policy as retry-safe **only until the artifact is published**, and unsafe
      afterwards, recording a failure for human action instead (`§25`'s "retry export build only when
      safe" mapped onto `TS-BL-006`'s task 6.8; `design.md` D9)
- [ ] 1.13 Implement idempotency at the export's effect boundary so a duplicate Pub/Sub delivery
      produces exactly one artifact per export identity, and assert no partially written artifact is
      ever delivered (`TS-BL-006` task 6.9 — at-least-once delivery with no native deduplication to
      inherit)
- [ ] 1.14 Generate the export from the **same projection under the same filters** as the view that
      offered it, and assert the export's column set equals the view's for one filter set
      (`design.md` D3 vector 3)
- [ ] 1.15 Apply suppression to exported content identically to on-screen content, and assert a
      suppressed aggregate is suppressed in the file
- [ ] 1.16 Gate export on the Export action permission through the evaluator, supplying the decision to
      `TS-BL-009`'s export control — which per its task 3.5 is caller-supplied and absent by default,
      and performs no evaluation itself (`PRV-007`, `UI-008`)
- [ ] 1.17 Attach `SEC-012`'s data classification label to every artifact, and write `§20.2` item 16's
      export audit event naming actor, report, time and filter — by reference, with no candidate personal
      data in the audit record itself
- [ ] 1.18 Enforce `§29` item 13's configured export limit, refusing an over-limit request with the limit
      and requested size named, and **never truncating silently** — the "silent success" class Sprint 0's
      handover identifies as the dangerous one
- [ ] 1.19 Audit a failed export attempt and deliver no artifact
- [ ] 1.20 Build the dashboard's seven `§20.1` elements — open postings, candidates by stage, pending
      interviews, scorecards due, selected candidates, offer status, closure blockers — each carrying the
      reference needed to navigate to the work it names (`UI-003`)
- [ ] 1.21 Read the closure-blockers element from `TS-BL-073`'s projection, and assert this item contains
      **no** fulfilment or blocker computation of its own — `decision/posting-closure` requires one
      predicate with a scenario that fails on a second, and this is a third reader
- [ ] 1.22 Apply read scope as a **query predicate** through the evaluator, never by discarding fetched
      rows, and verify a permission change is reflected on the next load with no cache serving the
      previous scope (`C-03`, `candidate-intake` task 6.17's precedent)
- [ ] 1.23 Build the surface from `TS-BL-007`'s tokens and primitives and `TS-BL-009`'s dense-data-table,
      adding no component; assert no chart, sparkline or visualization primitive and no locally defined
      color or spacing value exists (`design.md` D2, `design-spec.md` §1.6)
- [ ] 1.24 Render headline values prominently with supporting detail expandable rather than fully
      displayed at once, and give every element an explicit empty state rather than a bare zero (`S.5`)
- [ ] 1.25 Implement `GET /api/reports/practice-dashboard` and `POST /api/reports/export` (`§13.2`), each
      declaring its permission requirement, and deny both Administrator roles and the Auditor on the
      candidate-derived surface (`D16`, `C-02`, `TS-BL-022` task 5.8)
- [ ] 1.26 Ship the surface behind a declared feature flag, where a disabled capability answers 404
      (`platform-core` task 1.12), so a dashboard can be withdrawn without a deployment
- [ ] 1.27 Verify the whole surface renders from stored records with the AI provider, vector store and
      notification service unavailable, and assert no call to any of them exists (`G-12`, `NFR-004`)
- [ ] 1.28 Verify keyboard traversal and visible focus across the dashboard and its table at the compact
      density (`UI-009`)

---

## 2. TS-BL-072 — Recruitment Manager pipeline oversight dashboard

**Goal:** the oversight lens `C-02` gives the Recruitment Manager — who is carrying what, how the funnel
converts, where it slows — plus resurfacing yield, built now over a column that already exists. Covers
`reporting/pipeline-oversight`.

```yaml
backlog_items:
  - id: TS-BL-072
    feature: insight-and-reporting
    depends_on: [TS-BL-071, TS-BL-073]
    status: not-started
```

**On the added edge.** `D.10` gives only `TS-BL-071`. `TS-BL-073` is added because `§20.1` lists "aging"
on this dashboard's row *and* gives the SLA Aging Dashboard its own row; without the edge this item
reimplements `TS-BL-073`'s projection. A within-feature refinement under `D.11`, the same move
`access-control-and-admin` made adding `TS-BL-021` → `TS-BL-018` (`design.md` D5).

- [ ] 2.1 Build the seven `§20.1` elements — recruiter workload, postings, candidates submitted,
      shortlist conversion, interview scheduling, offer progress, aging
- [ ] 2.2 Derive recruiter workload from posting assignment (`job_postings.recruiter_ids[]`), not from
      any separately recorded judgement, and name recruiters as the internal users they are
- [ ] 2.3 Compute shortlist conversion over Applications dispositioned within the requested period,
      including those on postings since closed (`RET-003`, `WF-007`)
- [ ] 2.4 Read the aging element from `TS-BL-073`'s projection, and assert this item contains no stage-age
      or threshold computation of its own
- [ ] 2.5 Compute **resurfacing yield** as the share of hires whose Application source type records the
      existing candidate database, from `G-10`'s `source_type` created by `candidate-intake`'s
      `TS-BL-046` (its task 6.12) — and assert **no dependency** on the resurfacing engine, the
      suggestion pool or the priority lane exists (`design.md` D6)
- [ ] 2.6 Render the distinct *"no resurfaced applications yet"* state where no Application carries the
      existing-database source type, and assert it is **not** rendered as `0%` — a zero would assert that
      resurfacing was tried and yielded nothing, which is a false statement about an unbuilt feature
- [ ] 2.7 Label resurfacing yield as deriving from a recommendation that has not been through an explicit
      decision — `exploration-notes.md`'s **Recommendations made, not yet formally decided** table, the
      same status class as `D23a` and closure semantics
- [ ] 2.8 Assert no candidate name or contact detail appears in any element, and that a per-recruiter or
      per-practice breakdown producing a cohort below the floor is suppressed cell-wise
      (`reporting/read-model`, `design.md` D3)
- [ ] 2.9 Render conversion and aging as values and table rows on `TS-BL-009`'s pattern; assert no chart,
      sparkline or visualization primitive exists — these two elements are the most likely in the product
      to attract one, so the assertion belongs on this surface specifically (`design.md` D2)
- [ ] 2.10 Implement `GET /api/reports/recruiter-workload` (`§13.2`), declaring its permission
      requirement and inheriting `TS-BL-071`'s boundary rather than restating it
- [ ] 2.11 Ship behind a declared feature flag (`platform-core` task 1.12)
- [ ] 2.12 Verify keyboard traversal and visible focus across the surface (`UI-009`)

---

## 3. TS-BL-073 — Closure-readiness and SLA aging reporting view

**Goal:** one projection answering "where is work stuck?", read by every surface that shows an age, a
threshold breach or a closure blocker — computed from `decision-and-offers`' closure predicate and the
platform's configured thresholds, and owning neither. Covers `reporting/closure-and-aging`.

**Scope note: this item absorbs `§20.1`'s SLA Aging Dashboard,** which `D.10`'s four titles do not name.
Not this conversation's unaided judgement — two verified sibling features assigned it here by name:
`hiring-postings`' non-goal (*"the nudges and dashboards that act on them are `insight-and-reporting`"*)
and `interview-pipeline`'s task 2.14 (*"aging drives `insight-and-reporting`'s SLA views, not the state
machine"*). `§20.1`'s SLA row also ends with *"closure blockers"*, which is this item's original
subject. `design.md` D1(b) records the reasoning and the rejected alternatives.

```yaml
backlog_items:
  - id: TS-BL-073
    feature: insight-and-reporting
    depends_on: [TS-BL-069, TS-BL-071]
    status: not-started
```

- [ ] 3.1 Build the single aging-and-blocker projection carrying, per posting and per Application, the
      stage ages `§20.1`'s SLA row names — posting age, shortlist age, interview scheduling age, feedback
      delay, offer delay, onboarding delay — the threshold each is measured against, and the closure
      blockers
- [ ] 3.2 Read every threshold from `platform-core`'s audited runtime configuration (`§29` item 9, seeded
      by `hiring-postings` for postings and `interview-pipeline`'s task 2.14 for interviews and feedback);
      add the test that fails if a threshold value appears in source
- [ ] 3.3 Report an age without a breach determination where its threshold has no configured value,
      rather than against an invented default — and do not invent threshold values here
- [ ] 3.4 Verify a threshold change through the audited path with a mandatory reason is reflected in the
      next projection **without a deployment** (`D22` caps configurable windows by retention)
- [ ] 3.5 Compute cross-posting closure readiness by **invoking** `decision/posting-closure`'s fulfilment
      predicate across postings; add the test that fails if this item implements a second fulfilment
      computation — that spec's own scenario inspects for exactly this
- [ ] 3.6 Verify this surface reporting a posting ready to close agrees with the closure action
      succeeding for that posting, and that each posting's readiness equals what its own per-posting
      checklist reports (`CLS-002`, `CLS-004`)
- [ ] 3.7 Name blocking conditions by kind — missing Hubble ID, or unresolved salary, offer or onboarding
      state — as counts and conditions **without candidate identity**, with identification requiring
      navigation to the posting's own checklist where permission is evaluated on the Application
      (`CLS-003`, `CLS-004`, `D16`)
- [ ] 3.8 Add the test that fails if any state transition is triggered by an age or a threshold breach
      computed here — `interview-pipeline`'s task 2.14 asserts the same prohibition from its side, and
      `decision/posting-closure` requires closing to be actor-initiated and never fired by "a timer, a
      threshold, a data condition or a configuration change"
- [ ] 3.9 Include terminal postings in the projection so historical ages remain readable (`RET-003`,
      `WF-007`)
- [ ] 3.10 Expose the projection to `TS-BL-071`'s closure-blockers element and `TS-BL-072`'s aging
      element, and verify all three surfaces report consistent values for the same filters
- [ ] 3.11 Implement `GET /api/reports/aging` (`§13.2`) and the cross-posting closure-readiness route
      `§13.2` does not name — added with reasoning in `design.md` D8, because iterating the per-posting
      route client-side would be N calls and would move aggregation to the browser where the boundary
      cannot be enforced
- [ ] 3.12 Render ages and breaches as values and table rows, with breaches carried by `TS-BL-007`'s
      badge and status-surface components; assert no chart, sparkline or **magnitude bar** exists — the
      bar was considered and declined because a shared cell renderer belongs to `TS-BL-009`'s pattern,
      not to a consuming feature (`design.md` D2)
- [ ] 3.13 Ship behind a declared feature flag (`platform-core` task 1.12)
- [ ] 3.14 Verify keyboard traversal and visible focus across the surface (`UI-009`)

---

## 4. TS-BL-074 — Audit / AI-run reporting surface for the Auditor

**Goal:** the Auditor's governance view over AI activity — what ran, on which versions, how often it
failed, how often a human diverged — aggregated over substrates that are already PII-free at source,
adding no second store and no second override record. Covers `reporting/ai-audit-reporting`.

```yaml
backlog_items:
  - id: TS-BL-074
    feature: insight-and-reporting
    depends_on: [TS-BL-028, TS-BL-021, TS-BL-051, TS-BL-056]
    status: not-started
```

**Independent of the other three items in this feature, deliberately.** It shares no projection with
them, reads a different data domain, and uses a different export path (`TS-BL-021`'s audit export, not
`TS-BL-071`'s report export). A property of the design (`design.md` D11), not an ordering convenience —
it means this item can proceed in parallel with `TS-BL-071`–`073` from the start.

**Two cross-feature edges `D.10` does not carry**, added under `D.11` by `design.md` D7's correction:
`TS-BL-051` (the ranking version record and its preserved prior boards) and `TS-BL-056` (the shortlist
disposition, whose workflow transition supplies the timestamp). Shortlist concordance joins the two, so
neither can be built against here without them. Both land in Phase 2 and Phase 3 respectively, so no
phase order inverts.

- [ ] 4.1 Build the seven `§20.1` elements — AI runs, prompt versions, model versions, outputs, human
      overrides, failure rates, flagged outputs — grouping by the versions recorded on the run records
      themselves
- [ ] 4.2 Compute every element from `TS-BL-028`'s stored run, disclosure and override records and
      `TS-BL-021`'s audit records, using the search `ai-platform/ai-run-logging`'s `Run log access and
      search` requirement already provides — filterable by family, template version, model version,
      status, safety flag and time range. That requirement's own source note names this item as its
      reporting surface; consume it rather than adding a query path
- [ ] 4.3 Assert this item's persistence surface contains **no** aggregation table, no override record
      and no run log of its own, and that computing any element creates, modifies or removes no run,
      disclosure, override or audit record — the run log is append-only and reporting is a read
- [ ] 4.4 Compute the human override rate over a requested period from stored override records alone,
      with **no content comparison** — the operation
      `ai-platform/ai-run-logging`'s `Divergence is measurable` scenario was written for (`G-06`,
      `RANK-006`, `BR-016`)
- [ ] 4.5 Compute the scorecard-divergence half of AI agreement rate, distinguishing an approval with no
      adjustment from one with an adjustment — the property `interview-pipeline`'s task 6.14 verifies from
      its side for this consumer
- [ ] 4.6 Verify the original AI output remains retrievable for every override contributing to the rate
      (`G-06` — the override deletes nothing)
- [ ] 4.7 Compute **shortlist concordance** as a join, not an override query — a PM who shortlists rank
      seven overrode nothing, so no override record exists to read. For each disposition: take its
      timestamp from `platform/workflow-engine`'s transition record, resolve the ranking version active
      at that time from the **audited ranking-activation event** (`platform/audit-trail`'s record content
      gives previous and new values plus an immutable timestamp; `access-control/audit-review`'s search
      gives target-plus-action-plus-time-range in chronological order), then read that Application's rank
      from that version's preserved entries (`matching/ranking-score`'s `Prior board retrievable`)
- [ ] 4.8 Assert concordance is **never** computed against the currently active ranking version, and add
      the regression test that recomputing a past period after any number of subsequent re-ranks returns
      the **same value** — `platform/audit-trail`'s database-enforced append-only is what makes a past
      rate immutable, which is the property the earlier deferral wrongly assumed was missing
      (`design.md` D7)
- [ ] 4.9 Exclude dispositions made when the posting had **no active ranking version** from the
      denominator rather than counting them as discordant — `matching/ranking-score` permits zero active
      versions, and counting a decision the AI never informed measures the wrong thing in the direction
      that flatters the AI's absence. Report the excluded count alongside the rate rather than implicitly
- [ ] 4.10 Report contributions scored against a **stale** ranking entry distinguishably, and surface the
      stale share rather than absorbing it into the headline rate (`matching/ranking-score`'s staleness
      requirement and its `Stale entry readable` scenario; `G-03`'s insufficiency posture)
- [ ] 4.11 Bound concordance by the **ranking-version retention horizon**, reporting the horizon where a
      requested period reaches earlier rather than returning a rate over partly disposed boards; and
      treat a retained activation event whose ranking version has been disposed of as beyond the horizon,
      not as no-active-version (`matching/ranking-score`'s retention requirement, whose interval is
      `OD-004`'s — do not invent one)
- [ ] 4.12 Label the AI-agreement-rate elements as deriving from a recommendation that has not been
      through an explicit decision, the same treatment task 2.7 applies to resurfacing yield
- [ ] 4.13 Gate the surface on its own View permission and make it available to the Auditor; assert the
      Auditor resolves to denied for Create, Edit, Delete, Approve and Assign on this surface and every
      record it references (`C-02`, `TS-BL-022` task 5.7)
- [ ] 4.14 Assert no candidate personal data is rendered or returned, inheriting
      `ai-platform/ai-run-logging`'s `Reference-only inputs` guarantee rather than re-implementing a
      boundary; and make following a reference out of an aggregate reach `TS-BL-021`'s or `TS-BL-028`'s
      existing search surface with permission evaluated there (`design.md` D11)
- [ ] 4.15 Apply small-cohort suppression to per-candidate and per-actor aggregates — an aggregate over
      one candidate's runs is still an aggregate over one person, and a single-actor override rate would
      make this a per-person performance measure (`design.md` D11, `PRV-001`)
- [ ] 4.16 Route export through `TS-BL-021`'s existing audited, classification-labelled **audit** export,
      and assert no registration against the report export job type exists — `§20.2` item 16 audits
      "report and audit exports" as two things
- [ ] 4.17 Implement the AI-audit aggregate route `§13.2` does not name, added with reasoning in
      `design.md` D8; `GET /api/ai-run-logs` remains `ai-platform-governance`'s and `GET /api/audit-logs`
      remains `access-control-and-admin`'s, and this surface aggregates over them rather than replacing
      either
- [ ] 4.18 Build the surface on `TS-BL-009`'s dense-data-table — which its task 3.7 already verifies
      renders inside a detail view's main column, "which is how the audit log will use it" — adding no
      component and asserting no chart or visualization primitive exists
- [ ] 4.19 Ship behind a declared feature flag (`platform-core` task 1.12)
- [ ] 4.20 Verify keyboard traversal and visible focus across the surface at the compact density
      (`UI-009`)

---

## Cross-feature obligations raised by this feature

Recorded in `exploration-notes.md`'s **Pending cross-feature obligations** table, per `AGENTS.md`'s
revised "Correcting another feature's artifacts" section — each is a `/opsx:update` run against the named
change, in its own conversation. Listed here so this feature's own apply conversation sees them too.

| Owed by | What | Blocks apply here? |
|---|---|---|
| `access-control-and-admin` | Page-catalog granularity for the reporting surfaces. `TS-BL-022` task 5.1 seeds one entry per `§14.2` screen, and `§14.2`'s single **Reports** screen spans both the practice dashboard and AI audit. One key cannot grant the Auditor `TS-BL-074` without also granting `TS-BL-071`/`072`'s candidate-derived aggregates, nor deny it without making `TS-BL-074` unreachable. `design.md` D4. | **Yes** — `TS-BL-071` and `TS-BL-074` gate against its outcome |
| `matching-and-ranking` | Prose fix in its `design.md` D4. Its conclusion is correct — the "version current when the human decided" query *is* available — but it attributes that to the version table alone, which establishes which boards exist and not which was active at a given moment. The audited ranking-activation event supplies the second half, as its own task 3.13 already requires. Sharpen the sentence so an apply conversation reading D4 alone does not conclude the version table is sufficient. `design.md` D7. | No |
| `hiring-postings` | Close the `practice` vocabulary — an audited runtime-configuration list selected from rather than free text, the shape `interview/shortlisting` uses for disposition reason codes — so reporting can group on it reliably. Answers the question `hiring-postings`' Open Questions explicitly deferred to this feature. `design.md` D12. | No |

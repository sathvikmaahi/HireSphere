# Interview Pipeline — Backlog

This feature's complete backlog: eight items, `TS-BL-056` through `TS-BL-063`, decomposed in
`exploration-notes.md` D.10. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built, and nothing was inherited.** `talentsphere-wave-1-foundation` has no
`interview/` delta spec, none of its `design.md` D1–D18 was allocated here (`platform-core` D11
accounts for all eighteen elsewhere), Sprint 0 shipped platform-layer code only with no domain state
machine of any kind, and `KNOWN_ISSUES.md` carries nothing about shortlisting, interviews, notes or
scorecards. Checked rather than assumed — see `design.md` D13.

**Build `TS-BL-059` against `C-08`, not `D24`.** `C-08` amends `D24` with three separate reversals and
it is possible to satisfy one and miss the others: **no lock ever**, **no author-only rule** (the
`Edit` action on the Interview Console page decides it, seeded to Interviewer), and **edits are edits
that version**, not appended addenda. `design.md` D8 tabulates all three. `D24`'s own body still reads
*"Satisfies the D11 requirement that notes lock on scorecard approval"* — stale, and `D11`'s matching
consequence was un-annotated until this conversation (`design.md` D14a).

**`TS-BL-061` builds the drift mechanism, and that is a decision this feature made.**
`exploration-notes.md` records supersede-on-post-approval-note-edit as recommended and **"Not yet
decided."** `design.md` D3 decides it — build it — and records the two rejected alternatives.
`TS-BL-059` emits the signal, `TS-BL-060` records the note-version set that scopes it, `TS-BL-061`
implements it. Building any one of the three without the others produces a mechanism that looks
present and never fires.

**Two AI families had no owner and now do.** `D.10`'s dependency table wires neither `TS-BL-058` nor
`TS-BL-059` to `ai-platform-governance`'s `TS-BL-027`, which left `interview_questions` and
`interview_note_summary` — two of the ten registered families — authored by nobody. Both edges are
added here under `D.11`'s permission (`design.md` D1). **With `scorecard_generation`, this feature
authors three families and the ten are closed**, `resurfacing` being the only one whose owner
(`TS-BL-075`) is still unproposed.

**"Reason required on every disposition" is treated as adopted, not cited as settled.** It comes from
the *"Recommendations made, not yet formally decided"* table — the same status class `candidate-intake`
handled by name for `D23a`. `design.md` D2 separates the settled half (`WF-005` already requires a
reason on every negative transition, independently) from the unratified half (the coded vocabulary,
which lives in audited configuration so rejecting it costs a configuration change).

**Reuse, do not rebuild, six mechanisms that already exist.** `matching-and-ranking`'s **ranking-context
projection** (`matching/ai-context-visibility`) is what the console renders — it builds no second view
of ranking output; `ai-platform-governance`'s **override record** is what `SCR-007`/`BR-016` writes to;
its **closed evidence-source vocabulary** is where every source label comes from; its **run log** and
**gateway** carry all three families; `platform-core`'s **workflow service** executes every transition;
and `matching-and-ranking`'s **vector source-type seam** takes both of this feature's registrations
without a schema change (`design.md` D6, D11).

**Dependency edges pointing outside this feature.** `TS-BL-056` needs `matching-and-ranking`'s
`TS-BL-053` and `platform-core`'s `TS-BL-004`; `TS-BL-058`, `TS-BL-059` and `TS-BL-060` each need
`ai-platform-governance`'s `TS-BL-027` (two of those three added here — `design.md` D1). Eight further
interfaces are built against without being `depends_on` edges: `platform-core`'s `TS-BL-002` (audited
runtime configuration), `TS-BL-005` (notification delivery) and `TS-BL-006` (dispatch);
`access-control-and-admin`'s `TS-BL-018` (evaluator), `TS-BL-020` (durable audit writer) and
`TS-BL-022` (seeded matrix); `ai-platform-governance`'s `TS-BL-028`, `TS-BL-030`, `TS-BL-031` and
`TS-BL-032`; `matching-and-ranking`'s `TS-BL-049` (source-type seam) and `TS-BL-054` (projection);
`candidate-intake`'s `TS-BL-046` (the Application record); `hiring-postings`' `TS-BL-038` (§11.1's
posting state machine, on which `TS-BL-056`/`057`/`061` **invoke** three already-declared advances
without registering any — `design.md` D10a records why that is an interface rather than an edge); and
`design-system`'s `TS-BL-008`/`009`/`011`/`012`. Where an interface has not landed, build against its
declared shape and say so.

**Three posting-state advances are this feature's, and were found by another feature.**
`decision-and-offers`' verification (its `design.md` D9a) found that `hiring-postings`' `TS-BL-038`
declares §11.1's posting machine whole while **nothing invokes the six advances between `open` and
`onboarding`**. It built the bottom three; the top three are triggered by this feature's events and
land on `TS-BL-056` (1.19), `TS-BL-057` (2.16) and `TS-BL-061` (6.18). Each is **monotone,
idempotent, and attributed to the human whose action triggered it, inside that action's
transaction**. **Do not build D9a's third property, "recompute on reopen"** — that is
`decision-and-offers`' concern for its own three and is built there (`design.md` D10a). Until these
land, `decision-and-offers` task 6.16's end-to-end `open`→`filled` test is **expected to fail**;
it is that feature's artifact, and it must never be made to pass by relaxing its closure guard.

**Three questions handed to this feature by name are answered in `design.md`, and one is answered only
halfway on purpose.** D5 answers `design-system`'s presentation-shell question (**the Interview
Console's active-interview state**). D6 answers `matching-and-ranking`'s source-type-seam question (**by
using it**). D7 answers `D23a`'s which-application-survives rule **only for stages this feature owns**,
and **refuses** the merge beyond them — `decision-and-offers` owns those stages, and inventing their
rules would repeat the mistake `candidate-intake` and `matching-and-ranking` each declined to make.

**Four things are deliberately unresolved and must not be decided during apply.** `SCR-003`'s
dimensions carry **no `OD-007` approval** — ship them seeded and marked unapproved, not as product
truth. **No dimension weighting** is approved, so none ships. `innovation practice relevance` ships as
specified with its rename question recorded. `OD-003`'s provider is open, so all three families run on
the stub in Dev.

---

## 1. TS-BL-056 — Shortlisting decision UI

**Goal:** the first human decision about a person in this product is accountable by construction — an
actor, a reason from a maintained vocabulary, a workflow transition, an audit record — and the negative
outcomes are held to the same standard as the positive ones. Covers `interview/shortlisting`.

```yaml
backlog_items:
  - id: TS-BL-056
    feature: interview-pipeline
    depends_on: [TS-BL-053, TS-BL-004]
    status: not-started
```

**Read `design.md` D2 and D10 before starting.** D2 records that "reason on every disposition" is a
recommendation treated as adopted, and separates what `WF-005` already carries from what the
recommendation adds — build both, but do not cite the second as settled. D10 fixes that this item
registers the **whole** Application state machine for this feature's range, including states
`decision-and-offers` owns as declared-reachable, and that `TS-BL-057`/`058`/`062` invoke transitions
rather than declaring them.

- [ ] 1.1 Register the **Application state machine** declaratively against `platform-core`'s workflow
      service — §11.2's `Ranked`, `Shortlisted`, `InterviewScheduled`, `Interviewed`, `ScorecardReady`,
      their permitted transitions, the permission each declares, and `WF-005`'s reason-required marking
      on `NotSelected` (reachable from four states) and `Withdrawn`. Build against
      `platform/workflow-engine`'s finished spec as an interface if `TS-BL-004` has not landed
- [ ] 1.2 Declare `PrioritySelected` onward as **reachable but unowned**, so `WF-002`'s rejection can
      name the transitions available from `ScorecardReady` rather than making a correct Application
      look terminal (`design.md` D10)
- [ ] 1.3 Add the test that fails if Application-specific transition logic appears **inside** the
      workflow framework rather than in this registration
- [ ] 1.4 Implement the shortlist action requiring a **reason code from the maintained list plus
      optional free text**, rejecting a reasonless submission with a field-level validation error and
      no state change (`SHL-001`, `SHL-002`)
- [ ] 1.5 Implement negative disposition with the same reason requirement, and verify it holds from
      **each** of the four states §11.2 reaches `NotSelected` from — `WF-005` covers this
      independently of the recommendation, so assert it against the machine rather than the UI
- [ ] 1.6 Reject a reason code the maintained list does not contain, rather than storing it as free
      text under an unknown code
- [ ] 1.7 Seed the **disposition reason code list** into `platform-core`'s audited
      runtime-configuration registry, assert no unaudited setter exists, and verify a retired code
      leaves historical dispositions readable under the code they were recorded with (`design.md` D2 —
      this is what makes rejecting the unratified half a configuration change)
- [ ] 1.8 Implement `SHL-004`'s removal: its own transition, its own actor and reason, with the prior
      shortlist record and reason preserved. Verify shortlist → remove → re-shortlist leaves three
      individually readable events (`BR-020`, `WF-006`)
- [ ] 1.9 Write `candidate_posting_applications.status` and `shortlist_reason` for the first time —
      columns `candidate-intake`'s `TS-BL-046` created and left unwritten — **only through the workflow
      service**, and add the test that fails on any direct write to `status`, `shortlist_reason` or
      `final_outcome`
- [ ] 1.10 Implement `SHL-003`'s **scheduling task** for the posting's responsible recruiters, and
      verify a shortlist still succeeds and the task still exists when notification delivery is
      unavailable
- [ ] 1.11 Deliver the accompanying notification through `platform-core`'s internal engine, and add the
      test that fails if any message produced by a disposition is addressed to the candidate (`D04`,
      `OD-005`, `C-04`)
- [ ] 1.12 Gate every disposition through the central evaluator on the transition's declared
      permission, render unpermitted affordances as **absent** rather than disabled, refuse a directly
      submitted request server-side, and verify a denial and an empty permitted result are
      distinguishable (`AUTHZ-003`, `AUTHZ-004`, `UI-002`, `SEC-005`)
- [ ] 1.13 Assert the **advisory-only** guarantee from the transition side: no path from an AI output
      record to an Application state write, and the AI Service Account denied a disposition on the same
      evaluator terms as a human. This is the first capability that *has* the transitions the guarantee
      is about (`AI-010`, `BR-007`)
- [ ] 1.14 Write every disposition's audit record in the **same transaction**, carrying references and
      no candidate personal data, and verify a failed audit write fails the disposition. Build against
      `TS-BL-020`'s durable writer (§28 item 11)
- [ ] 1.15 Build the **Shortlist View** (§14.2) from `design-system`'s dense data table, page templates
      and form components, adding **no new component**, no raw color and no one-off spacing value
- [ ] 1.16 Implement §13.2's `POST /api/applications/{applicationId}/shortlist` and
      `.../remove-shortlist`, and verify a transition invalid from the current state is rejected with
      the current state and available transitions named (`WF-002`, `NFR-011`)
- [ ] 1.17 Verify concurrent disposition attempts on one Application leave exactly one winner with the
      other rejected and the current state reported back (`platform-core`'s concurrent-transition
      requirement, first exercised here)
- [ ] 1.18 Ship the surface and its endpoints behind a declared feature flag, disabled by default —
      a disabled capability answers 404, and an undeclared flag name raises rather than resolving false
- [ ] 1.19 Advance the posting `open → screening` on the **first** shortlist, attributed to the
      shortlisting actor inside that action's transaction, and as a **no-op** where the posting is not
      in `open` — never moving a posting backwards and never erroring on a second shortlist. One of
      three advances this feature owns; `hiring-postings` `TS-BL-038` declares the transition and
      nothing invoked it (`design.md` D10a). Assert no posting advance fires with no authenticated
      actor, which is what keeps `hiring-postings`' own assertion true

---

## 2. TS-BL-057 — Interview round scheduling

**Goal:** rounds exist against an Application with a real assignment on them, so that every downstream
interview surface has a scope to be evaluated against — and the shortlist gate that precedes them is a
recorded exception when overridden, never a silent bypass. Covers `interview/round-scheduling`.

```yaml
backlog_items:
  - id: TS-BL-057
    feature: interview-pipeline
    depends_on: [TS-BL-056]
    status: not-started
```

**Read `design.md` D1 before starting.** This item **publishes the round-created event and registers no
subscriber** — `TS-BL-058` registers the Question Generation job type against it. Publishing later
would mean editing the scheduling path once its consumer exists; generating here would produce a
question set no surface renders, which is `platform-core` D5's silent-success class. The same
producer-before-consumer seam `hiring-postings` used for `posting.opened`.

- [ ] 2.1 Migrate `interview_rounds` by reversible migration with §12.2's columns —
      `candidate_application_id`, `round_number`, `interview_type`, `interviewer_id`,
      `scheduled_start`, `scheduled_end`, `status`, `question_set_id` (nullable) — with the
      `interview_type` and `status` enums carrying all of §12.2's values from the first migration
- [ ] 2.2 Implement `INT-001`/`BR-009`'s multiple rounds per Application with sequential round numbers,
      and verify rounds on two Applications for the same candidate are independent and unshared
- [ ] 2.3 Reject an interview type outside the enumeration rather than storing it as free text, and
      verify two different types resolve to the **same** fixed note field set (`C-06` closed
      per-type templates)
- [ ] 2.4 Implement `SHL-005`'s block on scheduling a non-shortlisted candidate, with an **authorized
      override** carrying actor and mandatory reason, audited; reject a reasonless override and deny an
      unpermitted one (`G-06`, `BR-016`)
- [ ] 2.5 Implement panelist **assignment** as the fact that grants access, applied as a **query
      predicate**, denying an unassigned request rather than returning empty, and verify a denial and
      an empty permitted result are distinguishable (`INT-003`, `D01`, `C-03`,
      `access-control-and-admin`'s query-filter contract)
- [ ] 2.6 Verify the assignment scope this item creates is the one `matching-and-ranking`'s
      `TS-BL-054` projection resolves against — that feature's spec applies assignment scope to a
      projection whose assignments did not exist until now
- [ ] 2.7 Implement round status transitions through the **workflow service** against `TS-BL-056`'s
      registration, with cancellation and no-show reason-required, and add the test that fails on a
      direct status write (`WF-001`, `WF-005`, `BR-018`)
- [ ] 2.8 Verify a reschedule leaves the **prior scheduled window readable** on the transition history
      rather than overwriting it
- [ ] 2.9 Implement `POST /api/interviews/{interviewId}/complete` refusing completion until at least
      one note is submitted, so a completed round can never be a scorecard's evidence-free precondition
      (§13.2, `SCR-001`, `BR-010`)
- [ ] 2.10 Publish the **round-created event** through `platform-core`'s dispatch pattern carrying the
      correlation identifier, and assert this capability registers **no subscriber** for it
      (`design.md` D1)
- [ ] 2.11 Verify a round whose event was never delivered is still viewable and still conductable —
      the degradation property `hiring-postings` D7 established for `posting.opened`, from the producer
      side
- [ ] 2.12 Notify the assigned interviewer through internal delivery naming candidate, posting, type
      and window; add the test that fails if this capability calls a calendar API, and the test that
      fails if any message it produces is addressed to the candidate (`D04`, `D06`/`C-04`'s accepted
      gap, `OD-005`)
- [ ] 2.13 Implement `POST /api/applications/{applicationId}/interviews` and
      `GET /api/interviews/assigned` (§13.2), gated through the central evaluator with affordances
      absent rather than disabled
- [ ] 2.14 Seed §29 item 9's **interview and feedback aging thresholds** as configuration keys in
      `platform-core`'s audited registry, and assert **no transition** in this feature is driven by
      one — aging drives `insight-and-reporting`'s SLA views, not the state machine
      (`design.md` Open Questions; do not invent threshold values)
- [ ] 2.15 Ship scheduling behind a declared feature flag, disabled by default
- [ ] 2.16 Advance the posting `screening → interviewing` on the **first** round scheduled,
      attributed to the scheduling actor inside that action's transaction, and as a **no-op** where the
      posting is not in `screening` — never moving a posting backwards. Verify the `SHL-005`
      override case explicitly: a first round scheduled on a posting that never entered `screening`
      attempts no advance (`design.md` D10a)

---

## 3. TS-BL-058 — Interview Console, structured notes, multiple rounds

**Goal:** an interviewer works from one surface that shows them what they are permitted to see and
nothing else, suggests questions they are free to ignore, and records their account of the interview in
a shape every downstream evidence path can read. Covers `interview/interview-console`.

```yaml
backlog_items:
  - id: TS-BL-058
    feature: interview-pipeline
    depends_on: [TS-BL-057, TS-BL-027]
    status: not-started
```

**The `TS-BL-027` edge is added by this feature.** `D.10` gives this item one dependency; it authors
`interview_questions`, one of two families `D.10` left with no owner (`design.md` D1). **Read D1, D5,
D6 and D11 before starting.** D1 fixes that exactly **one** job type is registered — the round-created
subscriber — and none for the gateway call, the same both-halves-true rule
`matching-and-ranking` D1 named as the most likely error in a feature of this shape. D5 assigns the
presentation shell here and fixes that it is **defence in depth, never the mechanism**. D6 fixes the
vector registration. D11 fixes that source labels are consumed, never derived.

**Build the ranking context by consuming, not assembling.** `matching-and-ranking`'s
`matching/ai-context-visibility` spec carries a requirement that its projection be consumed *"rather
than assembling its own view of ranking output."* Build against `C-10` — **gaps yes, score no** — not
against `D05`'s reversed "full context upfront."

- [ ] 3.1 Migrate `interview_notes` by reversible migration with §12.2's columns, **including
      `version_number` and `is_current_version` from the first submission** — `TS-BL-059` owns
      edit-creates-a-version, this item owns the version row (the refinement `candidate-intake` made
      for resume versions, `design.md` Migration Plan step 4)
- [ ] 3.2 Implement `INT-006`'s eight fixed structured fields plus the five-point recommendation,
      rejecting a recommendation outside the scale and verifying the field set is identical across
      interview types
- [ ] 3.3 Implement `INT-007`'s stored forms — normalized structured fields, semantic tags, raw notes,
      embedding reference, interviewer identity, source references, timestamps — and verify each is
      present on a submitted note
- [ ] 3.4 Build the console listing **only assigned interviews**, keyed to the evaluator's verdict
      rather than a role name, and verify a broader grant widens the listing with no code change and is
      audited (`INT-003`'s "unless broader permission is granted", `AUTHZ-005`, `ADM-005`)
- [ ] 3.5 Add the test that fails if a role-name comparison decides what the console lists
- [ ] 3.6 **Consume `TS-BL-054`'s projection** for AI context and add the test that fails if this
      capability assembles its own view of ranking output. Build against the declared payload contract
      if `TS-BL-054` has not landed (`C-10`, `INT-005`)
- [ ] 3.7 Assert the console payload on the wire carries **no match score, no rank position, no cohort
      size** and **no reference to any other candidate** on the posting (`C-10`, `D05`'s still-standing
      per-candidate rule)
- [ ] 3.8 Build the **active-interview state** on `design-system`'s presentation shell — chrome
      removed, content full-viewport — answering that feature's open question, and verify the withheld
      values are absent in **both** shell states so the guarantee never rests on chrome
      (`design.md` D5, `config.yaml`'s produce-time-not-display-filter rule)
- [ ] 3.9 Assemble every surface from `design-system` components — dense table, page templates, forms,
      AI-disclosure marking, evidence-source labels — and add the test that fails if a component, raw
      color or one-off spacing value is defined here
- [ ] 3.10 Author `interview_questions` as a **new template version** of the family `TS-BL-030`
      registered, never by editing an existing version. Build against the registry's declared interface
      if it has not landed (`AI-003`)
- [ ] 3.11 Build the request inside the gateway's **fixed envelope** carrying posting and application
      **references**, and assert the payload carries no job description text, no resume text and no
      candidate personal data (`ai-platform-governance` D3)
- [ ] 3.12 Register the **Question Generation job type** as a subscriber to `TS-BL-057`'s round-created
      event through `platform-core`'s dispatch pattern, declaring §25's retry policy and carrying the
      correlation identifier. `TS-BL-057` publishes and registers no subscriber; this is it
- [ ] 3.13 Assert this capability's registered job types are **exactly one**, that none corresponds to
      a gateway call (`ai-platform-governance` D10), and that it makes **no outbound call to a model
      provider**
- [ ] 3.14 Implement idempotency at the effect boundary: the same round-created event delivered twice
      leaves one question set for that round (`platform-core` D6 — delivery is at-least-once)
- [ ] 3.15 Implement `INT-004`'s advisory questions: categorized, attached to the round, **usable,
      editable or ignorable**, with no completeness rule referencing the question set and the
      AI-generated marking present and uncleared by anything in this capability (`AI-001`, `UI-004`)
- [ ] 3.16 Verify a round remains conductable and notes remain submittable when question generation
      fails or the provider is unavailable, with the failure **visible** rather than silent (`G-12`,
      `NFR-004`)
- [ ] 3.17 Declare a **bound on every free-text field** of the question contract, provisional and
      labelled provisional in audited configuration, rejecting over-long output as a contract violation
      recorded as a failed run (`S.5`, `AGENTS.md`'s standing bar, `ai-platform-governance` D7)
- [ ] 3.18 **Register `interview_note_section`** against `matching-and-ranking`'s source-type seam, and
      verify the registration requires **no change to the vector record schema** — the property
      `matching-and-ranking` D2 designed for and recorded as an open question. If it does require one,
      that is a finding for `/opsx:update` against that feature, not a local workaround
      (`design.md` D6)
- [ ] 3.19 Pin a note section's vector records to the note **version** via `source_version`, so a new
      version produces new records rather than mutating existing ones — without this, the drift
      `TS-BL-061` prevents in the scorecard reappears in the index
- [ ] 3.20 Add the test that fails if this capability contains an embedding pipeline, document
      segmenter or vector store of its own
- [ ] 3.21 Implement §13.2's `POST /api/interviews/{interviewId}/notes`,
      `GET /api/interviews/{interviewId}` and `POST /api/applications/{applicationId}/ai/questions`,
      each gated server-side with unpermitted affordances absent
- [ ] 3.22 Add the test that fails if this capability writes an Application's disposition fields or
      triggers a transition other than a round's own completion — verify specifically that a
      `strong_no` note recommendation causes **no** disposition. A recommendation field is the most
      natural place for someone to wire one
- [ ] 3.23 Add `interview_questions` cases to `TS-BL-032`'s corpus — contract conformance, conciseness
      per bounded field — and record that production activation is refused until they pass
      (`C-09`, `ai-platform-governance` D8)
- [ ] 3.24 Ship the console and its endpoints behind a declared feature flag, disabled by default

---

## 4. TS-BL-059 — Interview note versioning without lock

**Goal:** a note's whole history stays readable and nothing about it ever freezes — editability is a
permission, not a state — and the change it represents is announced so the evaluations drawn from it
can respond. Covers `interview/note-versioning`.

```yaml
backlog_items:
  - id: TS-BL-059
    feature: interview-pipeline
    depends_on: [TS-BL-058, TS-BL-027]
    status: not-started
```

**The `TS-BL-027` edge is added by this feature.** `D.10` gives this item one dependency; it authors
`interview_note_summary`, the second family `D.10` left with no owner (`design.md` D1 records why the
summary belongs here rather than on the console — its output contract references note **versions**, and
building it one item earlier means building a summary that goes stale the first time anyone edits).

**Read `design.md` D8 in full, and `C-08` in full, before starting.** `C-08` makes **three** reversals
of `D24` and satisfying one while missing another is the failure mode here. `D24`'s own text still
claims it satisfies a `D11` locking requirement; that line is stale and `D11`'s matching consequence
was un-annotated until this conversation (`design.md` D14a). **The one `D24` consequence that
survives** — that the approved scorecard records which note version it was drafted from — is
`TS-BL-060`'s, and it is what makes this item's event useful.

- [ ] 4.1 Implement `INT-008`/`BR-017`: every edit to a submitted note creates a **new version**, with
      every prior version readable unchanged and exactly one marked current
- [ ] 4.2 Add the test that fails if any path overwrites a submitted note's content in place
- [ ] 4.3 Assert **no lock exists**: no lock flag, no freeze timestamp, no approval-derived edit
      condition anywhere in the edit path (`C-08`, reversing `D24`'s freeze-on-approval)
- [ ] 4.4 Verify a permitted actor can edit a note whose Application has an **approved scorecard**, and
      a note on an Application in a **terminal state**, in both cases creating a new version
- [ ] 4.5 Add the test that fails if an **append-only addendum** path exists as a substitute for
      editing — `D24`'s post-freeze correction mechanism, which `C-08` removed along with the freeze
- [ ] 4.6 Decide editability through the **central evaluator's verdict on the Interview Console `Edit`
      action**, and add the test that fails if an authorship comparison in code decides it (`C-08`,
      `C-02`, `C-03`, `AUTHZ-001`, `AUTHZ-005`)
- [ ] 4.7 Verify all three matrix cases without a code change: seeded actor, non-authoring role granted
      the action (audited), and an actor holding it by role with a **direct denial** recorded
- [ ] 4.8 Seed the Interview Console `Edit` action to **Interviewer only** as a recorded, auditable
      configuration state in `access-control-and-admin`'s seeded matrix — not as an unrecorded script
      effect (`C-08`'s recommended seeding, `Interaction B`, `AUTHZ-008`)
- [ ] 4.9 Record the **author of each version** and retain the note's original `submitted_by`, and
      verify a permitted non-author edit is visible as a non-author edit rather than absorbed into the
      original — this is the protection that survives `C-08` removing the author-only rule
      (`design.md` D8)
- [ ] 4.10 Publish the **note-version-created event** naming note, version, round and Application with
      the correlation identifier; verify the version still exists when the event is undelivered, and
      that duplicate delivery produces its downstream effects once
- [ ] 4.11 Author `interview_note_summary` as a **new template version** of the registered family,
      never by editing an existing version, and register the **Interview Summary job type** against
      `platform-core`'s dispatch pattern with §25's retry policy
- [ ] 4.12 Implement `INT-009`'s gate: refuse summary generation for a round with **no submitted
      note**, and verify the refusal rather than an empty summary
- [ ] 4.13 Build the summary request in the gateway's fixed envelope carrying **note references**, and
      assert it carries no note text and no candidate personal data
- [ ] 4.14 Implement §16.3's required output — a structured summary carrying **note-version
      references** — and verify a summary generated before a later note version is identifiable as
      drawn from a superseded version rather than presented as current
- [ ] 4.15 Carry a **source label from `TS-BL-031`'s closed vocabulary** and at least one note-version
      reference on every summary claim, rejecting a claim with neither, and assert this capability
      derives no source value of its own. This is the product's first producer of
      `interview_note`-labelled output (`G-02`, `AI-007`, `BR-008`, `design.md` D11)
- [ ] 4.16 Declare a **bound on every free-text field** of the summary contract, provisional and
      labelled provisional, rejecting over-long output as a contract violation with a failed run
      recorded and no content persisted. `design.md` Open Questions names this as the family most at
      risk of drifting long, since its input is prose (`S.5`)
- [ ] 4.17 Mark note-derived content as **untrusted document-derived content** at the prompt boundary
      and verify candidate text quoted in a note and shaped as an instruction alters neither the family,
      the permissions, nor the output contract (`AI-012`, `AI-013`, `SEC-014`)
- [ ] 4.18 Implement `PATCH /api/interviews/{interviewId}/notes/{noteId}` (§13.2) as **create a new
      version**, gated server-side, and audit each version creation with actor, references and no
      candidate personal data (§28 item 12)
- [ ] 4.19 Implement `RET-006`/`RET-007` from the note side: summaries, embeddings and scorecards
      derived from a note version are **discoverable from it**, and disposal is recorded. `OD-004`
      supplies the interval — do not invent one
- [ ] 4.20 Add `interview_note_summary` cases to `TS-BL-032`'s corpus, including `AI-015`'s
      **conflicting interview notes** class — this feature is the first that can construct it for real
      — and record that production activation is refused until they pass
- [ ] 4.21 Ship versioning and summary generation behind a declared feature flag, disabled by default

---

## 5. TS-BL-060 — Scorecard Center, AI-drafted generation

**Goal:** a draft evaluation exists only where a real interview happened, keeps resume evidence and
interview evidence apart in the record rather than on the screen, and names exactly which note versions
it rests on. Covers `interview/scorecard-generation`.

```yaml
backlog_items:
  - id: TS-BL-060
    feature: interview-pipeline
    depends_on: [TS-BL-058, TS-BL-027]
    status: not-started
```

**Read `design.md` D9 and D11 before starting.** D9 fixes the note-version set this item records and
why it is load-bearing: under `D24`'s lock it was a formality after approval; under `C-08` it is the
**only** thing that makes drift detectable, and `TS-BL-061` cannot scope supersession without it. D11
fixes that `SCR-004`/`SCR-005` are **output-contract clauses**, not display choices — `C-05`'s reversal
of `D11` rests on them being properties of the stored artifact.

**Take `TS-BL-059` first where the order is free.** Both depend only on `TS-BL-058`; recording a
note-version set against versions that already exist is cheaper than retrofitting a version scheme
around an existing draft (`design.md` Migration Plan step 5).

- [ ] 5.1 Migrate `scorecards` by reversible migration with §12.2's columns —
      `candidate_application_id`, `version_number`, `status`, `resume_evidence_json`,
      `interview_evidence_json`, `consolidated_evaluation_json`, `overall_recommendation`,
      `human_review_comments`, `approved_by`, `approved_at`, `ai_run_log_id` — with the `status` enum
      carrying **all four** values including `superseded` from the first migration
- [ ] 5.2 Implement `SCR-002`/`BR-011`'s eligible list — **interviewed candidates only** — and verify
      a shortlisted-but-uninterviewed candidate and a scheduled-but-incomplete round both fail to
      appear (§13.2's `GET /api/postings/{postingId}/scorecards/eligible`)
- [ ] 5.3 Refuse generation for an Application with **no completed round**, and verify the refusal
      holds for a request submitted **directly against the API** rather than through the list
      (`SCR-001`, `AI-009`, `BR-010`)
- [ ] 5.4 Verify carried evidence does not substitute: an Application holding evidence from a prior
      Application but no completed round of its own is still refused (`C-07` — a confirmatory round is
      required regardless of carried evidence)
- [ ] 5.5 **Record the note-version set** the draft was generated from, at generation time, spanning
      every round consumed, and verify the set still names those versions after later versions exist
      (`D24`'s surviving consequence; `design.md` D9)
- [ ] 5.6 Store resume-derived and interview-derived evidence as **separate bodies**, reject merged
      output as a contract violation, and add the test that fails if a presentation-layer split is
      relied upon for the separation (`SCR-004`, `UI-005`, `design.md` D11)
- [ ] 5.7 Require an **evidence source label from the closed vocabulary and a confidence level on every
      dimension**, rejecting output missing either, and assert this capability derives no source value
      of its own (`SCR-005`, `G-02`)
- [ ] 5.8 Implement insufficiency **reusing `matching-and-ranking`'s vocabulary and shape verbatim** —
      a marker naming the absent evidence, never an inferred value — and make **three** states
      distinguishable: insufficient, evaluated as weak, not evaluated (`G-03`, `AI-008`;
      `matching/ranking-insufficiency`; that feature's `design.md` D10 on why sameness matters more
      than local fit)
- [ ] 5.9 Reject as a contract violation any output assigning a dimension a value with no evidence
      reference, recorded as a failed run
- [ ] 5.10 Author `scorecard_generation` as a **new template version** of the registered family, and
      build the request in the gateway's fixed envelope carrying **resume-version and note-version
      references** — asserting no resume text, no note text and no candidate personal data in the
      payload
- [ ] 5.11 Register the **Scorecard Generation job type** against `platform-core`'s dispatch pattern
      with §25's retry policy; assert exactly one job type, none for the gateway call, and **no
      outbound call to a model provider**
- [ ] 5.12 Implement idempotency at the effect boundary: the same generation request delivered twice
      issues one run and produces one draft
- [ ] 5.13 Mark resume- and note-derived content as **untrusted document-derived content** at the
      boundary, and verify instruction-shaped text in either alters neither the family, the caller's
      permissions, the dimensions evaluated, nor the output contract (`AI-012`, `AI-013`, `AI-014`,
      `SEC-014`)
- [ ] 5.14 Declare a **bound on every free-text field** of §16.3's `scorecard_generation` contract,
      provisional and labelled provisional in audited configuration, rejecting over-long output as a
      contract violation with no content persisted (`S.5`, `ai-platform-governance` D7)
- [ ] 5.15 Create the draft in **Draft status** with the AI-generated marking, assert generation causes
      **no** Application state transition, and add the test that fails if any path in this capability
      clears the marking (`SCR-006`, `AI-001`, `AI-010`, `BR-007`, `UI-004`)
- [ ] 5.16 Implement `G-12`/`NFR-004` degradation: with the provider unavailable, existing scorecards,
      notes and rounds stay usable, the request reports temporary unavailability, and **no partial
      draft** is created — a partial draft presented as complete is the specific failure here
- [ ] 5.17 Implement `NFR-002`'s asynchronous execution with retrievable progress by identifier, and
      `ERR-007`'s retrievable run reference for every failure
- [ ] 5.18 Build the **Scorecard Center** (§14.2) from `design-system`'s dense table, page templates,
      AI-disclosure marking and evidence-source label components, adding **no new component**
- [ ] 5.19 Implement §13.2's `POST /api/applications/{applicationId}/scorecards/generate`, requiring
      the **Run AI** action on this surface distinct from view and edit, taken from the central
      evaluator, with the control absent for actors who lack it and a direct request refused
      server-side (§9.3, `C-02`, `AUTHZ-004`)
- [ ] 5.20 Add `scorecard_generation` cases to `TS-BL-032`'s corpus — contract conformance,
      conciseness per bounded field, insufficiency on low-information evidence, and `AI-015`'s
      **conflicting interview notes** class — and record that production activation is refused until
      they pass (`C-09`, `ai-platform-governance` D8)
- [ ] 5.21 Record in `KNOWN_ISSUES.md` that all three of this feature's families run against the stub
      provider in Dev pending `OD-003` — the same treatment the unconfirmed Hubble contract receives.
      **This does not duplicate `ai-platform-governance`'s task 1.18:** that task records `OD-003`
      itself, and this one records that this feature's three families are affected by it. If 1.18 has
      already landed, extend its entry rather than adding a second (`design.md` D13)
- [ ] 5.22 Ship the Scorecard Center and its endpoints behind a declared feature flag, disabled by
      default

---

## 6. TS-BL-061 — Scorecard human-approval workflow

**Goal:** an approval means a named human read *this* evidence — and stays meaning that, because the
moment the evidence moves the approval stops standing. Covers `interview/scorecard-approval`.

```yaml
backlog_items:
  - id: TS-BL-061
    feature: interview-pipeline
    depends_on: [TS-BL-060]
    status: not-started
```

**Read `design.md` D3 and D9 in full before starting.** D3 is the decision this feature made on an
`exploration-notes.md` open question marked **"Not yet decided"** — build supersede-and-re-approve —
with the two rejected alternatives (reintroduce the lock; warn and leave the approval standing) and the
four grounds. D9 fixes the join between the two version chains and the ordering hazard: a scorecard
superseded one second after being approved is technically consistent and reads as a malfunction, which
is why 6.9 exists separately from 6.5.

**Build the gate and the supersede path together, not the gate first** (`design.md` Migration Plan step
6). An approval gate shipped without the supersede path is an approval that cannot be invalidated —
exactly the state `C-08` created and this item exists to close. Same reasoning `matching-and-ranking`
used for shipping the re-index path with the pipeline.

- [ ] 6.1 Implement `SCR-006`'s gate: a generated scorecard stays **Draft** until an authorized human
      approves it, with approver and timestamp recorded and the AI-generated marking cleared only on
      approval
- [ ] 6.2 Require the **Approve** action from the central evaluator, refuse a direct request
      server-side with the control absent in the interface, and verify `Approve` is **ungrantable to
      the AI Service Account** (`ai-platform-governance`'s advisory-only capability, §9.3)
- [ ] 6.3 Implement `SCR-007`/`BR-016`'s mandatory-reason adjustment, rejecting a reasonless
      adjustment, and verify the **original AI output remains retrievable** after adjustment
      (`RANK-006`'s companion principle)
- [ ] 6.4 Write adjustments through `ai-platform-governance`'s **existing override record** — actor,
      reason, timestamp, original output reference, human value — and add the test that fails if an
      override store owned by this capability exists (`G-06`)
- [ ] 6.5 Implement **supersession**: a note-version-created event supersedes exactly those **approved**
      scorecards whose recorded note-version set names an earlier version of that note. Verify an
      unrelated note supersedes nothing, and that a still-Draft scorecard is **not** superseded —
      there is no approval to invalidate (`C-08`'s open consequence, `design.md` D3)
- [ ] 6.6 Verify a superseded scorecard reads as **superseded** rather than presenting as approved, and
      that it requires re-approval before being treated as an evaluation of record
- [ ] 6.7 Preserve the prior approved content, approver, timestamp and note-version set through
      supersession; verify re-approval creates a **new version** and that supersession **alone** does
      not (`SCR-008`, `design.md` D9 — otherwise every typo inflates the version chain)
- [ ] 6.8 Verify the full trail `design.md` D9 specifies: approved at T1 against {v2, v1}, superseded
      at T2 by note v3, re-approved at T3 against {v3, v1} — each step individually readable
- [ ] 6.9 Implement the **pre-approval evidence check**: refuse approval where the draft's recorded note
      versions are no longer current, naming the notes that changed and directing the approver to
      regenerate (`D24`'s surviving interaction note, `design.md` D9)
- [ ] 6.10 Build the **re-approval view** presenting the difference between the note versions the prior
      approval was made against and the current ones (`design.md` D3's stated proportionality measure,
      `UI-003`)
- [ ] 6.11 Add the test that fails if a **materiality rule** exists — any path classifying a note edit
      as too small to supersede. `design.md` D3 declines this deliberately: a rule that decides which
      evidence changes are unimportant is a rule that can be wrong silently
- [ ] 6.12 Execute approval, supersession, rejection and re-approval through the **workflow service**,
      and attribute system-initiated supersession to a **service account** correlated to the triggering
      note-version event (`WF-001`, `WF-004`)
- [ ] 6.13 Write each of the four events' audit records in the **same transaction**, carrying references
      and no candidate personal data, and verify a failed audit write fails the operation (§28 item 13)
- [ ] 6.14 Verify the **divergence rate between approved scorecards and AI drafts is computable from
      stored override records alone**, and that an approval with no adjustment is distinguishable from
      one with an adjustment — this is what makes the second half of `insight-and-reporting`'s
      AI-agreement-rate dashboard possible
- [ ] 6.15 Implement §13.2's `POST /api/scorecards/{scorecardId}/approve` and
      `PATCH /api/scorecards/{scorecardId}`, and verify an invalid status transition is rejected with
      the current status and available transitions named (`WF-002`)
- [ ] 6.16 Instrument the **rate of superseded scorecards** through OpenTelemetry — `design.md`'s Risks
      section names a near-zero rate as evidence of note-correction avoidance rather than of quality,
      and that signal is unavailable unless it is emitted (§26)
- [ ] 6.17 Ship approval and supersession behind a declared feature flag, disabled by default
- [ ] 6.18 Advance the posting `interviewing → scorecard_review` on the **first** scorecard approved,
      attributed to the approving actor inside that action's transaction, and as a **no-op** where the
      posting is not in `interviewing`. Verify the two cases this feature has that the others do not:
      a **re-approval after supersession** is not a first approval and attempts no advance, and
      **supersession itself moves no posting**. **This is what makes `decision-and-offers`'
      `scorecard_review → selection` advance reachable at all** (`design.md` D10a)

---

## 7. TS-BL-062 — Consolidated scorecard across rounds

**Goal:** one document a Practice Manager approves and a later posting can carry, where three panelists
disagreeing looks like three panelists disagreeing rather than a number that hides it. Covers
`interview/scorecard-consolidation`.

```yaml
backlog_items:
  - id: TS-BL-062
    feature: interview-pipeline
    depends_on: [TS-BL-061]
    status: not-started
```

**Read `design.md` D4, D6 and D7 before starting.** D4 answers `D11`'s surviving open item — the rule
for consolidating conflicting panelist recommendations — as **surfaced, never resolved
arithmetically**, and explains why that is load-bearing rather than cosmetic: `C-05` reversed `D11` on
the ground that consolidation is *not* averaging, so an arithmetic overall value would make `D11`'s
original objection correct. D6 fixes the second vector registration. D7 is the merge rule, answered
only as far as this feature's stages reach and **refused** beyond them.

- [ ] 7.1 Implement `C-05`'s shape: **one active scorecard per Application**, keyed to
      `candidate_application_id`, drawing on every completed round. Add the test that fails if a
      scorecard keyed to an interview round exists
- [ ] 7.2 Verify a second completed round updates the existing scorecard's evidence set rather than
      creating a second scorecard, and that two Applications for one candidate hold independent
      scorecards
- [ ] 7.3 Present **each contributing panelist's own recommendation and ratings**, attributed to that
      panelist and round, and verify a contribution traces back to its note version, author identity
      and round (`C-05`'s reconciliation finding — adopting the consolidated model does not lose
      per-panelist attribution, which was `D11`'s actual concern)
- [ ] 7.4 Render conflicting recommendations **as a disagreement**, and verify individual
      recommendations remain individually shown even when unanimous
- [ ] 7.5 Add the test that fails if the overall recommendation is derived by **any arithmetic** over
      panelist recommendations — mean, median, majority or weighted (`design.md` D4; `D11`'s "no
      averaged score, so no fake math papering over panelist disagreement")
- [ ] 7.6 Implement the overall recommendation as **AI-proposed and human-decided**: marked
      AI-generated until approved, changeable only through `TS-BL-061`'s mandatory-reason path, with
      the AI-proposed value still visible beside the human one, and a value outside the five-point
      scale rejected
- [ ] 7.7 Implement regeneration when a further round completes after approval, and verify the
      regenerated content **requires its own approval** — an approval against a two-round evidence set
      is not an approval of a three-round one (`SCR-006`, `SCR-008`)
- [ ] 7.8 Add the test that fails if completing a round causes any scorecard to become approved
      (`AI-010`, `BR-007`)
- [ ] 7.9 **Register `scorecard_summary`** against `matching-and-ranking`'s source-type seam, verify it
      requires no vector schema change, and add the test that fails if this capability contains an
      embedding pipeline or vector store of its own (`VEC-001`, `design.md` D6)
- [ ] 7.10 Assert this capability's surface area contains **no search over interview history** — `C-01`
      records it as a capability outside the original 19 features, and it stays out of scope and open
- [ ] 7.11 Make the consolidated scorecard **readable and citable from outside its own Application**
      with origin Application, posting, approval and evidence identifiable — the artifact
      `decision-and-offers`' `TS-BL-066` carries forward, and `C-05`'s second stated reason for
      reversing `D11`
- [ ] 7.12 Add the test that fails if carry-forward eligibility, attestation or a
      scorecard-to-second-Application link exists here. `D17`/`D21`/`C-07` belong to `TS-BL-066`; this
      item owes the carrier a document, not the carrying
- [ ] 7.13 Implement `design.md` D7's **merge rule for stages this feature owns**: the Application
      further along §11.2 survives, ties broken by earliest creation, with the other transitioned to a
      terminal state carrying a reason naming the merge and retaining its rounds, notes and scorecard
      (`BR-020`, `WF-005`, `WF-006`)
- [ ] 7.14 **Refuse** the merge where either Application has progressed beyond the scorecard stage,
      surfacing it for human resolution rather than resolving it — those stages are
      `decision-and-offers`', and `candidate-intake` D1(b) and `matching-and-ranking` D5 both declined
      to invent them for the same reason (`design.md` D7)
- [ ] 7.15 Add the test that fails if any path re-parents a note to a different round or Application —
      `INT-007` binds a note to its round and author, so moving one forges attribution
- [ ] 7.16 Verify the ranking half of the merge rule is honoured from this side: the surviving
      Application's ranking entry is **marked stale**, and nothing here deletes, merges or recomputes a
      ranking (`matching-and-ranking` `design.md` D5)
- [ ] 7.17 Ship consolidation behind a declared feature flag, disabled by default

---

## 8. TS-BL-063 — Fixed scorecard dimensions and competencies

**Goal:** every candidate in the product is evaluated against the same eight things, held as
configuration an owner can revise rather than a constant nobody can — and honestly labelled as
unapproved, because it is. Covers `interview/scorecard-dimensions`.

```yaml
backlog_items:
  - id: TS-BL-063
    feature: interview-pipeline
    depends_on: [TS-BL-062]
    status: not-started
```

**Read `design.md` D14(b) before starting, and do not implement `C-06`'s citation as written.** `C-06`
says the dimensions are *"configurable per §29 #14"*; **§29 item 14 is "Role and permission matrix
values"** and §29 carries no scorecard-dimension item at all. Building the citation literally would put
scorecard dimensions inside the permission matrix. The correct vehicle is `platform-core`'s audited
runtime-configuration registry. Corrected in `exploration-notes.md` during this feature's propose
conversation; the *conclusion* — seeded default, not settled product truth — is unaffected.

**Build against `C-06`, not `D15`.** `C-06` amends `D15`: AI-suggested competencies are **dropped**,
dimensions are **global not JD-scoped**, and `D15`'s "competencies become part of the JD version"
consequence is **superseded**.

- [ ] 8.1 Seed `SCR-003`'s eight dimensions — role fitment, technical depth, innovation practice
      relevance, project relevance, communication evidence, risk areas, gap closure status, overall
      recommendation — into `platform-core`'s **audited runtime-configuration registry** as a declared,
      closed key (`design.md` D14b)
- [ ] 8.2 Add the test that fails if a **hard-coded dimension list** exists anywhere in the codebase,
      and the test that fails if an unaudited setter for the dimension configuration exists
- [ ] 8.3 Verify a dimension rename is audited with previous value, new value and a mandatory reason,
      and takes effect **with no deployment**
- [ ] 8.4 Verify scorecards recorded before a rename remain readable **as they were recorded**
- [ ] 8.5 Reject a dimension outside the configured set as a contract violation rather than storing it
- [ ] 8.6 Add the test that fails if AI-proposed or per-role competency generation exists — `C-06`
      dropped it, and `D15`'s Option 2 is the superseded reading (`design.md` D14b's sibling case)
- [ ] 8.7 Verify dimensions are **global**: two Applications against different job descriptions use the
      same set, and no scorecard carries a job-description-version reference determining its structure
- [ ] 8.8 Verify **forking a job description changes no scorecard's structure** and forces no
      regeneration — `D15`'s superseded consequence would have required the opposite, and
      `hiring-postings`' fork-on-edit behaviour would have had to honour it
- [ ] 8.9 Implement the five-point scale `strong_yes | yes | maybe | no | strong_no` on dimension
      ratings and the overall recommendation, rejecting any other value, and verify it is **the same
      scale** the interview note recommendation uses (`D15` as confirmed by `C-06`, §12.2's two enums)
- [ ] 8.10 Add the test that fails if a **dimension weight or aggregate numeric score** exists.
      `OD-007` covers weightings and approves none; a weighted aggregate is averaging with extra steps,
      which `C-05`'s reversal of `D11` rules out (the same treatment `matching-and-ranking` gave §29
      item 8)
- [ ] 8.11 Attach **machine-readable provenance marking the dimension set unapproved by the business**,
      discoverable by an operator inspecting the configuration and updatable through the same audited
      path once `OD-007` resolves (`ai-platform-governance` D7's precedent for provisional values)
- [ ] 8.12 Seed `innovation practice relevance` **as specified** and mark it **pending confirmation or
      rename**, discoverable without reading `reference/spec.md` — `C-06` asks that it be confirmed or
      renamed and inventing a replacement is a product decision this feature does not own
- [ ] 8.13 Maintain the eight `INT-006` **note fields** and the eight `SCR-003` **dimensions** as
      separate sets, and verify renaming a note field leaves the dimension set unchanged — the two
      overlap in subject and conflating them would let a note field change silently restructure every
      scorecard
- [ ] 8.14 Record in `KNOWN_ISSUES.md` that `SCR-003`'s dimensions are seeded from an
      `OD-007`-unapproved set, that no weighting ships, and that `innovation practice relevance` is
      Miracle-Labs-specific pending confirmation
- [ ] 8.15 Ship the dimension configuration behind the same declared feature flags the scorecard
      surfaces use, and verify an undeclared flag name raises rather than resolving false

# Hiring — Job Descriptions and Postings — Backlog

This feature's complete backlog: eight items, `TS-BL-033` through `TS-BL-040`, decomposed in
`exploration-notes.md` D.10. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built, and nothing was inherited.** Sprint 0 of `talentsphere-wave-1-foundation`
shipped platform-layer code only — its handover states "no hiring feature exists yet, by design" —
and unlike the five Phase 1 features this one inherits nothing: that change has no `hiring/` delta
spec, and no `D1`–`D18` decision from its `design.md` was ever allocated here. `KNOWN_ISSUES.md`
carries nothing about job descriptions or postings. See `design.md` D10 for the check rather than
the assumption.

**This is the first feature that registers domain content rather than building substrate**, and it is
the first real consumer of four Phase 1 mechanisms that have so far only been asserted against their
own empty state: `platform-core`'s workflow engine (zero registered machines), its dispatch pattern,
`access-control-and-admin`'s `assigned-postings` scope predicate (which names
`job_postings.recruiter_ids[]`, a column created by `TS-BL-037`), and `ai-platform-governance`'s
prompt registry and promotion gate.

**Dependency edges pointing outside this feature.** `TS-BL-033` needs `access-control-and-admin`'s
`TS-BL-018`; `TS-BL-034` needs `ai-platform-governance`'s `TS-BL-030`. Four further interfaces are
built against without being `depends_on` edges, because the work is buildable without them:
`platform-core`'s `TS-BL-004` (the workflow engine `TS-BL-038` registers against), `TS-BL-006` (the
dispatch pattern `TS-BL-038` publishes through), `TS-BL-005` (internal notification for the review
queue), and `access-control-and-admin`'s `TS-BL-020` (the durable audit writer). Where an interface
has not landed, build against its declared shape and say so — the pattern Phase 1 used throughout.

**One production-activation obligation that is not a dependency edge.**
`ai-platform-governance`'s promotion gate refuses to activate a template version in production with
no passing corpus run recorded for it. `TS-BL-034` and `TS-BL-039` each author a prompt family, so
each needs cases in `TS-BL-032`'s corpus before it is live **in production** — but both are fully
buildable and deployable to Dev without it, which is why D.10 records no edge and why none is added
here. `ai-platform-governance` `design.md` D8 argues that the gate biting before the corpus exists is
the correct order; the obligation is stated on both items below rather than left to surface at a
promotion attempt.

**Two item-boundary refinements this feature makes**, under D.11's permission to refine internal item
boundaries, recorded in `design.md` D11: `TS-BL-033` creates versioned JD records from the first save
(because `TS-BL-034`'s already-fixed envelope takes `jd_draft_ref: <id@version>` and sits two items
earlier), and `TS-BL-039` carries the `job_posting_generation` prompt content as well as `G-08`'s
storage separation. **Neither changes a dependency edge, and neither moves work between features.**

**One decision is deliberately open and must not be resolved during apply.** `OD-005` — legal
sign-off on jurisdictional disclosure text — is owned outside engineering. `G-09` already decided how
to proceed without it: build the field and the gate, default to `not_required`. Do not build a hard
gate, and do not invent the disclosure text.

---

## 1. TS-BL-033 — JD drafting workspace, the Practice Manager drafts

**Goal:** a Practice Manager can write a complete, validated job description by hand, every free-text
section is bounded from the moment the schema exists, and the draft is already a versioned record
that something else can address. Covers `hiring/jd-workspace`.

```yaml
backlog_items:
  - id: TS-BL-033
    feature: hiring-postings
    depends_on: [TS-BL-018]
    status: not-started
```

- [ ] 1.1 Migrate `job_descriptions` and `job_description_versions` by reversible migration —
      parent record with title, practice, hiring manager, status and current version; version record
      with sequential number, structured input, generated content, human-edited content, approval
      metadata and source AI run reference (§12.2)
- [ ] 1.2 Implement the typed structured-input schema of `design.md` D1 — job title, practice, hiring
      manager, experience range, urgency, role summary, responsibilities, required skills, preferred
      skills, qualifications — validated on write, rejecting undeclared fields rather than storing
      them (`JOB-002`)
- [ ] 1.3 Implement work location, work mode and employment type as **JD-level defaults that seed a
      posting**, and record in the schema documentation that the posting's values are authoritative
      for that posting (`design.md` D2)
- [ ] 1.4 Reject vacancy count and evergreen flag as undeclared fields on a job description, and
      leave a comment citing `design.md` D2 — this is a deliberate divergence from `JOB-002`'s literal
      wording in favour of §12.2's table and `domain-model.md`'s one-JD-to-N-postings cardinality, and
      it will otherwise read as an omission
- [ ] 1.5 Implement the per-field bound validator: sentence, item and character counts enforced on
      write, applied identically to human-typed and model-produced content, with a field-level error
      naming the bound (`S.5`, `AGENTS.md`'s standing bar)
- [ ] 1.6 Add the registration-time check that fails the suite if the schema declares a free-text
      field with no bound — the mechanism that stops the standard being followed for nine fields and
      forgotten for the tenth (`ai-platform-governance` D7's argument, applied to the schema)
- [ ] 1.7 Hold every bound in the audited runtime-configuration registry with machine-readable
      provenance marking the value provisional, so changing one is an audited configuration change
      rather than a deployment (`S.5` is explicit that the numbers were never given and should not be
      invented; `design.md` D1)
- [ ] 1.8 Create a version record on first save with a sequential number, and make a draft addressable
      as identifier-and-version — required by `TS-BL-034`'s already-fixed `jd_draft_ref` envelope
      field two items before versioning appears in D.10's titles (`design.md` D3 point 1, D11)
- [ ] 1.9 Implement the job-description endpoints this item owns — create (`POST /api/job-descriptions`)
      and the draft save/edit path — each declaring its permission requirement, taking verdicts from
      the central evaluator and never from logic local to this module. §13.2 is representative rather
      than exhaustive and lists no JD update route; the three remaining JD endpoints it does list are
      built by the items that own their behavior — `generate` by `TS-BL-034`, version listing by
      `TS-BL-036` (task 4.5), version approval by `TS-BL-035` (task 3.3). The seven posting endpoints
      in the same §13.2 table belong to `TS-BL-037` (task 5.11) and `TS-BL-038` (task 6.13). Depends
      on `TS-BL-018`; build against its interface if it has not landed
- [ ] 1.10 Deny create and edit on job descriptions to the Recruiter role, and assert the denial is
      server-side rather than only a hidden control (`C-12`, §8, `UI-002`)
- [ ] 1.11 Build the JD Workspace screen from `design-system`'s form controls, field wrapper, page
      template and dense data table — adding no new component — showing state, owner, next action,
      blockers and last-updated time (`UI-003`, §14.2)
- [ ] 1.12 Render permission-aware affordances as **absent** rather than disabled, and assert the API
      rejects an edit submitted directly by a viewer who cannot edit (`design-system`'s app-shell
      requirement; `UI-002`)
- [ ] 1.13 Surface validation errors at field level and in a summary (`UI-007`)
- [ ] 1.14 Write every material job-description write through the audit path in the same transaction,
      carrying references and diffs and never personal data. Build against `TS-BL-020`'s durable
      writer; `app/audit/port.py` already exists and needs no call-site change when it substitutes
- [ ] 1.15 Ship the workspace behind a feature flag, disabled by default, so no window exists in which
      a partly-built surface is reachable — a disabled capability answers 404

---

## 2. TS-BL-034 — AI-drafted JD generation

**Goal:** a Practice Manager can ask for a first draft, the request goes through the one governed
egress in the exact envelope that feature already fixed, the output is bounded field by field, and
nothing about it can approve itself. Covers `hiring/jd-ai-drafting`.

```yaml
backlog_items:
  - id: TS-BL-034
    feature: hiring-postings
    depends_on: [TS-BL-033, TS-BL-030]
    status: not-started
```

**Read `ai-platform-governance`'s `design.md` D3 and D4 and its `ai-platform/ai-gateway` spec before
starting.** The request and response envelope was designed against *this item* as its worked example,
which means the envelope is not negotiable here and the payload inside it is entirely this item's —
D3 says so in one sentence: *"no JD schema is fixed, and `hiring-postings` remains free to refine its
own input shape."* Building a different envelope would invalidate the reasoning behind a shipped
Phase 1 contract.

**Production activation obligation:** the promotion gate refuses this family's new template version in
production until `TS-BL-032`'s corpus carries passing cases for it. Deployable to Dev without them.

- [ ] 2.1 Author the `job_description_generation` prompt content against the schema `TS-BL-033`
      built, replacing `TS-BL-030`'s deliberate stub — as a **new template version**, never by editing
      an existing one
- [ ] 2.2 Declare the family's output contract: the structured job description object, with each
      free-text field carrying the same bound the schema declares for it, so the contract restates the
      field's own bound rather than introducing a second number (`design.md` D1)
- [ ] 2.3 Implement markdown as a **deterministic application-side rendering of the validated
      structured object**, not a model-authored document — the concrete resolution of the tension
      `ai-platform-governance` D3 point 3 named and left open for want of a JD schema (`design.md` D4)
- [ ] 2.4 Assert markdown content cannot differ from the fields that were validated, which is what
      makes §16.3's "Markdown plus structured JSON summary" checkable rather than trusted
- [ ] 2.5 Build the request payload inside the fixed envelope: `jd_draft_ref` as identifier-and-version
      and bounded `notes`, with **no job description content in the request** — the property that makes
      the run log's reference-only requirement natural rather than a redaction step
- [ ] 2.6 Declare and enforce the `notes` bound the envelope leaves as "bounded" without a number —
      provisional, labelled, audited-configurable, so the notes field cannot become a back door for
      the unbounded prose the JD sections forbid (`design.md` D3 point 2)
- [ ] 2.7 Handle the three typed failure kinds — validation, provider, configuration — reporting the
      kind to the user and leaving the draft unchanged, and assert a run reference is returned on
      every outcome including failure
- [ ] 2.8 Persist output only as draft content carrying its AI-generated marking, and assert
      generation advances no approval state — the JD-side statement of `AI-010` and `JOB-005`
- [ ] 2.9 Retain generated content separately from human-edited content so the provenance survives an
      edit as AI-assisted rather than being discarded (§12.2's two columns; `JOB-006` needs both to
      compare)
- [ ] 2.10 Clear the marking only on a recorded human approval with actor and timestamp, and only for
      that version (`AI-001`, `UI-004`)
- [ ] 2.11 Require the **Run AI** action on the JD Workspace page, distinct from view and edit, with
      the control absent for users who lack it (§9.3, `C-02`)
- [ ] 2.12 Verify graceful degradation: with the provider unavailable, the generation control reports
      temporary unavailability and every non-AI job description action — create, edit, save, submit,
      approve — remains fully usable (`G-12`)
- [ ] 2.13 Assert no outbound call to a model provider exists anywhere in this feature's code; every
      invocation goes through the gateway
- [ ] 2.14 Add the family's cases to `TS-BL-032`'s corpus — low-information input, conciseness per
      bounded field, and contract conformance — and record that production activation is refused until
      they pass (`ai-platform-governance` D8)

---

## 3. TS-BL-035 — JD approval workflow, the Recruitment Manager approves

**Goal:** the gate is real rather than a formality — the person who owns the headcount is not the
person who signs off on it, and no configuration or role combination lets the drafter approve their
own work. Covers `hiring/jd-approval`.

```yaml
backlog_items:
  - id: TS-BL-035
    feature: hiring-postings
    depends_on: [TS-BL-034]
    status: not-started
```

**Read `C-12`, not `D12` alone.** `D12`'s body still reads "Practice Manager approves"; `C-12`'s
2026-08-14 resolution amends it to **PM drafts, RM approves** and removes the Recruiter from the flow.
This is the document's consistent house style rather than a defect — `D02` and `D07`'s bodies are
unamended the same way — but it means the anchor alone will give you the superseded answer
(`design.md` D10).

- [ ] 3.1 Register the job description approval machine — draft → submitted → approved, with
      request-changes returning to draft — declaratively against `platform-core`'s workflow service,
      declaring the permission and reason requirement per transition. Build against `TS-BL-004`'s
      interface if it has not landed
- [ ] 3.2 Assert no surface writes a job description's approval state directly, and that an invalid
      transition is rejected naming the current state and what is available from it (`WF-001`,
      `WF-002`)
- [ ] 3.3 Implement submission by an authorized Practice Manager, and approval by a Recruitment
      Manager, recording approver and approval time (`C-12`)
- [ ] 3.4 Implement the drafter prohibition **on actor identity rather than on role**, so a user
      holding both Practice Manager and Recruitment Manager cannot approve their own draft — the
      self-approval case `D12` rejected "wearing a different hat," and the reason `C-12` wrote "never
      the drafter" rather than "not a Practice Manager"
- [ ] 3.5 Implement the fallback approver: where no Recruitment Manager is assigned, another Practice
      Manager who is not the drafter, recorded as having used the fallback path. Assert no
      configuration permits the drafter to be the fallback, and that approval is refused with the
      blocker stated when the only candidate is the drafter (`C-12`'s "Fallback required")
- [ ] 3.6 Implement request-changes with a mandatory reason, returning the version to draft with the
      reason visible to the drafter, rejecting a submission with no reason as a field-level error
      (`WF-005`)
- [ ] 3.7 Implement review comments retained against the version across resubmission (`D12`'s
      "approve / request-changes loop, plus comments")
- [ ] 3.8 Build the Recruitment Manager review queue on `design-system`'s dense data table, showing
      state, drafter, submission time and waiting time, and denying a request from a user without
      approval permission rather than returning an empty list — the queue belongs to the RM, which is
      what `C-12` changed from `D12`
- [ ] 3.9 Deliver submission notification through `platform-core`'s internal notification engine, and
      assert no candidate-facing delivery path exists (`D04`). Build against `TS-BL-005`'s interface
      if it has not landed
- [ ] 3.10 Verify the approval and its audit record are written in one transaction, and that a failed
      audit write fails the approval

---

## 4. TS-BL-036 — JD versioning and fork-on-edit

**Goal:** an approved job description cannot change under a posting that was published against it, so
every ranking score computed against that JD stays explicable. Covers `hiring/jd-versioning`.

```yaml
backlog_items:
  - id: TS-BL-036
    feature: hiring-postings
    depends_on: [TS-BL-035]
    status: not-started
```

**The version *rows* already exist** — `TS-BL-033` task 1.8 creates them, because `TS-BL-034`'s fixed
envelope needed them two items earlier. This item adds immutability after approval, the fork rule, the
comparison view, and pinning (`design.md` D11).

- [ ] 4.1 Make an approved version immutable, rejecting any write against its content
- [ ] 4.2 Implement fork-on-edit: editing an approved version creates the next sequential version in
      draft, leaving the approved version's content unchanged, and requiring its own approval before
      any posting can link to it (`D12`, retained by `C-12`)
- [ ] 4.3 Make `job_postings.job_description_version_id` settable while the posting is in draft and
      **immutable from the open transition onward**, rejecting a re-point on an open posting
- [ ] 4.4 Assert approving a later version changes nothing about a posting already open — the property
      that keeps `RankingScore = f(resume_version, jd_version, prompt_template_version,
      model_version)` meaningful, and whose absence `D12` describes as silently invalidating every
      ranking score computed against that JD (`design.md` D9)
- [ ] 4.5 Implement version listing with number, state, author and approval metadata (§13.2)
- [ ] 4.6 Implement comparison of AI-generated against human-edited content within a version, and of
      two versions field by field (`JOB-006`) — load-bearing rather than convenient, since fork-on-edit
      makes a one-word correction a new version needing a new approval
- [ ] 4.7 Verify a version reference resolves to that version's content after later versions exist,
      including when resolved as an AI run's input reference — the property
      `ai-platform-governance`'s evidence-labeling spec depends on when it states that a claim on a
      versioned record must name the version, "since `D12` forks job descriptions on edit"
- [ ] 4.8 Verify a superseded version remains retrievable after the posting that pinned it has closed
      (`RET-003`, `WF-007`)

---

## 5. TS-BL-037 — Job Posting creation

**Goal:** the record the entire hiring spine attaches to exists, carries its vacancy mode, and makes
three already-written specs in other features exercisable against real data for the first time. Covers
`hiring/job-posting`.

```yaml
backlog_items:
  - id: TS-BL-037
    feature: hiring-postings
    depends_on: [TS-BL-036]
    status: not-started
```

- [ ] 5.1 Migrate `job_postings` by reversible migration with the §12.2 columns, including `owner_id`,
      `recruiter_ids[]`, `interview_panel_ids[]`, `work_mode`, `employment_type`, `priority`,
      `target_start_date`, `opened_at` and `closed_at` (`R.4`). **Do not create `vacancy_slots`** —
      slots are reserved and filled by `TS-BL-064` and the offer path
- [ ] 5.2 Enforce exactly one job description version reference, approved at the time it is set
      (`JOB-007`), and permit several postings from one approved version
- [ ] 5.3 Implement `vacancy_type: FINITE | EVERGREEN` — finite requires a positive integer count,
      evergreen carries none and creates no fixed slots, and a count supplied on an evergreen posting
      is rejected rather than silently ignored (`JOB-003`, `JOB-004`, `D14`)
- [ ] 5.4 Implement the vacancy-mode immutability rule `D14` explicitly left open and asked for:
      editable in draft and pending approval, immutable from the open transition, with a count
      increase on an open finite posting also refused (`design.md` D12)
- [ ] 5.5 Seed work location, work mode and employment type from the referenced job description
      version, and make the posting's values authoritative — a posting differing from its JD is
      legitimate, not an inconsistency to reconcile (`design.md` D2)
- [ ] 5.6 Implement practice as a data attribute usable for filtering and reporting, and assert it is
      **never** evaluated as an authorization boundary (`D02`, `glossary.md`)
- [ ] 5.6a Hold the practice vocabulary in `platform-core`'s audited runtime-configuration registry
      as a closed list, seeded per `ENG-010`, and reject a posting whose practice is not in it —
      selected from, never typed (`design.md` D13)
- [ ] 5.6b Assert the vocabulary's audit properties, mirroring `interview-pipeline`'s
      disposition-reason-code requirement: adding, retiring or renaming a practice is audited with
      previous value, new value and mandatory reason; no unaudited setter exists; and a posting
      recorded against a since-retired practice remains readable under the value it was recorded
      with — the property that stops a rename splitting a practice's history
- [ ] 5.7 Implement owner, recruiter and interview-panel assignment, gated on the **Assign** action
      (§9.3)
- [ ] 5.8 Verify `access-control-and-admin`'s `assigned-postings` scope predicate against a populated
      table — its authorization spec cites `job_postings.recruiter_ids[]` by name and has so far only
      been asserted against a table that does not exist. Verify a Recruiter's write on an unassigned
      posting is denied while their **read** is permitted, which is `C-03` as resolved: writes
      assignment-scoped, reads governed by the matrix
- [ ] 5.9 Verify an Interviewer or Hiring Panel Member reading postings is scoped to their own
      assignments (`D01`, `D02`)
- [ ] 5.10 Verify the scope filter is applied as a query predicate rather than by discarding fetched
      rows, per the authorization spec's own requirement
- [ ] 5.11 Implement the posting endpoints §13.2 enumerates for create and update, each declaring its
      permission requirement
- [ ] 5.12 Build the Job Posting Detail screen and the posting list from `design-system`'s page
      templates and dense data table, showing posting fields, assigned recruiters, interview panel and
      vacancy settings (§14.2), adding no new component
- [ ] 5.13 Write every posting create and update through the audit path in the same transaction with
      previous and new value, and assert a failed audit write fails the operation

---

## 6. TS-BL-038 — Posting status lifecycle

**Goal:** the product's first registered state machine, with a blocked posting that can be seen and
explained rather than a failed action that can only be retried — and no automation, which is exactly
the room evergreen needs. Covers `hiring/posting-lifecycle`.

```yaml
backlog_items:
  - id: TS-BL-038
    feature: hiring-postings
    depends_on: [TS-BL-037]
    status: not-started
```

**Read `design.md` D5 before starting.** The question this item answers is narrow: what must be
registered so that `TS-BL-070` adds evergreen rules rather than unpicking these. The answer —
register no automatic transition — is the complete and correct content of the item, not a stub. Do
not register a `vacancy_type` guard set; `D14` chose one flag and one code path specifically to avoid
a second one, and `platform-core`'s registration surface offers no guards.

- [ ] 6.1 Register the posting machine declaratively — draft, pending approval, open, screening,
      interviewing, scorecard review, selection, offer, onboarding, filled, paused, cancelled, closed
      — with §11.1's transition set, per-transition permissions and reason requirements. This is the
      **first machine registered in the product**; verify that `platform-core`'s deploy-time
      "framework ships with no machines" assertion is about the framework's own deployment and does
      not fail on a domain feature registering one (`design.md` Migration Plan step 5)
- [ ] 6.2 Assert one machine governs both vacancy types, and that no `vacancy_type`-keyed guard exists
      (`D14`, `design.md` D5)
- [ ] 6.3 Assert every registered transition is actor-initiated, and add the test that fails if a
      transition is registered that fires without an authenticated actor — no timer, threshold, data
      condition or configuration change
- [ ] 6.4 Register `filled` and `closed` as reachable states with **no automatic transition into
      either**, leaving finite auto-close to `TS-BL-069` and evergreen's manual-only rule to
      `TS-BL-070` — which then needs to add nothing at all, exactly as `D14` predicted
- [ ] 6.5 Implement draft → pending approval as the owner declaring readiness, and pending approval →
      open as validation passing — introducing **no human approver for posting content**, since
      `C-12` put that gate on the job description. Assert the machine's transition list contains no
      content-approval step (`design.md` D6)
- [ ] 6.6 Implement `JOB-009`'s open-validation checklist — required fields, vacancy mode, linked JD
      version approval status, compliance text status, owner, at least one assigned recruiter, panel
      configuration where required — reporting **every** unsatisfied item rather than the first
- [ ] 6.7 Render the blocked posting as a blocker list rather than an error toast, satisfying
      `UI-003`'s requirement that a page show state, owner, next action and blockers — the reason D6
      keeps the state rather than validating on the transition alone
- [ ] 6.8 Implement pause, reopen, cancel and close across §11.1's transition set, requiring a reason
      into cancelled and closed and rejecting a submission with none (`WF-005`)
- [ ] 6.9 Verify a cancelled or closed posting retains its full transition history and stays viewable,
      continuing to inform candidate history and analytics (`WF-006`, `WF-007`, `RET-003`)
- [ ] 6.10 Publish a `posting.opened` event through `platform-core`'s **single dispatch pattern**,
      carrying the opening request's correlation identifier, and register **no subscriber** —
      `JOB-010`'s consumers are `TS-BL-052` and `TS-BL-075`, both unbuilt (`design.md` D7). Build
      against `TS-BL-006`'s interface if it has not landed
- [ ] 6.11 Assert the open transition does not depend on publication: a publish failure leaves the
      posting open and is visible rather than silently dropped, so candidate matching never becomes
      load-bearing for opening a posting (`G-12`)
- [ ] 6.12 Include in the event what a consumer needs to detect a repeat, since the dispatch substrate
      delivers at least once and idempotency is the consumer's obligation
- [ ] 6.13 Implement the posting transition endpoints §13.2 enumerates — open, pause, reopen, cancel,
      close — each declaring its permission requirement and taking verdicts from the central evaluator
- [ ] 6.14 Verify every transition records actor, prior state, new state, reason where required,
      timestamp, correlation identifier and source module, and that an unauthenticated transition
      attempt is rejected (`WF-003`, `WF-004`)

---

## 7. TS-BL-039 — Posting text separate from the internal JD

**Goal:** what a candidate would read and what the business decided are two artifacts with two
lifecycles, and the AI has a second clearly-scoped target instead of one overloaded one. Covers
`hiring/posting-text`.

```yaml
backlog_items:
  - id: TS-BL-039
    feature: hiring-postings
    depends_on: [TS-BL-037]
    status: not-started
```

**Scope note:** this item carries the `job_posting_generation` prompt content as well as `G-08`'s
storage separation. D.10's title names only the separation, but `G-08`'s own rationale for it is that
it "gives the AI a second clearly-scoped generation target (`job_posting_generation`)", §16.1 lists
Job Posting Text Generation as its own capability with its own human gate, and no other item among the
eighty covers it (`design.md` D11). The separation stands independently — tasks 7.1–7.4 are complete
without a single AI call.

**Production activation obligation:** as with `TS-BL-034`, the promotion gate refuses this family's
new template version in production until `TS-BL-032`'s corpus carries passing cases for it.

- [ ] 7.1 Migrate the posting text artifact as its own record against the posting, distinct from the
      job description version's content (`JOB-008`, `G-08`)
- [ ] 7.2 Assert reading a posting's external text never returns internal job description content, and
      that neither is derived from the other at read time
- [ ] 7.3 Make posting text editable independently of the job description's approval state, and assert
      approving a later JD version alters, invalidates and regenerates nothing — building propagation
      would recreate the coupling `JOB-008` exists to break (`design.md` Non-Goals)
- [ ] 7.4 Verify a posting can open with entirely hand-written text and no AI run having occurred
- [ ] 7.5 Author the `job_posting_generation` prompt content as a **new template version** of the
      family `TS-BL-030` registered, never by editing an existing version
- [ ] 7.6 Build the request through the gateway carrying the approved JD version reference and posting
      metadata reference — references, not content (§16.1 row 2)
- [ ] 7.7 Declare the output contract as markdown plus compliance text flags (§16.3), with a bound on
      the markdown body **as a free-text field in its own right** — unlike the JD, this output is prose
      whose shaping is the point, so `design.md` D4's compose-from-bounded-parts answer deliberately
      does not transfer (`design.md` D11)
- [ ] 7.8 Record a failed run with the violation preserved and write no content when output exceeds the
      bound, and hold the bound in audited configuration with provisional provenance
- [ ] 7.9 Require the **Run AI** action, with the control absent for users who lack it, and mark
      generated text AI-generated until a human approves it (§16.1's "Human approval required")
- [ ] 7.10 Surface returned compliance flags as a **proposal a human confirms**, and assert generation
      never sets `compliance_text_status` — `G-01`'s adopted pattern is that AI proposes and a human
      confirms anything that blocks
- [ ] 7.11 Add the family's cases to `TS-BL-032`'s corpus — conciseness against the markdown bound,
      contract conformance, and compliance-flag output — and record that production activation is
      refused until they pass

---

## 8. TS-BL-040 — Compliance text handling

**Goal:** a jurisdictional exposure nobody can currently size becomes a concrete field with a gate
that is off today and switched on by configuration, not by a rebuild, whenever Legal lands. Covers
`hiring/posting-compliance`.

```yaml
backlog_items:
  - id: TS-BL-040
    feature: hiring-postings
    depends_on: [TS-BL-039]
    status: not-started
```

**`OD-005` stays open, and this item does not close it.** `G-09` already decided how to proceed
without it: build the field and the posting-open gate now, defaulting to `not_required`, because "a
hard gate from day one would block every posting on a timeline outside our control." Do not invent the
disclosure text and do not build a hard gate.

- [ ] 8.1 Add `compliance_text_status: missing | valid | not_required` to `job_postings` by reversible
      migration, rejecting any value outside the three (`G-09`, §12.2)
- [ ] 8.2 Stamp the status at posting creation from the configured default, as the posting's own
      stored field rather than a value computed at read time — a computed status cannot record that a
      posting was validated against a *specific* required text, which is the audit question the field
      exists to answer (`design.md` D8)
- [ ] 8.3 Hold the default and the required disclosure text in `platform-core`'s audited
      runtime-configuration registry — §29 item 11 already lists this among the values configurable
      without code changes — so the flip `G-09` anticipates is an audited change with a mandatory
      reason, previous and new value, and no deployment
- [ ] 8.4 Seed the configured default at `not_required`, which is the state `G-09` requires on day one
- [ ] 8.5 Assert no unaudited setter exists for either value
- [ ] 8.6 Wire the status into `TS-BL-038`'s open-validation checklist: `missing` blocks and is named
      among the unsatisfied items; `valid` and `not_required` pass (`JOB-009`)
- [ ] 8.7 Assert a configuration change **never** transitions a posting that is already open — that
      would be an automatic transition on a governed state field, which task 6.3 prohibits and
      `project.md`'s central rule prohibits generally (`design.md` D8)
- [ ] 8.8 Build the review list of open postings whose stored status no longer satisfies current
      configuration, stating the reason, so a human closes or amends each one through an ordinary
      permission-gated audited action
- [ ] 8.9 Implement setting the status to `valid` as an explicit permission-gated human action recorded
      with actor and timestamp, and assert no AI output can set it (`G-01`, `AI-010`)
- [ ] 8.10 Verify the end-to-end flip: with the default changed to `missing`, a newly created posting
      is stamped `missing`, its open transition is refused naming compliance text, and every posting
      already open is untouched — the scenario `G-09` describes as "a configuration change, not a
      rebuild"

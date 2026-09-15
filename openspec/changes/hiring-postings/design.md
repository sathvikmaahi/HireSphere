# Hiring — Job Descriptions and Postings — Design

## Context

See `proposal.md` — Why, for motivation, and the eight delta specs under `specs/` for the behavior
contracts. This document covers only what this feature must actually settle: the JD content model
(which nothing upstream fixed, deliberately), the reconciliations `JOB-002` and §11.1 force against
decisions made after the reference spec was written, and how much of `EVERGREEN` belongs here versus
`TS-BL-070`.

Constraints that shape the approach:

- **The request envelope for AI-drafted JD generation is already fixed, and this feature did not fix
  it.** `ai-platform-governance`'s `design.md` D3 designed the gateway envelope *against*
  `TS-BL-034` as its worked example, precisely because it was "the first AI consumer in dependency
  order, and the only one whose inputs are fully specified today." The envelope is a contract this
  feature consumes. What D3 deliberately left open is stated there in one sentence: *"no JD schema
  is fixed, and `hiring-postings` remains free to refine its own input shape."* That sentence is
  most of D1 and D4 below.
- **Four substrates exist to be used, and each has an unexercised path this feature is the first to
  walk.** `platform-core`'s workflow engine ships with zero registered machines and asserts it at
  deploy time; `access-control-and-admin`'s `assigned-postings` scope predicate names
  `job_postings.recruiter_ids[]`, a column that does not exist; `ai-platform-governance`'s registry
  holds ten stub families and a promotion gate "expected to refuse everything until `TS-BL-032`
  lands"; `platform-core`'s dispatch pattern has one registered job type. This feature is the first
  real consumer of all four.
- **The reference spec predates two decisions that bear directly on this module.** `C-12` moved JD
  approval from an unnamed approver to the Recruitment Manager and removed the Recruiter from the
  flow entirely; `D14` made evergreen a flag on one state machine rather than a second object.
  §19.2, §11.1 and §12.2 were written before both. `AGENTS.md` is unambiguous about the precedence —
  *"wherever `exploration-notes.md` and a reference spec disagree, the exploration notes win"* — so
  the reconciliations below adopt the spec's structure and the exploration notes' semantics, and say
  where they part company. See D2, D6.
- **Settled stack** (`D09`): FastAPI on Python, PostgreSQL on Cloud SQL, React and TypeScript,
  Terraform on GCP with GitLab CI/CD, background work on the landing zone's Pub/Sub → Eventarc →
  Workflows → Cloud Run Job chain.
- **Only Local and Dev are provisionable.** UAT and Prod are validated Terraform that is never
  applied. This matters to one thing here: the promotion gate's "no passing corpus run in
  production" refusal cannot be observed in a real production environment, so it is asserted in Dev
  against production-mode configuration. Same position `ai-platform-governance` D8 recorded.

## Goals / Non-Goals

**Goals:**

- A JD content model precise enough that `TS-BL-034` has something real to generate into and
  `matching-and-ranking` has something real to rank against — fixed here because it was explicitly
  deferred to here, not because it is convenient to decide now.
- Every free-text field bounded at the point the schema is declared, so no field can reach the
  prompt contract without a bound and no bound is a comment.
- One posting state machine covering both vacancy types, registering nothing automatic — so
  `TS-BL-070` adds evergreen behavior by adding rules, not by unpicking one.
- Reconciliations stated as reconciliations. Where §19.2 or §11.1 is adopted with changed meaning,
  the change is written down with both readings, not silently resolved.
- A posting record whose columns make three already-written specs in other features exercisable for
  the first time, rather than requiring them to be revised.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No prompt engineering beyond two families.** Eight of the ten registered families stay stubs.
  Each belongs to the feature that consumes it, which is the handoff `ai-platform-governance`
  already recorded when it registered them.
- **No JD ↔ posting content synchronization.** `G-08` makes the posting text a distinct artifact.
  Regenerating it after a JD version changes is a human action with a visible prompt, not a
  propagation rule. Building propagation would recreate the coupling `JOB-008` exists to break.
- **No approval delegation model.** `C-12` requires exactly one fallback — another Practice Manager
  who is not the drafter. A general delegation/out-of-office system is a larger feature that
  `D12`'s consequences list raised as a question and nothing has answered.
- **No posting search beyond the dense-data-table's supplied capabilities.** Filtering by practice,
  state, owner and priority is the table pattern doing its job (`UI-008`); a saved-view or
  query-builder surface is not in any backlog item.
- **No aging or SLA behavior.** §29 item 9's posting aging thresholds are configuration this feature
  can read; the nudges and dashboards that act on them are `insight-and-reporting`.

## Decisions

### D1 — The JD content model is fixed here, with a declared bound on every free-text field

`JOB-002` enumerates the mandatory fields; §12.2 stores them as `structured_inputs_json`, a JSONB
column. Taken literally, that pairing is a schema-shaped hole: a JSONB blob satisfies the storage
requirement while satisfying no contract at all, and `TS-BL-034`'s generation target would be
undefined. **The structured input is a typed, validated schema that happens to be persisted as
JSONB**, not freeform JSON.

The model, with the bound each field carries:

| Field | Kind | Bound | Source |
|---|---|---|---|
| `job_title` | single-line text | 120 characters | `JOB-002` |
| `practice` | closed vocabulary | selected from the audited list, never typed | `JOB-002`, `D02` (attribute, not permission boundary); D13 |
| `hiring_manager_id` | user reference | — | `JOB-002` |
| `experience_min_years` / `experience_max_years` | integer range | max ≥ min | `JOB-002` |
| `urgency` | enum: `low` \| `medium` \| `high` \| `urgent` | — | `JOB-002` |
| `role_summary` | free text | 3 sentences | `S.5`, §16.3 |
| `responsibilities[]` | list of free text | 10 items × 1 sentence each | `JOB-002`, `S.5` |
| `required_skills[]` | list of short text | 15 items × 60 characters | `JOB-002` |
| `preferred_skills[]` | list of short text | 10 items × 60 characters | `JOB-002` |
| `qualifications[]` | list of free text | 6 items × 1 sentence each | `JOB-002` (education/certification) |

*Why bounds land on the schema rather than only on the prompt contract:* `S.5`'s stated concern is
that "cramming a long AI-written paragraph into small, dense type is precisely the overcompacted
result this requirement rules out" — and a **human**-typed paragraph produces exactly the same
overcompacted result. `TS-BL-033` is manual drafting; if the bound arrived only with `TS-BL-034`,
every JD written in the interim would be unbounded and the field would have to be narrowed later
against existing data. Binding at the schema makes the prompt contract's bound a restatement of the
field's own, which is the cheap direction.

*Why these numbers, given that `S.5` says the numbers were never given:* they are **provisional and
labelled provisional**, carrying the machine-readable provenance `ai-platform-governance` D7
mandates and changeable through the audited configuration path. D7 settled the general form of this
answer already — *"a wrong-but-labelled-and-enforced bound is fixable configuration; an absent one
is an unenforceable contract."* This feature is not re-deciding that; it is supplying the two
families' worth of values D7 said would have to be supplied by whoever writes the contracts.

*Alternative considered:* keep `structured_inputs_json` genuinely freeform and let each consumer
interpret it. Rejected — `matching-and-ranking`'s `RankingScore` is a function of `jd_version`
(`domain-model.md`), and §16.4's permitted ranking signals are keyed to *required skill match*,
*preferred skill match*, *experience relevance* and *mandatory criteria status*. Those signals are
not computable from a blob. A freeform JD would push schema invention into the ranking engine, where
it would be invisible to the person drafting the JD.

### D2 — `JOB-002`'s field list is the drafter's checklist; the posting owns its own copies

`JOB-002` requires the JD to carry work location, work mode, employment type, and "vacancy count or
evergreen flag." §12.2 puts all four on `job_postings` and none on `job_description_versions`. Both
cannot be authoritative, and `domain-model.md` explains why the spec's own table layout is the
better guide: **one JD version → N postings.** Two postings opened from one approved JD can sit in
different cities, and one can be finite while another is evergreen.

**Resolution:**

- `work_location`, `work_mode` and `employment_type` exist on the JD version as **defaults that seed
  a new posting**, and on the posting as the **authoritative value for that posting**. Divergence is
  legitimate and visible, not an inconsistency to reconcile.
- `vacancy_type` and `vacancy_count` exist **only** on the posting. `JOB-003` and `JOB-004` are both
  phrased about postings ("Finite **postings** shall require a positive integer vacancy count"), and
  §12.2's `vacancy_slots` hangs off `job_postings`. A vacancy count on a JD reusable by N postings
  has no meaning — is it per posting or across them? — and no requirement anywhere reads it.

*What this costs, stated rather than discovered:* a reader comparing `JOB-002` against the JD schema
will find one of its listed fields absent. That is a deliberate, cited divergence from `JOB-002`'s
literal wording in favour of §12.2's own table and `domain-model.md`'s cardinality, and it is
recorded here so the gap is a decision rather than an omission.

### D3 — `TS-BL-034` builds against a fixed envelope and promotes a template version; it adds nothing to the gateway

The envelope is settled and reproduced here only so the boundary is unambiguous — this feature
supplies the payload inside it and nothing else:

```
request:
  family:        job_description_generation
  caller:        { actor, page: JD Workspace, action: Run AI, correlation_id }
  input:         { jd_draft_ref: <id@version>, notes: <free text, bounded> }
response:
  run_ref:       <ai_run id>
  status:        succeeded | failed
  output:        contract-satisfying draft, or
  failure:       { kind: validation | provider | configuration, detail }
```

Three consequences that are this feature's to act on:

1. **`jd_draft_ref` is `id@version`, so a draft must already be a versioned record before AI can be
   run against it.** This lands earlier than the backlog titles suggest — `TS-BL-036` is where
   versioning appears, and it sits two items after `TS-BL-034`. The resolution is not a dependency
   change: `TS-BL-033` creates `job_description_versions` rows from the first save, so a draft is
   always *version n* of something. What `TS-BL-036` adds is the **fork-on-edit rule for approved
   versions**, immutability after approval, and `JOB-006`'s comparison view — not the existence of
   versions. Recorded as a boundary refinement in D11.
2. **`notes` is bounded, and this feature declares the bound.** The envelope says "free text,
   bounded" without a number, consistently with D7 leaving numbers to the contract author. Set at
   1,000 characters, provisional and labelled — long enough for a paragraph of role context, short
   enough that the notes field is not a back door for the unbounded prose the JD sections forbid.
3. **The family is promoted, not edited.** `TS-BL-030` registered `job_description_generation` with
   stub text and requires that "a content change creates a new version rather than modifying one."
   So `TS-BL-034` authors a **new template version** and activates it through the audited
   configuration path. Its production activation is refused until a passing corpus run exists for
   that version — which means `TS-BL-032`'s harness must carry cases for the family before this
   item is live in production, even though `TS-BL-032` is not one of `TS-BL-034`'s `depends_on`
   edges. That is not an oversight in D.10's graph: the item is buildable and deployable to Dev
   without it, and `ai-platform-governance` D8 records the refusal as the intended behavior. Stated
   as an obligation in `tasks.md` rather than added as an edge, matching how
   `ai-platform-governance` handled `TS-BL-018`, `TS-BL-020` and `TS-BL-006`.

*Alternative considered for (1):* make `TS-BL-034` depend on `TS-BL-036`. Rejected — it would invert
D.10's ordering (`TS-BL-036` depends on `TS-BL-035` depends on `TS-BL-034`) and create a cycle. The
real content of `TS-BL-036` is approval-related, and inspecting it shows the cycle is only apparent.

### D4 — The markdown output is composed from the bounded fields, not generated as a document

§16.3 requires `job_description_generation` to return *"Markdown plus structured JSON summary,"* and
`ai-platform-governance` D3 named the resulting tension precisely: a single bound over a whole JD
document "is either vacuous (large enough to permit a JD) or impossible (small enough to satisfy
`S.5`)." D3 resolved the *form* of the answer — the bound is declared per free-text field — and left
the markdown half unresolved, because it had no JD schema to resolve it against. This feature does.

**The model returns the structured object only. The markdown is rendered from it, deterministically,
by the application.** The markdown is therefore not a separately-bounded free-text field; its length
is the sum of the field bounds in D1 plus fixed template text, which is a *derived* number that
cannot drift from what was validated.

*Why this is the resolution rather than a dodge of §16.3:* §16.3's requirement is about what the
family *produces* for consumers — markdown for display, JSON for structure — and a rendered markdown
document satisfies it in the only way that is also checkable. Bounding a model-authored markdown
document requires either counting sentences across headings (a bound that measures the wrong thing)
or trusting the model to have obeyed a bound the validator cannot express against prose it did not
structure.

*The real trade-off, not hidden:* the model loses the ability to shape the document — ordering,
emphasis, section headings are the application's. For a job description, whose sections are fixed by
`JOB-002`, there is nothing to shape; for a family whose output genuinely is a document, this answer
would not transfer. That limit is stated so a later feature does not adopt it by analogy. The
posting text (D11) is the near case, and it is decided separately for exactly this reason.

### D5 — The posting machine registers no automatic transition, which is the room `EVERGREEN` needs

`D14` chose "evergreen is finite with n = unlimited" on the grounds of **one state machine, one
flag, one code path**, and predicted that evergreen would then be "nearly free" in the item that
builds the lifecycle. D.10 nevertheless assigns the evergreen *lifecycle* to `decision-and-offers`'
`TS-BL-070`, depending on both `TS-BL-069` (closure rules) and `TS-BL-038` (this item) — because
"evergreen is a variant of the posting lifecycle, not a bolt-on."

The question this feature has to answer is therefore narrow and concrete: what must `TS-BL-038`
register so that `TS-BL-070` adds rules rather than unpicking them?

**Answer: every transition `TS-BL-038` registers is actor-initiated, and it registers no rule that
fires on its own.** In particular `filled` is registered as a reachable state with no automatic
transition into it, and `closed` likewise. This is not a stub or a placeholder — it is the complete
and correct content of this item, for two independent reasons:

- **`D14`'s divergence between the two types is entirely about automation**: "finite = automatic per
  rules plus manual; evergreen = manual only." A machine with no automation is already correct for
  evergreen and correct-but-incomplete for finite, and the finite completion is the automatic
  close-on-fill rule that `G-13`/`TS-BL-069` owns anyway. So the split falls exactly on the
  item boundary D.10 drew.
- **An automatic transition here would be built on facts that do not exist.** Closing a finite
  posting on fulfilment requires vacancy slots (`TS-BL-064`), onboarding records and a valid Hubble
  ID (`G-13`, `TS-BL-068`/`TS-BL-069`). Registering the transition now would mean registering a
  guard over three unbuilt tables.

`TS-BL-037` still introduces `EVERGREEN` as a `vacancy_type` value with its validation rule
(`JOB-003`: finite requires a positive integer count; `JOB-004`: evergreen carries none and creates
no fixed slots), so the flag exists and is enforced from this feature onward. What does not exist
yet is any behavior that reads it — correctly, since `D14`'s consequences (no 5-cap, per-hire
time-to-fill, the staleness nudge) all belong to selection, closure and reporting.

*Alternative considered:* register two machines, or one machine with `vacancy_type` guards on
transitions. Rejected on `D14`'s own reasoning — it chose Option 1 over "evergreen with a rolling
cap" and "a separate object entirely" specifically to avoid a second code path, and a guard set
keyed on `vacancy_type` reintroduces one under a different name. `platform-core`'s registration
surface offers states, transitions, reason requirements and per-transition permissions; it offers no
guards, and this design does not ask it to grow any.

### D6 — `pending_approval` on the posting is a readiness gate, not a second content approval

§11.1 puts `Draft → PendingApproval → Open` on the posting. Under `D12`/`C-12` the approval gate for
job content is on the **JD** — the Recruitment Manager approves a JD version, and `JOB-007` requires
each posting to link to one already-approved version. Read naively, §11.1's `PendingApproval` is
therefore either redundant with the JD gate or a second human approver that `C-12` never introduced.

**Resolution: `pending_approval` is where a posting waits while `JOB-009`'s open-validation checklist
is unsatisfied.** The transition `draft → pending_approval` is the owner declaring the posting ready;
`pending_approval → open` succeeds when every item on the checklist passes and is rejected, naming
what is missing, when it does not. **No second human approver is introduced**, and "approval status"
in `JOB-009`'s own list of checks means the linked JD version's approval status — which is what that
phrase can mean once `C-12` has assigned the approving role.

*Why keep the state at all rather than validating on the `open` transition and dropping it:* §11.1
defines `PendingApproval → Cancelled`, so a posting can die in that state. That edge only means
something if postings actually sit there. A posting blocked on a missing interview panel or absent
compliance text is a real, visible, assignable condition — `UI-003` requires every page to show
"workflow state, owner, next action, blockers"; a blocked posting with no state to be blocked in has
no way to render that. The alternative turns each failed open into an error toast and loses the
queue.

### D7 — `JOB-010` emits an event and registers no handler

*"Opening a posting shall trigger existing-candidate matching as a background job."* Both consumers
are unbuilt: `matching-and-ranking`'s `TS-BL-052` and `resurfacing-and-communications`'
`TS-BL-075`, and §25 lists Candidate Ranking and Existing Candidate Resurfacing as two separate job
types both triggered by "posting open."

**`TS-BL-038` publishes a `posting.opened` event through `platform-core`'s single dispatch pattern
and registers no subscriber.** The trigger obligation is satisfied here, where the transition
happens; the work is registered by whoever does it. This is the same shape `platform-core` used for
its own registration surface and `ai-platform-governance` used for its envelope: build the seam
against the real consumer, do not build the consumer.

*The property that must hold and is easy to lose:* the transition must not become dependent on
delivery. If publishing fails, the posting is still open — `G-12`'s degradation reasoning applies
directly, and the alternative would make candidate matching load-bearing for opening a posting,
which nothing requires. Publication is at-least-once with idempotency at the consumer, which is what
`platform-core`'s async spec already requires of every job type.

*Alternative considered:* have `TS-BL-052` poll for newly opened postings. Rejected — §25 names the
trigger, and a poll would make `RANK-007`'s "existing candidate matches shall appear before or
alongside new submissions when a posting opens" true only within the polling interval.

### D8 — The compliance default is configuration, and flipping it never closes an open posting

`G-09` decided the field and the gate ship now with the default at `not_required`, and stated the
intended future: *"When Legal defines the required text, the default flips to `missing` and the gate
begins enforcing — a configuration change, not a rebuild."* Two things have to be true for that
sentence to hold, and neither is automatic.

1. **The default is a runtime-configuration value, not a code constant or a column default.** §29
   item 11 already lists "compliance disclosure text for job postings" as configurable without code
   changes, so this reads a key from `platform-core`'s audited runtime-configuration registry —
   meaning the flip is an audited change with a mandatory reason and previous/new value, per
   `config.yaml`'s standing rule.
2. **The flip must not retroactively act on postings that are already open.** `compliance_text_status`
   is stamped on the posting at creation from the then-current configured default and is thereafter
   the posting's own field. Flipping the configuration changes what *new* postings get and what the
   gate demands at the *next* open transition; it changes nothing about a posting already open.

*Why (2) is a decision and not an implementation detail:* the opposite behavior — re-evaluating open
postings against new configuration — would mean a configuration change silently transitioning live
postings out of `open`. That is an automatic transition on a governed state field, which D5 rules
out here and which `project.md`'s central rule rules out generally: material state changes rest with
an accountable human. Postings that become non-compliant under new configuration surface as a
**reviewable list** for their owners, and a human closes or amends each one.

*Alternative considered:* gate on the configured value at read time rather than stamping the field.
Rejected — `G-09` specifies a field with three values on the posting, and a computed-at-read status
cannot record that a posting was validated as compliant against a *specific* required text, which is
the audit question this field exists to answer.

### D9 — Fork-on-edit, and what "pinned" protects

`D12` retained through `C-12`: approved JDs fork on edit, and live postings stay pinned to the
version they were published against. Concretely:

- A version in `approved` is **immutable**. Editing it creates version *n+1* in `draft`, which
  requires its own RM approval before any posting can link to it.
- `job_postings.job_description_version_id` is settable while the posting is in `draft` and
  **immutable from the `open` transition onward**.
- Version *n* remains retrievable forever, including after *n+1* is approved and after the posting
  that pinned it is closed (`RET-003`, `WF-007`).

*What pinning actually protects, in one sentence from elsewhere in the project:* `domain-model.md`
defines `RankingScore = f(resume_version, jd_version, prompt_template_version, model_version)` and
says without that tuple "a score can't be explained or reproduced once the JD or the model changes
underneath it." Re-pointing an open posting at a new JD version would leave every ranking score on
it computed against a document the posting no longer references — not wrong-looking, just quietly
meaningless. `D12` says the same thing in its own words: edit-in-place "would silently invalidate
every ranking score computed against that JD."

*A consequence worth naming:* a typo in an approved JD on a live posting cannot be corrected in
place. It requires a new version, a new approval, and — if the correction must reach the live
posting — closing or re-drafting it. That is the intended cost of the guarantee, and it is why
`JOB-006`'s comparison view matters: an approver needs to see exactly what changed between *n* and
*n+1* to approve a typo fix cheaply.

### D10 — What Sprint 0 and `talentsphere-wave-1-foundation` left for this feature: nothing

Checked rather than assumed, because every Phase 1 feature's design carried an inheritance section
and their absence here would otherwise read as an omission.

- **`talentsphere-wave-1-foundation` has no `hiring/` delta spec.** Its `specs/` tree covers
  `access-control/`, `ai-platform/`, `design-system/`, `identity/` and `platform/`. No requirement
  needs redistributing into this feature, which is why `proposal.md` has no "overlap to resolve"
  block and the five Phase 1 features all did.
- **No `D1`–`D18` decision from that change's `design.md` was allocated here.** `platform-core`'s
  `design.md` D11 distributed all eighteen across the five Phase 1 features.
- **Sprint 0 built no JD or posting code.** Its handover states that its ~2,000 lines are "all of it
  platform-layer; **no hiring feature exists yet**, by design."
- **`KNOWN_ISSUES.md` carries nothing about JD or postings.** Its seven open items are environment
  provisioning, the unconfirmed Hubble contract, the not-yet-durable audit trail, the unexercised
  `audit_logs` grant, hand-applied schema ownership, `/build` metadata, and a blocked npm registry.
  Two of them touch this feature only in the way they touch every feature: JD and posting endpoints
  are covered by the permission mechanism arriving in `TS-BL-018`, and the durable audit writer must
  exist before this feature's material writes are auditable — `TS-BL-020`, already a Phase 1 item.

*One reconciliation deliberately not made.* `D12`'s body still reads "Practice Manager approves,"
and its amendment to "PM drafts, RM approves" lives at `C-12` with the decision-index `Ref` column as
the link between them. That is this document's consistent style, not a defect — `D02`'s body still
says "global within role" after `C-03` narrowed it, and `D07`'s still says "No vector store" after
`C-01` adopted Vertex AI Vector Search. `AGENTS.md`'s gap-fixing convention is for real errors, and
a house style applied uniformly across three decisions is not one. Anyone building `TS-BL-035` reads
`C-12`; this feature's specs cite it directly for that reason.

*One mislabel, recorded because it will recur.* The JD and posting requirements are `reference/spec.md`
**§19.2**, not §16 — §16 is AI Implementation Requirements. Both are load-bearing here (§16.1 row 1
and §16.3's registry entries are `TS-BL-034` and `TS-BL-039`'s), which is likely how the two got
conflated. No shared document contains the error, so there is nothing to correct under `AGENTS.md`'s
convention; it is noted so the next reader looking for `JOB-001` in §16 does not conclude it is
missing.

### D11 — Two item-boundary refinements, recorded rather than assumed

Both taken under D.11's permission for a feature's propose conversation to refine its internal item
boundaries. **Neither changes a dependency edge, and neither moves work between features.**

1. **`TS-BL-033` creates versioned JD records from the first save.** D.10's titles put "versioning"
   at `TS-BL-036`, but `TS-BL-034`'s fixed envelope takes `jd_draft_ref: <id@version>` and sits two
   items earlier. `TS-BL-033` therefore owns the version *row*; `TS-BL-036` owns immutability after
   approval, fork-on-edit, and `JOB-006`'s comparison. See D3, point 1.
2. **`TS-BL-039` carries the `job_posting_generation` prompt content as well as the separation.**
   Its D.10 title names only `G-08`'s storage separation, but `G-08`'s own stated rationale for that
   separation is that it "gives the AI a second clearly-scoped generation target
   (`job_posting_generation`) rather than overloading the JD," and §16.1 lists Job Posting Text
   Generation as its own capability with its own human gate. No other item among the eighty covers
   it. The separation requirement stands independently — posting text can be written by hand, and
   `TS-BL-039` is complete without a single AI call — so this is an addition to the item, not a
   reinterpretation of it.

*Note on (2)'s output shape, since D4 explicitly declined to generalize:* §16.3 requires
`job_posting_generation` to return *"Markdown plus compliance text flags."* Unlike a JD, external
posting copy genuinely is prose whose shaping is the point, so the model authors the markdown
directly and it is bounded as a free-text field in its own right — provisionally 400 words — rather
than being composed from parts. The compliance flags are structured and feed `TS-BL-040`'s status
field as a **suggestion a human confirms**, never as the value itself, because `G-01`'s adopted
pattern is that AI proposes and a human confirms anything that blocks.

### D12 — A posting's vacancy type is fixed once it opens

`D14` left this open in its own words — *"can a posting change type mid-life (evergreen → finite, or
the reverse)? Probably should be blocked; needs an explicit rule"* — and this is the item that
introduces the field, so this is where the rule belongs.

**`vacancy_type` and `vacancy_count` are editable while the posting is in `draft` or
`pending_approval`, and immutable from `open` onward.** Increasing `vacancy_count` on an open finite
posting is likewise refused; it is a new posting.

*Why immutable rather than "blocked with an override":*

- **`FINITE(n) → EVERGREEN`** would orphan vacancy slots (§12.2) that may already be reserved or
  filled, since `JOB-004` says evergreen "shall not create fixed vacancy slots." There is no defined
  answer for what happens to a filled slot on a posting that no longer has slots.
- **`EVERGREEN → FINITE(n)`** requires choosing *n* retroactively against a selection set that was
  accumulated with no cap — `D14` records that evergreen carries "no 5-cap on evergreen — selection
  is unbounded." Any *n* smaller than what is already in flight invalidates decisions humans already
  made.
- `D14`'s own inclination was to block it, and no requirement anywhere asks for the conversion.

*Alternative considered:* permit the change while the posting has no applications. Rejected as a
rule whose condition is harder to state correctly than the thing it permits, and which would still
need the immutability rule for every other case. `draft` already provides the window in which the
type is freely changeable.

### D13 — Practice is a closed audited vocabulary, not free text and not a reference table

*(added 2026-08-27, closing this feature's own Open Question)*

This document's Open Questions deferred `practice` explicitly — "`insight-and-reporting` is the
feature with a real stake, and it is unproposed." It is proposed now, and its `design.md` D12
answers it. That entry is removed from Open Questions and the answer recorded here.

**The stake is real, and it is not authorization.** `D02` settles that practice never gates access,
so the wrong answer was never expensive *there*. It is expensive in grouping: free text fragments
every group-by practice appears in — "Data & AI", "Data and AI" and "data-ai" become three
practices — and a practice renamed later silently splits its own history across two labels, which
makes every historical rate on `insight-and-reporting`'s `TS-BL-072` wrong in a way nobody notices.
`ADM-004` filters by practice and §32 makes it a candidate-search filter, so the fragmentation
surfaces in three places, not one.

**A reference table is not what fixes it.** The cheaper mechanism is already proven in this
backlog: `interview-pipeline`'s `interview/shortlisting` holds its disposition reason codes in the
platform's audited runtime configuration, where adding, retiring or renaming a code is itself
audited with previous value, new value and a mandatory reason, no unaudited setter exists, and
retiring a code leaves historical records readable under the code they were recorded with. Practice
takes that identical shape — same mechanism, different vocabulary. **No schema change to
`job_postings`, no new join, no new table**: the column stays a string, and what changes is that the
string is *selected from* a closed list rather than typed.

*Why not a reference table, stated plainly:* it would buy referential integrity this product does
not need — practice is not a permission boundary (`D02`), has no attributes of its own, and is read
by aggregation rather than joined for detail. The cost is a migration, a join on every posting read,
and a second thing to seed. Audited configuration gets the property that actually matters — one
canonical spelling per practice, with rename history preserved — for the price of a configuration
key.

*Scope, stated so it is not mistaken for a prerequisite:* this is **quality, not an enabler**.
`insight-and-reporting`'s dashboards group on whatever values exist today and are not blocked by it.
Nothing in this feature or any other waits on it.

## Risks / Trade-offs

**[The JD schema is fixed before its largest consumer exists.]** `matching-and-ranking` will rank
against these fields, and §16.4's permitted signals were the check used to shape them — but that
feature is unproposed and may find a field it needs. → Mitigated by the correction mechanism
`AGENTS.md` specifies: `/opsx:update` against this change, per-feature, once real work reveals the
gap. The risk is bounded because the alternative is worse — `TS-BL-034` cannot generate into an
undefined target, and D1 records why a freeform blob pushes the problem somewhere less visible.

**[Provisional conciseness bounds may be wrong in both directions.]** Too tight and a real job
description will not fit; too loose and the bound never fires, which reads identically to compliance
— `ai-platform-governance` D7 named that third case explicitly. → Mitigated on both sides: bounds are
audited configuration rather than code, and `TS-BL-032`'s harness fails a family that declares a
bound no case exercises, so a vacuous bound is a test failure rather than a silent pass.

**[This feature is the first real consumer of four unbuilt substrates.]** Every one of them —
workflow registration, the scope predicate, the promotion gate, the dispatch pattern — has been
specified and asserted only against its own empty state. The first genuine use is where interface
mismatches surface. → Mitigated by sequencing rather than by hope: all four are Phase 1 items,
`TS-BL-033`'s only declared dependency is `TS-BL-018`, and each task that consumes an unlanded
interface says so and builds against the declared interface, matching the pattern already used
across Phase 1.

**[`pending_approval` will read as a human approval gate to anyone who has not read D6.]** The state
name comes from §11.1 and its meaning here comes from `JOB-009`. → Mitigated by keeping the state
name (renaming it would diverge from the reference machine for a cosmetic reason and break the
traceability §11.1 provides) and by making the blocked-checklist rendering the state's visible
identity: the UI shows what is missing, never "awaiting approver."

**[Two prompt families cannot go live in production until an item outside this feature lands.]**
`TS-BL-034` and `TS-BL-039`'s template versions are refused activation in production without a
passing corpus run from `TS-BL-032`. → Not mitigated, and correctly so — `ai-platform-governance` D8
argues at length that the gate biting before the corpus exists is the right order. What is mitigated
is surprise: `tasks.md` states the obligation on both items rather than leaving it to be discovered
at a promotion attempt.

**[Fork-on-edit makes trivial corrections expensive.]** A typo on a live posting's JD needs a new
version and a new approval. → Mitigated only partly, by `JOB-006`'s comparison view making a
one-word diff cheap to approve. The residual cost is accepted deliberately; D9 records what it buys.

## Migration Plan

Greenfield feature on a live Dev environment with no hiring data. Order follows the backlog's own
dependency chain; each item is independently deployable, and every step is a reversible migration
plus a feature-flagged surface, per `config.yaml`'s standing rules.

1. **`TS-BL-033`** — migrate `job_descriptions` and `job_description_versions`; ship the typed schema
   with its bound validator and the JD Workspace behind a flag, disabled by default. No prior data
   exists, so the migration is additive with an empty-table rollback.
2. **`TS-BL-034`** — author the `job_description_generation` template as a **new version**; leave the
   stub version active until the new one has a recorded passing corpus run. Activation is a
   configuration change, so rollback is re-activating the prior version, not a deployment.
3. **`TS-BL-035`, `TS-BL-036`** — approval and versioning. `TS-BL-036`'s immutability applies from
   deployment forward; any Dev-era approved version predating it is grandfathered by the migration
   marking existing approved rows immutable, which is a no-op on an empty table and correct if it is
   not.
4. **`TS-BL-037`** — migrate `job_postings`. On deployment, `access-control-and-admin`'s
   `assigned-postings` scope predicate becomes exercisable for the first time; its behavior against a
   populated table is verified here rather than assumed from its own item.
5. **`TS-BL-038`** — register the posting state machine. This is the first machine registered against
   `platform-core`'s workflow engine, which asserts at deploy time that it ships with none — that
   assertion is about the *framework's* deployment, not a prohibition on later registration, and the
   distinction is verified before this step rather than after it fails.
6. **`TS-BL-039`, `TS-BL-040`** — posting text and compliance status. `TS-BL-040`'s configuration key
   is seeded at `not_required`, which is the state `G-09` requires on day one; the flip to `missing`
   is a later, audited configuration change made by whoever owns `OD-005`'s outcome.

**Rollback:** every migration is reversible and every surface is flagged, so rollback is a flag flip
followed by a down-migration where necessary. The one asymmetry: once a posting has opened, its
pinned JD version and its `vacancy_type` are immutable by design (D9, D12), so rolling back a
*deployment* does not roll back those facts. On Dev this is acceptable and expected; it is recorded
because it is the one place where redeploying is not fully symmetric.

## Open Questions

Deferrable without changing the specs, the approach, or the task breakdown.

- **The concrete per-field bound values.** Every field has a bound, the bound is enforced, and the
  numbers are labelled provisional and changeable through audited configuration. Which numbers are
  *right* needs real job descriptions from real practices, which do not exist. Settling them later
  changes configuration values, not a requirement.
- **Whether the RM review queue needs an SLA or stalled-review nudge.** `D12`'s consequences list
  raised it as a question — *"SLA or nudges on stalled reviews?"* — and nothing has answered it. The
  approve/request-changes loop and its notification are specified; a nudge reads the same records and
  adds no requirement to this feature. §29 item 9's configurable posting aging thresholds are where
  it would attach.
- **Whether a posting needs a confidential/restricted-visibility flag.** `D02` left this open
  ("confidential/executive requisitions may still need a restricted-visibility flag") and nothing
  since has decided it. It would be an additional scope predicate on an existing evaluator rather
  than a change to the posting lifecycle, so deferring it does not shape anything here.

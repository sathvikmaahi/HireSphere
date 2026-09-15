# Interview Pipeline — Design

## Context

See `proposal.md` — Why, for motivation, and the eight delta specs under `specs/` for the behaviour
contracts. This document covers only the technical decisions this feature must settle, the three
questions earlier features handed it by name, the two decisions it must build against a **reversal**
rather than an original, and the reconciliation checks behind the proposal's claim that nothing was
inherited.

Constraints that shape the approach:

- **Two of the decisions this feature is named after were reversed.** `C-08` amends `D24` — notes
  version forever with **no lock** and **no author-only rule**. `C-05` amends `D11` — **one**
  consolidated scorecard per Application, not one per round. `AGENTS.md` is explicit that where the
  exploration notes and a reference spec disagree the notes win; here the notes and the reference
  spec *agree*, and it is the notes' own earlier decisions that lost.
- **`C-08` removed a lock without removing the goal the lock served.** `D24`'s stated reason for
  freezing was *"guaranteeing the approved evaluation and its source evidence agree."* `C-08` kept
  that goal and left the mechanism as a recommendation marked **"Not yet decided"** in
  `exploration-notes.md`'s Open questions. This feature is where it stops being deferrable.
- **The workflow framework ships with no domain machines.** `platform-core`'s
  `platform/workflow-engine` spec states it explicitly and names this feature as a registrant. The
  Application's lifecycle exists nowhere yet.
- **`OD-007` leaves the scorecard dimensions unapproved by the business that specified them.** So
  `SCR-003` is a seeded default, not product truth — and the mechanism `C-06` pointed at for
  seeding it does not exist (D14).
- **Three prompt families are unauthored and two of them have no `D.10` dependency edge.** The
  registry is fixed at ten (`TS-BL-030`); this feature authors versions of three of them, and
  `D.10`'s table wires only one (D1).
- **The Interview Console's AI ranking context is not this feature's to compute.**
  `matching-and-ranking`'s `TS-BL-054` is a server-side projection built to be consumed here, and
  its spec carries a requirement that this feature build no second view of ranking output.
- **Settled stack** (`D09`): FastAPI on Python, PostgreSQL on Cloud SQL, Terraform on GCP with
  GitLab CI/CD, OpenTelemetry, background work on the landing zone's Pub/Sub → Eventarc → Workflows
  → Cloud Run Job chain.
- **Only Local and Dev are provisionable**, and no generation provider is configured (`OD-003`), so
  all three families run on `ai-platform-governance`'s deterministic stub adapter.

## Goals / Non-Goals

**Goals:**

- A disposition that is always accountable: an actor, a reason from a maintained vocabulary, a
  workflow transition, and an audit record — on the negative outcomes as much as the positive ones.
- An evaluation that cannot silently stop matching its evidence, achieved **without** the lock
  `C-08` removed.
- Attribution preserved through consolidation: one document to approve and carry, with each
  panelist's own recommendation still individually attributable and disagreement visible rather
  than averaged.
- Guarantees checkable by absence: no note edit that leaves an approved scorecard untouched, no
  scorecard for an uninterviewed candidate, no score in the console, no averaged recommendation, no
  second view of ranking output, no hard-coded dimension list.
- Answers to three questions three earlier features handed forward, and an honest boundary on the
  one that is only half this feature's to answer.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No interview quality measurement.** Whether a panelist's notes are *good* is a question this
  feature makes answerable — structured fields, evidence per dimension — and does not answer.
- **No scorecard weighting or aggregate scoring.** `OD-007` covers weightings and approves none;
  `C-05` reversed `D11` partly by establishing that consolidation is **not** averaging.
- **No interview-history search surface.** The source types are registered; the capability
  `C-01`'s scope note flagged as outside the original 19 features is not built (D6).
- **No second workflow engine, dispatch substrate, gateway, registry, evaluator, audit writer,
  disclosure store, or design-system component.** All are consumed.
- **No note or resume content in a prompt payload.** Three families, three envelopes, references
  only — the rule `ai-platform-governance` D3 fixed and every consumer since has honoured.

## Decisions

### D1 — Two of the ten AI families had no owner, and both belong here

**The gap, stated first.** `reference/spec.md` §16.1 lists ten AI capabilities and §16.3 registers
ten prompt families; `exploration-notes.md`'s Open questions enumerates the same ten. Eight are
allocated by `D.10`'s backlog through the item that authors them:

| Family | Owner | Wired by a `D.10` dependency edge? |
|---|---|---|
| `job_description_generation` | `hiring-postings` `TS-BL-034` | Yes — `TS-BL-030` |
| `job_posting_generation` | `hiring-postings` `TS-BL-037`/`039` | Yes |
| `resume_extraction` | `candidate-intake` `TS-BL-044` | Yes — `TS-BL-027` |
| `candidate_ranking`, `fitment_summary`, `gap_summary` | `matching-and-ranking` `TS-BL-052` | Yes — `TS-BL-027` |
| `scorecard_generation` | **this feature**, `TS-BL-060` | Yes — `TS-BL-027` |
| `resurfacing` | `resurfacing-and-communications` `TS-BL-075` | Yes — via `TS-BL-052` |
| **`interview_questions`** | **nobody** | **No** |
| **`interview_note_summary`** | **nobody** | **No** |

`D.10`'s Phase-3 table gives `TS-BL-058` and `TS-BL-059` a single dependency each — `TS-BL-057` and
`TS-BL-058` respectively — and neither reaches `ai-platform-governance`'s `TS-BL-027`. Two of the ten
families were therefore about to be registered by `TS-BL-030` and authored by no one. Recorded here
rather than fixed silently, and the edges are added under `D.11`'s permission for a feature's propose
conversation to refine its internal item boundaries. **Neither edge moves work between features** —
both land inside `interview-pipeline`, which is where §16.1's own trigger and audience columns put
them.

**`interview_questions` → `TS-BL-058`.** §16.1 triggers it on *"shortlist or interview setup"* and
§25 repeats that trigger; `INT-004` puts the questions in front of an **Interviewer**, and §12.2
hangs `question_set_id` on `interview_rounds`.

*Why not `TS-BL-057`, which is the item that literally performs "interview setup":* generation would
then produce a question set with no surface rendering it, no interviewer reading it, and no way to
exercise `INT-004`'s "use, edit, or ignore" affordance — a code path with no reader, which is
`platform-core` D5's silent-success class and the same argument `matching-and-ranking` D2 used for
declining to embed source types whose records do not exist. Instead `TS-BL-057` **publishes the
round-created event and registers no subscriber**, and `TS-BL-058` registers the Question Generation
job type against it. That seam is not novel: `hiring-postings` published `posting.opened` with no
subscriber and `matching-and-ranking` registered it, which is the shape this product already uses
for a producer that lands before its consumer.

*Why not trigger at shortlist, the other half of §16.1's trigger column:* questions are generated
*from* job requirements, fitment and gaps *for* a particular round, and `INT-002` attaches the set to
a round. At shortlist there is no round, no interview type and nowhere to attach the result. The
trigger is implemented as interview setup, and the shortlist half of §16.1's phrasing is recorded as
satisfied transitively — shortlisting is what makes setup possible (`SHL-005`).

**`interview_note_summary` → `TS-BL-059`.** §16.1 triggers it *"after note submission"*, §25 on
*"interview note submitted"*, and `INT-009` gates it on *at least one note submitted*. Note
submission happens in `TS-BL-058`; note **versioning** is `TS-BL-059`.

*Why the later item rather than the one that first submits a note:* §16.3's required output format
for this family is *"structured summary with note references."* A reference to a note that can be
edited afterwards is a reference to a **version**, and the version chain — with its
`is_current_version` pointer and its edit-creates-a-version rule — is `TS-BL-059`'s entire subject.
Authoring the family in `TS-BL-058` would mean building a summary that goes stale the first time
anyone edits a note, then revisiting it one item later to add the regeneration path. It also puts the
summary in the same item as the note-version event that `TS-BL-061`'s supersede mechanism consumes
(D3), so one item owns "a note changed, and here is everything that follows."

*Alternative considered:* give both families to `TS-BL-058` and keep `TS-BL-059` purely about
versioning, on the grounds that the console is where both outputs are read. Rejected for the staleness
argument above, and because it would make `TS-BL-058` a four-concern item (console, notes,
questions, summaries) against the finer grain `D.8.2` established for this backlog.

**What this closes:** all ten families now have an authoring item. `resurfacing` is the only one whose
owner is still unproposed, which is a sequencing fact rather than a gap.

### D2 — "Reason required on every disposition" is treated as adopted, and the label is not glossed over

`TS-BL-056`'s title says *reason required on every disposition*. `SHL-002` requires a reason on
**shortlisting** only. The broader rule traces to `exploration-notes.md`'s **"Recommendations made,
not yet formally decided"** table:

> **Mandatory reason on rejection** — Feature 10 specifies mandatory reason on *shortlisting*. The
> reason that matters legally and for reporting is the one on **rejection**. Make reason mandatory on
> every disposition, from an Admin-maintained code list plus optional free text.

That is the same status class as `D23a`, which `candidate-intake` handled by name rather than citing
as settled (`candidate-intake` D1). The same discipline applies, and the honest answer is that this
one splits into a settled half and an unratified half — which is a **narrower** unratified surface
than D23a's, not merely a lower-stakes one.

**The settled half — the reason requirement itself — is not actually resting on the recommendation.**
Three adopted sources already require it independently:

- **`WF-005`**: *"Terminal and negative states shall require a reason."* `platform-core`'s
  `platform/workflow-engine` spec carries this as a built requirement with a scenario that rejects a
  reasonless transition. §11.2's `NotSelected` is reachable from `Ranked`, `Shortlisted`,
  `Interviewed` and `ScorecardReady` — so **every rejection in this feature is already a
  reason-required transition** before the recommendation is considered at all.
- **`SHL-002` and `SHL-004`**: reason on shortlist, reason on shortlist removal with history
  preserved.
- **`domain-model.md`'s spine**, which is canonical and already renders the Application's shortlist
  node as *"Shortlist (reason **required on every disposition**)"*.

So the recommendation's substance is confirmed. What it adds beyond `WF-005` is narrow: **a
maintained coded vocabulary plus optional free text**, rather than a free-text reason field.

**That narrow part is treated as adopted**, on three grounds:

1. **The cost of being wrong is a configuration change, not a revert.** The code list lives in
   `platform-core`'s audited runtime-configuration registry. Rejecting it means seeding a single
   `other` code and letting free text carry the load — no schema change, no state-machine change, no
   deleted surface.
2. **Its stated purpose is confirmed elsewhere.** The recommendation's own justification is legal and
   reporting value. `insight-and-reporting`'s dashboards need dispositions grouped, and a free-text
   corpus cannot be grouped; `WF-005` already mandates the reason, and only its *shape* is at issue.
3. **`G-06` establishes the pattern.** Human override as a first-class record with a mandatory
   reason is adopted, and `RANK-006`/`BR-016` apply it to ranking and scorecard overrides. A coded
   disposition reason is the same idea applied to the transition rather than to the AI output.

**What is not claimed:** that the table's label means "adopted" in general. `candidate-intake` D1
made this point precisely and it is not repeated as though it were stronger than it is — three
entries from that table are now canonical in `domain-model.md`, which shows the label marks an absent
ratification step rather than a live objection *in general*, and shows nothing about any particular
row. This row's ratification is genuinely absent, and `TS-BL-056` builds against it anyway, for the
reasons above and behind the feature flag every surface ships behind. The proposal says so, this
decision says so, and the `interview/shortlisting` spec says so at the requirement that depends on it.

### D3 — Scorecard drift is decided: a post-approval note edit supersedes the scorecard and forces re-approval

**This is the open consequence `C-08` created and `exploration-notes.md` recorded as "Not yet
decided."** Its Open questions entry, as it stood before this decision — it now carries a pointer
back to this section:

> **⚠ Scorecard drift after note edit** — with `C-08` adopting versioning without a lock, a note can
> change after the consolidated scorecard drawn from it was approved. Recommended: wire
> note-version-after-approval to `scorecards.status = superseded`, requiring re-approval. **Not yet
> decided.**

**Decision: build it.** A note version created after a scorecard drawing on that note was approved
sets that scorecard's status to `superseded`, and the Application returns to a state requiring
re-approval. `TS-BL-059` emits the event, `TS-BL-060` records the note versions a draft consumed so
the link exists to follow, and `TS-BL-061` implements the transition.

**Why decide rather than scope out.** Four reasons, in descending weight:

1. **`C-08` removed the lock; it did not remove the reason for the lock.** `D24`'s explicit purpose
   was *"guaranteeing the approved evaluation and its source evidence agree."* `C-08` chose a more
   flexible edit model and named the resulting exposure in the same breath. Adopting the flexibility
   and discarding the guarantee would take the half of the trade that costs nothing and drop the
   half that pays for it.
2. **The mechanism already exists in the schema.** §12.2's `scorecards.status` enum is
   `Draft, approved, rejected, superseded`. `superseded` has no writer anywhere in the backlog and no
   other plausible one — `C-08` says the spec *"provides the tool but does not wire it up."* This is
   wiring, not invention, and an enum value with no writer is the kind of dead-but-plausible surface
   that reads as implemented.
3. **`SCR-008` requires scorecard changes to be versioned and auditable.** An approved scorecard
   whose evidence silently moved underneath it is a change to what that scorecard *means* with no
   version and no audit record. Doing nothing does not leave `SCR-008` merely unhelped; it leaves it
   quietly false.
4. **The failure is invisible by construction.** The drift produces no error, no flag and no changed
   field — an approved evaluation simply stops matching its evidence while continuing to read as
   approved. That is `platform-core` D5's silent-success class, the one Sprint 0 paid three days of
   empty deploys to learn to distrust. The gates that caught real problems there were the ones
   comparing **two sources of truth**; a scorecard's recorded note versions against the notes'
   current versions is exactly such a comparison.

**Why supersede rather than the two alternatives:**

- *Block the edit instead* — i.e. reintroduce the lock for approved-scorecard notes. Rejected: that
  is `D24`, and `C-08` reversed it on the reference spec's authority (`INT-008`/`BR-017`: edits
  create versions, forever, no freeze). Re-deriving a lock as an implementation detail would
  contradict a resolved conflict without going through the conflict-resolution process.
- *Flag the drift and leave the approval standing* — a warning on the scorecard saying its evidence
  has moved. Rejected: an approval is a governance act, and `project.md`'s central rule makes the
  approver accountable for what they approved. A warning changes what a reader knows and leaves the
  accountable act attached to evidence its approver never saw. `SCR-006`'s Draft-until-approved gate
  exists precisely so that "approved" means a named human read *this* content.

**The cost, stated rather than hidden:** an interviewer correcting a typo in a note after approval
forces a Practice Manager to re-approve. That is real friction and it is accepted, with one
proportionality measure: supersession follows the **note version link recorded on the draft**, so
only scorecards that actually drew on the edited note are superseded, and re-approval presents a diff
of what changed rather than a blank re-read. What is *not* built is a materiality judgement — no
attempt to decide that a given edit was too small to matter. A rule that decides which evidence
changes are unimportant is a rule that can be wrong silently, which is the failure this decision
exists to prevent.

**What re-approval preserves:** the prior approved version is retained, not overwritten — `SCR-008`
requires versioning and `RANK-006`'s sibling principle (an override never deletes original output)
applies by analogy. The audit trail shows: approved at T1 against note versions {v2, v1}, superseded
at T2 by note v3, re-approved at T3 against {v3, v1}.

### D4 — Consolidation surfaces panelist disagreement; it never resolves it arithmetically

`D11`'s consequences left an open item that `C-05` did not close:

> **Open:** the rule for consolidating conflicting panelist scorecards into one selection decision —
> currently "the PM decides," which may be sufficient.

`C-05` reversed the structure around it, so the question now reads: when three panelists submit
`strong_yes`, `no` and `maybe`, what does the **one** consolidated scorecard's
`overall_recommendation` say?

**Decision: the AI draft proposes an overall recommendation and must present every panelist's own
recommendation beside it, individually attributed; disagreement is rendered as disagreement; the
approver decides.** No mean, no median, no majority rule, no weighting.

*Why this is the whole point rather than a display preference:* `C-05`'s reversal turned on exactly
this. `D11` rejected consolidation because it was framed as averaging — *"no averaged score, so no
fake math papering over panelist disagreement"* — and `C-05` reversed it on the grounds that
`SCR-004`/`SCR-005` *"explicitly are not"* averaging. If consolidation resolved conflicts by
arithmetic, `D11`'s original objection would be correct and `C-05` would have been reversed on a
false premise.

*Why the draft still proposes an overall value:* `SCR-003` fixes `overall_recommendation` as a
dimension and §12.2 carries the column, so the field exists and something must fill it. Filling it
with an AI-proposed value that is **marked AI-generated until approved** (`AI-001`, `UI-004`) and is
adjustable only with a mandatory reason (`SCR-007`, `BR-016`) is the shape every other AI output in
this product already takes. Leaving it null until a human types one would make the scorecard less
useful without making it more honest.

*What the spec asserts, so this cannot decay into a display choice:* a consolidated scorecard whose
source notes disagree must carry each note's recommendation with its author and round, and the
suite fails if a code path derives the overall value by arithmetic over panelist recommendations.
`C-05`'s stated benefit — *"someone must weigh three panelists at selection, and it is better as an
auditable artifact than as private reasoning"* — is only obtained if the weighing is visible.

*`D11`'s "the PM decides" is retained and made concrete:* the approver is the accountable human, and
their divergence from the AI-proposed overall value is recorded through
`ai-platform-governance`'s existing **override record** with its mandatory reason — the same
mechanism `matching-and-ranking`'s board uses, and the source of the *"how often approved scorecards
diverge from AI drafts"* half of `insight-and-reporting`'s AI-agreement-rate dashboard.

### D5 — The presentation shell gets its consumer: the Interview Console during an active interview

`design-system` built the presentation shell (chrome removed, content full-viewport), recorded that
it *"ships with no consumer,"* and left the assignment open, noting it is *"answerable by
`interview-pipeline` without changing anything here."* `S.3` names the two candidates: the Interview
Console during an active interview, or a full-screen document viewer.

**Decision: `TS-BL-058` uses it, for the active-interview state only.** The console has two states —
preparing or reviewing (authenticated shell, sidebar, navigation) and **conducting** (presentation
shell). The full-screen document viewer is not built and remains unassigned.

*Why the console is the right consumer:* an interview is the one activity in TalentSphere performed
with a person present and attention elsewhere. Navigation chrome during it is not merely unused; it
is a live risk, because the sidebar is a path to the ranking board and other candidates — precisely
what `C-10` and `D05`'s per-candidate rule withhold from an interviewing actor. Removing the chrome
makes the withholding physical as well as permission-enforced. That said, the guarantee still rests
on `TS-BL-054`'s server-side projection and the evaluator: **the shell is defence in depth, never the
mechanism**, and the spec says so, because a guarantee that depended on a chrome state would be a
display filter of exactly the kind `config.yaml` rules out.

*Why not the document viewer:* nothing in the eighty-item backlog builds one. Assigning the shell to
a screen that does not exist would leave it exactly as it is today.

*What this does not do:* it changes nothing in `design-system` and needs no `/opsx:update` against
it. The state was built, is selectable, and now has a caller — which is the outcome that feature's
Open Question anticipated.

### D6 — Two vector source types are registered here; no search surface is built

`matching-and-ranking` D2 embedded two of `VEC-001`'s four source types and built the pipeline
source-type-driven, with all four values in the `source_type` enum **from the first migration**,
specifically so the remaining two are *"registrations rather than modifications."* It named the
owners: `interview_note_section` with `TS-BL-058` (whose `INT-007` already lists embeddings among a
note's stored forms) and `scorecard_summary` with `TS-BL-062`. Its Open Questions asks whether that
seam holds.

**Decision: register both, exercise the seam, and report what it cost.** `TS-BL-058` registers
`interview_note_section`; `TS-BL-062` registers `scorecard_summary`. If either registration requires
a change to `matching-and-ranking`'s pipeline, that is a real finding and belongs in an
`/opsx:update` against that feature — not a local workaround here, and not a second embedding path.

*One thing the registration must get right:* `INT-007` requires notes to carry embeddings, and note
notes are **versioned** (`C-08`). §12.2's vector record carries `source_version`, so a note section's
embedding is pinned to a note version, and a new note version produces new records rather than
mutating existing ones — the same replace-not-duplicate rule `TS-BL-049`'s re-index path already
implements. Without this, the drift D3 prevents in the scorecard would reappear in the index.

**No search surface.** `C-01`'s scope note records semantic search over interview history as *"a
genuine capability that is **not** among the original 19 features"* and asks whether it is in scope.
It remains out of scope, and remains open. This feature makes it reachable and builds none of it.

### D7 — `D23a`'s which-application-survives rule: answered as far as this feature's stages reach, and no further

**Handed here twice, by name.** `candidate-intake` D1(b): *"an application's stage belongs to
`interview-pipeline`."* `matching-and-ranking` D5: *"the other half is which application survives, and
that depends on the application stage, which is `interview-pipeline`'s. Do not invent it."* Two
features declined to invent it because the stage model did not exist. It partly exists now.

**What is answerable here, and is answered:** for two applications on one posting whose stages both
fall **within this feature's range** (`Ranked` through `ScorecardReady`, plus `NotSelected`), the
survivor is **the one further along §11.2's lifecycle**; where both sit at the same stage, the
**earlier-created** application survives. The other application is not deleted — it is transitioned
to a terminal state with a reason naming the merge, per `WF-005` and `BR-020`'s history-preservation
rule, and its interview rounds, notes and scorecards remain readable and attached to it. Notes are
never re-parented: `INT-007` binds a note to its round and its author, and moving one would forge
attribution.

**Why "further along" rather than any other rule:** the alternatives lose information that cannot be
reconstructed. Keeping the earlier-stage application discards completed interviews and an approved
scorecard; merging the two records' contents produces one application with two rounds one panel
never conducted. Progress is the only ordering that is total over §11.2's linear spine and
monotone — a candidate never moves backwards through it.

**What is not answered, and why it is not evasion:** §11.2 continues past `ScorecardReady` into
`PrioritySelected`, `OfferInProgress`, `OfferAccepted`, `OnboardingComplete` and `Recruited`, and
`SelectedNotOffered`. Those stages, their transitions and their consequences belong to
`decision-and-offers` (`TS-BL-064`–`TS-BL-069`), unproposed. A merge where one application holds a
live offer raises questions this feature cannot see: whether an offer can be re-pointed, what
`BR-012`'s five-slot arithmetic does when a slot holder merges, whether `SEL-003`'s duplicate-active-
selection block already prevents the case. Inventing an answer would repeat the mistake
`candidate-intake` and `matching-and-ranking` each declined to make.

**The invariant this feature adds, which pre-empts nothing:** a merge involving an application at or
beyond `PrioritySelected` is **refused and surfaced for human resolution** rather than resolved
automatically. That is stricter than `candidate-intake`'s invariant (record and surface, take no
further action) in exactly the region where the rules do not exist, and it is consistent with it
everywhere else. `decision-and-offers` can replace the refusal with a rule; it cannot un-merge a
merge this feature performed wrongly.

### D8 — What "no lock" actually requires, and who may edit

`C-08` is three separate reversals of `D24` and each has a concrete implementation consequence that
is easy to satisfy on one and miss on the others:

| `D24` (superseded) | `C-08` (build this) |
|---|---|
| Only the authoring panelist may edit | **Permission-matrix rule** — the `Edit` action on the Interview Console page, per role |
| Notes freeze on scorecard approval | **No freeze, ever.** Editing is available for the life of the record |
| Post-freeze corrections are appended addenda | **Edits are edits**, each producing a new version (`INT-008`, `BR-017`) |

**Editability is unbounded in time and is not a state.** There is no `locked` flag, no
approval-derived condition on the edit path, and no "editable until" timestamp. The spec asserts the
absence: a note whose Application has reached a terminal state and whose scorecard is approved is
still editable by a permitted actor. What such an edit *causes* is D3's supersession — a
consequence, not a prohibition.

**Who may edit is a matrix question with a seeded answer.** `C-08`: *"the `Edit` action flag on the
Interview Console page decides it, per role. Recommended seeding: grant `Edit` on notes to
Interviewer only."* That seeding is implemented as a **seeded matrix value** in
`access-control-and-admin`'s `TS-BL-022`-managed configuration, not as a role comparison in code —
the same rule `matching-and-ranking` D7 applied to its projection, and for the same reason: `C-02`'s
nine roles and `ADM-005`'s grant/deny/unset cells make the matrix the decision surface, and a
hard-coded author check would silently ignore an override. `access-control-and-admin`'s proposal
already names *"`interview-pipeline`'s note-edit matrix control"* as a downstream consumer of
`TS-BL-018`.

*The consequence worth stating because it surprises people:* under `C-08`'s model a Practice Manager
**can** be granted edit rights on another person's interview note. `D24` forbade this by
construction; `C-08` moved it to a configuration decision, and the seeded value withholds it. The
protections that remain are the ones that matter: every version is retained with its author, and
`INT-007` binds each version to the identity that wrote it, so an edit by a non-author is visible as
such rather than absorbed into the original.

**Attribution survives editing.** A note's **version** records who wrote that version; the note
records who **submitted** it (§12.2's `submitted_by`). Both are retained on every version, which is
what makes a non-author edit auditable rather than merely permitted.

### D9 — Two version chains, one supersession rule, and the join between them

This feature carries two independently versioned records — `interview_notes.version_number` and
`scorecards.version_number` — and D3's mechanism is the join between them. Getting the join wrong in
either direction produces a plausible-looking system:

- **A scorecard records the note versions it was generated from**, as a set, written by `TS-BL-060`
  at generation time. This is `D24`'s one consequence that survives its own reversal intact: *"the
  approved scorecard must record which note version it was drafted from."* `C-08` makes it more
  necessary, not less — under `D24` the lock made the record a formality after approval; under
  `C-08` it is the only thing that makes drift detectable.
- **Supersession is scoped by that set.** A new version of note N supersedes exactly those approved
  scorecards whose recorded set names an earlier version of N. A note belonging to a different
  Application, or to a round no scorecard consumed, supersedes nothing.
- **A superseded scorecard is not a new scorecard version.** Status moves `approved → superseded`;
  regeneration or re-approval creates the next version. Otherwise every note typo inflates the
  version chain with rows nobody authored.

*The ordering hazard, named:* `TS-BL-060` generates from notes that are still editable, so a panelist
can edit between draft generation and approval. `D24` identified this before `C-08` existed and it is
unchanged by the reversal. The rule: **approval re-checks the recorded note versions against current
versions, and refuses to approve a draft whose evidence has already moved**, directing the approver
to regenerate. Superseding a scorecard one second after approving it would be technically consistent
and would read as a system malfunction.

### D10 — The Application state machine is registered here, and its declaration stops at `ScorecardReady`

`platform-core`'s `platform/workflow-engine` ships the framework with *"no posting, Application,
offer, or closure state machine defined"* and states that registering one *"is how a feature such as
`hiring-postings` or `interview-pipeline` obtains transition behavior."* `TS-BL-056` registers the
Application machine — the first domain machine in the product besides the posting's.

**What is declared:** §11.2's states this feature reaches — `Ranked`, `Shortlisted`,
`InterviewScheduled`, `Interviewed`, `ScorecardReady` — their permitted transitions, the permission
each transition requires, and `WF-005`'s reason-required marking on `NotSelected` (reachable from
four states) and `Withdrawn`.

**What is declared as reachable but not owned:** `PrioritySelected` onward. The states are named in
the machine so `WF-002`'s rejection can explain *"the transitions available from here,"* and their
transitions are registered by `decision-and-offers`. A machine that ended at `ScorecardReady` with no
forward edge would make a correct application look terminal.

*Why registration lands on `TS-BL-056` rather than being split across the items that use each
transition:* a state machine is one declaration; four items each appending states would mean four
migrations of one registry and no single place where §11.2 is checkable against the code. `TS-BL-056`
declares it whole for this feature's range; `TS-BL-057`, `TS-BL-058` and `TS-BL-062` *invoke*
transitions rather than declaring them. This is recorded as an item-boundary refinement in D12.

*What this feature must not do, asserted rather than assumed:* no surface writes `status` directly.
The framework's own spec fails the suite on a direct state write, and this feature is the first with
a real reason to be tempted — a shortlist button on a board row is one line of code away from being
an `UPDATE`.

### D10a — Three of §11.1's posting advances are this feature's, and were declared with no caller

**Found by `decision-and-offers`' verification and disclosed in its `design.md` D9a**, not by this
conversation. `hiring-postings`' `TS-BL-038` registers §11.1's posting machine **whole** — all
thirteen states, with cancellation available from `screening`, `interviewing`, `scorecard review`,
`selection`, `offer` and `onboarding`, which only means something if a posting can be in them. The
declaration is complete. **What was missing is a caller for the six advances between `open` and
`onboarding`**, so `job_postings.status` reached `open` and stopped, and `TS-BL-069`'s fulfilment
closure guarded a state no posting arrived at.

`decision-and-offers` built the bottom three, whose triggers are its own writes. **The top three are
this feature's**, by the rule D9a applies — the item that owns the triggering event registers the
transition that event causes:

| Advance | Triggering event | Item |
|---|---|---|
| `open → screening` | first candidate shortlisted on the posting | `TS-BL-056` |
| `screening → interviewing` | first interview round scheduled | `TS-BL-057` |
| `interviewing → scorecard_review` | first scorecard approved | `TS-BL-061` |

**Two properties, both easy to lose:**

- **Monotone and idempotent.** An advance fires only where the posting is in the immediately
  preceding state; otherwise it is a no-op. A second shortlist on an already-`screening` posting
  neither re-fires nor errors, and no advance ever moves a posting backwards.
- **Attributed to the human whose action triggered it, inside that action's own transaction.** The
  recruiter who shortlisted is the actor on `open → screening`. This is what keeps
  `hiring-postings`' *"no transition fires without an authenticated actor"* assertion true, and it
  is why these are advances-within-an-action rather than a scheduled sweep over postings.

**The third property D9a names — recompute once on reopen — is deliberately not built here.** It
belongs to `decision-and-offers`' own three advances and is built there. A reopen is a
`filled → open` transition this feature never performs and cannot observe; adopting it would mean
polling postings for a state change owned by another feature, which is the same duplication D9a
rejected when it declined to build these three itself.

**Why this feature rather than `hiring-postings`:** D9a considered flagging all six to
`hiring-postings` and rejected it — that feature can observe neither shortlists nor scorecards, so
it would need dependency edges onto Phase-3 items, inverting `D.10`'s direction.

*Why no new `depends_on` edge, stated because the opposite is arguable.* These three **invoke** a
transition `TS-BL-038` already declares; they do not register one. `decision-and-offers` drew
exactly this line — it added a `TS-BL-038` edge to `TS-BL-069`, which *registers* the closure
transition, and none to `TS-BL-064`/`TS-BL-067`, which only invoke advances. `TS-BL-038` sits on no
transitive path to `TS-BL-056` (`TS-BL-037` does; `TS-BL-038` depends on it), so this is the
standing build-against-the-declared-shape case, recorded in `tasks.md`'s interface list rather than
as an edge. The residual: an advance invoked before `TS-BL-038` lands fails at runtime rather than
merely lacking an interface — which is true of `decision-and-offers`' three as well, and is why both
features record it explicitly instead of relying on ordering.

**This closes `exploration-notes.md`'s pending-obligation row for this feature and unblocks
`decision-and-offers` task 6.16**, an end-to-end `open → filled` test deliberately written to fail
until these three land. That test is that feature's artifact and is not touched here.

### D11 — Evidence separation and per-dimension confidence are output-contract clauses, not display choices

`SCR-004` (resume-derived evidence separated from interview-derived) and `SCR-005` (every dimension
carries evidence source and confidence) read like presentation requirements. They are enforced as
**contract clauses on the `scorecard_generation` family**: output where a dimension lacks a source
label from `TS-BL-031`'s closed vocabulary, or lacks a confidence value, or merges the two evidence
bodies, is rejected as a contract violation and recorded as a failed run.

*Why:* `C-05`'s reversal rests on `SCR-004`/`SCR-005` being *"explicitly not"* averaging, and
`G-02`'s adopted labelling exists so an Admin's audit access does not become a personal-data back
door. Both properties are properties of the stored artifact. A separation applied at render time
leaves one merged evidence body in the database, which every later reader — the carry-forward path,
the resurfacing engine, the audit surface — sees unseparated. It is the same display-filter argument
`config.yaml` makes about redaction, applied to structure rather than to content.

*The corollary:* this feature derives **no** source value of its own. `ai-platform-governance`'s
`TS-BL-031` owns the closed vocabulary precisely so exactly one component defines it, and
`design-system`'s label component *"renders the source value it is given without interpreting or
deriving it."* `scorecard` and `interview_note` are values in that vocabulary already
(§12.2's `source_type` enum on evidence records lists `Resume, interview_note, scorecard,
human_decision, system`), so nothing new is needed — which is worth checking rather than assuming,
because this is the first feature to produce output labelled `interview_note`.

### D12 — Item-boundary refinements, recorded rather than assumed

Under `D.11`'s permission for a feature's propose conversation to refine its internal item
boundaries. None moves work between features; two add a dependency edge, and D1 explains why.

- **`TS-BL-058` authors `interview_questions` and therefore depends on `TS-BL-027`.** D1. `D.10`'s
  title says "Interview Console (structured notes, multiple rounds)" and carries no AI edge.
- **`TS-BL-059` authors `interview_note_summary` and therefore depends on `TS-BL-027`.** D1.
- **`TS-BL-056` registers the Application state machine** and creates the `status` /
  `shortlist_reason` writes. `D.10`'s title names only the decision UI, but a disposition without a
  machine to transition is a direct state write, which the framework's spec fails the suite on (D10).
- **`TS-BL-057` publishes the round-created event and registers no subscriber.** The seam
  `TS-BL-058` consumes, mirroring `hiring-postings`' `posting.opened` (D1).
- **`TS-BL-060` records the note-version set the draft consumed.** `D.10`'s title says "AI-drafted
  generation"; the recorded set is what makes `TS-BL-061`'s supersession scopable at all (D3, D9).
- **`TS-BL-061` carries supersede-and-re-approve** as well as the approval gate. `D.10`'s title says
  "human-approval workflow"; D3 explains why the drift mechanism belongs with the status it writes
  rather than as a ninth item.
- **`TS-BL-058` and `TS-BL-062` each register one vector source type.** D6; `matching-and-ranking`
  D2 assigned these in advance.
- **`TS-BL-063` seeds dimensions into audited runtime configuration** rather than into a constant.
  `D.10`'s title says "fixed dimensions/competencies," and `C-06`'s `OD-007` carry-forward requires
  they be changeable without a deployment — through a mechanism that had to be corrected first (D14).

### D13 — What Sprint 0 built and decided about interviews and scorecards: nothing, and no decision was allocated here

The same reconciliation check every proposed feature has run, and the same result. Checked, not
assumed.

**Sprint 0 built nothing relevant.** `sprint-0-outcome.md` records **13 of 196** Wave-1 tasks, all
platform-layer, with *"no hiring feature exist[ing] yet, by design."* The backend's directories are
`core/`, `api/`, `db/`, `audit/`, `features/`, `runtime_config/`, `middleware/` and `migrations/` —
no interview module, no `interview_rounds`, `interview_notes` or `scorecards` table, and no domain
state machine of any kind (which is the point `platform-core`'s workflow-engine spec asserts as a
scenario).

**No `D1`–`D18` decision from `talentsphere-wave-1-foundation`'s `design.md` was allocated here.**
`platform-core` D11 records the full split — D1–D7 to `access-control-and-admin`, D8 to
`identity-and-access`, D9–D12 to `ai-platform-governance`, D13–D14 and D17 to `design-system`,
D15/D16/D18 to `platform-core` — accounting for all eighteen with none left over, consistent with
`ai-platform-governance`'s statement that the change has *"no unaccounted content left."* Confirmed
rather than assumed, because the proposal's "nothing was inherited" claim depends on it.

**No `interview/` delta spec exists** in `talentsphere-wave-1-foundation`. Its fourteen cover
`access-control/`, `ai-platform/`, `design-system/`, `identity/` and `platform/` only. So, as with
the three Phase-2 features, there is nothing to redistribute and no interim double-description to
accept.

**`KNOWN_ISSUES.md` carries no entry about shortlisting, interviews, notes or scorecards.** Recorded
because an empty result can otherwise read as a clean bill of health rather than as an absence of
subject matter. **Three** of its entries bear on this feature without being about it: only Local and
Dev are provisioned, so every surface here is exercised at Dev scale; the Hubble login contract is
unconfirmed, which affects the identities this feature attributes notes and approvals to; and
Sprint 0's endpoints carry no declarative permission requirement, which is
`access-control-and-admin`'s task 8.2 to close and is mentioned here only so its absence is not
re-diagnosed as an interview-surface gap.

**A fourth constraint belongs beside them but is not in that file, and this design previously said it
was.** All three of this feature's prompt families run against the deterministic stub because the
model provider is undecided — which is `reference/spec.md` §35's **`OD-003`**, reasoned in
`ai-platform-governance`'s `design.md` D1, **not** a `KNOWN_ISSUES.md` entry: that file carries no
mention of a provider, `OD-003`, or a stub. The entry is owed but **unwritten** — it is
`ai-platform-governance`'s task 1.18, *"Record the unresolved provider decision (`OD-003`) in
`KNOWN_ISSUES.md`, in the same form the unconfirmed Hubble contract (`OD-001`) already takes,"* which
lands with `TS-BL-027` and is not built. Found during verification of this change; the constraint is
unchanged and still applies, only its citation was wrong. **The same misattribution existed in
`matching-and-ranking`'s `design.md` D13**, which listed the unconfirmed AI provider as one of two
`KNOWN_ISSUES.md` entries. It was flagged from here rather than edited, because correcting another
change's artifacts is `/opsx:update`'s job against that change — and it **has since been corrected
there** by that change's own run: its D13 now records a single qualifying entry and carries the same
`OD-003` citation this paragraph does.

**One `sprint-0-outcome.md` carry-forward touches this feature directly.** The `INSERT`/`SELECT`-only
grant on `audit_logs` — verified against local PostgreSQL, unexercised on Cloud SQL until task 2.3 —
is what makes this feature's disposition, approval and override records trustworthy. Every governance
claim `TS-BL-056` and `TS-BL-061` make rests on that narrowing actually holding on the real instance.

### D14 — Two shared-document gaps found and fixed in `exploration-notes.md`

Both corrected at the point of the error during this conversation, per `AGENTS.md`'s convention for
genuine gaps in the four documents every feature reads, with the prior wording quoted rather than
replaced silently — the treatment `D09`'s background-jobs row, `S.6`, `domain-model.md`'s mis-cited
AI-governance reasoning and `D05`'s re-asserted reversal each received.

**(a) `D11`'s note-locking consequence was reversed and never annotated.** Its second consequence
reads *"Interview notes must **lock on scorecard approval**, or the record is worthless."* `C-08`
reversed the mechanism on 2026-08-14 — but `C-08` is written as an amendment to **`D24`**, where the
mechanism was *chosen*, and never touched `D11`, where the *requirement for a mechanism* was stated.
`D11`'s Decision-index `Ref` column reads `✔ C-05`, and `C-05` genuinely resolves `D11` — but only
its **structure** half (per-round → consolidated). `C-05` is silent on locking. So a reader arriving
at `D11` through the index sees a decision marked resolved and finds a live-reading locking
requirement carrying no supersession marker anywhere.

*And this is the answer to the question about `D24`'s own stale line.* `D24`'s consequence — *"Satisfies
the `D11` requirement that notes lock on scorecard approval"* — is equally stale, and needs **no
fix**: `D24` is a *wholly superseded* decision whose body this document deliberately preserves rather
than rewrites (the same convention that keeps `D05`'s body reading "full AI context upfront" after
`C-10` reversed it), and its index row already reads *"~~Lock on approval~~ → version forever,
matrix-controlled edit."* `D11`'s is the half that reads as current, so `D11` is where the dated block
was added — with `D24`'s mirror-image staleness noted inside it so a reader who arrives from that
direction is not left thinking one of the two was missed.

*Why this mattered enough to fix rather than route around:* `TS-BL-059` is the item that has to build
one of the two readings, and `TS-BL-061`'s entire existence follows from which one. Picking silently
would leave `decision-and-offers` — which carries this feature's scorecards forward under
`D17`/`D21` — to pick again from the same contradiction.

**(b) `C-06`'s "configurable per §29 #14" cites the wrong item.** Its `OD-007` carry-forward reads
*"Treat `SCR-003` as a seeded default, configurable per §29 #14, not as settled product truth."*
**§29 item 14 is "Role and permission matrix values."** Counted against all fifteen items: §29 has
**no** scorecard-dimension entry at all, and `SCR-003` states the dimensions as a fixed list with no
configuration hook. The *conclusion* stands — `OD-007` genuinely leaves them unapproved, so a seeded
default is right — but the mechanism was borrowed from §29's pattern rather than cited from its
contents. The correct vehicle is `platform-core`'s audited runtime-configuration registry, extended
with a scorecard-dimension key; building against #14 as written would put scorecard dimensions inside
the permission matrix.

*Why it was missed:* `C-06` was one of twelve conflict resolutions written in a single pass on
2026-08-14, and this is its only clause citing a numbered §29 item rather than a requirement ID — the
surrounding `SCR-003`/`INT-006`/`OD-007` citations are all correct, so the row reads as verified by
association. Nothing had reason to open §29 and count until a feature owning scorecard dimensions was
proposed.

*Consequence for this feature:* `TS-BL-063` seeds `SCR-003`'s eight dimensions and the 5-point scale
into the runtime-configuration registry — declared, closed, every change audited with a mandatory
reason — and asserts no unaudited setter exists. Which is what `C-06` meant, and is now what it says.

## Risks / Trade-offs

- **[This is the first feature where a human decision about a person is recorded, and the reason
  vocabulary that makes those decisions reportable is unratified.]** D2's coded disposition list has
  no owner sign-off → Mitigated by the list living in audited runtime configuration, so rejection
  costs a configuration change rather than a revert, and by `WF-005` independently requiring the
  reason regardless of its shape. Residual: a poorly chosen initial vocabulary shapes what
  `insight-and-reporting` can report on before anyone notices, because disposition codes accumulate
  historically and cannot be re-coded retroactively without rewriting decisions people made.
- **[Supersede-and-re-approve adds friction that users will route around.]** D3 forces re-approval on
  any post-approval note edit that fed the scorecard → Partly mitigated by scoping supersession to
  the recorded note-version set and by presenting a diff at re-approval. Not mitigated: the
  predictable workaround is that panelists stop correcting notes, which trades a visible governance
  cost for an invisible accuracy cost. Worth instrumenting — the rate of superseded scorecards is
  the signal, and if it is near zero that is evidence of avoidance, not of quality.
- **[No materiality judgement means a typo supersedes an approval.]** D3 deliberately declines to
  decide which edits are too small to matter → Accepted as the lesser harm; a rule that classifies
  evidence changes as unimportant is a rule that is wrong silently, which is the exact failure the
  mechanism exists to prevent. If the friction proves intolerable, the right answer is a **human**
  declaring an edit immaterial with a reason and an audit record, not a heuristic.
- **[`C-08` permits a non-author to edit an interview note, and the only protection is a seeded
  matrix value.]** D8 → Mitigated by every version retaining its own author identity, so a non-author
  edit is visible as one rather than absorbed; by `AUTHZ-005`'s direct denial; and by the seeded value
  withholding the grant. Not mitigated: this is genuinely weaker than `D24`'s construction-level
  guarantee, and it is what the reference spec's model costs. Recorded as an accepted consequence of
  a resolved conflict rather than as a defect.
- **[Two prompt families were unowned until this conversation, which suggests the count was never
  checked end to end.]** D1 → Mitigated for these two by claiming them and by the accounting table,
  which makes the ten-family closure checkable rather than asserted. Residual: the same class of
  omission could exist in another enumeration nobody has counted against its owners — §25's eleven
  job types and §13.2's endpoint list are the two most likely candidates, and neither has been
  audited feature-by-feature.
- **[Three families multiply this feature's corpus obligation, and it is the first that can construct
  `AI-015`'s "conflicting interview notes" class for real.]** → Mitigated by the promotion gate
  refusing production activation without passing cases, so the obligation cannot be quietly skipped.
  Not mitigated: Dev runs all three before the cases exist, which `ai-platform-governance` D8 argues
  is the correct order and is still a window where a bad prompt is reachable in Dev.
- **[The consolidated scorecard is the artifact `decision-and-offers` carries forward, and it is
  designed here against a carry-forward path that does not exist.]** `D17`/`D21`/`C-07` describe
  linking one scorecard to multiple Applications → Mitigated by `C-05` having chosen consolidation
  partly *for* carry-forward simplicity, so the shapes were reasoned together, and by this feature
  building the one-per-Application constraint rather than a multi-link path a later feature would have
  to unpick. Residual: if `decision-and-offers` finds the constraint wrong, the fix is an
  `/opsx:update` against this change plus a migration.
- **[Registering two vector source types exercises another feature's seam that has never carried a
  second registrant.]** D6 → Mitigated by `matching-and-ranking` D2 having built the enum with all
  four values from the first migration and by having recorded the question as open, so a failure is
  an expected finding rather than a surprise. Not mitigated: if the seam does not hold, this feature
  is blocked on an `/opsx:update` against a feature that may itself be unbuilt.
- **[Half of `D23a`'s merge rule is now answered and the other half is not, so the rule is in two
  places.]** D7 → Mitigated by the unanswered region being **refused** rather than left undefined, so
  the boundary is enforced rather than documented. Residual: a reader looking for "the merge rule"
  finds a third partial answer here after `candidate-intake`'s and `matching-and-ranking`'s, and
  nobody has written the whole one.
- **[The presentation shell removes navigation that a real interviewer may need mid-interview.]** D5 →
  Mitigated by the state being exit-able and by the console's own content carrying everything
  `INT-004`/`INT-005` require. Not mitigated by design: reaching another candidate mid-interview is
  what the state exists to prevent, and if that proves wrong it is a product finding, not a bug.
- **[The whole feature is the first real consumer of the workflow framework, so framework defects
  surface here as interview defects.]** D10 → Mitigated by `platform-core`'s spec carrying the
  scenarios (invalid transition explained, reason enforced, concurrent transition safety) as testable
  requirements rather than as intentions. Residual: the framework has never run against a domain
  machine, and `sprint-0-outcome.md`'s lesson is that a gate which has never fired is not a gate.

## Migration Plan

There is no data migration; nothing here has a predecessor in production. Sequence:

1. **`TS-BL-056` first, and it is blocked.** Its dependencies are `matching-and-ranking`'s
   `TS-BL-053` and `platform-core`'s `TS-BL-004`, neither started. Where staffing forces an early
   start, the state machine declaration and the disposition rules are buildable against
   `platform/workflow-engine`'s finished spec as an interface — the stub-and-integrate shape
   `platform-core` D8 used for the audit port.
2. **Register the state machine before the first transition path.** D10: one declaration, whole for
   this feature's range, with `decision-and-offers`' states named as reachable. Writing a disposition
   endpoint first and registering afterwards means writing a direct state write and removing it.
3. **`TS-BL-057` next**, and publish the round-created event even though nothing subscribes to it
   yet. `TS-BL-058` registers the subscriber (D1); publishing later would mean editing the scheduling
   path to add an event once its consumer exists.
4. **`TS-BL-058` before `TS-BL-059`**, which `D.10`'s edge already requires. Build the console and
   the note record — including `version_number` and `is_current_version` from first submission, the
   same refinement `candidate-intake` made for resume versions — then `TS-BL-059` adds
   edit-creates-a-version, the matrix-controlled edit path and the note-version event.
5. **`TS-BL-060` and `TS-BL-059` are mutually independent** — both depend on `TS-BL-058` and neither
   on the other. Take **`TS-BL-059` first** where the order is free: `TS-BL-060` must record the note
   versions a draft consumed (D9), and versions it can point at are cheaper to record than a version
   scheme retrofitted around an existing draft.
6. **`TS-BL-061` after `TS-BL-060`**, and build the approval gate and the supersede path **together**,
   not the gate first. D3's mechanism reads the note-version set `TS-BL-060` records; an approval gate
   shipped without it is an approval that cannot be invalidated, which is the state `C-08` created and
   this feature exists to close. Same reasoning `matching-and-ranking` used for shipping the re-index
   path with the pipeline.
7. **`TS-BL-062`, then `TS-BL-063`.** Both are `D.10` edges. Consolidation defines the document the
   dimensions structure; seeding dimensions against a document that does not exist yet means seeding
   into nothing.
8. **Author each family as a new template version** of its already-registered family, never by editing
   an existing version, and add its cases to `TS-BL-032`'s corpus — including `AI-015`'s conflicting-
   interview-notes class, which this feature is the first able to construct. All three are
   Dev-deployable and production-blocked until those cases pass (`ai-platform-governance` D8).
9. **Feature-flag every surface, disabled by default.** Shortlist View, Interview Console and
   Scorecard Center each ship behind a declared flag; a disabled capability answers 404, so there is
   no window in which a half-built decision surface is reachable.
10. **Record the open items in `KNOWN_ISSUES.md`** as they land: that all three families run on the
    stub provider in Dev, that `SCR-003`'s dimensions are seeded from an `OD-007`-unapproved set, and
    that `innovation practice relevance` is Miracle-Labs-specific pending confirmation — the same
    treatment the unconfirmed Hubble contract already receives.
11. **Rollback** is redeploy-previous-artifact plus migration-down. This feature creates three tables
    and writes three columns another feature created; those columns are nullable, so reverting leaves
    them null — the documented not-yet-dispositioned state rather than a corrupt one. The state
    machine registration is declarative and unregisters with the deployment.

## Open Questions

Each is genuinely deferrable — none changes the specs, the approach, or the task breakdown.

- **Whether `innovation practice relevance` applies here or should be renamed.** `C-06` asks the
  question and `OD-007` owns the answer. It ships seeded as specified, because renaming a value in
  audited configuration is a configuration change and guessing at a replacement is not.
- **Per-family conciseness bounds for the three families** — the actual sentence and item counts.
  `S.5` records that the numbers were never given and *"shouldn't be invented here"*;
  `ai-platform-governance` D7 settled the form (provisional, labelled, enforced, changeable through
  the audited path). This feature supplies three families' worth of provisional values under that
  rule. The interview summary is the one most at risk of drifting long, since its input is prose.
- **Interview types beyond §12.2's enum** (`technical, managerial, culture, panel, final, other`).
  Seeded as specified; `INT-006`'s note fields are fixed across all types per `C-06`, so a new type
  needs no new note template — which is what makes this deferrable rather than structural.
- **Aging thresholds for interviews and feedback** (§29 item 9). The configuration keys are seeded;
  what a stalled interview or an overdue note *does* is `insight-and-reporting`'s SLA dashboard
  (`TS-BL-071`–`TS-BL-073`), not a transition here. No threshold value is invented.
- **Whether a scheduled round should be re-generatable for its question set** when the interview type
  or the underlying gaps change before the interview happens. The set is pinned to the round on
  creation; regeneration is an authorized action or it is not, and no requirement decides. Deferred
  because either answer is a permission-gated endpoint against a shape that already exists.
- **Whether the supersede signal should notify the approver** rather than only marking the scorecard.
  `platform-core`'s notification engine makes it cheap, and `D04` keeps internal delivery in early
  scope. Deferred because the state is correct either way and the notification is additive.

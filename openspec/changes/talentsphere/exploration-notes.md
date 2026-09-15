# TalentSphere — Exploration Notes

Running decision log for the `/opsx:explore` session on TalentSphere.
Every decision is recorded with the question asked, the options considered, what was
decided, and why. Updated after each decision.

- **Started:** 2026-08-13 · **Last updated:** 2026-08-25
- **Status:** **Explore is complete; propose is underway, feature by feature (see Propose
  progress below).** The delivery model went through a further correction on 2026-08-24 — see
  **Part D**, especially **D.8**, which supersedes D.2/D.3's propose-per-backlog-item model with
  propose-per-**feature**. **D.9 and D.10** (2026-08-25) complete the full backlog decomposition —
  **79 items (`TS-BL-001`-`TS-BL-079`) across all 12 finalized features** — **now 80, after
  [D.10.1](#d101--gap-correction-ownership-reassignment-was-asserted-twice-and-decomposed-never-2026-08-25)
  added the missing `TS-BL-080` during `identity-and-access`'s propose conversation**. **D.11** (2026-08-25)
  settles propose-before-schedule ordering. `AGENTS.md` reflects the corrected model, including a
  convention (added 2026-08-25) for fixing genuine gaps found in *this* document from inside a
  propose conversation, rather than working around them. A standing product-quality bar —
  visually excellent, never overcompacted, all AI output concise and precise — is recorded in
  [S.5](#s5--ai-output-conciseness--visual-craft-standard-added-2026-08-20).
- **Repo state at start:** greenfield; empty `openspec/` scaffold, no code, no tech stack

### Propose progress

One row per feature. **Two different orderings, don't conflate them:**
- **`Plan #`** — the fixed dependency order from D.9/D.10, decided once, never renumbered.
- **`Seq`** — the *actual* chronological order features get proposed in, filled in only once a
  feature finishes. These can diverge (a later-`Plan #` feature can genuinely finish before an
  earlier one) — `Seq` is what answers "what order did this really happen in," which `Plan #`
  alone can't once that happens.

**A propose conversation may update its own row to `🟡 Proposed, pending verification` and fill in
its own `Seq` number when it finishes — it may not mark itself ✅.** To assign `Seq`, check the
highest `Seq` number already used in this table and take the next one (same convention as wave
numbering in `AGENTS.md` — check the max already assigned, don't guess). Verification is an
independent audit run from the explore conversation after the fact (see rows 1-2 for what that
looked like); a change grading its own homework isn't verification. ✅ only gets written here once
that audit has actually run and passed.

| Plan # | Seq | Feature | Phase | Items | Status |
|---|---|---|---|---|---|
| 1 | 1 | `platform-core` | 1 | TS-BL-001–006 | ✅ Proposed & verified 2026-08-25 |
| 2 | 2 | `design-system` | 1 | TS-BL-007–012 | ✅ Proposed & verified 2026-08-25 |
| 3 | 3 | `identity-and-access` | 1 | TS-BL-013–017 | ✅ Proposed & verified 2026-08-25 |
| 4 | 4 | `access-control-and-admin` | 1 | TS-BL-018–026, 080 | ✅ Proposed & verified 2026-08-25 |
| 5 | 5 | `ai-platform-governance` | 1 | TS-BL-027–032 | ✅ Proposed & verified 2026-08-25 |
| 6 | 6 | `hiring-postings` | 2 | TS-BL-033–040 | ✅ Proposed & verified 2026-08-26 |
| 7 | 7 | `candidate-intake` | 2 | TS-BL-041–048 | ✅ Proposed & verified 2026-08-26 |
| 8 | 8 | `matching-and-ranking` | 2 | TS-BL-049–055 | ✅ Proposed & verified 2026-08-26 |
| 9 | 9 | `interview-pipeline` | 3 | TS-BL-056–063 | ✅ Proposed & verified 2026-08-26 |
| 10 | 10 | `decision-and-offers` | 3 | TS-BL-064–070 | ✅ Proposed & verified 2026-08-26 |
| 11 | 11 | `insight-and-reporting` | 4 | TS-BL-071–074 | ✅ Proposed & verified 2026-08-27 |
| 12 | 12 | `resurfacing-and-communications` | 5 | TS-BL-075–079 | ✅ Proposed & verified 2026-08-27 |

### Pending cross-feature obligations

Fixes a propose conversation identified in **another, already-verified** feature's artifacts.
Recorded here because the change that needs fixing doesn't know it needs fixing — an apply
conversation reading only that feature's own artifacts would never see the gap. Each is a
`/opsx:update` run against the named change, in its own conversation
([AGENTS.md](../../AGENTS.md)'s per-change rule). **Clear these before `/opsx:apply` touches the
items involved**; none of them blocks further propose work.

| Owed by | What | Found by | Status |
|---|---|---|---|
| `interview-pipeline` | Its `TS-BL-056` / `057` / `061` declare no caller for §11.1's posting-state advances into `screening`, `interviewing` and `scorecard_review` — the transitions are declared by `hiring-postings`' `TS-BL-038` but nothing invokes them. Triggers, per `decision-and-offers` D9a: first shortlist, first round scheduled, first scorecard approved. Until this lands, `decision-and-offers` task 6.16's end-to-end `open`→`filled` test is **expected to fail**, and must not be made to pass by relaxing the closure guard. | `decision-and-offers` (2026-08-26) | ✅ **Resolved 2026-08-27** via `/opsx:update` on `interview-pipeline`. All three advances are built, one per named item — `open → screening` on the first shortlist (`TS-BL-056`, task 1.19), `screening → interviewing` on the first round scheduled (`TS-BL-057`, task 2.16), `interviewing → scorecard_review` on the first scorecard approved (`TS-BL-061`, task 6.18) — each with a requirement and scenarios in its own delta spec, and the reasoning in that change's `design.md` **D10a**. Both required properties hold: **monotone and idempotent** (fires only from the immediately preceding state, no-op otherwise, never backwards) and **attributed to the human whose action triggered it, inside that action's transaction**, which is what keeps `hiring-postings`' "no transition fires without an authenticated actor" assertion true. **Two scope calls worth recording:** D9a's third property, *recompute once on reopen*, is **deliberately not built** here — a reopen is a `filled → open` transition this feature never performs and cannot observe, so it stays `decision-and-offers`' concern for its own three, where it is built. And **no new `depends_on` edge onto `TS-BL-038` was added**: these three *invoke* an already-declared transition rather than *registering* one, which is the same line `decision-and-offers` drew when it gave `TS-BL-069` that edge and `TS-BL-064`/`TS-BL-067` none. `decision-and-offers` task 6.16's end-to-end `open`→`filled` test is no longer expected to fail; it was left untouched, as that feature's own artifact. |
| `ai-platform-governance` | Its task 1.18 is the only thing that records `OD-003` (the undecided model provider) in `KNOWN_ISSUES.md`, and it lands with `TS-BL-027`. Until then, any feature writing a reconciliation paragraph can miscite that constraint as already-recorded — four features hit this before it was pinned down. Not a defect in 1.18; noted so a fifth doesn't repeat it. | `interview-pipeline`, `matching-and-ranking` (2026-08-26) | ⬜ Open (resolves when `TS-BL-027` is applied) |
| `access-control-and-admin` | **Page-catalog granularity for the reporting surfaces.** `TS-BL-022` task 5.1 seeds one catalog entry per `§14.2` screen, and `§14.2` carries a **single** `Reports` screen whose capabilities span both *"practice dashboard"* and *"AI audit"*. One page key cannot express the posture `C-02` and `D16` require: granting the Auditor View to reach `insight-and-reporting`'s `TS-BL-074` also grants `TS-BL-071`/`072`'s candidate-derived aggregates, while denying it makes `TS-BL-074` unreachable for the only role it is built for. No override fixes it — `user_permission_overrides` act on a page, so a deny lands on the whole key. Owed: at minimum two entries (operational reporting vs. governance reporting) with `TS-BL-022`'s seed granting the Auditor the second and denying the first. See `insight-and-reporting` `design.md` D4. | `insight-and-reporting` (2026-08-27) | ✅ **Resolved 2026-08-27** via `/opsx:update` on `access-control-and-admin`. `Reports` now seeds as two entries — operational and governance reporting — under a general splitting rule (a screen splits when its capabilities are required by two roles whose postures differ), recorded in that change's `design.md` D5 and its `access-control/permission-seed` spec, with tasks 5.1a, 5.7, 5.7a and 5.14. **One refinement to the fix as originally worded:** the Auditor's operational-reporting cell is seeded **unset, not an explicit deny** — `AUTHZ-005`'s deny-wins crosses roles, so an explicit deny would silently strip operational reporting from a user holding Practice Manager alongside Auditor. Deny-by-default gives the required posture for an Auditor-only user at no such cost. `/opsx:apply` on `TS-BL-071` and `TS-BL-074` is no longer gated. |
| `matching-and-ranking` | **Prose fix in its `design.md` D4.** D4's conclusion is correct — *"`insight-and-reporting`'s AI-agreement-rate metric... needs the version that was current when the human decided... That query is available from the version table"* — but it attributes the capability to the version table alone. The version table establishes which boards exist; it does not establish which one was **active at a given moment**. The audited ranking-activation event supplies that second half, which D4's own task 3.13 already mandates (*"Write every ranking activation... through the audit path"*), so the conclusion holds — the sentence is imprecise, not false. Owed: sharpen it to name the audited activation event alongside the version table, so an apply conversation reading D4 alone does not build the version table and conclude the historical query is available without ensuring activation is audited and queryable. See `insight-and-reporting` `design.md` D7. | `insight-and-reporting` (2026-08-27) | ✅ **Resolved 2026-08-27** via `/opsx:update` on `matching-and-ranking`. D4's closing paragraph now names both halves: the version table establishes **which boards exist** (and, since no entry is updated in place, that each still reads as it did), while the **audited activation event** — which that feature's own task 3.13 already mandates for every activation — establishes which board was **live at a given moment**. **Kept deliberately short:** the correction is stated and the full three-step join is left to `insight-and-reporting` `design.md` D7 rather than restated there, so the two explanations cannot drift apart as either feature is revised. Wording only — the version table, the two-column pointer, its single writer and task 3.13 are all untouched, and no spec or task was edited. |
| `hiring-postings` | **Close the `practice` vocabulary.** Its own Open Questions asks whether `practice` becomes a reference table or stays a string and defers explicitly: *"`insight-and-reporting` is the feature with a real stake, and it is unproposed."* Now answered: a reference table is not what fixes it — an **audited runtime-configuration list, selected from rather than typed**, is (the shape `interview/shortlisting` already uses for disposition reason codes, where retiring a code leaves historical records readable). Free text fragments every group-by — *"Data & AI"*, *"Data and AI"*, *"data-ai"* become three practices — and a later rename silently splits a practice's own history, which makes `TS-BL-072`'s historical rates wrong invisibly. See `insight-and-reporting` `design.md` D12. | `insight-and-reporting` (2026-08-27) | ✅ **Resolved 2026-08-27** via `/opsx:update` on `hiring-postings`. Practice stays a string on `job_postings` — **no schema change, no new join, no new table** — and what changes is that the string is selected from a closed list held in `platform-core`'s audited runtime-configuration registry. Built as `hiring/job-posting`'s new requirement **"The practice vocabulary is audited configuration"**, mirroring `interview/shortlisting`'s disposition-reason-code requirement clause for clause (audited add/retire/rename with previous value, new value and mandatory reason; no unaudited setter; a retired value leaves historical postings readable under the value they were recorded with), plus one scenario that requirement does not need — a practice outside the configured list is **rejected**, which is what makes it selected-from rather than typed. Tasks 5.6a and 5.6b under `TS-BL-037`; reasoning in that change's `design.md` **D13**, and its own Open Questions entry removed since it is no longer open. **Two things deliberately not done:** no `depends_on` edge was added — this consumes the existing runtime-config registry, exactly as `TS-BL-040` already does with none — and §29 is cited as a *principle* alongside `ENG-010` rather than as enumerating practice, which it does not, matching how `interview/shortlisting` cites it. Scope held to quality, as recorded: nothing was gated on it and nothing in `insight-and-reporting` was touched. |

**Layout:** decisions are recorded round by round in the order they were made. A reference
specification arrived on 2026-08-14 and is reconciled against every decision in
[Part R — Reference Specification Reconciliation](#part-r--reference-specification-reconciliation)
at the end of this document. The `Ref` column below links each decision to its reconciliation
entry.

---

## Decision index

`Ref` legend: ✅ validated by the reference spec · 🔓 resolved an open item · ⚠️ conflict open,
**our decision stands** · ✔ conflict **resolved** · ➖ spec is silent

| ID | Topic | Decision | Round | Ref |
|----|-------|----------|-------|-----|
| [D01](#d01--interviewer-role) | Interviewer role | 4th role: Interviewer, assignment-scoped — **amended: 9 roles** | 1 | ✅ ✔ [C-02](#c-02--nine-roles-vs-four) |
| [D02](#d02--data-scoping) | Data scoping | ~~Global within role~~ → **configurable reads, scoped writes** | 1 | ✔ [C-03](#c-03--recruiters-scoped-to-assigned-postings) |
| [D03](#d03--where-the-5-cap-binds) | 5-candidate cap | Binds at final selection only | 1 | ✅ |
| [D04](#d04--outbound-communication-scope) | Outbound comms | ~~Full: calendar + candidate email~~ → **internal email early, rest deferred** | 1 | ✔ [C-04](#c-04--calendar-and-candidate-email-deferred-to-the-final-phase) |
| [D05](#d05--ai-context-visible-to-interviewers) | AI context to interviewers | ~~Full context~~ → **gaps yes, score no** | 2 | ✔ [C-10](#c-10--interviewers-and-the-ranking-board) |
| [D06](#d06--calendar-integration-depth) | Calendar depth | Full calendar API — **deferred to a late slice** | 2 | ✔ [C-04](#c-04--calendar-and-candidate-email-deferred-to-the-final-phase) |
| [D07](#d07--target-volume) | Target volume | Small: <5k candidates, <30 postings — **amended: vector retrieval adopted (Vertex AI Vector Search)** | 2 | ✔ [C-01](#c-01--vector-store) |
| [D08](#d08--hubble-id-at-onboarding) | Hubble ID direction | **Manual capture**; provisioning ruled out | 2 | 🔓 |
| [D09](#d09--tech-stack) | Tech stack | **Resolved: FastAPI/Python** + React/TS + PostgreSQL | 3 | 🔓 |
| [D10](#d10--calendar-platform) | Calendar platform | Google Workspace | 3 | ➖ |
| [D11](#d11--scorecard-structure) | Scorecard structure | ~~Per round~~ → **notes per round + consolidated scorecard** | 3 | ✔ [C-05](#c-05--one-scorecard-per-application) |
| [D12](#d12--job-description-approval) | JD approval | ~~Recruiter drafts, PM approves~~ → **PM drafts, RM approves** | 3 | ✅ ✔ [C-12](#c-12--who-drafts-the-jd) |
| [D13](#d13--phasing--change-decomposition) | Phasing | ~~Nine slices~~ → 14 slices, 5 waves → **FINAL: consolidated, 1-week sprints** | 4 | ✅ |
| [D14](#d14--evergreen-posting-rules) | Evergreen postings | Finite with n = unlimited | 4 | ✅ |
| [D15](#d15--scorecard-competencies) | Competencies | ~~AI-suggested~~ → **fixed dimensions**; 5-point scale | 4 | 🔓 ✔ [C-06](#c-06--fixed-scorecard-dimensions) |
| [D16](#d16--admin-access-to-candidate-pii) | Admin PII | Config only, audited break-glass | 4 | ✅ |
| [D17](#d17--priority-lane-carry-forward) | Carry-forward | Carry forward + **one confirmatory round** | 5 | ✔ [C-07](#c-07--carry-forward-has-no-basis-in-the-spec) |
| [D18](#d18--resume-parsing-approach) | Resume parsing | Hybrid — deterministic + LLM | 5 | ✅ |
| [D19](#d19--offer-tracker-scope) | Offer scope | Status tracker only | 5 | ✅ |
| [D20](#d20--priority-slot-mechanics) | Slot mechanics | 5 per vacancy, ranked, displacement w/ reason | 5 | ✅ |
| [D21](#d21--carry-forward-eligibility) | Carry-forward eligibility | Same JD auto; different JD needs PM attestation | 6 | ✔ [C-07](#c-07--carry-forward-has-no-basis-in-the-spec) |
| [D22](#d22--staleness-windows) | Staleness windows | 90d / 12mo — **configurable, audited, capped by retention** | 6 | ✔ [C-11](#c-11--staleness-windows-fixed-vs-configurable) |
| [D23](#d23--candidate-deduplication) | Deduplication | File hash + email exact = merge; three fuzzy signals = review | 6 | ✅ 🔓 |
| [D24](#d24--interview-note-locking) | Note locking | ~~Lock on approval~~ → **version forever, matrix-controlled edit** | 6 | ✔ [C-08](#c-08--note-versioning-without-a-lock) |

## Adopted from the reference specification

| ID | Adoption | Status |
|----|----------|--------|
| [G-01](#g-01--mandatory-criteria-status) | Mandatory criteria status — AI proposes, human confirms blocks | Adopted |
| [G-02](#g-02--evidence-references-and-source-labelling) | Evidence references and source labelling | Adopted |
| [G-03](#g-03--insufficiency-outputs) | Insufficiency outputs | Adopted |
| [G-04](#g-04--resume-file-hash) | Resume file hash — document-level, review if cross-candidate | Adopted |
| [G-05](#g-05--extraction-confidence) | Extraction confidence | Adopted |
| [G-06](#g-06--human-override-as-a-first-class-record) | Human override as a first-class record | Adopted |
| [G-07](#g-07--ranking-versions) | Ranking versions | Adopted |
| [G-08](#g-08--posting-text-separate-from-internal-jd) | Posting text separate from internal JD | Adopted |
| [G-09](#g-09--compliance-text-status) | Compliance text status — gate defaults to `not_required` | Adopted |
| [G-10](#g-10--application-source-type) | Application source type | Adopted |
| [G-11](#g-11--forced-session-revocation) | Forced session revocation | Adopted |
| [G-12](#g-12--graceful-degradation) | Graceful degradation | Adopted |
| [G-13](#g-13--closure-requires-onboarding-complete--valid-hubble-id) | Closure requires onboarding complete + valid Hubble ID | Adopted |
| [G-14](#g-14--joining-status-and-blocker-reason) | Joining status and blocker reason | Adopted |

---

## Round 1 — scope forks

### D01 — Interviewer role

**Question:** Who writes the structured interview notes in the Interview Console, and do
non-recruiter interviewers log into TalentSphere?

**Options considered:**
1. **4th role: Interviewer** — technical panelists get scoped logins
2. Recruiter transcribes — panelists give feedback verbally, recruiter enters it
3. Practice Manager is the interviewer — PMs interview and write notes directly
4. Not decided yet

**Decision:** Option 1 — add a fourth role, **Interviewer**, scoped to only their own
assigned interviews.

**Why:** Multi-round technical hiring needs real panelists. Options 2 and 3 make interview
notes second-hand and muddy scorecard attribution — a recruiter "approving" an AI-drafted
evaluation of an interview they did not conduct is not meaningful human review.

**Consequences:**
- Roles become **many-to-many** — a PM who also sits on panels is one user with two roles;
  permission = union of grants, with the most restrictive scope applying per resource.
- User population jumps from dozens (recruiters/PMs) to potentially hundreds (engineers).
- Recommended provisioning model: **assignment triggers provisioning** — assigning a
  panelist creates/activates a scoped Interviewer account; account auto-dormants after N
  days with no assignments. This gives feature 2 ("user activation") a concrete meaning.
- Interviewer sees: their assigned interviews, the JD and resume for that candidate, their
  own notes, their own scorecard. Not other candidates, not other panelists' notes.
- **Collides with [D02](#d02--data-scoping)** — see that entry.

---

### D02 — Data scoping

**Question:** How is data visibility scoped beyond the page/action matrix — can a Practice
Manager see another practice's candidates?

**Options considered:**
1. Scoped by Practice — row-level filtering per practice on every query
2. **Global within role** — any Recruiter sees all postings/candidates; any PM can shortlist anywhere
3. Own records + practice read — write on what you own, read across your practice
4. Not decided yet

**Decision:** Option 2 — **global within role**.

**Why:** Dramatically simpler. No Practice entity needed for permissions, no row-level
filtering on every query, and resurfacing draws trivially from the entire candidate
database.

**Consequences:**
- **Conflicts with [D01](#d01--interviewer-role).** An Interviewer with global visibility is
  a recruiter with extra steps. Resolution: **global for Recruiter/PM/Admin, assignment-scoped
  for Interviewer** — exactly one row-level predicate (`interview.panelist = current_user`)
  instead of the dozen practice-scoping would have required.
- The page/action matrix needs a **scope column**, not just a checkbox grid.
- Recommendation (not yet confirmed): still model `Practice` as a **data attribute** on
  JobPosting for filtering, routing, and reporting — just don't enforce it in permissions.
  Cheap insurance; retrofitting row-level scoping onto a system built global is expensive.
- Open: confidential/executive requisitions may still need a restricted-visibility flag.
- Open: Admin access to candidate PII ([see open questions](#open-questions)).

---

### D03 — Where the 5-cap binds

**Question:** Where in the pipeline does the 5-candidate cap actually bind?

**Options considered:**
1. **Final selection only** — shortlist/interview uncapped, cap on priority slots feeding Offer
2. At shortlisting — only 5 may ever be shortlisted per vacancy
3. Soft cap with override — 5 is a warned threshold requiring a reason to exceed
4. Not decided yet

**Decision:** Option 1 — the cap binds at **final selection**.

**Why:** Matches the ordering in the original feature list (feature 15 sits after scorecards,
before Offer) and does not constrain recruiters mid-funnel. Capping shortlisting would mean
never interviewing a sixth person — no recruiter would accept that.

**Consequences:**
- Shortlisting and interviewing are uncapped; overflow sits on a **ranked bench** per posting.
- A vacated slot (declined offer, withdrawal) promotes the next bench candidate, logged.
- On posting closure: slot holders not offered → **priority lane**; bench → standard
  resurfacing pool.

**Sub-decisions still open:**
- 5 per vacancy (so a 3-vacancy posting allows 15) or 5 per posting? Original wording says
  "per finite vacancy," implying the former.
- Slots ranked 1..n, or an unordered set?
- Can a better candidate **displace** a slot holder? If yes, mandatory reason + audit.
- Vacancy count drops mid-flight (3 → 1): who evicts the surplus, and where do they go?
- A candidate holding slots on two postings, both extending offers → need a
  **one active offer per candidate** rule.
- Evergreen postings have no vacancy count, so the cap is undefined for them
  ([see open questions](#open-questions)).

---

### D04 — Outbound communication scope

**Question:** Does TalentSphere communicate outward — calendar invites, panelist
notifications, candidate emails — or is it entirely internal?

**Options considered:**
1. Internal tasks + in-app only — nothing leaves the system
2. Internal email, no candidate email — notify employees only
3. **Full: calendar + candidate email**
4. Not decided yet

**Decision:** Option 3 — **full outbound**: calendar invites, panelist notifications, and
candidate-facing email.

**Why:** Chosen for completeness of the hiring workflow. Option 1 would leave the Console
holding an interview time that nothing in the world enforces — the recruiter would still
have to contact the candidate manually.

**Consequences:**
- Adds a **`Communication` entity** not in the original 19 features: every outbound message
  logged with template, timestamp, delivery status, and bounce handling.
- Adds an external dependency with an owner outside the team (see [D06](#d06--calendar-integration-depth),
  [D10](#d10--calendar-platform)).
- Compliance edge: emailing candidates makes them aware they are in the database, which makes
  data-subject requests (access, correction, erasure) practically reachable. In some
  jurisdictions, notifying a candidate about an AI-assisted process carries its own disclosure
  requirement. The candidate email template is a **governance artifact**, not just copy.
- Recommendation (not yet confirmed): **outbound only** — log what is sent, do not parse
  replies. Inbound mail processing is a separate project.

---

## Round 2 — consequences

### D05 — AI context visible to interviewers

**Question:** Can an Interviewer see the AI ranking score, fitment summary, and gap summary
for the candidate they are about to interview?

**Options considered:**
1. No score, questions only (**recommended by Claude**) — reveal score/fitment/gaps only after notes are submitted
2. **Full AI context upfront** — score, fitment, gaps, and questions all visible pre-interview
3. Gaps yes, score no — directive without revealing the verdict
4. Not decided yet

**Decision:** Option 2 — **full AI context upfront**.

**Why (user's rationale):** Efficient use of interview time — the panelist can probe the
gaps the AI identified directly.

**Tradeoff accepted (recorded deliberately):** This was chosen against the recommendation.
The consequence is that the interview no longer produces a signal independent of the AI
ranking — it becomes confirmation of it rather than a check on it. Anchoring on a visible
score is a known effect, and it weakens the "human validates the AI" story for feature 19.

**Agreed mitigation:**
- Structured note templates require **observed evidence per competency** before a scorecard
  can be submitted.
- Every `AIRun` records that the score was **visible pre-interview**, so a later review can
  account for it.

**Consequence / design detail:** AI context shown to an Interviewer should be **per-candidate**,
not the comparative leaderboard. Rank position ("#1 of 12") leaks the existence and standing
of other candidates and should stay hidden from the Interviewer role.

> ### ✔ CORRECTED 2026-08-25 — the mitigation cannot be a field on the `AIRun`; it is a separate disclosure record
>
> **The second agreed mitigation above is unbuildable as written.** It says: *"Every `AIRun`
> records that the score was **visible pre-interview**, so a later review can account for it."*
> Two decisions made *after* this one make that impossible:
>
> - **The run record is written before invocation** — `talentsphere-wave-1-foundation`'s
>   `design.md` D9, and `AI-002`. At write time, whether the output will later be shown to an
>   interviewer is simply unknowable: a ranking run happens at submission or posting time, the
>   interview happens days later.
> - **The run record is append-only** — `openspec/config.yaml`'s standing rule that "audit and AI
>   run tables are append-only: the application role holds INSERT and SELECT only," enforced by a
>   database grant per `changes/access-control-and-admin/design.md` D7. So the field cannot be set
>   after the fact either.
>
> A ranking output is also disclosed to many actors on many occasions, so a single boolean on the
> run could not represent it even if the ordering allowed it.
>
> **What replaces it:** a separate, append-only **disclosure record**, linked to the run, capturing
> actor, output, time, and context of disclosure. It answers exactly the question this mitigation
> wanted answered — whether a human judgement was formed before or after seeing AI output — from
> stored data rather than from a flag nothing could have set. Specified in
> `changes/ai-platform-governance/specs/ai-platform/ai-run-logging/spec.md` and reasoned in that
> change's `design.md` D11. **The rest of D05 is unaffected**: the decision (full context upfront),
> the accepted anchoring trade-off, the structured-note mitigation, and the per-candidate/no-rank
> rule all stand exactly as written.
>
> **Why this was missed.** D05 was decided in Round 2, before write-before-invoke ordering or the
> append-only grant existed as decisions at all — neither arrived until the wave-1 design and the
> access-control audit design respectively. Nothing subsequently revisited D05 against them, and
> the mitigation was never carried into any spec by any feature, so no propose conversation before
> this one had reason to test whether it could be built. `TS-BL-054` (`matching-and-ranking`) owns
> the AI-context *visibility rules*; the record those rules must leave had no owner until now.
>
> Recorded here rather than fixed silently, per this document's convention and `AGENTS.md`'s
> "Finding a genuine gap in `exploration-notes.md` itself" rule — the same treatment
> [D09](#d09--tech-stack)'s background-jobs row and [S.6](#s6--two-things-this-reconciliation-missed-found-during-design-systems-propose-conversation-2026-08-25)
> received. Found during `ai-platform-governance`'s propose conversation, 2026-08-25.

> ### ⚠ CORRECTED 2026-08-26 — the block above re-asserts a decision [C-10](#c-10--interviewers-and-the-ranking-board) had already reversed
>
> **What was wrong.** The correction block immediately above, added 2026-08-25, closes by listing
> what its own fix leaves untouched. Its wording was:
>
> > **The rest of D05 is unaffected**: the decision (full context upfront), the accepted anchoring
> > trade-off, the structured-note mitigation, and the per-candidate/no-rank rule all stand exactly
> > as written.
>
> **"The decision (full context upfront)" does not stand.** It was reversed on **2026-08-14** —
> eleven days before that block was written — by [C-10](#c-10--interviewers-and-the-ranking-board),
> whose resolution states in its first line that it *"**Amends [D05](#d05--ai-context-visible-to-interviewers), reversing it**"*
> and settles the opposite outcome: **gaps yes, score no.** Interviewers see source-labelled fitment
> and gaps plus the JD and resume; they do not see `latest_match_score`, rank position, or other
> candidates.
>
> **What is true now, item by item**, replacing that sentence:
>
> | D05 element | Status |
> |---|---|
> | The decision — full AI context upfront | **Reversed** by `C-10`. Build "gaps yes, score no." |
> | The accepted anchoring trade-off | **Moot for Interviewers.** `C-10` removed the score that caused it. It still describes the board's own audience, who see the score by design. |
> | The logging mitigation ("`AIRun` records the score was visible pre-interview") | **Doubly superseded** — `C-10` made it unnecessary for Interviewers, and the block above independently shows it was unbuildable in that form. |
> | The structured-note mitigation | **Stands.** `C-10` explicitly retains it: *"the evidence-required note template remains valuable and is retained."* |
> | The per-candidate / no-rank-position rule | **Stands**, untouched by `C-10`, and is the tighter of the two constraints now that the score is withheld anyway. |
>
> **The disclosure record itself is unaffected and still required.** It was reasoned from D05's
> mitigation, but it does not depend on it: disclosure records answer "was this AI output seen, by
> whom, when" for *any* AI output shown to *any* actor — the board's score, fitment and gaps shown
> to a Practice Manager, and the fitment and gaps shown to an Interviewer under `C-10`. Only the
> motivating example in that reasoning is stale, not the mechanism or its ownership.
>
> **Why this was missed.** The [Decision index](#decision-index) row for D05 has read
> *"~~Full context~~ → **gaps yes, score no**"* since `C-10` landed, so the reversal was never lost
> from this document — it was lost from *one paragraph* of it. The 2026-08-25 block was written while
> reasoning about `AIRun` ordering and the append-only grant, and its closing "what this doesn't
> touch" list was assembled from D05's own body text (which correctly still reads "Option 2 — full
> AI context upfront," since a superseded decision is preserved rather than rewritten here) without
> re-checking that body text against Part R. A reader who arrives at D05 through that block rather
> than through the index therefore reads a reversed decision as current — and at least one artifact
> already did: `ai-platform-governance`'s `ai-platform/ai-run-logging` spec sources its
> disclosure-record requirement to *"`D05`, whose accepted trade-off — full AI context shown to an
> Interviewer before the interview."* That citation's conclusion survives per the paragraph above;
> its premise does not. Correcting another change's artifacts is `/opsx:update`'s job, not this
> document's, so it is flagged rather than edited here.
>
> Found during `matching-and-ranking`'s propose conversation, 2026-08-26 — the feature that owns
> `TS-BL-054` and therefore had to build against one of the two readings. Recorded here per
> `AGENTS.md`'s shared-document convention, quoting the prior wording rather than replacing it
> silently, the same treatment [D09](#d09--tech-stack)'s background-jobs row and
> [S.6](#s6--two-things-this-reconciliation-missed-found-during-design-systems-propose-conversation-2026-08-25)
> received.

---

### D06 — Calendar integration depth

**Question:** How deep does the calendar integration go?

**Options considered:**
1. **Full calendar API** — event CRUD plus free/busy lookup
2. ICS attachments in email (**recommended by Claude**) — standards-compliant invites, no API
3. ICS now, API later — sequences the external dependency off the critical path
4. Not decided yet

**Decision:** Option 1 — **full calendar API**.

**Why (user's rationale):** Best experience; automated panelist availability lookup is worth
the integration cost.

**Consequences:**
- This is the **critical path item whose timeline the team does not control**. App
  registration / delegation approval is owned by IT or Workspace admins.
- **Action recommended: start the approval process immediately**, before the proposal is
  written. It costs nothing and is the most likely thing to be blocking in week six.
- Belongs in the proposal as a **dependency with a named owner**, not as a story point.
- Recommendation (not yet confirmed): **TalentSphere is the source of truth; the calendar is
  a one-way projection.** Bidirectional sync is where this kind of integration fails. Trade-off:
  reschedules must happen in TalentSphere, which is a user-behavior bet.
- Platform resolved separately in [D10](#d10--calendar-platform).

---

### D07 — Target volume

**Question:** What volume should this be designed for? (Decides whether two-stage
retrieve→rerank matching is necessary or over-engineering.)

**Options considered:**
1. **Small: <5k candidates, <30 open postings**
2. Medium: ~5k–50k candidates, 30–150 postings
3. Large: 50k+ candidates, 150+ postings
4. Don't know yet

**Decision:** Option 1 — **small**.

**Why:** Single practice / early-stage volume. Brute-force-ish ranking is affordable at this
scale.

**Consequences — this is the single biggest scope reduction in the session:**
- **No vector store. No embedding pipeline. No batch resurfacing windows.** Dropped from scope.
- Matching is: SQL filter (skills, experience band, availability, recency, not-already-applied,
  not do-not-resurface) → typically 20–60 candidates pass → LLM-rank all of them at roughly
  $0.50–2 per posting.
- **Reverses an earlier recommendation:** at this volume, **fully automatic resurfacing on
  every posting publish is affordable**. No need to compromise with scheduled batches or
  manual-only triggers. Still want an async job queue so publishing does not block on LLM
  calls, plus a manual "find candidates" button.
- Design so it *degrades into* the two-stage machinery later: keep ranking behind an
  interface, keep the filter stage separate from the rank stage. Do not build it now.

---

### D08 — Hubble ID at onboarding

**Question:** At the Offer/Onboarding stage, what actually happens with the Hubble ID?

**Options considered:**
1. Manual capture only — a validated text field, ~1 day
2. Capture + verify via lookup — read API confirms the ID exists, ~1 week
3. TalentSphere provisions in Hubble — write API creates the employee record, ~1 month+
4. Need to confirm with platform team

**Decision (originally DEFERRED; resolved 2026-08-14 from the reference spec):**
**Manual capture, with optional validation. Provisioning is ruled out.**

| Finding | Source | Confidence |
|---|---|---|
| TalentSphere never creates employee records | §4.2 out-of-scope #10 — *"HRIS employee master creation"* | **High** — explicit exclusion |
| Recruiter enters the Hubble ID manually after onboarding | `OFF-004` | **High** |
| Validation is conditional, not guaranteed | `hubble_id_validated` — *"True when validation available and successful"*; §24 *"Exact endpoint must be confirmed"*; `OD-002` still open | **Low** — still needs your own answer |

**New rules inherited:** `OFF-006`/`OFF-007` — Hubble ID **uniqueness** validated among recruited
candidates, duplicates blocked except via controlled administrative correction.
`BR-019` — a candidate does not count toward vacancy fulfillment without a valid Hubble ID
(see [G-13](#g-13--closure-requires-onboarding-complete--valid-hubble-id)).

**Still open:** whether a validation endpoint exists at all (`OD-002`).

**Context on what Hubble is:** Not independently verifiable — it is internal to the
organization. The user's read (internal identity/HR platform) is the most likely one given
"REST Login API" plus "Hubble ID at onboarding." To confirm with the platform team.

**Integration surface implied regardless of the answer:**
- Feature 2 already fixes the architecture: **authentication external, authorization internal.**
  Hubble authenticates any employee; TalentSphere separately decides who gets in and as what.
- What claims does the login response return (employee ID, email, department, manager, title)?
  Every claim received is a field not maintained locally.
- Token lifetime, refresh, revocation. Is MFA Hubble's responsibility?
- **Leaver/deactivation sync** — push notification, poll, or discover on failed login? A
  terminated recruiter with a live session is an audit finding. And a departing recruiter who
  owned 12 open postings needs **ownership reassignment** (not in the original feature list).
- JIT user creation on first login, or Admin pre-provisioning?
- Does Hubble own the practice/org hierarchy? If so it is the source of truth and should not
  be duplicated.
- **Internal candidates / rehires** — an employee applying internally already has a Hubble ID
  at *submission*, not at onboarding. Affects dedup, permissions, and confidentiality.

---

## Round 3 — foundations

### D09 — Tech stack

**Question:** What's the tech stack? The repo is empty and `openspec/config.yaml` has an
empty `context:` block.

**Options considered:**
1. Java / Spring Boot + React + PostgreSQL
2. .NET + React
3. Node/TypeScript full-stack
4. **Something else / not decided**

**Decision (updated 2026-08-14): PARTIALLY RESOLVED. The backend choice remains open and still
blocks `/opsx:propose`.**

The reference spec §6 recommends a stack, explicitly non-binding — it is headed *"Recommended"*
and states the final stack *"shall be confirmed by the engineering leadership team."*

| Layer | Recommended | Treat as |
|---|---|---|
| Frontend | React + TypeScript | **Settled** — uncontroversial |
| Database | PostgreSQL | **Settled** — maps to Cloud SQL on GCP |
| Backend | FastAPI/Python **or** Node.js/NestJS | **Still your call** |
| Background jobs | Celery, RQ, BullMQ, or Cloud Tasks | Follows the backend choice |
| Observability | OpenTelemetry | Settled |
| OCR | Tesseract or cloud OCR | Deferred — readiness only |

**GCP note:** Cloud Tasks and Vertex AI Vector Search already appear in the recommended stack.
There is no Azure-specific content anywhere in the reference spec, so there is nothing to migrate
away from.

> ### ✔ RESOLVED 2026-08-17 — FastAPI/Python
>
> **Backend: FastAPI/Python.** Full stack now settled:
>
> | Layer | Choice |
> |---|---|
> | Frontend | React + TypeScript |
> | Backend | **FastAPI (Python)** |
> | Database | PostgreSQL → Cloud SQL |
> | Background jobs | ~~Celery, RQ, or **Cloud Tasks**~~ — **SUPERSEDED 2026-08-25**, see the block below |
> | Vector store | **Vertex AI Vector Search** — resolved 2026-08-17 during infra planning, superseding the pgvector-on-Cloud-SQL recommendation in [C-01](#c-01--vector-store) below; the landing zone already enables `aiplatform.googleapis.com` per app |
> | Observability | OpenTelemetry |
>
> **No longer blocking.** `openspec/config.yaml`'s `context:` block can now be filled in.
>
> **New scope surfaced alongside this:** the target deployment is **GCP with Terraform and
> GitLab CI/CD**, following patterns from a prior "GCP AI landing zone" project the user
> referenced. That project was built under a different Claude Code account/session — **no memory
> of it exists in this session**, and nothing about its architecture should be assumed. See
> [Open questions](#open-questions) for how this gets resolved.

> ### ✔ SUPERSEDED 2026-08-25 — background jobs run the landing zone's async chain
>
> **The Background jobs row above is superseded.** Background work runs on **Pub/Sub → Eventarc
> → dispatcher → Cloud Workflows → Cloud Run Job** — the chain the user's GCP AI landing zone
> already implements as **reusable Terraform modules**, which TalentSphere consumes rather than
> reimplements. Confirmed by the user 2026-08-25, including the reusability.
>
> **Why the original row was never really a decision.** It says so itself: *"follows from
> FastAPI/Python."* It was derived from the backend language choice, and `reference/spec.md` §6
> lists Cloud Tasks as one of five equivalent options rather than recommending it. No comparison
> on merits was ever made.
>
> **Why this was missed on 2026-08-17.** The landing zone was read in full that same day, and
> that reading *did* amend one row of this very table on landing-zone grounds — the vector store,
> superseding [C-01](#c-01--vector-store). The background-jobs row simply was not revisited in
> the same pass. A gap in that reconciliation, not a reversal of it.
>
> **Why it stayed hidden until now.**
> [D.9](#d9--phase-1-backlog-decomposition-at-the-corrected-finer-grain-2026-08-25)'s `TS-BL-006`
> row already names this chain — but as a backlog title, with no rationale and no note that it
> superseded anything. Two conflicting mechanisms therefore sat in this document unreconciled,
> each looking authoritative, until the `platform-core` propose conversation hit the conflict.
> Recorded here rather than fixed silently, per this document's convention.
>
> **Honest trade-off, stated once.** Cloud Tasks is genuinely simpler at TalentSphere's scale —
> one concept instead of five, native task-name deduplication, per-queue retry configuration,
> and a much smaller Terraform and IAM surface for a team of 3-4 junior developers. Were
> TalentSphere standalone, it would likely be the better call. It loses because the chain's
> complexity is **already paid for** in modules that exist and work, and adopting Cloud Tasks
> would mean diverging from the organization's platform to avoid a cost already sunk. This is
> the same reasoning that made Vertex AI Vector Search beat pgvector, and the same lesson Sprint
> 0 learned expensively (`talentsphere-wave-1-foundation/sprint-0-outcome.md` §3).
>
> **Consequence for `TS-BL-006`:** its scope is *consuming* the landing zone's modules and
> registering TalentSphere's job types against them — not building the chain from primitives.
> The same relationship this repository already has with the shared pipeline template. See
> `changes/platform-core/design.md` D6.

**Still blocking:** `openspec/config.yaml` has an empty `context:` block that cannot be filled
until the backend is chosen.

---

### D10 — Calendar platform

**Question:** Which calendar platform, and who owns getting the approval?

**Options considered:**
1. Microsoft 365 / Graph — Entra app registration + tenant admin consent
2. **Google Workspace** — service account + domain-wide delegation
3. Both / mixed environment — ~2× integration surface
4. Need to check

**Decision:** Option 2 — **Google Workspace**.

**Why this is the easier combination:** The identity provider (Hubble) and the calendar
provider (Google) are different systems, which normally means the login token cannot touch the
calendar. Google's **domain-wide delegation** sidesteps that entirely — no per-user OAuth, no
consent screens, no second login, no per-user token refresh. One Workspace admin approval,
once. Materially smaller than the Microsoft path would have been given Hubble is the IdP.

**Consequences:**
- Service account can create/update/cancel events as the recruiter, do free/busy lookups for
  panelists, and send candidate email via the Gmail API under the same delegation.
- **Needs a user → Google email mapping.** If Hubble's login response includes corporate email
  and it matches the Workspace primary address, this is free. If not, a mapping table is
  required. To check when confirming Hubble claims ([D08](#d08--hubble-id-at-onboarding)).
- **Open:** candidate-facing sending address — a shared `talent@` mailbox (replies land in one
  place) vs. the individual recruiter (more personal, replies scatter).
- Still recommended: outbound only, no inbound parsing.

---

### D11 — Scorecard structure

**Question:** How are scorecards structured, and where do the evaluation criteria come from?

**Options considered:**
1. **Per round + consolidated view** — separate scorecard per round, shown side by side, no aggregation
2. Per round + AI consolidation — adds an AI summary across rounds with its own approval gate
3. One scorecard per application — a single document appended to each round
4. Not decided yet

**Decision:** Option 1 — **per round, with a consolidated view at selection**.

**Why:** Cleanest attribution — each panelist owns and approves their own scorecard. No
averaged score, so no fake math papering over panelist disagreement; the human weighs the
rounds. Also avoids adding a fifth AI touchpoint with its own approval gate and its own
failure mode when panelists disagree.

**Consequences:**
- AI touchpoints stay at four: JD drafting, ranking bundle, scorecard drafting, resume parsing.
- Interview notes must **lock on scorecard approval**, or the record is worthless.
- **Open:** where competencies come from (global library / AI-suggested per JD / fixed per
  interview type) and the rating scale.
- **Open:** the rule for consolidating conflicting panelist scorecards into one selection
  decision — currently "the PM decides," which may be sufficient.

> ### ⚠ CORRECTED 2026-08-26 — D11's note-locking consequence was reversed by [C-08](#c-08--note-versioning-without-a-lock) and never annotated here
>
> **What was wrong.** D11's second consequence above reads:
>
> > - Interview notes must **lock on scorecard approval**, or the record is worthless.
>
> **That is no longer true, and nothing on D11 said so.** [C-08](#c-08--note-versioning-without-a-lock)'s
> resolution of 2026-08-14 adopted `INT-008`/`BR-017`'s model — **versioning, not freezing**: every
> edit to a submitted note creates a new version, forever, with **no lock on scorecard approval** and
> no author restriction. C-08 is written as an amendment to [D24](#d24--interview-note-locking), which
> is where the *mechanism* was chosen; it never touched D11, which is where the *requirement for the
> mechanism* was stated.
>
> **Why the Decision index did not catch it.** D11's `Ref` column reads `✔ C-05`, and
> [C-05](#c-05--one-scorecard-per-application) genuinely does resolve D11 — but only its **structure**
> half (per-round scorecards → per-round notes plus one consolidated scorecard). C-05 says nothing
> about locking. So a reader who arrives at D11 through the index sees a decision marked resolved,
> reads its consequences, and finds a locking requirement carrying no supersession marker of any
> kind. D24's body text has the mirror-image staleness — its consequence *"Satisfies the D11
> requirement that notes lock on scorecard approval"* is equally stale — but D24 is a **wholly
> superseded decision** whose body this document preserves rather than rewrites, and its index row
> already reads *"~~Lock on approval~~ → **version forever, matrix-controlled edit**"*. D11's is the
> half that reads as live, so D11 is where the note belongs.
>
> **What is true now:** notes never lock. `INT-008`/`BR-017` versioning applies from submission
> onward and forever; who may edit is a permission-matrix question (`Edit` on the Interview Console
> page), recommended seeded to Interviewer only. D11's underlying *concern* — that an approved
> evaluation and the evidence it was drafted from must agree — survives the reversal and is answered
> by a different mechanism: `scorecards.status = superseded` on a post-approval note edit, requiring
> re-approval. That mechanism was recorded under
> [Open questions](#open-questions) as "not yet decided" and is **decided** in
> `changes/interview-pipeline/design.md` D3, which builds it.
>
> **One further consequence above is stale but already annotated elsewhere**, so it is left alone:
> *"AI touchpoints stay at four"* was superseded by C-05's accepted cost (*"an additional AI
> touchpoint and approval gate"*) and by the ten-family registry recorded under
> [Open questions](#open-questions). Noted here only so a reader does not take this block's silence
> on it as endorsement.
>
> Found during `interview-pipeline`'s propose conversation, 2026-08-26 — the feature that owns
> `TS-BL-059` and therefore had to build against one of the two readings. Recorded per `AGENTS.md`'s
> shared-document convention, quoting the prior wording rather than replacing it silently, the same
> treatment [D09](#d09--tech-stack)'s background-jobs row, [D05](#d05--ai-context-visible-to-interviewers)'s
> re-asserted reversal and [S.6](#s6--two-things-this-reconciliation-missed-found-during-design-systems-propose-conversation-2026-08-25)
> received.

---

### D12 — Job Description approval

**Question:** Who approves a Job Description before postings can be published against it?

**Options considered:**
1. **Practice Manager approves** — real second-party gate
2. Recruiter self-approves — fast, but human-in-loop becomes a checkbox
3. Either, configurable per practice — two code paths, two test matrices
4. Not decided yet

**Decision:** Option 1 — **Practice Manager approves**.

**Why:** A real gate with real accountability — the person who owns the headcount signs off on
how the role is described. Self-approval would make "human approval" a formality and weaken the
governance story for feature 19.

**Consequences:**
- New UI surface: a **PM review queue** with an approve / request-changes loop, plus comments.
- Creates a bottleneck that needs designing, not discovering: PM unavailable → backup approver
  or delegation? PM is the drafter → who approves then? SLA or nudges on stalled reviews?
- **Approved JDs fork on edit.** Editing an approved JD creates a new version; live postings
  stay pinned to the version they were published against. Edit-in-place would silently
  invalidate every ranking score computed against that JD.

---

## Round 4 — structure and governance

### D13 — Phasing / change decomposition

**Question:** 19 features is far too much for one change. How should this be decomposed for
OpenSpec?

**Options considered:**
1. **Dependency-ordered slices** — nine changes, each finished before the next
2. Walking skeleton first — one thin end-to-end slice, then thicken
3. Two releases — R1 core loop, R2 resurfacing/dashboards/governance
4. Not decided yet

**Decision (original, 2026-08-13):** Option 1 — **nine dependency-ordered slices**, as below.

| # | Slice | Contents |
|---|-------|----------|
| 1 | Foundation | Hubble auth, user activation, RBAC + page/action matrix + scope column, Admin Cockpit, audit substrate |
| 2 | Jobs | JD Workspace with AI drafting + PM approval + versioning; Job Posting management; **AI-run log + versioned prompts** |
| 3 | Intake | Resume upload pipeline (validation, malware, parse, secure storage, OCR-ready); Candidate DB, dedup, resume versions, history |
| 4 | Matching | AI ranking, fitment/gap summaries, interview questions, output feedback |
| 5 | Funnel | Shortlisting + mandatory disposition reasons; interview scheduling tasks; Google Calendar; candidate email; Interview Console |
| 6 | Evaluation | Scorecard Center, AI-assisted scorecards, Interviewer UI |
| 7 | Decision | Priority slots + 5-cap, Offer & Onboarding tracker, closure rules, evergreen lifecycle |
| 8 | Resurfacing | Standard pool + priority lane |
| 9 | Insight | Dashboards, reports, export controls |

**Why (original):** Clearest to estimate, review, and course-correct. Critically, the
Responsible-AI substrate lands in **slice 2**, not at the end — JD drafting is the first LLM
call in the system, so `AIRun` logging and versioned prompts must exist by then or the history
cannot be backfilled.

> ### ✔ SUPERSEDED 2026-08-17 — revised to fourteen slices
>
> The nine-slice plan was sized before the C-01…C-04 conflict resolutions landed. Three of those
> resolutions roughly doubled what was folded into single slices — 9 roles × 9 actions with a
> permission-explanation engine (was inside slice 1), a full AI governance substrate with 10
> prompt families and §27.1 test coverage (was inside slice 2), and a new vector-retrieval stage
> (wasn't accounted for at all). The revision below splits along those seams and removes calendar
> integration from the critical path now that [C-04](#c-04--calendar-and-candidate-email-deferred-to-the-final-phase)
> deferred it.
>
> ```
> S1  Identity & Access Foundation
>      └─ S2  Authorization & Admin Cockpit
>              ├─ S3  Workflow & Notification Core ──┐
>              └─ S4  AI Platform & Governance ──────┤
>                                                    ▼
>                                       S5  Job Descriptions & Postings
>                                                    │
>                                                    ▼
>                                       S6  Candidate Intake & Database
>                                                    │
>                                                    ▼
>                                       S7  Vector Retrieval
>                                                    │
>                                                    ▼
>                                       S8  AI Ranking & Insights   ★ money demo
>                                                    │
>                                                    ▼
>                                       S9  Shortlist & Interview Console
>                                                    │
>                                     ┌──────────────┼──────────────┐
>                                     ▼              ▼              ▼
>                             S10 Scorecard    S14 Calendar &   (S13 draws
>                                 Center           Candidate      from S5–S11)
>                                     │            Comms
>                                     ▼
>                             S11 Selection, Offer & Closure
>                                     │
>                                     ▼
>                             S12 Resurfacing & Priority Lane
>
>                             S13 Insight, Reporting & Export
> ```
>
> | # | Slice | Contents | Size |
> |---|-------|----------|------|
> | **S1** | Identity & Access Foundation | Hubble REST login, session create/refresh/expire, **forced revocation** ([G-11](#g-11--forced-session-revocation)), user activation + access-denied enforcement, `users`/`roles` tables, audit-log substrate | M |
> | **S2** | Authorization & Admin Cockpit | 9 roles × 9 action flags, `grant/deny/unset` cells, **deny-overrides-grant** (`AUTHZ-005`), **permission explanation endpoint** (`AUTHZ-006`), per-user overrides with mandatory reason, matrix UI, permission audit, `ENG-010` seed scripts | **L** |
> | **S3** | Workflow & Notification Core | Workflow Service transition framework (validation, reason enforcement, actor attribution incl. **AI Service Account**), task model, in-app + email notifications. No domain state machines — those ship with their features | M |
> | **S4** | AI Platform & Governance | AI gateway abstraction, prompt template registry (10 families scaffolded), per-family model config, `ai_run_logs`, safety flags, output feedback, **§27.1 test corpus + CI harness**, graceful degradation ([G-12](#g-12--graceful-degradation)) | **L** |
> | **S5** | Job Descriptions & Postings | JD workspace: **PM drafts → RM approves** ([C-12](#c-12--who-drafts-the-jd)), versioning, fork-on-edit; posting management: finite/evergreen, `vacancy_slots`, **compliance-text gate** ([G-09](#g-09--compliance-text-status)), **posting text separate from JD** ([G-08](#g-08--posting-text-separate-from-internal-jd)) | **L** |
> | **S6** | Candidate Intake & Database | Upload validation, malware scan + quarantine, **hybrid parse** ([D18](#d18--resume-parsing-approach)), OCR-readiness, **extraction confidence** ([G-05](#g-05--extraction-confidence)), **file-hash dedup** ([G-04](#g-04--resume-file-hash)) + review queue, resume versions, candidate timeline, **`source_type`** ([G-10](#g-10--application-source-type)) | **L** |
> | **S7** | Vector Retrieval | Embedding pipeline, `DOC-008` chunking, `vector_index_records`, **query-time authorization** ([Interaction A](#interaction-a--permission-changes-now-trigger-vector-re-indexing)), re-index path, model pinning | M |
> | **S8** | AI Ranking & Insights | Retrieve→rerank, `candidate_ranking`/`fitment_summary`/`gap_summary`, **mandatory criteria status** ([G-01](#g-01--mandatory-criteria-status)), **evidence references** ([G-02](#g-02--evidence-references-and-source-labelling)), **insufficiency outputs** ([G-03](#g-03--insufficiency-outputs)), **ranking versions** ([G-07](#g-07--ranking-versions)), **override records** ([G-06](#g-06--human-override-as-a-first-class-record)), ranking board UI | **L** |
> | **S9** | Shortlist & Interview Console | Shortlist + mandatory reasons on every disposition, scheduling task workflow, interview rounds, Console with `INT-006` fixed note fields, note versioning, question sets, **Interviewer scoped UI without score** ([C-10](#c-10--interviewers-and-the-ranking-board)) | **L** |
> | **S10** | Scorecard Center | **Consolidated scorecard per application** ([C-05](#c-05--one-scorecard-per-application)), **fixed dimensions** ([C-06](#c-06--fixed-scorecard-dimensions)), resume/interview evidence separation, draft→approve gate, supersede-on-note-edit | M |
> | **S11** | Selection, Offer & Closure | Priority slots 5×n ranked + displacement, `selection_tag`, offer/onboarding tracker with **joining status** ([G-14](#g-14--joining-status-and-blocker-reason)), Hubble ID + uniqueness, slots `reserved`→`filled`, **closure on onboarded + Hubble ID** ([G-13](#g-13--closure-requires-onboarding-complete--valid-hubble-id)), closure checklist, evergreen lifecycle | **L** |
> | **S12** | Resurfacing & Priority Lane | Event-driven trigger on posting publish, suggestion queue, priority lane, **carry-forward + confirmatory round** ([C-07](#c-07--carry-forward-has-no-basis-in-the-spec)), **configurable staleness windows** ([C-11](#c-11--staleness-windows-fixed-vs-configurable)) | M |
> | **S13** | Insight, Reporting & Export | Practice / Recruitment Manager / candidate timeline / AI audit / SLA aging dashboards, **resurfacing yield**, **AI agreement rate**, export controls | M |
> | **S14** | Calendar & Candidate Communications | Google Calendar API + free/busy, candidate communication templates — **gated on `OD-005`** | M |
>
> **What moved, and why:**
> - **Split — old slice 1 → S1 + S2.** Nine roles × nine actions plus deny-overrides plus an
>   explanation engine plus a matrix UI is not one slice alongside Hubble integration. The split
>   also creates a real milestone at S2: a user can log in, be blocked if unactivated, and an
>   Admin can configure access.
> - **New — S3 Workflow & Notification Core.** `WF-001` requires all transitions through a
>   Workflow Service; `NFR-009` forbids duplicating business rules in UI code. Without this as
>   infrastructure, every feature slice reinvents transition handling. Scoped to the *framework*
>   only — no domain state machines — so it isn't speculative abstraction.
> - **New — S4 AI Platform & Governance, promoted ahead of any AI feature.** Was buried inside
>   old slice 2. With ten prompt families and §27.1 coverage now mandatory, the substrate earns
>   its own slice — and must precede S5, since JD generation is the first LLM call and `AIRun`
>   history cannot be backfilled.
> - **New — S7 Vector Retrieval, isolated.** The newest, riskiest piece (chunking strategy,
>   embedding model choice, query-time authorization). Isolating it means S6 ships a working
>   candidate database whether or not S7 goes smoothly.
> - **Split — old slice 6 → S10, separate from S9.** Scorecards have their own eligibility rule,
>   their own AI family, and their own approval gate. Interview capture and evaluation are
>   different concerns.
> - **Deferred — S14.** No longer blocks anything; the Workspace delegation request has no
>   urgency, though filing it early is still free.
>
> **MVP boundary**, mapped against the reference spec's §34 MVP Definition of Done:
>
> ```
> MVP        S1 … S11  +  S13          ← their Phases 1-3, incl. dashboards
> POST-MVP   S12  Resurfacing          ← their Phase 4
>            S14  Calendar & comms     ← their Phase 5
> ```
>
> Worth flagging: this places **resurfacing post-MVP**, and it's arguably the most distinctive
> feature in the product. If it matters more than that ordering implies, S12 can move ahead of
> S13 — it only depends on S11.
>
> **Cross-cutting, stated once, applied per slice:** §27.1 AI evaluation + security test coverage
> (built in S4, extended in every AI-bearing slice: S5, S6, S8, S9, S10, S12) · graceful
> degradation ([G-12](#g-12--graceful-degradation)) · audit on every material write (`API-003`) ·
> server-side authz on every endpoint (`AUTHZ-003`) · advisory-only AI — no code path where AI
> output auto-rejects, auto-shortlists, or auto-closes.

> ### ✔ ADDED 2026-08-17 — wave/sprint grouping for the backlog
>
> **Directive (via the user's manager):** OpenSpec backlogs should be written in full, with
> tasks grouped into **waves and sprints**, not a flat task list. Saved to memory as a standing
> preference for future TalentSphere backlog artifacts
> (`project_wave_sprint_backlog.md`).
>
> **Wave ≠ slice, deliberately.** A 1:1 mapping (14 waves) would collapse the two-tier hierarchy
> a wave/sprint structure is meant to provide back down to one tier, and would make "wave" a
> reporting unit at sprint-level granularity. Waves instead group slices by **dependency phase**,
> giving each wave a coherent, demoable, one-sentence-describable outcome — sprints (2-week
> increments, see below) become the real execution unit inside a wave.
>
> **Wave boundaries are aligned to the MVP boundary already established above**, not to an
> arbitrary four-way split. An earlier draft grouping mixed MVP and post-MVP slices inside the
> same wave (S11 next to S12); that undermines a wave's purpose as a milestone that means
> something when it's called "done." Corrected:
>
> ```
> Wave 1 — Foundation & Governance        S1 S2 S3 S4
> Wave 2 — Core Hiring Loop        ★demo  S5 S6 S7 S8
> Wave 3 — Funnel, Evaluation, Decision   S9 S10 S11
> Wave 4 — Insight & Reporting            S13
> ──────────────────── MVP complete ────────────────────
> Wave 5 — Resurfacing & Communications   S12 S14
> ```
>
> Waves 1–4 deliver the reference spec's full §34 MVP Definition of Done as one continuous run;
> Wave 5 is cleanly everything post-MVP, with nothing straddling the boundary.
>
> **Sprint length: 2 weeks** (recommended default; no stated org standard yet — confirm before
> the backlog is finalized).
>
> **Illustrative sprint sizing**, from the t-shirt sizes already assigned to each slice — **not a
> commitment**, a placeholder until `tasks.md` produces a real task breakdown with actual
> estimates:
>
> | Wave | Slices | Sizes | ~Sprints |
> |---|---|---|---|
> | 1 | S1 S2 S3 S4 | M L M L | 5–6 |
> | 2 | S5 S6 S7 S8 | L L M L | 6–7 |
> | 3 | S9 S10 S11 | L M L | 4–5 |
> | 4 | S13 | M | 1 |
> | 5 | S12 S14 | M M | 2 |
>
> **CONFIRMED by the user, 2026-08-17.** The five-wave grouping and 2-week sprint length above are
> settled, not provisional. This is the structure the eventual `tasks.md` backlog will follow.

**Consequences (original, still applicable):**
- **The Interviewer role splits across slices.** The role itself, and its scope predicate,
  must be defined in S1/S2's RBAC; its UI surface arrives in S9.
- **Honest tradeoff:** nothing is demoable end-to-end until S9. The mitigation is that the money
  demo — upload a resume, get an AI ranking with fitment and gaps — lands at the end of **S8**,
  the core value proposition, ahead of the heavier funnel work.
- ~~**Structural question for OpenSpec**~~ — **RESOLVED, see the block immediately below.**

> ### ✔ APPROVED 2026-08-17 — one OpenSpec change per wave
>
> **`talentsphere` (this change) stays exploration-only, permanently.** It never gets its own
> `proposal.md`/`design.md`/`specs/`/`tasks.md` — `exploration-notes.md` is its whole and final
> contents, kept as the historical decision record.
>
> **Five sibling changes get created for the actual build, one per wave** — not fourteen (one per
> slice, as originally floated above), and not one giant change covering the whole application:
>
> ```
> openspec/changes/
>   talentsphere/                            ← exploration record only, never proposed
>   talentsphere-wave-1-foundation/          ← S1-S4
>   talentsphere-wave-2-core-loop/           ← S5-S8
>   talentsphere-wave-3-funnel-decision/     ← S9-S11
>   talentsphere-wave-4-insight/             ← S13
>   talentsphere-wave-5-resurfacing-comms/   ← S12, S14
> ```
>
> Each is a complete, independently-lifecycled change — its own proposal, its own design
> decisions, its own delta specs, and its own `tasks.md` doubling as that wave's sprint backlog.
>
> **Why wave-level, not slice-level or one-giant-change:** a single change covering all 14 slices
> can't be marked "done" until the entire application is built, and a `proposal.md` covering all
> 19 features at once is unreviewable as a unit. Fourteen slice-level changes is more fragmented
> than the natural milestones warrant. Wave-level lets each change be reviewed, built, and
> **archived independently** — Wave 1 can be complete and merged into the permanent spec tree
> while Wave 2 is still being proposed — and it directly satisfies the manager's wave/sprint
> backlog directive: each wave-change's `tasks.md` *is* that wave's sprint backlog, not something
> assembled after the fact from a flat list.
>
> **Practical consequence for picking this up in a new session:** a fresh chat running
> `/opsx:propose` should be told explicitly to read this file first and to create
> `talentsphere-wave-1-foundation` (not add artifacts into `talentsphere` itself, and not create
> all five at once — each wave proposes when its turn comes).

> ### ✔ FINAL 2026-08-20 — consolidated backlog, post style/layout reconciliation
>
> Exploration is complete (all references read and reconciled — `reference/spec.md`, the GCP
> landing-zone spec, `reference/design-spec.md`, `reference/layout-spec.md`). This block is the
> single current-state reference for the backlog — everything above this point is the reasoning
> trail for *how* it got here, kept intact rather than deleted.
>
> **Re-audit approach:** rather than re-deriving the backlog from a blank page, every one of the
> 14 slices was checked against everything now known — the style/layout guide, the
> build-fresh-don't-wait-on-Azure direction, and the finalized infra plan. The 14-slice/5-wave
> grouping itself did not need to change; it had already been aligned to the reference spec's own
> MVP boundary through 12 rounds of conflict resolution. What changed: **Wave 1 gains explicit
> UI-foundation scope**, and four later slices gain explicit call-outs of which shared UI pattern
> they consume — both threaded in from [Part S](#part-s--visual-style--layout-guide-reconciliation).
>
> **Sprint length: 1 week** (switched from the earlier 2-week default, per the 2026-08-20
> decision — see the answered question above this block). Sprint counts below are re-derived from
> the same underlying effort estimates, not inflated by the shorter sprint alone — a 2-week
> estimate of "5–6 sprints" becomes "10–12" one-week sprints for the *same* amount of work. Wave 1
> is the one exception: it gained **real new scope** (the design-system foundation), so its count
> increases beyond a mechanical doubling.
>
> ```
> WAVE 1 — Foundation & Governance                                    (11-14 sprints)
>   S1  Identity & Access Foundation        M   Hubble login, sessions, activation
>                                               → renders via the Bare Shell (layout §1);
>                                                 sign-in is the ONLY bare-shell screen in scope
>   S2  Authorization & Admin Cockpit       L   9 roles × 9 actions, permission matrix
>                                               → NEW: design tokens (color/type/icon/shape,
>                                                 design-spec.md) + Authenticated Shell
>                                                 (sidebar, 3 states, layout-spec.md) built here
>                                               → NEW: dense-data-table pattern (design-time gap,
>                                                 resolved here) — the matrix UI is its first use
>   S3  Workflow & Notification Core        M   transition engine, task/notification model
>   S4  AI Platform & Governance            L   prompt registry, AI run logs, eval harness
>
> WAVE 2 — Core Hiring Loop            ★ first demo lands here        (12-14 sprints)
>   S5  Job Descriptions & Postings         L   JD drafting/approval, posting management
>                                               → standard card-grid / detail-view templates
>   S6  Candidate Intake & Database         L   resume upload, parsing, dedup
>                                               → embedding pipeline setup begins (C-01)
>   S7  Vector Retrieval                    M   Vertex AI Vector Search (not pgvector — see the
>                                                 correction under D09/C-01), re-index path
>   S8  AI Ranking & Insights               L   ranking, fitment/gap, mandatory criteria
>                                               → USES: dense-data-table (S2) for the ranking
>                                                 board, evidence-source tags (NEW pattern,
>                                                 resolved here, G-02), AI-disclosure banner
>                                                 (status-surface component, RANK-004/UI-006)
>
> WAVE 3 — Funnel, Evaluation & Decision                              (8-10 sprints)
>   S9  Shortlist & Interview Console       L   shortlisting, interview rounds, notes
>                                               → DECIDE HERE: Presentation/Fullscreen shell
>                                                 (layout §1, state 3) for an active interview —
>                                                 the design-time gap's likeliest home
>   S10 Scorecard Center                    M   consolidated scorecards, approval gate
>                                               → USES: evidence-source tags (S8) for resume vs.
>                                                 interview evidence separation
>   S11 Selection, Offer & Closure          L   priority slots, offers, closure rules
>                                               → USES: Floating Action Panel (layout §6) for
>                                                 Priority Selection — see Part S.1's discovery
>
> WAVE 4 — Insight & Reporting                                        (2 sprints)
>   S13 Insight, Reporting & Export         M   dashboards, audit/export controls
>                                               → USES: dense-data-table (S2) for audit/AI-run logs
>
> ──────────────────────── MVP COMPLETE (Waves 1-4) ────────────────────────
>
> WAVE 5 — Resurfacing & Communications                               (4 sprints)
>   S12 Resurfacing & Priority Lane         M   resurfacing pool, carry-forward
>   S14 Calendar & Candidate Comms          M   Google Calendar, candidate email
> ```
>
> **Illustrative MVP total (Waves 1-4): 33-40 one-week sprints, roughly 7.5-9 months** — slightly
> larger than the pre-restructure estimate (32-38 weeks), because Wave 1 now deliberately carries
> the design-system foundation instead of leaving it to be reinvented ad hoc inside S2, S8, S9,
> S10, and S13 individually. That's a real, small schedule cost taken on in exchange for building
> the shared UI patterns once, correctly, before the slices that depend on them. **Wave 5
> (post-MVP): 4 sprints, about one month.**
>
> **Change structure is unaffected** — still one OpenSpec change per wave
> (`talentsphere-wave-1-foundation` through `talentsphere-wave-5-resurfacing-comms`), per the
> approval immediately above. This block only refines what each wave's `tasks.md` will need to
> cover, not how the changes are split.
>
> **Design-time items resolved by this pass, no longer open:** the dense-data-table pattern now
> has an owner (S2, first use) and consumers (S8, S13). The evidence-source tagging pattern now
> has an owner (S8, first use) and a consumer (S10). **Still genuinely open, deferred to S9:**
> which screen(s) use the Presentation/Fullscreen shell — a candidate is named, not decided.

---

### D14 — Evergreen posting rules

**Question:** Evergreen postings have no vacancy count — what governs selection and closure
for them?

**Options considered:**
1. **Finite with n = unlimited** — same object, unbounded counter, manual close
2. Evergreen with a rolling cap — unlimited total hires, max 5 in active selection at once
3. Separate object entirely — a TalentPool that spins off finite postings
4. Not decided yet

**Decision:** Option 1 — **evergreen is finite with n = unlimited**.

**Why:** Simplest. One state machine, one flag (`vacancy_type: FINITE(n) | EVERGREEN`), one
code path. Pairs well with [D13](#d13--phasing--change-decomposition) — evergreen becomes
nearly free inside slice 7 rather than its own build.

**Consequences:**
- No auto-close for evergreen; **manual pause/close only**. Feature 17 resolves to: finite =
  automatic per rules plus manual; evergreen = manual only.
- **No 5-cap on evergreen** — selection is unbounded. UI implication: the priority-slots
  surface needs a variant that is not five fixed boxes.
- Recommended: a **staleness nudge** (open 240 days, 0 hires → prompt a review) so evergreen
  postings do not become zombies.
- **Reporting consequence:** time-to-fill is a per-posting metric that never resolves for a
  posting that never closes. For evergreen, time-to-fill must be computed **per hire**, not
  per posting. Feature 18 needs to handle both.
- **Open edge case:** can a posting change type mid-life (evergreen → finite, or the reverse)?
  Probably should be blocked; needs an explicit rule.

---

### D15 — Scorecard competencies

**Question:** Where do scorecard competencies come from, and what is the rating scale?

**Options considered:**
1. Global library, per-JD selection — Admin curates, JD picks
2. **AI-suggested from the JD** — AI proposes, PM approves as part of JD approval
3. Fixed template per interview type — same criteria for every role
4. Not decided yet

**Decision:** Option 2 — **AI-suggested from the JD, approved by the PM as part of JD approval**.

**Why:** No library to curate, criteria always fit the role, and — importantly — it folds into
the **existing** JD approval gate from [D12](#d12--job-description-approval) rather than adding
a fifth AI touchpoint with its own approval step.

**Consequences:**
- **Competencies become part of the JD version.** Editing them forks the JD version (per
  [D12](#d12--job-description-approval)), and scorecards pin to the competencies of the JD
  version their posting was published against — consistent with the score reproducibility tuple.
- **Known weakness: cross-role comparability.** If each JD invents its own competency names,
  "Technical Depth" on one role is not comparable to anything on another, which degrades
  feature 18 reporting.
  **Recommended mitigation (not yet decided):** seed the AI with a soft vocabulary so names
  converge naturally, let the PM rename during approval, and track competency-name frequency
  so a library emerges organically and can be formalised later without an upfront curation
  project.
- Recommended: cap suggestions at **5–7 competencies**, or scorecards become homework and
  panelists stop completing them.
- **Rating scale — RESOLVED 2026-08-14 from the reference spec.** **5-point:**
  `strong_yes | yes | maybe | no | strong_no`, applied to both `interview_notes.recommendation`
  and `scorecards.overall_recommendation`. This supersedes the earlier provisional 4-point
  suggestion. Confidence: medium-high — it is baked into their schema in two places.
- **Dimensions remain contested** — the spec fixes them (`SCR-003`) →
  [C-06](#c-06--fixed-scorecard-dimensions). Note `OD-007` means the dimensions are unapproved
  even in their own organisation.

---

### D16 — Admin access to candidate PII

**Question:** Can Admins read candidate PII — resumes, interview notes, scorecards — or only
configure the system?

**Options considered:**
1. **Config only, audited break-glass** — no candidate data by default, explicit logged elevation
2. Full access — Admins see everything, audit log as the only control
3. Config only, hard block — no elevation path at all
4. Not decided yet

**Decision:** Option 1 — **config only, with audited break-glass**.

**Why:** Strong governance posture for a small extra build. Option 2 gives the most privileged
role unrestricted PII access with only after-the-fact detection; option 3 makes real support
work impossible.

**Consequences — one of these is a back door that would otherwise be missed:**
- **Audit logs must be readable without exposing PII.** Audit entries reference candidate IDs
  and event types, not names or resume text. This is a schema constraint on slice 1, not a UI
  filter.
- **`AIRun` records are a PII back door.** Admins own prompt governance, so they will have
  access to AI run logs — which store raw prompt inputs, which contain resume text. Therefore
  `AIRun` must store **input references** (resume_version_id, jd_version_id) with the raw
  payload either redacted, hashed, or gated behind the same break-glass. Without this, the
  config-only boundary is decorative.
- **Export controls (feature 18) follow from this:** Admins are not default exporters.
  Export is a Recruiter/PM capability, audited, with the field set defined.
- Break-glass needs a **notification target** — a second Admin, or security. To decide.
- Out of scope but worth naming: engineer access to production data is a separate infra
  question this decision does not cover.

---

## Round 5 — pipeline mechanics

### D17 — Priority-lane carry-forward

**Question:** Can a priority-lane candidate (previously selected, never offered) skip
re-interviewing for a similar new posting?

**Options considered:**
1. **Carry forward with attestation** — prior scorecards applied to a new posting on explicit, audited human attestation
2. Always re-interview — prior scorecards readable as context only
3. Skip early rounds only — carry the screen, re-run technical and managerial
4. Not decided yet

**Decision:** Option 1 — **carry forward with attestation**.

**Why:** This is the entire practical payoff of tracking selected-not-offered candidates. The
attestation preserves the decision record: a named human states that evidence gathered for a
different posting is applicable to this one, with a reason, audited.

**Consequences:**
- **`Scorecard` cannot be owned by a single application.** It needs to be linkable to multiple
  applications — a link record carrying `carried_forward`, the source application, the
  attesting user, timestamp, and reason.
- **The Application state machine needs a non-round-1 entry point** — a carried-forward
  candidate enters at selection.
- **Direct conflict with feature 13.** "Scorecard Center restricted to interviewed candidates
  only" would block this path, because a carried-forward candidate has not been interviewed
  *for this posting*. The rule must be restated as: **has an interview record for this
  application, OR a carried-forward attestation.** This needs to be written into the spec
  explicitly or the two features will contradict each other in slice 6/7.
- **Governance note:** this is the one place in the system where a hiring decision rests on
  evidence gathered for a different role. The attestation *is* the control, so it must require
  a typed reason, not a checkbox.
- **Open: the role-similarity guard.** What makes two postings similar enough? Recommended
  rule: postings sharing the **same JD** (any version) are auto-eligible; a different JD
  requires a stronger attestation or is disallowed. Not yet decided.
- **Open: priority-lane TTL.** Vetting goes stale. Provisionally 90 days; not yet decided.

---

### D18 — Resume parsing approach

**Question:** How are resumes parsed into structured data?

**Options considered:**
1. LLM structured extraction — Claude with a JSON schema, no vendor
2. Licensed resume parser — Affinda / Textkernel / HireAbility
3. **Hybrid** — deterministic for structural fields, LLM for interpretive fields
4. Not decided yet

**Decision:** Option 3 — **hybrid**.

- **Deterministic** (regex/heuristics on extracted PDF text): email, phone, name, dates, links
- **LLM** (a governed `AIRun`): skills, seniority, role summaries, domain experience

**Why:** Dedup keys come from deterministic logic rather than model output. Candidate identity
should not depend on a model's mood — this is the right reason to accept the extra code.

**Consequences:**
- **Two independent failure modes, handled separately:**
  - Deterministic extraction fails → contact fields empty → **manual entry required before the
    candidate record can be created**, because dedup depends on those fields.
  - LLM enrichment fails → the candidate still exists, just unenriched → **retryable**, no
    blocking.
  - This usefully decouples the pipeline: a candidate can exist on deterministic fields alone.
- Requires a **PDF text-extraction library**, chosen with the stack ([D09](#d09--tech-stack)).
- Adds a fourth governed `AIRun` type (JD drafting, ranking, scorecard drafting, enrichment).
- **Prompt injection:** resume text is untrusted input. It must be delimited and treated as
  data, never as instructions. Note that the **larger injection surface is the ranking stage**,
  not parsing — ranking reads the full resume text with far more consequential output.

---

### D19 — Offer tracker scope

**Question:** Is the Offer & Onboarding Tracker (feature 16) a status tracker or a full offer
workflow?

**Options considered:**
1. **Status tracker only** — states, dates, Hubble ID
2. Tracker + comp and approvals
3. Full workflow including letter generation and e-signature
4. Not decided yet

**Decision:** Option 1 — **status tracker only**.

**Why:** Closes the loop without absorbing an offer-management product. Letter generation,
approvals, and negotiation stay outside TalentSphere.

**Consequences:**
- Fields: `status (EXTENDED | ACCEPTED | DECLINED | RENEGED)`, `extended_on`, `responded_on`,
  `start_date`, `hubble_id` (per [D08](#d08--hubble-id-at-onboarding)), notes.
- **No compensation data**, which means no extra access tier on top of
  [D16](#d16--admin-access-to-candidate-pii). A meaningful simplification.
- The statuses are sufficient to drive closure rules — `offers_accepted == vacancy_count`
  triggers auto-close, and `RENEGED` triggers the audited reopen path.
- Confirms the corresponding non-goal.

---

### D20 — Priority-slot mechanics

**Question:** How does the cap arithmetic work, and can slot holders be displaced?

**Options considered:**
1. **5 per vacancy, ranked, displacement with reason**
2. 5 per posting, ranked, no displacement
3. 5 per vacancy, unordered set
4. Not decided yet

**Decision:** Option 1 — **5 slots per vacancy, ranked in preference order, displacement
allowed with a mandatory reason and audit entry**.

**Why:** Matches the original "up to 5 per finite vacancy" wording, and displacement-with-reason
mirrors the mandatory-reason pattern used elsewhere in the system.

**Consequences:**
- A 3-vacancy posting has **15 ranked slots**.
- Displacement and insertion require **re-sequencing ranks** — a reorder operation that must
  itself be audited, not a silent renumber.
- **Vacancy count reduction** (3 → 1) shrinks 15 slots to 5; the 10 evicted candidates require
  a reason and flow into the **priority lane** — which is consistent, since "selected but not
  offered" is exactly what they are.
- **Evergreen postings have no slots at all** ([D14](#d14--evergreen-posting-rules)): selection
  there is a flat unranked list. The UI needs both presentations.
- **UX concern (not a blocker):** ranking 15 people in strict preference order may be more
  precision than a PM can genuinely supply. Tiered ranking within the slot set may be worth
  considering during design.

---

## Round 6 — eligibility, staleness, identity, immutability

### D21 — Carry-forward eligibility

**Question:** What makes two postings "similar enough" for scorecard carry-forward
([D17](#d17--priority-lane-carry-forward))?

**Options considered:**
1. Same JD only — carry-forward restricted to postings sharing a Job Description
2. **Same JD auto; different JD requires stronger PM attestation**
3. AI-assessed JD similarity above a threshold
4. Not decided yet

**Decision:** Option 2 — **same-JD carry-forward is the fast path; similar-but-different JDs go
through PM attestation as the exception.**

**Why:** Same-JD reopens are expected to be the common case, and competencies match by
construction there ([D15](#d15--scorecard-competencies) ties competencies to the JD version).
Option 3 was rejected on principle: it would put AI in the position of **gating** a decision
about a person rather than advising one, which cuts against the advisory-only rule established
for ranking.

**Consequences:**
- Two paths with different friction: same JD = fast, different JD = PM sign-off with an explicit
  competency-difference acknowledgement.
- **Open point to resolve:** [D17](#d17--priority-lane-carry-forward) established that "the
  attestation *is* the control" and must be a typed reason, not a checkbox. If the same-JD path
  becomes fully automatic with no recorded human step, that control disappears for the common
  case. Recommended reconciliation: same-JD still records a lightweight attestation (one click
  plus a short reason), while different-JD requires the fuller justification.
- See [OA-01](#oa-01--carry-forward-volume-distribution) for the assumption this rests on.

---

### D22 — Staleness windows

**Question:** How long does vetting stay valid — priority-lane TTL and resurfacing recency
window?

**Options considered:**
1. **90 days priority lane / 12 months resurfacing**
2. 180 days / 24 months
3. Configurable in the Admin Cockpit
4. Not decided yet

**Decision:** Option 1 — **90-day priority-lane TTL, 12-month resurfacing recency window**.

**Why:** Tight enough that carried-forward vetting is still meaningful, loose enough to be
useful. A six-month-old interview carried into a live selection decision is a stretch.

**Consequences:**
- After 90 days, a candidate still resurfaces but as a **standard match** — prior scorecards
  become readable context rather than carry-forward-eligible evidence. The lane does not delete
  people; it demotes them.
- Resumes older than 12 months drop out of the resurfacing pool unless a **new resume version**
  arrives, which resets the clock.
- These are fixed values, not Admin-configurable. If they are ever made configurable, changes
  must be **audited** — they alter eligibility for a decision shortcut.

---

### D23 — Candidate deduplication

**Question:** How does candidate deduplication work — match keys, and what happens on a probable
match?

**Decision (revised from the options as originally framed):** the fuzzy signals must be
**independent**, not secondary checks hanging off email. Final rule:

| Signal | Action |
|--------|--------|
| **Resume file hash** matches, **same** candidate | **No new resume version** — link to existing *(added [G-04](#g-04--resume-file-hash))* |
| **Resume file hash** matches, **different** candidate | **REVIEW QUEUE** *(added [G-04](#g-04--resume-file-hash))* |
| **Exact email match** | **AUTO-MERGE** |
| **Name + phone match**, emails differ | **REVIEW QUEUE** |
| **Name + employer + role match**, email *and* phone differ | **REVIEW QUEUE** |
| No match on any signal | New candidate |

**Note on the file hash:** it is a **document-level** signal, not a person-level one. An identical
file proves two *documents* are the same, not that two *candidate records* are the same person —
the failure case being a recruiter uploading candidate A's PDF under candidate B's name. So a
cross-candidate hash match raises a review item rather than auto-merging people.

**Why the revision:** as originally framed, the fuzzy checks were tied to email and would have
missed a common real case — the same candidate applying with a personal email once and a work
email another time, with no other shared identifier surfacing. Each signal above can now raise a
review item on its own.

**Consequences:**
- Merge is **reversible and audited** in all cases.
- A "not a duplicate" verdict must be **remembered** (suppression list) so the same pair does not
  re-queue on every submission.
- Match inputs come from the **deterministic** half of parsing
  ([D18](#d18--resume-parsing-approach)) — identity does not depend on model output.
- Still needs a rule: merging two candidates who are **live in different pipelines** — which
  applications survive, and what happens to their separate ranking scores.
- **Backstop for missed matches — see [D23a](#d23a--duplicate-backstop-at-posting-level) below.**

---

### D23a — Duplicate backstop at posting level

**Question raised:** if the initial dedup check misses entirely and the same person ends up as
two candidate records, both submitted against the same posting — should the AI Ranker or a
Recruiter get a possible-duplicate warning at ranking time as a fallback?

**Status:** recommendation made, **not yet confirmed**.

**Why this matters more than it first appears:** two duplicate records both ranked against one
posting means the same person can occupy **two of the five priority slots**
([D20](#d20--priority-slot-mechanics)). That does not just clutter a list — it corrupts the cap
arithmetic and wastes a vacancy's worth of selection capacity.

**Recommendation: two checks, at different depths.**

1. **Submission-time check** (deterministic fields, cheap) — runs on every submission against the
   whole candidate database. This is [D23](#d23--candidate-deduplication) as decided.
2. **Posting-scoped content check** (richer, fallback) — when ranking a posting, compare the
   candidates *within that posting* for near-duplicates using the **parsed** signals now
   available (employer history, education, dates, skill fingerprint). Scope keeps it cheap: a
   posting holds tens of candidates, not thousands, so pairwise comparison is trivial at
   [D07](#d07--target-volume) volumes.

Surfaced as a **non-blocking warning on the ranked list** ("possible duplicate of #7"), actioned
by the Recruiter, never auto-merged at this stage — consistent with the advisory-only principle.

**Follow-on rule needed:** when a post-submission duplicate is confirmed and merged, the **two
applications against the same posting must also merge**. Which application survives, which
ranking score is retained (or is a re-rank forced), and what happens if the two sat at different
stages, all need explicit rules.

---

### D24 — Interview note locking

**Question:** Who can edit interview notes, and when do they become immutable?

**Options considered:**
1. **Author-only, locked on scorecard approval**
2. Author edits, Recruiter may amend with attribution
3. Locked on submit, before the scorecard is drafted
4. Not decided yet

**Decision:** Option 1 — **only the authoring panelist may edit; all notes for a round freeze
permanently once that round's scorecard is approved.**

**Why:** Clean ownership and a defensible record — the notes are the panelist's account and
nobody else's. Freezing on scorecard approval gives a grace period for corrections while
guaranteeing the approved evaluation and its source evidence agree.

**Consequences:**
- Recruiters and PMs are **read-only on notes, always**.
- Post-freeze corrections require an **appended addendum**, not an edit; the original version is
  preserved.
- Versioning runs from draft through submission; every version is retained.
- Satisfies the [D11](#d11--scorecard-structure) requirement that notes lock on scorecard
  approval.
- Interaction to note: the AI drafts the scorecard from notes that are still editable, so a
  panelist could edit notes *after* the AI draft is generated but *before* approving it. The
  approved scorecard must therefore record which **note version** it was drafted from.

---

## Open assumptions to revisit

Assumptions the current design rests on, with the signal that should trigger a re-examination.

### OA-01 — Carry-forward volume distribution

**Assumption:** exact-same-JD reopens will be the **common case** for scorecard carry-forward,
with the PM attestation path for similar-but-different JDs being the **exception**
([D21](#d21--carry-forward-eligibility)).

**Risk if wrong:** if roles rarely reopen under the literal same JD, and most carry-forward
candidates actually land on new-but-similar JDs, then the PM attestation step stops being an
edge case and becomes a **routine bottleneck** in the priority-lane flow — undermining the
speed benefit that justified [D17](#d17--priority-lane-carry-forward) in the first place.

**Revisit trigger:** once there is real usage data, compare attestation-path volume against
same-JD auto-carry-forward volume. If attestation volume is higher, revisit:
- whether the **attestation UX needs to be faster** — e.g. a saved list of "commonly compared"
  JD pairs so repeat comparisons are one step rather than a fresh justification each time; and/or
- whether the **similar-JD threshold for requiring PM sign-off** needs to be reconsidered.

**Instrumentation implied:** the carry-forward path must record which route was taken (same-JD
vs. attestation) so this comparison is possible at all. That is a slice 7 reporting requirement,
not something to add later.

---

## Recommendations made, not yet formally decided

These were proposed and not objected to, but have not been through an explicit decision:

| Topic | Recommendation |
|-------|----------------|
| **Pipeline spine** | Introduce an **`Application`** entity (candidate × posting). Stage cannot live on `Candidate` because one candidate is at different stages for different postings simultaneously. Everything downstream hangs off it. |
| **Score reproducibility** | `RankingScore` stores the tuple `(resume_version, jd_version, prompt_template_version, model_version)`. Without it, feature 19 is unimplementable retroactively. |
| **Suggestions ≠ applications** | Resurfacing produces a lightweight `MatchSuggestion` that a human **promotes** into a real `Application`. Otherwise pipelines fill with candidates nobody submitted and funnel metrics become meaningless. |
| **AI approval gates** | Gate where AI output becomes an **authoritative record** (JD, scorecard). No gate on **decision support** (ranking, fitment, gaps, questions) — instead enforce advisory-only by construction (AI may never auto-reject, auto-shortlist, or auto-close) plus a logged human disposition. Approving a score would produce rubber-stamping, which manufactures false accountability. |
| **Mandatory reason on rejection** | Feature 10 specifies mandatory reason on *shortlisting*. The reason that matters legally and for reporting is the one on **rejection**. Make reason mandatory on every disposition, from an Admin-maintained code list plus optional free text. |
| **Closure semantics** | Auto-close finite postings on `offers_accepted == vacancy_count`, with an intermediate `FILLING` state and an audited **reopen** path for reneges. Closure must **cascade**: cancel pending interview tasks, notify panelists, move slot holders to the priority lane, move bench to the resurfacing pool. |
| **Practice as data** | Keep `Practice` as an attribute for filtering/reporting even though it is not enforced in permissions ([D02](#d02--data-scoping)). |
| **Calendar sync direction** | One-way: TalentSphere is source of truth, calendar is a projection. |
| **Interviewer provisioning** | Assignment triggers provisioning; auto-dormant after N days idle. |
| **Signature dashboards** | **Resurfacing yield** (% of hires from the resurfacing pool — the number that proves feature 9 was worth building) and **AI agreement rate** (how often humans shortlist the AI's top picks; how often approved scorecards diverge from AI drafts). |

---

## Recommended non-goals

To state explicitly in the proposal so nobody assumes them in:

- Candidate portal, self-apply, careers page, job-board integration — ✅ reference spec §4.2 agrees
- OCR **implementation** — feature 6 says "OCR *readiness*"; readiness only
- ~~Vector store, embedding pipeline~~ — **reversed** by [C-01](#c-01--vector-store): vector
  retrieval adopted, backend later settled as **Vertex AI Vector Search** (see the note under
  [D09](#d09--tech-stack)), not the pgvector originally recommended in the C-01 resolution.
  Batch resurfacing sweeps remain out (event-driven per D07)
- ~~Offline eval harness~~ — **reversed** by [C-09](#c-09--adversarial-ai-testing-as-a-requirement):
  §27.1 AI Evaluation Tests and Security Tests adopted in full
- Prompt A/B testing, fine-tuning, full model registry — **still a separate initiative**
- Statistical bias-audit tooling and disparate-impact analysis — ✅ **survives**, reinforced by
  `PRV-006` and `OD-010`: fairness measurement needs demographic data that cannot be collected
  without legal sign-off. Note this is a *different* non-goal from the eval harness above, which
  was reversed.
- Two-way calendar sync; inbound email processing
- `.ics` calendar attachments — considered and **not adopted** ([C-04](#c-04--calendar-and-candidate-email-deferred-to-the-final-phase))
- Hubble write/provisioning — ✅ **confirmed** by reference spec §4.2 #10 (*"HRIS employee master
  creation"* out of scope); see [D08](#d08--hubble-id-at-onboarding)
- Offer letter generation, e-signature, comp amounts, and approval chains — confirmed by
  [D19](#d19--offer-tracker-scope) and ✅ by the reference schema, which contains no salary amounts
- Licensed resume-parsing vendor — confirmed by [D18](#d18--resume-parsing-approach)
- ATS replacement, payroll processing, background-check automation — ✅ reference spec §4.2

---

## Open questions

**Blocking the proposal:**
- None currently outstanding.

**Resolved, no longer blocking:**
- ~~**Reviewing the manager-provided layout and style guide**~~ — **RESOLVED 2026-08-20.** Both
  documents read in full and reconciled — see
  [Part S](#part-s--visual-style--layout-guide-reconciliation). ~~No conflicts found; two patterns
  flagged not-applicable, three genuine gaps carried forward as design-time items below.~~
  **Amended 2026-08-25:** two patterns flagged not-applicable and **five** genuine gaps — the
  original three plus two found during `design-system`'s propose conversation, and one of them *is*
  a conflict, internal to the style guide (§1.4 against §1.5). See
  [S.6](#s6--two-things-this-reconciliation-missed-found-during-design-systems-propose-conversation-2026-08-25).
  The "no conflicts found" claim was about the guides versus TalentSphere's own decisions, which
  still holds; it was never a claim about either guide's internal consistency, and shouldn't have
  been written as though it were.
- ~~**Reopened 2026-08-17 — the real, currently-deployed Azure TalentSphere repo has not been
  read.**~~ — **CLOSED BY DIRECTION CHANGE, 2026-08-20.** The user's manager has directed that
  TalentSphere **not wait on Azure repo access** — build fresh from the requirements already
  scoped in this document, rather than reconciling against the existing Azure deployment. This is
  a genuinely different resolution than the other two references got: `reference/spec.md` and the
  landing-zone spec were each **read and reconciled**; the Azure repo is instead **descoped by
  explicit direction**, not reconciled. The distinction matters for anyone reading this later —
  there is no guarantee this plan matches the live Azure system, and that is now accepted as
  intentional rather than an open risk to close.
  - The manager has separately provided a **layout and style guide** to inform the build instead.
    This is new input requiring the same discipline as the other two references — read in full,
    then reconciled against whatever UI/UX-relevant decisions already exist in this document
    (currently sparse — mostly the D05/C-10 rule that Interviewers see gaps but not scores) —
    before it can be considered incorporated. Not yet provided to this session.
  - Saved to memory (`project_build_direction.md`) as a standing project fact, since it changes
    how future sessions should treat the Azure repo (a non-goal to actively pursue, not a
    pending dependency).

**Resolved, no longer blocking (as of the prior update — see above for the reopened item):**
- ~~**Wave/sprint grouping for the backlog**~~ — **CONFIRMED by the user, 2026-08-17.** Five-wave
  structure and 2-week sprint length, logged under
  [D13](#d13--phasing--change-decomposition), are settled — this is the structure the eventual
  `tasks.md` backlog will follow.
- ~~**Infrastructure / deployment target**~~ — **RESOLVED 2026-08-17.** GCP with Terraform and
  GitLab CI/CD, following the user's prior GCP AI Landing Zone project (`platform-infra`,
  `gitlab-ci-templates`, `demoapp` on GitLab, plus the full landing-zone spec document — all read
  in full). Vector store backend: **Vertex AI Vector Search** (amends
  [C-01](#c-01--vector-store)'s pgvector recommendation — the landing zone already enables
  `aiplatform.googleapis.com` per app and grants `roles/aiplatform.user`, making it the more
  native choice on this platform). The two sub-items below are resolved by a single fact.
- ~~**QA environment placement**~~ / ~~**Full spec compliance vs. prototype pattern**~~ —
  **RESOLVED 2026-08-17, by the same constraint.** The user's GCP billing account is free-tier,
  capped at **5 billing-linked projects total** (3 currently used: `bootstrap`, `network-dev`,
  `demoapp-dev`). Adding TalentSphere's Dev project brings usage to 4 of 5. Consequences:
  - **Only Local and Dev can be real environments for TalentSphere.** QA, UAT, and Prod get
    Terraform code written the same way `network-uat`/`network-prod` already do in
    `platform-infra` — defined and validatable, never applied with billing — until the account's
    quota is raised. This is not a design choice; a real QA/UAT/Prod project for TalentSphere is
    currently infeasible on this billing account regardless of which pattern is chosen.
  - **Matching the prototype pattern is the only pattern that fits**, not merely the cheaper one.
    The full spec's model is one project per app per environment (§6.4) — two apps × three
    environments alone is 6 app projects, before bootstrap or network projects are even counted.
  - Full detail and the billing math: saved to memory as
    `project_gcp_billing_constraint.md`, since this governs every future GCP project-count
    decision, not just this one.
  - Not yet confirmed: whether the billing account's project quota can be raised (Cloud Billing
    support, sometimes automatic once trial status ends and a payment method is verified) — the
    plan does not assume this is available.
- ~~[D09](#d09--tech-stack) — tech stack~~ — **RESOLVED 2026-08-17:** FastAPI/Python + React/TS +
  PostgreSQL. `openspec/config.yaml` `context:` can now be filled in.
- ~~[D08](#d08--hubble-id-at-onboarding) — Hubble ID direction~~ — **RESOLVED 2026-08-14:** manual
  capture, provisioning ruled out. Validation endpoint existence remains `OD-002`, tracked below
  under external confirmations, but no longer blocks proposing.
- ~~**Scorecard rating scale**~~ — **RESOLVED 2026-08-14** by [D15](#d15--scorecard-competencies):
  5-point `strong_yes…strong_no`, confirmed unchanged by
  [C-06](#c-06--fixed-scorecard-dimensions).

**Left over from answered decisions:**
- ~~**Change decomposition structure**~~ — **APPROVED 2026-08-17.** `talentsphere` stays
  exploration-only; five sibling changes, one per wave, carry the actual proposals — see the
  block under [D13](#d13--phasing--change-decomposition).
- **Break-glass notification target** — who gets told when an Admin elevates
  ([D16](#d16--admin-access-to-candidate-pii)).
- ~~**Competency vocabulary seeding**~~ — **closed** by
  [C-06](#c-06--fixed-scorecard-dimensions): dimensions are fixed and global.
- ~~**Note templates per interview type**~~ — **closed** by `INT-006`: one fixed field set across
  all interview types.
- **Posting type changes** — can a posting switch between finite and evergreen mid-life
  ([D14](#d14--evergreen-posting-rules))?

**Raised by the 2026-08-20 style/layout guide reconciliation — resolved by the backlog
consolidation, same date:**
- ~~**Dense-data-table pattern**~~ — **RESOLVED.** Owned by S2 (Admin permission matrix, first
  use); consumed by S8 and S13. See the
  [✔ FINAL 2026-08-20 backlog block](#d13--phasing--change-decomposition).
- ~~**Evidence-source tagging pattern**~~ — **RESOLVED.** Owned by S8 (AI Ranking Board, first
  use); consumed by S10. Same block.
- **Presentation/fullscreen shell** — **still genuinely open.** A candidate screen is named (the
  Interview Console during an active interview, owned by S9) but not decided — deferred to that
  slice's own design work, not resolved here.

**Raised 2026-08-20 — AI output conciseness ([S.5](#s5--ai-output-conciseness--visual-craft-standard-added-2026-08-20)):**
- **Per-capability length/conciseness bounds** — the principle (concise, precise, no long-form AI
  text) is settled; exact bounds per prompt family (sentence counts, word caps) are not, and
  shouldn't be invented here — deferred to S4's own prompt-design work.

**Raised by the 2026-08-14 conflict resolutions:**
- **⚠ Scorecard drift after note edit** — with [C-08](#c-08--note-versioning-without-a-lock)
  adopting versioning without a lock, a note can change after the consolidated scorecard drawn
  from it was approved. Recommended: wire note-version-after-approval to
  `scorecards.status = superseded`, requiring re-approval. Not yet decided.
  **— RESOLVED 2026-08-26 by `changes/interview-pipeline/design.md` D3, which adopts exactly this
  recommendation: `TS-BL-061` implements supersede-and-re-approve, scoped by the note-version set
  `TS-BL-060` records on each draft. "Not yet decided" above describes the state before that
  conversation and is retained rather than rewritten.**
- **Ten prompt template families to govern** (§16.3) — `job_description_generation`,
  `job_posting_generation`, `resume_extraction`, `candidate_ranking`, `fitment_summary`,
  `gap_summary`, `interview_questions`, `interview_note_summary`, `scorecard_generation`,
  `resurfacing`. Materially more than the four AI touchpoints assumed earlier in this session;
  each needs versioning, an output contract, and adversarial testing. A slice-2 scope item.
- **`innovation practice relevance`** as a scorecard dimension is Miracle-Labs-specific — confirm
  it applies here or rename it.
- **Seed `Edit` on interview notes to Interviewer only** — now a matrix decision per
  [C-08](#c-08--note-versioning-without-a-lock).
- **⚠ Seeded permission-matrix values** — with configurable reads
  ([C-03](#c-03--recruiters-scoped-to-assigned-postings)) and a least-privilege default
  (`AUTHZ-008`), the seeded matrix determines whether cross-pool resurfacing works on day one.
  A decision, not an implementation detail. See
  [Interaction B](#interaction-b--the-seeded-matrix-is-a-decision-not-a-default).
- **Embedding model choice and versioning** — required by
  [C-01](#c-01--vector-store); `embedding_model` is pinned per vector record and `VEC-005`
  requires a re-index path when it changes.
- **Chunking strategy** for `DOC-008` traceable resume sections.
- **Semantic search over interview history** — unlocked by embedding interview notes and
  scorecard summaries (`VEC-001`), but **not among the original 19 features**. Confirm whether it
  is in scope as a user-facing capability or merely a retrieval substrate.
- ~~Revised slice plan~~ — **RESOLVED 2026-08-17.** [D13](#d13--phasing--change-decomposition)
  now specifies fourteen slices (S1–S14); calendar and candidate communications moved to S14.

**Raised but not yet put to a decision:**
- **Duplicate backstop** ([D23a](#d23a--duplicate-backstop-at-posting-level)) — recommendation
  made (submission-time check plus a posting-scoped content check surfaced as a non-blocking
  warning), awaiting confirmation
- ~~**Same-JD attestation reconciliation**~~ — **closed** by
  [C-07](#c-07--carry-forward-has-no-basis-in-the-spec): a `carried_forward` record with attesting
  user, timestamp, and typed reason is written on **every** carry-forward. Same-JD is lighter
  justification, never zero.
- **Merging candidates live in different pipelines** — which applications survive, which ranking
  scores are retained or re-run, and what happens when the two sit at different stages
- "Do not resurface" flag — candidate declined, withdrew, or was blacklisted
- **Prompt injection in resume text** — a real current attack against AI screeners; cheap to
  mitigate by design, awkward to bolt on. Largest surface is the **ranking** stage, not parsing
  ([D18](#d18--resume-parsing-approach))
- Malware scanning tool; parse-failure and quarantine handling paths
- Confidential/executive requisitions needing restricted visibility despite [D02](#d02--data-scoping)
- Candidate-facing sending address (shared mailbox vs. individual recruiter)
- Jurisdictions in which candidates will be ranked — affects whether bias-audit, candidate
  notice, or alternative-process obligations apply (EU AI Act high-risk classification, NYC
  Local Law 144, and adjacent state rules). Not legal advice; a question to route to counsel
  before the proposal is finalized. Partially addressed by
  [G-09](#g-09--compliance-text-status) and `OD-005`.

---

# Part R — Reference Specification Reconciliation

*Added 2026-08-14, after `reference/spec.md` arrived. Decisions D01–D24 above are unchanged
except where explicitly marked resolved (D08, D09, D15 rating scale, D23 file hash).*

## R.0 — What the reference specification actually is

**Source:** `reference/spec.md` — *TalentSphere Software Implementation Requirements
Specification v1.0*, Miracle Software Systems Architecture Team, source basis dated 2026-06-20.

**It is a forward-looking implementation requirements draft, not a record of a deployed system.**

| Signal | Evidence |
|---|---|
| Status line | *"Implementation-ready **draft**"* |
| Purpose §1 | *"**converts** the TalentSphere product and governance requirements **into** a software implementation requirements specification"* |
| Stack §6 | *"The final stack **shall be confirmed** by the engineering leadership team. The following stack is **recommended**"* |
| §33–34 | Implementation *Phases* and an *MVP Definition of Done* — forward-looking |
| §35 | **Ten unresolved open decisions**, including the Hubble login contract and the AI model provider |
| Language | `shall` throughout — requirements language, not `is`/`does` |

**No Azure content anywhere.** The only named cloud services are **Cloud Tasks** and **Vertex AI
Vector Search** — both GCP. There is no Azure lock-in to migrate away from; the recommended stack
already leans in the GCP direction.

**Consequence for how to use it:** where the reference spec conflicts with a decision above,
neither document has field data behind it. Both are plans. Conflicts are recorded in
[R.3](#r3--open-conflicts) for individual decision and are **never silently applied**.

**Encouraging cross-check:** the spec's ten open decisions (§35) overlap heavily with the opens
identified independently in this session — Hubble contract, Hubble ID validation, retention
period, AI disclosure by jurisdiction, calendar/email integration scope, scorecard dimensions,
fairness monitoring approach.

## R.1 — Validated decisions

Same conclusion reached independently, in different words.

| Decision | Reference confirmation |
|---|---|
| [D03](#d03--where-the-5-cap-binds) + [D20](#d20--priority-slot-mechanics) | `BR-012` — *"the maximum number of priority-selected candidates equals vacancy count multiplied by five"*; `SEL-002` unique priority rank; `priority_selections.reason` mandatory; `status: active/removed/superseded` (displacement); `SEL-003` duplicate active selection blocked. Shortlisting uncapped there too. |
| [D14](#d14--evergreen-posting-rules) | `BR-014`, `CLS-005` — evergreen does not close by fulfillment, uses pause/reopen/manual close with reason. `JOB-004` — evergreen *"shall not create fixed vacancy slots."* |
| [D12](#d12--job-description-approval) | `job_description_versions`, `current_version_id`, `job_postings.job_description_version_id` — postings pin to a **version**. `JOB-006` requires comparing AI-generated against human-edited versions. Validates fork-on-edit. |
| [D16](#d16--admin-access-to-candidate-pii) | **Strongest validation in the document.** `ai_run_logs.input_refs_json` is annotated *"References to source objects, **not raw sensitive data**"* — exactly the AIRun back-door fix. Reinforced by `CAN-008`, `AI-006`, `API-006`, and encrypted `primary_email_encrypted`/`primary_phone_encrypted`. |
| [D18](#d18--resume-parsing-approach) | `DOC-006` lists the same extraction targets. `AI-012`/`AI-013`/`SEC-014` require separating trusted instructions from untrusted document content — confirms the prompt-injection stance. |
| [D19](#d19--offer-tracker-scope) | Confirmed by schema and omission — **no salary amounts anywhere** (only a `salary_status` enum), no letter generation, no e-signature. §4.2 excludes payroll and background checks. |
| [D01](#d01--interviewer-role) | `INT-003` — *"Interviewers shall see only assigned interviews"* — the assignment-scoped predicate exactly. |
| [D13](#d13--phasing--change-decomposition) | §33's five coarser phases share the same relative ordering; foundation first, resurfacing late. `AI-002` and MVP DoD #10 require AI run logging from the start, agreeing with the slice-2 placement of the RAI substrate. |
| **Application spine** | `candidate_posting_applications` is precisely the `Application` entity recommended in this session. |
| **AI approval gates** | §2, `AI-010`, `BR-007` — *"shall never automatically reject, shortlist, select, offer, hire, onboard, or close ... purely through AI output."* |

## R.2 — Adopted items

Fourteen items from the reference spec taken into scope. Ten were additive with no behaviour
change; four were decided explicitly on 2026-08-14 and are marked.

### G-01 — Mandatory criteria status
`candidate_posting_applications.mandatory_criteria_status`:
`meets | does_not_meet | unclear | manager_review_required`.

A structured **hard-requirements gate separate from the AI fit score**. A single score conflates
*"excellent fit but lacks the mandatory clearance"* with *"mediocre fit, meets everything"* — those
need opposite handling. It also makes the priority lane safe: `BR-015` grants
selected-not-offered candidates precedence *subject to mandatory criteria*, so a pre-vetted
candidate cannot surface for a role they are categorically ineligible for.

**DECIDED — AI proposes, human confirms any block.** The AI may output only
`meets | unclear | manager_review_required`. It may **never** assert `does_not_meet`; only a human
may set a blocking value, with a reason, audited. This keeps the advisory-only principle intact —
otherwise the AI would be gating a decision about a person.

### G-02 — Evidence references and source labelling
`BR-008`, `UI-005`, `AI-007`, `VEC-008`. Every AI claim labels whether its evidence came from
**resume, interview note, scorecard, or human decision**, and carries references to the source;
the UI visibly separates them. Major explainability capability absent from this session's
thinking. It also makes the [D05](#d05--ai-context-visible-to-interviewers) mitigation real — an
interviewer can see which claims rest on the resume versus an earlier interview.

### G-03 — Insufficiency outputs
`AI-008` — AI returns *insufficiency reasons* rather than guessing when evidence is missing;
`DOC-009` marks low-confidence extraction with insufficient-information markers. Cheap, and it
attacks the failure mode most likely to erode trust in ranking.

### G-04 — Resume file hash
`CAN-004`, `candidate_resumes.file_hash`. Catches the identical-PDF-uploaded-twice case the three
signals in [D23](#d23--candidate-deduplication) miss entirely.

**DECIDED — document-level signal, review if cross-candidate.** Same hash under the same candidate
means the same resume: link to the existing record, do not create a new version. Same hash
pointing at a *different* candidate raises a review-queue item rather than auto-merging, because
that pattern can equally mean a recruiter uploaded the wrong file under the wrong name. Folded
into the D23 table.

### G-05 — Extraction confidence
`candidate_resumes.extraction_confidence` plus `DOC-009`. Drives recruiter review of low-confidence
parses — a third state between "parsed" and "failed" in the
[D18](#d18--resume-parsing-approach) failure model.

### G-06 — Human override as a first-class record
`RANK-006`, `BR-016` — any override of AI ranking, scorecard, or recommendation requires a
mandatory reason and **must not delete the original AI output**. The concrete implementation of
advisory-only: AI output stays on the record, the human decision sits beside it, and the
divergence becomes measurable — which is what makes the **AI agreement rate** dashboard possible.

### G-07 — Ranking versions
`RANK-005` — regenerating rankings creates a new version while preserving prior versions. The
reproducibility tuple identified *what* to record; this adds versioning around it, so a re-rank
after a JD edit or model change does not destroy the earlier board.

### G-08 — Posting text separate from internal JD
`JOB-008`. External posting text is a distinct artifact from the internal job description; this
session conflated them. Gives the AI a second clearly-scoped generation target
(`job_posting_generation`) rather than overloading the JD.

### G-09 — Compliance text status
`job_postings.compliance_text_status: missing | valid | not_required`, with `JOB-009` making
posting-open validation check it. Turns the jurisdictional AI-disclosure exposure into a concrete
field with a gate.

**DECIDED — build the field and the posting-open gate now, defaulting to `not_required`.** Nothing
is blocked while Legal works `OD-005`. When Legal defines the required text, the default flips to
`missing` and the gate begins enforcing — a configuration change, not a rebuild. A hard gate from
day one would block every posting on a timeline outside our control.

### G-10 — Application source type
`candidate_posting_applications.source_type`:
`recruiter | referral | job_board | campus | agency | direct_application | existing_database`.
The `existing_database` value is resurfacing, which makes the **resurfacing-yield metric
computable** rather than inferred.

### G-11 — Forced session revocation
`AUTH-006`/`AUTH-007` — Application Administrators can revoke sessions, and revoked sessions become
invalid immediately for subsequent requests. Addresses the leaver problem raised under
[D08](#d08--hubble-id-at-onboarding). Pairs with ownership reassignment, still open.

> **Updated 2026-08-25:** "still open" stayed open too long — it was never decomposed into any
> backlog item. Closed as `TS-BL-080` in
> [D.10.1](#d101--gap-correction-ownership-reassignment-was-asserted-twice-and-decomposed-never-2026-08-25).
> Revocation itself (`TS-BL-015`) is unaffected and belongs to `identity-and-access`.

### G-12 — Graceful degradation
`DEP-008`, `NFR-004` — outages of the AI provider, OCR, notification service, or vector store must
not prevent viewing records or performing non-AI workflow actions. With four AI touchpoints and an
external calendar, this is a necessary posture.

### G-13 — Closure requires onboarding complete + valid Hubble ID
`BR-013`, `BR-019`, `CLS-001`–`CLS-004`. A finite posting closes as *Filled* only when recruited
candidates with **onboarding complete and valid Hubble IDs** equal the vacancy count, with a
closure checklist surfacing missing IDs and blocking states.

**DECIDED — adopted, with the slot state machine explicit.**

```
vacancy_slot:   open ──accept──▶ reserved ──onboard + Hubble ID──▶ filled
posting:  ... Selection ──▶ Offer ──▶ Onboarding ──▶ Filled
```

This supersedes the earlier working assumption of closing on `offers_accepted == vacancy_count`.

**Why it won on re-discussion:** an initial objection — that postings would sit *open* for weeks
or months — was **wrong**. The spec's posting lifecycle (§11.1) already routes through a distinct
`Onboarding` state, and `vacancy_slots.status` is `open | reserved | filled | cancelled`. So a
posting awaiting joiners is visibly in `Onboarding` with slots `reserved`, not lingering in
`Open`. A dashboard can show *"3 of 3 reserved, awaiting joining"* distinctly from *"still
recruiting."*

| | Close on acceptance | **Close on onboarded + Hubble ID** |
|---|---|---|
| "Filled" means | someone said yes | **someone actually started** |
| Renege / counter-offer | must **reopen a closed posting** | slot returns to `open`; posting was never closed |
| Hubble ID role | captured whenever | **proof of hire**, links to HR records |
| Metric | time-to-accept | time-to-join |
| Cost | phantom fills | needs the reserved/filled distinction |

In an IT services context with notice periods and counter-offers, the renege row is not
hypothetical. The audited **reopen** path remains necessary for reneges discovered after a posting
does reach `Filled`.

### G-14 — Joining status and blocker reason
`offer_onboarding_records.joining_status: pending | confirmed | no_show | delayed` and
`blocker_reason` (required when blocked). Extends [D19](#d19--offer-tracker-scope) without adding
compensation data — the tracker now covers the acceptance-to-joining gap where candidates are most
often lost. Directly supports [G-13](#g-13--closure-requires-onboarding-complete--valid-hubble-id).

### Validation of an existing non-goal
`PRV-006` — *"Demographic or protected-class analytics shall **not** be collected or processed for
fairness monitoring without legal or compliance approval"* — plus `OD-010`. This **supports** the
non-goal that bias-audit tooling is a separate initiative, and adds a reason: fairness measurement
requires demographic data that cannot be collected without sign-off.

## R.3 — Open conflicts

**Not applied. Our decision stands until changed. Each needs an owner decision.**

### C-01 — Vector store
| | Our position ([D07](#d07--target-volume)) | Reference spec |
|---|---|---|
| Retrieval | SQL filter → LLM rank | Embeddings + semantic retrieval |
| Infrastructure | none added | §24 — **"Mandatory for semantic resurfacing"** |
| Scope of embedding | — | resumes, interview notes, scorecards, skill summaries (§17) |
| Extra obligations | — | `VEC-003` auth filters on retrieval; `VEC-005` re-index on model/parse/auth change |

**Tradeoff:** smaller than it looks infrastructurally — `pgvector` is listed first and is a
Postgres extension, near-free on Cloud SQL. Bigger than it looks operationally — `VEC-003` and
`VEC-005` are ongoing costs regardless of backend. The spec also embeds **interview notes and
scorecards**, enabling semantic search across interview history, a capability never scoped here.

> ### ✔ RESOLVED 2026-08-14 — adopt §17 in full (retrieval backend then open)
>
> Vectors **retrieve**, the LLM **ranks**, and the relational database stays authoritative
> (§12: *"the vector store shall never be the only source of truth"*).
>
> **Amends [D07](#d07--target-volume).** The volume assessment (<5k candidates, <30 postings)
> stands; its consequence *"no vector store, no embedding pipeline"* is **superseded**.
>
> **Adds to slice 3 (Intake) / S6 in the current 14-slice plan:** embedding generation job (§25),
> chunking into traceable sections (`DOC-008`), `embedding_model` pinning, `vector_index_records`
> with `authorization_scope_json`.
> **Adds to slice 4 (Matching) / S7-S8 in the current 14-slice plan:** vector kNN + auth filters
> as retrieval stage 1.
> **Ongoing:** the `VEC-005` re-index path.
>
> **⚠ Backend superseded 2026-08-17.** This resolution originally named `pgvector` as the retrieval
> backend, on Cloud SQL. That was revisited during infra planning and **replaced with Vertex AI
> Vector Search** — see the note under [D09](#d09--tech-stack). Everything else in this
> resolution (vectors retrieve / LLM ranks / relational stays authoritative, the embedding
> pipeline, `VEC-003`/`VEC-005` obligations) is unaffected by the backend swap.
>
> **Scope note:** `VEC-001` embeds **interview notes and scorecard summaries**, not just resumes.
> This unlocks semantic search across interview history — a genuine capability that is **not in
> the original 19 features**. Recorded as an intentional scope addition.

### C-02 — Nine roles vs. four
Spec §9.1: System Administrator, Application Administrator, Practice Manager, **Recruitment
Manager**, Recruiter, Interviewer, **Hiring Panel Member**, **Auditor/Compliance Reviewer**,
**AI Service Account**.
- **Auditor/Compliance Reviewer** — read-only on audit and AI logs, structurally unable to modify
  recruitment decisions. A cleaner answer to the [D16](#d16--admin-access-to-candidate-pii) problem
  than Admin break-glass.
- **AI Service Account** — not a human role but a first-class actor
  (`audit_logs.actor_service_account`, `WF-004`). We implied it; they modelled it.
- **Recruitment Manager** — oversight above Recruiter: workload, pipeline, closure readiness.
- **Hiring Panel Member** — distinct from Interviewer; participates in review and comparison.
- **System vs. Application Admin split** — technical configuration vs. app access.

> ### ✔ RESOLVED 2026-08-14 — seed all nine roles
>
> **Amends [D01](#d01--interviewer-role).** The Interviewer role and its assignment-scoped
> predicate survive intact; the role set expands from four to nine.
>
> **Adopted with it — the full enforcement model:** `AUTHZ-001` deny-by-default · `AUTHZ-003`
> server-side on every page *and* endpoint · `AUTHZ-004` direct API calls fail even when the UI
> hides the control · `AUTHZ-005` **direct denial overrides role grants** · `AUTHZ-006` a
> permission **explanation** endpoint · `AUTHZ-007` permission changes audited with previous and
> new value · `ADM-005` matrix cells are **grant / deny / unset** · `ENG-010` seed scripts.
>
> **Nine action flags** (§9.3): View, Create, Edit, Delete, Approve, **Run AI**, Export, Assign,
> Administer. `Run AI` as a distinct permission means *who may trigger AI* is itself controllable.
>
> **Interaction with [D16](#d16--admin-access-to-candidate-pii):** role separation now does most
> of the work that break-glass was carrying. Application Administrator manages access, Auditor
> reads audit and AI run logs, System Administrator handles technical configuration — none of them
> reads candidate PII by default. Break-glass remains only for support cases.
>
> **Cost:** slice 1 grows — nine permission sets across nine actions on every page, plus the
> explanation engine. The engine would have been needed at any role count.

### C-03 — Recruiters scoped to assigned postings
`BR-005`, `CAN-001`, `job_postings.recruiter_ids[]`, `AUTHZ-001` deny-by-default, `AUTHZ-008` least
privilege, `user_permission_overrides` with mandatory reason. Materially tighter than
[D02](#d02--data-scoping)'s global-within-role. Our resolution matches the spec's **Interviewer**
rule but not its **Recruiter** rule. Adopting it means assignment-based scoping on postings — more
than one row-level predicate, still far less than practice-based scoping.

**Key finding on reads:** the spec never states a global-read rule *or* a scoped-read rule.
`CAN-006` — *"searchable **according to retention and access rules**"* — deliberately defers read
scope to the permission matrix. Its answer to "can a recruiter see another recruiter's
candidates?" is *"whatever the Administrator configures"*, with `AUTHZ-008` making the **default
restrictive**.

> ### ✔ RESOLVED 2026-08-14 — configurable read scope, per the spec
>
> **Amends [D02](#d02--data-scoping).** Global-within-role is replaced by: **writes scoped by
> posting assignment** (`BR-005`, `job_postings.recruiter_ids[]`), **reads governed by the
> permission matrix** with a least-privilege default (`AUTHZ-008`).
>
> **⚠ Follow-up required — the seeded default.** `AUTHZ-008` defaults restrictive. If read scope
> ships closed, **cross-pool resurfacing silently does not work** until an Administrator widens
> it, because a recruiter cannot see candidates outside their assigned postings. The seeded matrix
> values are a decision in their own right, not an implementation detail. See
> [Open questions](#open-questions).

### C-04 — Calendar and candidate email deferred to the final phase
| Reference | Position |
|---|---|
| §24 | Email Notification Service — *"Recommended"* |
| §24 | Calendar Service — *"Future or configurable"* |
| §33 | Phase 5, the final phase — calendar, panel orchestration, candidate communication templates |
| §34 | MVP Definition of Done — contains neither |

Their MVP ships without either. The **largest scope divergence** between the two documents, and it
lands on the item identified as our critical path ([D06](#d06--calendar-integration-depth)).

**Nuance — the spec splits this three ways, it does not defer everything.** Internal notification
email is a **core module** (§8 Notification Service; §24 *"Recommended"*; §25 notification job with
retry-and-backoff), sending shortlist, scheduling, feedback-reminder, offer, onboarding, and
blocker notifications — all to **internal** recipients. Only the calendar API and
candidate-facing communication are deferred to Phase 5. `OD-006` leaves the overall integration
scope open.

> ### ✔ RESOLVED 2026-08-14 — adopt the spec's split exactly
>
> **Amends [D04](#d04--outbound-communication-scope) and [D06](#d06--calendar-integration-depth).**
>
> | Piece | Placement |
> |---|---|
> | Internal notification email (Notification Service module) | **Early** — with slice 5's scheduling tasks |
> | Google Calendar API + free/busy | **Deferred** to a late slice |
> | Candidate-facing email | **Deferred**, gated on `OD-005` (Legal) |
> | `.ics` attachment | **Not adopted** |
>
> **Consequences:**
> - **Slice 5 shrinks substantially** — the largest scope reduction from this round.
> - **The Workspace delegation request is no longer critical path.** Still worth filing early since
>   it costs nothing, but nothing blocks on it now.
> - [D10](#d10--calendar-platform) (Google Workspace, domain-wide delegation) remains the chosen
>   platform when the calendar slice arrives.
> - **Accepted gap:** panelists receive an assignment email but no calendar entry; they add the
>   meeting themselves. Candidates are contacted manually, outside the system, until `OD-005`
>   resolves.

### C-05 — One scorecard per application
Spec `scorecards` keys on `candidate_application_id` — one per application — with
`resume_evidence_json`, `interview_evidence_json`, `consolidated_evaluation_json` (*"AI-assisted
evaluation"*), versioned, single `approved_by`. Closer to the two options rejected in
[D11](#d11--scorecard-structure) than the one chosen. **Attribution is the real difference:** ours
gives each panelist a scorecard they own and approve; theirs produces one consolidated document
with a single approver.

**Reconciliation finding:** the models are closer than the labels suggest. Their `interview_notes`
already carries `recommendation` (`strong_yes`…`strong_no`) and `ratings_json` **per round, per
panelist** — that *is* a per-round panelist evaluation, just named a note rather than a scorecard.
Adopting their model does not lose per-panelist attribution.

> ### ✔ RESOLVED 2026-08-14 — consolidated scorecard on top of per-round notes
>
> **Amends [D11](#d11--scorecard-structure).**
>
> - **Per-panelist evaluation** lives in `interview_notes` — `ratings_json` + `recommendation`,
>   submitted per round by its author.
> - **One AI-drafted consolidated scorecard per application**, `Draft` until human-approved
>   (`SCR-006`), with resume-derived and interview-derived evidence separated (`SCR-004`) and
>   evidence source plus confidence on each dimension (`SCR-005`). Human adjustment requires a
>   mandatory reason (`SCR-007`).
>
> **Why the reversal:** D11 rejected consolidation partly on a mischaracterisation — it was framed
> as averaging, which `SCR-004`/`SCR-005` explicitly are not. Two arguments not on the table at
> D11: someone must weigh three panelists at selection, and it is better as an auditable artifact
> than as private reasoning; and [D17](#d17--priority-lane-carry-forward) carry-forward is far
> simpler carrying one consolidated document than N per-round ones.
>
> **Cost accepted:** an additional AI touchpoint and approval gate.

### C-06 — Fixed scorecard dimensions
`SCR-003` fixes: role fitment, technical depth, innovation practice relevance, project relevance,
communication evidence, risk areas, gap closure status, overall recommendation. `INT-006` likewise
fixes note fields. Solves the cross-role comparability weakness in
[D15](#d15--scorecard-competencies), at the cost of role-specific fit. `OD-007` means the
dimensions are unapproved even in their own organisation.

**New argument since D15:** [C-01](#c-01--vector-store) now embeds interview notes. Semantic
retrieval over notes works materially better against a stable field schema than against
per-JD-varying competency names.

> ### ✔ RESOLVED 2026-08-14 — fixed dimensions only
>
> **Amends [D15](#d15--scorecard-competencies).** AI-suggested competencies are dropped.
>
> - **Scorecard dimensions** (`SCR-003`, fixed): role fitment, technical depth, innovation
>   practice relevance, project relevance, communication evidence, risk areas, gap closure status,
>   overall recommendation.
> - **Interview note fields** (`INT-006`, fixed): strengths, gaps, technical validation, project
>   depth, communication, concerns, recommendation, next-step suggestion.
> - **Rating scale** unchanged from D15: 5-point `strong_yes`…`strong_no`.
>
> **Consequences:**
> - D15's consequence *"competencies become part of the JD version"* is **superseded** — dimensions
>   are global, not JD-scoped, so editing a JD no longer touches scorecard structure.
> - **Closes two open items:** competency vocabulary seeding, and note templates per interview type
>   (`INT-006` is one fixed set across all interview types).
> - **Carries forward `OD-007`** — the dimension set is unapproved even in their organisation, so
>   expect business stakeholders to revise it. Treat `SCR-003` as a seeded default, configurable
>   per §29 #14, not as settled product truth.
> - `innovation practice relevance` is Miracle-Labs-specific; confirm it applies here or rename.

> ### ⚠ CORRECTED 2026-08-26 — "configurable per §29 #14" cites the wrong item; §29 has no scorecard-dimension entry at all
>
> **What was wrong.** The `OD-007` bullet immediately above reads:
>
> > Treat `SCR-003` as a seeded default, configurable per §29 #14, not as settled product truth.
>
> **`reference/spec.md` §29 item 14 is "Role and permission matrix values."** It has nothing to do
> with scorecard dimensions. Checked against all fifteen items: 1 session expiration, 2 PDF size
> limit, 3 MIME types, 4 OCR enablement, 5 malware scanner, 6 AI model provider/name/version by
> family, 7 prompt template active version, 8 ranking criteria weights, 9 aging thresholds, 10
> notification templates, 11 compliance disclosure text, 12 retention period, 13 export limits, **14
> role and permission matrix values**, 15 vector re-indexing batch size. **No item makes scorecard
> dimensions configurable.** §29 is silent on them, and `SCR-003` states them as a fixed list with no
> configuration hook.
>
> **What is true now, in two parts:**
>
> - **The conclusion stands.** `OD-007` genuinely means the dimension set is unapproved even in the
>   organisation that wrote it, so treating `SCR-003` as a **seeded default rather than settled
>   product truth** is right, and is unaffected by this correction.
> - **The mechanism was borrowed, not cited.** What C-06 was reaching for is §29's *pattern* —
>   values changeable without a code change — not a §29 entry that exists. The correct vehicle is
>   `platform-core`'s audited runtime-configuration registry (closed, declared, every change audited
>   with a mandatory reason), extended with a scorecard-dimension key. That is a real mechanism this
>   product already has; §29 #14 is not it, and building against #14 as written would put scorecard
>   dimensions inside the permission matrix.
>
> **Why it was missed.** C-06 was written on 2026-08-14 as one of twelve conflict resolutions in a
> single pass, and this is the only clause in it that cites a numbered §29 item rather than a
> requirement ID — the surrounding `SCR-003`/`INT-006`/`OD-007` citations are all correct, so the row
> reads as verified by association. Nothing subsequently had reason to open §29 and count, because no
> feature owning scorecard dimensions had been proposed.
>
> Found during `interview-pipeline`'s propose conversation, 2026-08-26 — the feature that owns
> `TS-BL-063` and would have implemented the citation as written. Recorded per `AGENTS.md`'s
> shared-document convention, at the point of the error, quoting the prior wording. The same class of
> defect as `domain-model.md`'s mis-cited AI-governance reasoning: right idea, wrong reference.

### C-07 — Carry-forward has no basis in the spec
`SCR-001` requires *"at least one completed interview **for the posting**"*; `BR-010`, `BR-011`,
`AI-009` repeat it. No carry-forward mechanism exists. This is **us extending beyond the spec**,
not the spec contradicting us. Their substitute is weaker but simpler: selected-not-offered
candidates get resurfacing precedence subject to mandatory criteria, but still re-enter the funnel
and re-interview.

**What changed the calculus:** [C-06](#c-06--fixed-scorecard-dimensions) made scorecard dimensions
**fixed and global**, so a scorecard from posting A is structurally identical to one from posting
B. The comparability objection largely dissolves; what remains is role-specific *evidence*, not
structural mismatch.

> ### ✔ RESOLVED 2026-08-14 — carry forward, but require one confirmatory round
>
> **Amends [D17](#d17--priority-lane-carry-forward) and
> [D21](#d21--carry-forward-eligibility).** The carried consolidated scorecard is cited as **prior
> evidence**; the candidate still completes **one interview round for the new posting**.
>
> **This dissolves the conflict rather than overriding it.** One round for the posting satisfies
> `SCR-001`, `BR-010`, `BR-011`, and `AI-009` — so no spec rule is broken and **feature 13 no
> longer needs restating**. The rounds skipped are the repetitive ones, not all of them.
>
> **Retained from D17/D21:** scorecards link to multiple applications with a `carried_forward`
> record (attesting user, timestamp, typed reason); same-JD is the fast path, different-JD needs
> fuller PM justification; the route taken is instrumented for
> [OA-01](#oa-01--carry-forward-volume-distribution).
> **Dropped:** the non-round-1 entry point — candidates now enter at a round, not at selection.
> **Reinforced by [G-01](#g-01--mandatory-criteria-status):** mandatory criteria are re-evaluated
> for the new posting regardless of carried evidence.

### C-08 — Note versioning without a lock
`INT-008`/`BR-017` — every edit creates a version, forever, with no freeze and no restriction to
the author. [D24](#d24--interview-note-locking) chose author-only with a hard lock on scorecard
approval. Theirs is more flexible; ours better guarantees that an approved evaluation matches the
evidence it was drafted from.

> ### ✔ RESOLVED 2026-08-14 — the spec's model exactly
>
> **Amends [D24](#d24--interview-note-locking).** Immutability is achieved by **versioning, not
> freezing**: every edit to a submitted note creates a new version (`INT-008`, `BR-017`), forever.
> No lock on scorecard approval, and no hard-coded author restriction.
>
> **Who may edit** is now a **permission-matrix question**, consistent with
> [C-02](#c-02--nine-roles-vs-four) and [C-03](#c-03--recruiters-scoped-to-assigned-postings) — the
> `Edit` action flag on the Interview Console page decides it, per role. Recommended seeding:
> grant `Edit` on notes to Interviewer only.
>
> **⚠ Open consequence — scorecard drift.** With no lock, a note can change *after* the
> consolidated scorecard drawn from it was approved, so an approved evaluation can silently stop
> matching its evidence. The spec provides the tool but does not wire it up: `scorecards.status`
> already includes **`superseded`**. Recommended follow-up: a note version created after scorecard
> approval marks that scorecard `superseded` and requires re-approval. Tracked in
> [Open questions](#open-questions).

### C-09 — Adversarial AI testing as a requirement
`AI-015` — ranking templates *"shall be tested against adverse examples, low-information resumes,
adversarial resumes, and conflicting interview notes before production release"*; §27.1 makes AI
Evaluation Tests a required coverage category. Conflicts with the non-goal that an offline eval
harness is a separate initiative — the spec treats a **minimal** version as MVP-blocking. Middle
path: a fixed adversarial test set run in CI, without the full golden-dataset apparatus.

> ### ✔ RESOLVED 2026-08-14 — adopt §27.1 in full
>
> **Reverses the non-goal** that an offline eval harness is a separate initiative.
>
> **AI Evaluation Tests** across all ten prompt families (§16.3): low-information resumes,
> adversarial resumes, conflicting evidence, protected-attribute redaction, insufficiency outputs,
> evidence citation. Plus `AI-015` — templates tested against adverse examples **before production
> release**, and `ENG-008` — production changes affecting ranking behaviour require prompt/model
> version review.
>
> **Security Tests** (also §27.1): broken access control, file upload attacks, prompt injection
> samples, sensitive data exposure, export permission checks.
>
> **What is still out of scope:** statistical disparate-impact analysis and fairness metrics —
> not because they are unimportant, but because `PRV-006` and `OD-010` **forbid collecting
> demographic or protected-class data without legal approval**. The bias-audit non-goal therefore
> survives; only the eval-harness non-goal is reversed.
>
> **Cost:** a maintained test corpus and CI integration touching slice 2 and every AI-bearing
> slice thereafter. Materially more than the minimal version, and the strongest guard available
> against the injection and protected-attribute failure modes.

### C-10 — Interviewers and the ranking board
`INT-004`/`INT-005` give interviewers suggested questions and source-labelled fitment and gaps. But
§8 lists AI Ranking Board users as *"Practice Managers, Recruitment Managers, permitted
Recruiters"* — **Interviewer is not among them** — and §9.1 describes the role as reviewing
*"candidate fitment and gaps"* with no mention of score or rank. Effectively the "gaps yes, score
no" option rejected in [D05](#d05--ai-context-visible-to-interviewers). The spec does not forbid
showing the score, but its surface allocation implies it.

> ### ✔ RESOLVED 2026-08-14 — gaps yes, score no
>
> **Amends [D05](#d05--ai-context-visible-to-interviewers), reversing it.** Interviewers see
> AI-suggested questions and source-labelled fitment and gaps (`INT-004`, `INT-005`), plus the JD
> and resume. They do **not** see `latest_match_score`, rank position, or other candidates — the
> AI Ranking Board remains a Practice Manager / Recruitment Manager surface (§8).
>
> **What changed since D05:** [G-02](#g-02--evidence-references-and-source-labelling) was adopted,
> so fitment and gaps now arrive with evidence references and source labels. That carries the
> substantively useful content; the numeric score added little beyond the anchoring effect.
>
> **Consequences:**
> - D05's mitigation *"log that the score was visible pre-interview"* is **no longer needed**.
> - The evidence-required note template remains valuable and is retained.
> - Interview Console UI simplifies — no score display, no comparative view.
> - **The AI agreement rate metric becomes meaningful.** With the score hidden, comparing a
>   panelist's verdict against the AI ranking measures genuine agreement rather than a
>   self-fulfilling one.

### C-11 — Staleness windows: fixed vs. configurable
§29 makes aging thresholds configurable without code changes; `RET-001` puts retention under
Company policy; `OD-004` assigns the retention period to HR, Legal, and Privacy.
[D22](#d22--staleness-windows) fixed 90 days and 12 months. The 12-month resurfacing window is
effectively a retention/eligibility policy that **may not be ours to set**.

**Three distinct concepts, worth separating:** §29 #9 aging thresholds drive **SLA dashboards**;
`OD-004`/`RET-001` govern **data retention**; D22's windows are **eligibility rules** the spec has
no equivalent for.

> ### ✔ RESOLVED 2026-08-14 — configurable, seeded at D22's values, audited
>
> **Amends [D22](#d22--staleness-windows).** Both windows become configuration (following the §29
> pattern), seeded at **90 days** priority-lane TTL and **12 months** resurfacing recency. Changes
> are **audited** — they alter eligibility for a decision shortcut.
>
> **Hard constraint recorded:** the resurfacing window can never exceed the retention period set
> by `OD-004`. You cannot resurface a candidate whose data has been deleted. If Legal sets a
> shorter retention period, the seeded 12 months must be reduced to match.

### C-12 — Who drafts the JD
`JOB-001` — *"Authorized **Practice Managers** shall create job description records"*; §8 lists JD
Workspace users as Practice Managers and Recruitment Managers, not Recruiters.
[D12](#d12--job-description-approval) has the Recruiter drafting and the PM approving. If the PM
both creates and approves, that is functionally the self-approval option D12 rejected — wearing a
different hat.

**But the spec never names the approver.** `job_description_versions.approved_by` carries no role
constraint and `JOB-005` requires only *"human review before approval."* Self-approval is an
inference, not a requirement. And [C-02](#c-02--nine-roles-vs-four) introduced a **Recruitment
Manager** role with little function beyond dashboards.

> ### ✔ RESOLVED 2026-08-14 — Practice Manager drafts, Recruitment Manager approves
>
> **Amends [D12](#d12--job-description-approval).** The Recruiter is removed from the JD flow, per
> §8 (JD Workspace users are Practice Managers and Recruitment Managers).
>
> ```
> PM drafts (AI-assisted) ──▶ SUBMITTED ──▶ RM approves / requests changes ──▶ APPROVED
> ```
>
> **Why:** the PM owns the headcount and holds the role knowledge; the RM provides genuine
> separation of duties and the role gains a real function.
>
> **Retained from D12:** approved JDs **fork on edit**; live postings stay pinned to the version
> they were published against.
> **Changed:** the review queue belongs to the Recruitment Manager, not the Practice Manager.
> **Fallback required:** when no Recruitment Manager is assigned, approval falls to another
> Practice Manager — **never the drafter**.

## R.4 — Reconciliation items to fold into design

Minor modelling differences noted for design time, not requiring a decision now.

- **`selection_tag` vs. bench.** The spec's `priority_selections.selection_tag`
  (`primary | backup | hold | selected_not_offered`) puts backups **inside** the 5×n cap;
  [D20](#d20--priority-slot-mechanics) puts the bench **outside** it. Reconcile during design.
- **`question_set_id`** on interview rounds — AI-generated questions stored as a retrievable
  artifact rather than transient output.
- **`work_mode`, `employment_type`, `priority`, `target_start_date`** on postings — posting
  metadata fields not previously enumerated.
- **`final_outcome`** on applications (`recruited | not_selected | selected_not_offered |
  withdrawn | declined | no_show`) — a terminal disposition distinct from workflow `status`.
- **`profile_summary`, `current_location`, `global_status`** on candidates
  (`active | archived | deletion_requested | restricted`) — `deletion_requested` supports the
  data-subject-request path raised under [D04](#d04--outbound-communication-scope).
- **Seed scripts** (`ENG-010`) for roles, permissions, prompt templates, lifecycle states, and
  initial admin users — needed for slice 1 to be testable at all.

## R.5 — Effects of the 2026-08-14 conflict resolutions

C-01 through C-04 were all resolved toward the reference spec. Two consequences appear only when
they are combined, and one affects phasing.

### Interaction A — permission changes now trigger vector re-indexing

[C-01](#c-01--vector-store) adopted `VEC-003` (*vector retrieval applies the same authorization
rules as relational data access*) and [C-03](#c-03--recruiters-scoped-to-assigned-postings) made
read scope **matrix-configurable**. Together:

```
  Admin changes the permission matrix
        │
        ▼
  read scope for some users changes
        │
        ▼
  authorization_scope_json baked into vector records is now stale
        │
        ▼
  VEC-005 re-index  ← the spec anticipates exactly this: re-indexing is
                      required when "authorization metadata changes"
```

Neither decision implies this alone. The design choice it forces: **evaluate authorization at
query time rather than baking scope into the index**, so a matrix change does not require
re-embedding. `authorization_scope_json` should carry the *inputs* to an authorization decision
(candidate ID, posting ID, practice) rather than a resolved verdict.

### Interaction B — the seeded matrix is a decision, not a default

`AUTHZ-008` makes least privilege the default. Combined with
[C-03](#c-03--recruiters-scoped-to-assigned-postings)'s configurable reads, **shipping with a
closed matrix means cross-pool resurfacing does not work** — a recruiter cannot see candidates
outside their assigned postings, so the resurfacing suggestion queue is empty for them. The seeded
matrix values determine whether a headline feature functions on day one. Tracked in
[Open questions](#open-questions).

### Phasing impact on [D13](#d13--phasing--change-decomposition)

**Resolved 2026-08-17.** The nine-slice plan was rebuilt into fourteen slices to absorb exactly
these effects — see the ✔ SUPERSEDED block under [D13](#d13--phasing--change-decomposition) for
the full breakdown, the dependency diagram, and the MVP boundary. In short: weight moved
**earlier** (the new S2 Authorization, S4 AI Governance, and S7 Vector Retrieval slices), the
old "Funnel" slice split apart with calendar work pushed to a new S14 that no longer blocks
anything, and the externally-owned Workspace-approval dependency has left the critical path
entirely.

---

# Part S — Visual Style & Layout Guide Reconciliation

**Source:** two documents provided by the user's manager on 2026-08-20, read in full and saved to
`reference/design-spec.md` (187 lines — color, typography, iconography, shape, elevation) and
`reference/layout-spec.md` (180 lines — app shell, sidebar, headers, grid, floating panel,
breakpoints, spacing). Both are explicitly **product-agnostic design system specifications**,
written for "applications that should share a consistent look and feel," not authored for
TalentSphere specifically.

**Reconciliation posture:** unlike `reference/spec.md` and the GCP landing-zone spec, this pass
found **no direct conflicts** with any of D01–D24, C-01–C-12, or G-01–G-14 — because no prior
visual-design decision existed to conflict with. This reconciliation is therefore mostly
additive, with a few things flagged as *not applicable* (rather than silently built) and a few
genuine gaps the guide doesn't resolve.

### S.1 — A fit worth noting: Floating Action Panel ↔ Priority Selection

The layout spec's §6 Floating Action Panel — a persistent bottom-right panel for "an in-progress,
cross-page task," carrying selected-item chips plus a running output — is a strong, close match
for **Priority Selection** ([D20](#d20--priority-slot-mechanics)): a Practice Manager browsing
the ranked list across a posting, picking candidates into the 5×n slots, reviewing the running
selection, confirming. This emerged from reading the guide; it wasn't designed in from our side.
Worth carrying into Wave 3's design.md ([S11](#d13--phasing--change-decomposition)) as the
intended UI for that workflow.

### S.2 — Not applicable — flagged, not silently built

- **Product switcher** (layout §2.3) — assumes the shell hosts multiple distinct products or
  workspaces under one nav rail. TalentSphere is a single product. **Do not build this.**
- **Bare shell's "embeddable/shared read-only document" case** (layout §1) — assumes a
  public-facing content view exists. This conflicts with the standing non-goal that candidates
  have zero system access and no public routes exist. **The bare shell applies only to the
  sign-in screen.**

### S.3 — Genuine gaps: not resolved by the guide as given

- **No dense-data-table pattern.** The guide's three page templates (§4.2: browsable card grid,
  detail view, standalone) are card/tile-oriented. TalentSphere has several inherently tabular,
  dense-data screens with no good fit among them: the AI Ranking Board (`S8`), the Admin
  permission matrix (`S2`, `ADM-003`'s row/column grid), and audit/AI-run logs (`S13`). These need
  their own pattern in `design.md`, built from the guide's existing spacing/radius/color tokens
  rather than inventing new ones — a fourth template, not a repurposing of the card grid.
- **No evidence-source tagging pattern.** `reference/spec.md` requires visually separating
  resume-sourced, interview-sourced, scorecard, and human-decision evidence
  ([G-02](#g-02--evidence-references-and-source-labelling)). The style guide's only categorical
  color system is semantic status (success/warning/info/error — §1.4), which means something
  different (state, not source). A small, consistent solution is needed — most likely icon +
  label rather than a new color family, consistent with the guide's own §1.6 rule against
  introducing one-off colors.
- **Presentation/fullscreen shell has no assigned screen yet.** The layout spec defines this shell
  state (§1, state 3) but nothing in TalentSphere's scope has been mapped to it. Candidate:
  the Interview Console during an active interview, or a full-screen resume/document viewer —
  not decided, tracked as an open item.

### S.4 — Confirmed reinforcements (no new decision, existing requirements now have a component)

- The guide's mandatory, always-visible focus indication (layout §4.3-equivalent /
  design §4.2 Focus Ring) directly satisfies `UI-009`'s WCAG 2.1 AA requirement — nothing to add,
  just confirms an implementation vehicle exists.
- The badge/pill component (design §4.3) plus the status-surface tint/border/text triplets
  (design §1.5) give a natural home for two previously-unimplemented requirements: labeling
  AI-generated content until approved (`UI-004`/`AI-001`) and the ranking board's human-review
  disclaimer (`RANK-004`/`UI-006`).

### S.5 — AI output conciseness & visual craft standard (added 2026-08-20)

**New requirement, not previously captured anywhere:** the application should be visually
excellent and never feel overcompacted with text — and, most importantly, **every piece of
AI-generated content in the product must be concise, precise, and clear.** No long-form AI text
anywhere in the UI.

**Relationship to the existing design system — reinforcing, not conflicting.** Design §2.2
already biases interface typography toward "compact, information-dense" text — a font-**size**
choice. That only reads well if the actual **content** is also short: cramming a long AI-written
paragraph into small, dense type is precisely the overcompacted result this requirement rules
out. This sharpens the existing type philosophy rather than fighting it, provided content length
is genuinely constrained — which, until now, nothing did.

**The real gap this closes:** `reference/spec.md` §16.3's Prompt Template Registry specifies an
**output shape** for all ten prompt families (strict JSON, source-labeled, evidence-referenced)
but never a **length**. Nothing in scope so far stops a model from writing three paragraphs where
two sentences would serve.

**How this applies, concretely:**
1. **Every AI capability's output contract needs an explicit conciseness constraint**, not just a
   JSON shape — e.g. a fitment summary capped at a small number of sentences, a gap summary as a
   short bulleted list rather than prose, interview questions as single-line prompts. Enforced
   structurally in the prompt template itself, not left as unenforced writing-style guidance.
2. **The AI evaluation harness** (adopted in full per [C-09](#c-09--adversarial-ai-testing-as-a-requirement),
   §27.1) should test for length/conciseness compliance as one of its checks, alongside the
   adversarial, low-information, and protected-attribute tests already planned.
3. **A general UI-execution discipline, not a one-time task:** every screen from Wave 1 onward
   should surface the single most important thing prominently, with supporting detail
   progressive/expandable rather than always fully displayed on screen at once.

**Where this lands in the backlog:** primarily **S4 — AI Platform & Governance**, since that's
where the prompt template registry and evaluation harness both live; each of the ten prompt
families gets an explicit conciseness constraint added to its output contract as part of S4's
scope. Reinforced practically wherever AI output actually renders: S5 (JD drafts), S6 (resume
extraction), S8 (ranking/fitment/gap — the highest-volume AI surface in the product), S9
(interview questions), S10 (scorecards), S12 (resurfacing summaries).

**Genuinely open, not yet specified:** exact length bounds per prompt family (sentence counts,
word caps) — a concrete number wasn't given and shouldn't be invented here; this is a design
decision for whoever writes S4's prompt templates, informed by this principle, not resolved by
it.

### S.6 — Two things this reconciliation missed, found during `design-system`'s propose conversation (2026-08-25)

Part S opens by stating that reading both guides in full "found **no direct conflicts**," and S.3
enumerates the gaps it did find. Two real problems are absent from that list. Recorded here rather
than worked around inside `design-system`'s own artifacts, per this document's convention and
`AGENTS.md`'s "Finding a genuine gap in `exploration-notes.md` itself" rule — the same treatment
[D09](#d09--tech-stack)'s background-jobs row got the same day.

**1. `design-spec.md` §1.5's Success status-surface triplet is the Info triplet, and this
contradicts §1.4.**

Every other row of §1.5 is demonstrably derived from that status's own §1.4 semantic color: the
Warning row's text values `#92560A` (light) and `#F59E0B` (dark) are shades of §1.4's Warning
`#F59E0B`; the Error row's `#C81E26` / `#EF4048` are shades of §1.4's Error `#EF4048`. In both
rows the dark-mode text value *is* the §1.4 color itself.

The Success row breaks this. Its `#006A90` / `#00AAE7` are shades of the **brand accent**
`#00AAE7`, not of §1.4's Success `#22C55E` — and all six of its values are character-for-character
identical to the Info row's, in both themes. So a success banner and an info banner render as the
same pixels, and §1.4's Success green has no surface expression anywhere in the system. This reads
as a copy-paste of the Info row rather than a deliberate choice, because a deliberate choice would
not have left §1.4's Success color orphaned.

*Why this is load-bearing rather than a cosmetic nit:* `TS-BL-012`'s toast renders `platform-core`'s
notification severity, which `platform-core`'s `platform/notifications` spec deliberately defines as
**semantics only**, deferring appearance to §1.5. If two severity values map to identical
appearance, that enumeration is decorative at the one place a user actually meets it.

*Why S.1–S.5 missed it:* that pass compared the two guides against TalentSphere's existing
decisions (D01–D24, C-01–C-12, G-01–G-14) and against each other. This defect is *within a single
guide* — §1.4 against §1.5 — a comparison the pass never ran, because neither section conflicted
with anything TalentSphere had already decided.

**Resolution — §1.5's Success row is superseded.** Success gets a triplet built by the same
construction the Warning and Error rows demonstrably use (pale tint background / mid-tint border /
dark readable text in light mode; very dark background / mid-dark border / the §1.4 color itself as
text in dark mode), applied to §1.4's Success `#22C55E`. Info keeps the accent-blue family
unchanged:

| Status | Light — background / border / text | Dark — background / border / text |
|---|---|---|
| Success *(replaces §1.5's row)* | `#E9F9EF` / `#8FE0AE` / `#0F6B36` | `#04170C` / `#17512F` / `#22C55E` |

These values are **derived here, not supplied by the guide** — its author gave none for this case.
They may be adjusted by whoever owns the visual language, provided the construction rule above and
WCAG 2.1 AA contrast both still hold (as checked: `#0F6B36` on `#E9F9EF` is 6.1:1, `#22C55E` on
`#04170C` is 8.1:1, against `UI-009`'s 4.5:1 threshold).

**2. The sidebar's surface color has no token anywhere, and S.3 should have listed it.**

`layout-spec.md` §2 requires the sidebar to render "against its own dark navy surface" which "does
not participate in the light/dark theme toggle," and §2.2 requires two further fixed constants for
its hover and active tints, explicitly "not the shared light/dark theme colors." None of those
three colors exists in `design-spec.md` §1 — its darkest defined surface is `#09090B`, a neutral
near-black, not a navy — while §1.6 forbids introducing a one-off color for a single use case and
permits only extending an existing set. The layout guide therefore mandates three colors the style
guide neither supplies nor allows inventing at a call site.

This belongs in S.3's list of genuine gaps and isn't there. It is a gap of the same kind as the
missing dense-data-table pattern: something TalentSphere must build that the guide left undefined.

**Resolution — extend the token set, don't improvise.** A named, theme-independent `sidebar` token
family (surface, hover tint, active tint, and its own text/icon values) defined once in
`TS-BL-007`'s token set. Extension is precisely what §1.6 asks for; the thing it forbids is a hex
written at a call site, which a named family avoids. Concrete values are `design-system`'s to fix
and record in its `design.md`, since the guide supplies none.

**Neither item reopens a D/C/G/S decision.** Nothing in this document had decided anything about
either point. What was wrong is Part S's claim to be a *complete* reconciliation — S.1, S.2, S.4
and S.5's own conclusions all stand unchanged.

---

# Part D — Delivery Model Restructure (2026-08-24)

Everything in this Part is **process/organizational**, not functional. Nothing here changes any
of D01–D24, C-01–C-12, or G-01–G-14 — those decisions describe *what TalentSphere does*; this
Part describes *how the work gets planned, scheduled, and proposed*. Triggered by explicit
direction from the user's manager, received after Wave 1 Sprint 0 was built and deployed (see
`openspec/changes/talentsphere-wave-1-foundation/sprint-0-outcome.md`).

## D.0 — The manager's instruction, verbatim

> "I plan to run this in weekly sprints. Every sprint must contain backlog of items. Every
> backlog items must have a unique number. Backlog items within a sprint must be grouped into
> waves with each wave having a unique number too. Every item inside a wave is independent and
> can be worked in parallel. Every backlog item must be not too small nor too large. Every
> backlog item must be a cohesive unit of work/code which can travel independently to dev, uat
> and prod environments - thus ensuring that sprint spill-overs do NOT block the UAT. Make
> adjustments as needed in the OpenSpec structure. Follow all best-practices to promote a
> cursor-heavy development of the final application sprint-by-sprint."

Followed by two further clarifications from the same conversation:

1. **Generate all sprints and waves for the entire product now**, not incrementally per phase as
   its turn arrives.
2. **Every backlog item gets a full `/opsx:propose` cycle** — `proposal.md`, `design.md`,
   `specs/`, and `tasks.md` — not just `proposal.md` with the rest deferred. The manager wants the
   complete spec for the whole product before more implementation continues, so he can review and
   approve it.

## D.1 — Terminology resolution: the word "wave" now means something different

The manager's model inverts our existing hierarchy. Reflected precisely:

```
MANAGER'S MODEL                              OUR OLD MODEL (renamed below)
Sprint  = 1 week (time-box)                  Wave  = 5 macro-groupings spanning many sprints
  └─ Wave = parallel grouping WITHIN           └─ Slice (S1-S14)
       a sprint — items inside a wave              └─ Sprint = 1 week, within a slice
       are independent, worked in parallel
       └─ Backlog item = independently
            deployable unit, uniquely numbered
```

Our old "wave" and the manager's new "wave" describe opposite things (a multi-sprint macro
grouping vs. a same-sprint parallel grouping) and cannot both keep the name.

**Resolution — our old "wave" is renamed "Phase."** Phase 1 through Phase 5 carry forward
everything already decided about them (contents, MVP boundary, dependency order) — only the
label changes. `talentsphere-wave-1-foundation` conceptually becomes "Phase 1," though see D.5
below for what actually happens to that change.

**Sprint** keeps its existing meaning (1 week) — no change needed, we'd already switched to
1-week sprints before this instruction arrived.

**Wave** now means: a same-sprint grouping of mutually-independent backlog items. A sprint may
contain one or several waves. Unique IDs: `TS-SPR-XXX` (sprint), `TS-SPR-XXX-WV-XXX` (wave).

## D.2 — Backlog items are the propose-mode unit, not waves or sprints

**Core finding:** the manager's own definition of a backlog item — *"a cohesive unit of
work/code which can travel independently to dev, uat and prod"* — is functionally identical to
what an OpenSpec `change` is: a bounded, independently reviewable and buildable unit of
specification. A sprint is a time-box; a wave is a parallelism grouping; neither represents
coherent functionality, so neither is sensible to write one `proposal.md`/`design.md` for. A
backlog item is.

**Resolution:** `/opsx:propose` runs once per backlog item, not once per Phase and not once per
sprint. This directly answers "should each feature go through openspec propose mode?" — yes,
because a backlog item **is** a feature at exactly the grain OpenSpec expects.

**Consequence for `changes/` vs. `delivery/`:**

```
changes/<backlog-item-id>/     WHAT to build — proposal, design, specs, tasks.
                                Written once per item, independent of which sprint
                                it ends up scheduled into.

delivery/sprints/<SPR>/         WHEN it's being built — pure scheduling and status,
  waves/<SPR-WV>.md              references the change-id, never repeats its content.

delivery/roadmap.md             Phase 1-5 narrative — the one place "Phase" survives
                                 as a formal grouping.

delivery/release-plan.md        how a finished backlog item promotes dev -> uat -> prod.
```

`delivery/` is **not** an OpenSpec-native concept — the `openspec` CLI does not read or
understand it. It is a project-specific tracking layer maintained by convention, sitting
alongside OpenSpec's own `changes/`/`specs/`/`config.yaml`.

## D.3 — The master backlog: 14 slices become 23 backlog items

Four slices were too large to be one independently-deployable unit and were split, informed by
where the *real* Sprint 0/Phase 1 work naturally seamed (see D.5). One new item
(`TS-BL-0000`) was added for infrastructure, which was never one of the original 14 slices but
was always necessary — see the original Wave 1 proposal's "Enabling scope."

| Backlog ID | Item | Depends on | Phase | Spec source |
|---|---|---|---|---|
| TS-BL-0000 | Delivery Foundation | — | 1 | `platform/delivery-foundation` |
| TS-BL-0001 | Identity & Session Foundation | TS-BL-0000 | 1 | `identity/authentication`, `identity/user-activation` |
| TS-BL-0002a | Permission Evaluator & Audit Substrate | TS-BL-0001 | 1 | `access-control/authorization`, `platform/audit-trail` |
| TS-BL-0002b | Seed Data & Permission Matrix | TS-BL-0002a | 1 | *(carved from admin-cockpit)* |
| TS-BL-0003 | Admin Cockpit UI & Break-glass | TS-BL-0002b, TS-BL-0004 | 1 | `access-control/admin-cockpit` |
| TS-BL-0004 | Design System Foundation | TS-BL-0000 | 1 | `design-system/foundations`, `design-system/app-shell` |
| TS-BL-0005 | Workflow & Notification Engine | TS-BL-0000 | 1 | `platform/workflow-engine`, `platform/notifications` |
| TS-BL-0006 | AI Gateway & Run Logging | TS-BL-0000 | 1 | `ai-platform/ai-gateway`, `ai-platform/ai-run-logging` |
| TS-BL-0007 | Prompt Registry & Eval Harness | TS-BL-0006 | 1 | `ai-platform/prompt-registry`, `ai-platform/ai-evaluation` |
| TS-BL-0008 | Job Description Workspace | TS-BL-0002a, TS-BL-0006 | 2 | S5 (split) |
| TS-BL-0009 | Job Posting Management | TS-BL-0008 | 2 | S5 (split) |
| TS-BL-0010 | Resume Intake Pipeline | TS-BL-0000 | 2 | S6 (split) |
| TS-BL-0011 | Candidate Database & Dedup | TS-BL-0010 | 2 | S6 (split) |
| TS-BL-0012 | Vector Retrieval Substrate | TS-BL-0011 | 2 | S7 |
| TS-BL-0013 | AI Ranking Engine | TS-BL-0009, TS-BL-0012, TS-BL-0006 | 2 | S8 (split) |
| TS-BL-0014 | Ranking Board UI & Insights | TS-BL-0013, TS-BL-0003 | 2 | S8 (split) |
| TS-BL-0015 | Shortlisting Workflow | TS-BL-0014, TS-BL-0005 | 3 | S9 (split) |
| TS-BL-0016 | Interview Console | TS-BL-0015 | 3 | S9 (split) |
| TS-BL-0017 | Scorecard Center | TS-BL-0016 | 3 | S10 |
| TS-BL-0018 | Priority Selection & Offer Tracker | TS-BL-0017 | 3 | S11 (split) |
| TS-BL-0019 | Closure Rules & Evergreen Lifecycle | TS-BL-0018 | 3 | S11 (split) |
| TS-BL-0021 | Dashboards & Reporting | TS-BL-0019 | 4 | S13 |
| TS-BL-0020 | Resurfacing & Priority Lane | TS-BL-0019 | 5 | S12 |
| TS-BL-0022 | Calendar & Candidate Communications | TS-BL-0009, TS-BL-0016 | 5 | S14 |

`TS-BL-0002a` absorbs `platform/audit-trail` — `design.md` D6/D7 were written specifically about
auditing permission and admin changes, so pairing them keeps that reasoning intact rather than
splitting a single decision's rationale across two changes.

## D.4 — Full artifact generation, not proposal-only: the tradeoff accepted once, on record

Earlier in this Part's own timeline, the plan was: front-load `proposal.md` for every item
(stable, low-risk), defer `design.md`/`specs/`/`tasks.md` per-phase (avoids designing against
technical realities that don't exist yet). **The manager's explicit instruction overrides this**
— full `/opsx:propose` (all four artifacts) runs for every backlog item across all five phases,
before further implementation continues.

**Risk accepted, stated once:** `design.md` for Phase 4-5 items will reference technical
decisions (schemas, APIs) that Phases 1-3 haven't actually built yet. Some of it will need
revision once those phases are real — via `openspec-update-change`, not by treating the risk as
a reason not to proceed. This is the manager's informed, explicit call, not an oversight.

**One simplification this creates:** the earlier plan required deliberately *avoiding*
`/opsx:propose` (its guardrail always completes the full artifact set, which conflicted with
wanting proposal-only). That conflict no longer exists — `/opsx:propose`, run normally, is now
exactly the right tool for every item.

## D.5 — Migration: talentsphere-wave-1-foundation splits, doesn't rewrite

The existing change already has 14 real spec files and `design.md` decisions D1-D18 of
publication quality (see the Sprint 0 verification in this conversation). Under the new model it
becomes **nine separate changes** (`TS-BL-0000` through `TS-BL-0007`, with `0002` split into
`0002a`/`0002b`), each inheriting its slice of the existing content rather than being rewritten:

- D1, D2, D6, D7 (evaluator, scope predicates, audit design) → `ts-bl-0002a-evaluator-audit`
- D3, D4 (seed matrix, seeded values) → `ts-bl-0002b-seed-permission-matrix`
- D13, D14 (design tokens, dense data table) → `ts-bl-0004-design-system-foundation`
- D9, D10, D11, D12 (AI gateway, advisory-only, conciseness, prompt registry) split across
  `ts-bl-0006-ai-gateway-run-logging` and `ts-bl-0007-prompt-registry-eval-harness`
- D18 (IAM database auth) → `ts-bl-0000-delivery-foundation`, cross-referencing D7 explicitly
  rather than losing the connection between them

**Named risk:** some decisions cross-reference across this split (D18 depends on D7's
reasoning). Each split-out `design.md` must carry an explicit pointer to the other change when
this happens, not silently drop the connection.

Sprint 0 itself retrofits into `delivery/sprints/TS-SPR-000/`, referencing `TS-BL-0000` as
already complete (13/13, verified live Dev deployment) — the historical record in
`sprint-0-outcome.md` is not rewritten, only referenced.

## D.6 — Team-based parallel scheduling (3-4 junior developers, not AI-agent parallelism)

Reframed once the actual staffing became clear: a team lead (the user) plus 3-4 junior
developers, assigning backlog items to people weekly. This makes real parallelism concretely
achievable (unlike solo or AI-agent-only execution, where "waves" risk being labels on paper
without real concurrent execution) — but caps genuinely simultaneous waves at team size, and
makes the team lead's own review/integration time a shared bottleneck resource across every wave.

**Worked example — Phase 1 compressed from 16 real sprints to a target of 8**, by identifying
which of the 9 backlog items are genuinely independent of the Identity -> Evaluator -> Admin
Cockpit critical path (Design System, Workflow Engine, and AI Gateway/Prompt Registry all
qualify) and running them in parallel waves alongside it, plus splitting the heaviest single item
(`TS-BL-0002`, ~4 of the original 15 sprints) into `0002a`/`0002b` so two people can work it at
once instead of one:

```
Sprint 1   WV-1: TS-BL-0001 (start)  WV-2: TS-BL-0004  WV-3: TS-BL-0005  WV-4: TS-BL-0006 (start)
Sprint 2   WV-1: TS-BL-0001 (cont.)  WV-2: TS-BL-0006 (cont.)
Sprint 3   WV-1: TS-BL-0002a (start, needs 0001)   WV-2: TS-BL-0007 (start, needs 0006)
Sprint 4-5 WV-1: TS-BL-0002a (cont.)  WV-2: TS-BL-0002b (needs 0002a)  WV-3: TS-BL-0007 (cont.)
Sprint 6-8 WV-1: TS-BL-0003 (needs 0002b AND 0004)
```

Dependency notation replaces named assignment — who picks up an item each week is a staffing
decision that changes; what it's blocked on is a structural fact recorded once, e.g.:

```yaml
backlog_items:
  - id: TS-BL-0002a
    depends_on: [TS-BL-0001]
    status: in-progress
```

## D.7 — Execution plan

```
1. Log this Part (now)
2. Write project.md, glossary.md, domain-model.md, AGENTS.md — prerequisites for
   every propose conversation below, since none of them share memory with each other
   and all need the same conventions to stay consistent
3. Five dedicated conversations, one per Phase, each running /opsx:propose repeatedly
   for that phase's backlog items in dependency order:
     Phase 1 (9 items — mostly a split of existing content)
     Phase 2 (~7 items)
     Phase 3 (~5 items)
     Phase 4 (1 item)
     Phase 5 (2 items)
4. Each conversation reads exploration-notes.md, AGENTS.md, glossary.md, and
   domain-model.md first, then works its phase's items in dependency order
```

**Why per-phase, not per-item or all-at-once:** ~23 separate conversations is too fragmented for
closely related, interdependent work; one giant conversation for everything violates the
"each stage gets a clean slate" discipline already established. Per-phase is the balance point.

## D.8 — Correction: propose at the FEATURE level, and the true backlog is much finer-grained

Two corrections made in the same follow-up conversation as D.0–D.7, after checking this delivery
model against a real example from another of the manager's projects. **Both supersede specific
claims above; neither reopens D.0's core instruction, which both corrections still satisfy.**

### D.8.1 — "Backlog item" and "OpenSpec change" are not the same granularity

**D.2's core claim — "a backlog item's definition is functionally identical to an OpenSpec
change" — was wrong.** The real reference example's own category tags (`Organizations & Teams`,
`Identity & Access`) group *several* backlog items under one shared design domain —
`Organizations & Teams` alone covers eight items (`BL-023`–`BL-031`). That category is the
feature; a backlog item is something smaller living inside it.

**Corrected model:**

```
FEATURE   = the design-cohesion unit → ONE OpenSpec change (proposal.md, design.md, specs/)
            ~12-15 of these for TalentSphere, not ~23

BACKLOG ITEM = the deployment-tracking unit → a numbered, dependency-tracked entry inside
               its feature's tasks.md, AND inside delivery/'s wave/sprint scheduling
               ~85-90 of these once fully decomposed (see D.8.2)
```

**Why this is better, not just different:**
- Solves a real problem named in D.5: splitting decisions like D18→D7's cross-reference across
  separate small changes was awkward. A feature-level change keeps that reasoning together.
- Matches standard practice (Epic containing Stories) rather than inventing a flatter structure.
- ~12-15 proposals is a genuinely reviewable unit for the manager; ~86 tiny ones risks becoming
  *more* total review burden than the single giant document this whole model was meant to avoid.
- **"Independently deployable to dev/uat/prod" is a code/CI-CD property, not an OpenSpec-planning
  property.** A backlog item can share a `design.md` with siblings while still being its own
  separable, independently mergeable and deployable slice of code — these are different layers,
  and D.2 conflated them.

**D.3's master backlog table is superseded as a change list** (its 23 entries are not what gets
`openspec new change`'d) **but preserved as a source of dependency relationships** — the `depends
on` column and the slice-to-item mapping remain useful input for the finer decomposition below.

### D.8.2 — The true backlog is far finer-grained than D.3's 23 items

Checked against the same real reference example: items like `BL-019 Miracle Staff Login` and
`BL-044 Comments` are narrow, single-purpose — much smaller than, e.g., `TS-BL-0002a Permission
Evaluator & Audit Substrate`, which bundled an entire evaluator engine, an explanation endpoint,
and audit table design into one item. That's several reference-grain items pretending to be one.

**Revised estimate: ~85-90 backlog items total**, phase-by-phase:

| Phase | Estimated items |
|---|---|
| 1 — Foundation & Governance | ~29 |
| 2 — Core Hiring Loop | ~25 |
| 3 — Funnel, Evaluation & Decision | ~19 |
| 4 — Insight & Reporting | ~5 |
| 5 — Resurfacing & Communications | ~8 |
| **Total** | **~86** |

This is a projection, not yet a fully dependency-checked decomposition for Phases 2–5 (Phase 1
has a partial worked example — see D.8.4). Sanity-checked against the reference example's own
density (58 items for a comparably-scoped platform); TalentSphere landing higher is explained by
real complexity that example doesn't carry — nine roles with a full permission-matrix system, ten
separately-contracted AI prompt families plus a dedicated evaluation harness, and a fully
governed shortlist→interview→scorecard→selection→offer→closure pipeline.

### D.8.3 — Wave semantics corrected: global numbering, cuts across features, delivery per item

Checked clause-by-clause against the manager's original instruction (D.0) using the real
reference example as ground truth — every clause holds under the corrected model:

- **Waves are numbered globally and sequentially across the whole product** (`Wave 1` … `Wave
  11`...), never restarted per sprint — D.6's `TS-SPR-XXX-WV-XXX` per-sprint restart numbering is
  superseded.
- **Multiple waves routinely share one sprint.** The reference example's Sprint 1 alone holds
  three waves (10, 2, and 7 items). A sprint defaults to **one wave** whenever everything
  concurrent that week is mutually independent; a second wave appears only for a genuine reason
  — most often an item that depends on something else scheduled the same sprint (a
  stub-and-integrate overlap) and therefore cannot share a wave with what it depends on.
- **A wave cuts freely across features.** Independence, not feature membership, determines
  grouping — a wave can and typically will mix items from several different features.
- **Delivery happens per backlog item, not per wave.** A wave is a tracking lens; if one item
  spills into the next sprint, its wave-mates that finished on time still move toward UAT without
  waiting for it. This is the literal mechanism that satisfies D.0's "sprint spill-overs do NOT
  block the UAT."
- **D.6's own schedule violated this once, corrected in the same conversation:** its final
  version had crammed `TS-BL-0017 → 0018 → 0019` — a genuinely sequential chain — into one sprint
  and called it one wave, which breaks "every item inside a wave is independent." Left as a
  documented mistake rather than quietly fixed, per this document's own convention.

### D.8.4 — What's settled vs. what's still open after this correction

**Settled:** the feature-vs-backlog-item distinction, the wave/sprint semantics, the delivery/
vs. changes/ split (`tasks.md` = one feature's complete, self-contained backlog; `delivery/` = the
cross-feature scheduling view — a recipe vs. a kitchen ticket), the correction mechanism
(`/opsx:update`, per-feature not per-backlog-item).

**Still open, required before propose conversations start cleanly:**
- The ~12-15 feature list is a plausible candidate set, not yet finalized: `identity-and-access`,
  `access-control-and-admin`, `design-system`, `platform-core`, `ai-platform-governance`,
  `hiring-postings`, `candidate-intake`, `matching-and-ranking`, `interview-pipeline`,
  `decision-and-offers`, `insight-and-reporting`, `resurfacing-and-communications` (~12) — needs
  reconciling against the full ~86-item decomposition once that's done.
- The full ~86-item decomposition is not complete — only a partial worked example exists for
  Phase 1 (`TS-BL-001` through `TS-BL-010`, ~10 of its estimated ~29).
- `talentsphere-wave-1-foundation`'s eventual migration target changes from "9 backlog-item
  changes" (D.5) to a smaller number of feature-changes covering the same ground — D.5's
  cross-reference-preservation guidance still applies, just at feature rather than backlog-item
  granularity.
- The full sprint/wave schedule (D.6's 8-sprint, then corrected 14-sprint plan) was built at the
  old per-item-change granularity and needs redoing at the corrected finer grain — worth doing
  once the full decomposition exists, since smaller items may compress the critical path
  differently than the 14-sprint estimate did.

`AGENTS.md` has been updated in place to reflect this correction (propose-per-feature, corrected
naming, corrected wave rules) — it did not need a superseded-but-preserved history the way this
decision log does, since it's a living conventions document, not a decision record.

## D.9 — Phase 1 backlog decomposition at the corrected finer grain (2026-08-25)

**Correcting a documentation gap found during the 2026-08-25 consistency audit:** D.8.4's claim
that "a partial worked example exists for Phase 1 (`TS-BL-001` through `TS-BL-010`)" was
aspirational, not actual — no such table existed anywhere in this repo. This section is that
missing decomposition, built fresh from D.3's nine old Phase-1 items (`TS-BL-0000`–`TS-BL-0007`,
old 4-digit numbering) using D.8.1's guidance to preserve dependency *relationships* without
treating the old table as a literal change list.

**This also settles the first open item in D.8.4 for Phase 1's slice of the feature list**: the
nine old items map cleanly onto exactly 5 of the 12 candidate features, with no leftover and no
overlap — `platform-core`, `design-system`, `identity-and-access`, `access-control-and-admin`,
`ai-platform-governance`. That clean mapping is itself evidence the candidate feature list's
Phase-1 slice is correctly cut, not just plausible.

**Result: 32 backlog items, against a ~29 estimate** — close enough not to force a fake number
either direction; the three extra came from splitting the audit substrate's write/query/break-glass
integration into separate items once actually decomposed, which the original estimate couldn't
have seen at that grain.

| Backlog ID | Item | Feature | Depends on | Old D.3 source |
|---|---|---|---|---|
| TS-BL-001 | Base infra: workload Terraform skeleton + CI/CD pipeline | platform-core | — | TS-BL-0000 |
| TS-BL-002 | Cloud SQL + IAM DB auth wiring | platform-core | TS-BL-001 | TS-BL-0000 |
| TS-BL-003 | API Gateway ingress skeleton | platform-core | TS-BL-001 | TS-BL-0000 |
| TS-BL-004 | Workflow / state-machine engine substrate (Application stage transitions) | platform-core | TS-BL-002 | TS-BL-0005 |
| TS-BL-005 | Notification engine (internal delivery) | platform-core | TS-BL-002 | TS-BL-0005 |
| TS-BL-006 | Async orchestration pattern (Pub/Sub → Eventarc → dispatcher → Workflows → Cloud Run Job) | platform-core | TS-BL-001, TS-BL-004 | TS-BL-0005 |
| TS-BL-007 | Design tokens (color/type/spacing/shape, light + dark) | design-system | — | TS-BL-0004 |
| TS-BL-008 | Authenticated Shell (sidebar 3-states) | design-system | TS-BL-007 | TS-BL-0004 |
| TS-BL-009 | Dense-data-table pattern component | design-system | TS-BL-007 | TS-BL-0004 |
| TS-BL-010 | Floating Action Panel component | design-system | TS-BL-007 | TS-BL-0004 |
| TS-BL-011 | Core form / input components | design-system | TS-BL-007 | TS-BL-0004 |
| TS-BL-012 | Notification / toast component | design-system | TS-BL-007, TS-BL-005 | TS-BL-0004 |
| TS-BL-013 | User model & Hubble SSO login | identity-and-access | TS-BL-001, TS-BL-003 | TS-BL-0001 |
| TS-BL-014 | Session issuance & token handling | identity-and-access | TS-BL-013 | TS-BL-0001 |
| TS-BL-015 | Forced session revocation (G-11) | identity-and-access | TS-BL-014 | TS-BL-0001 |
| TS-BL-016 | Account activation / first-login flow | identity-and-access | TS-BL-013 | TS-BL-0001 |
| TS-BL-017 | Login → audit event hook | identity-and-access | TS-BL-014 | TS-BL-0001 |
| TS-BL-018 | Permission evaluator engine (role × page × action, deny-wins) | access-control-and-admin | TS-BL-014 | TS-BL-0002a |
| TS-BL-019 | Explanation / introspection endpoint ("why can/can't I") | access-control-and-admin | TS-BL-018 | TS-BL-0002a |
| TS-BL-020 | Audit log write substrate (append-only, correlation ID) | access-control-and-admin | TS-BL-002, TS-BL-017 | TS-BL-0002a |
| TS-BL-021 | Audit log query UI (Auditor role) | access-control-and-admin | TS-BL-020 | TS-BL-0002a |
| TS-BL-022 | Seed data: 9-role permission matrix | access-control-and-admin | TS-BL-018 | TS-BL-0002b |
| TS-BL-023 | Admin Cockpit UI shell (role assignment, override management) | access-control-and-admin | TS-BL-022, TS-BL-008 | TS-BL-0003 |
| TS-BL-024 | UserPermissionOverride grant/deny flow (mandatory reason) | access-control-and-admin | TS-BL-023 | TS-BL-0003 |
| TS-BL-025 | Break-glass request / approval / expiry flow | access-control-and-admin | TS-BL-024 | TS-BL-0003 |
| TS-BL-026 | Break-glass → audit trail integration | access-control-and-admin | TS-BL-025, TS-BL-020 | TS-BL-0003 |
| TS-BL-027 | AI Gateway (sole egress to model provider) | ai-platform-governance | TS-BL-003 | TS-BL-0006 |
| TS-BL-028 | AIRun logging (logged before invocation) | ai-platform-governance | TS-BL-027, TS-BL-002 | TS-BL-0006 |
| TS-BL-029 | Advisory-only structural enforcement (no transition capability) | ai-platform-governance | TS-BL-027, TS-BL-004 | TS-BL-0006 |
| TS-BL-030 | Prompt template registry (10 families, versioned) | ai-platform-governance | TS-BL-027 | TS-BL-0007 |
| TS-BL-031 | Evidence-source labeling substrate | ai-platform-governance | TS-BL-028 | TS-BL-0007 |
| TS-BL-032 | AI evaluation harness (adversarial testing, C-09) | ai-platform-governance | TS-BL-030 | TS-BL-0007 |

Notes on the cross-feature dependency edges (the ones that aren't just "next item in the same
feature"):
- `TS-BL-013` (identity-and-access) depends on `TS-BL-003` (platform-core's API Gateway) because
  SSO login has to land somewhere real to redirect through.
- `TS-BL-018` (access-control-and-admin's evaluator) depends on `TS-BL-014` (session/token
  handling), not `TS-BL-013` — you can't evaluate a permission without a session to evaluate it
  for.
- `TS-BL-020` (audit write substrate) depends on both `TS-BL-002` (DB wiring, since audit rows are
  DB rows) and `TS-BL-017` (the login event hook, since that's the first event it needs to accept).
- `TS-BL-023` (Admin Cockpit UI) depends on `TS-BL-008` (design-system's Authenticated Shell), not
  `TS-BL-009`'s dense-data-table — confirmed against the real Cockpit's actual page structure
  needs a shell first; the table pattern is consumed later, inside individual Cockpit screens, not
  tracked as a top-level dependency here.
- `TS-BL-012` (design-system's toast component) depends on `TS-BL-005` (platform-core's
  notification engine) because a toast needs a real payload shape to render against, not just a
  visual mock.

This does not touch `talentsphere-wave-1-foundation` — per D.5/D.8.4, that change's real, already-
built content still needs migrating onto whichever of these finer items match it; this table is
the target shape for that migration, not a redo of work already shipped in Sprint 0.

**Still open after this section:** Phases 2–5's decomposition (~54-58 more items), the full
feature-list finalization (Phase 1's 5 features are now validated; the other ~7-10 candidates for
Phases 2–5 are still unvalidated), and the sprint/wave schedule redo. Phase 1's own decomposition
is complete.

## D.10 — Phases 2-5 backlog decomposition, and the feature list is now finalized (2026-08-25)

Same method as D.9: take D.3's old Phase 2-5 items, decompose each at the finer grain, translate
old dependency edges to their real finer-grain enabler rather than copying them mechanically (per
D.8.1's "relationships to preserve, not a literal list"), and check whether the result maps
cleanly onto the remaining 7 candidate features.

**It does, with no leftover and no overlap** — the same clean-mapping result D.9 found for Phase 1:

| Old D.3 items | → Feature |
|---|---|
| TS-BL-0008 JD Workspace, TS-BL-0009 Job Posting Mgmt | `hiring-postings` |
| TS-BL-0010 Resume Intake, TS-BL-0011 Candidate DB & Dedup | `candidate-intake` |
| TS-BL-0012 Vector Retrieval, TS-BL-0013 AI Ranking Engine, TS-BL-0014 Ranking Board UI | `matching-and-ranking` |
| TS-BL-0015 Shortlisting, TS-BL-0016 Interview Console, TS-BL-0017 Scorecard Center | `interview-pipeline` |
| TS-BL-0018 Priority Selection & Offer Tracker, TS-BL-0019 Closure Rules & Evergreen | `decision-and-offers` |
| TS-BL-0021 Dashboards & Reporting | `insight-and-reporting` |
| TS-BL-0020 Resurfacing & Priority Lane, TS-BL-0022 Calendar & Candidate Comms | `resurfacing-and-communications` |

**The 12-candidate feature list is therefore finalized as-is** — all 12 (5 from D.9 + 7 here) are
now validated by a clean decomposition, not just plausible. No feature needs renaming, merging,
or splitting.

### Phase 2 — Core Hiring Loop

| Backlog ID | Item | Feature | Depends on |
|---|---|---|---|
| TS-BL-033 | JD drafting workspace (PM drafts, manual) | hiring-postings | TS-BL-018 |
| TS-BL-034 | AI-drafted JD generation | hiring-postings | TS-BL-033, TS-BL-030 |
| TS-BL-035 | JD approval workflow (RM approves, C-12) | hiring-postings | TS-BL-034 |
| TS-BL-036 | JD versioning & fork-on-edit | hiring-postings | TS-BL-035 |
| TS-BL-037 | Job Posting creation (FINITE(n)\|EVERGREEN) | hiring-postings | TS-BL-036 |
| TS-BL-038 | Posting status lifecycle (draft→open→...→closed) | hiring-postings | TS-BL-037 |
| TS-BL-039 | Posting text separate from internal JD (G-08) | hiring-postings | TS-BL-037 |
| TS-BL-040 | Compliance text handling (G-09) | hiring-postings | TS-BL-039 |
| TS-BL-041 | Resume upload endpoint + malware scan | candidate-intake | TS-BL-001 |
| TS-BL-042 | Resume file hash (G-04) | candidate-intake | TS-BL-041 |
| TS-BL-043 | Deterministic contact-field extraction (hybrid parsing) | candidate-intake | TS-BL-041 |
| TS-BL-044 | LLM enrichment layer (skills/experience) | candidate-intake | TS-BL-043, TS-BL-027 |
| TS-BL-045 | Extraction confidence scoring (G-05) | candidate-intake | TS-BL-044 |
| TS-BL-046 | Candidate identity model & dedup logic (D23) | candidate-intake | TS-BL-041 |
| TS-BL-047 | Duplicate backstop at posting level (D23a) | candidate-intake | TS-BL-046, TS-BL-037 |
| TS-BL-048 | Resume versioning (resume_version_ref) | candidate-intake | TS-BL-046 |
| TS-BL-049 | Vector embedding pipeline (resume → embedding) | matching-and-ranking | TS-BL-044 |
| TS-BL-050 | Vertex AI Vector Search integration (C-01) | matching-and-ranking | TS-BL-049 |
| TS-BL-051 | RankingScore tuple model | matching-and-ranking | TS-BL-050, TS-BL-030 |
| TS-BL-052 | AI Ranking Engine (fitment/gap scoring) | matching-and-ranking | TS-BL-037, TS-BL-051, TS-BL-027 |
| TS-BL-053 | Ranking Board UI | matching-and-ranking | TS-BL-052, TS-BL-009, TS-BL-018 |
| TS-BL-054 | AI context visibility rules for Interviewers (D05/C-10) | matching-and-ranking | TS-BL-053 |
| TS-BL-055 | Insufficiency outputs (G-03) | matching-and-ranking | TS-BL-052 |

23 items against a ~25 estimate.

### Phase 3 — Funnel, Evaluation & Decision

| Backlog ID | Item | Feature | Depends on |
|---|---|---|---|
| TS-BL-056 | Shortlisting decision UI (reason required on every disposition) | interview-pipeline | TS-BL-053, TS-BL-004 |
| TS-BL-057 | Interview round scheduling | interview-pipeline | TS-BL-056 |
| TS-BL-058 | Interview Console (structured notes, multiple rounds) | interview-pipeline | TS-BL-057 |
| TS-BL-059 | Interview note versioning without lock (C-08/D24) | interview-pipeline | TS-BL-058 |
| TS-BL-060 | Scorecard Center: AI-drafted generation | interview-pipeline | TS-BL-058, TS-BL-027 |
| TS-BL-061 | Scorecard human-approval workflow | interview-pipeline | TS-BL-060 |
| TS-BL-062 | Consolidated scorecard across rounds (C-05/D11) | interview-pipeline | TS-BL-061 |
| TS-BL-063 | Fixed scorecard dimensions/competencies (C-06/D15) | interview-pipeline | TS-BL-062 |
| TS-BL-064 | Priority Slot mechanics (D20: up to 5 × vacancy) | decision-and-offers | TS-BL-062 |
| TS-BL-065 | Priority Selection UI (Floating Action Panel, S.1) | decision-and-offers | TS-BL-064, TS-BL-010 |
| TS-BL-066 | Carry-forward eligibility & scorecard reuse (D17/D21/C-07) | decision-and-offers | TS-BL-065 |
| TS-BL-067 | Offer creation & tracking | decision-and-offers | TS-BL-065 |
| TS-BL-068 | Onboarding record & Hubble ID capture (D08) | decision-and-offers | TS-BL-067 |
| TS-BL-069 | Closure rules (onboarded + valid Hubble ID, G-13) | decision-and-offers | TS-BL-068 |
| TS-BL-070 | Evergreen posting lifecycle rules (D14) | decision-and-offers | TS-BL-069, TS-BL-038 |

15 items against a ~19 estimate — honestly under, not padded; see note below.

### Phase 4 — Insight & Reporting

| Backlog ID | Item | Feature | Depends on |
|---|---|---|---|
| TS-BL-071 | Recruiter/PM workload dashboard | insight-and-reporting | TS-BL-069, TS-BL-009 |
| TS-BL-072 | Recruitment Manager pipeline oversight dashboard | insight-and-reporting | TS-BL-071 |
| TS-BL-073 | Closure-readiness reporting view | insight-and-reporting | TS-BL-069 |
| TS-BL-074 | Audit/AI-run reporting surface (Auditor role) | insight-and-reporting | TS-BL-028, TS-BL-021 |

4 items against a ~5 estimate.

### Phase 5 — Resurfacing & Communications

| Backlog ID | Item | Feature | Depends on |
|---|---|---|---|
| TS-BL-075 | MatchSuggestion resurfacing engine | resurfacing-and-communications | TS-BL-052, TS-BL-069 |
| TS-BL-076 | Priority lane precedence logic (selected-but-not-offered) | resurfacing-and-communications | TS-BL-075, TS-BL-064 |
| TS-BL-077 | Internal email notification delivery (D04: internal early) | resurfacing-and-communications | TS-BL-005 |
| TS-BL-078 | Candidate-facing communication gate (deferred pending legal sign-off, OD-005) | resurfacing-and-communications | TS-BL-077 |
| TS-BL-079 | Calendar integration (deferred per D06/C-04, stub scope only) | resurfacing-and-communications | TS-BL-057 |

5 items against a ~8 estimate — Phase 5's estimate assumed full calendar/candidate-comms scope;
D04/D06/C-04 already deferred most of that to a later phase, so 5 real items is what's actually
in scope today, not a shortfall.

### Notable dependency-translation calls (old edge → real finer-grain enabler)

- **TS-BL-0014's old dependency on TS-BL-0003 (Admin Cockpit UI)** did not carry forward literally
  — Ranking Board UI has no functional relationship to the Admin Cockpit. Translated to what it
  actually needs: `TS-BL-009` (design-system's dense-data-table pattern, which is what a ranking
  board *is*) and `TS-BL-018` (the permission evaluator, since C-10's "gaps yes, score no" display
  rule for Interviewers is a permission-gated rendering decision). This is exactly the kind of
  mechanical-copy trap D.8.1 warned about.
- `TS-BL-047` (posting-level dedup backstop) depends on `TS-BL-037` (Job Posting creation) in
  addition to `TS-BL-046` (candidate dedup) — a posting-scoped backstop needs postings to exist
  before it can scope anything to one.
- `TS-BL-070` (Evergreen lifecycle rules) depends on `TS-BL-038` (posting status lifecycle) as well
  as closure rules — evergreen is a variant of the posting lifecycle, not a bolt-on.
- `TS-BL-078` (candidate-facing comms) is recorded as blocked on legal sign-off (`OD-005`) as well
  as its technical dependency — a real blocker, not just a sequencing note.

### Running total

| Phase | Items | Estimate |
|---|---|---|
| 1 — Foundation & Governance (D.9) | 32 | ~29 |
| 2 — Core Hiring Loop | 23 | ~25 |
| 3 — Funnel, Evaluation & Decision | 15 | ~19 |
| 4 — Insight & Reporting | 4 | ~5 |
| 5 — Resurfacing & Communications | 5 | ~8 |
| **Total** | **79** | **~86** |

79 lands under the ~86 estimate by about 8%, mostly from Phase 5's deferred scope. Left as the
honest number rather than adjusted to match the earlier projection — consistent with this
document's practice of reporting real critical-path/decomposition results even when they don't
match an earlier estimate (see D.6's 8→14-sprint correction).

**This closes the feature-list-finalization item from D.8.4.** All 12 features — `identity-and-
access`, `access-control-and-admin`, `design-system`, `platform-core`, `ai-platform-governance`,
`hiring-postings`, `candidate-intake`, `matching-and-ranking`, `interview-pipeline`, `decision-
and-offers`, `insight-and-reporting`, `resurfacing-and-communications` — are validated, and the
complete 79-item backlog (`TS-BL-001`–`TS-BL-079`) is decomposed with dependencies. The only item
left open from D.8.4 is the sprint/wave schedule redo at this corrected grain.

### D.10.1 — GAP CORRECTION: ownership reassignment was asserted twice and decomposed never (2026-08-25)

**Found during `identity-and-access`'s propose conversation**, under the convention `AGENTS.md`
records for fixing genuine gaps in *this* document rather than working around them — the same
convention `platform-core` used for D09's superseded Cloud Tasks row.

**What was wrong.** Two separate places in this document assert that ownership reassignment is
needed work:

- [D08](#d08--hubble-id-at-onboarding), on the leaver problem: *"a departing recruiter who owned
  12 open postings needs **ownership reassignment** (not in the original feature list)."*
- [G-11](#g-11--forced-session-revocation), adopted from the reference spec: *"Pairs with
  ownership reassignment, still open."*

Neither was ever closed, and **no item in D.9 or D.10's 79 covers it.** The nearest candidates do
not: `TS-BL-016` is account activation, `TS-BL-023`/`TS-BL-024` are role assignment and permission
overrides — *who a user is and what they may do*, not *what work they hold*. Meanwhile
`talentsphere-wave-1-foundation`'s `identity/user-activation` delta spec already carries a
**Leaver handling and ownership continuity** requirement stating that owned items "remain
reassignable to another user." So a written spec requirement exists with no backlog item to build
it, while D.10 and D.11 both declare the decomposition complete. That is the contradiction: the
completeness claim and the still-open work cannot both be true.

**Why it was missed.** D.9 and D.10 decomposed forward from D.3's old 23-item table. Ownership
reassignment was never *in* that table — D08 says so in its own parenthesis — so a decomposition
that faithfully translated every old item was structurally incapable of finding it. Faithful
translation was the right method; it just cannot surface work that no source item contained.

**What supersedes this now.** One new item, at the same grain as its neighbours:

| Backlog ID | Item | Feature | Depends on |
|---|---|---|---|
| TS-BL-080 | Ownership reassignment on deactivation (departing user's owned work handed to a named successor, attribution on historical records preserved) | access-control-and-admin | TS-BL-016, TS-BL-023 |

**Why `access-control-and-admin` and not `identity-and-access`:** deactivation is performed from
the Admin Cockpit (`ADM-001`), and reassignment is the administrator's action in the same
sitting — putting it anywhere else would split one screen's behaviour across two features.
`identity-and-access` keeps the *invariant* (deactivating a user revokes their sessions and must
neither delete, orphan, nor silently re-attribute what they owned); `TS-BL-080` builds the
*workflow* that satisfies it. `access-control-and-admin`'s own propose conversation owns the
detail, and may refine the item boundary per D.11.

**Running total is therefore 80, not 79** (`TS-BL-001`–`TS-BL-080`). D.10's table above is left
unedited on purpose — it is the accurate record of what that decomposition produced; this block
is what corrects it. Phase 1's count moves from 32 to 33, and
`access-control-and-admin`'s range from `TS-BL-018`–`TS-BL-026` to those nine plus `TS-BL-080`
(non-contiguous, which is expected — `AGENTS.md` fixes IDs globally and permanently, so a later
discovery takes the next free number rather than renumbering anything).

**Still genuinely open, and deliberately not closed here:** the *direction* of leaver detection —
whether Hubble pushes deactivation, TalentSphere polls, or a leaver is discovered on failed login
(`D08`). `TS-BL-080` is triggered by a TalentSphere-side deactivation however that deactivation
arrives, so it is not blocked on the answer; `OD-001` still is.

## D.11 — Propose before schedule, and explore is now complete (2026-08-25)

**Decision:** all 12 features go through `/opsx:propose` first; the sprint/wave schedule redo
(D.8.4's last open item) happens after, in its own dedicated conversation — not before, and not
interleaved feature-by-feature.

**Why:** `AGENTS.md`/D.4 already commit to generating full artifacts for every feature before most
of them are built — this ordering was implicit in that decision, not a new one. Made explicit here
because it also settles *how* to schedule well: right now the 79 items in D.9/D.10 are title-level
guesses with no real effort signal; each feature's finished `tasks.md` will carry actual design
detail per item. Scheduling against that is strictly better than scheduling against guesses and
redoing it once designs land — and the dependency order proposing should happen in (D.9/D.10's
graph) doesn't need a schedule to determine, so nothing is blocked by deferring it.

**How to apply:** the next conversation after this one starts `/opsx:propose` for the first
feature in dependency order — `platform-core` or `design-system` (neither depends on anything).
The delivery-scheduling conversation (`delivery/roadmap.md`, `release-plan.md`,
`sprints/.../waves/*.md`) is its own separate conversation, run only after all 12 features'
`tasks.md` exist.

**This closes explore.** Every item D.8.4 opened is now resolved: the feature list is finalized
(D.10), the full 79-item backlog is decomposed with dependencies (D.9/D.10), and the propose-vs-
schedule ordering is decided (this section). Nothing else is blocking the first propose
conversation. The five-phase, twelve-feature, seventy-nine-item shape of the plan is considered
settled — a feature's own propose conversation may still refine its internal item boundaries
(splitting or merging within that feature, per D.8.1), but should not need to revisit which
feature something belongs to or reopen a D/C/G/S decision without a real reason surfacing.
